# Approval

Handy includes a simple but surprisingly useful permissions system.

Imagine your request is a small schoolchild. And it needs to collect permission slips for what it wants to do. Sometimes, the teachers might look at the permission slips the child has already got, and say "well, if you're allowed to play soccer, of course you're allowed to get a football".

`Approval[U]` works like a little wallet for permission slips. `U` is your user type, and it contains a Ref to its owner:

```scala
case class Approval[U](val who: Ref[U]) { 
    //...
}
```

`Perm` is a permission. It is a trait with a single method:

```scala
trait Perm[U] {
  def resolve(prior: Approval[U]):Ref[Approved]
}
```

`Approved` is a type that represents a permission has been approved, and `Refused` is a Throwable to represent that something has not been approved. So:

```scala
for { 
    a <- approval ask SomePermission 
} yield doSomething()
```

`doSomething()` will only occur if the permission is approved, and otherwise the result will be `RefFailed(Refused(message))`.

## prior

The `resolve` method takes a parameter called `prior: Approval[U]` so that the method (the permissions check) can use what permissions you've already been granted when making its determination.

```scala
def resolve(prior: Perm[User]) = {
  // If the user is an administrator, we don't need to check any further
  (prior ask CanAdminister) recoverWith { case Refused(_) => 
    // ... ok, they're not an admin so we ought to check...
  } 
}
```

And because we usually test permissions with

```scala
approval ask permission
```

and approval has a cache of permissions already granted, it will automatically check if it's already been granted a permission. The permissions wallet uses a ConcurrentHashMap, so entries can be invalidated. 

If you want to check a permission without caching (for example, for time-sensitive checks on the client), there's two ways of doing that which are equivalent:

```scala
permission.resolve(approval)

approval askAfresh permission
```

Usually on the server, the lifetime of an Approval (a permissions wallet) is short -- a single request, while on the client it may be longer (while the user is logged in).

## Approvals and asynchronicity

The return type of `resolve` is `Ref[Approved]`. So it *might* be asynchronous. This means your permissions check can also go and look things up. "You may only edit your assignment submission if it is *your* assignment submission and the due date for that assignment has not passed". 

```scala
def resolve(prior: Perm[User]) = for {
    user       <- prior.who
    s          <- submission if s.submittedBy == user.id
                  orIfNone Refused("This isn't your assignment")
    assignment <- s.assignmentId.lookUp(lu) if assignment.due > getTime() 
                  orIfNone Refused("Assignment is past due")
} yield Approved("Yes you can edit it")
```

## Permission equality

Usually, we want a permission's equality to be based around the ID of an object, not the object itself. 

Consider a request to the server to edit a user's profile.
- The ID will remain the same
- Some fields of the object will change

So in a request to edit a record it is usually the case that `submittedRecord.id == originalRecord.id` but `submittedRecord != originalRecord`. (Assuming you've written your business objects as case classes.) But we don't want "permission to edit the submitted record" and "permission to edit the original record" to be different items because conceptually they are permission to edit *that* user's profile.

## Perm.onId

It is entirely reasonable for a permission to be a case class on an ID. For example:

```scala
case class CanEditCourse(id:Id[Course, String]) extends Perm[User] {
  def resolve(p:Prior[User]) = ???
}
```

But you might want it to work so that you can also pass in the `Course` itself if you have it, so that `resolve` doesn't try to look it up from the database again. So, perhaps you would like your permission to do its equality-check based on the ID of an item, but be happy receiving any kind a `Ref`, whether it's a `LazyId` or a `RefItself` or a `RefFuture`.

Handy includes a little generator method `Perm.onId[U, T, K]` that is useful for these cases. `U` is your user type, `T` is the item type, and `K` is the key type. It requires an implicit `GetsId[T, K]`, because it needs to know how to extract the ID from the item.

```scala
  val EditTask = Perm.onId[User, Task, String] { case (prior, refTask) =>
    for (
        t <- refTask;
        a <- prior ask EditCourse(t.course)
    ) yield a
  }
```

And then `EditTask(getUser(3)) == EditTask(user3.itself)`


