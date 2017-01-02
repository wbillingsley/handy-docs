# Ids

Handy includes a few classes for dealing with IDs of things, getting them, caching them etc. They are all fairly simple and unexciting. 

## Id[T, K]

Handy includes a typed Id class: `Id[T, K]`. 

Suppose you are writing an application that uses [Snowflake](https://github.com/twitter/snowflake) for its IDs. You may wish to serialise those IDs always as Strings between the server and the client. But in your code, keeping it as an `Id[Person, String]` has some advantages:

- If you want to serialise them as a different type into the database -- for example, store them there as numbers, you can tell a string from a string-representing-an-ID by its type.
- As the ID is also typed by what it is an ID for (this is a person's ID not a booking ID), the compiler can catch cases of assigning the wrong ID variable.

## Ids[T, K]

Handy also includes a class for a sequence of IDs: `Ids[T, K]`. If you have a bunch of IDs you want to look up, you probably want to do them in batch rather than one at a time. 

Converting between `Seq[Id[T,K]` and `Ids[T, K]` is fairly straightforward:

```scala
import Id._
import Ids._

val seqIds:Seq[Id[Foo, Int]] = Seq(1.asId[Foo], 2.asId[Foo], 3.asId[Foo])
val ids:Ids[Foo, Int] = seqIds.asIds
ids.toSeqId == seqIds
```

## LookUp

A `LookUp` knows how to look up an `Id` or `Ids`. It is a very simple trait:

```scala
trait LookUp[T, -K] {

  def one[KK <: K](r:Id[T,KK]): Ref[T]

  def many[KK <: K](r:Ids[T,KK]): RefMany[T]

}
``` 

## LazyId

LazyId allows a very particular case of indecisiveness -- it lets you be indecisive about whether you want an object or just its ID.

Suppose you wanted to describe a permissions check on whether a user can edit a page. Do you define that function to accept the page itself, or the ID of the page? It might be that when you start writing your app, your check is "actually, any user can edit pages", and later on you need to lock it down. 

With handy, you just define your function as accepting a `Ref[Page]`. A `LazyId` is a pair of the ID and a LookUp. So if the function needs to look up the item, it can. But if it just needs to pull out the ID, it happens immediately without fetching from the database.

```scala
def canEditPage(p:Ref[Page], u:Ref[User]):Ref[Boolean] = {
  // An unusual permission rule, but if you've passed in a LazyId, this won't try to fetch the item
  // but if you've passed in any other kind of Ref[Page] it'll still work
  for {
      pId <- p.refId
  } yield if (pId > 1000) true else false   
}
```

That might sound a bit obscure, but in the permissions system it turns out to be useful because
sometimes you'll have rules like "if they can edit the whole book, we don't need to fetch the page
because they're allowed to edit anything within it".

## refId and GetsId

Suppose we have a `Ref[User]` and want to get its ID. Well, if it is a `RefItself[User]` (we have the user itself), we can get it from the `User` object. And if it's a `LazyId[User, _]` and we haven't fetched the user at all yet, we can just get if from the LazyId. But what if it's a request in-flight -- a `RefFuture[User]` and the future hasn't completed yet? Well, we'll have to get it asynchronously when the `Future` has completed. So, `refId` returns `Ref[K]` because it *might* have to be asynchronous.

What if we have a `Ref[String]` and call `.refId` on it? Well, a String doesn't have an ID, so we'd rather make that a compile error because it doesn't make sense to do that. To tell the difference between things that have IDs and things that don't, `refId` takes an implicit `GetsId` parameter that knows how to get the ID from the object. It has a fairly simple type:

```scala
trait GetsId[-T, K] {
  def getId[TT <: T](obj: TT): Option[Id[TT, K]]

  def canonical[TT <: T](o:Any):Option[Id[TT,K]]
}
```

The `canonical` method is needed because the compiler can't know if `LazyId` uses exactly the same key type that `GetsId` wants to produce. If the compiler is given a `Ref[T]` there's no way for it to be sure that the `K`s in a `LazyId[T, K]` and a `GetsId[T, K]` will always be the same. They just usually are.


If your type implements `HasId[T]`, there will be one already implicitly in scope.

```scala

``` 