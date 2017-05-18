# Resolving Type-Constructor using dependent typing

Imagine we want to construct a method whose return type needs to be flexible. In particular, we would want
to derive a one-argument type-constructor implicitly based on some input-type and then supply it with some type. 
Let's make an attempt at encoding this:

```tut:silent
trait TCResolver[A] {
  type TC[_]
  def wrap[T](t: T): TC[T]
}
object TCResolver {
  type Aux[A, TC_[_]] = TCResolver[A]{type TC[T] = TC_[T]}
}
```

This is a straightforward extension from the Aux pattern applied in other libraries such as shapeless and Scalaz.
Instead of a concrete type, it resolves a type-constructor, and provides a method to wrap a given `T` instance inside
of that type constructor.

Here's an example implementation relying on `TCResolver` for its return type:

```tut:silent
def resolveTest[T, TC_[_]](t: T)(implicit resolver: TCResolver.Aux[T, TC_]): TC_[T] = resolver.wrap(t)
```

Let's provide a dummy implicit instance that allows an `Int` type-argument to imply an implicit `List` type-constructor 

```tut:silent
implicit val intTCResolver: TCResolver.Aux[Int, List] = new TCResolver[Int] {
  override type TC[T] = List[T]
  override def wrap[T](t: T): List[T] = List(t)
}
```

and show that it works, if we supply both the type arguments and implicit arguments explicitly

```tut
resolveTest[Int, List](1)(intTCResolver)
```

it also works just fine when _only_ supplying the type arguments explicitly, so it locates the implicits correctly
```tut
resolveTest[Int, List](1)
```

But unfortunately, not supplying the type-arguments fails:
```tut:fail
resolveTest(1)(intTCResolver)
```

!![Halp](https://catmacros.files.wordpress.com/2009/07/cat-halp-1-1.jpg)