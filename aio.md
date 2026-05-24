# Asynchronous IO (AIO) on the JVM with Scala 3  

## Introduction  
AIO lets a program start an IO request and continue doing other work.  
The operation finishes later, and the runtime reports the result through  
a callback or another completion mechanism.  
This model is useful when reads and writes can take unpredictable time.  
Disk latency, network latency, and slow remote peers all fit that case.  
Blocking IO stops the current thread until the operation completes.  
Non-blocking IO returns immediately too, but it usually requires polling  
or readiness notification through selectors.  
AIO differs from classic non-blocking IO because the operating system or  
runtime completes the whole operation and then notifies your code.  
On the JVM, `AsynchronousFileChannel` supports asynchronous file reads,  
writes, and locks.  
`AsynchronousSocketChannel` provides the same idea for TCP clients, and  
`AsynchronousServerSocketChannel` does it for async accepts.  
These APIs are built around `CompletionHandler`, which has `completed`  
and `failed` methods for success and error paths.  
That callback style maps well to event-driven code, but it can become  
hard to compose when many steps depend on each other.  
Scala helps here because a callback can complete a `Promise`, and that  
promise can expose a `Future` with richer combinators.  
With futures, you can sequence AIO steps with `map`, `flatMap`, and a  
for-comprehension instead of nesting callbacks.  
AIO still uses threads internally, because completion events must run on  
some executor or channel-group thread pool.  
The goal is not to remove threads, but to avoid keeping one blocked per  
in-flight operation.  
That difference matters when a program handles many slow operations at  
once, such as thousands of socket connections or many file requests.  
Thread pools should stay bounded, because too many worker threads cause  
context switching, cache misses, and higher memory use.  
AIO is most beneficial when latency is high and concurrency is also high.  
It is often less helpful for tiny local file reads that complete almost  
instantly from the page cache.  
For network servers, long waits are common, so async sockets can improve  
scalability and keep callback threads free for useful work.  
For file workloads, the benefit depends on the platform, the storage, and  
how much overlapping work the application can do.  
The examples below show the low-level Java AIO APIs and then bridge them  
into the more composable Scala concurrency tools.  

## Asynchronous file IO basics  
These examples show how `AsynchronousFileChannel` performs low-level AIO.  

## Opening an AsynchronousFileChannel  
Open an async file channel for later reads and writes.  

```scala
import java.nio.channels.AsynchronousFileChannel
import java.nio.file.{Files, StandardOpenOption}

@main def main() =
  val path = Files.createTempFile("aio-open", ".txt")
  val channel =
    AsynchronousFileChannel.open(
      path,
      StandardOpenOption.READ,
      StandardOpenOption.WRITE
    )

  try
    println(s"Channel open: ${channel.isOpen}")
  finally
    channel.close()
    Files.deleteIfExists(path)
end main
```

`AsynchronousFileChannel.open` creates a handle that can start file work  
without blocking the current thread until completion.  
This first example only checks the channel state, but the same handle is  
used by later examples for async reads and writes.  

## Reading a File Asynchronously with a CompletionHandler  
Read bytes into a buffer and print them from the completion callback.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, StandardOpenOption}
import java.util.concurrent.CountDownLatch

@main def main() =
  val path = Files.createTempFile("aio-read", ".txt")
  Files.writeString(path, "asynchronous file read")

  val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
  val buffer = ByteBuffer.allocate(64)
  val latch = CountDownLatch(1)

  try
    channel.read(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          buffer.flip()
          val bytes = Array.ofDim[Byte](buffer.remaining())
          buffer.get(bytes)
          println(new String(bytes, StandardCharsets.UTF_8))
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"read failed: ${exc.getMessage}")
          latch.countDown()
    )

    latch.await()
  finally
    channel.close()
    Files.deleteIfExists(path)
end main
```

The `read` call returns immediately, and the handler runs later when the  
data is available.  
A `CountDownLatch` keeps `main` alive long enough for the callback to run  
and then allows the cleanup logic to close the channel safely.  

## Writing to a File Asynchronously with a CompletionHandler  
Write bytes to a temp file and read them back after the callback fires.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, StandardOpenOption}
import java.util.concurrent.CountDownLatch

@main def main() =
  val path = Files.createTempFile("aio-write", ".txt")
  val channel =
    AsynchronousFileChannel.open(
      path,
      StandardOpenOption.WRITE,
      StandardOpenOption.CREATE
    )
  val buffer = ByteBuffer.wrap("written asynchronously".getBytes())
  val latch = CountDownLatch(1)

  try
    channel.write(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          println(s"bytes written: $result")
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"write failed: ${exc.getMessage}")
          latch.countDown()
    )

    latch.await()
    println(Files.readString(path, StandardCharsets.UTF_8))
  finally
    channel.close()
    Files.deleteIfExists(path)
end main
```

The `ByteBuffer` holds the outgoing bytes, and the write starts at file  
position `0L`.  
After the latch opens, the example reads the file synchronously only to  
show that the asynchronous write completed successfully.  

## Handling Success and Failure in CompletionHandler  
Implement both callback branches so success and failure stay explicit.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, StandardOpenOption}
import java.util.concurrent.CountDownLatch

@main def main() =
  val path = Files.createTempFile("aio-handler", ".txt")
  Files.writeString(path, "handler demo")

  val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
  val buffer = ByteBuffer.allocate(32)
  val latch = CountDownLatch(1)

  try
    channel.read(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          buffer.flip()
          val bytes = Array.ofDim[Byte](buffer.remaining())
          buffer.get(bytes)
          println(s"success: ${new String(bytes, StandardCharsets.UTF_8)}")
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"failure: ${exc.getClass.getSimpleName}")
          latch.countDown()
    )

    latch.await()
  finally
    channel.close()
    Files.deleteIfExists(path)
end main
```

A completion handler should always define a clear error path, even when a  
demo is expected to succeed.  
Real failures can come from permission problems, closed channels, or IO  
errors reported later by the operating system.  

## Using a Custom Thread Pool for AsynchronousFileChannel  
Create a channel that reports completions on a dedicated executor.  

```scala
import java.nio.channels.AsynchronousFileChannel
import java.nio.file.{Files, StandardOpenOption}
import java.util.concurrent.Executors
import scala.jdk.CollectionConverters.*

@main def main() =
  val path = Files.createTempFile("aio-pool", ".txt")
  val pool = Executors.newFixedThreadPool(2)
  val options =
    Set(
      StandardOpenOption.CREATE,
      StandardOpenOption.READ,
      StandardOpenOption.WRITE
    ).asJava

  val channel = AsynchronousFileChannel.open(path, options, pool)

  try
    println(s"open on custom pool: ${channel.isOpen}")
  finally
    channel.close()
    pool.shutdown()
    Files.deleteIfExists(path)
end main
```

A custom executor gives you control over how many threads are available  
for completion work.  
That matters when you want to isolate AIO callbacks from other thread  
pools or cap resource use under load.  

## Reading Multiple Regions of a File in Parallel  
Start two async reads at different positions and wait for both results.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, StandardOpenOption}
import java.util.concurrent.CountDownLatch

@main def main() =
  val path = Files.createTempFile("aio-regions-read", ".txt")
  Files.writeString(path, "alpha-beta-gamma")

  val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
  val left = ByteBuffer.allocate(5)
  val right = ByteBuffer.allocate(5)
  val latch = CountDownLatch(2)

  def startRead(buffer: ByteBuffer, position: Long): Unit =
    channel.read(
      buffer,
      position,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          buffer.flip()
          val bytes = Array.ofDim[Byte](buffer.remaining())
          buffer.get(bytes)
          println(new String(bytes, StandardCharsets.UTF_8))
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"region read failed: ${exc.getMessage}")
          latch.countDown()
    )

  try
    startRead(left, 0L)
    startRead(right, 6L)
    latch.await()
  finally
    channel.close()
    Files.deleteIfExists(path)
end main
```

Each read uses its own buffer and file position, so the requests can be  
in flight at the same time.  
This pattern is useful when an application knows the offsets of several  
independent regions that can be fetched concurrently.  

## Writing Multiple Regions of a File in Parallel  
Write separate chunks to different offsets without blocking `main`.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, StandardOpenOption}
import java.util.concurrent.CountDownLatch

@main def main() =
  val path = Files.createTempFile("aio-regions-write", ".txt")
  val channel =
    AsynchronousFileChannel.open(
      path,
      StandardOpenOption.CREATE,
      StandardOpenOption.WRITE,
      StandardOpenOption.READ
    )
  val latch = CountDownLatch(2)

  def startWrite(text: String, position: Long): Unit =
    val buffer = ByteBuffer.wrap(text.getBytes(StandardCharsets.UTF_8))
    channel.write(
      buffer,
      position,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          println(s"wrote $result bytes at $position")
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"region write failed: ${exc.getMessage}")
          latch.countDown()
    )

  try
    startWrite("alpha", 0L)
    startWrite("beta", 6L)
    latch.await()
    println(Files.readString(path, StandardCharsets.UTF_8))
  finally
    channel.close()
    Files.deleteIfExists(path)
end main
```

The writes target different offsets, so they do not depend on a shared  
current file position.  
Positioned writes are a natural fit for AIO because each request carries  
all the information needed to complete independently.  

## Cancelling an In-Flight Operation  
Use the `Future` returned by the channel API to request cancellation.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.AsynchronousFileChannel
import java.nio.file.{Files, StandardOpenOption}

@main def main() =
  val path = Files.createTempFile("aio-cancel", ".txt")
  Files.writeString(path, "cancel me" * 1000)

  val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
  val buffer = ByteBuffer.allocate(1024 * 16)

  try
    val operation = channel.read(buffer, 0L)
    val cancelled = operation.cancel(true)
    println(s"cancel requested: $cancelled")
  finally
    channel.close()
    Files.deleteIfExists(path)
end main
```

