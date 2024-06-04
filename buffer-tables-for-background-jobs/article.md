# Buffer Tables for Background Jobs

Here is a pattern I've seen several times, especially in applications that have a data-intensive integration with a third-party service.

Suppose we have the following method for recording that a sales `Lead` has been contacted.  We save the `ContactActivity` in our database, but we also have a third-party marketing service that needs to be apprised of this fact.  We accomplish this by enqueuing a background job that sends this update via REST API to that service. Something like this:

```ruby
class Lead < ApplicationRecord
  has_many :contact_activities

  def record_activity!(type:, comment:)
    ContactActivity.new(type:, comment:).tap do |activity|
      contact_activities << activity
      save!
      RecordMarketingActivityJob.perform_async(activity.id)
    end
  end
end
```

In general, it is a good idea to scope background jobs to small, single transactions.  Reasons include:

- It is easier to make them idempotent.
- Single job failures affect at most a single transaction.
- They are easier to reason about.
- They will generally complete quicker.  Long-running jobs are problematic.  They can delay deployments or risk being interrupted, potentially leaving data in an inconsistent state.

But what happens when we scale up our sales operation and start recording, say, 10s or 100s of thousands of contact activities per day?  We'll be enqueuing 100s of thousands of background jobs, each sending an API request to that marketing service.  We'll start hitting rate or concurrent connection limits for that API; job queues will back up; data sync latency will increase; marketers will complain.

Fortunately, the marketing service's API includes endpoints for bulk operations.  Instead of sending a single update per request, we can send, say, 1,000 per request.  This would cut down our job and request volume by a few orders of magnitude and make it easier to avoid the above issues.

We don't, however, want to restructure all of our code and APIs for recording contact activities to accommodate this background process.  Ideally, we would keep our existing single-record operations for recording contact activities, but take advantage of bulk operations for syncing these with the marketing service.  In other words,  with respect to the number of records we process at a time, we want to keep the main operation decoupled from handling its side effect.  These have different reasons to change--the requirements of the users of our application for the former and constraints imposed by an external service for the latter.

In this article, I'll describe one strategy for doing this.  We are going to keep it [boring](https://boringtechnology.club/) and use technologies that you are probably already using.  In particular, we'll assume we're using Postgres and Sidekiq as our job processor (using Redis).  But you could probably apply the same strategy using your preferred RDBMS and background job runner of choice.

## Buffer table

The basic idea is to have a database table dedicated to storing events that we want to process in batch background jobs.  At the point in our code where we would enqueue a single-record job, we instead insert into this table.  We will then have a scheduled job that polls this table every few minutes (or whatever time interval makes sense for our use-case) and dispatches batches of records to jobs to perform the target operation(s) in bulk.

The changes to our example method above will be small.  Instead of enqueuing a `RecordMarketingActivityJob` with the newly created `ContactActivity`'s ID, we create a `ContactActivityEvent` associated to the `ContactActivity`.  The table behind this model `contact_activity_events` will be our "buffer table" dedicated to batch processing.

```ruby
class Lead < ApplicationRecord
  has_many :contact_activities

  def record_activity!(type:, comment:)
    ContactActivity.new(type:, comment:).tap do |activity|
      activity.contact_activity_events << ContactActivityEvent.new
      contact_activities << activity
      save!
    end
  end
end
```

By using a database table, we get a few of benefits for free:

- *Transactions*: Enqueuing an event for background processing can now participate in a transaction with the main operation.  We can guarantee that, for example, either the contact activity was successfully created *and* an associated event was enqueued or neither operation completed.
- *Data integrity*: We can enforce familiar data integrity checks on our events.  We should have a foreign key constraint on our contact activity events table referencing our contact activities table to ensure that we don't have any orphaned events.  Also, for example, suppose we want to guarantee that we are only handling a single event per contact activity at a time.  We could add a unique constraint to the `contact_activity_id` column to enforce that.
- *Familiarity*: We can use the same tools (e.g., ActiveRecord) and patterns that we are already using to interact with the database.

One downside, however, especially when compared to an alternative implementation in, say, Redis, is that we could quickly run into performance issues with a naive implementation and a relatively high volume of events.  Assuming our buffer table is in our application database, performing updates and queries on this table could end up consuming database resources and slowing down other areas of our application.  Let's consider some strategies for avoiding this.

