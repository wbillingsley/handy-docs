# Ref

`Ref` is a "reference to something". It is a monad that allows us to write code like this: 

```scala
for {
  page <- LazyId(pId).of[Page]
  approved <- request.approval ask editPage(page.itself)
  updated <- update(page, json)
  saved <- pageDAO.save(updated)
} yield saved
```

...and not worry about which of those lines are producing `Option`, `Future`, `Try`, or `LazyId`.

Eventually, it's equivalent to a `Future[Option]` (and indeed has a `.toFutOpt` method) but a little more concise, 
and with a couple of extra features.

Rather than promote every type to `Future[Option]`, it defines `Ref` wrappers for each of the subtypes and composes 
together a structure. So, for example, if your chain of calls only involved `Option`s and `Try`s, your code will
execute synchronously on the same thread.

## Composing with RefMany

`Ref` and [`RefMany`](RefMany) can be combined in for-comprehensions, and it'll get the plurality of the result right.

```scala
for { a <- refA; b <- refB } yield b           // Ref[B]
for { a <- refA; b <- refManyB } yield b       // RefMany[B]
for { a <- refManyA; b <- refB } yield b       // RefMany[B]
for { a <- refManyA; b <- refManyB } yield b   // RefMany[B]
```


