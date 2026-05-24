# Advanced NIO.2 & Channels in Scala 3

NIO.2 is the second-generation non-blocking IO API introduced in Java 7  
under the `java.nio.file` and `java.nio.channels` packages. It was created  
to address the well-known shortcomings of the original `java.io` package:  
poor symbolic-link support, missing metadata APIs, no directory-watch  
facility, and a stream abstraction that ties a thread to every IO  
operation. NIO.2 brings a rich, composable file-system API together with  
channel-based IO that decouples threads from data movement.  

**Classic IO vs NIO.2.** The classic `java.io` API uses streams — ordered  
sequences of bytes where reads and writes are inherently blocking. A  
thread calling `InputStream.read` stays parked until data arrives. NIO.2  
replaces streams with channels and buffers. A channel represents an open  
connection to an entity such as a file or socket. Data always travels  
through a `ByteBuffer`, giving the caller full control over alignment and  
capacity. Channels can also operate in non-blocking mode, allowing a  
single thread to service many connections.  

**Channels vs streams.** A stream is unidirectional: an `InputStream` only  
reads; an `OutputStream` only writes. A channel is typically bidirectional  
and can be both read from and written to through the same handle. Channels  
also support scatter/gather IO, memory mapping, and locking, none of which  
are available in the stream API.  

**Buffers and how they work.** A `ByteBuffer` is a fixed-capacity block of  
memory. Three integer cursors govern every operation:  

- *capacity* — total allocated bytes, never changes after creation.  
- *limit* — last valid index for a read or write; set to capacity after  
  `clear`, and set to the current position after `flip`.  
- *position* — next byte to be read or written; advanced by every IO call.  

A typical write cycle: allocate → write data (position advances) → `flip`  
(limit = position, position = 0) → pass to channel for reading. To reuse  
the buffer for more writes, call `clear` (position = 0, limit = capacity).  
`compact` is a variant that preserves unread bytes at the start of the  
buffer before accepting more data.  

**Blocking vs non-blocking IO.** In blocking mode every channel operation  
waits until the kernel satisfies the request. In non-blocking mode channel  
operations return immediately, possibly with zero bytes transferred. The  
caller registers the channel with a `Selector` and wakes only when the OS  
signals that data is ready. Non-blocking IO is essential for high-  
concurrency servers that cannot afford one thread per connection.  

**Memory-mapped IO.** `FileChannel.map` exposes a region of a file as a  
`MappedByteBuffer` backed directly by the OS page cache. Reads and writes  
go through memory addresses rather than system-call read/write paths. This  
eliminates one user-space copy and is the fastest way to process large  
files sequentially or with random access. It is ideal for read-heavy  
workloads such as search indexes, log parsing, and database storage  
engines. The trade-off is that mappings consume virtual address space and  
can complicate error handling because IO errors appear as signals rather  
than exceptions.  

**File watchers and event-driven IO.** `WatchService` registers  
directories with the kernel-level file-system notification mechanism  
(inotify on Linux, FSEvents on macOS, ReadDirectoryChangesW on Windows).  
When a file is created, modified, or deleted the OS queues a  
`WatchEvent`. A polling thread calls `take` or `poll` to drain the queue  
and dispatch events without scanning the directory repeatedly.  

**Performance characteristics.** NIO.2 outperforms classic IO in several  
dimensions. Large sequential reads benefit from larger buffers that amortise  
system-call overhead. Scatter/gather IO batches multiple buffers into a  
single kernel transition. Direct buffers avoid a heap-to-native copy on  
every channel operation. Memory-mapped files leverage the OS page cache  
most efficiently. Non-blocking selectors allow a handful of threads to  
handle thousands of simultaneous connections.  

---

## Advanced Path manipulation

Paths can be dissected and reassembled using the standard `Path` API.  

```scala
import java.nio.file.Paths

@main def main() =
  val p = Paths.get("/home/user/projects/scala/src/Main.scala")

  println(s"root      : ${p.getRoot}")
  println(s"parent    : ${p.getParent}")
  println(s"filename  : ${p.getFileName}")
  println(s"name count: ${p.getNameCount}")
  println(s"element 0 : ${p.getName(0)}")
  println(s"subpath   : ${p.subpath(2, 4)}")
  println(s"absolute  : ${p.isAbsolute}")
end main
```

`getRoot` returns the filesystem root (`/` on Unix, a drive letter on  
Windows) or `null` for relative paths. `getParent` strips the last  
component, while `getFileName` returns only the leaf name. `subpath`  
extracts a range of name elements without the root. These methods give  
precise control over path decomposition without resorting to string  
splitting.  

## Resolving siblings

`resolveSibling` produces a new path in the same directory as the  
original without navigating up and back down manually.  

```scala
import java.nio.file.Paths

@main def main() =
  val source = Paths.get("/data/reports/2024/january.csv")
  val sibling = source.resolveSibling("february.csv")
  val subDir  = source.resolveSibling(Paths.get("archive/jan.csv"))

  println(sibling)
  println(subDir)
end main
```

`resolveSibling(name)` is shorthand for `parent.resolve(name)`. It  
avoids the noise of explicitly calling `getParent` first and is  
particularly readable when renaming or rotating files in a batch  
process. Passing a full relative `Path` instead of a string allows  
multi-level sibling navigation in a single call.  

## Relativizing paths

`relativize` computes the path that navigates from one location to  
another, enabling portable path expressions.  

```scala
import java.nio.file.Paths

@main def main() =
  val base    = Paths.get("/home/user/projects")
  val target  = Paths.get("/home/user/projects/scala/src/Main.scala")
  val reports = Paths.get("/home/user/documents/reports")

  println(base.relativize(target))   // scala/src/Main.scala
  println(base.relativize(reports))  // ../../documents/reports
end main
```

Both paths must be either absolute or relative; mixing them raises an  
`IllegalArgumentException`. The resulting path contains `..` components  
whenever target lies outside the base tree. `relativize` is the inverse  
of `resolve`: `base.resolve(base.relativize(target))` reconstructs the  
original target.  

## Walking directory trees with Files.walk

`Files.walk` streams every path reachable under a root directory.  

```scala
import java.nio.file.{Files, Paths}
import scala.jdk.CollectionConverters.*

@main def main() =
  val root = Paths.get(sys.props("user.home"))

  val scalaFiles = Files.walk(root, 4)
    .filter(p => p.toString.endsWith(".scala"))
    .limit(10)
    .toArray
    .map(_.toString)

  scalaFiles.foreach(println)
end main
```

`walk` returns a lazy `java.util.stream.Stream` so the traversal is  
depth-first and does not load the entire tree into memory at once. The  
second argument limits recursion depth, preventing runaway traversal on  
deep file systems. Wrapping with `.filter` and `.limit` before `.toArray`  
keeps memory use bounded. Always call `.close` on the stream (or use  
try-with-resources) to release the directory handles it holds.  

## Filtering with predicates

Custom predicates narrow `Files.walk` results to exactly the paths that  
match a business rule.  

```scala
import java.nio.file.{Files, Paths, Path}
import java.nio.file.attribute.BasicFileAttributes

@main def main() =
  val root = Paths.get(".")

  val largeSources = Files.find(
    root, 5,
    (p: Path, attrs: BasicFileAttributes) =>
      attrs.isRegularFile &&
      p.toString.endsWith(".scala") &&
      attrs.size > 1024
  )

  largeSources.forEach(p => println(s"$p  ${Files.size(p)} bytes"))
end main
```

`Files.find` accepts a `BiPredicate[Path, BasicFileAttributes]` so the  
filter has access to size, timestamps, and file kind without a second  
system call for each path. This is more efficient than calling  
`Files.readAttributes` separately inside a `walk` filter. The predicate  
runs on every visited path; return `false` to skip a file without  
short-circuiting the traversal.  

## Symbolic links

Symbolic links can be created, detected, and read through the  
`Files` utility class.  

```scala
import java.nio.file.{Files, Paths}

@main def main() =
  val target = Paths.get("/tmp/original.txt")
  val link   = Paths.get("/tmp/link.txt")

  Files.writeString(target, "symbolic link target\n")

  if Files.exists(link) then Files.delete(link)
  Files.createSymbolicLink(link, target)

  println(s"isSymbolicLink : ${Files.isSymbolicLink(link)}")
  println(s"readSymbolicLink: ${Files.readSymbolicLink(link)}")
  println(s"content: ${Files.readString(link)}")

  Files.delete(link)
  Files.delete(target)
end main
```

`createSymbolicLink` requires the `target` argument to be the path the  
link points to; it need not yet exist. `isSymbolicLink` returns `true`  
only when the path itself is a link, not its destination. `readSymbolicLink`  
returns the raw link value without resolving intermediate components.  
Reading through a link transparently follows it, so `Files.readString`  
returns the target's content.  

## Hard links

A hard link shares the same inode as its original, making both  
paths identical references to the same data.  

```scala
import java.nio.file.{Files, Paths}

@main def main() =
  val original = Paths.get("/tmp/data.txt")
  val hardLink = Paths.get("/tmp/data-link.txt")

  Files.writeString(original, "shared content\n")

  if Files.exists(hardLink) then Files.delete(hardLink)
  Files.createLink(hardLink, original)

  val attrO = Files.readAttributes(original, "unix:ino,nlink")
  println(s"original inode : $attrO")

  Files.writeString(hardLink, "updated content\n")
  println(s"original sees: ${Files.readString(original)}")

  Files.delete(hardLink)
  Files.delete(original)
end main
```

Both paths refer to the same underlying data blocks. Writing through  
either one immediately affects the other because they share a single  
inode. Deleting one path leaves the other fully intact; the data is  
reclaimed only when the link count drops to zero. Hard links cannot  
span file system boundaries, which is a key difference from symbolic  
links.  

## Reading file attributes

`BasicFileAttributes`, `PosixFileAttributes`, and `DosFileAttributes`  
expose platform-specific metadata for any path.  

```scala
import java.nio.file.{Files, Paths}
import java.nio.file.attribute.{BasicFileAttributes, PosixFileAttributes}

@main def main() =
  val p = Paths.get(sys.props("user.home"))

  val basic = Files.readAttributes(p, classOf[BasicFileAttributes])
  println(s"size          : ${basic.size}")
  println(s"created       : ${basic.creationTime}")
  println(s"last modified : ${basic.lastModifiedTime}")
  println(s"isDirectory   : ${basic.isDirectory}")

  try
    val posix = Files.readAttributes(p, classOf[PosixFileAttributes])
    println(s"owner  : ${posix.owner.getName}")
    println(s"group  : ${posix.group.getName}")
    println(s"perms  : ${posix.permissions}")
  catch case _: UnsupportedOperationException =>
    println("POSIX attributes not supported on this platform")
end main
```

`BasicFileAttributes` is available on every platform and covers the  
essentials: size, creation time, last-modified time, and file kind.  
`PosixFileAttributes` extends it with UNIX owner, group, and permission  
bits; it throws on Windows. Prefer `readAttributes` with a typed class  
reference over the string-keyed `getAttribute` variant because the  
compiler enforces the return type and avoids casting.  

## Setting file permissions

POSIX file permissions can be modified through `PosixFilePermission`  
and `Files.setPosixFilePermissions`.  