### Performance heuristics

The main performance heuristic is to keep the buffer table small and single-purpose.  Its purpose is to store and track events for batch processing.  Once a job has finished with an event, it should be deleted.  If you need to keep these events in your database for logging or reporting, copy them to another table (an "archive" or "completed" table) before deleting them in the same transaction.

We will be querying this table at least every few minutes or so.  Controlling the table size will make it easier to keep these queries fast.  But this workload will also be relatively write-heavy.  We will be inserting each event one at a time; these will be updated when they are assigned to a batch; and then they will be deleted.  Accordingly, we should not add indexes to the table that do not directly support these operations, since they would unnecessarily slow down writes.  Again, if you need to query these events for other purposes, use a separate table.

Also, after the initial event creation, we should avoid individual updates and deletes.  If our batch size is 1,000, we should update/delete a batch in a single database command instead of issuing 1,000 separate update/delete commands.  This will probably be faster since it requires fewer network round-trips and it reduces overall load on the database.

Finally, if possible, we should take advantage of our background job processor and run many batch jobs concurrently.  We can process more events in less time, i.e., increase throughput.  This also reduces the likelihood of the buffer table growing to an unwieldy size during periods of high volume, since we are processing and deleting them at a faster rate.  (Although, note that we will still need to stay within the rate limiting confines of the marketing service API.)

## Implementation

What does an implementation look like that follows these performance heuristics?

### Schema

The schema for the buffer table will be simple.  We will have the typical boilerplate (primary key, timestamp columns) and whatever application-specific columns we need.  In the present case, this would be a foreign key to `contact_activities`.  We will probably want an index on this column, since we will join this table upon selecting the `contact_activity_events` so that we have the requisite information to send to the marketing service.

The only batch processing specific column will be a nullable `uuid` column called `group_id`, which identifies the batch a record is a member of.  We will put a `BTREE` index on this column, since this will be our main query filter after events have been assigned to a group/batch.  This `group_id` will allow us to efficiently query and delete records by batch.

### Jobs

