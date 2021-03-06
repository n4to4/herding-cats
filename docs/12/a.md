---
out: Traverse.html
---

  [iterator2009]: http://www.comlab.ox.ac.uk/jeremy.gibbons/publications/iterator.pdf
  [TraverseSource]: $catsBaseUrl$/core/src/main/scala/cats/Traverse.scala
  [ControlMonadSequence]: https://downloads.haskell.org/~ghc/7.8.4/docs/html/libraries/base-4.7.0.2/Control-Monad.html#v:sequence
  [McBride2008]: http://strictlypositive.org/IdiomLite.pdf
  [FutureSequence]: http://www.scala-lang.org/api/2.11.6/index.html#scala.concurrent.Future\$

### Traverse

[The Essence of the Iterator Pattern][iterator2009]:

> Two of the three motivating examples McBride and Paterson provide for idiomatic computations — sequencing a list of monadic effects and transposing a matrix — are instances of a general scheme they call *traversal*.
> This involves iterating over the elements of a data structure, in the style of a `map`, but interpreting certain function applications idiomatically.
> ...
> We capture this via a type class of *Traversable* data structures.

In Cats, this typeclass is called [Traverse][TraverseSource]:

```scala
@typeclass trait Traverse[F[_]] extends Functor[F] with Foldable[F] { self =>

  /**
   * given a function which returns a G effect, thread this effect
   * through the running of this function on all the values in F,
   * returning an F[A] in a G context
   */
  def traverse[G[_]: Applicative, A, B](fa: F[A])(f: A => G[B]): G[F[B]]

  /**
   * thread all the G effects through the F structure to invert the
   * structure from F[G[_]] to G[F[_]]
   */
  def sequence[G[_]: Applicative, A](fga: F[G[A]]): G[F[A]] =
    traverse(fga)(ga => ga)
  ....
}
```

Note that the `f` takes the shape of `A => G[B]`.

> When *m* is specialised to the identity applicative functor,
> traversal reduces precisely (modulo the wrapper) to the functorial map over lists.

Cats' identity applicative functor is defined as follows:


```scala
  type Id[A] = A
  implicit val Id: Bimonad[Id] =
    new Bimonad[Id] {
      def pure[A](a: A): A = a
      def extract[A](a: A): A = a
      def flatMap[A, B](a: A)(f: A => B): B = f(a)
      def coflatMap[A, B](a: A)(f: A => B): B = f(a)
      override def map[A, B](fa: A)(f: A => B): B = f(fa)
      override def ap[A, B](fa: A)(ff: A => B): B = ff(fa)
      override def flatten[A](ffa: A): A = ffa
      override def map2[A, B, Z](fa: A, fb: B)(f: (A, B) => Z): Z = f(fa, fb)
      override def lift[A, B](f: A => B): A => B = f
      override def imap[A, B](fa: A)(f: A => B)(fi: B => A): B = f(fa)
  }
```

Here's how we can traverse over `List(1, 2, 3)` using `Id`.


```console:new
scala> import cats._, cats.instances.all._
scala> import cats.syntax.traverse._
scala> List(1, 2, 3) traverse[Id, Int] { (x: Int) => x + 1 }
```

> In the case of a monadic applicative functor, traversal specialises to monadic map, and has the same uses. In fact, traversal is really just a slight generalisation of monadic map.

Let's try using this for `List`:

```console
scala> List(1, 2, 3) traverse { (x: Int) => (Some(x + 1): Option[Int]) }
scala> List(1, 2, 3) traverse { (x: Int) => None }
```

> For a Naperian applicative functor, traversal transposes results.

We're going to skip this one.

> For a monoidal applicative functor, traversal accumulates values.
> The function `reduce` performs that accumulation, given an argument that assigns a value to each element

```console
scala> import cats.data.Const
scala> def reduce[A, B, F[_]](fa: F[A])(f: A => B)
         (implicit FF: Traverse[F], BB: Monoid[B]): B =
         {
           val g: A => Const[B, Unit] = { (a: A) => Const((f(a))) }
           val x = FF.traverse[Const[B, ?], A, Unit](fa)(g)
           x.getConst
         }
```

Here's how we can use this:

```console
scala> reduce(List('a', 'b', 'c')) { c: Char => c.toInt }
```

It sort of works, but it would be nicer if we can reduce the type annotation a bit.
The problem is that as it stands, Scala 2.11 compiler cannot infer `Const[B, ?]`.
There's a actually a trick to nudge the compiler into looking into several shapes called `Unapply`,
and we can use `traverseU` instead of `traverse` to take advantage of it:

```console
scala> def reduce[A, B, F[_]](fa: F[A])(f: A => B)
         (implicit FF: Traverse[F], BB: Monoid[B]): B =
         {
           val x = fa traverseU { (a: A) => Const((f(a))) }
           x.getConst
         }
```

We'll find out what this is about later.

### sequence function

`Applicative` and `Traverse` are mentioned together by
McBride and Paterson in [Applicative programming with effects][McBride2008].

As a background, until a few months ago (March 2015), the [sequence][ControlMonadSequence] function
in `Control.Monad` package used to look like this:

```haskell
-- | Evaluate each action in the sequence from left to right,
-- and collect the results.
sequence :: Monad m => [m a] -> m [a]
```

If I translate this into Scala, it would look like:

```scala
def sequence[G[_]: Monad, A](gas: List[G[A]]): G[List[A]]
```

It takes a list of monadic values, and returns a monadic value of a list.
This already looks pretty useful on its own,
but whenever you find a hardcoded `List` like this, we should suspect if it should
be replaced with a better typeclass.

McBride and Paterson first generalized the `sequence` function to
`dist`, by replacing `Monad` with `Applicative`:

```scala
def dist[G[_]: Applicative, A](gas: List[G[A]]): G[List[A]]
```

Next, they realized that `dist` is often called with `map` so they
added another parameter for applicative function, and called it `traverse`:

```scala
def traverse[G[_]: Applicative, A, B](as: List[A])(f: A => G[B]): G[List[B]]
```

Finally they generalized the above signature into a typeclass named `Traversible`:

```scala
@typeclass trait Traverse[F[_]] extends Functor[F] with Foldable[F] { self =>

  /**
   * given a function which returns a G effect, thread this effect
   * through the running of this function on all the values in F,
   * returning an F[A] in a G context
   */
  def traverse[G[_]: Applicative, A, B](fa: F[A])(f: A => G[B]): G[F[B]]

  /**
   * thread all the G effects through the F structure to invert the
   * structure from F[G[_]] to G[F[_]]
   */
  def sequence[G[_]: Applicative, A](fga: F[G[A]]): G[F[A]] =
    traverse(fga)(ga => ga)
  ....
}
```

Thus as the matter of course, `Traverse` implements a datatype-generic,
`sequence` function, which is just `traverse` with `identity`, conceptually easier to remember,
because it simply flips `F[G[A]]` into `G[F[A]]`.
You might have seen [this function][FutureSequence] for `Future` in the standard library.

```console
scala> import scala.concurrent.{ Future, ExecutionContext, Await }
scala> import scala.concurrent.duration._
scala> val x = {
         implicit val ec = scala.concurrent.ExecutionContext.global
         List(Future { 1 }, Future { 2 }).sequence
       }
scala> Await.result(x, 1 second)
```

Another useseful thing might be to turn a `List` of `Either` into an `Either`.

```console
scala> List(Right(1): Either[String, Int]).sequenceU
scala> List(Right(1): Either[String, Int], Left("boom"): Either[String, Int]).sequenceU
```

Note we just used `sequenceU`, which is the `Unapply` variant of `sequence`.
Let's look into that next.