```scala
import java.nio.file.{Files, Paths}
import java.nio.file.attribute.PosixFilePermission
import java.nio.file.attribute.PosixFilePermissions

@main def main() =
  val file = Paths.get("/tmp/secure.txt")
  Files.writeString(file, "sensitive data\n")

  val perms = PosixFilePermissions.fromString("rw-------")
  Files.setPosixFilePermissions(file, perms)

  val actual = Files.getPosixFilePermissions(file)
  println(s"permissions: ${PosixFilePermissions.toString(actual)}")

  // Add group-read permission programmatically
  actual.add(PosixFilePermission.GROUP_READ)
  Files.setPosixFilePermissions(file, actual)
  println(s"after add  : ${PosixFilePermissions.toString(actual)}")

  Files.delete(file)
end main
```

`PosixFilePermissions.fromString` parses a standard nine-character  
permission string (`rwxrwxrwx`) into an `EnumSet`. The set is mutable  
so individual permissions can be toggled before applying. On JVM,  
`setPosixFilePermissions` delegates to `chmod`; the call throws  
`UnsupportedOperationException` on Windows where POSIX semantics  
do not apply.  

## Reading file stores and disk space

`FileStore` exposes total, usable, and unallocated space for each  
mounted partition or volume.  

```scala
import java.nio.file.{Files, FileSystems, Paths}

@main def main() =
  val fs = FileSystems.getDefault

  for store <- fs.getFileStores.nn.iterator.nn do
    val total   = store.getTotalSpace / (1024 * 1024)
    val usable  = store.getUsableSpace / (1024 * 1024)
    val unalloc = store.getUnallocatedSpace / (1024 * 1024)
    println(f"${store.name}%-20s  total=${total}%6d MB" +
            f"  usable=${usable}%6d MB" +
            f"  free=${unalloc}%6d MB")

  // Query the store for a specific path
  val home  = Paths.get(sys.props("user.home"))
  val store = Files.getFileStore(home)
  println(s"\nhome partition: ${store.name} (${store.`type`})")
end main
```

`FileSystems.getDefault.getFileStores` iterates all mounts visible to  
the JVM. `getUsableSpace` reports space available to the current user  
(respecting quota limits), which may be less than `getUnallocatedSpace`.  
`Files.getFileStore(path)` is a convenience that returns the store for  
any specific path without iterating them all.  

---

## DirectoryStream with glob filters

`Files.newDirectoryStream(dir, glob)` lists only entries whose names  
match a shell-style glob pattern.  

```scala
import java.nio.file.{Files, Paths}
import scala.jdk.CollectionConverters.*

@main def main() =
  val dir = Paths.get(".")

  val stream = Files.newDirectoryStream(dir, "*.{scala,md}")
  try
    for entry <- stream.asScala do
      println(s"${entry.getFileName}  " +
              s"${Files.size(entry)} bytes")
  finally
    stream.close()
end main
```

The glob is matched against the filename component only, not the full  
path. Braces expand to alternatives, `*` matches any sequence within a  
single path segment, and `**` is not supported here (use `Files.walk`  
for deep traversal). `DirectoryStream` implements `Closeable` so it  
must be closed; wrapping it in try/finally or Scala's `Using` guarantees  
that the underlying directory handle is released.  

## DirectoryStream with custom filters

A `DirectoryStream.Filter[Path]` lambda lets you apply arbitrary logic  
when listing a directory.  

```scala
import java.nio.file.{Files, Paths, DirectoryStream, Path}
import scala.jdk.CollectionConverters.*

@main def main() =
  val dir = Paths.get(".")

  val filter = new DirectoryStream.Filter[Path]:
    def accept(entry: Path): Boolean =
      Files.isRegularFile(entry) &&
      Files.size(entry) > 500 &&
      !entry.getFileName.toString.startsWith(".")

  val stream = Files.newDirectoryStream(dir, filter)
  try
    for path <- stream.asScala do
      println(f"${path.getFileName}%-30s ${Files.size(path)}%6d B")
  finally
    stream.close()
end main
```

The anonymous `DirectoryStream.Filter` in Scala 3 uses the concise  
anonymous-class syntax with a colon. Each `accept` call receives the  
path of one directory entry; returning `false` excludes it from the  
stream. Filters are evaluated lazily as the caller iterates, so only  
entries actually consumed by the loop trigger the predicate.  

## Recursive traversal with Files.walkFileTree

`Files.walkFileTree` gives complete control over depth-first traversal  
by routing every event through a `FileVisitor`.  

```scala
import java.nio.file.{Files, Paths, FileVisitResult, SimpleFileVisitor, Path}
import java.nio.file.attribute.BasicFileAttributes
import java.io.IOException

@main def main() =
  var fileCount = 0
  var totalBytes = 0L

  Files.walkFileTree(Paths.get("."), new SimpleFileVisitor[Path]:
    override def visitFile(
        file: Path,
        attrs: BasicFileAttributes): FileVisitResult =
      fileCount += 1
      totalBytes += attrs.size
      FileVisitResult.CONTINUE

    override def visitFileFailed(
        file: Path,
        exc: IOException): FileVisitResult =
      System.err.println(s"failed: $file - $exc")
      FileVisitResult.CONTINUE
  )

  println(s"files: $fileCount  total: $totalBytes bytes")
end main
```

`SimpleFileVisitor` provides no-op defaults for all four callbacks  
(`preVisitDirectory`, `visitFile`, `visitFileFailed`, `postVisitDirectory`),  
so you only override the ones you need. Returning  
`FileVisitResult.SKIP_SUBTREE` from `preVisitDirectory` prevents descent  
into a specific subdirectory. Errors during traversal call `visitFileFailed`  
instead of propagating an exception, which keeps the walk going.  

## Custom FileVisitor implementations

A full `FileVisitor` implementation enables bidirectional communication  
between the traversal and the caller.  

```scala
import java.nio.file.*
import java.nio.file.attribute.BasicFileAttributes
import java.io.IOException
import scala.collection.mutable.ListBuffer

class ScalaFileCollector extends FileVisitor[Path]:
  val found = ListBuffer[Path]()
  private var depth = 0

  def preVisitDirectory(dir: Path,
      attrs: BasicFileAttributes): FileVisitResult =
    depth += 1
    if depth > 3 then FileVisitResult.SKIP_SUBTREE
    else FileVisitResult.CONTINUE

  def visitFile(file: Path,
      attrs: BasicFileAttributes): FileVisitResult =
    if file.toString.endsWith(".scala") then found += file
    FileVisitResult.CONTINUE

  def visitFileFailed(file: Path,
      exc: IOException): FileVisitResult =
    FileVisitResult.CONTINUE

  def postVisitDirectory(dir: Path,
      exc: IOException): FileVisitResult =
    depth -= 1
    FileVisitResult.CONTINUE

@main def main() =
  val visitor = ScalaFileCollector()
  Files.walkFileTree(Paths.get("."), visitor)
  visitor.found.foreach(println)
end main
```

Implementing the full interface rather than extending `SimpleFileVisitor`  
is useful when the visitor needs to maintain complex state across all  
four events. The `depth` counter here enforces a soft limit without the  
`maxDepth` parameter of `walkFileTree`. Storing results in a `ListBuffer`  
field decouples collection from reporting, keeping each method focused.  

## Skipping subtrees

`FileVisitResult.SKIP_SUBTREE` returned from `preVisitDirectory` avoids  
descending into unwanted branches entirely.  

```scala
import java.nio.file.*
import java.nio.file.attribute.BasicFileAttributes
import java.io.IOException

@main def main() =
  val skipDirs = Set(".git", "target", "node_modules", ".idea")

  Files.walkFileTree(Paths.get("."), new SimpleFileVisitor[Path]:
    override def preVisitDirectory(
        dir: Path,
        attrs: BasicFileAttributes): FileVisitResult =
      if skipDirs.contains(dir.getFileName.toString) then
        FileVisitResult.SKIP_SUBTREE
      else
        FileVisitResult.CONTINUE

    override def visitFile(
        file: Path,
        attrs: BasicFileAttributes): FileVisitResult =
      println(file)
      FileVisitResult.CONTINUE
  )
end main
```

Skipping build and version-control directories often eliminates the  
vast majority of paths in a typical project, dramatically reducing  
traversal time. The check uses only the filename component so the  
filter applies uniformly regardless of where the tree is rooted.  
`SKIP_SUBTREE` only has effect from `preVisitDirectory`; returning it  
from `visitFile` is treated as `CONTINUE`.  

## Collecting results during traversal

A traversal can accumulate structured results by combining a visitor  
with an immutable builder.  

```scala
import java.nio.file.*
import java.nio.file.attribute.BasicFileAttributes
import java.io.IOException
import scala.collection.mutable

@main def main() =
  val index = mutable.Map.empty[String, List[Path]]

  Files.walkFileTree(Paths.get("."), new SimpleFileVisitor[Path]:
    override def visitFile(
        file: Path,
        attrs: BasicFileAttributes): FileVisitResult =
      val ext = file.toString.split('.').lastOption.getOrElse("none")
      index(ext) = file :: index.getOrElse(ext, Nil)
      FileVisitResult.CONTINUE
  )

  index.toSeq.sortBy(_._1).foreach: (ext, paths) =>
    println(s".$ext  (${paths.size} files)")
end main
```

Grouping by extension demonstrates how a visitor can build a rich data  
structure rather than just printing. The `mutable.Map` is updated inside  
`visitFile` using a prepend to avoid the O(n) cost of `append`. After  
the walk, the map is converted to a sorted sequence for deterministic  
output. This pattern composes cleanly: the visitor is a factory detail;  
the caller receives a plain data structure.  

---

## Reading with FileChannel

`FileChannel` reads bytes directly into a `ByteBuffer` with a single  
system call.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption}

@main def main() =
  val path = Paths.get("nio2-channels.md")
  val channel = FileChannel.open(path, StandardOpenOption.READ)

  try
    val buffer = ByteBuffer.allocate(256)
    var bytesRead = channel.read(buffer)
    while bytesRead != -1 do
      buffer.flip()
      val bytes = new Array[Byte](buffer.limit)
      buffer.get(bytes)
      print(String(bytes))
      buffer.clear()
      bytesRead = channel.read(buffer)
  finally
    channel.close()
end main
```

`FileChannel.open` with `StandardOpenOption.READ` creates a read-only  
channel. Each call to `channel.read(buffer)` fills the buffer from the  
current channel position and returns the number of bytes transferred, or  
`-1` at end-of-file. `flip` prepares the buffer for draining by setting  
`limit = position` and resetting `position` to zero. After consuming all  
bytes, `clear` resets both cursors for the next fill cycle.  

## Writing with FileChannel

`FileChannel` with `WRITE` and `TRUNCATE_EXISTING` overwrites a file  
using buffered channel semantics.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption}

@main def main() =
  val path = Paths.get("/tmp/channel-write.txt")
  val channel = FileChannel.open(
    path,
    StandardOpenOption.WRITE,
    StandardOpenOption.CREATE,
    StandardOpenOption.TRUNCATE_EXISTING
  )

  try
    val lines = List("line one\n", "line two\n", "line three\n")
    for line <- lines do
      val buf = ByteBuffer.wrap(line.getBytes("UTF-8"))
      while buf.hasRemaining do
        channel.write(buf)
    channel.force(true)
    println(s"wrote ${channel.size} bytes")
  finally
    channel.close()
end main
```

