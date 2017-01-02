# Handy

Handy is a small library of Scala classes to make asynchronous, lazy, reactive programming concise and easy for the web -- both in the browser (Scala.js) and on the server (Scala on the JVM). Rather than
make everything reactive (like other frameworks that use the observer pattern throughout), it tries to make reactive programming *lazy* -- only be reactive when it needs to be, and otherwise just look like
function calls.

It also includes classes I find commonly useful for implementing web systems, for example typed IDs, and a simple asynchonrous permissions system. This makes it possible, for instance, to write
permission logic that can be run transparently in the browser or on the server.

Programs using handy usually make heavy use of Scala's for-comprehension style. Suppose we want to organise a surprise birthday party for a bot called Algernon, from the bots it normally interacts with...

```scala
val invitees:RefMany[Bot] = for {
  canGetFriends <- approval ask CanLookupFriends(Algernon)
  friend <- socialNetwork.friends(Algernon).take(10)
  canMessage <- approval ask CanSendNotificationTo(friend)
  sent <- invite(friend) if (sent.successful)
} yield friend
```

And we'll get an asynchronous list of those friends from Algernon's top 10 best friends who we were allowed to send an invite notification to. 

Note that this for-comprehension translates to a sequence of flatMap and map calls, and it can be asynchronous. For example, looking up the friends is probably an asynchonrous call. As, often, are the
permissions checks.

Its only dependency is on the interface of the Reactive-Streams standard, so that it can produce and consume reactive streams.

## What's the philosophy behind all this

Really, the library was written from a philosophy of "maximum indecisiveness". 

I'd written a few apps that I'd needed to migrate between web frameworks, and between styles of database (SQL or NoSQL). And it irked me how much of the model of an app depended on the framework it used. For 
example, who can do what is a business decision -- that should be part of the model code, not an annotation on an HTTP controller.

So I tried to write a little library that makes as few assumptions as possible, and lets decisions happen as late as possible. It is designed for programs that run on the Web -- either in the browser 
or on the server, so you can delay deciding where something will be calculated too. So it was designed through thinking fairly abstractly and unopinionatedly about programming for networked environments.

### Network programming in general

When writing servers that need to talk to services and databases, or clients that need to talk to servers, we tend to load data only when we need it, and when we go to fetch it:

- it might take some time (asynchronous)
- it might fail
- it might not be there

You could describe that as a `Future[Option[T]]`, but actually that'd be a bit painful because it's a nested type. Calling `map` would let you work with the Option inside the Future, but then you'd need to call `map` again to 
work with the contained value. I'll let your imagination do the rest, but `Future[Option[T]]` does not compose well if you're trying to write an algorithm more than a few steps long.

Instead let's call our hypothetical type `Ref[T]`, and say it has those properties. It *might* be asynchronous. It *might* fail. It *might* produce nothing.

Instead of backing this type with a `Future[Option[T]]`, or some other specific structure, handy makes `Ref[T]` a trait.
So that for instance we can have `RefItself[T]` if actually it turns out you've already got the object in your hands.
The principle here is about being lazy about converting anything. If you are writing an HTTP controller, then at the end of your algorithm, you will probably need to give your web framework a type it understands -- probably something like `Future[Result]`.
But until the end, each step in your algorithm can just talk about a `Ref[This]` and `Ref[That]`.

In the case of a single value, that might seem a bit trivial, but this also lets us be very composable. Especially when we bring in `RefMany[T]` -- a reference to many things.
A `RefMany[T]` could be an array of things. It could be a reactive stream. It could be an array of `Ref[T]`s. It could be an array-of-refs-of-arrays-of-reactive-streams-of-futures. 
Until we spit it out at the end, our algorithm doesn't have to care -- it's just a possibly-asynchronous reference to many things of type `T`. At the end, we'll usually decide to output the result as a 
reactive stream. But we can be lazy about that and only convert to a reactive stream when we need to -- along the way we can let the different kinds of `RefMany[T]` (those backed by arrays, or streams, 
or arrays of `Ref[T]`s, etc) compose naturally.

### The browser environment

In the browser, we don't usually want all the information about our app, just enough to display the UI and take actions as needed. Information gets loaded on demand. JavaScript is single-threaded, but calls
to the server can involve significant latency.

So, we have an environment where we want to be:

- lazy 
- asynchronous
- reactive (cache some information, but react to changes)

But there's not actually going to be a high degree of concurrency (JavaScript is a mostly single-threaded environment). 

### The server environment

In the server, we're usually responding to a request made by the client. Things happen asynchronously, because we may need to talk to databases and other systems. And we may need to fetch values along the way,
that might be used in several steps of the processing.

So, we have an environment where we want to be:

- lazy 
- asynchronous
- reactive (because sometimes we have a websocket / persistent connection and want to stream out events)

But we tend not to share variables between requests (at least not mutable ones), so the individual variables within a request won't see a high degree of concurrency. The server itself will -- lots of requests
going on at once, but they won't be sharing many variables.

The upshot of which is it's not too much of a stretch to make the same abstractions work in both environments.