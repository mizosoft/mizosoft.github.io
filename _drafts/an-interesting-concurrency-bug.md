---
layout: post
title:  "An Interesting Concurrency Bug"
---

![Are you going to sleep?](/assets/are-you-going-to-sleep.png)

Concurrency bugs, out of all, are perhaps the most insideous[^1]. They're sneaky, intermittent, hard to reproduce, and only like to announce themselves when and where you least expect (usually in production). However, that only makes them all the more interesting! Add to that the fulfilling victory you get (right after the usual headache) when you catch & solve one.

## The Hunt

It all began when our sneaky concurrency bug kindly decided to manifest itself on CI, while I was developing a [redis](https://redis.io/) HTTP cache storage backend for [Methanol](https://mizosoft.github.io/methanol/), through some [`HttpCacheTest`](https://github.com/mizosoft/methanol/blob/master/methanol/src/test/java/com/github/mizosoft/methanol/HttpCacheTest.java) failure. The failed test looked more or less like this:

```java
@StoreParameterizedTest
void cacheGetWithMaxAge(Store store) throws Exception {
  setUpCache(store);
  
  server.enqueue(
      new MockResponse()
        .setBody("Pikachu")
        .setHeader("Cache-Control", "max-age=2"));
  verifyThat(send(serverUri))
      .isCacheMiss()
      .hasBody("Pikachu");

  clock.advance(Duration.ofSeconds(1));
  verifyThat(send(serverUri)) 
    .isCacheHit()
    .hasBody("Pikachu");
}
```

(yup, I use Pokémon names when testing, makes it fun!)

We're testing basic cache behavior here. On first encounter, the cache forwards the request to network, and saves the response as it arrives.
This results in an initial cache miss. Later on, we expected the response to be cached within its allowed time window (here it's 2 seconds).

The above test failed with:

```
HttpCacheTest > cacheGetWithMaxAge(Store) > cacheGetWithMaxAge(Store)[3]: com.github.mizosoft.methanol.internal.cache.DiskStore@24018c8b FAILED
    org.opentest4j.AssertionFailedError: 
    expected: "Pikachu"
    but was: ""
... omitted stacktrace ...
```

Hmm, we're getting an empty response body from cache. Looks like we're either not saving the body properly or truncating it while reading. The cache has been around for more than a year, and I've never came across a similar failure. Instead of lamely staving this off as yet-another-flaky-test, I was determined to find the cause. This kind of bugs, however, are tough because you don't exactly know where to look. Additionally, debuggers won't help much[^2]. However, we can pin down potential cultprits:

- `(Redis|Disk|Memory)Store`: These handle low-level storage of response data.
- [`CacheWritingPublisher`](CacheWritingPublisher): Intercepts body flow from the HTT client and writes it as it's consumed.
- [`CacheReadingPublisher`](CacheReadingPublisher): Reads the response body from cache. 
- [`CacheInterceptor`](CacheInterceptor): Decides, amongst other things, whether to forward request to network or return a cached response if appropriate.

The trick is to know where to look first.
We can overlook store implementations for now because they use reliable means for storage (redis, files & memory buffers). 
The interceptor has little to do with how data is actually written or read. `CacheWritingPublisher` & 
`CacheReadingPublisher` are the main suspects, particulary because they have involved concurrent implementations, so there's a larger surface area for bugs.
The former is less probable to be the cause than the latter, as reading the response body from network succeeds (as you can see from the first part in the test), which makes it likely that it has been fully written. `CacheReadingPublisher` is our main suspect! Let's see how it's structured.

### Reading From Cache

`CacheReadingPublisher` is basically a producer and a consumer bundled together. The producer actively reads data from a store and dumps it into a queue till end of file is reached (with cool-downs if the consumer is slow).
The consumer polls data
from the queue and passes it downstream when requested. As the consumer can be faster than the producer, it can perceive an 
empty queue while data is still being read. That's why there's a `volatile boolean exhausted` that's set to true by the producer when it's done,
so the consumer knows for sure no more data is expected and can go on to complete downstream when the queue is empty.

The producer and consumer are interleaved. There's no syncyhronization
between them aside from the implicit synchronization enforced by the queue (a [*happens-before*](https://en.wikipedia.org/wiki/Happened-before)
between adding an item and polling it) and sharing `exhausted`. 
However, actions in either happen sequentually. 

The relevant consumer code looks(ed) like this:

```java
@Override
protected long emit(Subscriber<? super T> downstream, long emit) {
  ByteBuffer next;
  long submitted = 0;
  while (true) {
    if (queue.isEmpty() && exhausted) {
      cancelOnComplete(downstream);
      return submitted;
    } else if (submitted >= emit || (next = queue.poll()) == null) {
      return submitted; // Exhausted demand or items.
    } else if (submitOnNext(downstream, next)) {
      submitted++;
    } else {
      return 0;
    }
  }
}
```

And this is producer's callback after an asynchronous read:

```java
private void onReadCompletion(
    ByteBuffer buffer, @Nullable Integer read, @Nullable Throwable exception) {
  assert read != null ^ exception != null;

  // The subscription could've been cancelled while this read was in progress.
  if (state == State.DONE) {
    return;
  }

  if (exception != null) {
    state = State.DONE;
    listener.onReadFailure(exception);
    fireOrKeepAliveOnError(exception);
  } else if (read < 0) { // End of file.
    state = State.DONE;
    exhausted = true;
    listener.onReadSuccess();
    fireOrKeepAlive();
  } else {
    readQueue.offer(buffer.flip().asReadOnlyBuffer());
    if (!tryScheduleRead(true) && STATE.compareAndSet(this, State.READING, State.IDLE)) {
      // There might've been missed signals just before CASing to IDLE.
      tryScheduleRead(false);
    }
    fireOrKeepAliveOnNext();
  }
}
```

There's a number of things happening here, but the relevant parts are setting `exhausted = true` and
`readQueue.offer(...)`, 
followed by `fireOrKeepAlive[OnNext]()`. The latter basically ensures the consumer is up and running to see what we've produced.
Now give yourself a moment to find the bug! (hint: the culprit is the consumer).

## The Bug

I like thinking about thread safety issues because most of the time it's like a guessing game. You're basically trying to work out scenarios that break your code. There's however more formal ways of proving correctness, but let's now imagine this scenario:

- The queue is now empty, and there's an ongoing read.
- The consumer gets called and executes the `if (queue.isEmpty() && exhausted)` part.
- `queue.isEmpty()` returns true, and just before testing `exhausted` the consumer thread gets paused[^3].
- The read completes, triggering another one that also completes (reads are triggered one after another upto a certain prefetch limit).
- The first read results in a data buffer being pushed. The second read observes end of file and sets `exhausted` to true.
- The consumer thread gets unpaused, and goes on testing `exhausted`, which has just become true.
- The test succeeds and the consumer completes downstream.
- Boom! A data buffer gets completely ignored.

Because our response in the test was small, this happened at the beginning, so we got an empty response body. Now give yourself a moment to come up with a simple fix! (hint: one hacky fix is to change only one statement in consumer code).

## The Hacky Fix

Well, we can basically switch the testing order from `if (queue.isEmpty() && exhausted)` to `if (exhausted && queue.isEmpty())` (normally, you wouldn't think testing order in an `&&` statement affects correctness, but our single-threaded assumptions are often broken when applied to multi-threaded programs). This works because once `exhausted` is set to true, no more buffers are expected. That is, once `exhausted` is true, we're bloody sure we've exhausted all buffers, and if we see an empty queue henceforth, that'll be its final state, and it doesn't matter when or where the thread gets paused.

Note that our main problem is Java's lack of a closeable queue, which implies 3 states (non-empty, empty, closed) instead of two (non-empty, empty). In Go, we would've simply used a channel, which is basically a closeable `BlockingQueue`[^4].

## A Better Fix

Having correctness rely on testing order is fragile. Interestingly, we only care about closing from one side: the producer[^5]. We can use this fact to come up with a better fix using what's fancily called [sentinel values](https://en.wikipedia.org/wiki/Sentinel_value)[^6
]. A sentinel value is basically a made-up constant that marks a certain event. We can create an empty buffer that indicates end-of-file and add it to the queue when end of file is reached. This modifies producer's code as follows:

```java
private static final ByteBuffer SENTINEL = ByteBuffer.allocate(0);

private void onReadCompletion(
    ByteBuffer buffer, @Nullable Integer read, @Nullable Throwable exception) {
  
  ...

  if (exception != null) {
    ...
  } else if (read < 0) { // End of file.
    state = State.DONE;
    queue.add(SENTINEL);
    listener.onReadSuccess();
    fireOrKeepAlive();
  } else {
    ...
  }
}
```

And the consumer can detect this as follows:

```java
private @Nullable ByteBuffer lastNext;

@Override
protected long emit(Subscriber<? super T> downstream, long emit) {
  var next = lastNext;
  lastNext = null;
  if (next == null) {
    next = queue.poll();
  }

  long submitted = 0;
  while (true) {
    if (next == SENTINEL) {
      cancelOnComplete(downstream);
      return submitted.
    } else if (submitted >= emit || next == null) {
      lastNext = next; // next might be non-null.
      return submitted; // Exhausted demand or items.
    } else if (submitOnNext(downstream, next)) {
      submitted++;
    } else {
      return 0;
    }
  }
}
```

This'll work because seeing `SENTINEL` strictly happens after seeing all the buffers read by the producer (due to the FIFO nature of the queue), so we won't miss anything. Note that we had to store the last polled buffer as, in order to check for completion, we poll before checking demand. So we might end up polling a buffer that hasn't yet been requested[^7].

<!-- 
## A Formal Method

Reasoning about concurrency is hard. That's why it's always good to come up with notation that facilitates this. Let's first define two kinds of objects: an action and an interval. An action is an atomic event that occurs instantaneously[^1] (e.g. $ a = read(val = 1)$), while an interval is basically a sequence of actions that *happen-before* each other (e.g. $ I = [a_1, a_2, ..., a_n] $).

For two actions $ a $ and $ b $, either: 
 - $ a $ *happens-before* $ b $.
 - $ b $ *happens-before* $ a $.
 - $ a $ and $ b $ happen *concurrenctly*.

And an action `a` *happens-before* `b` if:
 - `b` knows that about `a`.
 - `b` is after `a` in program order.


Although nothing in real-life happens instantaneously[^5], the atomicity of an action makes it seem as if it happened instantaneously, because no other action can perceive it midway (it either happened or did not). Therefore, we can assume an action happens instantaneously.  -->

[^1]: Indeed, race conditions can sometimes be deadly, [literally](https://www.bugsnag.com/blog/bug-day-race-condition-therac-25).

[^2]: Concurrency bugs are prime examples of [Heisenbugs](https://de.wikipedia.org/wiki/Heisenbug).

[^3]: When you're writing multi-threaded code, it is a healthy, although headache-inducing, practice to assume that threads can pause (i.e. get context-switched) when you least want them to.

[^4]: Of course, the dawn of loom makes us less shy of mentioning blocking things.

[^5]: Closing from the reader's side can also be helpful as a way of saying to the producer, somewhat crudely, that we're not buying what it's selling anymore.

[^6]: This is actually how I ended up fixing the bug.

[^7]: Recall that the consumer runs sequentially as if it is single-threaded (see [`AbstractSubscription`](AbstractSubscription)), so we don't need to use `volatile` for `lastNext`. 

<!-- [^5]: As per the theory of relativity, an oberver can only perceive an action after the time it takes light to travel from the source of that action to the observer. Therefore, instantaneity only exists theoritically. -->

[CacheWritingPublisher]: https://github.com/mizosoft/methanol/blob/master/methanol/src/main/java/com/github/mizosoft/methanol/internal/cache/CacheWritingPublisher.java
[CacheReadingPublisher]: https://github.com/mizosoft/methanol/blob/master/methanol/src/main/java/com/github/mizosoft/methanol/internal/cache/CacheReadingPublisher.java
[CacheInterceptor]: https://github.com/mizosoft/methanol/blob/master/methanol/src/main/java/com/github/mizosoft/methanol/internal/cache/CacheInterceptor.java
[AbstractSubscription]: https://github.com/mizosoft/methanol/blob/master/methanol/src/main/java/com/github/mizosoft/methanol/internal/flow/AbstractSubscription.java