`TRUNCATE_EXISTING` clears the file on open, while `CREATE` creates it  
if missing. A single `write` call may not transfer all bytes if the  
channel is in non-blocking mode or the buffer is very large, so the  
`while buf.hasRemaining` loop ensures all data is written. `force(true)`  
flushes both file content and metadata (including the directory entry) to  
durable storage.  

## Scattering reads

A scattering read fills an array of buffers in sequence from a single  
channel read call.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption}

@main def main() =
  val path = Paths.get("/tmp/scatter.bin")

  // Write 12 bytes of structured data
  val out = FileChannel.open(path, StandardOpenOption.WRITE,
    StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING)
  out.write(ByteBuffer.wrap(Array[Byte](1, 2, 3, 4,
                                        10, 20, 30, 40,
                                        100.toByte, 110.toByte,
                                        120.toByte, 127.toByte)))
  out.close()

  // Scatter into three separate buffers
  val header  = ByteBuffer.allocate(4)
  val payload = ByteBuffer.allocate(4)
  val trailer = ByteBuffer.allocate(4)
  val buffers = Array[java.nio.ByteBuffer](header, payload, trailer)

  val channel = FileChannel.open(path, StandardOpenOption.READ)
  try
    channel.read(buffers)
    header.flip();  payload.flip();  trailer.flip()
    println(s"header:  ${header.get}, ${header.get}, " +
            s"${header.get}, ${header.get}")
    println(s"payload: ${payload.get}, ${payload.get}, " +
            s"${payload.get}, ${payload.get}")
    println(s"trailer: ${trailer.get}, ${trailer.get}, " +
            s"${trailer.get}, ${trailer.get}")
  finally
    channel.close()
    Files.delete(path)
end main

import java.nio.file.Files
```

Scatter IO is ideal for fixed-format binary protocols where a message  
consists of a header, variable-length body, and trailer. The JVM  
translates the buffer array into a single `readv` syscall on POSIX  
systems, avoiding multiple round-trips. Each buffer is filled  
completely before the next one receives data.  

## Gathering writes

A gathering write drains an array of buffers to a channel in one  
logical operation.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path = Paths.get("/tmp/gather.bin")

  val header  = ByteBuffer.wrap("HEADER|".getBytes("UTF-8"))
  val body    = ByteBuffer.wrap("Hello, NIO.2!".getBytes("UTF-8"))
  val trailer = ByteBuffer.wrap("|TRAILER".getBytes("UTF-8"))
  val buffers = Array[java.nio.ByteBuffer](header, body, trailer)

  val channel = FileChannel.open(
    path,
    StandardOpenOption.WRITE,
    StandardOpenOption.CREATE,
    StandardOpenOption.TRUNCATE_EXISTING
  )
  try
    val written = channel.write(buffers)
    println(s"total bytes written: $written")
  finally
    channel.close()

  println(Files.readString(path))
  Files.delete(path)
end main
```

The `write(ByteBuffer[])` overload maps to the POSIX `writev` syscall  
and sends all non-empty buffers atomically from the OS perspective. This  
is especially useful when assembling protocol frames: each logical  
section lives in its own buffer, avoiding the cost of copying them into  
a single contiguous array. Buffers are drained left-to-right, and the  
channel position advances by the total bytes transferred.  

## Using ByteBuffer (allocate, wrap)

`ByteBuffer.allocate` creates heap storage; `wrap` uses an existing  
array directly.  

```scala
import java.nio.ByteBuffer

@main def main() =
  // allocate: JVM-managed heap buffer
  val alloc = ByteBuffer.allocate(16)
  alloc.putInt(42)
  alloc.putDouble(3.14)
  alloc.flip()
  println(s"int   : ${alloc.getInt}")
  println(s"double: ${alloc.getDouble}")

  // wrap: backed by an existing array
  val data = Array[Byte](1, 2, 3, 4, 5, 6, 7, 8)
  val wrapped = ByteBuffer.wrap(data, 2, 4) // offset=2, length=4
  println(s"position: ${wrapped.position}, limit: ${wrapped.limit}")
  println(s"first byte in window: ${wrapped.get}")

  // Mutation through the buffer is reflected in the array
  wrapped.put(0, 99.toByte)
  println(s"data(0): ${data(0)}")
end main
```

`allocate` gives the JVM freedom to place the buffer anywhere on the  
heap; the backing array is accessible via `.array()`. `wrap` is a  
zero-copy operation: the buffer is a view over the supplied array,  
so writes through the buffer immediately update the array and vice  
versa. The two-argument overload of `wrap` sets the initial position  
and limit within the array, enabling sub-array views without copying.  

## Flipping, clearing, and compacting buffers

These three state-transition methods are the heart of the  
read–process–refill pattern.  

```scala
import java.nio.ByteBuffer

@main def main() =
  val buf = ByteBuffer.allocate(8)

  // Write three bytes
  buf.put(1.toByte).put(2.toByte).put(3.toByte)
  println(s"after puts   pos=${buf.position} lim=${buf.limit}")

  // Prepare to drain
  buf.flip()
  println(s"after flip   pos=${buf.position} lim=${buf.limit}")
  println(s"first two: ${buf.get}, ${buf.get}")

  // compact: copy unread bytes to front
  buf.compact()
  println(s"after compact pos=${buf.position} lim=${buf.limit}")
  println(s"remaining unread byte at 0: ${buf.get(0)}")

  // clear: discard everything
  buf.clear()
  println(s"after clear  pos=${buf.position} lim=${buf.limit}")
end main
```

`flip` is called after writing data into the buffer to prepare it  
for reading: limit moves to the current position and position resets  
to zero. `compact` is the partial-drain variant: unread bytes shift  
to index zero, and position is placed just after them so new data  
can be appended immediately. `clear` simply resets the cursors  
without erasing bytes — data written before `clear` is still  
readable by index but will be overwritten by the next fill.  

## Reading into multiple buffers

Scattering an HTTP-like message into separate header and body  
buffers demonstrates a practical protocol-parsing pattern.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path = Paths.get("/tmp/multi-read.bin")

  // Create a test file: 8-byte header + 16-byte body
  val testData = Array.tabulate[Byte](24)(i => i.toByte)
  Files.write(path, testData)

  val headerBuf = ByteBuffer.allocate(8)
  val bodyBuf   = ByteBuffer.allocate(16)
  val targets   = Array[java.nio.ByteBuffer](headerBuf, bodyBuf)

  val ch = FileChannel.open(path, StandardOpenOption.READ)
  try
    ch.read(targets)
  finally
    ch.close()

  headerBuf.flip()
  bodyBuf.flip()

  val header = Array.ofDim[Byte](8)
  headerBuf.get(header)
  println(s"header bytes: ${header.mkString("[", ", ", "]")}")

  val body = Array.ofDim[Byte](16)
  bodyBuf.get(body)
  println(s"body bytes  : ${body.mkString("[", ", ", "]")}")

  Files.delete(path)
end main
```

Using separate buffers for message sections avoids parsing offsets  
into a single large array. Each buffer acts as a typed region with  
its own position and limit, simplifying downstream processing. The  
scatter read fills the header buffer completely before starting on  
the body, which guarantees a clean boundary even if the underlying  
channel delivers data in chunks.  

## Writing from multiple buffers

Gathering a response frame from header, status, and body buffers  
shows how gather writes compose protocol sections.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path = Paths.get("/tmp/multi-write.dat")

  val magic   = ByteBuffer.wrap(Array[Byte](0xDE.toByte,
                                            0xAD.toByte,
                                            0xBE.toByte,
                                            0xEF.toByte))
  val length  = ByteBuffer.allocate(4)
  val payload = ByteBuffer.wrap("SCALA-NIO2".getBytes("UTF-8"))

  length.putInt(payload.limit)
  length.flip()

  val ch = FileChannel.open(
    path,
    StandardOpenOption.WRITE,
    StandardOpenOption.CREATE,
    StandardOpenOption.TRUNCATE_EXISTING
  )
  try
    val n = ch.write(Array[java.nio.ByteBuffer](magic, length, payload))
    println(s"wrote $n bytes")
  finally
    ch.close()

  val raw = Files.readAllBytes(path)
  println(raw.mkString("[", ", ", "]"))
  Files.delete(path)
end main
```

Building the `length` field from the payload's actual limit prevents  
hard-coded sizes that diverge when strings change. The gather write  
sends all three buffers in one kernel call, preserving the binary  
layout without merging them into a single array. This technique is  
common in binary serialisation frameworks where different sections  
have independent life cycles in memory.  

## Direct vs non-direct buffers

Direct buffers bypass the JVM heap on IO paths, trading allocation  
cost for per-operation speed.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path = Paths.get("/tmp/direct-test.bin")
  Files.write(path, Array.tabulate[Byte](4096)(_.toByte))

  def readWith(direct: Boolean): Long =
    val buf =
      if direct then ByteBuffer.allocateDirect(4096)
      else ByteBuffer.allocate(4096)
    val ch = FileChannel.open(path, StandardOpenOption.READ)
    val t0 = System.nanoTime()
    var n = 0L
    var r = ch.read(buf)
    while r != -1 do
      n += r
      buf.clear()
      r = ch.read(buf)
    val elapsed = System.nanoTime() - t0
    ch.close()
    println(f"direct=$direct  $n bytes  ${elapsed}%,d ns")
    elapsed

  readWith(false)
  readWith(true)
  Files.delete(path)
end main
```

A direct buffer is allocated outside the JVM heap using native memory.  
When a direct buffer is passed to a channel, the JVM can transfer bytes  
straight from the kernel page cache to the buffer without copying through  
a temporary heap array. The benefit is most visible for large reads or  
writes that run in tight loops. The trade-off is that  
`allocateDirect` is slower than `allocate` and the memory is not subject  
to garbage collection in the normal sense.  

## Buffer slicing

`slice` creates a view over a sub-region of a buffer with its own  
independent position and limit.  

```scala
import java.nio.ByteBuffer

@main def main() =
  val original = ByteBuffer.allocate(16)
  for i <- 0 until 16 do original.put(i.toByte)

  // Create a slice starting at byte 4, length 8
  original.position(4)
  original.limit(12)
  val slice = original.slice()

  println(s"slice capacity: ${slice.capacity}")
  println(s"slice position: ${slice.position}")
  println(s"slice limit   : ${slice.limit}")

  // Modify through the slice, observe in original
  slice.put(0, 0xFF.toByte)
  original.position(0)
  original.limit(16)
  println(s"original[4] = ${original.get(4) & 0xFF}")
end main
```

A slice shares the same backing array as the original; modifying bytes  
through either view immediately affects the other. Slices are useful for  
partitioning a large shared buffer into independent worker regions  
without copying. Crucially, each slice has its own position and limit,  
so advancing one does not affect the other.  

## Buffer duplication

`duplicate` creates a second independent cursor pair over the same  
backing storage, enabling two readers to walk the buffer at different  
speeds.  

```scala
import java.nio.ByteBuffer