Cancellation is best effort, so a request may finish before the runtime  
can stop it.  
Even so, the returned future gives you a way to express time budgets and  
shutdown behaviour around asynchronous work.  

## Mixing Synchronous and Asynchronous File Operations  
Write a file synchronously and then read it back with AIO.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, StandardOpenOption}
import java.util.concurrent.CountDownLatch

@main def main() =
  val path = Files.createTempFile("aio-mixed", ".txt")
  Files.writeString(path, "written with Files.writeString")

  val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
  val buffer = ByteBuffer.allocate(64)
  val latch = CountDownLatch(1)

  try
    channel.read(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          buffer.flip()
          val bytes = Array.ofDim[Byte](buffer.remaining())
          buffer.get(bytes)
          println(new String(bytes, StandardCharsets.UTF_8))
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"mixed read failed: ${exc.getMessage}")
          latch.countDown()
    )

    latch.await()
  finally
    channel.close()
    Files.deleteIfExists(path)
end main
```

Many real programs mix styles because some APIs are synchronous and newer  
code paths are asynchronous.  
The important point is to keep ownership and cleanup clear when both kinds  
of operation touch the same resource.  

## Comparing Blocking vs Asynchronous File Read Latency  
Measure a simple blocking read and a simple AIO read on the same file.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.file.{Files, StandardOpenOption}
import java.util.concurrent.CountDownLatch

@main def main() =
  val path = Files.createTempFile("aio-latency", ".txt")
  Files.writeString(path, "latency demo" * 1000)

  val syncStart = System.nanoTime()
  Files.readAllBytes(path)
  val syncElapsed = System.nanoTime() - syncStart

  val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
  val buffer = ByteBuffer.allocate(Files.size(path).toInt)
  val latch = CountDownLatch(1)
  val asyncStart = System.nanoTime()

  try
    channel.read(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          val asyncElapsed = System.nanoTime() - asyncStart
          println(s"blocking ns: $syncElapsed")
          println(s"async ns: $asyncElapsed")
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"latency read failed: ${exc.getMessage}")
          latch.countDown()
    )

    latch.await()
  finally
    channel.close()
    Files.deleteIfExists(path)
end main
```

The absolute times from a tiny benchmark are not portable, but the shape  
of the code is the key lesson.  
Blocking time is paid directly by the caller, while async time is observed  
when the completion callback finally runs.  

## Asynchronous file IO with Futures  
These examples wrap callback-based file AIO in Scala futures.  

## Wrapping AsynchronousFileChannel Read in a Future  
Convert a file read callback into a composable `Future[String]`.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Path, StandardOpenOption}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Future, Promise}

@main def main() =
  def readText(path: Path): Future[String] =
    val promise = Promise[String]()
    val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
    val buffer = ByteBuffer.allocate(64)

    channel.read(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          try
            buffer.flip()
            val bytes = Array.ofDim[Byte](buffer.remaining())
            buffer.get(bytes)
            promise.success(new String(bytes, StandardCharsets.UTF_8))
          finally
            channel.close()

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally channel.close()
    )

    promise.future

  val path = Files.createTempFile("aio-future-read", ".txt")
  Files.writeString(path, "future read")

  readText(path).foreach(println)
  Thread.sleep(200)
  Files.deleteIfExists(path)
end main
```

The callback completes a `Promise`, and the promise exposes a `Future` to  
the rest of the program.  
Once the read is wrapped this way, callers can compose it with ordinary  
Scala future operators instead of manual callback nesting.  

## Wrapping AsynchronousFileChannel Write in a Future  
Represent an async write as a `Future[Int]` that reports bytes written.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Path, StandardOpenOption}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Future, Promise}

@main def main() =
  def writeText(path: Path, text: String): Future[Int] =
    val promise = Promise[Int]()
    val channel =
      AsynchronousFileChannel.open(
        path,
        StandardOpenOption.CREATE,
        StandardOpenOption.WRITE
      )
    val buffer = ByteBuffer.wrap(text.getBytes(StandardCharsets.UTF_8))

    channel.write(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          try promise.success(result)
          finally channel.close()

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally channel.close()
    )

    promise.future

  val path = Files.createTempFile("aio-future-write", ".txt")

  writeText(path, "future write").foreach: count =>
    println(s"bytes written: $count")
    println(Files.readString(path))

  Thread.sleep(200)
  Files.deleteIfExists(path)
end main
```

Returning a future makes the write result easy to observe, combine, or  
transform later.  
The example still uses a callback internally, but that detail is hidden  
behind a much friendlier Scala API.  

## Sequencing Async File Operations with for-Comprehensions  
Write a file and then read it back in a single future pipeline.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Path, StandardOpenOption}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Future, Promise}

@main def main() =
  def writeText(path: Path, text: String): Future[Int] =
    val promise = Promise[Int]()
    val channel =
      AsynchronousFileChannel.open(
        path,
        StandardOpenOption.CREATE,
        StandardOpenOption.WRITE
      )
    val buffer = ByteBuffer.wrap(text.getBytes(StandardCharsets.UTF_8))

    channel.write(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          try promise.success(result)
          finally channel.close()

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally channel.close()
    )

    promise.future

  def readText(path: Path): Future[String] =
    val promise = Promise[String]()
    val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
    val buffer = ByteBuffer.allocate(64)

    channel.read(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          try
            buffer.flip()
            val bytes = Array.ofDim[Byte](buffer.remaining())
            buffer.get(bytes)
            promise.success(new String(bytes, StandardCharsets.UTF_8))
          finally
            channel.close()

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally channel.close()
    )

    promise.future

  val path = Files.createTempFile("aio-seq", ".txt")

  val program =
    for
      _ <- writeText(path, "first write")
      text <- readText(path)
    yield text

  program.foreach(println)
  Thread.sleep(300)
  Files.deleteIfExists(path)
end main
```

A for-comprehension turns `flatMap` and `map` chains into a direct linear  
style.  
That makes dependent async steps easier to read, especially when cleanup  
and error handling must stay obvious.  

## Running Multiple Async File Reads in Parallel  
Launch several future-based reads and combine them with `Future.sequence`.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Path, StandardOpenOption}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Future, Promise}

@main def main() =
  def readText(path: Path): Future[String] =
    val promise = Promise[String]()
    val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
    val buffer = ByteBuffer.allocate(32)

    channel.read(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          try
            buffer.flip()
            val bytes = Array.ofDim[Byte](buffer.remaining())
            buffer.get(bytes)
            promise.success(new String(bytes, StandardCharsets.UTF_8))
          finally
            channel.close()

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally channel.close()
    )

    promise.future

  val paths =
    List("one", "two", "three").map: value =>
      val path = Files.createTempFile("aio-par", ".txt")
      Files.writeString(path, value)
      path

  Future.sequence(paths.map(readText)).foreach: texts =>
    println(texts.mkString(", "))

  Thread.sleep(300)
  paths.foreach(Files.deleteIfExists)
end main
```

`Future.sequence` starts from a list of futures and produces one future  
that completes when all of them finish.  
Because each file has its own channel and buffer, the reads can progress  
independently and complete in any order.  

## Aggregating Results from Multiple Async Operations  
Collect several async reads and reduce them to a single value.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Path, StandardOpenOption}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Future, Promise}

@main def main() =
  def readText(path: Path): Future[String] =
    val promise = Promise[String]()
    val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
    val buffer = ByteBuffer.allocate(32)

    channel.read(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          try
            buffer.flip()
            val bytes = Array.ofDim[Byte](buffer.remaining())
            buffer.get(bytes)
            promise.success(new String(bytes, StandardCharsets.UTF_8))
          finally
            channel.close()

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally channel.close()
    )

    promise.future

  val paths =
    List("4", "5", "6").map: value =>
      val path = Files.createTempFile("aio-agg", ".txt")
      Files.writeString(path, value)
      path

  val total =
    Future.sequence(paths.map(readText)).map: texts =>
      texts.map(_.toInt).sum

  total.foreach(sum => println(s"sum: $sum"))
  Thread.sleep(300)
  paths.foreach(Files.deleteIfExists)
end main
```

Aggregation is often the next step after running operations in parallel.  
Here the async reads produce strings, and a later future converts them to  
integers and sums them once every result is available.  

## Handling Errors in Async File Operations with Try and Either  
Turn failed async reads into values that can be inspected explicitly.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Path, StandardOpenOption}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Future, Promise}
import scala.util.{Failure, Success, Try}

@main def main() =
  def readText(path: Path): Future[String] =
    val promise = Promise[String]()
    val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
    val buffer = ByteBuffer.allocate(32)

    channel.read(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          try
            buffer.flip()
            val bytes = Array.ofDim[Byte](buffer.remaining())
            buffer.get(bytes)
            promise.success(new String(bytes, StandardCharsets.UTF_8))
          finally
            channel.close()

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally channel.close()
    )

    promise.future

  val path = Files.createTempFile("aio-try", ".txt")
  Files.writeString(path, "safe")

  val asTry: Future[Try[String]] = readText(path).map(Success(_)).recover:
    case ex => Failure(ex)

  val asEither = asTry.map(_.toEither.left.map(_.getMessage))

  asEither.foreach(result => println(result))
  Thread.sleep(300)
  Files.deleteIfExists(path)
end main
```

`Try` and `Either` are helpful when you want failure to stay in the data  
flow instead of throwing an exception later.  
That style is useful for reporting, retries, and validation steps that  
should keep running even when one operation fails.  

## Timeouts Around Async File Operations  
Race an async file read against a scheduled timeout future.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Path, StandardOpenOption}
import java.util.concurrent.Executors
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration.*
import scala.concurrent.{Await, Future, Promise}

