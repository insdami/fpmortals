

# For Comprehensions

Scala's `for` comprehension is the ideal FP abstraction for sequential
programs that interact with the world. Since we'll be using it a lot,
we're going to relearn the principles of `for` and how cats can help
us to write cleaner code.

This chapter doesn't try to write pure programs and the techniques are
applicable to non-FP codebases.

## Syntax Sugar

Scala's `for` is just a simple rewrite rule, also called *syntax
sugar*, that doesn't have any contextual information.

To see what a `for` comprehension is doing, we use the `show` and
`reify` feature in the REPL to print out what code looks like after
type inference.

{lang="text"}
~~~~~~~~
  scala> import scala.reflect.runtime.universe._
  scala> val a, b, c = Option(1)
  scala> show { reify {
           for { i <- a ; j <- b ; k <- c } yield (i + j + k)
         } }
  
  res:
  $read.a.flatMap(
    ((i) => $read.b.flatMap(
      ((j) => $read.c.map(
        ((k) => i.$plus(j).$plus(k)))))))
~~~~~~~~

There is a lot of noise due to additional sugarings (e.g. `+` is
rewritten `$plus`, etc). We'll skip the `show` and `reify` for brevity
when the REPL line is `reify>`, and manually clean up the generated
code so that it doesn't become a distraction.

{lang="text"}
~~~~~~~~
  reify> for { i <- a ; j <- b ; k <- c } yield (i + j + k)
  
  a.flatMap {
    i => b.flatMap {
      j => c.map {
        k => i + j + k }}}
~~~~~~~~

The rule of thumb is that every `<-` (called a *generator*) is a
nested `flatMap` call, with the final generator a `map` containing the
`yield` body.

### Assignment

We can assign values inline like `ij = i + j` (a `val` keyword is not
needed).

{lang="text"}
~~~~~~~~
  reify> for {
           i <- a
           j <- b
           ij = i + j
           k <- c
         } yield (ij + k)
  
  a.flatMap {
    i => b.map { j => (j, i + j) }.flatMap {
      case (j, ij) => c.map {
        k => ij + k }}}
~~~~~~~~

A `map` over the `b` introduces the `ij` which is flat-mapped along
with the `j`, then the final `map` for the code in the `yield`.

Unfortunately we cannot assign before any generators. It has been
requested as a language feature but has not been implemented:
<https://github.com/scala/bug/issues/907>

{lang="text"}
~~~~~~~~
  scala> for {
           initial = getDefault
           i <- a
         } yield initial + i
  <console>:1: error: '<-' expected but '=' found.
~~~~~~~~

We can workaround the limitation by defining a `val` outside the `for`

{lang="text"}
~~~~~~~~
  scala> val initial = getDefault
  scala> for { i <- a } yield initial + i
~~~~~~~~

or create an `Option` out of the initial assignment

{lang="text"}
~~~~~~~~
  scala> for {
           initial <- Option(getDefault)
           i <- a
         } yield initial + i
~~~~~~~~

A> `val` doesn't have to assign to a single value, it can be anything
A> that works as a `case` in a pattern match.
A> 
A> {lang="text"}
A> ~~~~~~~~
A>   scala> val (first, second) = ("hello", "world")
A>   first: String = hello
A>   second: String = world
A>   
A>   scala> val list: List[Int] = ...
A>   scala> val head :: tail = list
A>   head: Int = 1
A>   tail: List[Int] = List(2, 3)
A> ~~~~~~~~
A> 
A> The same is true for assignment in `for` comprehensions
A> 
A> {lang="text"}
A> ~~~~~~~~
A>   scala> val maybe = Option(("hello", "world"))
A>   scala> for {
A>            entry <- maybe
A>            (first, _) = entry
A>          } yield first
A>   res: Some(hello)
A> ~~~~~~~~
A> 
A> But be careful that you don't miss any cases or you'll get a runtime
A> exception (a *totality* failure).
A> 
A> {lang="text"}
A> ~~~~~~~~
A>   scala> val a :: tail = list
A>   caught scala.MatchError: List()
A> ~~~~~~~~

### Filter

It is possible to put `if` statements after a generator to filter
values by a predicate

{lang="text"}
~~~~~~~~
  reify> for {
           i  <- a
           j  <- b
           if i > j
           k  <- c
         } yield (i + j + k)
  
  a.flatMap {
    i => b.withFilter {
      j => i > j }.flatMap {
        j => c.map {
          k => i + j + k }}}