@main def main() =
  val source = ByteBuffer.allocate(8)
  source.put(Array[Byte](10, 20, 30, 40, 50, 60, 70, 80))
  source.flip()

  val dup = source.duplicate()

  // Reader 1 advances quickly
  println(s"reader1 first: ${source.get}, ${source.get}")

  // Reader 2 sees the same data independently
  println(s"reader2 first: ${dup.get}")

  // Write through source changes dup's view
  source.put(1, 99.toByte)
  dup.position(1)
  println(s"dup sees write: ${dup.get}")
end main
```

Unlike `slice`, `duplicate` copies the current position and limit from  
the original at the moment of the call. Both views remain connected to  
the same backing array, so mutations are visible across all duplicates.  
A common use-case is reading the same header from a buffer in two  
different parts of a decode pipeline without resetting the primary  
cursor.  

---

## Mapping a file into memory

`FileChannel.map` projects a region of a file into the virtual address  
space as a `MappedByteBuffer`.  

```scala
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path = Paths.get("/tmp/mapped.dat")
  Files.write(path, Array.tabulate[Byte](1024)(i => (i % 128).toByte))

  val channel = FileChannel.open(
    path,
    StandardOpenOption.READ,
    StandardOpenOption.WRITE
  )
  try
    val mapped = channel.map(
      FileChannel.MapMode.READ_WRITE,
      0L,
      channel.size
    )
    println(s"capacity  : ${mapped.capacity}")
    println(s"isLoaded  : ${mapped.isLoaded}")
    mapped.load()
    println(s"isLoaded  : ${mapped.isLoaded}")
    println(s"byte[0]   : ${mapped.get(0)}")
  finally
    channel.close()
  Files.delete(path)
end main
```

`map` creates a view of exactly `size` bytes starting at `position`  
in the file. The resulting `MappedByteBuffer` lets you read and write  
through memory addresses instead of system calls. `load` hints to the OS  
to pre-fault all pages into RAM, which can improve latency for random-  
access workloads at the cost of an upfront delay. The mapping persists  
until the buffer is garbage-collected; calling `channel.close` does not  
unmap it.  

## Reading from a mapped buffer

Reading a large binary file through a mapping is efficient because the  
OS handles paging without explicit read syscalls.  

```scala
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path = Paths.get("/tmp/read-mapped.dat")
  val size = 8192
  Files.write(path, Array.tabulate[Byte](size)(i => (i % 256).toByte))

  val ch = FileChannel.open(path, StandardOpenOption.READ)
  val buf = ch.map(FileChannel.MapMode.READ_ONLY, 0, size)
  ch.close()

  var sum = 0L
  while buf.hasRemaining do
    sum += buf.get & 0xFF

  println(s"sum of all bytes: $sum")
  Files.delete(path)
end main
```

`READ_ONLY` mode prevents accidental modifications and allows the OS to  
share pages between processes mapping the same file. The `& 0xFF` mask  
converts the signed `Byte` to an unsigned value before accumulation.  
Because the mapping lives in the page cache, repeated reads of the same  
region hit RAM rather than disk after the first access, making  
sequential scans of large files extremely fast.  

## Writing to a mapped buffer

Writing through a `MappedByteBuffer` updates file content directly  
via the page cache.  

```scala
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path = Paths.get("/tmp/write-mapped.dat")
  val size = 256

  // Create or truncate the file to the desired size first
  val raf = FileChannel.open(
    path,
    StandardOpenOption.READ,
    StandardOpenOption.WRITE,
    StandardOpenOption.CREATE,
    StandardOpenOption.TRUNCATE_EXISTING
  )
  raf.write(java.nio.ByteBuffer.wrap(Array.ofDim[Byte](size)))

  val buf = raf.map(FileChannel.MapMode.READ_WRITE, 0, size)

  for i <- 0 until size do
    buf.put(i, (i * 2 % 256).toByte)

  buf.force()
  raf.close()

  val written = Files.readAllBytes(path)
  println(s"first 8 bytes: ${written.take(8).mkString(", ")}")
  Files.delete(path)
end main
```

Writes through a `MappedByteBuffer` are immediately visible in memory  
but may remain in the kernel's page cache. `force()` explicitly flushes  
dirty pages to the underlying storage device. Because writes go through  
virtual memory rather than the `write` syscall, they are synchronous  
from the programmer's perspective: after `put` the byte is in the buffer,  
and after `force` it is on disk.  

## Forcing changes to disk

`MappedByteBuffer.force` and `FileChannel.force` both flush pending  
writes, but at different levels of granularity.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path = Paths.get("/tmp/force-demo.dat")

  val ch = FileChannel.open(
    path,
    StandardOpenOption.READ,
    StandardOpenOption.WRITE,
    StandardOpenOption.CREATE,
    StandardOpenOption.TRUNCATE_EXISTING
  )

  // Write via channel
  ch.write(ByteBuffer.wrap("channel write\n".getBytes("UTF-8")))

  // Write via mapping
  ch.write(ByteBuffer.wrap(Array.ofDim[Byte](64))) // expand file
  val mapped = ch.map(FileChannel.MapMode.READ_WRITE, 14, 20)
  mapped.put("mapped write  xxxxxx".getBytes("UTF-8"))

  // Force mapping first, then the channel
  mapped.force()         // flush dirty mapping pages
  ch.force(true)         // flush file data + metadata

  println(s"file size: ${ch.size}")
  ch.close()
  Files.delete(path)
end main
```

`force(true)` instructs the OS to flush both the file's data and its  
metadata (modification time, size) to durable storage. Passing `false`  
skips the metadata flush, which is faster but may leave the directory  
entry stale in a crash scenario. When using both a mapping and direct  
channel writes to the same file, call `mapped.force()` before  
`channel.force(true)` to guarantee a consistent flush order.  

## Using mapped files for large datasets

A single mapping can represent a multi-gigabyte index by splitting it  
into fixed-size record windows.  

```scala
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path     = Paths.get("/tmp/large-index.dat")
  val records  = 1_000
  val recSize  = 16  // each record: 8-byte key + 8-byte value

  // Write synthetic records
  val ch = FileChannel.open(
    path,
    StandardOpenOption.READ,
    StandardOpenOption.WRITE,
    StandardOpenOption.CREATE,
    StandardOpenOption.TRUNCATE_EXISTING
  )
  val total = records.toLong * recSize
  ch.write(java.nio.ByteBuffer.wrap(Array.ofDim[Byte](1)), total - 1)

  val buf = ch.map(FileChannel.MapMode.READ_WRITE, 0, total)
  for i <- 0 until records do
    buf.putLong(i.toLong * recSize,       i.toLong * 100)
    buf.putLong(i.toLong * recSize + 8,   i.toLong * 200)

  // Random-access read of record 500
  val key   = buf.getLong(500L * recSize)
  val value = buf.getLong(500L * recSize + 8)
  println(s"record 500: key=$key  value=$value")

  ch.close()
  Files.delete(path)
end main
```

`getLong(absoluteIndex)` is the absolute-position variant that reads  
without moving the buffer cursor, making random-access patterns free  
from cursor management. For files larger than 2 GB (the limit of a  
single `Int`-indexed mapping), the file should be mapped in  
`Integer.MAX_VALUE`-sized windows and navigated with `Long` offsets.  
This technique underpins high-performance databases and search engines  
that need random access over gigabytes of data.  

## Performance considerations for memory-mapped IO

Benchmarking mapped vs channel IO reveals the conditions under which  
each approach excels.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path = Paths.get("/tmp/perf.dat")
  val size = 1 * 1024 * 1024  // 1 MB

  Files.write(path, Array.tabulate[Byte](size)(_.toByte))

  def timeMs(label: String)(block: => Unit): Unit =
    val t0 = System.nanoTime()
    block
    println(f"$label%-25s ${(System.nanoTime()-t0)/1_000_000.0}%.2f ms")

  timeMs("channel read"):
    val ch  = FileChannel.open(path, StandardOpenOption.READ)
    val buf = ByteBuffer.allocateDirect(65536)
    while ch.read(buf) != -1 do buf.clear()
    ch.close()

  timeMs("mmap read"):
    val ch  = FileChannel.open(path, StandardOpenOption.READ)
    val buf = ch.map(FileChannel.MapMode.READ_ONLY, 0, size)
    while buf.hasRemaining do buf.get
    ch.close()

  Files.delete(path)
end main
```

The direct-buffer channel read performs well for sequential access  
because large transfer sizes amortise syscall overhead. Memory-mapped  
IO often wins on warm runs when the OS has already paged the file into  
the page cache, because reads become pure memory operations. Cold reads  
(first access after boot) are similar, as both approaches ultimately  
trigger page faults. Memory-mapped IO shines most for random access  
and for processes that map the same file multiple times without copying.  

---

## Seeking to a position

`FileChannel.position(offset)` moves the read/write cursor  
to an arbitrary byte offset.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path = Paths.get("/tmp/seek.dat")
  Files.write(path, "ABCDEFGHIJKLMNOPQRSTUVWXYZ".getBytes("UTF-8"))

  val ch = FileChannel.open(path, StandardOpenOption.READ)
  try
    // Read first 3 bytes
    val buf = ByteBuffer.allocate(3)
    ch.read(buf)
    buf.flip()
    println(s"start   : ${String(buf.array)}")

    // Seek to byte 10 and read 5 bytes
    ch.position(10)
    buf.clear()
    buf.limit(5)
    ch.read(buf)
    buf.flip()
    println(s"at 10   : ${String(buf.array, 0, buf.limit)}")

    // Seek to end minus 3
    ch.position(ch.size - 3)
    buf.clear()
    buf.limit(3)
    ch.read(buf)
    buf.flip()
    println(s"last 3  : ${String(buf.array, 0, buf.limit)}")
  finally
    ch.close()
  Files.delete(path)
end main
```

`position` returns the channel itself for chaining, and `position()`  
(no argument) reports the current offset. Seeking has zero IO cost;  
the kernel simply adjusts the file offset in its descriptor table.  
Random-access patterns that seek frequently benefit from large buffer  
sizes to amortise the cost of each subsequent read, and from  
`READ_WRITE` mode when mixing reads with in-place updates.  

## Reading at an offset

The two-argument `read(buf, position)` overload reads at an absolute  
offset without disturbing the channel's current position.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path = Paths.get("/tmp/offset-read.dat")
  Files.write(path, "ABCDEFGHIJKLMNOPQRSTUVWXYZ".getBytes("UTF-8"))

  val ch  = FileChannel.open(path, StandardOpenOption.READ)
  val buf = ByteBuffer.allocate(5)

  try
    // Read 5 bytes at offset 5 without moving the channel position
    ch.read(buf, 5L)
    buf.flip()
    println(s"offset 5 : ${String(buf.array, 0, 5)}")
    println(s"position still: ${ch.position}")

    buf.clear()
    ch.read(buf, 20L)
    buf.flip()
    println(s"offset 20: ${String(buf.array, 0, buf.limit)}")
  finally
    ch.close()
  Files.delete(path)