@main def main() =
  def slowRead(path: Path): Future[String] =
    val promise = Promise[String]()
    val scheduler = Executors.newSingleThreadScheduledExecutor()
    val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
    val buffer = ByteBuffer.allocate(64)

    channel.read(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          scheduler.schedule(
            () =>
              try
                buffer.flip()
                val bytes = Array.ofDim[Byte](buffer.remaining())
                buffer.get(bytes)
                promise.success(new String(bytes, StandardCharsets.UTF_8))
              finally
                channel.close()
                scheduler.shutdown(),
            200,
            java.util.concurrent.TimeUnit.MILLISECONDS
          )

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally
            channel.close()
            scheduler.shutdown()
    )

    promise.future

  def withTimeout[A](future: Future[A], timeoutMs: Long): Future[A] =
    val promise = Promise[A]()
    val scheduler = Executors.newSingleThreadScheduledExecutor()
    scheduler.schedule(
      () =>
        promise.tryFailure(new RuntimeException("operation timed out"))
        scheduler.shutdown(),
      timeoutMs,
      java.util.concurrent.TimeUnit.MILLISECONDS
    )
    future.onComplete(result => promise.tryComplete(result))
    promise.future

  val path = Files.createTempFile("aio-timeout", ".txt")
  Files.writeString(path, "timeout demo")

  val result = withTimeout(slowRead(path), 100)
  println(Await.result(result.failed.map(_.getMessage), 1.second))
  Files.deleteIfExists(path)
end main
```

Timeouts protect a program from waiting forever when a completion signal  
never arrives or arrives too late to matter.  
The scheduled executor fails a promise after a deadline and leaves the  
caller with a normal Scala future to compose.  

## Using a Dedicated ExecutionContext for AIO  
Run future callbacks on a bounded execution context reserved for AIO work.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Path, StandardOpenOption}
import java.util.concurrent.Executors
import scala.concurrent.{ExecutionContext, Future, Promise}

@main def main() =
  val pool = Executors.newFixedThreadPool(2)
  given ExecutionContext = ExecutionContext.fromExecutor(pool)

  def readText(path: Path): Future[String] =
    val promise = Promise[String]()
    val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
    val buffer = ByteBuffer.allocate(64)

    channel.read(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          try
            buffer.flip()
            val bytes = Array.ofDim[Byte](buffer.remaining())
            buffer.get(bytes)
            promise.success(new String(bytes, StandardCharsets.UTF_8))
          finally
            channel.close()

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally channel.close()
    )

    promise.future

  val path = Files.createTempFile("aio-ec", ".txt")
  Files.writeString(path, "dedicated execution context")

  readText(path).foreach: text =>
    println(text)
    Files.deleteIfExists(path)
    pool.shutdown()

  Thread.sleep(300)
end main
```

A dedicated execution context keeps AIO callback work away from unrelated  
CPU-bound tasks.  
That separation improves predictability and makes it easier to tune the  
thread count for a specific subsystem.  

## Asynchronous socket IO basics  
These examples introduce async TCP clients with JVM socket channels.  

## Opening an AsynchronousSocketChannel  
Create an async socket channel without connecting it yet.  

```scala
import java.nio.channels.AsynchronousSocketChannel

@main def main() =
  val socket = AsynchronousSocketChannel.open()

  try
    println(s"socket open: ${socket.isOpen}")
  finally
    socket.close()
end main
```

Opening a socket channel is cheap and does not contact a remote peer yet.  
The channel becomes useful after a connect call or when it is accepted by  
an asynchronous server socket.  

## Connecting to a Remote Server Asynchronously  
Bind a local server and connect a client to it without blocking `main`.  

```scala
import java.net.InetSocketAddress
import java.nio.channels.
  {AsynchronousServerSocketChannel, AsynchronousSocketChannel, CompletionHandler}
import java.util.concurrent.CountDownLatch

@main def main() =
  val server = AsynchronousServerSocketChannel.open()
  server.bind(InetSocketAddress("127.0.0.1", 0))
  val address = server.getLocalAddress.asInstanceOf[InetSocketAddress]

  val client = AsynchronousSocketChannel.open()
  val latch = CountDownLatch(2)

  try
    server.accept(
      null,
      new CompletionHandler[AsynchronousSocketChannel, Null]:
        def completed(result: AsynchronousSocketChannel, attachment: Null) =
          println("server accepted connection")
          result.close()
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"accept failed: ${exc.getMessage}")
          latch.countDown()
    )

    client.connect(
      address,
      null,
      new CompletionHandler[Void, Null]:
        def completed(result: Void, attachment: Null) =
          println("client connected")
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"connect failed: ${exc.getMessage}")
          latch.countDown()
    )

    latch.await()
  finally
    client.close()
    server.close()
end main
```

The client connect starts immediately, and the server completes an async  
accept when the handshake finishes.  
A latch is useful here because two independent completion paths must run  
before the program can clean up both channels.  

## Reading from a Socket Asynchronously  
Accept a connection, send bytes from the server, and read them async.  

```scala
import java.net.InetSocketAddress
import java.nio.ByteBuffer
import java.nio.channels.
  {AsynchronousServerSocketChannel, AsynchronousSocketChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.util.concurrent.CountDownLatch

@main def main() =
  val server = AsynchronousServerSocketChannel.open()
  server.bind(InetSocketAddress("127.0.0.1", 0))
  val address = server.getLocalAddress.asInstanceOf[InetSocketAddress]

  val client = AsynchronousSocketChannel.open()
  val buffer = ByteBuffer.allocate(32)
  val latch = CountDownLatch(2)

  try
    server.accept(
      null,
      new CompletionHandler[AsynchronousSocketChannel, Null]:
        def completed(socket: AsynchronousSocketChannel, attachment: Null) =
          val out = ByteBuffer.wrap("hello".getBytes(StandardCharsets.UTF_8))
          socket.write(out).get()
          socket.close()
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"accept failed: ${exc.getMessage}")
          latch.countDown()
    )

    client.connect(address).get()
    client.read(
      buffer,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          buffer.flip()
          val bytes = Array.ofDim[Byte](buffer.remaining())
          buffer.get(bytes)
          println(new String(bytes, StandardCharsets.UTF_8))
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"socket read failed: ${exc.getMessage}")
          latch.countDown()
    )

    latch.await()
  finally
    client.close()
    server.close()
end main
```

The client read returns immediately, and the handler runs when bytes from  
the server arrive in the buffer.  
A socket read may complete with fewer bytes than expected, so protocols  
often need loops or framing rules on top of the raw channel API.  

## Writing to a Socket Asynchronously  
Connect to a local server and write request bytes with a callback.  

```scala
import java.net.InetSocketAddress
import java.nio.ByteBuffer
import java.nio.channels.
  {AsynchronousServerSocketChannel, AsynchronousSocketChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.util.concurrent.CountDownLatch

@main def main() =
  val server = AsynchronousServerSocketChannel.open()
  server.bind(InetSocketAddress("127.0.0.1", 0))
  val address = server.getLocalAddress.asInstanceOf[InetSocketAddress]

  val client = AsynchronousSocketChannel.open()
  val latch = CountDownLatch(2)

  try
    server.accept(
      null,
      new CompletionHandler[AsynchronousSocketChannel, Null]:
        def completed(socket: AsynchronousSocketChannel, attachment: Null) =
          val buffer = ByteBuffer.allocate(32)
          socket.read(buffer).get()
          buffer.flip()
          val bytes = Array.ofDim[Byte](buffer.remaining())
          buffer.get(bytes)
          println(new String(bytes, StandardCharsets.UTF_8))
          socket.close()
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"accept failed: ${exc.getMessage}")
          latch.countDown()
    )

    client.connect(address).get()
    val out = ByteBuffer.wrap("ping".getBytes(StandardCharsets.UTF_8))
    client.write(
      out,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          println(s"client wrote: $result")
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"socket write failed: ${exc.getMessage}")
          latch.countDown()
    )

    latch.await()
  finally
    client.close()
    server.close()
end main
```

The client write only reports how many bytes the operating system accepted  
for that call.  
For longer messages, a real implementation should continue writing until  
all bytes in the buffer have been consumed.  

## Handling Connection Failures  
Observe the failure path when an async connect cannot reach a server.  

```scala
import java.net.InetSocketAddress
import java.nio.channels.{AsynchronousSocketChannel, CompletionHandler}
import java.util.concurrent.CountDownLatch

@main def main() =
  val socket = AsynchronousSocketChannel.open()
  val latch = CountDownLatch(1)
  val target = InetSocketAddress("127.0.0.1", 65000)

  try
    socket.connect(
      target,
      null,
      new CompletionHandler[Void, Null]:
        def completed(result: Void, attachment: Null) =
          println("unexpected success")
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"connect failed: ${exc.getClass.getSimpleName}")
          latch.countDown()
    )

    latch.await()
  finally
    socket.close()
end main
```

Connection attempts can fail because no server is listening, because the  
network is down, or because a firewall rejects the traffic.  
Async code must treat the failure callback as a normal part of the socket  
lifecycle instead of an exceptional edge case.  

## Closing Sockets Safely in Async Code  
Use a small helper so repeated close logic stays reliable and concise.  

```scala
import java.net.InetSocketAddress
import java.nio.channels.
  {AsynchronousServerSocketChannel, AsynchronousSocketChannel, CompletionHandler}
import java.util.concurrent.CountDownLatch

@main def main() =
  def closeQuietly(channel: AsynchronousSocketChannel): Unit =
    if channel != null && channel.isOpen then channel.close()

  val server = AsynchronousServerSocketChannel.open()
  server.bind(InetSocketAddress("127.0.0.1", 0))
  val address = server.getLocalAddress.asInstanceOf[InetSocketAddress]

  val client = AsynchronousSocketChannel.open()
  val latch = CountDownLatch(2)

  try
    server.accept(
      null,
      new CompletionHandler[AsynchronousSocketChannel, Null]:
        def completed(socket: AsynchronousSocketChannel, attachment: Null) =
          println("accepted client")
          closeQuietly(socket)
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"accept failed: ${exc.getMessage}")
          latch.countDown()
    )

    client.connect(
      address,
      null,
      new CompletionHandler[Void, Null]:
        def completed(result: Void, attachment: Null) =
          println("connected")
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"connect failed: ${exc.getMessage}")
          latch.countDown()
    )

    latch.await()
  finally
    closeQuietly(client)
    server.close()
end main
```

