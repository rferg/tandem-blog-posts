# Against Randomness in Unit Tests

TODO


## Arguments for randomness and why they fail

Again, at first glance, introducing sources of indeterminism into your tests seems like a bad idea.
There must be some benefits that apparently outweigh the costs.
I think that there's a few thoughts here:

- __Brevity__: The input has N possible values, where N is smallish.  Instead of writing N tests to cover each case, choose the input randomly.  The input space will effectively be covered, since tests are run many times.
- __Sampling__: The input can have N possible values, where N is large or infinite.  I can't write N tests, but randomly sampling from the space of inputs is better than picking a few values, which may be biased and miss certain parts of the input space.
- __Surfacing__: This object's field F has N possible values.  This object is used in many places, often buried one or more references away from the main operation.  It's possible that a regression could be introduced for some subset of the values of F and this would go undetected by a test writer, who will be ignorant of or easily miss all of the possible values.  Pick the value of F randomly in shared test set up so that they will be forced to confront all N cases at least some of the time.

I think that all of these justifications fail to overcome the tradeoffs for randomness described above.
All things considered, the simpler, deterministic implementation is probably better.
Part of the reason that these justifications might seem more compelling than they should is that they tacitly overestimate the probability that bugs will be detected with these random tests, while underestimating the costs.

Proponents of random tests often miss that decisions about whether "the tests pass" for a given PR or changeset are based on a small number of test runs,
including local test runs on a developer's machine or test runs in CI.
The effectiveness of tests for detecting bugs should primarily be evaluated in this context, since this is the gate to being deployed into production.
The probability that a test fails at least once in the totality of test runs over time is less relevant.
Even if this probability is 1, the bug could have been deployed to production long before the relevant random test scenario was chosen to fail the test.

Let's consider a contrived scenario to understand what the relevant probabilities might be.
Suppose that we have a situation much like that imagined in the __Brevity__ scenario.
We're testing a method that can take in four possible inputs.
Instead of writing a test for each case, we randomly select one in our test.

```ruby
  it "is true" do
    status = %i[a b c d].sample
    expect(object.acceptable?(status)).to be(true)
  end
```

Suppose that a bug was introduced where `acceptable?` returns `false` only when it's given `:d`, failing the test.  Assuming nothing else is going on, the probability that a single run of this test will pass is 3/4 or 0.75 and the probability of failure is 1/4 or 0.25.  1/4 seems like really bad odds for detecting the bug with this test, but that's only a single run.

Before moving on to the multi-run scenario, note that running a test a single time before approving a PR is quite common, especially on larger projects.
If this test is not obviously related to the PR at hand and the test suite is large and/or slow, developers may never run it locally on their machines.
It might only run in CI and CI pipelines are almost always configured to only require a single successful run.

I think it's important to emphasize this point.
CI pipelines generally only require one successful run to "approve" the PR.  Any additional runs are optional.  Tests that rely on multiple runs to cover all cases or reach some reasonable level of reliability are therefore unreliable by the lights of the explicit CI policies that almost every project uses.

Nevertheless, let's consider a multi-run case.  The probability of getting some number of boolean outcomes (the test passes) in some number of independent trials (test runs) follows the [binomial distribution](https://en.wikipedia.org/wiki/Binomial_distribution).  The probability that you get *k* successes in *n* trials when the probability of success is *p* is $p^k$ times $1 - p ^ {n-k}$ times the binomial coefficient *(n choose k)*.

$$
  {n\choose k}p^k(1-p)^{n-k}
$$

The binomial coefficient can be calculated like this:

$$
  {n\choose k} = {{n!} \over {k! (n-k)!}}
$$

The bad case for us is when the test passes every time, meaning the assumed bug did not cause a test failure and would go undetected.
Say we run the test 5 times, which seems realistic to me in the course of local and CI test runs.
So $k=n=5$.  This makes the calculation easy since it simplifies both the binomial coefficient and the term $(1 - p) ^ {n - k}$ to $1$.  Thus, the probability of 5 out of 5 passes is $p ^ k$ or $0.75 ^ 5 \approx 0.24$.
To me at least, this is a pretty high chance of missing that bug.
Even if we double it to 10 test runs, we still have a $0.75^{10} \approx 0.06$ chance of all of the tests passing.

These probabilities, I think, are unacceptable especially given the alternative implementation of writing a test for each case.  Something simple like:

```ruby
it "is true when given :a"
  expect(object.acceptable?(:a)).to be(true)
end

it "is true when given :b"
  expect(object.acceptable?(:b)).to be(true)
end

it "is true when given :c"
  expect(object.acceptable?(:c)).to be(true)
end

it "is true when given :d"
  expect(object.acceptable?(:d)).to be(true)
end
```

It's a few more lines, but other than that, it seems superior in every way that matters.
In my opinion, these tests are much easier to read.
When one of them fails, we'll know exactly which input caused the failure (without having to find the seed and rerun the tests).
And, to the point, the `:d` test will *always* fail when there's a bug such that `acceptable?` returns `false` in that case.
So, why would we choose an alternative that randomly misses bugs around 1 out of 4 or even 1 out of 20 times in a multi-run scenario to save a few lines of code?

The proponent of __Brevity__ will say, "what if we have 10 or 20 possible inputs?  Are you really going to write a test for each of those?"