end main
```

`read(buf, pos)` is especially valuable in concurrent scenarios where  
multiple threads read non-overlapping regions of the same file. Because  
the call does not modify the shared channel position, no synchronisation  
is needed between threads reading at different offsets. It mirrors the  
POSIX `pread` syscall and should always be preferred over `seek` + `read`  
when the access pattern is random.  

## Writing at an offset

`write(buf, position)` writes at an absolute offset without seeking.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path = Paths.get("/tmp/offset-write.dat")

  val ch = FileChannel.open(
    path,
    StandardOpenOption.READ,
    StandardOpenOption.WRITE,
    StandardOpenOption.CREATE,
    StandardOpenOption.TRUNCATE_EXISTING
  )

  try
    // Sparse write: leave gaps filled with zero bytes
    ch.write(ByteBuffer.wrap("HEADER".getBytes("UTF-8")), 0L)
    ch.write(ByteBuffer.wrap("MIDDLE".getBytes("UTF-8")), 20L)
    ch.write(ByteBuffer.wrap("FOOTER".getBytes("UTF-8")), 40L)
    println(s"file size: ${ch.size}")

    val all = ByteBuffer.allocate(ch.size.toInt)
    ch.read(all, 0L)
    all.flip()
    val bytes = Array.ofDim[Byte](all.limit)
    all.get(bytes)
    println(bytes.map(b => if b == 0 then '.' else b.toChar).mkString)
  finally
    ch.close()
  Files.delete(path)
end main
```

Sparse writes create a file whose logical size exceeds the number of  
bytes explicitly written. On most file systems, the zero-filled gaps  
are not stored on disk; they are synthesised by the OS on demand (sparse  
file). Absolute-position writes are thread-safe when performed on  
non-overlapping regions, making them the building block for parallel  
file-writing strategies used in download managers and parallel CSV  
exporters.  

## Truncating files

`FileChannel.truncate` shortens a file to a specified size.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path = Paths.get("/tmp/truncate.dat")
  Files.write(path, "0123456789ABCDEF".getBytes("UTF-8"))

  val ch = FileChannel.open(
    path,
    StandardOpenOption.READ,
    StandardOpenOption.WRITE
  )
  try
    println(s"before: ${ch.size} bytes")
    ch.truncate(10)
    println(s"after : ${ch.size} bytes")

    val buf = ByteBuffer.allocate(ch.size.toInt)
    ch.read(buf, 0L)
    buf.flip()
    val bytes = Array.ofDim[Byte](buf.limit)
    buf.get(bytes)
    println(s"content: ${String(bytes, "UTF-8")}")
  finally
    ch.close()
  Files.delete(path)
end main
```

`truncate(n)` discards all bytes beyond position `n`. If the channel  
position was beyond `n`, it is moved back to `n`; otherwise it remains  
unchanged. Truncation is instantaneous from the caller's perspective  
because it only updates the inode metadata; no data is zeroed. This  
operation is commonly used to overwrite log files atomically: write the  
new content to a temporary file, then truncate the original and  
overwrite, or simply rename.  

## Extending files

Writing a single byte at a large offset creates a sparse file,  
effectively extending it.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path = Paths.get("/tmp/extend.dat")
  val ch = FileChannel.open(
    path,
    StandardOpenOption.READ,
    StandardOpenOption.WRITE,
    StandardOpenOption.CREATE,
    StandardOpenOption.TRUNCATE_EXISTING
  )
  try
    println(s"initial size: ${ch.size}")
    // Write one byte at offset 1 MB - 1
    val oneByteAtEnd = ByteBuffer.wrap(Array[Byte](0xFF.toByte))
    ch.write(oneByteAtEnd, 1024L * 1024L - 1)
    println(s"after extend: ${ch.size} bytes")

    // Read the gap — should be zero
    val buf = ByteBuffer.allocate(1)
    ch.read(buf, 512L)
    buf.flip()
    println(s"byte at 512 (gap): ${buf.get & 0xFF}")
  finally
    ch.close()
  Files.delete(path)
end main
```

Positioning a write beyond the current file size atomically extends the  
logical file without writing all intermediate bytes to disk; on most  
file systems the hole is stored as a gap in the inode extent map. Reads  
into the hole return zero bytes. This is the technique used by virtual  
machine disk images and database engines that reserve space upfront  
and fill it lazily.  

## Locking regions of a file

`FileChannel.lock` and `tryLock` coordinate access between competing  
processes at the OS level.  

```scala
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}
import java.nio.ByteBuffer

@main def main() =
  val path = Paths.get("/tmp/lockfile.dat")
  Files.write(path, Array.ofDim[Byte](256))

  val ch = FileChannel.open(
    path,
    StandardOpenOption.READ,
    StandardOpenOption.WRITE
  )

  try
    // Exclusive lock over the first 128 bytes
    val lock = ch.lock(0L, 128L, false)
    try
      println(s"lock acquired: shared=${lock.isShared}")
      ch.write(ByteBuffer.wrap("locked region\n".getBytes("UTF-8")), 0L)
    finally
      lock.release()
      println("lock released")

    // Shared (read) lock over bytes 128-255
    val sharedLock = ch.lock(128L, 128L, true)
    try
      println(s"shared lock: ${sharedLock.isShared}")
    finally
      sharedLock.release()
  finally
    ch.close()
  Files.delete(path)
end main
```

File locks in NIO are advisory on most Unix systems, meaning they only  
work when all cooperating processes use the locking API. An exclusive  
lock (`shared = false`) grants sole write access; multiple holders of  
overlapping shared locks are permitted simultaneously. `tryLock` returns  
`null` instead of blocking when the lock cannot be acquired, making it  
suitable for non-blocking coordination patterns. Locks are automatically  
released when the channel closes.  

---

## Creating a WatchService

A `WatchService` subscribes to kernel-level file-system notifications  
for a directory.  

```scala
import java.nio.file.{FileSystems, StandardWatchEventKinds}

@main def main() =
  val watcher = FileSystems.getDefault.newWatchService()
  println(s"WatchService created: $watcher")

  // The service must be closed to release OS resources
  watcher.close()
  println("WatchService closed")
end main
```

`FileSystems.getDefault.newWatchService()` returns a platform-specific  
implementation: `InotifyWatchService` on Linux, `PollingWatchService`  
as a fallback, and so on. The service itself is a lightweight handle;  
registering directories and polling events are separate operations. It  
implements `Closeable`, so a `Using` block or try/finally is required  
to avoid leaking the underlying OS descriptor.  

## Registering directories

Directories are registered with a `WatchService` by calling  
`Path.register`.  

```scala
import java.nio.file.{FileSystems, Paths, StandardWatchEventKinds => K}

@main def main() =
  val dir     = Paths.get("/tmp")
  val watcher = FileSystems.getDefault.newWatchService()

  val key = dir.register(
    watcher,
    K.ENTRY_CREATE,
    K.ENTRY_MODIFY,
    K.ENTRY_DELETE
  )

  println(s"key valid  : ${key.isValid}")
  println(s"watchable  : ${key.watchable}")

  key.cancel()
  watcher.close()
end main
```

`register` returns a `WatchKey` that identifies this directory–service  
pairing. The vararg event-kind list controls which operations generate  
events; omitting `ENTRY_MODIFY`, for example, silences change  
notifications. `key.cancel()` unregisters the directory before  
closing the service, which is good practice but not strictly required  
since `watcher.close()` invalidates all keys automatically.  

## Watching for create/modify/delete events

A polling loop drains the `WatchService` queue and dispatches each  
event to the appropriate handler.  

```scala
import java.nio.file.{FileSystems, Paths, WatchEvent, StandardWatchEventKinds => K}
import java.util.concurrent.TimeUnit

@main def main() =
  val dir     = Paths.get("/tmp")
  val watcher = FileSystems.getDefault.newWatchService()
  dir.register(watcher, K.ENTRY_CREATE, K.ENTRY_MODIFY, K.ENTRY_DELETE)

  println("watching /tmp for 3 seconds...")

  val deadline = System.currentTimeMillis + 3000
  while System.currentTimeMillis < deadline do
    val key = watcher.poll(200, TimeUnit.MILLISECONDS)
    if key != null then
      for ev <- key.pollEvents.iterator.nn do
        val kind = ev.kind
        val ctx  = ev.context.asInstanceOf[java.nio.file.Path]
        kind match
          case K.ENTRY_CREATE => println(s"CREATED : $ctx")
          case K.ENTRY_MODIFY => println(s"MODIFIED: $ctx")
          case K.ENTRY_DELETE => println(s"DELETED : $ctx")
          case _              => println(s"OTHER   : $ctx")
      key.reset()

  watcher.close()
  println("done")
end main
```

`poll(timeout, unit)` blocks until an event arrives or the timeout  
expires. After processing all events on a key, `reset()` must be  
called to re-arm it; failing to call `reset` permanently invalidates  
the key and stops future notifications for that directory.  
`pollEvents` returns an unmodifiable list; iterating with a Java  
`Iterator` avoids the Scala/Java collection conversion overhead.  

## Handling overflow events

An `OVERFLOW` event indicates that some events were dropped because  
the kernel queue filled up.  

```scala
import java.nio.file.{FileSystems, Paths, StandardWatchEventKinds => K}
import java.util.concurrent.TimeUnit

@main def main() =
  val dir     = Paths.get("/tmp")
  val watcher = FileSystems.getDefault.newWatchService()
  dir.register(watcher, K.ENTRY_CREATE, K.ENTRY_MODIFY,
                         K.ENTRY_DELETE, K.OVERFLOW)

  println("watching /tmp for 2 seconds (overflow aware)...")
  val deadline = System.currentTimeMillis + 2000

  while System.currentTimeMillis < deadline do
    val key = watcher.poll(100, TimeUnit.MILLISECONDS)
    if key != null then
      for ev <- key.pollEvents.iterator.nn do
        ev.kind match
          case K.OVERFLOW =>
            println(s"OVERFLOW! count=${ev.count} — " +
                    "rescanning directory")
          case k =>
            println(s"$k: ${ev.context}")
      key.reset()

  watcher.close()
end main
```

`OVERFLOW` events carry a count indicating how many events were lost.  
When an overflow occurs, the correct response is to rescan the watched  
directory from scratch to restore a consistent view of its contents.  
The overflow threshold depends on the platform and kernel version;  
production watcher implementations should always register for  
`OVERFLOW` and treat it as a trigger for a full directory snapshot.  

## Watching multiple directories

Registering several directories with a single `WatchService` instance  
lets one thread monitor an entire subtree.  

```scala
import java.nio.file.{FileSystems, Files, Paths, Path,
                      StandardWatchEventKinds => K}
import java.util.concurrent.TimeUnit
import scala.jdk.CollectionConverters.*

@main def main() =
  val root    = Paths.get("/tmp")
  val watcher = FileSystems.getDefault.newWatchService()
  val keys    = collection.mutable.Map.empty[java.nio.file.WatchKey, Path]

  // Register /tmp and all its immediate subdirectories
  val dirs = Files.walk(root, 1)
    .filter(Files.isDirectory(_))
    .toList.asScala

  for dir <- dirs do
    val key = dir.register(watcher, K.ENTRY_CREATE,
                                    K.ENTRY_MODIFY,
                                    K.ENTRY_DELETE)
    keys(key) = dir

  println(s"watching ${keys.size} directories...")
  val deadline = System.currentTimeMillis + 2000

  while System.currentTimeMillis < deadline do
    val k = watcher.poll(200, TimeUnit.MILLISECONDS)
    if k != null then
      val dir = keys.getOrElse(k, Paths.get("?"))
      for ev <- k.pollEvents.iterator.nn do
        println(s"[${ev.kind.name}] $dir/${ev.context}")
      k.reset()

  watcher.close()
end main
```