Safe close helpers are useful when success and failure paths both need to  
release the same resources.  
This keeps cleanup logic short and reduces the chance of leaking sockets  
when a callback exits through an error path.  

## Configuring Socket Options  
Set common TCP options before using an async socket channel.  

```scala
import java.net.InetSocketAddress
import java.net.StandardSocketOptions
import java.nio.channels.AsynchronousSocketChannel

@main def main() =
  val socket = AsynchronousSocketChannel.open()

  try
    socket.setOption(StandardSocketOptions.SO_KEEPALIVE, true)
    socket.setOption(StandardSocketOptions.TCP_NODELAY, true)
    socket.setOption(StandardSocketOptions.SO_REUSEADDR, true)

    println(socket.getOption(StandardSocketOptions.SO_KEEPALIVE))
    println(socket.getOption(StandardSocketOptions.TCP_NODELAY))
    println(socket.getOption(StandardSocketOptions.SO_REUSEADDR))
  finally
    socket.close()
end main
```

Socket options affect behaviour such as packet coalescing, idle probing,  
and address reuse.  
It is best to set them before the channel enters the hot path so each new  
connection starts with predictable network settings.  

## Asynchronous server sockets  
These examples build async servers on top of accept callbacks.  

## Opening an AsynchronousServerSocketChannel  
Open a server socket, bind it, and inspect its local address.  

```scala
import java.net.InetSocketAddress
import java.nio.channels.AsynchronousServerSocketChannel

@main def main() =
  val server = AsynchronousServerSocketChannel.open()

  try
    server.bind(InetSocketAddress("127.0.0.1", 0))
    println(server.getLocalAddress)
  finally
    server.close()
end main
```

An asynchronous server socket starts with the same lifecycle as a classic  
server: open, bind, accept, and close.  
The main difference is that incoming connections are reported through a  
completion callback instead of a blocking `accept` call.  

## Accepting Connections Asynchronously  
Register an accept callback and connect one local client to trigger it.  

```scala
import java.net.InetSocketAddress
import java.nio.channels.
  {AsynchronousServerSocketChannel, AsynchronousSocketChannel, CompletionHandler}
import java.util.concurrent.CountDownLatch

@main def main() =
  val server = AsynchronousServerSocketChannel.open()
  server.bind(InetSocketAddress("127.0.0.1", 0))
  val address = server.getLocalAddress.asInstanceOf[InetSocketAddress]

  val client = AsynchronousSocketChannel.open()
  val latch = CountDownLatch(2)

  try
    server.accept(
      null,
      new CompletionHandler[AsynchronousSocketChannel, Null]:
        def completed(socket: AsynchronousSocketChannel, attachment: Null) =
          println("accepted asynchronously")
          socket.close()
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"accept failed: ${exc.getMessage}")
          latch.countDown()
    )

    client.connect(address).get()
    latch.countDown()
    latch.await()
  finally
    client.close()
    server.close()
end main
```

After each successful accept, a real server usually posts another accept  
so it remains ready for the next client.  
This small demo handles one client only, which keeps the basic callback  
pattern easy to see.  

## Handling Multiple Clients Concurrently  
Re-arm `accept` so several clients can connect during one server run.  

```scala
import java.net.InetSocketAddress
import java.nio.channels.
  {AsynchronousServerSocketChannel, AsynchronousSocketChannel, CompletionHandler}
import java.util.concurrent.CountDownLatch
import java.util.concurrent.atomic.AtomicInteger

@main def main() =
  val server = AsynchronousServerSocketChannel.open()
  server.bind(InetSocketAddress("127.0.0.1", 0))
  val address = server.getLocalAddress.asInstanceOf[InetSocketAddress]
  val accepted = AtomicInteger(0)
  val latch = CountDownLatch(3)

  def acceptNext(): Unit =
    server.accept(
      null,
      new CompletionHandler[AsynchronousSocketChannel, Null]:
        def completed(socket: AsynchronousSocketChannel, attachment: Null) =
          val count = accepted.incrementAndGet()
          println(s"accepted client $count")
          socket.close()
          latch.countDown()
          if count < 3 then acceptNext()

        def failed(exc: Throwable, attachment: Null) =
          println(s"accept failed: ${exc.getMessage}")
    )

  try
    acceptNext()
    val clients = List.fill(3)(AsynchronousSocketChannel.open())
    clients.foreach(_.connect(address).get())
    latch.await()
    clients.foreach(_.close())
  finally
    server.close()
end main
```

The server is concurrent because it does not wait synchronously for one  
client before becoming ready for the next.  
Each accepted socket can now continue with its own async read or write  
workflow while the server keeps accepting more peers.  

## Per-Connection Read and Write Loops with CompletionHandler  
Create recursive callbacks that keep reading and echoing one connection.  

```scala
import java.net.InetSocketAddress
import java.nio.ByteBuffer
import java.nio.channels.
  {AsynchronousServerSocketChannel, AsynchronousSocketChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.util.concurrent.CountDownLatch

@main def main() =
  val server = AsynchronousServerSocketChannel.open()
  server.bind(InetSocketAddress("127.0.0.1", 0))
  val address = server.getLocalAddress.asInstanceOf[InetSocketAddress]
  val client = AsynchronousSocketChannel.open()
  val latch = CountDownLatch(2)

  def startEchoLoop(socket: AsynchronousSocketChannel): Unit =
    val buffer = ByteBuffer.allocate(32)
    socket.read(
      buffer,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          if result <= 0 then
            socket.close()
            latch.countDown()
          else
            buffer.flip()
            socket.write(
              buffer,
              null,
              new CompletionHandler[Integer, Null]:
                def completed(written: Integer, attachment: Null) =
                  startEchoLoop(socket)

                def failed(exc: Throwable, attachment: Null) =
                  println(s"write loop failed: ${exc.getMessage}")
                  socket.close()
                  latch.countDown()
            )

        def failed(exc: Throwable, attachment: Null) =
          println(s"read loop failed: ${exc.getMessage}")
          socket.close()
          latch.countDown()
    )

  try
    server.accept(
      null,
      new CompletionHandler[AsynchronousSocketChannel, Null]:
        def completed(socket: AsynchronousSocketChannel, attachment: Null) =
          startEchoLoop(socket)

        def failed(exc: Throwable, attachment: Null) =
          println(s"accept failed: ${exc.getMessage}")
    )

    client.connect(address).get()
    client.write(ByteBuffer.wrap("hi".getBytes(StandardCharsets.UTF_8))).get()
    val response = ByteBuffer.allocate(8)
    client.read(response).get()
    response.flip()
    val bytes = Array.ofDim[Byte](response.remaining())
    response.get(bytes)
    println(new String(bytes, StandardCharsets.UTF_8))
    latch.countDown()
    client.close()
    latch.await()
  finally
    server.close()
end main
```

Per-connection loops are common in servers because reads and writes rarely  
finish all the work in one callback.  
Recursive handlers let one socket keep processing data without blocking  
the accept path for other clients.  

## Echo Server Using Asynchronous Sockets  
Return the same bytes to the client after reading one request.  

```scala
import java.net.InetSocketAddress
import java.nio.ByteBuffer
import java.nio.channels.
  {AsynchronousServerSocketChannel, AsynchronousSocketChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.util.concurrent.CountDownLatch

@main def main() =
  val server = AsynchronousServerSocketChannel.open()
  server.bind(InetSocketAddress("127.0.0.1", 0))
  val address = server.getLocalAddress.asInstanceOf[InetSocketAddress]
  val client = AsynchronousSocketChannel.open()
  val latch = CountDownLatch(1)

  try
    server.accept(
      null,
      new CompletionHandler[AsynchronousSocketChannel, Null]:
        def completed(socket: AsynchronousSocketChannel, attachment: Null) =
          val buffer = ByteBuffer.allocate(32)
          socket.read(
            buffer,
            null,
            new CompletionHandler[Integer, Null]:
              def completed(result: Integer, attachment: Null) =
                buffer.flip()
                socket.write(buffer).get()
                socket.close()
                latch.countDown()

              def failed(exc: Throwable, attachment: Null) =
                println(s"echo read failed: ${exc.getMessage}")
                socket.close()
                latch.countDown()
          )

        def failed(exc: Throwable, attachment: Null) =
          println(s"accept failed: ${exc.getMessage}")
          latch.countDown()
    )

    client.connect(address).get()
    client.write(ByteBuffer.wrap("echo".getBytes(StandardCharsets.UTF_8))).get()
    val reply = ByteBuffer.allocate(32)
    client.read(reply).get()
    reply.flip()
    val bytes = Array.ofDim[Byte](reply.remaining())
    reply.get(bytes)
    println(new String(bytes, StandardCharsets.UTF_8))
    latch.await()
  finally
    client.close()
    server.close()
end main
```

An echo server is a small but complete protocol: request bytes arrive and  
response bytes leave on the same connection.  
It is a useful starting point because the data path is simple and easy to  
inspect when learning async socket APIs.  

## Simple Line-Based Protocol over Async Sockets  
Exchange newline-delimited messages on a local async TCP connection.  