~~~~~~~~

Older versions of scala used `filter`, but `Traversable.filter`
creates new collections for every predicate, so `withFilter` was
introduced as the more performant alternative.

We can accidentally trigger a `withFilter` by providing type
information: it's actually interpreted as a pattern match.

{lang="text"}
~~~~~~~~
  reify> for { i: Int <- a } yield i
  
  a.withFilter {
    case i: Int => true
    case _      => false
  }.map { case i: Int => i }
~~~~~~~~

Like in assignment, a generator can use a pattern match on the left
hand side. But unlike assignment (which throws `MatchError` on
failure), generators are *filtered* and will not fail at runtime.
However, there is an inefficient double application of the pattern.

### For Each

Finally, if there is no `yield`, the compiler will use `foreach`
instead of `flatMap`, which is only useful for side-effects.

{lang="text"}
~~~~~~~~
  reify> for { i <- a ; j <- b } println(s"$i $j")
  
  a.foreach { i => b.foreach { j => println(s"$i $j") } }
~~~~~~~~

### Summary

The full set of methods supported by `for` comprehensions do not share
a common super type; each generated snippet is independently compiled.
If there were a trait, it would roughly look like:

{lang="text"}
~~~~~~~~
  trait ForComprehensible[C[_]] {
    def map[A, B](f: A => B): C[B]
    def flatMap[A, B](f: A => C[B]): C[B]
    def withFilter[A](p: A => Boolean): C[A]
    def foreach[A](f: A => Unit): Unit
  }
~~~~~~~~

If the context (`C[_]`) of a `for` comprehension doesn't provide its
own `map` and `flatMap`, all is not lost. If an implicit
`cats.FlatMap[T]` is available for `T`, it will provide `map` and
`flatMap`.

A> It often surprises developers when inline `Future` calculations in a
A> `for` comprehension do not run in parallel:
A> 
A> {lang="text"}
A> ~~~~~~~~
A>   import scala.concurrent._
A>   import ExecutionContext.Implicits.global
A>   
A>   for {
A>     i <- Future { expensiveCalc() }
A>     j <- Future { anotherExpensiveCalc() }
A>   } yield (i + j)
A> ~~~~~~~~
A> 
A> This is because the `flatMap` spawning `anotherExpensiveCalc` is
A> strictly **after** `expensiveCalc`. To ensure that two `Future`
A> calculations begin in parallel, start them outside the `for`
A> comprehension.
A> 
A> {lang="text"}
A> ~~~~~~~~
A>   val a = Future { expensiveCalc() }
A>   val b = Future { anotherExpensiveCalc() }
A>   for { i <- a ; j <- b } yield (i + j)
A> ~~~~~~~~
A> 
A> `for` comprehensions are fundamentally for defining sequential
A> programs. We will show a far superior way of defining parallel
A> computations in a later chapter.

## Unhappy path

So far we've only looked at the rewrite rules, not what is happening
in `map` and `flatMap`. Let's consider what happens when the `for`
context decides that it can't proceed any further.

In the `Option` example, the `yield` is only called when `i,j,k` are
all defined.

{lang="text"}
~~~~~~~~
  for {
    i <- a
    j <- b
    k <- c
  } yield (i + j + k)
~~~~~~~~

If any of `a,b,c` are `None`, the comprehension short-circuits with
`None` but it doesn't tell us what went wrong.

A> How often have you seen a function that takes `Option` parameters but
A> requires them all to exist? An alternative to throwing a runtime
A> exception is to use a `for` comprehension, giving us totality (a
A> return value for every input):
A> 
A> {lang="text"}
A> ~~~~~~~~
A>   def namedThings(
A>     someName  : Option[String],
A>     someNumber: Option[Int]
A>   ): Option[String] = for {
A>     name   <- someName
A>     number <- someNumber
A>   } yield s"$number ${name}s"
A> ~~~~~~~~
A> 
A> but this is verbose, clunky and bad style. If a function requires
A> every input then it should make its requirement explicit, pushing the
A> responsibility of dealing with optional parameters to its caller ---
A> don't use `for` unless you need to.
A> 
A> {lang="text"}
A> ~~~~~~~~
A>   def namedThings(name: String, num: Int) = s"$num ${name}s"
A> ~~~~~~~~

If we use `Either`, then a `Left` will cause the `for` comprehension
to short circuit with extra information, much better than `Option` for
error reporting:

{lang="text"}
~~~~~~~~
  scala> val a = Right(1)
  scala> val b = Right(2)
  scala> val c: Either[String, Int] = Left("sorry, no c")
  scala> for { i <- a ; j <- b ; k <- c } yield (i + j + k)
  
  Left(sorry, no c)
~~~~~~~~

And lastly, let's see what happens with a `Future` that fails:

{lang="text"}
~~~~~~~~
  scala> import scala.concurrent._
  scala> import ExecutionContext.Implicits.global
  scala> for {
           i <- Future.failed[Int](new Throwable)
           j <- Future { println("hello") ; 1 }
         } yield (i + j)
  scala> Await.result(f, duration.Duration.Inf)
  caught java.lang.Throwable
~~~~~~~~

The `Future` that prints to the terminal is never called because, like
`Option` and `Either`, the `for` comprehension short circuits.

Short circuiting for the unhappy path is a common and important theme.
`for` comprehensions cannot express resource cleanup: there is no way
to `try` / `finally`. This is good, in FP it puts a clear ownership of
responsibility for unexpected error recovery and resource cleanup onto
the context (which is usually a `Monad` as we'll see later), not the
business logic.

## Gymnastics

Although it's easy to rewrite simple sequential code as a `for`
comprehension, sometimes we'll want to do something that appears to
require mental summersaults. This section collects some practical
examples and how to deal with them.

### Fallback Logic

Let's say we are calling out to a method that returns an `Option` and
if it's not successful we want to fallback to another method (and so
on and so on), like when we're using a cache:

{lang="text"}
~~~~~~~~
  def getFromRedis(s: String): Option[String]
  def getFromSql(s: String): Option[String]
  
  getFromRedis(key) orElse getFromSql(key)
~~~~~~~~

If we have to do this for an asynchronous version of the same API

{lang="text"}
~~~~~~~~
  def getFromRedis(s: String): Future[Option[String]]
  def getFromSql(s: String): Future[Option[String]]
~~~~~~~~

then we have to be careful not to do extra work because

{lang="text"}
~~~~~~~~
  for {
    cache <- getFromRedis(key)
    sql   <- getFromSql(key)
  } yield cache orElse sql
~~~~~~~~

will run both queries. We can pattern match on the first result but
the type is wrong

{lang="text"}
~~~~~~~~
  for {
    cache <- getFromRedis(key)
    res   <- cache match {
               case Some(_) => cache !!! wrong type !!!
               case None    => getFromSql(key)
             }
  } yield res
~~~~~~~~

We need to create a `Future` from the `cache`

{lang="text"}
~~~~~~~~
  for {
    cache <- getFromRedis(key)
    res   <- cache match {
               case Some(_) => Future.successful(cache)
               case None    => getFromSql(key)
             }
  } yield res
~~~~~~~~

`Future.successful` creates a new `Future`, much like an `Option` or
`List` constructor.

If functional programming was like this all the time, it'd be a
nightmare. Thankfully these tricky situations are the corner cases.

A> We could code golf it and write
A> 
A> {lang="text"}
A> ~~~~~~~~
A>   getFromRedis(key) orElseM getFromSql(key)
A> ~~~~~~~~
A> 
A> by defining <https://github.com/typelevel/cats/issues/1625> but it can
A> be a cognitive burden to remember all these helper methods. The level
A> of verbosity of a codebase vs code reuse of trivial functions is a
A> stylistic decision for each team.

### Early Exit

Let's say we have some condition that should exit early.

If we want to exit early and flag it as an error we just use the
context's shortcut, e.g. synchronous code that throws an exception

{lang="text"}
~~~~~~~~
  def getA: Int = ...
  
  val a = getA
  require(a > 0, s"$a must be positive")
  a * 10
~~~~~~~~

can be rewritten as async

{lang="text"}
~~~~~~~~
  def getA: Future[Int] = ...
  def error(msg: String): Future[Nothing] =
    Future.fail(new RuntimeException(msg))
  
  for {
    a <- getA
    b <- if (a <= 0) error(s"$a must be positive")
         else Future.successful(a)
  } yield b * 10
~~~~~~~~

But if we want to exit early with a successful return value, we have
to use a nested `for` comprehension. e.g.

{lang="text"}
~~~~~~~~
  def getA: Int = ...
  def getB: Int = ...
  
  val a = getA
  if (a <= 0) 0
  else a * getB
~~~~~~~~

is rewritten asynchronously as

