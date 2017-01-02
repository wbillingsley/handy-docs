# Latch

A `Latch` is a little wrapper around a Scala `Future` that makes it clearable. Think of it as a single item 
asynchronous cache. Latches can be dependent on one another, in what I call a *lazy observer* pattern. 

Suppose we start with a variable for who my closest friend is:

```scala
val myClosestFriend = Latch.immediate(Algernon)
```

And we have an asynchronous calculation that is dependent on it -- in this case to see if my closest friend interacts most with me on a social network.

```scala
val closestFriendReciprocates:Latch[Boolean] = for { 
    friend <- myClosestFriend 
    hisClosest <- socialNetwork.getMostInteractedWithUser(friend)
} yield (hisClosest == me)
```

We now have two latches that depend on each other. Suppose my closest friend changes for some reason. 

```scala
myClosestFriend.fill(Bertie)
```

The variable on whether my closest friend reciprocates is now invalid. In an observer pattern, we might see that the closest friend has updated, and 
re-calculate the dependent value (make the call out to the social network). But actually, we might not need to use the result yet -- it might be something we were interested in a while ago
but that we've stopped being interested in. 

So instead, what happens is the dependent latch will just clear itself and de-register its listener. And then if you ask it for its value again, it'll quickly make the call and re-register its listener.

This creates a *lazy observer* pattern: invalidations happen by the observer pattern, but calculations are always lazy (triggered by the next request for the value). And the listeners are only in-place if
there is a cached value that might need invalidating, so a cleared latch that goes out of scope can be garbage collected.


### flatMap takes a Future

`Latch.flatMap` takes a function that returns a `Future`, not a `Latch`. That is, it's defined as:

```scala
def flatMap[B](f: T => Future[B]):Latch[B]
```

This might seem odd to those that think of "flatMap" as being like "bind" in a monad. However, it makes a for comprehension using a Latch work in a sensible way: 

```scala 
val result:Latch[C] = for { 
  a <- latch
  b <- doSomethingWith(a)
  c <- doSomethingElseWith(b)
} yield c
```

`result` (the result of the computation) is a latch. Whereas `doSomethingWith()` are just asynchronous functions returning `Future[B]` and `Future[C]`.

If the original latch is cleared, `result` will be invalidated. The intermediate steps (`b` and `c`) will be recomputed when `result` is next asked for its value, because that's the way the computation is written, but they don't themselves need to be inside latches.

### If you don't want the result to be a latch

In the example above, `result` was a Latch. If you'd just like it to be a Future, just call `latch.request` at the beginning of the for comprehension, so you're flatMapping over the Future instead of
over the Latch itself.

```scala
def result:Future[C] = for { 
  a <- latch.request
  b <- doSomethingWith(a)
  c <- doSomethingElseWith(b)
} yield c
```


## Use in Scala.js with React.js

The place I find Latches particularly useful is when working with React.js in Scala.js programs. React regenerates the UI whenever any state in the program changes. This creates a very nice kind of UI-driven laziness:

In the model for our client-side program, we can keep some set of Latches that can cache variables of interest so we don't have to keep requesting them from the server. When any cached values are invalidated (updated or cleared), any dependent latches will be cleared automatically too. And if those variables are still on-screen (needed by the UI), then the UI rerender will trigger them to be recomputed.  

To help make this even easier, you can also register a listener that will be called when *any* Latch updates. Usually this is so that in React you can just say

```scala
Latch.addGlobalListener(_ => rerender())
```