```scala
import java.net.InetSocketAddress
import java.nio.ByteBuffer
import java.nio.channels.
  {AsynchronousServerSocketChannel, AsynchronousSocketChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.util.concurrent.CountDownLatch

@main def main() =
  val server = AsynchronousServerSocketChannel.open()
  server.bind(InetSocketAddress("127.0.0.1", 0))
  val address = server.getLocalAddress.asInstanceOf[InetSocketAddress]
  val client = AsynchronousSocketChannel.open()
  val latch = CountDownLatch(1)

  try
    server.accept(
      null,
      new CompletionHandler[AsynchronousSocketChannel, Null]:
        def completed(socket: AsynchronousSocketChannel, attachment: Null) =
          val in = ByteBuffer.allocate(64)
          socket.read(in).get()
          in.flip()
          val bytes = Array.ofDim[Byte](in.remaining())
          in.get(bytes)
          val line = new String(bytes, StandardCharsets.UTF_8).trim
          val reply = s"pong for $line\n"
          socket.write(ByteBuffer.wrap(reply.getBytes(StandardCharsets.UTF_8))).get()
          socket.close()
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"protocol failed: ${exc.getMessage}")
          latch.countDown()
    )

    client.connect(address).get()
    val request = ByteBuffer.wrap("ping\n".getBytes(StandardCharsets.UTF_8))
    client.write(request).get()
    val response = ByteBuffer.allocate(64)
    client.read(response).get()
    response.flip()
    val bytes = Array.ofDim[Byte](response.remaining())
    response.get(bytes)
    println(new String(bytes, StandardCharsets.UTF_8).trim)
    latch.await()
  finally
    client.close()
    server.close()
end main
```

Line-based protocols are handy for demos because a newline gives a simple  
message boundary.  
Real protocols often use the same idea with text commands, length fields,  
or other framing rules layered above the byte stream.  

## Graceful Shutdown of an Async Server  
Stop accepting new clients and then close the listening socket cleanly.  

```scala
import java.net.InetSocketAddress
import java.nio.channels.
  {AsynchronousServerSocketChannel, AsynchronousSocketChannel, CompletionHandler}
import java.util.concurrent.{CountDownLatch, Executors}
import java.util.concurrent.atomic.AtomicBoolean

@main def main() =
  val server = AsynchronousServerSocketChannel.open()
  server.bind(InetSocketAddress("127.0.0.1", 0))
  val address = server.getLocalAddress.asInstanceOf[InetSocketAddress]
  val running = AtomicBoolean(true)
  val latch = CountDownLatch(1)
  val scheduler = Executors.newSingleThreadScheduledExecutor()

  def acceptNext(): Unit =
    if running.get() then
      server.accept(
        null,
        new CompletionHandler[AsynchronousSocketChannel, Null]:
          def completed(socket: AsynchronousSocketChannel, attachment: Null) =
            socket.close()
            if running.get() then acceptNext()

          def failed(exc: Throwable, attachment: Null) =
            println(s"server stopped: ${exc.getClass.getSimpleName}")
            latch.countDown()
      )

  try
    acceptNext()
    scheduler.schedule(
      () =>
        running.set(false)
        server.close(),
      200,
      java.util.concurrent.TimeUnit.MILLISECONDS
    )
    val client = AsynchronousSocketChannel.open()
    client.connect(address).get()
    client.close()
    latch.await()
  finally
    scheduler.shutdown()
    if server.isOpen then server.close()
end main
```

Graceful shutdown means the server stops taking new work before the final  
resource cleanup happens.  
This approach reduces surprises for callers and gives existing operations  
a chance to finish in a controlled way.  

## AIO and Scala concurrency  
These examples connect low-level AIO with Scala concurrency primitives.  

## Integrating AIO with Scala Future  
Wrap an async socket connect in a future and react to its completion.  

```scala
import java.net.InetSocketAddress
import java.nio.channels.
  {AsynchronousServerSocketChannel, AsynchronousSocketChannel, CompletionHandler}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Future, Promise}

@main def main() =
  def connectAsync(
      socket: AsynchronousSocketChannel,
      address: InetSocketAddress
  ): Future[Unit] =
    val promise = Promise[Unit]()
    socket.connect(
      address,
      null,
      new CompletionHandler[Void, Null]:
        def completed(result: Void, attachment: Null) =
          promise.success(())

        def failed(exc: Throwable, attachment: Null) =
          promise.failure(exc)
    )
    promise.future

  val server = AsynchronousServerSocketChannel.open()
  server.bind(InetSocketAddress("127.0.0.1", 0))
  val address = server.getLocalAddress.asInstanceOf[InetSocketAddress]
  val client = AsynchronousSocketChannel.open()

  try
    val accepted = server.accept()
    connectAsync(client, address).foreach(_ => println("future connected"))
    accepted.get().close()
    Thread.sleep(200)
  finally
    client.close()
    server.close()
end main
```

Once a socket operation becomes a future, it can participate in normal  
Scala concurrency workflows.  
That means one integration step unlocks retries, timeouts, sequencing,  
and parallel composition with the rest of the standard library.  

## Integrating AIO with Promise  
Complete a promise directly from an AIO callback.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, StandardOpenOption}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.Promise

@main def main() =
  val path = Files.createTempFile("aio-promise", ".txt")
  Files.writeString(path, "promise bridge")
  val promise = Promise[String]()
  val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
  val buffer = ByteBuffer.allocate(64)

  channel.read(
    buffer,
    0L,
    null,
    new CompletionHandler[Integer, Null]:
      def completed(result: Integer, attachment: Null) =
        try
          buffer.flip()
          val bytes = Array.ofDim[Byte](buffer.remaining())
          buffer.get(bytes)
          promise.success(new String(bytes, StandardCharsets.UTF_8))
        finally
          channel.close()

      def failed(exc: Throwable, attachment: Null) =
        try promise.failure(exc)
        finally channel.close()
  )

  promise.future.foreach(println)
  Thread.sleep(200)
  Files.deleteIfExists(path)
end main
```

A promise is the minimal bridge between callback-based APIs and future-  
based programs.  
You manually complete it from the handler, and every later consumer sees  
a standard future interface instead of a raw callback contract.  

## Composing Async File and Network Operations  
Read a file asynchronously and send its content over a local async socket.  

```scala
import java.net.InetSocketAddress
import java.nio.ByteBuffer
import java.nio.channels.
  {
    AsynchronousFileChannel,
    AsynchronousServerSocketChannel,
    AsynchronousSocketChannel,
    CompletionHandler
  }
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Path, StandardOpenOption}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Future, Promise}

@main def main() =
  def readText(path: Path): Future[String] =
    val promise = Promise[String]()
    val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
    val buffer = ByteBuffer.allocate(64)
    channel.read(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          try
            buffer.flip()
            val bytes = Array.ofDim[Byte](buffer.remaining())
            buffer.get(bytes)
            promise.success(new String(bytes, StandardCharsets.UTF_8))
          finally channel.close()

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally channel.close()
    )
    promise.future

  def writeSocket(
      socket: AsynchronousSocketChannel,
      text: String
  ): Future[Int] =
    val promise = Promise[Int]()
    val buffer = ByteBuffer.wrap(text.getBytes(StandardCharsets.UTF_8))
    socket.write(
      buffer,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          promise.success(result)

        def failed(exc: Throwable, attachment: Null) =
          promise.failure(exc)
    )
    promise.future

  val path = Files.createTempFile("aio-compose", ".txt")
  Files.writeString(path, "hello over async TCP")

  val server = AsynchronousServerSocketChannel.open()
  server.bind(InetSocketAddress("127.0.0.1", 0))
  val address = server.getLocalAddress.asInstanceOf[InetSocketAddress]
  val client = AsynchronousSocketChannel.open()

  try
    val accepted = server.accept()
    client.connect(address).get()
    val serverSide = accepted.get()
    val program =
      for
        text <- readText(path)
        count <- writeSocket(client, text)
      yield count

    program.foreach(count => println(s"sent bytes: $count"))
    Thread.sleep(300)
    serverSide.close()
  finally
    client.close()
    server.close()
    Files.deleteIfExists(path)
end main
```

This example shows why futures are useful for AIO: they make unrelated  
resources feel uniform.  
A file read and a socket write become ordinary steps in one pipeline even  
though each started from a different callback API.  

## Back-Pressure Strategies  
Limit concurrent async work with a semaphore so producers cannot run wild.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Path, StandardOpenOption}
import java.util.concurrent.Semaphore
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Future, Promise}

@main def main() =
  val permits = Semaphore(2)

  def readText(path: Path): Future[String] =
    permits.acquire()
    val promise = Promise[String]()
    val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
    val buffer = ByteBuffer.allocate(32)

    channel.read(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          try
            buffer.flip()
            val bytes = Array.ofDim[Byte](buffer.remaining())
            buffer.get(bytes)
            promise.success(new String(bytes, StandardCharsets.UTF_8))
          finally
            channel.close()
            permits.release()

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally
            channel.close()
            permits.release()
    )

    promise.future

  val paths =
    List("a", "b", "c", "d").map: value =>
      val path = Files.createTempFile("aio-pressure", ".txt")
      Files.writeString(path, value)
      path

  Future.sequence(paths.map(readText)).foreach(values => println(values.mkString))
  Thread.sleep(400)
  paths.foreach(Files.deleteIfExists)
end main
```

Back-pressure means the consumer side can slow the producer side down when  
resources are limited.  
A semaphore is a small but effective tool here because it caps the number  
of active operations without changing the callback logic itself.  

## Using Bounded Thread Pools for AIO  
Combine AIO with a fixed-size execution context for predictable load.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Path, StandardOpenOption}
import java.util.concurrent.Executors
import scala.concurrent.{ExecutionContext, Future, Promise}

@main def main() =
  val executor = Executors.newFixedThreadPool(2)
  given ExecutionContext = ExecutionContext.fromExecutor(executor)

  def readText(path: Path): Future[String] =
    val promise = Promise[String]()
    val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
    val buffer = ByteBuffer.allocate(32)
    channel.read(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          try
            buffer.flip()
            val bytes = Array.ofDim[Byte](buffer.remaining())
            buffer.get(bytes)
            promise.success(new String(bytes, StandardCharsets.UTF_8))
          finally channel.close()

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally channel.close()
    )
    promise.future

  val paths =
    List("one", "two").map: value =>
      val path = Files.createTempFile("aio-bounded", ".txt")
      Files.writeString(path, value)
      path

  Future.sequence(paths.map(readText)).foreach(values => println(values.mkString(",")))
  Thread.sleep(300)
  paths.foreach(Files.deleteIfExists)
  executor.shutdown()
