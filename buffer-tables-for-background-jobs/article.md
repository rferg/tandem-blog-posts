# Buffer Tables for Background Jobs

Here is a pattern I've seen several times, especially in applications that have a data-intensive integration with a third-party service.

Suppose we have the following method for recording that a sales `Lead` has been contacted.  We save this `ContactActivity` in our database, but we also use a third-party marketing service that needs to be apprised of this fact.  We accomplish this by enqueuing a background job that sends this update via REST API to that service. Something like this:

```ruby
class Lead < ApplicationRecord
  has_many :contact_activities

  def record_activity!(type:, comment:)
    ContactActivity.new(type:, comment:).tap do |activity|
      contact_activities << activity
      save!
      RecordMarketingServiceActivityJob.perform_async(activity.id)
    end
  end
end
```

In general, it is a good idea to scope background jobs to small, single transactions.  Reasons include:

- It is easier to make them idempotent.
- Single job failures affect at most a single transaction.
- They are easier to reason about.
- They will generally complete quicker.  Long-running jobs are problematic.  They can delay deployments or risk being interrupted, potentially leaving data in an inconsistent state.

But what happens when we scale up our sales operation and start recording, say, 10s or 100s of thousands of contact activities per day during business hours?  We'll be enqueuing 100s of thousands of background jobs, each sending an API request to that marketing service.  We'll probably start hitting rate or concurrent connection limits for that API; job queues will back up; data sync latency will increase; marketers will complain.

Fortunately, the marketing service's API includes endpoints for bulk operations.  Instead of sending a single update per request, we can send, say, 1,000 per request.  This would cut down our job and request volume by a few orders of magnitude and make it easier to avoid rate limiting.

We don't, however, want to restructure all of our code and APIs for recording contact activities to accommodate this background process--assuming that this part of the code is handling the load just fine.  We'd prefer if we can keep our existing single-record-level data operations, but take advantage of bulk operations for syncing with the marketing service.

In this article, I'll describe one strategy for doing this.  It has some constraints, which I'll mention up front.  First, we are not processing events in realtime.  We assume that latencies on the order of minutes are acceptable.  Second, we are handling a moderate to small amount of events (less than 1,000s per second).  Third, we are going to keep it [boring](https://boringtechnology.club/) and use technologies that you are probably already using.  In particular, we'll assume we're using Postgres and Sidekiq as our job processor (using Redis).  But you could probably apply the same strategy using your preferred RDBMS and background job runner of choice.

## Buffer table

The basic idea is to have a database table dedicated to storing events that we want to process in batch background jobs.  At the point in our code where we would enqueue a single-record job, we instead insert into this table.  We will then have a scheduled job that polls this table every few minutes (or whatever time interval makes sense for our use-case) and dispatches batches of records to jobs to perform the actual processing.

The changes to our example method above will be small.  Instead of enqueuing a `RecordMarketingServiceActivityJob` with the newly created `ContactActivity`'s ID, we create a `ContactActivityEvent` associated to the `ContactActivity`.  The table behind this model `contact_activity_events` will be our "buffer table" dedicated to batch processing these and sending updates to our marketing service.

```ruby
class Lead < ApplicationRecord
  has_many :contact_activities

  def record_activity!(type:, comment:)
    transaction do
      ContactActivity.new(type:, comment:).tap do |activity|
        activity.contact_activity_events << ContactActivityEvent.new
        contact_activities << activity
        save!
      end
    end
  end
end
```

By using a database table, we get a few of benefits for free:

- *Transactions*: Enqueuing an event for background processing can now participate in a transaction with the main operation.  We can guarantee that, for example, either the contact activity was successfully created *and* an associated event was enqueued or neither operation completed.
- *Data integrity*: We can enforce familiar data integrity checks on our events.  We should have a foreign key constraint on our buffer table referencing our contact activities table to ensure that we don't have any orphaned events.  Moreover, perhaps, we only want to process a single event per contact activity record at a time.  We could add a unique constraint to the `contact_activity_id` column to enforce that.
- *Familiarity*: We can use the same tools (e.g., ActiveRecord) and patterns that we are already using to interact with the database.

One downside, however, especially when compared to an alternative implementation in, say, Redis, is that we could quickly run into performance issues with a naive implementation and a relatively high volume of events.  Assuming our buffer table is in our application database, performing updates and queries on this table could end up consuming database resources and slowing down other areas of our application.  Let's consider some strategies for avoiding this.

### Performance heuristics

The main performance heuristic is to keep the buffer table small and single-purpose.  Its purpose is to store and track events for batch processing.  Once a job has finished with an event, it should be deleted.  If you need to keep these events in your database for logging or reporting, copy them to another table (an "archive" or "completed" table) before deleting them in the same transaction.

We will be continuously querying this table every few minutes or so.  Controlling the table size will make it easier to keep these queries fast.  But, this workload will also be relatively write-heavy.  We will be inserting each event one at a time; these will be updated when they are assigned to a batch; and then they will be deleted.  Accordingly, we should not add indexes to the table that do not directly support these operations, since they would unnecessarily slow down writes.  Again, if you need to query these events for other purposes, use a separate table.

Moreover, after the initial event creation, we should avoid individual updates and deletes.  If our batch size is 1,000, we should update/delete a batch in a single database command instead of issuing 1,000 separate update/delete commands.  This will probably be faster since it requires fewer network round-trips.  And it reduces overall load on the database.

Finally, if possible, we should take advantage of our background job processor and run many batch jobs concurrently.  We can process more events in less time, i.e., increase throughput.  This also reduces the likelihood of the buffer table growing to an unwieldy size during periods of high volume, since we are processing and deleting them at a faster rate.  (Although, note that we will still need to stay within the rate limiting confines of the marketing service API.)

### Schema

Given the above, the schema for the buffer table will be simple.  We will have the typical boilerplate (primary key, timestamp columns) and whatever application-specific columns we need.  In the present case, this would be a foreign key to `contact_activities`, which you will presumably want an index on.

The only batch processing specific column will be a nullable `uuid` column called `group_id`, which identifies the batch a record is a member of.  We will put a `BTREE` index on this column, since this is what we will mainly be querying on this table after batches have been assigned.

To see how this schema facilitates following the performance heuristics above, let's look at the background jobs that will actually run the batch processing.

## Jobs

TODO

## Failures and retries

TODO

## Code

TODO
