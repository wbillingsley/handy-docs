# RefMany

`RefMany` is a reference to many of something. As with `Ref`, it tries to unify across a few different common types

`RefMany` might be asynchronous. They might fail to start. They might produce an error as they stream
out elements. Eventually, RefMany is equivalent to a reactive stream. And again, there is a method to stream out the data as a 
reactive stream. But any conversion to a reactive stream happens as late as possible. So, for instance,
if you were just to use `Seq`s, it would all happen synchronously on the same thread:

```scala
val s1 = Seq(1, 2, 3).toRefMany
val s2 = Seq(4, 5, 6).toRefMany

(for {
  a <- s1
  b <- s2
} yield a & b).fold(_ + _) // Ref[Int], but computed synchronously
```

Introducing some asynchronous functions, it all still produces `RefMany` and `Ref` but working 
asynchronously where needed

```scala
val s1 = Seq(100, 200, 300).toRefMany
val s2 = Seq(400, 500, 600).toRefMany

def getNthPrime(i:Int):Future[Int] = ???

(for {
  a <- s1
  athPrime <- getNthPrime(a)
  b <- s2
  bthPrime <- getNthPrime(b)
} yield a & b).fold(_ + _) // Ref[Int], computed asynchronously
```

There are some calls that can cause your RefMany to be connected to a reactive stream early. For 
example, if you have a `RefTraversableRefMany`, ie a sequence of possibly-asynchronous sequences,
and call `take(5)` the implementation will currently use a reactive stream to achieve this. 

## Composing with Ref

`Ref` and `RefMany` compose together using Scala's for comprehensions

```scala
for {
  foo <- somethingReturningRefFoo()
  bar <- somethingReturningRefManyBar(foo)
  baz <- somethingReturningRefBaz(bar)
} yield baz // RefMany[Baz]
```