end main
```

Bounded pools keep a subsystem from creating more runnable work than the  
process can handle efficiently.  
That matters for callback-heavy designs because futures can schedule extra  
continuations even when the underlying IO is already asynchronous.  

## Avoiding Blocking Calls in Async Callbacks  
Move slow follow-up work away from the completion handler thread.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, StandardOpenOption}
import java.util.concurrent.Executors

@main def main() =
  val cpuPool = Executors.newSingleThreadExecutor()
  val path = Files.createTempFile("aio-callbacks", ".txt")
  Files.writeString(path, "dispatch work")
  val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
  val buffer = ByteBuffer.allocate(64)

  channel.read(
    buffer,
    0L,
    null,
    new CompletionHandler[Integer, Null]:
      def completed(result: Integer, attachment: Null) =
        cpuPool.execute: () =>
          buffer.flip()
          val bytes = Array.ofDim[Byte](buffer.remaining())
          buffer.get(bytes)
          println(new String(bytes, StandardCharsets.UTF_8).toUpperCase)
          channel.close()

      def failed(exc: Throwable, attachment: Null) =
        println(s"callback failed: ${exc.getMessage}")
        channel.close()
  )

  Thread.sleep(300)
  cpuPool.shutdown()
  Files.deleteIfExists(path)
end main
```

Async callbacks should stay short because they often run on shared AIO or  
executor threads.  
Dispatching heavier work elsewhere keeps those threads responsive and lets  
other completions be processed promptly.  

## High-latency and high-concurrency patterns  
These examples focus on scaling patterns rather than single operations.  

## Running Many Small Async File Reads Concurrently  
Start many tiny file reads and collect them with one future.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Path, StandardOpenOption}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Future, Promise}

@main def main() =
  def readText(path: Path): Future[String] =
    val promise = Promise[String]()
    val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
    val buffer = ByteBuffer.allocate(8)
    channel.read(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          try
            buffer.flip()
            val bytes = Array.ofDim[Byte](buffer.remaining())
            buffer.get(bytes)
            promise.success(new String(bytes, StandardCharsets.UTF_8))
          finally channel.close()

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally channel.close()
    )
    promise.future

  val paths =
    (1 to 10).toList.map: i =>
      val path = Files.createTempFile("aio-many-read", ".txt")
      Files.writeString(path, s"$i")
      path

  Future.sequence(paths.map(readText)).foreach(values => println(values.mkString("|")))
  Thread.sleep(400)
  paths.foreach(Files.deleteIfExists)
end main
```

This is a typical high-concurrency pattern: many operations are tiny, but  
any one of them may still stall on latency.  
Async APIs let the program keep many requests in flight without blocking a  
separate thread for each one.  

## Running Many Small Async Socket Operations Concurrently  
Accept several clients and let each one perform a short async exchange.  

```scala
import java.net.InetSocketAddress
import java.nio.ByteBuffer
import java.nio.channels.
  {AsynchronousServerSocketChannel, AsynchronousSocketChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.util.concurrent.CountDownLatch

@main def main() =
  val server = AsynchronousServerSocketChannel.open()
  server.bind(InetSocketAddress("127.0.0.1", 0))
  val address = server.getLocalAddress.asInstanceOf[InetSocketAddress]
  val latch = CountDownLatch(5)

  def acceptNext(remaining: Int): Unit =
    if remaining > 0 then
      server.accept(
        null,
        new CompletionHandler[AsynchronousSocketChannel, Null]:
          def completed(socket: AsynchronousSocketChannel, attachment: Null) =
            val out = ByteBuffer.wrap("ok".getBytes(StandardCharsets.UTF_8))
            socket.write(out).get()
            socket.close()
            latch.countDown()
            acceptNext(remaining - 1)

          def failed(exc: Throwable, attachment: Null) =
            println(s"accept failed: ${exc.getMessage}")
      )

  try
    acceptNext(5)
    val clients = List.fill(5)(AsynchronousSocketChannel.open())
    clients.foreach: client =>
      client.connect(address).get()
      val buffer = ByteBuffer.allocate(8)
      client.read(buffer).get()
      client.close()
    latch.await()
  finally
    server.close()
end main
```

Socket-heavy systems often spend most of their time waiting for peers to  
send or receive small messages.  
Async channels help here because each connection can remain active without  
consuming a permanently blocked application thread.  

## Measuring Throughput and Latency of Async Operations  
Record how long a batch of async reads takes from start to finish.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Path, StandardOpenOption}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Future, Promise}

@main def main() =
  def readText(path: Path): Future[String] =
    val promise = Promise[String]()
    val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
    val buffer = ByteBuffer.allocate(16)
    channel.read(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          try
            buffer.flip()
            val bytes = Array.ofDim[Byte](buffer.remaining())
            buffer.get(bytes)
            promise.success(new String(bytes, StandardCharsets.UTF_8))
          finally channel.close()

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally channel.close()
    )
    promise.future

  val paths =
    (1 to 5).toList.map: i =>
      val path = Files.createTempFile("aio-metric", ".txt")
      Files.writeString(path, s"item-$i")
      path

  val start = System.nanoTime()
  Future.sequence(paths.map(readText)).foreach: values =>
    val elapsed = System.nanoTime() - start
    println(s"count: ${values.size}")
    println(s"elapsed ns: $elapsed")
    println(s"avg ns: ${elapsed / values.size}")

  Thread.sleep(400)
  paths.foreach(Files.deleteIfExists)
end main
```

Throughput asks how much work completes in a time window, while latency  
asks how long one request takes.  
Both matter for AIO, because a design can complete many operations overall  
while still giving poor response time to individual requests.  

## Comparing AIO with Non-Blocking NIO and Selector  
Show the setup difference between callback AIO and readiness-based NIO.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, Pipe, Selector}
import java.nio.file.{Files, StandardOpenOption}

@main def main() =
  val selector = Selector.open()
  val pipe = Pipe.open()
  val source = pipe.source()
  source.configureBlocking(false)
  source.register(selector, source.validOps())

  val path = Files.createTempFile("aio-vs-nio", ".txt")
  val aio = AsynchronousFileChannel.open(path, StandardOpenOption.READ)

  try
    println(s"selector keys: ${selector.keys().size()}")
    println(s"aio open: ${aio.isOpen}")
  finally
    aio.close()
    source.close()
    pipe.sink().close()
    selector.close()
    Files.deleteIfExists(path)
end main
```

Selector-based NIO tells you when a channel is ready, but your code still  
performs the actual read or write step.  
AIO reports operation completion instead, which can simplify some designs  
at the cost of a more callback-oriented API.  

## Choosing Between AIO, NIO, and Classic IO  
Express a simple decision rule for three common JVM IO styles.  

```scala
@main def main() =
  enum Workload:
    case SmallScript, ManySockets, SelectorProtocol

  def choose(workload: Workload): String =
    workload match
      case Workload.SmallScript => "classic blocking IO"
      case Workload.ManySockets => "AIO or Future-based async IO"
      case Workload.SelectorProtocol => "non-blocking NIO with Selector"

  println(choose(Workload.SmallScript))
  println(choose(Workload.ManySockets))
  println(choose(Workload.SelectorProtocol))
end main
```

Classic IO is often the best choice for small utilities and one-off batch  
programs because the code is shorter.  
AIO is attractive when latency and concurrency are both high, while NIO  
selectors remain useful for explicit event loops and custom protocols.  

## Designing an Async Pipeline  
Chain read, transform, and write steps into one async data pipeline.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Path, StandardOpenOption}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Future, Promise}

@main def main() =
  def readText(path: Path): Future[String] =
    val promise = Promise[String]()
    val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
    val buffer = ByteBuffer.allocate(64)
    channel.read(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          try
            buffer.flip()
            val bytes = Array.ofDim[Byte](buffer.remaining())
            buffer.get(bytes)
            promise.success(new String(bytes, StandardCharsets.UTF_8))
          finally channel.close()

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally channel.close()
    )
    promise.future

  def writeText(path: Path, text: String): Future[Int] =
    val promise = Promise[Int]()
    val channel =
      AsynchronousFileChannel.open(
        path,
        StandardOpenOption.CREATE,
        StandardOpenOption.WRITE,
        StandardOpenOption.TRUNCATE_EXISTING
      )
    val buffer = ByteBuffer.wrap(text.getBytes(StandardCharsets.UTF_8))
    channel.write(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          try promise.success(result)
          finally channel.close()

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally channel.close()
    )
    promise.future

  val source = Files.createTempFile("aio-pipe-in", ".txt")
  val target = Files.createTempFile("aio-pipe-out", ".txt")
  Files.writeString(source, "pipeline")

  val pipeline =
    for
      text <- readText(source)
      upper = text.toUpperCase
      _ <- writeText(target, upper)
      done <- readText(target)
    yield done

  pipeline.foreach(println)
  Thread.sleep(400)
  Files.deleteIfExists(source)
  Files.deleteIfExists(target)
end main
```

Pipelines are a natural way to think about async programs because each  
stage produces input for the next one.  
Using futures keeps the stages explicit and makes it easy to add logging,  
timeouts, or retries around individual steps later.  

## Practical use cases  
These examples show patterns you can adapt in real applications.  

## Async Log Writer  
Append log messages to a file with async writes and tracked positions.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, StandardOpenOption}
import java.util.concurrent.CountDownLatch
import java.util.concurrent.atomic.AtomicLong