{lang="text"}
~~~~~~~~
  def getA: Future[Int] = ...
  def getB: Future[Int] = ...
  
  for {
    a <- getA
    c <- if (a <= 0) Future.successful(0)
         else for { b <- getB } yield a * b
  } yield c
~~~~~~~~

A> If there is an implicit `Monad[T[_]]` for `T` (i.e. `T` is monadic)
A> then cats lets us create a `T[V]` from a value `v:V` by calling
A> `v.pure[T]`.
A> 
A> Cats provides `Monad[Future]` and `.pure[Future]` simply calling
A> `Future.successful`. Apart from `pure` being slightly shorter to type,
A> it is good practice to use it because it is a general concept that
A> works for all monadic contexts.
A> 
A> {lang="text"}
A> ~~~~~~~~
A>   for {
A>     a <- getA
A>     c <- if (a <= 0) 0.pure[Future]
A>          else for { b <- getB } yield a * b
A>   } yield c
A> ~~~~~~~~

## Incomprehensible

You may have noticed that the context we're comprehending over must
stay the same: we can't mix contexts.

{lang="text"}
~~~~~~~~
  scala> def option: Option[Int] = ...
  scala> def future: Future[Int] = ...
  scala> for {
           a <- option
           b <- future
         } yield a * b
  <console>:23: error: type mismatch;
   found   : Future[Int]
   required: Option[?]
           b <- future
                ^
~~~~~~~~

Nothing can help us mix arbitrary contexts in a `for` comprehension,
because the meaning is not well defined.

But when we have nested contexts the intention is usually obvious yet
the compiler still doesn't accept our code.

{lang="text"}
~~~~~~~~
  scala> def getA: Future[Option[Int]] = ...
  scala> def getB: Future[Option[Int]] = ...
  scala> for {
           a <- getA
           b <- getB
         } yield a * b
  <console>:30: error: value * is not a member of Option[Int]
         } yield a * b
                   ^
~~~~~~~~

Here we want `for` to take care of the outer context and let us write
our code on the inner `Option`. Hiding the outer context is exactly
what a *monad transformer* does, and cats provides implementations for
`Option`, `Future` and `Either` named `OptionT`, `FutureT` and
`EitherT` respectively.

The outer context can be anything that normally works in a `for`
comprehension, but it needs to stay the same throughout.

We create an `OptionT` from each method call. This changes the context
of the `for` from `Future[Option, _]` to `OptionT[Future, _]`.

Don't forget the import statements from the Practicalities chapter.

{lang="text"}
~~~~~~~~
  scala> val result = for {
           a <- OptionT(getA)
           b <- OptionT(getB)
         } yield a * b
  result: OptionT[Future, Int] = OptionT(Future(<not completed>))
~~~~~~~~

Call `.value` to return to the original context

{lang="text"}
~~~~~~~~
  scala> result.value
  res: Future[Option[Int]] = Future(<not completed>)
~~~~~~~~

Alternatively, `OptionT[Future, Int]` has `getOrElse` and `getOrElseF`
methods, taking an `Int` or `Future[Int]` respectively, and returning
a `Future[Int]`.

The monad transformer also allows us to mix `Future[Option[_]]` calls
with methods that just return plain `Future` via `OptionT.liftF`

{lang="text"}
~~~~~~~~
  scala> def getC: Future[Int] = ...
  scala> val result = for {
           a <- OptionT(getA)
           b <- OptionT(getB)
           c <- OptionT.liftF(getC)
         } yield a * b / c
  result: OptionT[Future, Int] = OptionT(Future(<not completed>))
~~~~~~~~

and we can mix with methods that return plain `Option` by wrapping
them in `Future.successful` (`.pure[Future]`) followed by `OptionT`

{lang="text"}
~~~~~~~~
  scala> def getD: Option[Int] = ...
  scala> val result = for {
           a <- OptionT(getA)
           b <- OptionT(getB)
           c <- OptionT.liftF(getC)
           d <- OptionT(getD.pure[Future])
         } yield (a * b) / (c * d)
  result: OptionT[Future, Int] = OptionT(Future(<not completed>))
~~~~~~~~

It's gotten messy again, but it's better than writing nested `flatMap`
and `map` by hand. We can clean this up with a DSL that handles all
the required conversions into `OptionT[Future, _]`

