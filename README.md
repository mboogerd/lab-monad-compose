# Flexible Monadic Compositions

## Introduction
This is an experimental project facilitating loose monadic compositions. The idea comes from the observation that many
transformations we do to get monadic types aligned are both mechanical and repetitive. It gave me the incentive to
explore how such transformations could be simplified. I am as of yet uncertain whether the things that are needed are
worth the hassle, but let's find out!

## Overview
The general idea here is that we are looking at substituting computations of the type:
```scala
def flatMap[M[_], A, B](fa: M[A])(f: A => M[B]): M[B]
```

where `M` is a Monad (left out for brevity), with computations like:
```scala
def flatMap[M[_], N[_], L[_], A, B](fa: M[A])(f: A => N[B]): L[B]
```

where `N` and `L` are also Monads, and `L` is somehow derived from `M` and `N`.

## Transformations
- Preferred Monad in a class of isomorphic Monads; scalaz.Maybe and Scala's Option are isomorphic and should thus
compose easily (see doc/isomorphism.md)
- Unit wrapping Monads; `M[X].flatMap[x => N[M[X]]`
- Order for Monads; Future is strictly larger than Try, both are sum types of Failure and Success, Future adds time-uncertainty.
- Least upper bound (`lub`) for Monads, generalizing orders. The `lub[A, B]` can be `Either[A, B]` (some co-product more 
generally). we can distribute disjunction over possible sub-types: `lub[Either[A, B], Either[C, B]]` is `Either[Either[A, C], B]`,
Note that here `lub` is neither commutative, nor associative