@main def main() =
  val path = Files.createTempFile("aio-log", ".log")
  val channel =
    AsynchronousFileChannel.open(
      path,
      StandardOpenOption.CREATE,
      StandardOpenOption.WRITE,
      StandardOpenOption.READ
    )
  val position = AtomicLong(0L)
  val latch = CountDownLatch(2)

  def append(line: String): Unit =
    val bytes = s"$line\n".getBytes(StandardCharsets.UTF_8)
    val start = position.getAndAdd(bytes.length.toLong)
    val buffer = ByteBuffer.wrap(bytes)
    channel.write(
      buffer,
      start,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) = latch.countDown()
        def failed(exc: Throwable, attachment: Null) =
          println(s"log write failed: ${exc.getMessage}")
          latch.countDown()
    )

  try
    append("INFO startup complete")
    append("WARN cache miss")
    latch.await()
    println(Files.readString(path))
  finally
    channel.close()
    Files.deleteIfExists(path)
end main
```

Log writers are a good match for positioned async appends because each new  
entry can reserve a file region up front.  
The atomic position counter preserves ordering without forcing the caller  
to block on every write completion.  

## Async File Copier  
Read source bytes asynchronously and write them to a target file.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.file.{Files, StandardOpenOption}
import java.util.concurrent.CountDownLatch

@main def main() =
  val source = Files.createTempFile("aio-copy-src", ".txt")
  val target = Files.createTempFile("aio-copy-dst", ".txt")
  Files.writeString(source, "copy this text")

  val src = AsynchronousFileChannel.open(source, StandardOpenOption.READ)
  val dst =
    AsynchronousFileChannel.open(
      target,
      StandardOpenOption.WRITE,
      StandardOpenOption.TRUNCATE_EXISTING
    )
  val buffer = ByteBuffer.allocate(64)
  val latch = CountDownLatch(1)

  try
    src.read(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          buffer.flip()
          dst.write(
            buffer,
            0L,
            null,
            new CompletionHandler[Integer, Null]:
              def completed(written: Integer, attachment: Null) =
                println(Files.readString(target))
                latch.countDown()

              def failed(exc: Throwable, attachment: Null) =
                println(s"copy write failed: ${exc.getMessage}")
                latch.countDown()
          )

        def failed(exc: Throwable, attachment: Null) =
          println(s"copy read failed: ${exc.getMessage}")
          latch.countDown()
    )

    latch.await()
  finally
    src.close()
    dst.close()
    Files.deleteIfExists(source)
    Files.deleteIfExists(target)
end main
```

A copier has two phases: pull bytes from the source and push them to the  
destination.  
Even in this tiny example, AIO makes the handoff between those phases very  
clear because each step begins in the prior completion callback.  

## Async File Downloader over HTTP Using Async Sockets  
Send a simple HTTP request to a local async server and read the response.  

```scala
import java.net.InetSocketAddress
import java.nio.ByteBuffer
import java.nio.channels.
  {AsynchronousServerSocketChannel, AsynchronousSocketChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.util.concurrent.CountDownLatch

@main def main() =
  val server = AsynchronousServerSocketChannel.open()
  server.bind(InetSocketAddress("127.0.0.1", 0))
  val address = server.getLocalAddress.asInstanceOf[InetSocketAddress]
  val client = AsynchronousSocketChannel.open()
  val latch = CountDownLatch(1)

  try
    server.accept(
      null,
      new CompletionHandler[AsynchronousSocketChannel, Null]:
        def completed(socket: AsynchronousSocketChannel, attachment: Null) =
          val request = ByteBuffer.allocate(256)
          socket.read(request).get()
          val body = "downloaded body"
          val response =
            s"HTTP/1.1 200 OK\r\nContent-Length: ${body.length}\r\n\r\n$body"
          socket.write(ByteBuffer.wrap(response.getBytes(StandardCharsets.UTF_8))).get()
          socket.close()
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"http server failed: ${exc.getMessage}")
          latch.countDown()
    )

    client.connect(address).get()
    val request =
      ByteBuffer.wrap("GET / HTTP/1.1\r\nHost: localhost\r\n\r\n".getBytes())
    client.write(request).get()
    val response = ByteBuffer.allocate(256)
    client.read(response).get()
    response.flip()
    val bytes = Array.ofDim[Byte](response.remaining())
    response.get(bytes)
    println(new String(bytes, StandardCharsets.UTF_8))
    latch.await()
  finally
    client.close()
    server.close()
end main
```

A downloader is just a protocol client that speaks HTTP over a socket.  
This local demo avoids external dependencies while still showing how an  
async request and response exchange can work on raw TCP channels.  

## Async Request and Response Protocol over TCP  
Send a request string and return a computed response on the same socket.  

```scala
import java.net.InetSocketAddress
import java.nio.ByteBuffer
import java.nio.channels.
  {AsynchronousServerSocketChannel, AsynchronousSocketChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.util.concurrent.CountDownLatch

@main def main() =
  val server = AsynchronousServerSocketChannel.open()
  server.bind(InetSocketAddress("127.0.0.1", 0))
  val address = server.getLocalAddress.asInstanceOf[InetSocketAddress]
  val client = AsynchronousSocketChannel.open()
  val latch = CountDownLatch(1)

  try
    server.accept(
      null,
      new CompletionHandler[AsynchronousSocketChannel, Null]:
        def completed(socket: AsynchronousSocketChannel, attachment: Null) =
          val buffer = ByteBuffer.allocate(64)
          socket.read(buffer).get()
          buffer.flip()
          val bytes = Array.ofDim[Byte](buffer.remaining())
          buffer.get(bytes)
          val request = new String(bytes, StandardCharsets.UTF_8).trim
          val reply = s"response for $request"
          socket.write(ByteBuffer.wrap(reply.getBytes(StandardCharsets.UTF_8))).get()
          socket.close()
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"protocol failed: ${exc.getMessage}")
          latch.countDown()
    )

    client.connect(address).get()
    client.write(ByteBuffer.wrap("status".getBytes(StandardCharsets.UTF_8))).get()
    val response = ByteBuffer.allocate(64)
    client.read(response).get()
    response.flip()
    val bytes = Array.ofDim[Byte](response.remaining())
    response.get(bytes)
    println(new String(bytes, StandardCharsets.UTF_8))
    latch.await()
  finally
    client.close()
    server.close()
end main
```

Request and response protocols are the basis of many application-level  
services.  
The async part is the transport handling, while the protocol itself is the  
rule that maps incoming bytes to outgoing bytes.  

## Async Batching of Writes for Performance  
Group several messages into one buffer and issue one async write.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, StandardOpenOption}
import java.util.concurrent.CountDownLatch

@main def main() =
  val path = Files.createTempFile("aio-batch", ".txt")
  val channel =
    AsynchronousFileChannel.open(
      path,
      StandardOpenOption.CREATE,
      StandardOpenOption.WRITE,
      StandardOpenOption.READ
    )
  val lines = List("red", "green", "blue")
  val payload = lines.mkString("\n") + "\n"
  val buffer = ByteBuffer.wrap(payload.getBytes(StandardCharsets.UTF_8))
  val latch = CountDownLatch(1)

  try
    channel.write(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          println(Files.readString(path))
          latch.countDown()

        def failed(exc: Throwable, attachment: Null) =
          println(s"batch failed: ${exc.getMessage}")
          latch.countDown()
    )

    latch.await()
  finally
    channel.close()
    Files.deleteIfExists(path)
end main
```

Batching reduces the number of system calls and completion events, which  
can improve throughput.  
The trade-off is added buffering delay, so batching works best when small  
latency increases are acceptable.  

## Error-Resilient Async Retry Logic  
Retry a flaky async operation until it succeeds or the limit is reached.  

```scala
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Future, Promise}
import java.util.concurrent.atomic.AtomicInteger

@main def main() =
  val attempts = AtomicInteger(0)

  def flakyRead(): Future[String] =
    Future:
      val n = attempts.incrementAndGet()
      if n < 3 then throw new RuntimeException(s"failed attempt $n")
      else "eventual success"

  def retry[A](max: Int)(op: => Future[A]): Future[A] =
    op.recoverWith:
      case ex if max > 0 =>
        println(ex.getMessage)
        retry(max - 1)(op)

  retry(3)(flakyRead()).foreach(println)
  Thread.sleep(300)
end main
```

Retries help when failures are temporary, such as a busy peer or a short  
network hiccup.  
They should remain bounded, because endless retries hide real problems and  
can amplify load on an already struggling system.  

## Structured Logging of Async Failures  
Log async errors in a key-value form that is easy to search later.  

```scala
import java.nio.file.{Files, Path}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.Future

@main def main() =
  def logFailure(stage: String, ex: Throwable): Unit =
    println(s"level=error stage=$stage type=${ex.getClass.getSimpleName}")
    println(s"message=${ex.getMessage}")

  val missing = Path.of("does-not-exist.txt")

  Future(Files.readString(missing)).failed.foreach: ex =>
    logFailure("load-config", ex)

  Thread.sleep(300)
end main
```

Structured logs make asynchronous failures easier to correlate because the  
important fields are already separated.  
That is especially useful when many requests run concurrently and plain  
free-form text logs become hard to filter.  

## Reading a Large File in Chunks Asynchronously  
Use repeated async reads to process a file chunk by chunk.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, StandardOpenOption}
import java.util.concurrent.CountDownLatch

@main def main() =
  val path = Files.createTempFile("aio-chunks", ".txt")
  Files.writeString(path, "abcdefghij" * 10)
  val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
  val latch = CountDownLatch(1)

  def readChunk(position: Long): Unit =
    val buffer = ByteBuffer.allocate(10)
    channel.read(
      buffer,
      position,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          if result <= 0 then
            latch.countDown()
          else
            buffer.flip()
            val bytes = Array.ofDim[Byte](buffer.remaining())
            buffer.get(bytes)
            println(new String(bytes, StandardCharsets.UTF_8))
            readChunk(position + result)

        def failed(exc: Throwable, attachment: Null) =
          println(s"chunk read failed: ${exc.getMessage}")
          latch.countDown()
    )

  try
    readChunk(0L)
    latch.await()
  finally
    channel.close()
    Files.deleteIfExists(path)
end main
```