Mapping `WatchKey` → `Path` is the standard way to identify which  
directory triggered an event, because the event's context is only the  
filename component. When a new subdirectory is created during  
watching, it should be registered dynamically in the `ENTRY_CREATE`  
handler to maintain complete coverage.  

## Building a simple file watcher

A reusable `FileWatcher` encapsulates the event loop and dispatches  
to a user-supplied callback.  

```scala
import java.nio.file.{FileSystems, Paths, WatchEvent, Path,
                      StandardWatchEventKinds => K}
import java.util.concurrent.{TimeUnit, Executors}

class FileWatcher(dir: Path, handler: (String, Path) => Unit):
  private val watcher = FileSystems.getDefault.newWatchService()
  private val key     = dir.register(watcher,
                          K.ENTRY_CREATE, K.ENTRY_MODIFY, K.ENTRY_DELETE)
  private val exec    = Executors.newSingleThreadExecutor()

  def start(): Unit =
    exec.submit(new Runnable:
      def run(): Unit =
        var running = true
        while running do
          val k = watcher.poll(500, TimeUnit.MILLISECONDS)
          if k != null then
            for ev <- k.pollEvents.iterator.nn do
              handler(ev.kind.name,
                      dir.resolve(ev.context.asInstanceOf[Path]))
            if !k.reset() then running = false
    )

  def stop(): Unit =
    key.cancel()
    watcher.close()
    exec.shutdown()

@main def main() =
  val watcher = FileWatcher(Paths.get("/tmp"),
    (event, path) => println(s"[$event] $path"))

  watcher.start()
  Thread.sleep(3000)
  watcher.stop()
  println("watcher stopped")
end main
```

The `FileWatcher` class separates concerns: the constructor sets up  
resources, `start` launches the polling loop on a dedicated thread, and  
`stop` cleans up in the correct order. Checking the return value of  
`key.reset()` handles the case where the watched directory itself is  
deleted, stopping the loop gracefully instead of spinning indefinitely.  

## Debouncing rapid events

Rapid saves trigger many `ENTRY_MODIFY` events; debouncing collapses  
them into a single notification.  

```scala
import java.nio.file.{FileSystems, Paths, Path, StandardWatchEventKinds => K}
import java.util.concurrent.{TimeUnit, Executors, ScheduledFuture}
import scala.collection.mutable

@main def main() =
  val dir     = Paths.get("/tmp")
  val watcher = FileSystems.getDefault.newWatchService()
  dir.register(watcher, K.ENTRY_MODIFY)

  val scheduler = Executors.newSingleThreadScheduledExecutor()
  val pending   = mutable.Map.empty[Path, ScheduledFuture[?]]
  val delayMs   = 300L

  println("debouncing modify events for 5 seconds...")
  val deadline = System.currentTimeMillis + 5000

  while System.currentTimeMillis < deadline do
    val key = watcher.poll(100, TimeUnit.MILLISECONDS)
    if key != null then
      for ev <- key.pollEvents.iterator.nn do
        val p = dir.resolve(ev.context.asInstanceOf[Path])
        pending.get(p).foreach(_.cancel(false))
        pending(p) = scheduler.schedule(
          new Runnable:
            def run() =
              println(s"DEBOUNCED MODIFY: $p")
              pending.remove(p)
          ,
          delayMs, TimeUnit.MILLISECONDS
        )
      key.reset()

  watcher.close()
  scheduler.shutdown()
end main
```

A `ScheduledFuture` acts as a timer: each new event for the same path  
cancels the previous timer and starts a fresh one. The handler only  
fires after `delayMs` milliseconds of silence. This is identical to  
the debounce pattern used in UI frameworks and avoids flooding downstream  
processors with thousands of events when an editor writes a file  
in many small increments.  

---

## Opening a SocketChannel

`SocketChannel.open` creates an unconnected TCP channel.  

```scala
import java.nio.channels.SocketChannel
import java.net.InetSocketAddress

@main def main() =
  val ch = SocketChannel.open()
  println(s"open        : ${ch.isOpen}")
  println(s"connected   : ${ch.isConnected}")
  println(s"blocking    : ${ch.isBlocking}")

  // Set options before connecting
  ch.setOption(java.net.StandardSocketOptions.TCP_NODELAY, true)
  ch.setOption(java.net.StandardSocketOptions.SO_KEEPALIVE, true)
  println(s"TCP_NODELAY : ${ch.getOption(
              java.net.StandardSocketOptions.TCP_NODELAY)}")

  ch.close()
  println(s"open after close: ${ch.isOpen}")
end main
```

`SocketChannel.open()` without an address argument creates the channel  
without initiating a connection, giving the caller time to configure  
socket options. `TCP_NODELAY` disables Nagle's algorithm for  
low-latency messaging; `SO_KEEPALIVE` instructs the OS to send periodic  
probes to detect dead peers. All configuration must happen before  
`connect` because some options are immutable once the handshake begins.  

## Connecting to a server

`SocketChannel.connect` initiates a TCP handshake to a remote address.  

```scala
import java.nio.channels.SocketChannel
import java.net.InetSocketAddress

@main def main() =
  // Connect to a public HTTP server (echo service or httpbin)
  val addr = InetSocketAddress("httpbin.org", 80)
  val ch   = SocketChannel.open()

  try
    val connected = ch.connect(addr)
    if !connected then
      while !ch.finishConnect() do
        Thread.sleep(10)

    println(s"connected   : ${ch.isConnected}")
    println(s"remote addr : ${ch.getRemoteAddress}")
    println(s"local  addr : ${ch.getLocalAddress}")
  catch
    case e: java.io.IOException =>
      println(s"connection failed: ${e.getMessage}")
  finally
    ch.close()
end main
```

In blocking mode `connect` returns `true` only when the handshake  
completes; it always returns `true` for blocking channels. In  
non-blocking mode it may return `false`, requiring the caller to poll  
`finishConnect()` until it returns `true`. `getRemoteAddress` and  
`getLocalAddress` are useful for logging and diagnostics, especially  
in multi-homed hosts where the OS selects the outgoing interface.  

## Reading and writing from a SocketChannel

After connecting, a `SocketChannel` can be used like any other  
`ReadableByteChannel` and `WritableByteChannel`.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.SocketChannel
import java.net.InetSocketAddress

@main def main() =
  val ch = SocketChannel.open()
  try
    ch.connect(InetSocketAddress("httpbin.org", 80))

    val request = "GET /get HTTP/1.0\r\nHost: httpbin.org\r\n\r\n"
    val reqBuf  = ByteBuffer.wrap(request.getBytes("UTF-8"))
    while reqBuf.hasRemaining do ch.write(reqBuf)

    val respBuf = ByteBuffer.allocate(4096)
    val sb      = StringBuilder()
    var n       = ch.read(respBuf)
    while n > 0 do
      respBuf.flip()
      val bytes = Array.ofDim[Byte](respBuf.limit)
      respBuf.get(bytes)
      sb.append(String(bytes, "UTF-8"))
      respBuf.clear()
      n = ch.read(respBuf)

    println(sb.toString.take(300))
  catch
    case e: Exception => println(s"error: ${e.getMessage}")
  finally
    ch.close()
end main
```

The write loop accounts for the possibility that a single `write` call  
does not drain the entire buffer, which can happen when the socket send  
buffer is full. The read loop accumulates the response into a  
`StringBuilder`, flipping and clearing the buffer between reads.  
`read` returns `0` when no data is immediately available in  
non-blocking mode and `-1` at end-of-stream (connection close).  

## Non-blocking mode

Switching a channel to non-blocking mode makes `read` and `write`  
return immediately even when no data is available.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.SocketChannel
import java.net.InetSocketAddress

@main def main() =
  val ch = SocketChannel.open()
  ch.configureBlocking(false)

  println(s"blocking: ${ch.isBlocking}")

  // connect returns false in non-blocking mode
  val immediate = ch.connect(InetSocketAddress("httpbin.org", 80))
  println(s"connect immediate: $immediate")

  // Poll until handshake completes (up to 3 s)
  val deadline = System.currentTimeMillis + 3000
  while !ch.isConnected && System.currentTimeMillis < deadline do
    if ch.finishConnect() then println("handshake done")
    Thread.sleep(20)

  if ch.isConnected then
    val buf = ByteBuffer.allocate(64)
    val n   = ch.read(buf)
    println(s"immediate read returned: $n bytes")  // likely 0

  ch.close()
end main
```

`configureBlocking(false)` puts the channel into non-blocking mode.  
In this mode `connect` starts the TCP handshake but returns `false`  
before it finishes; `finishConnect` must be polled or the channel must  
be registered with a `Selector` on `OP_CONNECT`. A non-blocking `read`  
that finds no data returns `0` rather than blocking, which is why  
multiplexing via `Selector` is mandatory for correct event-driven code.  

## Configuring blocking vs non-blocking

Switching between blocking and non-blocking during a channel's  
lifetime is valid and sometimes useful for initialisation.  

```scala
import java.nio.channels.SocketChannel
import java.net.InetSocketAddress

@main def main() =
  val ch = SocketChannel.open()

  // Initial connection in blocking mode for simplicity
  ch.connect(InetSocketAddress("httpbin.org", 80))
  println(s"connected in blocking: ${ch.isConnected}")

  // Switch to non-blocking for event-driven IO
  ch.configureBlocking(false)
  println(s"now blocking: ${ch.isBlocking}")

  // Demonstrate reconfiguration back
  ch.configureBlocking(true)
  println(s"back to blocking: ${ch.isBlocking}")

  ch.close()
end main
```

Reconfiguring a channel between modes is legal as long as it is not  
currently registered with a `Selector`; attempting to change the mode  
of a registered channel throws `IllegalBlockingModeException`. The  
pattern of connecting in blocking mode and then switching to non-  
blocking is common in connection-pool implementations where the  
connection setup latency matters but IO throughput is handled by a  
selector loop.  

## ServerSocketChannel basics

`ServerSocketChannel` binds to a local port and accepts incoming  
TCP connections.  

```scala
import java.nio.channels.ServerSocketChannel
import java.net.InetSocketAddress

@main def main() =
  val server = ServerSocketChannel.open()
  server.bind(InetSocketAddress(9090))

  println(s"bound to  : ${server.getLocalAddress}")
  println(s"blocking  : ${server.isBlocking}")
  println(s"open      : ${server.isOpen}")

  // In a real server, call server.accept() here in a loop
  server.close()
  println(s"open after close: ${server.isOpen}")
end main
```

`bind` associates the channel with a local port. Passing `null` or  
an address with port `0` lets the OS choose a free port, which is  
useful for tests and dynamic services. The backlog defaults to 50  
pending connections; pass a second integer to `bind` to override it.  
Like all channels, `ServerSocketChannel` implements `Closeable` and  
must be closed to release the port.  

## Accepting connections

`accept` returns a connected `SocketChannel` for each client,  
or `null` in non-blocking mode when no clients are waiting.  