{lang="text"}
~~~~~~~~
  implicit class Ops[In](in: In) {
    def |>[Out](f: In => Out): Out = f(in)
  }
  def liftFutureOption[A](f: Future[Option[A]]) = OptionT(f)
  def liftFuture[A](f: Future[A]) = OptionT.liftF(f)
  def liftOption[A](o: Option[A]) = OptionT(o.pure[Future])
  def lift[A](a: A)               = liftOption(Some(a))
~~~~~~~~

which has a visual separation of the logic from transformation

{lang="text"}
~~~~~~~~
  scala> val result = for {
           a <- getA       |> liftFutureOption
           b <- getB       |> liftFutureOption
           c <- getC       |> liftFuture
           d <- getD       |> liftOption
           e <- 10         |> lift
         } yield e * (a * b) / (c * d)
  result: OptionT[Future, Int] = OptionT(Future(<not completed>))
~~~~~~~~

This approach also works for `EitherT` and `FutureT` as the inner
context, but their lifting methods are more complex as they require
parameters to construct the `Left` and an implicit `ExecutionContext`.
cats provides monad transformers for a lot of its own types, so it's
worth checking if one is available.

Notably absent is `ListT` (or `TraversableT`) because it is difficult
to create a well-behaved monad transformer for collections. It comes
down to the unfortunate fact that grouping of the operations can
unintentionally reorder `flatMap` calls.
<https://github.com/typelevel/cats/issues/977> aims to implement
`ListT`. Implementing a monad transformer is an advanced topic.

# Application Design

In this chapter we will write the business logic and tests for a
purely functional server application.

## Specification