Chunked reads keep memory use stable even when a file is large.  
They also create a natural place to add progress reporting, decoding, or  
streaming transformations between chunks.  

## Async File Append with Sequential Guarantees  
Reserve file positions atomically so appends stay ordered.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, StandardOpenOption}
import java.util.concurrent.CountDownLatch
import java.util.concurrent.atomic.AtomicLong

@main def main() =
  val path = Files.createTempFile("aio-append", ".txt")
  val channel =
    AsynchronousFileChannel.open(
      path,
      StandardOpenOption.CREATE,
      StandardOpenOption.WRITE,
      StandardOpenOption.READ
    )
  val nextPos = AtomicLong(0L)
  val latch = CountDownLatch(3)

  def append(text: String): Unit =
    val bytes = s"$text\n".getBytes(StandardCharsets.UTF_8)
    val position = nextPos.getAndAdd(bytes.length.toLong)
    val buffer = ByteBuffer.wrap(bytes)
    channel.write(
      buffer,
      position,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) = latch.countDown()
        def failed(exc: Throwable, attachment: Null) =
          println(s"append failed: ${exc.getMessage}")
          latch.countDown()
    )

  try
    append("first")
    append("second")
    append("third")
    latch.await()
    println(Files.readString(path))
  finally
    channel.close()
    Files.deleteIfExists(path)
end main
```

Sequential guarantees matter whenever the order of records is meaningful.  
Using explicit positions avoids races that would exist if several writes  
shared one implicit file pointer.  

## Non-Blocking Timeout Using ScheduledExecutorService  
Fail a promise later without blocking the current thread.  

```scala
import java.util.concurrent.Executors
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Future, Promise}

@main def main() =
  def timeoutAfter[A](timeoutMs: Long): Future[A] =
    val promise = Promise[A]()
    val scheduler = Executors.newSingleThreadScheduledExecutor()
    scheduler.schedule(
      () =>
        promise.tryFailure(new RuntimeException("timeout"))
        scheduler.shutdown(),
      timeoutMs,
      java.util.concurrent.TimeUnit.MILLISECONDS
    )
    promise.future

  timeoutAfter[String](100).failed.foreach(ex => println(ex.getMessage))
  Thread.sleep(200)
end main
```

This is non-blocking because no thread waits in `sleep` or `await` for the  
deadline to pass.  
The scheduler delivers the timeout event later, which fits naturally into  
future-based async workflows.  

## Writing File Metadata Alongside Content Asynchronously  
Store content in one file and metadata in a sibling file with AIO.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Path, StandardOpenOption}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Future, Promise}

@main def main() =
  def writeText(path: Path, text: String): Future[Int] =
    val promise = Promise[Int]()
    val channel =
      AsynchronousFileChannel.open(
        path,
        StandardOpenOption.CREATE,
        StandardOpenOption.WRITE,
        StandardOpenOption.TRUNCATE_EXISTING
      )
    val buffer = ByteBuffer.wrap(text.getBytes(StandardCharsets.UTF_8))
    channel.write(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          try promise.success(result)
          finally channel.close()

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally channel.close()
    )
    promise.future

  val content = Files.createTempFile("aio-meta", ".txt")
  val meta = Path.of(content.toString + ".meta")

  val program = Future.sequence:
    List(
      writeText(content, "payload"),
      writeText(meta, "content-type=text/plain")
    )

  program.foreach(_ =>
    println(Files.readString(content))
    println(Files.readString(meta))
  )
  Thread.sleep(400)
  Files.deleteIfExists(content)
  Files.deleteIfExists(meta)
end main
```

Many storage formats keep payload bytes and descriptive metadata in two  
separate places.  
Async writes let those files be created independently while later logic  
still observes them as one combined outcome.  

## Async Fan-Out: Broadcasting to Multiple Writers  
Write the same payload to several files at the same time.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Path, StandardOpenOption}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Future, Promise}

@main def main() =
  def writeText(path: Path, text: String): Future[Int] =
    val promise = Promise[Int]()
    val channel =
      AsynchronousFileChannel.open(
        path,
        StandardOpenOption.CREATE,
        StandardOpenOption.WRITE,
        StandardOpenOption.TRUNCATE_EXISTING
      )
    val buffer = ByteBuffer.wrap(text.getBytes(StandardCharsets.UTF_8))
    channel.write(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          try promise.success(result)
          finally channel.close()

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally channel.close()
    )
    promise.future

  val targets = List.fill(3)(Files.createTempFile("aio-fanout", ".txt"))

  Future.sequence(targets.map(path => writeText(path, "broadcast"))).foreach: _ =>
    targets.foreach(path => println(Files.readString(path)))

  Thread.sleep(400)
  targets.foreach(Files.deleteIfExists)
end main
```

Fan-out sends one upstream result to many downstream consumers.  
It is common in caching, replication, and logging pipelines where several  
outputs must observe the same input data.  

## Async Fan-In: Merging Reads from Multiple Files  
Read several files concurrently and join their contents into one string.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Path, StandardOpenOption}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Future, Promise}

@main def main() =
  def readText(path: Path): Future[String] =
    val promise = Promise[String]()
    val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
    val buffer = ByteBuffer.allocate(32)
    channel.read(
      buffer,
      0L,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          try
            buffer.flip()
            val bytes = Array.ofDim[Byte](buffer.remaining())
            buffer.get(bytes)
            promise.success(new String(bytes, StandardCharsets.UTF_8))
          finally channel.close()

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally channel.close()
    )
    promise.future

  val paths =
    List("north", "south", "east").map: text =>
      val path = Files.createTempFile("aio-fanin", ".txt")
      Files.writeString(path, text)
      path

  Future.sequence(paths.map(readText)).foreach(values => println(values.mkString(" + ")))
  Thread.sleep(400)
  paths.foreach(Files.deleteIfExists)
end main
```

Fan-in is the opposite of fan-out: many independent sources become one  
combined result.  
`Future.sequence` is a direct expression of this pattern for fixed-size  
collections of asynchronous tasks.  

## Chaining Dependent Async Operations  
Use one async result to determine the next write position and content.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.{AsynchronousFileChannel, CompletionHandler}
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Path, StandardOpenOption}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.{Future, Promise}

@main def main() =
  def writeAt(path: Path, position: Long, text: String): Future[Int] =
    val promise = Promise[Int]()
    val channel =
      AsynchronousFileChannel.open(
        path,
        StandardOpenOption.CREATE,
        StandardOpenOption.WRITE,
        StandardOpenOption.READ
      )
    val buffer = ByteBuffer.wrap(text.getBytes(StandardCharsets.UTF_8))
    channel.write(
      buffer,
      position,
      null,
      new CompletionHandler[Integer, Null]:
        def completed(result: Integer, attachment: Null) =
          try promise.success(result)
          finally channel.close()

        def failed(exc: Throwable, attachment: Null) =
          try promise.failure(exc)
          finally channel.close()
    )
    promise.future

  val path = Files.createTempFile("aio-chain", ".txt")

  val program =
    for
      headerSize <- writeAt(path, 0L, "header\n")
      _ <- writeAt(path, headerSize.toLong, "body\n")
    yield Files.readString(path)

  program.foreach(println)
  Thread.sleep(400)
  Files.deleteIfExists(path)
end main
```

Dependent operations are common when later steps need a position, token,  
or other value produced by earlier work.  
Futures make that dependency explicit without forcing the code back into  
manual nested callbacks.  

## Async Resource Pool for File Channels  
Reuse a small number of open file channels through a simple pool.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.AsynchronousFileChannel
import java.nio.charset.StandardCharsets
import java.nio.file.{Files, StandardOpenOption}
import java.util.concurrent.ArrayBlockingQueue

@main def main() =
  val path = Files.createTempFile("aio-pool-file", ".txt")
  Files.writeString(path, "pooled channel")
  val pool = ArrayBlockingQueue[AsynchronousFileChannel](2)
  pool.put(AsynchronousFileChannel.open(path, StandardOpenOption.READ))
  pool.put(AsynchronousFileChannel.open(path, StandardOpenOption.READ))

  def borrowAndRead(): String =
    val channel = pool.take()
    try
      val buffer = ByteBuffer.allocate(32)
      channel.read(buffer, 0L).get()
      buffer.flip()
      val bytes = Array.ofDim[Byte](buffer.remaining())
      buffer.get(bytes)
      new String(bytes, StandardCharsets.UTF_8)
    finally
      pool.put(channel)

  try
    println(borrowAndRead())
    println(borrowAndRead())
  finally
    pool.forEach(_.close())
    Files.deleteIfExists(path)
end main
```

A resource pool limits how many expensive handles are open at one time and  
lets callers reuse them safely.  
This pattern is useful when channel creation cost is noticeable or when a  
service must cap the number of open descriptors deliberately.  

## Benchmarking Async Versus Synchronous File IO  
Run a small loop that compares repeated sync and async file reads.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.AsynchronousFileChannel
import java.nio.file.{Files, StandardOpenOption}

@main def main() =
  val path = Files.createTempFile("aio-bench", ".txt")
  Files.writeString(path, "benchmark-data" * 200)

  val syncStart = System.nanoTime()
  (1 to 20).foreach(_ => Files.readAllBytes(path))
  val syncElapsed = System.nanoTime() - syncStart

  val channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ)
  val asyncStart = System.nanoTime()

  try
    (1 to 20).foreach: _ =>
      val buffer = ByteBuffer.allocate(Files.size(path).toInt)
      channel.read(buffer, 0L).get()
    val asyncElapsed = System.nanoTime() - asyncStart
    println(s"sync ns: $syncElapsed")
    println(s"async ns: $asyncElapsed")
  finally
    channel.close()
    Files.deleteIfExists(path)
end main
```

A tiny benchmark cannot settle every performance question, but it shows  
how to compare two approaches under the same local conditions.  
Always benchmark with realistic workloads, because filesystem caches and  
small data sizes can hide the real trade-offs of asynchronous designs.  