```scala
import java.nio.channels.{ServerSocketChannel, SocketChannel}
import java.nio.ByteBuffer
import java.net.InetSocketAddress
import java.util.concurrent.Executors

@main def main() =
  val server = ServerSocketChannel.open()
  server.bind(InetSocketAddress(9091))
  println(s"server listening on ${server.getLocalAddress}")

  val pool = Executors.newCachedThreadPool()

  // Accept one client for demonstration, then stop
  pool.submit(new Runnable:
    def run() =
      val client: SocketChannel = server.accept()
      println(s"accepted: ${client.getRemoteAddress}")
      val buf = ByteBuffer.wrap("Hello from server\n".getBytes("UTF-8"))
      client.write(buf)
      client.close()
  )

  // Connect a client
  Thread.sleep(100)
  val ch = SocketChannel.open(InetSocketAddress("localhost", 9091))
  val buf = ByteBuffer.allocate(64)
  ch.read(buf)
  buf.flip()
  val msg = Array.ofDim[Byte](buf.limit)
  buf.get(msg)
  println(s"client received: ${String(msg, "UTF-8")}")
  ch.close()

  pool.shutdown()
  server.close()
end main
```

`accept` in blocking mode waits indefinitely for the next client.  
In non-blocking mode it returns `null` immediately when no client is  
queued. The accepted `SocketChannel` inherits the default blocking  
mode from the server channel; configure it explicitly before handing  
it to a worker thread. The server thread pool is unbounded here for  
simplicity; a production server would use a fixed pool or virtual  
threads.  

## Reading from multiple clients

Spawning a thread per client is the simplest concurrent server  
strategy, albeit wasteful for thousands of connections.  

```scala
import java.nio.channels.{ServerSocketChannel, SocketChannel}
import java.nio.ByteBuffer
import java.net.InetSocketAddress
import java.util.concurrent.Executors

def handleClient(ch: SocketChannel): Unit =
  val buf = ByteBuffer.allocate(256)
  var n   = ch.read(buf)
  while n > 0 do
    buf.flip()
    val echo = buf.duplicate()
    ch.write(echo)
    buf.clear()
    n = ch.read(buf)
  ch.close()

@main def main() =
  val server = ServerSocketChannel.open()
  server.bind(InetSocketAddress(9092))
  val pool = Executors.newFixedThreadPool(4)

  // Accept 3 synthetic clients
  var accepted = 0
  while accepted < 3 do
    val client = server.accept()
    if client != null then
      accepted += 1
      pool.submit(new Runnable:
        def run() = handleClient(client))

  pool.shutdown()
  server.close()
end main
```

Each accepted channel is dispatched to a pool worker via a `Runnable`  
closure. The `handleClient` function implements a simple echo protocol:  
it reads a chunk into the buffer, duplicates the buffer to avoid  
modifying the read position, and writes it back. This pattern scales  
to a few hundred concurrent connections before thread overhead becomes  
a bottleneck; beyond that, a `Selector`-based approach is required.  

## Using Selectors for multiplexing

A `Selector` multiplexes many non-blocking channels over a single  
thread by polling readiness.  

```scala
import java.nio.channels.{Selector, SelectionKey, ServerSocketChannel}
import java.net.InetSocketAddress

@main def main() =
  val selector = Selector.open()
  val server   = ServerSocketChannel.open()
  server.bind(InetSocketAddress(9093))
  server.configureBlocking(false)

  val key = server.register(selector, SelectionKey.OP_ACCEPT)
  println(s"registered with selector, key valid: ${key.isValid}")

  // One iteration to show the select call
  val ready = selector.select(500) // block up to 500 ms
  println(s"channels ready: $ready")

  key.cancel()
  selector.close()
  server.close()
end main
```

`Selector.open()` creates a platform-specific multiplexer backed by  
`epoll` on Linux, `kqueue` on macOS, or `select` on older systems.  
`select(timeout)` blocks until at least one registered channel becomes  
ready or the timeout elapses, returning the count of ready channels.  
Channels must be in non-blocking mode before registration; attempting  
to register a blocking channel throws `IllegalBlockingModeException`.  

## Registering channels with a Selector

Each channel is registered with one or more interest operations that  
define which readiness events should wake the selector.  

```scala
import java.nio.channels.{Selector, SelectionKey,
                           ServerSocketChannel, SocketChannel}
import java.net.InetSocketAddress

@main def main() =
  val selector = Selector.open()

  // Server channel: interested in accepting new connections
  val server = ServerSocketChannel.open()
  server.bind(InetSocketAddress(9094))
  server.configureBlocking(false)
  val serverKey = server.register(selector, SelectionKey.OP_ACCEPT)
  println(s"server key ops: ${serverKey.interestOps}")

  // Client channel: interested in connecting
  val client = SocketChannel.open()
  client.configureBlocking(false)
  client.connect(InetSocketAddress("localhost", 9094))
  val clientKey = client.register(selector,
    SelectionKey.OP_CONNECT | SelectionKey.OP_READ)
  println(s"client key ops: ${clientKey.interestOps}")

  println(s"total keys: ${selector.keys.size}")

  serverKey.cancel()
  clientKey.cancel()
  selector.close()
  client.close()
  server.close()
end main
```

`interestOps` is a bitmask built from `OP_ACCEPT`, `OP_CONNECT`,  
`OP_READ`, and `OP_WRITE` constants on `SelectionKey`. After  
registration, the interest set can be updated via  
`key.interestOps(newMask)` to change which events wake the selector —  
for example, enabling `OP_WRITE` only when there is data to send  
avoids spinning on always-writable sockets.  

## Handling selected keys

`selectedKeys` returns the set of channels that became ready since  
the last `select` call.  

```scala
import java.nio.channels.{Selector, SelectionKey, ServerSocketChannel,
                           SocketChannel}
import java.nio.ByteBuffer
import java.net.InetSocketAddress

@main def main() =
  val selector = Selector.open()
  val server   = ServerSocketChannel.open()
  server.bind(InetSocketAddress(9095))
  server.configureBlocking(false)
  server.register(selector, SelectionKey.OP_ACCEPT)

  // Spawn a client that connects after a short delay
  val t = Thread(() =>
    Thread.sleep(50)
    SocketChannel.open(InetSocketAddress("localhost", 9095)).close()
  )
  t.setDaemon(true)
  t.start()

  selector.select(1000)
  val it = selector.selectedKeys.iterator
  while it.hasNext do
    val key = it.next()
    it.remove() // MUST remove before processing
    if key.isAcceptable then
      val ch = server.accept()
      println(s"accepted: ${ch.getRemoteAddress}")
      ch.close()

  selector.close()
  server.close()
end main
```

`it.remove()` must be called for each key before processing it;  
`selectedKeys` is a mutable set and the selector never removes keys  
automatically. Forgetting `remove()` means the same key is processed  
again on the next iteration, causing duplicate handling. The  
`isAcceptable`, `isReadable`, `isWritable`, and `isConnectable`  
convenience methods test the ready set without masking manually.  

## Building a simple non-blocking TCP server

Combining selector and channel APIs produces a complete single-  
threaded echo server that handles many clients concurrently.  

```scala
import java.nio.channels.{Selector, SelectionKey, ServerSocketChannel,
                           SocketChannel}
import java.nio.ByteBuffer
import java.net.InetSocketAddress

@main def main() =
  val selector = Selector.open()
  val server   = ServerSocketChannel.open()
  server.bind(InetSocketAddress(9096))
  server.configureBlocking(false)
  server.register(selector, SelectionKey.OP_ACCEPT)

  println(s"echo server on ${server.getLocalAddress}")

  val stop = System.currentTimeMillis + 4000

  while System.currentTimeMillis < stop do
    selector.select(200)
    val it = selector.selectedKeys.iterator
    while it.hasNext do
      val key = it.next()
      it.remove()

      if key.isAcceptable then
        val client = server.accept()
        client.configureBlocking(false)
        client.register(selector, SelectionKey.OP_READ)
        println(s"connected: ${client.getRemoteAddress}")

      else if key.isReadable then
        val ch  = key.channel.asInstanceOf[SocketChannel]
        val buf = ByteBuffer.allocate(256)
        val n   = ch.read(buf)
        if n == -1 then
          println(s"client disconnected")
          ch.close()
          key.cancel()
        else
          buf.flip()
          ch.write(buf)  // echo back

  selector.close()
  server.close()
  println("server stopped")
end main
```

The single event loop alternates between accepting new connections and  
echoing data from existing ones. `OP_ACCEPT` fires only on the server  
channel; `OP_READ` fires on each connected client when data arrives.  
Detecting `n == -1` from `read` signals a clean client disconnect,  
after which the channel is closed and the key cancelled to remove it  
from the selector. This architecture supports thousands of concurrent  
clients on a single thread because no thread ever blocks waiting for  
a specific client.  

---

## Choosing buffer sizes

Buffer size is a key tuning parameter: too small wastes syscall  
overhead; too large wastes heap memory.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path = Paths.get("/tmp/bufsize.dat")
  Files.write(path, Array.tabulate[Byte](1024 * 1024)(_.toByte))

  def readWith(bufSize: Int): Long =
    val ch  = FileChannel.open(path, StandardOpenOption.READ)
    val buf = ByteBuffer.allocateDirect(bufSize)
    val t0  = System.nanoTime()
    var ops = 0
    while ch.read(buf) != -1 do
      buf.clear()
      ops += 1
    val ns = System.nanoTime() - t0
    ch.close()
    println(f"bufSize=${bufSize}%6d  ops=${ops}%5d  " +
            f"${ns / 1_000_000.0}%.2f ms")
    ns

  for size <- List(512, 4096, 65536, 524288) do
    readWith(size)

  Files.delete(path)
end main
```

Larger buffers reduce the number of `read` syscalls proportionally,  
as shown by the `ops` counter. For sequential file reading, 64 KB to  
256 KB is a common sweet spot that matches typical hardware sector and  
OS page-cache boundaries. Very large buffers (>1 MB) offer diminishing  
returns and increase GC pressure if allocated on the heap. Direct  
buffers at these sizes show the most dramatic improvement over small  
heap buffers.  

## Avoiding unnecessary copies

`transferTo` and `transferFrom` move data between channels without  
routing it through a user-space buffer.  

```scala
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val src  = Paths.get("/tmp/src-copy.dat")
  val dst  = Paths.get("/tmp/dst-copy.dat")
  Files.write(src, Array.tabulate[Byte](256 * 1024)(_.toByte))

  val srcCh = FileChannel.open(src, StandardOpenOption.READ)
  val dstCh = FileChannel.open(
    dst,
    StandardOpenOption.WRITE,
    StandardOpenOption.CREATE,
    StandardOpenOption.TRUNCATE_EXISTING
  )

  try
    val t0 = System.nanoTime()
    var transferred = 0L
    val total = srcCh.size
    while transferred < total do
      transferred += srcCh.transferTo(
        transferred, total - transferred, dstCh)
    val ms = (System.nanoTime() - t0) / 1_000_000.0
    println(f"transferred ${transferred} bytes in ${ms}%.2f ms")
  finally
    srcCh.close()
    dstCh.close()

  Files.delete(src)
  Files.delete(dst)