For our background jobs, there will be two main phases.  In the first phase, events will be grouped or "claimed" by a job.  Claiming events consists of assigning them a `group_id` and enqueuing a job, call it Events Job, to carry out the actual target operation(s) for the events in that group.  The Events Job performs the second phase.  It takes a `group_id` as an argument and selects the events with that `group_id`, probably `JOIN`ing other tables.  Then, it performs the target operation and deletes the events with the given `group_id`.  (That is, if the operation is successful.  We'll address failures and retries [later](#failures-and-retries).)

A simple implementation would have a scheduled Claim Job that runs every few minutes or so and does the following

  1. Select `batch_size` number of IDs of unclaimed events (i.e., where `group_id` is `NULL`).  If there are none, exit.  Otherwise:
  2. Generate a `group_id` using `SecureRandom.uuid`.
  3. Issue a bulk update setting that `group_id` for those records.
  4. Enqueue an Events Job with `group_id` as an argument.
  5. Repeat until either all events in the table are claimed or we reach some configured limit of number of batches per job.  This is to avoid very long-running jobs and/or overlap with the next scheduled job Claim Job.

![A simple two job set up with a looping claim job](images/two_jobs.png)

The main benefit of this approach is that it is straightforward.  It will probably work fine in cases where we have a consistent volume of events and where maximizing throughput is not a priority.  However, there are some tradeoffs.

The Claim Job is not designed for concurrency.  This puts a potentially significant limit on how quickly we can claim events and enqueue Events Jobs, since we query for unclaimed events one at a time.  If we get a big spike in events such that these are being inserted at a faster rate than we are claiming them and we are limiting how many queries run per Claim Job, the job will end before all events are claimed.  The claiming process can fall behind, growing the size of the events table and possibly slowing down queries.  It might take a while for these to catch up and claim all of the events after the spike subsides, delaying the target operation.

Also, we could end up with multiple Claim Jobs running concurrently, unless we take measures[^1] to prevent this.  There could be an unrelated backup of jobs in the same queue.  Claim Jobs would be enqueued on schedule, but if the backup is long enough, they would get stuck in the queue such that multiple Claim Jobs are enqueued before any one runs.  Once the backup subsides, they could get picked up by different worker threads and run concurrently.  This presents a problem if the queries for unclaimed events do not include any concurrency controls, like row-level locking.  Two Claim Jobs running concurrently could end up "claiming" the same events and overwriting each other's `group_id`s.  This could have unexpected results, like an Events Job running with a `group_id` that no longer exists.

To fix the potential concurrency issues with these queries, we can add the `FOR UPDATE SKIP LOCKED` phrase on our query for unclaimed events.  `FOR UPDATE` acquires a row-level lock on the batch of unclaimed records, which prevents another transaction (e.g., in a concurrent Claim Job) from modifying that row until the current transaction ends.  `SKIP LOCKED` says what to do if this query encounters a row that already has a lock on it.  Namely, it skips it, behaving like the row doesn't exist for this query.  This is fine for our purposes, since we only care that the record is being claimed by some single Claim Job.

Here's what our `ContactActivityEvent` might look like with this modification:

```ruby
class ContactActivityEvent < ApplicationRecord
  belongs_to :contact_activities
  
  scope :unclaimed, -> { where(group_id: nil) }

  def self.claim(limit)
    raise ArgumentError, 'block required' unless block_given?

    transaction do
      ids = unclaimed.lock('FOR UPDATE SKIP LOCKED').limit(limit).pluck(:id)
      if ids.present?
        group_id = SecureRandom.uuid
        where(id: ids).update_all(group_id:) # <- Bulk assign events to group
        yield(group_id)
      end
      ids
    end
  end
end

# ClaimJob would then claim events like this:
ContactActivityEvent.claim(Rails.configuration.batch_size) do |group_id|
  ContactActivityEventsJob.perform_async(group_id)
end
```

We can now safely run Claim Jobs concurrently in the off-chance this happens.  But this structure, while simple, is not ideal for maximizing throughput or handling large spikes in events.  It does not really take advantage of the concurrency now available to us.  If we need this for our use-case, we can do something like the following.

Instead of having a scheduled job that assigns batches one at a time from the buffer table, we can have many jobs running concurrently, each responsible for assigning a single batch (if available).  In effect, we put each iteration of the loop into its own job.  Our Claim Job, then, would simply run the lines above for claiming a single batch of events if they exist:

```ruby
class ClaimContactActivityEventsJob
  include Sidekiq::Job

  def perform
    ContactActivityEvent.claim(Rails.configuration.batch_size) do |group_id|
      ContactActivityEventsJob.perform_async(group_id)
    end
  end
end
```

These single batch Claim Jobs won't be scheduled.  Instead, they'll be enqueued by a new scheduled job, called a 'Distribution Job'.  The purpose of this job is to enqueue the number of Claim Jobs it estimates will be needed to batch up the remaining unclaimed events.  A simple estimate of the number of needed Claim Jobs is to divide the current count of unclaimed events by the batch size, rounded up.  Of course, this is not guaranteed to be accurate, since events could be enqueued or claimed by the time those Claim Jobs actually run.  But, in general, this estimate does not need to be particularly accurate.  If we underestimate, we should have another scheduled Distribution Job running in a few minutes to pick up the remaining unclaimed events.  If we overestimate, some number of Claim Jobs will find no unclaimed events and complete without enqueuing an Events Job.  The Claim Job's query for unclaimed events should be inexpensive, given that we have an index on `group_id` and we've kept the buffer table small.  Thus a few wasted queries is not a big deal in most cases.

![A three job set up to increase throughput and limit job duration](images/three_jobs.png)

Like in the simpler set up above, we probably want to put an upper bound on the number of Claim Jobs enqueued by a single Distribution Job.  In extreme, unanticipated spikes of events, we don't want to end up dumping thousands or more Claim Jobs into our queue.  Our Distribution Job, then, might look something like this:

```ruby
class DistributeContactActivityEventsJob
  include Sidekiq::Job

  def perform
    available_batches_count = (unclaimed_events_count / batch_size.to_f).ceil
    args = [available_batches_count, max_batches].min.times.map { [] }
    ClaimContactActivityEventsJob.perform_bulk(args)
  end

  private

  def unclaimed_events_count
    ContactActivityEvent.unclaimed.count
  end

  def batch_size
    Rails.configuration.batch_size
  end

  def max_batches
    Rails.configuration.max_distribution_batches
  end
end
```

Note that we estimate the number of Claim Jobs to enqueue based on the current unclaimed events count alone.  We do not consider the number of Claim Jobs currently enqueued and waiting to run.  In cases where the queue for Claim Jobs has a prolonged back up for some reason, we could end up enqueuing a bunch of unnecessary Claim Jobs, since the number of unclaimed events is not decreasing.  Again, the downside of this is that we have a bunch of unnecessary, but hopefully inexpensive queries running against our database.  This is not ideal, but it's potentially an acceptable tradeoff for keeping our Distribution Job simple, especially if such queue slowdowns are rare.[^2]

### Error handling

A nice feature of Sidekiq (and other background job processors) is that it will handle [most aspects of job failures and retries for us](https://github.com/sidekiq/sidekiq/wiki/Error-Handling#automatic-job-retry).  We don't want to reimplement this ourselves with our buffer table.  And we don't have to.

Let's consider the Events Job.  It is the only interesting case since it is associated with a group of events and actually carries out the target operation, which is likely to have at least some intermittent failures.  By giving a `group_id` as the job argument, Sidekiq will persist it as such and we can offload the failure and retry logic for that group to Sidekiq.  No other job is going to process these events, since the Claim Job only selects events without `group_id`s.  We only need to act at the terminal points of the job to "complete" the events, i.e., delete them from the buffer table.  Either an event was successfully processed (after some number of retries) or all of the job's retries were exhausted by repeated failures.  For the former, we delete the event at the end of the job's `#perform` method.  For the latter, we use Sidekiq's `sidekiq_retries_exhausted` callback and delete any remaining events from the group there, along with whatever error logging, etc. we need to perform.

Here's an example:

```ruby
class ContactActivityEventsJob
  include Sidekiq::Job
  sidekiq_options retry: 3

  sidekiq_retries_exhausted do |msg|
    group_id = msg['args'].first
    # Possibly do something here with these records to perform compensating actions.
    ContactActivityEvent.where(group_id:).delete_all
    Rails.logger.error("ContactActivityEvent group #{group_id} exhausted retries: #{msg['error_message']}")
  end

  def perform(group_id)
    events = ContactActivityEvent.includes(:contact_activity).where(group_id:)
    sync_with_marketing(events) # Send request(s) to marketing service API here. Raise on retryable failure.
    ContactActivityEvent.where(group_id:).delete_all
  end
end
```

Notice that our `#perform` method does not consider individual events for deletion; it deletes them in bulk as a group.  This is the simplest case, where our target operation is all-or-none--either all of the events in the group succeed or none do.

If the operation is not all-or-none, we will instead delete only the ones that succeed and raise an error if some failed so that the job can be retried.  The initial query to get the events to be processed can remain the same.  Only events from that group that failed will remain in the table when the job is retried.  Same goes for the bulk deletion in the `sidekiq_retries_exhausted` callback.

## Conclusion

The scenario we considered is relevant in more cases than one would initially think.  Applications are increasingly integrating with more external systems, each carrying its own performance profile and constraints.  It's not uncommon that two or more of these are incompatible if we want optimal or even acceptable performance.  By using something like this buffer table strategy, we can decouple these integration points and tune each operation to fit its particular profile.  And we can do this with relatively few changes, using technologies that we're already familiar with.

[^1]: Examples of these include using a [rate limiter for Sidekiq](https://github.com/sidekiq/sidekiq/wiki/Ent-Rate-Limiting#concurrent) or running these jobs in a queue with only one worker thread.
[^2]: We could of course complicate things a bit by incorporating the current number enqueued Claim Jobs into our estimate, if this is really necessary.  We might be able to get this number from `Sidekiq.stats` or we might have to track it ourselves.  A quick way to do the latter would be to have a key in Redis for this count and issue `INCRBY` commands from the Distribution Job and `DECRBY` commands from the Claim Jobs.
