---
layout: docsplus
title:  "Tutorial"
position: 2
---

<nav role="navigation" id="toc"></nav>

## Introduction

This tutorial tries to help newcomers to cats-effect to get familiar with its
main concepts by means of code examples, in a learn-by-doing fashion. Two small
programs will be coded, each one in its own section. [The first
one](#copyingfiles) copies the contents from one file to another, safely
handling _resources_ and _cancellation_ in the process. That should help us to
flex our muscles. [The second one](#producerconsumer) implements a solution to
the producer-consumer problem to introduce cats-effect _fibers_.

This tutorial assumes certain familiarity with functional programming. It is
also a good idea to read cats-effect documentation prior to starting this
tutorial, at least the 
[excellent documentation about `IO` data type](../datatypes/io.md).

Please read this tutorial as training material, not as a best-practices
document. As you gain more experience with cats-effect, probably you will find
your own solutions to deal with the problems presented here. Also, bear in mind
that using cats-effect for copying files or implementing basic concurrency
patterns (such as the producer-consumer problem) is suitable for a 'getting
things done' approach, but for more complex systems/settings/requirements you
might want to take a look at [fs2](http://fs2.io) or [Monix](https://monix.io)
to find powerful network and file abstractions that integrate with cats-effect.
But that is beyond the purpose of this tutorial, which focuses solely on
cats-effect.

That said, let's go!

## Setting things up

This [Github repo](https://github.com/lrodero/cats-effect-tutorial) includes all
the software that will be developed during this tutorial. It uses `sbt` as the
build tool. To ease coding, compiling and running the code snippets in this
tutorial it is recommended to use the same `build.sbt`, or at least one with the
same dependencies and compilation options:

```scala
name := "cats-effect-tutorial"

version := "2.2.0"

scalaVersion := "2.12.8"

libraryDependencies += "org.typelevel" %% "cats-effect" % "2.2.0" withSources() withJavadoc()

scalacOptions ++= Seq(
  "-feature",
  "-deprecation",
  "-unchecked",
  "-language:postfixOps",
  "-language:higherKinds",
  "-Ypartial-unification")
```

Code snippets in this tutorial can be pasted and compiled right in the scala
console of the project defined above (or any project with similar settings).

## <a name="copyingfiles"></a>Copying files - basic concepts, resource handling and cancellation

Our goal is to create a program that copies files. First we will work on a
function that carries such task, and then we will create a program that can be
invoked from the shell and uses that function.

First of all we must code the function that copies the content from a file to
another file. The function takes the source and destination files as parameters.
But this is functional programming! So invoking the function shall not copy
anything, instead it will return an [`IO`](../datatypes/io.md) instance that
encapsulates all the side effects involved (opening/closing files,
reading/writing content), that way _purity_ is kept.  Only when that `IO`
instance is evaluated all those side-effectful actions will be run. In our
implementation the `IO` instance will return the amount of bytes copied upon
execution, but this is just a design decision. Of course errors can occur, but
when working with any `IO` those should be embedded in the `IO` instance. That
is, no exception is raised outside the `IO` and so no `try` (or the like) needs
to be used when using the function, instead the `IO` evaluation will fail and
the `IO` instance will carry the error raised.

Now, the signature of our function looks like this:

```scala
import cats.effect.IO
import java.io.File

def copy(origin: File, destination: File): IO[Long] = ???
```

Nothing scary, uh? As we said before, the function just returns an `IO`
instance. When run, all side-effects will be actually executed and the `IO`
instance will return the bytes copied in a `Long` (note that `IO` is
parameterized by the return type). Now, let's start implementing our function.
First, we need to open two streams that will read and write file contents.

### Acquiring and releasing `Resource`s
We consider opening a stream to be a side-effect action, so we have to
encapsulate those actions in their own `IO` instances. For this, we will make
use of cats-effect [`Resource`](../datatypes/resource.md), that allows to
orderly create, use and then release resources. See this code:

```scala
import cats.effect.{IO, Resource}
import cats.syntax.all._ 
import java.io._ 

def inputStream(f: File): Resource[IO, FileInputStream] =
  Resource.make {
    IO(new FileInputStream(f))                         // build
  } { inStream =>
    IO(inStream.close()).handleErrorWith(_ => IO.unit) // release
  }

def outputStream(f: File): Resource[IO, FileOutputStream] =
  Resource.make {
    IO(new FileOutputStream(f))                         // build 
  } { outStream =>
    IO(outStream.close()).handleErrorWith(_ => IO.unit) // release
  }

def inputOutputStreams(in: File, out: File): Resource[IO, (InputStream, OutputStream)] =
  for {
    inStream  <- inputStream(in)
    outStream <- outputStream(out)
  } yield (inStream, outStream)
```

We want to ensure that streams are closed once we are done using them, no matter
what. That is precisely why we use `Resource` in both `inputStream` and
`outputStream` functions, each one returning one `Resource` that encapsulates
the actions for opening and then closing each stream.  `inputOutputStreams`
encapsulates both resources in a single `Resource` instance that will be
available once the creation of both streams has been successful, and only in
that case. As seen in the code above `Resource` instances can be combined in
for-comprehensions as they implement `flatMap`. Note also that when releasing
resources we must also take care of any possible error during the release
itself, for example with the `.handleErrorWith` call as we do in the code above.
In this case we just swallow the error, but normally it should be at least
logged.

Optionally we could have used `Resource.fromAutoCloseable` to define our
resources, that method creates `Resource` instances over objects that implement
`java.lang.AutoCloseable` interface without having to define how the resource is
released. So our `inputStream` function would look like this:

```scala
import cats.effect.{IO, Resource}
import java.io.{File, FileInputStream}

def inputStream(f: File): Resource[IO, FileInputStream] =
  Resource.fromAutoCloseable(IO(new FileInputStream(f)))
```

That code is way simpler, but with that code we would not have control over what
would happen if the closing operation throws an exception. Also it could be that
we want to be aware when closing operations are being run, for example using
logs. In contrast, using `Resource.make` allows to easily control the actions
of the release phase.

Let's go back to our `copy` function, which now looks like this:

```scala
import cats.effect.{IO, Resource}
import java.io._

// as defined before
def inputOutputStreams(in: File, out: File): Resource[IO, (InputStream, OutputStream)] = ???

// transfer will do the real work
def transfer(origin: InputStream, destination: OutputStream): IO[Long] = ???

def copy(origin: File, destination: File): IO[Long] = 
  inputOutputStreams(origin, destination).use { case (in, out) => 
    transfer(in, out)
  }
```

The new method `transfer` will perform the actual copying of data, once the
resources (the streams) are obtained. When they are not needed anymore, whatever
the outcome of `transfer` (success or failure) both streams will be closed. If
any of the streams could not be obtained, then `transfer` will not be run. Even
better, because of `Resource` semantics, if there is any problem opening the
input file then the output file will not be opened.  On the other hand, if there
is any issue opening the output file, then the input stream will be closed.

### What about `bracket`?
Now, if you are familiar with cats-effect's
[`Bracket`](../typeclasses/bracket.md) you may be wondering why we are not using
it as it looks so similar to `Resource` (and there is a good reason for that:
`Resource` is based on `bracket`). Ok, before moving forward it is worth to take
a look to `bracket`.

There are three stages when using `bracket`: _resource acquisition_, _usage_,
and _release_. Each stage is defined by an `IO` instance.  A fundamental
property is that the _release_ stage will always be run regardless whether the
_usage_ stage finished correctly or an exception was thrown during its
execution. In our case, in the _acquisition_ stage we would create the streams,
then in the _usage_ stage we will copy the contents, and finally in the release
stage we will close the streams.  Thus we could define our `copy` function as
follows:

```scala
import cats.effect.IO
import cats.syntax.all._ 
import java.io._ 

// function inputOutputStreams not needed

// transfer will do the real work
def transfer(origin: InputStream, destination: OutputStream): IO[Long] = ???

def copy(origin: File, destination: File): IO[Long] = {
  val inIO: IO[InputStream]  = IO(new FileInputStream(origin))
  val outIO:IO[OutputStream] = IO(new FileOutputStream(destination))

  (inIO, outIO)              // Stage 1: Getting resources 
    .tupled                  // From (IO[InputStream], IO[OutputStream]) to IO[(InputStream, OutputStream)]
    .bracket{
      case (in, out) =>
        transfer(in, out)    // Stage 2: Using resources (for copying data, in this case)
    } {
      case (in, out) =>      // Stage 3: Freeing resources
        (IO(in.close()), IO(out.close()))
        .tupled              // From (IO[Unit], IO[Unit]) to IO[(Unit, Unit)]
        .handleErrorWith(_ => IO.unit).void
    }
}
```

New `copy` definition is more complex, even though the code as a whole is way
shorter as we do not need the `inputOutputStreams` function. But there is a
catch in the code above.  When using `bracket`, if there is a problem when
getting resources in the first stage, then the release stage will not be run.
Now, in the code above, first the origin file and then the destination file are
opened (`tupled` just reorganizes both `IO` instances into a single one). So
what would happen if we successfully open the origin file (_i.e._ when
evaluating `inIO`) but then an exception is raised when opening the destination
file (_i.e._ when evaluating `outIO`)? In that case the origin stream will not
be closed! To solve this we should first get the first stream with one `bracket`
call, and then the second stream with another `bracket` call inside the first.
But, in a way, that's precisely what we do when we `flatMap` instances of
`Resource`. And the code looks cleaner too. So, while using `bracket` directly
has its place, `Resource` is likely to be a better choice when dealing with
multiple resources at once.

### Copying data 
Finally we have our streams ready to go! We have to focus now on coding
`transfer`. That function will have to define a loop that at each iteration
reads data from the input stream into a buffer, and then writes the buffer
contents into the output stream. At the same time, the loop will keep a counter
of the bytes transferred. To reuse the same buffer we should define it outside
the main loop, and leave the actual transmission of data to another function
`transmit` that uses that loop. Something like:

```scala
import cats.effect.IO
import cats.syntax.all._ 
import java.io._ 

def transmit(origin: InputStream, destination: OutputStream, buffer: Array[Byte], acc: Long): IO[Long] =
  for {
    amount <- IO(origin.read(buffer, 0, buffer.size))
    count  <- if(amount > -1) IO(destination.write(buffer, 0, amount)) >> transmit(origin, destination, buffer, acc + amount)
              else IO.pure(acc) // End of read stream reached (by java.io.InputStream contract), nothing to write
  } yield count // Returns the actual amount of bytes transmitted

def transfer(origin: InputStream, destination: OutputStream): IO[Long] =
  for {
    buffer <- IO(new Array[Byte](1024 * 10)) // Allocated only when the IO is evaluated
    total  <- transmit(origin, destination, buffer, 0L)
  } yield total
```

Take a look at `transmit`, observe that both input and output actions are
encapsulated in (suspended in) `IO`. `IO` being a monad, we can sequence them
using a for-comprehension to create another `IO`. The for-comprehension loops as
long as the call to `read()` does not return a negative value that would signal
that the end of the stream has been reached. `>>` is a Cats operator to sequence
two operations where the output of the first is not needed by the second (_i.e._
it is equivalent to `first.flatMap(_ => second)`). In the code above that means
that after each write operation we recursively call `transmit` again, but as
`IO` is stack safe we are not concerned about stack overflow issues. At each
iteration we increase the counter `acc` with the amount of bytes read at that
iteration. 

We are making progress, and already have a version of `copy` that can be used.
If any exception is raised when `transfer` is running, then the streams will be
automatically closed by `Resource`. But there is something else we have to take
into account: `IO` instances execution can be **_canceled!_**. And cancellation
should not be ignored, as it is a key feature of cats-effect. We will discuss
cancellation in the next section.

### Dealing with cancellation
Cancellation is a powerful but non-trivial cats-effect feature. In cats-effect,
some `IO` instances can be canceled ( _e.g._ by other `IO` instaces running
concurrently) meaning that their evaluation will be aborted. If the programmer is
careful, an alternative `IO` task will be run under cancellation, for example to
deal with potential cleaning up activities.

Now, `IO`s created with `Resource.use` can be canceled. The cancellation will
trigger the execution of the code that handles the closing of the resource. In
our case, that would close both streams. So far so good! But what happens if
cancellation happens _while_ the streams are being used? This could lead to data
corruption as a stream where some thread is writing to is at the same time being
closed by another thread. For more info about this problem see
[Gotcha: Cancellation is a concurrent action](../datatypes/io.md#gotcha-cancellation-is-a-concurrent-action)
in cats-effect site.

To prevent such data corruption we must use some concurrency control mechanism
that ensures that no stream will be closed while the `IO` returned by `transfer`
is being evaluated.  Cats-effect provides several constructs for controlling
concurrency, for this case we will use a
[_semaphore_](../concurrency/semaphore.md). A semaphore has a number of permits,
its method `.acquire` 'blocks' if no permit is available until `release` is
called on the same semaphore. It is important to remark that _there is no actual
thread being really blocked_, the thread that finds the `.acquire` call will be
immediately recycled by cats-effect. When the `release` method is invoked then
cats-effect will look for some available thread to resume the execution of the
code after `.acquire`.

We will use a semaphore with a single permit. The `.withPermit` method acquires
one permit, runs the `IO` given and then releases the permit.  We could also
use `.acquire` and then `.release` on the semaphore explicitly, but
`.withPermit` is more idiomatic and ensures that the permit is released even if
the effect run fails.

```scala
import cats.syntax.all._
import cats.effect.{Concurrent, IO, Resource}
import cats.effect.concurrent.Semaphore
import java.io._

// transfer and transmit methods as defined before
def transfer(origin: InputStream, destination: OutputStream): IO[Long] = ???

def inputStream(f: File, guard: Semaphore[IO]): Resource[IO, FileInputStream] =
  Resource.make {
    IO(new FileInputStream(f))
  } { inStream => 
    guard.withPermit {
     IO(inStream.close()).handleErrorWith(_ => IO.unit)
    }
  }

def outputStream(f: File, guard: Semaphore[IO]): Resource[IO, FileOutputStream] =
  Resource.make {
    IO(new FileOutputStream(f))
  } { outStream =>
    guard.withPermit {
     IO(outStream.close()).handleErrorWith(_ => IO.unit)
    }
  }

def inputOutputStreams(in: File, out: File, guard: Semaphore[IO]): Resource[IO, (InputStream, OutputStream)] =
  for {
    inStream  <- inputStream(in, guard)
    outStream <- outputStream(out, guard)
  } yield (inStream, outStream)

def copy(origin: File, destination: File)(implicit concurrent: Concurrent[IO]): IO[Long] = {
  for {
    guard <- Semaphore[IO](1)
    count <- inputOutputStreams(origin, destination, guard).use { case (in, out) => 
               guard.withPermit(transfer(in, out))
             }
  } yield count
}
```

Before calling to `transfer` we acquire the semaphore, which is not released
until `transfer` is done. The `use` call ensures that the semaphore will be
released under any circumstances, whatever the result of `transfer` (success,
error, or cancellation). As the 'release' parts in the `Resource` instances are
now blocked on the same semaphore, we can be sure that streams are closed only
after `transfer` is over, _i.e._ we have implemented mutual exclusion of
`transfer` execution and resources releasing. An implicit `Concurrent` instance
is required to create the semaphore instance, which is included in the function
signature.

Mark that while the `IO` returned by `copy` is cancelable (because so are `IO`
instances returned by `Resource.use`), the `IO` returned by `transfer` is not.
Trying to cancel it will not have any effect and that `IO` will run until the
whole file is copied! In real world code you will probably want to make your
functions cancelable, section 
[Building cancelable IO tasks](../datatypes/io.md#building-cancelable-io-tasks) 
of `IO` documentation explains how to create such cancelable `IO` instances
(besides calling `Resource.use`, as we have done for our code).

And that is it! We are done, now we can create a program that uses this
`copy` function.

### `IOApp` for our final program

We will create a program that copies files, this program only takes two
parameters: the name of the origin and destination files. For coding this
program we will use [`IOApp`](../datatypes/ioapp.md) as it allows to maintain
purity in our definitions up to the program main function.

`IOApp` is a kind of 'functional' equivalent to Scala's `App`, where instead of
coding an effectful `main` method we code a pure `run` function. When executing
the class a `main` method defined in `IOApp` will call the `run` function we
have coded. Any interruption (like pressing `Ctrl-c`) will be treated as a
cancellation of the running `IO`. Also `IOApp` provides implicit instances of
`Timer[IO]` and `ContextShift[IO]` (not discussed yet in this tutorial).
`ContextShift[IO]` allows for having a `Concurrent[IO]` in scope, as the one
required by the `copy` function.

When coding `IOApp`, instead of a `main` function we have a `run` function,
which creates the `IO` instance that forms the program. In our case, our `run`
method can look like this:

```scala
import cats.effect._
import cats.syntax.all._
import java.io.File

object Main extends IOApp {

  // copy as defined before
  def copy(origin: File, destination: File): IO[Long] = ???

  override def run(args: List[String]): IO[ExitCode] =
    for {
      _      <- if(args.length < 2) IO.raiseError(new IllegalArgumentException("Need origin and destination files"))
                else IO.unit
      orig = new File(args(0))
      dest = new File(args(1))
      count <- copy(orig, dest)
      _     <- IO(println(s"$count bytes copied from ${orig.getPath} to ${dest.getPath}"))
    } yield ExitCode.Success
}
```

Heed how `run` verifies the `args` list passed. If there are fewer than two
arguments, an error is raised. As `IO` implements `MonadError` we can at any
moment call to `IO.raiseError` to interrupt a sequence of `IO` operations.

#### Copy program code
You can check the [final version of our copy program
here](https://github.com/lrodero/cats-effect-tutorial/blob/replace_tcp_section_with_prodcons/src/main/scala/catseffecttutorial/copyfile/CopyFile.scala).

The program can be run from `sbt` just by issuing this call:

```scala
> runMain catseffecttutorial.CopyFile origin.txt destination.txt
```

It can be argued that using `IO{java.nio.file.Files.copy(...)}` would get an
`IO` with the same characteristics of purity as our function. But there is a
difference: our `IO` is safely cancelable! So the user can stop the running code
at any time for example by pressing `Ctrl-c`, our code will deal with safe
resource release (streams closing) even under such circumstances. The same will
apply if the `copy` function is run from other modules that require its
functionality. If the `IO` returned by this function is canceled while being
run, still resources will be properly released. But recall what we commented
before: this is because `use` returns `IO` instances that are cancelable, in
contrast our `transfer` function is not cancelable.

WARNING: To properly test cancelation, You should also ensure that
`fork := true` is set in the sbt configuration, otherwise sbt will
intercept the cancelation because it will be running the program
in the same JVM as itself.

### Polymorphic cats-effect code
There is an important characteristic of `IO` that we shall be aware of. `IO` is
able to encapsulate side-effects, but the capacity to define concurrent and/or
async and/or cancelable `IO` instances comes from the existence of a
`Concurrent[IO]` instance. [`Concurrent`](../typeclasses/concurrent.md) is a
type class that, for an `F[_]` carrying a side-effect, brings the ability to
cancel or start concurrently the side-effect in `F`. `Concurrent` also extends
type class [`Async`](../typeclasses/async.md), that allows to define
synchronous/asynchronous computations. `Async`, in turn, extends type class
[`Sync`](../typeclasses/sync.md), which can suspend the execution of side
effects in `F`.

So well, `Sync` can suspend side effects (and so can `Async` and `Concurrent` as
they extend `Sync`). We have used `IO` so far mostly for that purpose. Now,
going back to the code we created to copy files, could have we coded its
functions in terms of some `F[_]: Sync` instead of `IO`? Truth is we could and
**in fact it is recommendable** in real world programs.  See for example how we
would define a polymorphic version of our `transfer` function with this
approach, just by replacing any use of `IO` by calls to the `delay` and `pure`
methods of the `Sync[F[_]]` instance!

```scala
import cats.effect.Sync
import cats.syntax.all._
import java.io._

def transmit[F[_]: Sync](origin: InputStream, destination: OutputStream, buffer: Array[Byte], acc: Long): F[Long] =
  for {
    amount <- Sync[F].delay(origin.read(buffer, 0, buffer.size))
    count  <- if(amount > -1) Sync[F].delay(destination.write(buffer, 0, amount)) >> transmit(origin, destination, buffer, acc + amount)
              else Sync[F].pure(acc) // End of read stream reached (by java.io.InputStream contract), nothing to write
  } yield count // Returns the actual amount of bytes transmitted
```

We leave as an exercise to code the polymorphic versions of `inputStream`,
`outputStream`, `inputOutputStreams` and `transfer` functions. You will
find that transformation similar to the one shown for `transfer` in the snippet
above. Function `copy` is different however. If you try to implement that
function as well you will realize that we need a full instance of
`Concurrent[F]` in scope, this is because it is required by the `Semaphore`
instantiation:

```scala
import cats.effect._
import cats.effect.concurrent.Semaphore
import cats.syntax.all._
import java.io._

def transmit[F[_]: Sync](origin: InputStream, destination: OutputStream, buffer: Array[Byte], acc: Long): F[Long] = ???
def transfer[F[_]: Sync](origin: InputStream, destination: OutputStream): F[Long] = ???
def inputStream[F[_]: Sync](f: File, guard: Semaphore[F]): Resource[F, FileInputStream] = ???
def outputStream[F[_]: Sync](f: File, guard: Semaphore[F]): Resource[F, FileOutputStream] = ???
def inputOutputStreams[F[_]: Sync](in: File, out: File, guard: Semaphore[F]): Resource[F, (InputStream, OutputStream)] = ???

def copy[F[_]: Concurrent](origin: File, destination: File): F[Long] = 
  for {
    guard <- Semaphore[F](1)
    count <- inputOutputStreams(origin, destination, guard).use { case (in, out) => 
               guard.withPermit(transfer(in, out))
             }
  } yield count
```

Only in our `main` function we will set `IO` as the final `F` for
our program. To do so, of course, a `Concurrent[IO]` instance must be in scope,
but that instance is brought transparently by `IOApp` so we do not need to be
concerned about it.

During the remaining of this tutorial we will use polymorphic code, only falling
to `IO` in the `run` method of our `IOApp`s. Polymorphic code is less
restrictive, as functions are not tied to `IO` but are applicable to any `F[_]`
as long as there is an instance of the type class required (`Sync[F[_]]` ,
`Concurrent[F[_]]`...) in scope. The type class to use will depend on the
requirements of our code. For example, if the execution of the side-effect
should be cancelable, then we must stick to `Concurrent[F[_]]`. Also, it is
actually easier to work on `F` than on any specific type.

#### Copy program code, polymorphic version
The polymorphic version of our copy program in full is available
[here](https://github.com/lrodero/cats-effect-tutorial/blob/replace_tcp_section_with_prodcons/src/main/scala/catseffecttutorial/copyfile/CopyFilePolymorphic.scala).

### Exercises: improving our small `IO` program

To finalize we propose you some exercises that will help you to keep improving
your IO-kungfu:

1. Modify the `IOApp` so it shows an error and abort the execution if the origin
   and destination files are the same, the origin file cannot be open for
   reading or the destination file cannot be opened for writing. Also, if the
   destination file already exists, the program should ask for confirmation
   before overwriting that file.
2. Modify `transmit` so the buffer size is not hardcoded but passed as
   parameter.
3. Use some other concurrency tool of cats-effect instead of `semaphore` to
   ensure mutual exclusion of `transfer` execution and streams closing.
4. Create a new program able to copy folders. If the origin folder has
   subfolders, then their contents must be recursively copied too. Of course the
   copying must be safely cancelable at any moment.

## <a name="producerconsumer"></a>Producer-consumer problem - concurrency and fibers
The _producer-consumer_ pattern is often found in concurrent setups. Here one or
more producers insert data on a shared data structure like a queue or buffer
while one or more consumers extract data from it. Readers and writers run
concurrently. If the queue is empty then readers will block until data is
available, if the queue is full then writers will wait for some 'bucket' to be
free. Only one writer at a time can add data to the queue to prevent data
corruption. Also only one reader can extract data from the queue so no two
readers get the same data item.

Variations of this problem exists depending on whether there are more than one
consumer/producer, or whether the data structure siting between them is
size-bounded or not. Unless stated otherwise, the solutions discussed here are
suited for multi consumer and multi reader settings. Initially the solutions
will assume an unbounded data structure, to then present a solution for a
bounded one.

But before we work on the solution for this problem we must introduce _fibers_,
which are the basic building block of cats-effect concurrency.

### Intro to fibers
A fiber carries an `F` action to execute (typically an `IO` instance). Fibers
are like 'light' threads, meaning they can be used in a similar way than threads
to create concurrent code. However, they are _not_ threads. Spawning new fibers
does not guarantee that the action described in the `F` associated to it will be
run if there is a shortage of threads. Internally cats-effect uses thread pools
to run fibers. So if there is no thread available in the pool then the fiber
execution will 'wait' until some thread is free again. On the other hand fibers
are, unlike threads, very cheap entities. We can spawn millions of them at ease
without impacting the performance.

`ContextShift[F]` is in charge of assigning threads to the fibers waiting to be
run. When using `IOApp` we get also the `ContextShift[IO]` that we need to run
the fibers in our code.  But the developer can also create new `ContextShift[F]`
instances using custom thread pools.

Cats-effect implements some concurrency primitives to coordinate concurrent
fibers: [Deferred](../concurrency/deferred.md),
[MVar2](../concurrency/mvar.md),
[Ref](../concurrency/ref.md) and
[Semaphore](../concurrency/semaphore.md)
(semaphores we already discussed in the first part of this tutorial). It is
important to understand that, when a fiber gets blocked by some concurrent data
structure, cats-effect recycles the thread so it becomes available for other
fibers. Cats-effect also recovers threads of finished and cancelled fibers.  But
keep in mind that, in contrast, if the fiber is blocked by some external action
like waiting for some input from a TCP socket, then cats-effect has no way to
recover back that thread until the action finishes.

Way more detailed info about concurrency in cats-effect can be found in [this
other tutorial 'Concurrency in Scala with
Cats-Effect'](https://github.com/slouc/concurrency-in-scala-with-ce). It is also
strongly advised to read the 
[Concurrency section of cats-effect docs](../concurrency/index.md). But for the
remaining of this tutorial we will focus on a practical approach to those
concepts.

Ok, now we have briefly discussed fibers we can start working on our
producer-consumer problem.

### First (and inefficient) implementation
We need an intermediate structure where producer(s) can insert data to and
consumer(s) extracts data from. Let's assume a simple queue. Initially there
will be only one producer and one consumer. Producer will generate a sequence of
integers (`1`, `2`, `3`...), consumer will just read that sequence.  Our shared
queue will be an instance of an immutable `Queue[Int]`.

As accesses to the queue can (and will!) be concurrent, we need some way to
protect the queue so only one fiber at a time is handling it. The best way to
ensure an ordered access to some shared data is 
[Ref](../concurrency/ref.md). A `Ref` instance
wraps some given data and implements methods to manipulate that data in a safe
manner. When some fiber is runnning one of those methods, any other call to any
method of the `Ref` instance will be blocked.

The `Ref` wrapping our queue will be `Ref[F, Queue[Int]]` (for some `F[_]`).

Now, our `producer` method will be:

```scala
import cats.effect._
import cats.effect.concurrent.Ref
import cats.syntax.all._
import collection.immutable.Queue

def producer[F[_]: Sync: ContextShift](queueR: Ref[F, Queue[Int]], counter: Int): F[Unit] =
  (for {
    _ <- if(counter % 10000 == 0) Sync[F].delay(println(s"Produced $counter items")) else Sync[F].unit
    _ <- queueR.getAndUpdate(_.enqueue(counter + 1)) // Putting data in shared queue
    _ <- ContextShift[F].shift
  } yield ()) >> producer(queueR, counter + 1)
```

First line just prints some log message every `10000` items, so we know if it is
'alive'. Then it calls `queueR.getAndUpdate` to add data into the queue. Note
that `.getAndUpdate` provides the current queue, then we use `.enqueue` to
insert the next value `counter+1`. This call returns a new queue with the value
added that is stored by the ref instance. If some other fiber is accessing to
`queueR` then the fiber is blocked.

Finally we run `ContextShift[F].shift` before calling again to the producer
recursively with the next counter value. In fact, the call to `.shift` is not
strictly needed but it is a good policy to include it in recursive functions.
Why is that? Well, internally cats-effect tries to assign fibers so all tasks
are given the chance to be executed. But in a recursive function that
potentially never ends it can be that the fiber is always running and
cats-effect has no change to recover the thread being used by it. By calling to
`.shift` the programmer explicitly tells cats-effect that the current thread
can be re-assigned to other fiber if so decides. Strictly speaking, maybe our
`producer` does not need such call as the access to `queueR.getAndUpdate` will
get the fiber blocked if some other fiber is using `queueR` at that moment, and
cats-effect can recycle blocked fibers for other tasks. Still, we keep it there
as good practice.

The `consumer` method is a bit different. It will try to read data from the
queue but it must be aware that the queue must be empty:

```scala
import cats.effect._
import cats.effect.concurrent.Ref
import cats.syntax.all._
import collection.immutable.Queue

def consumer[F[_] : Sync: ContextShift](queueR: Ref[F, Queue[Int]]): F[Unit] =
  (for {
    iO <- queueR.modify{ queue =>
      queue.dequeueOption.fold((queue, Option.empty[Int])){case (i,queue) => (queue, Option(i))}
    }
    _ <- if(iO.exists(_ % 10000 == 0)) Sync[F].delay(println(s"Consumed ${iO.get} items"))
      else Sync[F].unit
    _ <- ContextShift[F].shift
  } yield iO) >> consumer(queueR)
```

The call to `queueR.modify` allows to modify the wrapped data (our queue) and
return a value that is computed from that data. In our case, it returns an
`Option[Int]` that will be `None` if queue was empty. Next line is used to log
a message in console every `10000` read items. Finally `consumer` is called
recursively to start again.

We can now create a program that instantiates our `queueR` and runs both
`producer` and `consumer` in parallel:

```scala
import cats.effect._
import cats.effect.concurrent.Ref
import cats.syntax.all._
import collection.immutable.Queue

object InefficientProducerConsumer extends IOApp {

  def producer[F[_]: Sync: ContextShift](queueR: Ref[F, Queue[Int]], counter: Int): F[Unit] = ??? // As defined before
  def consumer[F[_] : Sync: ContextShift](queueR: Ref[F, Queue[Int]]): F[Unit] = ??? // As defined before

  override def run(args: List[String]): IO[ExitCode] =
    for {
      queueR <- Ref.of[IO, Queue[Int]](Queue.empty[Int])
      res <- (consumer(queueR), producer(queueR, 0))
        .parMapN((_, _) => ExitCode.Success) // Run producer and consumer in parallel until done (likely by user cancelling with CTRL-C)
        .handleErrorWith { t =>
          IO(println(s"Error caught: ${t.getMessage}")).as(ExitCode.Error)
        }
    } yield res

}
```

The full implementation of this naive producer consumer is available
[here](https://github.com/lrodero/cats-effect-tutorial/blob/replace_tcp_section_with_prodcons/src/main/scala/catseffecttutorial/producerconsumer/InefficientProducerConsumer.scala).

Our `run` function instantiates the shared queue wrapped in a `Ref` and boots
the producer and consumer in parallel. To do to it uses `parMapN`, that creates
and runs the fibers that will run the `IO`s passed as paremeter. Then it takes
the output of each fiber and and applies a given function to them. In our case
both producer and consumer shall run forever until user presses CTRL-C which
will trigger a cancellation.

Alternatively we could have used `start` method to explicitely create new
`Fiber` instances that will run the producer and consumer, then use `join` to
wait for them to finish, something like:

```scala
def run(args: List[String]): IO[ExitCode] =
  for {
    queueR <- Ref.of[IO, Queue[Int]](Queue.empty[Int])
    producerFiber <- producer(queueR, 0).start
    consumerFiber <- consumer(queueR, 0).start
    _ <- producerFiber.join
    _ <- consumerFiber.join
  } yield ExitCode.Error
```

Problem is, if there is an error in any of the fibers the `join` call will not
hint it, nor it will return. In contrast `parMapN` does promote the error it
finds to the caller. _In general, if possible, programmers should prefer to use
higher level commands such as `parMapN` or `parSequence` to deal with fibers_.

Ok, we stick to our implementation based on `.parMapN`. Are we done? Does it
Work? Well, it works... but it is far from ideal. If we run it we will find that
the producer runs faster than the consumer so the queue is constantly growing.
And, even if that was not the case, we must realize that the consumer will be
continually running regardless if there are elements in the queue, which is far
from ideal. We will try to improve it in the next section by using
[Deferred](../concurrency/deferred.md). Also we will use several consumers and
producers to balance production and consumption rate.

### A more solid implementation of the producer/consumer problem
In our producer/consumer code we already protect access to the queue (our shared
resource) using a `Ref`. Now, instead of using `Option` to represent elements
retrieved from a possibly empty queue, we should instead block the caller
somehow if queue is empty until some element can be returned. This will be done
by creating and keeping instances of `Deferred`. A `Deferred[F, A]` instance can
hold one single element of some type `A`. `Deferred` instances are created
empty, and can be filled only once. If some fiber tries to read the element from
an empty `Deferred` then it will be blocked until some other fiber fills
(completes) it.

Thus, alongside the queue of produced but not yet consumed elements, we have to
keep track of the `Deferred` instances created when the queue was empty and
waiting for elements to be available. This instances will be kept in a new queue
`takers`. We will keep both queues in a new type `State`:

```scala
import cats.effect.concurrent.Deferred
import scala.collection.immutable.Queue
case class State[F[_], A](queue: Queue[A], takers: Queue[Deferred[F,A]])
```

Both producer and consumer will access the same shared state instance, which
will be carried and safely modified by an instance of `Ref`. Consumer shall:
1. If `queue` is not empty, it will extract and return its head. The new state
   will keep the tail of the queue, and need not modify `takers`.
2. If `queue` is empty it will instantiate a new `taker`, add it to the
   `takers` queue, and 'block' the caller by invoking `taker.get`

Assuming that in our setting we produce and consume `Int`s (just as before),
then new consumer code will then be:

```scala
import cats.effect.{ContextShift, Sync}
import cats.effect.concurrent.{Deferred, Ref}
import scala.collection.immutable.Queue

case class State[F[_], A](queue: Queue[A], takers: Queue[Deferred[F,A]])

def consumer[F[_]: Concurrent: ContextShift](id: Int, stateR: Ref[F, State[F, Int]]): F[Unit] = {

  val take: F[Int] =
    Deferred[F, Int].flatMap { taker =>
      stateR.modify {
        case State(queue, takers) if queue.nonEmpty =>
          val (i, rest) = queue.dequeue
          State(rest, takers) -> Sync[F].pure(i)
        case State(queue, takers) =>
          State(queue, takers.enqueue(taker)) -> taker.get
      }.flatten
    }

  (for {
    i <- take
    _ <- if(i % 10000 == 0) Sync[F].delay(println(s"Consumer $id has reached $i items")) else Sync[F].unit
    _ <- ContextShift[F].shift
  } yield ()) >> consumer(id, stateR)
}
```

The `id` parameter is only used to identify the consumer in console logs (recall
we will have now several producers and consumers running in parallel). The
`take` instance implements the checking and updating of the state in `stateR`.
Note how it will block on `taker.get` when the queue is empty.

The producer, for its part, will:
1. If there are waiting `takers`, it will take the first in the queue and offer
   it the newly produced element (`taker.complete`).
2. If no `takers` are present, it will just enqueue the produced element.

Thus the producer will look like:

```scala
import cats.effect.{ContextShift, Sync}
import cats.effect.concurrent.{Deferred, Ref}
import scala.collection.immutable.Queue

case class State[F[_], A](queue: Queue[A], takers: Queue[Deferred[F,A]])

def producer[F[_]: Sync: ContextShift](id: Int, counterR: Ref[F, Int], stateR: Ref[F, State[F,Int]]): F[Unit] = {

  def offer(i: Int): F[Unit] =
    stateR.modify {
      case State(queue, takers) if takers.nonEmpty =>
        val (taker, rest) = takers.dequeue
        State(queue, rest) -> taker.complete(i).void
      case State(queue, takers) =>
        State(queue.enqueue(i), takers) -> Sync[F].unit
    }.flatten

  (for {
    i <- counterR.getAndUpdate(_ + 1)
    _ <- offer(i)
    _ <- if(i % 10000 == 0) Sync[F].delay(println(s"Producer $id has reached $i items")) else Sync[F].unit
    _ <- ContextShift[F].shift
  } yield ()) >> producer(id, counterR, stateR)
}
```

Finally we modify our main program so it instantiates the counter and state
`Ref`s. Also it will create several consumers and producers, 10 of each, and
will start all of them in parallel:

```scala
import cats.effect.concurrent.{Ref, Deferred}
import cats.effect._
import cats.instances.list._
import cats.syntax.all._
import scala.collection.immutable.Queue

object ProducerConsumer extends IOApp {

  case class State[F[_], A](queue: Queue[A], takers: Queue[Deferred[F,A]])

  object State {
    def empty[F[_], A]: State[F, A] = State(Queue.empty, Queue.empty)
  }

  def producer[F[_]: Sync: ContextShift](id: Int, counterR: Ref[F, Int], stateR: Ref[F, State[F,Int]]): F[Unit] = ??? // As defined before

  def consumer[F[_]: Concurrent: ContextShift](id: Int, stateR: Ref[F, State[F, Int]]): F[Unit] = ??? // As defined before

  override def run(args: List[String]): IO[ExitCode] =
    for {
      stateR <- Ref.of[IO, State[IO,Int]](State.empty[IO, Int])
      counterR <- Ref.of[IO, Int](1)
      producers = List.range(1, 11).map(producer(_, counterR, stateR)) // 10 producers
      consumers = List.range(1, 11).map(consumer(_, stateR))           // 10 consumers
      res <- (producers ++ consumers)
        .parSequence.as(ExitCode.Success) // Run producers and consumers in parallel until done (likely by user cancelling with CTRL-C)
        .handleErrorWith { t =>
          IO(println(s"Error caught: ${t.getMessage}")).as(ExitCode.Error)
        }
    } yield res
}
```

The full implementation of this producer consumer with unbounded queue is
available
[here](https://github.com/lrodero/cats-effect-tutorial/blob/queue_using_Ref_and_Defer/src/main/scala/catseffecttutorial/producerconsumer/ProducerConsumer.scala).

Producers and consumers are created as two list of `IO` instances. All of them
are started in their own fiber by the call to `parSequence`, which will wait for
all of them to finish and then return the value passed as parameter. As in the
previous example it shall run forever until the user presses CTRL-C.

Having several consumers and producers improves the balance between consumers
and producers... but still, on the long run, queue tends to grow in size. To
fix this we will ensure the size of the queue is bounded, so whenever that max
size is reached producers will block as consumers do when the queue is empty.

### Producer consumer with bounded queue
Having a bounded queue implies that producers, when queue is full, will wait (be
'blocked') until there is some empty bucket available to be filled. So an
implementation needs to keep track of these waiting producers. To do so we will
add a new queue `offerers` that will be added to the `State` alongside `takers`.
For each waiting producer the `offerers` queue will keep a `Deferred[F, Unit]`
that will be used to block the producer until the element it offers can be added
to `queue` or directly passed to some consumer. Alongside the `Deferred`
instance we need to keep as well the actual element offered by the producer in
the `offerers` queue. Thus `State` class now becomes:

```scala
import cats.effect.concurrent.Deferred
import scala.collection.immutable.Queue

case class State[F[_], A](queue: Queue[A], capacity: Int, takers: Queue[Deferred[F,A]], offerers: Queue[(A, Deferred[F,Unit])])
```

Of course both consumer and producer have to be modified to handle this new
queue `offerers`. A consumer can find four escenarios, depending on if `queue`
and `offerers` are each one empty or not. For each escenario a consumer shall:
1. If `queue` is not empty:
  1.1 If `offerers` is empty then it will extract and return `queue`'s head.
  1.2 If `offerers` is not empty (there is some producer waiting) then things
      are more complicated. The `queue` head will be returned to the consumer.
      Now we have a free bucket available in `queue`. So the first waiting
      offerer can be retrieved from `offerers`. The element it produced will be
      added to `queue`, and the `Deferred` instance will be completed so the
      producer is released (unblocked).
2. If `queue` is empty:
  2.1 If `offerers` is empty then there is nothing we can give to the caller,
      so a new `taker` is created and added to `takers` while caller is
      blocked with `taker.get`.
  2.2 If `offerers` is not empty then the first offerer in queue is extracted,
      its `Deferred` instance released while the offered element is returned to
      the caller.

So consumer code looks like this:

```scala
import cats.effect.{ContextShift, Sync}
import cats.effect.concurrent.{Deferred, Ref}
import scala.collection.immutable.Queue

case class State[F[_], A](queue: Queue[A], capacity: Int, takers: Queue[Deferred[F,A]], offerers: Queue[(A, Deferred[F,Unit])])

def consumer[F[_]: Concurrent: ContextShift](id: Int, stateR: Ref[F, State[F, Int]]): F[Unit] = {

  val take: F[Int] =
    Deferred[F, Int].flatMap { taker =>
      stateR.modify {
        case State(queue, capacity, takers, offerers) if queue.nonEmpty && offerers.isEmpty =>
          val (i, rest) = queue.dequeue
          State(rest, capacity, takers, offerers) -> Sync[F].pure(i)
        case State(queue, capacity, takers, offerers) if queue.nonEmpty =>
          val (i, rest) = queue.dequeue
          val ((move, release), tail) = offerers.dequeue
          State(rest.enqueue(move), capacity, takers, tail) -> release.complete(()).as(i)
        case State(queue, capacity, takers, offerers) if offerers.nonEmpty =>
          val ((i, release), rest) = offerers.dequeue
          State(queue, capacity, takers, rest) -> release.complete(()).as(i)
        case State(queue, capacity, takers, offerers) =>
          State(queue, capacity, takers.enqueue(taker), offerers) -> taker.get
      }.flatten
    }

  (for {
    i <- take
    _ <- if(i % 10000 == 0) Sync[F].delay(println(s"Consumer $id has reached $i items")) else Sync[F].unit
    _ <- ContextShift[F].shift
  } yield ()) >> consumer(id, stateR)
}
```

Producer functionality is a bit easier:
1. If there is any waiting `taker` then the produced element will be passed to
   it releasing the blocked fiber.
2. If there is no waiting `taker` but `queue` is not empty, then offered element
   will be enqueued there.
3. If there is no waiting `taker` and `queue` is already full then a new
   `offerer` is created, blocking the producer fiber on the `.get` method of the
   `Deferred` instance.

Now producer code looks like this:

```scala
import cats.effect.{ContextShift, Sync}
import cats.effect.concurrent.{Deferred, Ref}
import scala.collection.immutable.Queue

case class State[F[_], A](queue: Queue[A], capacity: Int, takers: Queue[Deferred[F,A]], offerers: Queue[(A, Deferred[F,Unit])])

def producer[F[_]: Concurrent: ContextShift](id: Int, counterR: Ref[F, Int], stateR: Ref[F, State[F,Int]]): F[Unit] = {

  def offer(i: Int): F[Unit] =
    Deferred[F, Unit].flatMap[Unit]{ offerer =>
      stateR.modify {
        case State(queue, capacity, takers, offerers) if takers.nonEmpty =>
          val (taker, rest) = takers.dequeue
          State(queue, capacity, rest, offerers) -> taker.complete(i).void
        case State(queue, capacity, takers, offerers) if queue.size < capacity =>
          State(queue.enqueue(i), capacity, takers, offerers) -> Sync[F].unit
        case State(queue, capacity, takers, offerers) =>
          State(queue, capacity, takers, offerers.enqueue(i -> offerer)) -> offerer.get
      }.flatten
    }

  (for {
    i <- counterR.getAndUpdate(_ + 1)
    _ <- offer(i)
    _ <- if(i % 10000 == 0) Sync[F].delay(println(s"Producer $id has reached $i items")) else Sync[F].unit
    _ <- ContextShift[F].shift
  } yield ()) >> producer(id, counterR, stateR)
}
```

As you see, producer and consumer are coded around the idea of keeping and
modifying state, just as with unbounded queues.

As the final step we must adapt the main program to use these new consumers and
producers, so if we limit the queue size to 100 then we have:

```scala
import cats.effect.concurrent.{Deferred, Ref}
import cats.effect.{Concurrent, ContextShift, ExitCode, IO, IOApp, Sync}
import cats.instances.list._
import cats.syntax.all._

import scala.collection.immutable.Queue

object ProducerConsumerBounded extends IOApp {

  case class State[F[_], A](queue: Queue[A], capacity: Int, takers: Queue[Deferred[F,A]], offerers: Queue[(A, Deferred[F,Unit])])

  object State {
    def empty[F[_], A](capacity: Int): State[F, A] = State(Queue.empty, capacity, Queue.empty, Queue.empty)
  }

  def producer[F[_]: Concurrent: ContextShift](id: Int, counterR: Ref[F, Int], stateR: Ref[F, State[F,Int]]): F[Unit] = ??? // As defined before

  def consumer[F[_]: Concurrent: ContextShift](id: Int, stateR: Ref[F, State[F, Int]]): F[Unit] = ??? // As defined before

  override def run(args: List[String]): IO[ExitCode] =
    for {
      stateR <- Ref.of[IO, State[IO, Int]](State.empty[IO, Int](capacity = 100))
      counterR <- Ref.of[IO, Int](1)
      producers = List.range(1, 11).map(producer(_, counterR, stateR)) // 10 producers
      consumers = List.range(1, 11).map(consumer(_, stateR))           // 10 consumers
      res <- (producers ++ consumers)
        .parSequence.as(ExitCode.Success) // Run producers and consumers in parallel until done (likely by user cancelling with CTRL-C)
        .handleErrorWith { t =>
          IO(println(s"Error caught: ${t.getMessage}")).as(ExitCode.Error)
        }
    } yield res
}
```

The full implementation of this producer consumer with bounded queue is
available
[here](https://github.com/lrodero/cats-effect-tutorial/blob/queue_using_Ref_and_Defer/src/main/scala/catseffecttutorial/producerconsumer/ProducerConsumerBounded.scala).

### Taking care of cancellation
We shall ask ourselves, is this implementation cancellation-safe? That is, what
happens if the fiber running a consumer or a producer gets cancelled? Does state
become inconsistent? Let's focus on `offer` first. For the sake of clarity in our
analysis let's reformat the code using a for-comprehension:

```scala
def offer(i: Int): F[Unit] =
  for {
    offerer <- Deferred[F, Int]
    op      <- stateR.modify {...} // It returns an F[] that must be run!
    _       <- op
  } yield ()
```

So far so good. Now, cancellation steps into action in each `.flatMap` in `F`,
that is, in each step of our for-comprehension. If the fiber gets canceled right
before or after the first step, well, that is not an issue. The `offerer` will
be eventually garbage collected, that's all. But what if the cancellation
happens right after the call to `modify`? Well, that `op` will not be run. 



HERE
HERE
HERE
HERE
HERE
HERE
HERE
HERE
HERE
HERE
HERE
HERE
HERE


### Why not using semaphores in the producer-consumer problem?
The producer-consumer problem is far from new in concurrent programming.
_Semaphores_ are a classical tool used for this problem. As discussed in the
first section of this tutorial, semaphores are used to control access to
critical sections of our code that handle shared resources, when those sections
can be run concurrently. Each semaphore contains a number that represents
_permits_. To access a critical section permits have to be obtained, later to be
released when the critical section is left.

However, in cats-effect (at least until 3.0 is released), semaphores are _not_ a
good fit for the producer-consumer problem. Why is that? Well, semaphores would
be used to keep count of the number of produced elements available to be
consumed.  Consumers would ask for permits to retrieve those elements, while
producers will increase the number of permits to after adding new elements. So,
a bare bones consumer can look something like:

```scala
for {
    _ <- filled.acquire // Wait for some item in queue by acquiring a permit
    i <- queueR.modify(_.dequeue.swap) // Retrieving element
    _ <- consume(e)
} yield(())
``` 

Where `queueR` is a `Ref` that wraps a `Queue` of elements, and `filled` is our
instance of `Semaphore` counting the elements in the queue. Conversely the producer
implementation will increase the semaphore after adding a new element.

And what is wrong with that code? Well, nothing by itself, but we have to take
_cancellation_ into account. It is possible that the consumer is canceled just
after getting a permit but before extracting the element from the queue. In that
case the semaphore will not be a proper representation of the number of elements
in the queue.

But wait! Alternatively, if the consumer is an `IO` instance,  we can try to
make its code 'uncancelable'. Our producer then becomes:
```scala
IO.uncancelable {
  for {
    _ <- filled.acquire // Wait for some item in queue by acquiring a permit
    e <- queueR.modify(_.dequeue.swap) // Retrieving element
    _ <- consume(e)
  } yield(())
}
``` 

Unfortunately there is a new problem here: now we cannot cancel our consumer! So
it will be 'stuck' forever if no elements are added to the queue. For example,
the handy `IO.timeout` method will _not_ work as it tries to cancel the `IO`
instance it is invoked upon, which is not possible if we make the `IO`
uncancelable. The `uncancelable` method in cats-effect 3 will introduce a
different signature to circunvent this problem. But in cats-effect 2 this is not
solvable.

### Exercise: build a concurrent queue
A _concurrent queue_ is, well, a queue data structure that allows safe
concurrent access. That is, several concurrent processes can safely add and
retrieve data from the queue. It is easy to realize that during the previous
sections we already implemented that kind of functionality, it was
embedded in our `producer` and `consumer` functions. To build a concurrent queue
we only need to extract from those methods the part that handles the concurrent
access.

A possible implementation of the concurrent queue is given
[here](https://github.com/lrodero/cats-effect-tutorial/blob/queue_using_Ref_and_Defer/src/main/scala/catseffecttutorial/producerconsumer/exerciseconcurrentqueue/Queue.scala)
for reference. **NOTE** this concurrent queue implementation is basically the
same [queue that will be available in cats-effect 3 std
package](https://github.com/typelevel/cats-effect/blob/series/3.x/std/shared/src/main/scala/cats/effect/std/Queue.scala)
with minor modifications so it runs on cats-effect 2.

## Conclusion

With all this we have covered a good deal of what cats-effect has to offer (but
not all!). Now you are ready to use to create code that operate side effects in
a purely functional manner. Enjoy the ride!