end main
```

`transferTo` maps to `sendfile(2)` on Linux, which copies data entirely  
inside the kernel, avoiding two buffer copies and two context switches  
compared to the read–write loop. The while loop is necessary because  
`transferTo` may not move the entire file in one call (network sockets  
can block mid-transfer). `transferFrom` is the symmetric operation;  
both accept a count parameter to copy only a sub-range.  

## Using direct buffers for performance

Allocating channel buffers outside the JVM heap reduces copy overhead  
on every IO operation.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path = Paths.get("/tmp/direct-bench.dat")
  val size = 512 * 1024
  Files.write(path, Array.tabulate[Byte](size)(_.toByte))

  def bench(direct: Boolean, iterations: Int): Double =
    val ch  = FileChannel.open(path, StandardOpenOption.READ)
    val buf = if direct then ByteBuffer.allocateDirect(65536)
              else ByteBuffer.allocate(65536)
    val t0  = System.nanoTime()
    for _ <- 1 to iterations do
      ch.position(0)
      buf.clear()
      while ch.read(buf) != -1 do buf.clear()
    val ms = (System.nanoTime() - t0) / 1_000_000.0
    ch.close()
    ms

  // Warm up
  bench(false, 5); bench(true, 5)

  val heapMs   = bench(false, 20)
  val directMs = bench(true, 20)

  println(f"heap   20 iters: ${heapMs}%.1f ms")
  println(f"direct 20 iters: ${directMs}%.1f ms")
  println(f"speedup: ${heapMs / directMs}%.2fx")

  Files.delete(path)
end main
```

When a heap `ByteBuffer` is passed to a channel, the JVM may need to  
copy its contents into a temporary direct buffer before invoking the  
native IO call. Direct buffers eliminate this extra copy by residing  
in memory that can be addressed directly by the kernel's DMA engine.  
The improvement is most visible for repeated reads of the same large  
file and for write-heavy workloads. Allocate direct buffers once and  
reuse them across many operations to amortise the higher allocation  
cost.  

## When to use memory-mapped IO

A comparison of channel vs mapped IO for read-heavy and write-heavy  
workloads guides the choice of strategy.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path = Paths.get("/tmp/mmio-compare.dat")
  val size = 2 * 1024 * 1024

  Files.write(path, Array.tabulate[Byte](size)(_.toByte))

  val ch = FileChannel.open(path, StandardOpenOption.READ,
                                  StandardOpenOption.WRITE)

  def timeMs(label: String)(block: => Unit): Unit =
    val t0 = System.nanoTime()
    block
    println(f"$label%-30s ${(System.nanoTime()-t0)/1e6}%.2f ms")

  timeMs("channel sequential read"):
    val buf = ByteBuffer.allocateDirect(65536)
    ch.position(0)
    while ch.read(buf) != -1 do buf.clear()

  timeMs("mmap sequential read"):
    val m = ch.map(FileChannel.MapMode.READ_ONLY, 0, size)
    while m.hasRemaining do m.getLong

  timeMs("channel random read"):
    val buf = ByteBuffer.allocateDirect(8)
    for i <- 0 until 10_000 do
      val pos = (i.toLong * 199) % (size - 8)
      ch.read(buf, pos)
      buf.clear()

  timeMs("mmap random read"):
    val m = ch.map(FileChannel.MapMode.READ_ONLY, 0, size)
    for i <- 0 until 10_000 do
      val pos = (i.toLong * 199) % (size.toLong - 8)
      m.getLong(pos.toInt)

  ch.close()
  Files.delete(path)
end main
```

Memory-mapped IO typically wins for random access because each  
`getLong(pos)` is a direct memory read with no syscall. Channel-based  
IO with direct buffers wins for sequential access when transfer sizes  
are large (≥64 KB) and the file is not already in the page cache.  
Use memory mapping when the access pattern is random, the file is  
read many times, or multiple processes share the same data. Use  
channels when writing large sequential streams or when you need  
precise control over transfer boundaries.  

## Avoiding small reads/writes

Aggregating many small operations into fewer large ones dramatically  
reduces kernel-crossing overhead.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path  = Paths.get("/tmp/small-writes.dat")
  val count = 100_000

  def writeSmall(): Long =
    val ch = FileChannel.open(path, StandardOpenOption.WRITE,
      StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING)
    val t0 = System.nanoTime()
    for _ <- 0 until count do
      ch.write(ByteBuffer.wrap(Array[Byte](1, 2, 3, 4)))
    ch.close()
    System.nanoTime() - t0

  def writeLarge(): Long =
    val buf = ByteBuffer.allocateDirect(count * 4)
    for _ <- 0 until count do buf.put(1.toByte).put(2.toByte)
                                  .put(3.toByte).put(4.toByte)
    buf.flip()
    val ch = FileChannel.open(path, StandardOpenOption.WRITE,
      StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING)
    val t0 = System.nanoTime()
    while buf.hasRemaining do ch.write(buf)
    ch.close()
    System.nanoTime() - t0

  val small = writeSmall() / 1_000_000.0
  val large = writeLarge() / 1_000_000.0
  println(f"${count} small writes : ${small}%.1f ms")
  println(f"one large write  : ${large}%.1f ms")
  println(f"speedup          : ${small / large}%.1fx")

  Files.delete(path)
end main
```

Each `channel.write` call with a tiny buffer is a separate syscall.  
Accumulating all data into a single direct buffer and issuing one  
`write` saturates the underlying storage controller far more  
efficiently. The speedup factor is typically 10× to 100× depending  
on storage latency. When individual items arrive asynchronously, a  
`ByteArrayOutputStream` or off-heap accumulator can collect them  
before a bulk flush.  

## Batching operations

Batching reads across a `FileChannel` with a size hint avoids  
per-record seeks.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val path    = Paths.get("/tmp/batch.dat")
  val records = 10_000
  val recSize = 8

  // Write synthetic records
  val buf = ByteBuffer.allocateDirect(records * recSize)
  for i <- 0 until records do buf.putLong(i.toLong)
  buf.flip()
  val wch = FileChannel.open(path, StandardOpenOption.WRITE,
    StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING)
  while buf.hasRemaining do wch.write(buf)
  wch.close()

  // Batch read: entire file in one call
  val ch = FileChannel.open(path, StandardOpenOption.READ)
  val rBuf = ByteBuffer.allocateDirect(ch.size.toInt)
  while rBuf.hasRemaining do ch.read(rBuf)
  ch.close()
  rBuf.flip()

  var sum = 0L
  while rBuf.hasRemaining do sum += rBuf.getLong
  println(s"sum of all record keys: $sum")
  println(s"expected: ${(records.toLong * (records - 1)) / 2}")

  Files.delete(path)
end main
```

Reading the entire file into a single buffer and then processing  
records from the buffer is the highest-throughput strategy for fixed-  
size record files. The buffer holds all data in RAM, eliminating  
repeated channel transitions. The expected sum formula validates the  
result without printing all records. For files that exceed available  
memory, split the file into batch-sized windows and process each  
window in turn.  

## Minimising syscalls

`FileChannel.transferTo` and large buffers both reduce the number of  
kernel entries per byte of data.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Paths, StandardOpenOption, Files}

@main def main() =
  val src  = Paths.get("/tmp/syscall-src.dat")
  val dst1 = Paths.get("/tmp/syscall-manual.dat")
  val dst2 = Paths.get("/tmp/syscall-transfer.dat")

  Files.write(src, Array.tabulate[Byte](256 * 1024)(_.toByte))

  val srcCh = FileChannel.open(src, StandardOpenOption.READ)

  // Manual copy with a 4 KB buffer
  def manualCopy(dest: java.nio.file.Path): Unit =
    srcCh.position(0)
    val buf = ByteBuffer.allocateDirect(4096)
    val dch = FileChannel.open(dest, StandardOpenOption.WRITE,
      StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING)
    while srcCh.read(buf) != -1 do
      buf.flip()
      while buf.hasRemaining do dch.write(buf)
      buf.clear()
    dch.close()

  // transferTo: may map to sendfile
  def transferCopy(dest: java.nio.file.Path): Unit =
    srcCh.position(0)
    val total = srcCh.size
    var done  = 0L
    val dch   = FileChannel.open(dest, StandardOpenOption.WRITE,
      StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING)
    while done < total do
      done += srcCh.transferTo(done, total - done, dch)
    dch.close()

  val t0 = System.nanoTime(); manualCopy(dst1)
  val t1 = System.nanoTime(); transferCopy(dst2)
  val t2 = System.nanoTime()

  println(f"manual  : ${(t1-t0)/1e6}%.2f ms")
  println(f"transfer: ${(t2-t1)/1e6}%.2f ms")

  srcCh.close()
  List(src, dst1, dst2).foreach(Files.delete)
end main
```

With a 4 KB buffer the manual copy issues 64 `read` and 64 `write`  
syscalls for a 256 KB file. `transferTo` collapses this into a single  
or a few kernel transitions. On Linux the `sendfile` path avoids  
copying data through user space entirely. The practical speedup depends  
on the OS, storage type, and whether the source file is in the page  
cache; solid-state storage narrows the gap while cold spinning-disk  
reads amplify it.  

## Designing high-performance IO pipelines

An IO pipeline chains `transferTo`, direct buffers, and a selector  
to maximise throughput with minimal threads.  

```scala
import java.nio.channels.{FileChannel, Selector, SelectionKey,
                           ServerSocketChannel, SocketChannel}
import java.nio.ByteBuffer
import java.nio.file.{Paths, StandardOpenOption, Files}
import java.net.InetSocketAddress

@main def main() =
  val filePath = Paths.get("/tmp/pipeline-src.dat")
  Files.write(filePath, Array.tabulate[Byte](8192)(_.toByte))

  val server = ServerSocketChannel.open()
  server.bind(InetSocketAddress(9097))
  server.configureBlocking(false)
  val sel = Selector.open()
  server.register(sel, SelectionKey.OP_ACCEPT)

  // Simple pipeline: accept → send file → close
  val stop = System.currentTimeMillis + 3000

  Thread(() =>
    Thread.sleep(200)
    val ch = SocketChannel.open(InetSocketAddress("localhost", 9097))
    val buf = ByteBuffer.allocateDirect(65536)
    var total = 0
    var n = ch.read(buf)
    while n > 0 do
      total += n; buf.clear(); n = ch.read(buf)
    println(s"client received $total bytes")
    ch.close()
  ).start()

  while System.currentTimeMillis < stop do
    sel.select(100)
    val it = sel.selectedKeys.iterator
    while it.hasNext do
      val key = it.next(); it.remove()
      if key.isAcceptable then
        val client = server.accept()
        if client != null then
          client.configureBlocking(true)
          val fc = FileChannel.open(filePath, StandardOpenOption.READ)
          var sent = 0L
          while sent < fc.size do
            sent += fc.transferTo(sent, fc.size - sent, client)
          fc.close()
          client.close()

  sel.close()
  server.close()
  Files.delete(filePath)
  println("pipeline complete")
end main
```

The pipeline accepts connections through a non-blocking selector and  
then switches the accepted channel to blocking mode for the  
`transferTo` call, which simplifies the send loop without sacrificing  
throughput. `transferTo` maps to `sendfile` on Linux, so the file  
data never passes through the JVM heap. For a production pipeline,  
add back-pressure by registering `OP_WRITE` and tracking the send  
offset per key, allowing the selector to resume a stalled transfer  
when the socket becomes writable again.  
