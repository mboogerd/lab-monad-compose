# Default typeclass for a set of isomorphic typeclasses

Some typeclasses are isomorphic, which implies that we can safely convert between them. If a conversion 
of instance `x` of typeclass `F` returns an instance `y` of typeclass `G`, then the isomorphism provides
us with a transformation of `y` back to `x`.

Isomorphisms exist for many types and type-constructors, often because different libraries define the same 
concept in different ways. An example is `Option` from the Scala collection library and its isomorphic
`Maybe` in Scalaz. 

```scala
import scalaz.Isomorphism.<~>
import scalaz.Maybe
```

When we have multiple isomorphic type-classes, it can be useful to pick a favorite or default and have
conversion be done automatic, to reduce such manual plumbing.

```scala
trait Default[F[_]]
object Default {
  def apply[F[_]]: Default[F] = new Default[F]{}
}

implicit class DefaultOps[T, F[_]](f: F[T]) {
  def default[G[_]: Default](implicit iso: F <~> G): G[T] = iso.to(f)
}

implicit val maybeOptionIso: Maybe <~> Option = scalaz.Maybe.optionMaybeIso.flip
implicit val optionMaybeIso: Option <~> Maybe = scalaz.Maybe.optionMaybeIso
```

The `Default` class enriches any applied type-constructor `f` with a `default` method, which will
convert it to the preferred instance `G`, if a `Default` is in scope for `G`, and an isomorphism exists
between the two type-constructors. Note that the supplied `Default` is never actually used, it is only used 
to constrain the implicit search space.

Enough talk, demo time:

```scala
scala> {
     |   implicit val maybeDefault: Default[Maybe] = Default[Maybe]
     |   Option(5).default
     | }
res2: scalaz.Maybe[Int] = Just(5)

scala> {
     |   implicit val maybeDefault: Default[Maybe] = Default[Maybe]
     |   Maybe.just(5).default
     | }
res3: scalaz.Maybe[Int] = Just(5)

scala> {
     |   implicit val optionDefault: Default[Option] = Default[Option]
     |   Option(5).default
     | }
res4: Option[Int] = Some(5)

scala> {
     |   implicit val optionDefault: Default[Option] = Default[Option]
     |   Maybe.just(5).default
     | }
res5: Option[Int] = Some(5)
```

Note that in the middle two cases, the used instance is the preferred one. The isomorphism at work behind the scenes
is the default reflexive one from scalaz itself, which effectively returns the instance.

```scala
implicit def isoNaturalRefl[F[_]]: F <~> F = ... // Implementation can be found in scalaz.Isomorphism
```

# Default monadic composition

Once we have established a method to set a Default for a given set of isomorphic type(-constructor)s,
nothing stops us to apply these to more complicated cases. Let's try our hand at monadic composition:

```scala
import scalaz.Monad
import scalaz.std.option.optionInstance
import scalaz.Maybe.maybeInstance

class DefaultMonad[S, M[_]: Monad](m: M[S]) {
  import scalaz.syntax.monad._
  def map[T](f: S ⇒ T): M[T] = m.map(f)
  def flatMap[T, N[_]: Monad, D[_]: Default: Monad](f: S ⇒ N[T])(implicit md: M <~> D, nd: N <~> D): D[T] =
    md.to(m).flatMap(s ⇒ nd.to(f(s)))
}

implicit class DefaultMonadOps[S, M[_]: Monad](m: M[S]) {
  def defaultMonad: DefaultMonad[S, M] = new DefaultMonad(m)
}
```
Here we have defined a `DefaultMonad` and an extension method to make it available. It has a non-standard `flatMap`
that is slightly more flexible in the function that it can receive. A Monad `M[S]`'s `flatMap` normally takes functions
of the shape `S => M[T]`, i.e. `M` is kept constant. Here we allow for variability, as long as an isomorphism exists
between either Monad and the Default one.

Let's set up some demo scenario:

```scala
def some1 = Option(1)
def just2 = Maybe.just(2)
def some3 = Option(3)
def just4 = Maybe.just(4)

def crossMonadComposition[M[_]: Default: Monad](implicit md: Option <~> M, nd: Maybe <~> M): M[Int] = {
  for {
    first ← some1.defaultMonad
    second ← just2.defaultMonad
    third ← some3.defaultMonad
    fourth ← just4.defaultMonad
  } yield first + second + third + fourth
}
```

Here our `crossMonadComposition` type signature contains complexity simply so we can vary the Default in the demo-code
below. Normally, these implicits would be globally available, here we want to provide them at will:

```scala
scala> {
     |   implicit val optionDefault: Default[Option] = Default[Option]
     |   crossMonadComposition
     | }
res9: Option[Int] = Some(10)

scala> {
     |   implicit val maybeDefault: Default[Maybe] = Default[Maybe]
     |   crossMonadComposition
     | }
res10: scalaz.Maybe[Int] = Just(10)
```

Shown in this way, it probably comes across as a funny, but over-engineered gimmick. Whenever isomorphisms exist, it is
usually trivial to provide a combinator that transforms to a desired default explicitly, if so desired. However, here we
only scratched the surface, keeping things simple for ourselves on purpose. Being agile developers, what is the first
improvement we could make to this?

One thing with our DefaultMonad is that it seems somewhat overzealous. Both 'source' and 'target' Monad's are converted
to the Default. This might be entirely unnecessary. We could say we stay 'truer' to the original computation if only
one of them gets converted, i.e. the 'worst' Monad of the two gets converted to the 'better' of the two. We could do so
if there was a way of providing a preference ordering over an entire class of isomorphic types. If ever our favorite
Monad gets involved in a composition, the resulting Monad is our favorite one, but otherwise all Monad instances are 
upgraded just to reach a common, most preferred one.
