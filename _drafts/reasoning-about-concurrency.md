---
layout: post
title:  "Reasoning about Concurrency"
---

<!-- [^5]: As per the theory of relativity, an oberver can only perceive an action after the time it takes light to travel from the source of that action to the observer. Therefore, instantaneity only exists theoritically. -->

## Formallity is Generality

Reasoning about concurrency is hard. That's why it's always good to come up with notation that facilitates this. Let's first define two kinds of objects: an action and an interval. An action $ a $ is an atomic event that occurs instantaneously[^1], while an interval is basically a sequence of actions, where each action *happens-before* the next one (e.g. $ I = [a_1, a_2, ..., a_n] $). How does this translate to code? Well, actions basically translate to reads & writes (e.g. $ a = read(val = 1) $ indicates action $ a $ is a read of variable `val` as `1`). Intervals then, can be a sequence of reads & writes, however, we can regard intervals as function calls (e.g. $ I = q.poll() = 1 $ indicates calling `q.poll()` and returning `1` as a result), where the function is recursively "flattened" into a sequence of reads & writes.

For intervals, we can basically assume they're method calls (e.g. $ I = q.poll() = 1 $ indicates calling `q.poll()` and returning `1` as a result). This makes sense as a method call can be recursively "flattened" into a sequence of atomic actions, indicating an interval.

For two actions $ a $ and $ b $, either:
 - $ a $ *happens-before* $ b $.
 - $ b $ *happens-before* $ a $.

If $ a $ *happens-before* $ b $, we write: $ a \rightarrow  b $. Note that this relation is transitive (e.g. $ a \rightarrow  b $ and $ b \rightarrow  c $ implies $ a \rightarrow  c $).

For an action $ a $ and an interval $ I $, either: 
 - $ a $ *happens-before* $ b $.
 - $ b $ *happens-before* $ a $.
 - $ a $ is *concurrent* with $ I $.



And an action $ a $ *happens-before* $ b $ if either:
 - $ b $ knows that about $ a $ (e.g. $ b $ is a read that's read what $ a $ has written).
 - $ b $ is after $ a $ in program order.

Well, then, what do we do with all of this? Let's see. 

Let's define the expected chain of actions for the buggy consumer code to complete downstream. From the code: 

```java
...
while (true) {
  if (queue.isEmpty() && exhausted) {
    cancelOnComplete(downstream);
    return submitted;
  } ...
}
```

We have: 

  $ queue.isEmpty() = true \rightarrow read(exhausted = true) \rightarrow cancelOnComplete(downstream) $

As for the producer, checking the code: 

```java
private void onReadCompletion(
    ByteBuffer buffer, @Nullable Integer read, @Nullable Throwable exception) {

  ...

  if (exception != null) {
    ...
  } else if (read < 0) { // End of file.
    state = State.DONE;
    exhausted = true;
    listener.onReadSuccess();
    fireOrKeepAlive();
  } else {
    ...
  }
}
```

The entire relevant producer workflow can be translated into:

$ *queue.offer(buffer) \rightarrow write(exhausted = true) $

Which basically means the producer offers zero or more buffers into the queue, then sets `exhausted` to true (when end of file is reached). 

As a volatile read *happens-before* the volatile write resulting in the read value, merging the two pipelines on `exhausted` gives us: 

$$
\begin{aligned}


 
\end{aligned}
$$

Although nothing in real-life happens instantaneously[^5], the atomicity of an action makes it seem as if it happened instantaneously, because no other action can perceive it midway (it either happened or did not). Therefore, we can assume an action happens instantaneously. 