Our application will manage a just-in-time build farm on a shoestring
budget. It will listen to a [Drone](https://github.com/drone/drone) Continuous Integration server, and
spawn worker agents using [Google Container Engine](https://cloud.google.com/container-engine/) (GKE) to meet the
demand of the work queue.

{width=60%}
![](images/architecture.png)

Drone receives work when a contributor submits a github pull request
to a managed project. Drone assigns the work to its agents, each
processing one job at a time.

The goal of our app is to ensure that there are enough agents to
complete the work, with a cap on the number of agents, whilst
minimising the total cost. Our app needs to know the number of items
in the *backlog* and the number of available *agents*.

Google can spawn *nodes*, each can host multiple drone agents. When an
agent starts up, it registers itself with drone and drone takes care
of the lifecycle (including keep-alive calls to detect removed
agents).

GKE charges a fee per minute of uptime, rounded up to the nearest hour
for each node. One does not simply spawn a new node for each job in
the work queue, we must re-use nodes and retain them until their 59th
minute to get the most value for money.

Our apps needs to be able to start and stop nodes, as well as check
their status (e.g. uptimes, list of inactive nodes) and to know what
time GKE believes it to be.

In addition, there is no API to talk directly to an *agent* so we do
not know if any individual agent is performing any work for the drone
server. If we accidentally stop an agent whilst it is performing work,
it is inconvenient and requires a human to restart the job.

The failure mode should always be to take the least costly option.

Both Drone and GKE have a JSON over REST API with OAuth 2.0
authentication.

## Interfaces / Algebras

Let's codify the architecture diagram from the previous section.

In FP, an *algebra* takes the place of an `interface` in Spring Java,
or the set of valid messages for an Actor in Akka. This is the layer
where we define all side-effecting interactions of our system.

We only define operations that we use in our business logic, avoiding
implementation detail. In reality, there is tight iteration between
writing the logic and the algebra: it is just the right level of
abstraction to design a system.

The `@freestyle.free` annotation is a macro that generates
boilerplate. The details of the boilerplate are not important right
now, we will explain as required and go into gruelling detail in the
Appendix.

`@free` requires that all methods return `FS[A]`, which we will
replace with `Id[A]` or `Future[A]` when we implement them, just like
in the Introduction.

{lang="text"}
~~~~~~~~
  package algebra
  
  import java.time.ZonedDateTime
  import cats.data.NonEmptyList
  import freestyle._
  
  object drone {
    @free trait Drone {
      def getBacklog: FS[Int]
      def getAgents: FS[Int]
    }
  }
  
  object machines {
    case class Node(id: String)
  
    @free trait Machines {
      def getTime: FS[ZonedDateTime] // current time
      def getManaged: FS[NonEmptyList[Node]]
      def getAlive: FS[Map[Node, ZonedDateTime]] // node and its start zdt
      def start(node: Node): FS[Unit]
      def stop(node: Node): FS[Unit]
    }
  }
~~~~~~~~

We've used `cats.data.NonEmptyList`, a wrapper around the standard
library's `List`, otherwise everything should be familiar.

A> It is good practice in FP to encode constraints in parameters **and**
A> return types --- it means we never need to handle situations that are
A> impossible. However, this often conflicts with the *Effective Java*
A> wisdom of unconstrained parameters and specific return types.
A> 
A> Although we agree that parameters should be as general as possible, we
A> do not agree that a function should take `Traversable` unless it can
A> handle empty collections. If it is not possible to handle the empty
A> case the only course of action would be to signal an error, breaking
A> totality and causing a side effect.

## Business Logic

Now we write the business logic that defines the app's behaviour,
considering only the happy path.

First, the imports

{lang="text"}
~~~~~~~~
  package logic
  
  import java.time.ZonedDateTime
  import java.time.temporal.ChronoUnit
  import scala.concurrent.duration._
  import cats.data.NonEmptyList
  import cats.implicits._
  import freestyle._
  import algebra.drone._
  import algebra.machines._
~~~~~~~~

and a `WorldView` class that holds a snapshot of our knowledge of the
world. If we were designing this application in Akka, `WorldView`
would probably be a `var` in an `Actor`.

`WorldView` aggregates the return values of all the methods in the
algebras, and adds a *pending* field to track unfulfilled requests.

{lang="text"}
~~~~~~~~
  final case class WorldView(
    backlog: Int,
    agents: Int,
    managed: NonEmptyList[Node],
    alive: Map[Node, ZonedDateTime],
    pending: Map[Node, ZonedDateTime], // requested at zdt
    time: ZonedDateTime
  )
~~~~~~~~

We use the freestyle `@module` boilerplate generator and declare all
the algebras that our business logic depends on. Then we create a
`class` to hold our business logic, taking the `Deps` module as an
implicit parameter. We're just doing dependency injection, it should
be a familiar pattern if you've ever used Spring. `@module` has
generated the type `F` for us, combining the algebras.

{lang="text"}
~~~~~~~~
  @module trait Deps {
    val d: Drone
    val c: Machines
  }
  
  final case class DynAgents[F[_]](implicit D: Deps[F]) {
    import D._
~~~~~~~~

Our business logic will run in an infinite loop (pseudocode)

{lang="text"}
~~~~~~~~
  state = initial()
  loop {
    state = update(state)
    state = act(state)
  }
~~~~~~~~

Which means we must write three functions: `initial`, `update` and
`act`, all returning a `WorldView`.

`@free` and `@module` together expand type declarations like `FS[A]`
into `FreeS[F, A]`. When we return a `WorldView`, it must be a
`FreeS[F, WorldView]`.

`FreeS[F, A]` is *monadic*, meaning that an implementation of
`Monad[A]` is available for the operations in our combined algebras,
`F`. As we discovered in the Introduction, a `Monad` is the
description of a program, interpreted by an execution context at
runtime.

### initial

In `initial` we call all external services and aggregate their results
into a `WorldView`. We default the `pending` field to an empty `Map`.

{lang="text"}
~~~~~~~~
  def initial: FreeS[F, WorldView] = for {
    w  <- d.getBacklog
    a  <- d.getAgents
    av <- c.getManaged
    ac <- c.getAlive
    t  <- c.getTime
  } yield WorldView(w, a, av, ac, Map.empty, t)
~~~~~~~~

### update

We will need a convenience function to calculate the time difference
between two `ZonedDateTime` instances

{lang="text"}
~~~~~~~~
  def diff(from: ZonedDateTime, to: ZonedDateTime): FiniteDuration =
    ChronoUnit.MINUTES.between(from, to).minutes
~~~~~~~~

`update` calls `initial` to refresh our world view, but preserves
`pending` actions.

If a pending action is taking longer than 10 minutes to do anything,
we assume that it failed and just forget that we ever asked to do it.

Note that we're using a generator (`flatMap`) when we call a method
from an algebra, but we can use assignment for pure functions like
`copy` and `diff`. The compiler keeps us right.

{lang="text"}
~~~~~~~~
  def update(world: WorldView): FreeS[F, WorldView] = for {
    snap <- initial
    update = snap.copy(
      pending = world.pending.filterNot {
        case (n, started) => diff(started, snap.time) >= 10.minutes
      }
    )
  } yield update
~~~~~~~~

### TODO act

## TODO Unit Tests

An architect's dream: you can focus on algebras, business logic and
functional requirements, and delegate the implementations to your
teams.

## TODO Parallel

## TODO Implementing OAuth 2.0