In all honesty, I probably would hand write 20 such tests and abstract any test setup into helper methods or classes (if it's that complicated).  But not every one has such a high tolerance for repetitive test-writing.  The other obvious deterministic solution is to construct the tests by looping over all of the cases.  Something like this:

```ruby
all_of_the_inputs.each do |input|
  it "is true when given #{input}" do
    expect(object.acceptable?(input)).to be(true)
  end
end
```

That's the first point in response: there are perfectly fine deterministic solutions for such cases.
The second point is that test effectiveness gets worse as we increase the number of possible inputs, assuming the bug lies with one particular input (or a small subset of them).
For example, if we have 20 cases and there's a bug for one of them, the probability of the test passing is 0.95 instead of 0.75.
Thus, if we have 5 runs, the probability of the test passing in all of them is $0.95^5 \approx 0.77$.
At 10 runs, it's $0.95^{10} \approx 0.6$.  In other words, as the random tests prescribed by __Brevity__ supposedly get more attractive because they offer smaller or fewer tests than the alternative, they get less effective at actually detecting bugs or regressions.

### Flakiness and peeking

I think, however, it gets even worse for the randomist, on two counts.

First, the probabilities mentioned above of missing a bug due to the tests passing are, practically speaking, a lower bound.
In reality, there's no guarantee that if the test *does* happen to fail that the developer or reviewer will attribute this failure to a potential bug and investigate.
This is especially true if the test fails infrequently and the test suite has a reputation for flakiness, which is more likely if it contains a lot of randomness.
The developer will probably rerun the tests, dismiss the previous failure as a "flake", and move on with their work.

Pulling numbers out of thin air to make the math easy, let's suppose that if the tests pass 90% of the time or more, any failures will be dismissed as flakes.
If we have 10 test runs, we need to calculate the cumulative probability of getting 9 or 10 passes.  Since the outcomes are discrete, we can calculate this by simply summing up the probability at each $k$ up through $n$, that is:

$$
\sum_{i=k}^{n} {n\choose i}p^i(1-p)^{n-i}
$$

For our 4 possible inputs case (i.e., $p=0.75$) we already calculated the probability when $k = n = 10$, viz. $0.75 ^ {10}$.  To account for flakiness dismissal, we just need to add this to the probability at $k=9$.
The binomial coefficient  ${10 \choose 9} = 10! / 9!(10 - 9)! = 10$.
Thus in total, we have $(10 \times 0.75^9 \times 0.25^{10-9}) + 0.75^{10} \approx 0.24$, which if you remember is quite a bit higher than the $0.06$ we got before in the 10 run case.
Of course, that "flakiness threshold" of 90% is completely made up and probably depends on a bunch of variables.
But I think it's pretty uncontroversial to say that that value is less than 100% for almost everyone and so has some negative impact on the effectiveness of random tests specifically.

Second, we've been assuming a fixed number of test runs.
However, developers are much more likely to stop running the tests (locally or in CI) on a successful run than a failure.
They don't say ahead of time, "I'm going to run the test suite 10 times and if it passes 9 or more times, then we'll call that a pass".
If the tests happen to pass on the first run, they are unlikely to attempt a second run.
If the tests fail on the first run, and there's a reputation of flakiness, they are more likely to run the tests a few more times to see if the failure persists.

This is a blatant example of a flawed experimental practice called 'peeking' or 'unaccounted optional stopping'.
Experimenters will collect some data and look at the calculated *p-value* (i.e., false positive rate or rate of incorrectly rejecting the null hypothesis).
They will then decide to stop the experiment or collect more data depending on whether the *p-value* has reached the desired threshold for statistical significance--typically 0.05.
In simulations and observed cases, peeking, possibly combined with other bad techniques, has been shown to yield false positive rates 5-10 times greater than the nominal *p-value*[^1].  Therefore, we can assume that the nominal probabilities we've calculated for tests incorrectly passing in these contrived examples are almost certainly much lower than the real rates when peeking is employed as a common practice.

### When N is large

When there's a relatively small number of cases to test, the probabilities for random tests spuriously passing are relatively high.
Since simple, nonrandom alternatives are easy to come by, they should be preferred.
But, what about when there is a large or infinite number of cases, as the __Sampling__ argument points out?

Obviously, all of the points in the last section still apply.
The relevant term $p$, the probability that the test passes, is a ratio; the size of the denominator in itself is strictly irrelevant for calculating probabilities.
However, bugs aren't guaranteed to spread evenly across the input space as it increases in size.
And with a larger input space, there are more places for bugs to hide, so to speak.
If there are only 5 possible inputs, the smallest probability $(1-p)$ that the random test will fail besides 0 is 1/5.
If the possible input is any 32-bit integer, then the smallest $1-p$ besides 0 is $1/2^{32}$ (think, for example, an integer overflow bug).
The probability is so low that we will actually hit that failure case, even when running the test many times, it's as if we're not testing it at all.
Given that introducing randomness into tests carries with it significant downsides, adding it to "cover" such cases seems unwarranted.

TODO:
- Deterministic alternatives and throwing away information

### Surfacing?



[^1]: See [Simmons et al. 2011](https://journals.sagepub.com/doi/pdf/10.1177/0956797611417632), [Johari et al. 2017](http://library.usc.edu.ph/ACM/KKD%202017/pdfs/p1517.pdf)
