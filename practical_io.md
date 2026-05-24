# Practical IO in Scala 3

Input and output (IO) describes every operation that moves data between a  
running program and the outside world. On the JVM that means reading and  
writing files, communicating over sockets, accessing standard streams, and  
interacting with operating-system resources such as pipes and devices.  
Understanding IO is essential for building real applications, because nearly  
every useful program must eventually read configuration, persist state, or  
exchange data with external services.  

## How IO works on the JVM

The JVM delegates all IO to the host operating system through system calls.  
When a Scala program opens a file, the JVM asks the kernel to create a file  
descriptor. Every subsequent read or write travels through that descriptor  
via kernel buffers before touching the physical storage device. The kernel  
maintains a page cache, so repeated reads of the same file are often served  
entirely from memory without hitting the disk again. The JVM itself adds  
another layer through java.io and java.nio, which translate Scala and Java  
API calls into the appropriate native operations for the underlying platform.  

## Blocking versus non-blocking IO

Blocking IO suspends the calling thread until the operation completes. This  
is the default behaviour of java.io streams and most java.nio.file methods.  
A thread blocked on a read call cannot do other work while waiting for bytes  
to arrive. Non-blocking IO lets the thread proceed and check later whether  
data is ready. The java.nio package provides Selector and Channel APIs for  
non-blocking network IO, but on most platforms file IO still blocks at the  
kernel level even when using NIO channels.  

## Text versus binary IO

Text IO reads and writes characters using an encoding such as UTF-8 or  
ISO-8859-1 to convert between byte sequences and Unicode code points.  
Binary IO treats the byte stream as raw data with no encoding layer applied.  
Use text IO for human-readable content such as configuration files, logs,  
and CSV data. Use binary IO for images, audio, serialised objects, and any  
format where exact byte control is required. Mixing the two without explicit  
encoding management causes data corruption that can be hard to reproduce.  

## Streams versus channels

The java.io package models IO as streams: sequential byte or character flows  
with read and write operations on individual bytes or small arrays. The  
java.nio package models IO as channels connected to buffers. You fill a  
ByteBuffer from a channel and then inspect the buffer contents in memory.  
Channels support scatter/gather operations, memory mapping, and file  
locking. Streams are simpler to use; channels offer finer control and  
better throughput when processing large volumes of data.  

## Paths versus files

A path is a description of a location in the filesystem hierarchy, which  
may or may not correspond to an actual file on disk. It can be absolute,  
starting from the filesystem root, or relative, evaluated against a working  
directory. The java.io.File class conflates path representation with  
filesystem operations in a single type. The java.nio.file.Path interface  
keeps them separate, offering a cleaner API for normalisation, resolution,  
and comparison that does not touch the disk until you explicitly call a  
Files method. Prefer Path over File in all new code.  

## Resource safety

Every IO resource — file handle, socket, stream, or channel — must be  
closed after use. Failing to close a resource leaks the underlying  
operating-system file descriptor. Because the JVM limits the number of open  
descriptors per process, unclosed resources eventually cause runtime failures  
with cryptic "too many open files" errors. Scala 3 provides  
scala.util.Using, which guarantees that a resource is closed when the block  
exits, whether normally or through an exception. This mirrors the Java  
try-with-resources statement and eliminates the need for verbose try-finally  
blocks around every IO operation.  

## Classic IO versus NIO.2

Use scala.io.Source for quick read-only text tasks in scripts and small  
utilities. Use java.io streams when integrating with older Java APIs or the  
Java serialisation framework. Prefer java.nio.file.Files and Path for all  
production file work: the API is cleaner, error messages are more specific,  
and operations such as atomic moves and directory walks are built in. Reserve  
FileChannel and ByteBuffer for high-throughput workloads, binary protocols,  
or memory-mapped access patterns.  

## Common pitfalls

Forgetting to specify an encoding turns valid UTF-8 content into garbage on  
Windows systems whose default charset is a legacy code page. Omitting  
buffering forces the kernel to process thousands of tiny system calls instead  
of a few large ones, which can reduce throughput by orders of magnitude.  
Leaving streams open inside loops exhausts file descriptors very quickly.  
Calling scala.io.Source.fromFile without closing the result leaks a  
descriptor on every invocation. Always close resources, always buffer bulk  
IO, and always specify the character encoding explicitly.  

---

## Reading a text file as a whole string

Read the entire content of a text file into a single String using  
scala.io.Source.  

```scala
import scala.io.Source
import java.nio.file.Files

@main def main() =
  val path = Files.createTempFile("demo", ".txt")
  Files.writeString(path, "Hello, Scala 3 IO!\nSecond line.")

  val source = Source.fromFile(path.toFile, "UTF-8")
  try
    val content = source.mkString
    println(content)
  finally
    source.close()

  Files.deleteIfExists(path)
end main
```

`Source.fromFile` opens a buffered connection to the file and returns a  
`Source` object whose `mkString` method reads every character and  
concatenates them into one string. The file is opened with an explicit  
`"UTF-8"` encoding to avoid platform-specific default charset issues.  
The `try`/`finally` block guarantees that `close` is called even when an  
exception is thrown during reading, preventing a file descriptor leak.  

## Reading a file line by line

Iterate over the lines of a text file without loading the entire content  
into memory at once.  

```scala
import scala.io.Source
import java.nio.file.Files

@main def main() =
  val path = Files.createTempFile("lines", ".txt")
  Files.writeString(path, "line one\nline two\nline three")

  val source = Source.fromFile(path.toFile, "UTF-8")
  try
    for line <- source.getLines() do
      println(line)
  finally
    source.close()

  Files.deleteIfExists(path)
end main
```

`getLines` returns an `Iterator[String]` that yields one line at a time,  
reading only as much data from disk as necessary. This approach works for  
files of any size because it never requires the full content to reside in  
heap memory simultaneously. Each line has its trailing newline character  
stripped automatically by the iterator, so no additional trimming is needed.  

## Reading with a specific encoding

Open a file using an explicit character encoding to prevent data corruption  
on systems with a non-UTF-8 default charset.  

```scala
import scala.io.Source
import java.nio.file.Files
import java.nio.charset.StandardCharsets

@main def main() =
  val path = Files.createTempFile("enc", ".txt")
  val iso  = StandardCharsets.ISO_8859_1
  Files.write(path, "Héllo wörld".getBytes(iso))

  val source = Source.fromFile(path.toFile, "ISO-8859-1")
  try
    val text = source.mkString
    println(text)
    println(s"Length: ${text.length} chars")
  finally
    source.close()

  Files.deleteIfExists(path)
end main
```

The file is written using ISO-8859-1, which encodes each accented character  
as a single byte. Reading it back with the matching encoding faithfully  
reconstructs the original string. If the file were opened with UTF-8  
instead, the interpreter would either produce garbled output or throw a  
`MalformedInputException`. Specifying the encoding explicitly on every  
file open call is the most effective way to prevent this class of bug.  

## Writing text to a file

Write a string to a new text file using a PrintWriter backed by a  
FileWriter.  

```scala
import java.io.{FileWriter, PrintWriter}
import java.nio.file.Files

@main def main() =
  val path   = Files.createTempFile("out", ".txt")
  val writer = PrintWriter(FileWriter(path.toFile, false))
  try
    writer.println("First line of output")
    writer.println("Second line of output")
  finally
    writer.close()

  println(Files.readString(path))
  Files.deleteIfExists(path)
end main
```

`FileWriter` opens the file with `false` as the append flag, which  
truncates any existing content before writing begins. `PrintWriter` wraps  
it to provide the convenient `println` method that appends the platform  
line separator automatically. Both writers buffer data internally, so  
content may not reach the disk until `close` is called. The `finally`  
block ensures the buffer is flushed and the handle released in all cases.  

## Appending to a file

Add new text to the end of an existing file without overwriting its current  
content.  

```scala
import java.io.{FileWriter, PrintWriter}
import java.nio.file.Files

@main def main() =
  val path = Files.createTempFile("append", ".txt")
  Files.writeString(path, "original content\n")

  val writer = PrintWriter(FileWriter(path.toFile, true))
  try
    writer.println("appended line one")
    writer.println("appended line two")
  finally
    writer.close()

  println(Files.readString(path))
  Files.deleteIfExists(path)
end main
```

Passing `true` as the second argument to `FileWriter` enables append mode,  
which positions the write cursor at the end of the existing file content  
rather than at the beginning. The original text is preserved and new lines  
follow it immediately. This is the standard pattern for log files, audit  
trails, and any scenario where a program adds entries incrementally over  
time without overwriting earlier data.  

## Buffered writing

Wrap a FileWriter in a BufferedWriter to accumulate small writes in memory  
before flushing them to disk as one system call.  

```scala
import java.io.{BufferedWriter, FileWriter}
import java.nio.file.Files

@main def main() =
  val path = Files.createTempFile("bwrite", ".txt")
  val bw   = BufferedWriter(FileWriter(path.toFile))
  try
    for i <- 1 to 1000 do
      bw.write(s"Line $i\n")
    bw.flush()
  finally
    bw.close()

  val lineCount = Files.readAllLines(path).size
  println(s"Wrote $lineCount lines")
  Files.deleteIfExists(path)
end main
```

Without buffering, each `write` call would propagate through the JVM into  
the OS, potentially triggering a system call for every tiny string. A  
`BufferedWriter` accumulates data in an internal byte array and flushes it  
to the underlying `FileWriter` only when the buffer is full or when `flush`  
is called explicitly. The `close` method always flushes any remaining  
buffer contents before releasing the file handle.  

## Reading binary files with FileInputStream

Read the raw bytes of a binary file using FileInputStream.  

```scala
import java.io.FileInputStream
import java.nio.file.Files

@main def main() =
  val path = Files.createTempFile("bin", ".dat")
  Files.write(path, Array[Byte](0x48, 0x65, 0x6c, 0x6c, 0x6f))

  val fis = FileInputStream(path.toFile)
  try
    val bytes = fis.readAllBytes()
    println(s"Read ${bytes.length} bytes")
    println(new String(bytes, "UTF-8"))
  finally
    fis.close()

  Files.deleteIfExists(path)
end main
```

`FileInputStream` opens a raw byte channel to the file without any  
character encoding layer. `readAllBytes` allocates a byte array large  
enough to hold the entire file and fills it in a single call, which is  
convenient for small files. The bytes 0x48, 0x65, 0x6c, 0x6c, and 0x6f  
are the ASCII codes for "Hello", so interpreting the result as UTF-8  
confirms the round-trip. For large files, reading in chunks is preferable  
to avoid large heap allocations.  

## Writing binary files with FileOutputStream

Write a raw byte array to a file using FileOutputStream.  

```scala
import java.io.FileOutputStream
import java.nio.file.Files

@main def main() =
  val path = Files.createTempFile("bwr", ".dat")
  val data = Array[Byte](0x89.toByte, 0x50, 0x4e, 0x47)

  val fos = FileOutputStream(path.toFile)
  try
    fos.write(data)
  finally
    fos.close()

  val read = Files.readAllBytes(path)
  println(s"Bytes match: ${java.util.Arrays.equals(data, read)}")
  println(s"File size: ${Files.size(path)} bytes")
  Files.deleteIfExists(path)
end main
```

`FileOutputStream` opens a direct write channel to the named file and  
truncates any existing content by default. The `write(byte[])` method  
copies every byte in the array to the file in a single call. Verifying  
the result with `Arrays.equals` confirms that no bytes were altered during  
the write and read cycle. The four bytes written here are the start of a  
PNG file signature, illustrating how binary formats embed magic numbers  
at fixed byte offsets.  

## Reading bytes in chunks

Read a binary file incrementally in fixed-size chunks to bound memory  
usage when the file is large.  

```scala
import java.io.FileInputStream
import java.nio.file.Files

@main def main() =
  val path = Files.createTempFile("chunks", ".dat")
  Files.write(path, (1 to 256).map(_.toByte).toArray)

  val fis   = FileInputStream(path.toFile)
  val buf   = Array.ofDim[Byte](64)
  var total = 0
  try
    var n = fis.read(buf)
    while n != -1 do
      total += n
      println(s"Read chunk of $n bytes (total so far: $total)")
      n = fis.read(buf)
  finally
    fis.close()

  Files.deleteIfExists(path)
end main
```

The `read(byte[])` method attempts to fill the supplied buffer but may  
return fewer bytes than the buffer capacity, particularly near the end of  
the file or when reading from a network socket. The loop continues until  
`read` returns `-1`, which signals end-of-file. Using a 64-byte buffer  
means the 256-byte test file is processed in four full chunks. For real  
workloads, buffer sizes of 8 KiB to 64 KiB significantly reduce system  
call overhead.  

## Writing bytes in chunks

Split a large byte array into fixed-size pieces and write each piece  
separately to illustrate the chunked-write pattern.  

```scala
import java.io.FileOutputStream
import java.nio.file.Files

@main def main() =
  val path      = Files.createTempFile("wchunks", ".dat")
  val data      = (0 until 256).map(_.toByte).toArray
  val chunkSize = 64

  val fos = FileOutputStream(path.toFile)
  try
    data.grouped(chunkSize).foreach(chunk => fos.write(chunk))
  finally
    fos.close()

  println(s"File size: ${Files.size(path)} bytes")
  Files.deleteIfExists(path)
end main
```

`Array.grouped` partitions the source array into sub-arrays of at most  
`chunkSize` bytes each, with the final group potentially smaller if the  
total length is not a multiple of `chunkSize`. Writing in chunks is useful  
when data arrives incrementally — from a network buffer or a compression  
pipeline — and cannot all be accumulated in memory before writing begins.  
The pattern generalises naturally to streaming pipelines where the source  
and destination run concurrently.  

## Detecting end of file

Recognise the end-of-file condition using the sentinel value returned by  
the single-byte `read` method.  

```scala
import java.io.FileInputStream
import java.nio.file.Files

@main def main() =
  val path = Files.createTempFile("eof", ".dat")
  Files.write(path, Array[Byte](10, 20, 30, 40, 50))

  val fis = FileInputStream(path.toFile)
  try
    var b     = fis.read()
    var count = 0
    while b != -1 do
      println(s"Byte $count: $b")
      count += 1
      b = fis.read()
    println(s"EOF reached after $count bytes")
  finally
    fis.close()

  Files.deleteIfExists(path)
end main
```

The no-argument `read()` returns the next byte as an integer in the range  
0–255, or `-1` when the stream is exhausted. Using `Int` rather than  
`Byte` for the return value is essential because all 256 byte values are  
valid data, so the sentinel must lie outside that range. The loop  
processes each byte individually, which is inefficient for large files but  
clearly illustrates the EOF-detection pattern used throughout the java.io  
API. In production code, always prefer the buffered chunk-read pattern.  

## Working with BufferedInputStream and BufferedOutputStream

Copy a file by wrapping both the input and output streams in buffers to  
reduce the number of kernel system calls.  

```scala
import java.io.{BufferedInputStream, BufferedOutputStream,
                FileInputStream, FileOutputStream}
import java.nio.file.Files

@main def main() =
  val src = Files.createTempFile("bsrc", ".dat")
  val dst = Files.createTempFile("bdst", ".dat")
  Files.write(src, (0 to 255).map(_.toByte).toArray)

  val bis = BufferedInputStream(FileInputStream(src.toFile), 8192)
  val bos = BufferedOutputStream(FileOutputStream(dst.toFile), 8192)
  try
    val buf = Array.ofDim[Byte](256)
    var n   = bis.read(buf)
    while n != -1 do
      bos.write(buf, 0, n)
      n = bis.read(buf)
    bos.flush()
  finally
    bis.close()
    bos.close()

  println(s"Copied ${Files.size(dst)} bytes")
  Files.deleteIfExists(src)
  Files.deleteIfExists(dst)
end main
```

`BufferedInputStream` reads from disk in blocks of 8192 bytes and stores  
them in an internal array. Subsequent `read` calls on the buffered stream  
draw bytes from that array without hitting the OS again until the buffer is  
drained. `BufferedOutputStream` works in reverse, collecting bytes until  
its internal buffer is full before flushing them to the underlying stream.  
Closing a buffered stream automatically flushes any remaining buffered  
bytes, ensuring data integrity.  

## Reading resources from the classpath

Load a file bundled inside the application JAR by retrieving it as a  
classpath resource stream.  

```scala
import scala.io.Source

@main def main() =
  val name   = "/application.properties"
  val stream = getClass.getResourceAsStream(name)
  if stream == null then
    println(s"Resource not found: $name")
  else
    val source = Source.fromInputStream(stream, "UTF-8")
    try
      source.getLines().take(3).foreach(println)
    finally
      source.close()
      stream.close()
end main
```

`getResourceAsStream` asks the class loader to locate a resource at the  
given path on the classpath. A leading slash makes the path absolute with  
respect to the root of the classpath rather than relative to the package  
of the calling class. The method returns `null` when the resource is not  
found, so a null check is always required. Wrapping the raw `InputStream`  
in a `Source` provides line-iteration and character-level access without  
needing to manage byte-to-character conversion manually.  

## Safe resource handling with Using

Use `scala.util.Using` to automatically close a resource and capture the  
result or failure in a `Try`.  

```scala
import scala.util.{Using, Success, Failure}
import scala.io.Source
import java.nio.file.Files

@main def main() =
  val path = Files.createTempFile("using", ".txt")
  Files.writeString(path, "safe resource handling")

  val result = Using(Source.fromFile(path.toFile, "UTF-8")) { source =>
    source.mkString.toUpperCase
  }

  result match
    case Success(text) => println(text)
    case Failure(ex)   => println(s"Error: ${ex.getMessage}")

  Files.deleteIfExists(path)
end main
```

`Using` accepts any `AutoCloseable` value and a function that uses it.  
The function's return value is wrapped in `Success`, or any exception is  
captured in `Failure`, and `close` is called on the resource in both cases.  
This eliminates the need for explicit `try`/`finally` blocks and makes  
the intent clear at a glance. If the closing step itself throws while the  
body also threw, `Using` attaches the close exception as a suppressed  
exception on the primary failure so no error is silently swallowed.  

## Nested resources with Using.Manager

Acquire multiple resources simultaneously and have all of them closed  
automatically when the managed block exits.  

```scala
import scala.util.Using
import java.io.{FileInputStream, FileOutputStream}
import java.nio.file.Files

@main def main() =
  val src = Files.createTempFile("mgrsrc", ".dat")
  val dst = Files.createTempFile("mgrdst", ".dat")
  Files.write(src, "hello manager".getBytes("UTF-8"))

  val result = Using.Manager { use =>
    val in  = use(FileInputStream(src.toFile))
    val out = use(FileOutputStream(dst.toFile))
    out.write(in.readAllBytes())
    Files.size(dst)
  }

  result.foreach(n => println(s"Copied $n bytes"))
  Files.deleteIfExists(src)
  Files.deleteIfExists(dst)
end main
```

`Using.Manager` provides a `use` function that registers each resource for  
automatic closure. Resources are closed in reverse acquisition order when  
the block exits, mirroring the behaviour of nested try-with-resources  
statements in Java. This is the idiomatic way to manage multiple related  
resources in Scala 3 without deeply nesting `Using` calls. The entire  
block returns a `Try` containing either the last expression's value or  
any exception that was raised.  

---

## Checking file existence

Determine whether a path refers to an existing entry in the filesystem  
before attempting to read or write it.  

```scala
import java.nio.file.{Files, Paths}

@main def main() =
  val existing = Files.createTempFile("exist", ".txt")
  val missing  = Paths.get("/tmp/no_such_file_x7z9.txt")

  println(s"Existing (File API): ${existing.toFile.exists()}")
  println(s"Existing (NIO.2):    ${Files.exists(existing)}")
  println(s"Missing  (NIO.2):    ${Files.exists(missing)}")
  println(s"Not exists:          ${Files.notExists(missing)}")

  Files.deleteIfExists(existing)
end main
```

`Files.exists` consults the operating system to verify that the path  
refers to a real filesystem entry at the moment of the call. It returns  
`false` if the path does not exist and also if the check fails due to a  
permissions error, so a `false` result is not a definitive proof of  
absence. `Files.notExists` is the explicit complement and has the same  
ambiguity in permissions-denied scenarios. For critical operations, it  
is safer to open the file directly and handle `NoSuchFileException`.  

## Checking if a path is a directory or file

Determine the type of a filesystem entry using the Files predicate  
methods.  

```scala
import java.nio.file.Files

@main def main() =
  val dir  = Files.createTempDirectory("typedemo")
  val file = Files.createTempFile(dir, "demo", ".txt")

  println(s"dir  isDirectory:    ${Files.isDirectory(dir)}")
  println(s"dir  isRegularFile:  ${Files.isRegularFile(dir)}")
  println(s"file isDirectory:    ${Files.isDirectory(file)}")
  println(s"file isRegularFile:  ${Files.isRegularFile(file)}")
  println(s"file isSymbolicLink: ${Files.isSymbolicLink(file)}")

  Files.deleteIfExists(file)
  Files.deleteIfExists(dir)
end main
```

`Files.isDirectory` and `Files.isRegularFile` both follow symbolic links  
by default, reporting on the target rather than the link itself. A path  
can be neither a directory nor a regular file if it is a symbolic link  
whose target does not exist, a device node, or a named pipe. Passing  
`LinkOption.NOFOLLOW_LINKS` as a second argument checks the link entry  
itself instead of its target, which is useful when inspecting symlinks  
directly.  

## Creating directories

Create a single directory and a full nested tree using the Files API.  

```scala
import java.nio.file.{Files, Paths}

@main def main() =
  val tmp    = Paths.get(System.getProperty("java.io.tmpdir"))
  val single = tmp.resolve("myapp_single")
  val nested = tmp.resolve("myapp").resolve("sub").resolve("deep")

  Files.createDirectory(single)
  println(s"Single created: ${Files.isDirectory(single)}")

  Files.createDirectories(nested)
  println(s"Nested created: ${Files.isDirectory(nested)}")

  // Clean up deepest to shallowest
  Files.deleteIfExists(nested)
  Files.deleteIfExists(nested.getParent)
  Files.deleteIfExists(nested.getParent.getParent)
  Files.deleteIfExists(single)
end main
```

`Files.createDirectory` creates exactly one directory and throws  
`FileAlreadyExistsException` if it already exists or if any intermediate  
parent is missing. `Files.createDirectories` creates the full path  
including every missing parent, and it does not throw if the directory  
already exists. This makes `createDirectories` safe to call  
unconditionally at application startup to ensure the required directory  
structure is in place.  

## Listing directory contents

List all entries in a directory using a Files.list stream and process each  
path lazily.  

```scala
import java.nio.file.Files
import scala.util.Using

@main def main() =
  val dir = Files.createTempDirectory("listdemo")
  Files.createTempFile(dir, "alpha", ".txt")
  Files.createTempFile(dir, "beta",  ".txt")
  Files.createTempFile(dir, "gamma", ".log")

  Using(Files.list(dir)) { stream =>
    stream.forEach(p => println(p.getFileName))
  }

  Using(Files.list(dir))(_.forEach(Files.delete))
  Files.deleteIfExists(dir)
end main
```

`Files.list` returns a `java.util.stream.Stream[Path]` that implements  
`AutoCloseable`, so it must be closed after use. `scala.util.Using`  
handles the closure automatically even if an exception is thrown while  
iterating. The stream is not ordered by default; add `.sorted()` before  
`forEach` if alphabetical output is required. For very large directories,  
`DirectoryStream` offers more explicit iteration control and lower memory  
overhead.  

## Filtering files by extension

Select only files matching a given suffix from a directory listing.  

```scala
import java.nio.file.Files
import scala.util.Using

@main def main() =
  val dir = Files.createTempDirectory("filterdemo")
  Files.createTempFile(dir, "a", ".txt")
  Files.createTempFile(dir, "b", ".txt")
  Files.createTempFile(dir, "c", ".log")
  Files.createTempFile(dir, "d", ".csv")

  Using(Files.list(dir)) { stream =>
    println("Text files:")
    stream
      .filter(p => p.toString.endsWith(".txt"))
      .forEach(p => println(s"  ${p.getFileName}"))
  }

  Using(Files.list(dir))(_.forEach(Files.delete))
  Files.deleteIfExists(dir)
end main
```

`Stream.filter` applies the predicate to each entry and passes only  
matching paths downstream. Filtering by `toString.endsWith(".txt")` is a  
simple approach suitable for flat directories. For nested trees or complex  
patterns, `Files.walk` combined with `filter` or `DirectoryStream` with a  
glob pattern provides more control. Note that hidden files and files with  
mixed-case extensions are matched exactly as their names appear on disk.  

## File metadata and timestamps

Read a file's size and modification time, then update the timestamp using  
`Files.setLastModifiedTime`.  

```scala
import java.nio.file.Files
import java.nio.file.attribute.FileTime
import java.time.Instant

@main def main() =
  val path = Files.createTempFile("meta", ".txt")
  Files.writeString(path, "some content for metadata demo")

  println(s"Size:          ${Files.size(path)} bytes")
  println(s"Is readable:   ${Files.isReadable(path)}")
  println(s"Is writable:   ${Files.isWritable(path)}")

  val before = Files.getLastModifiedTime(path)
  println(s"Before: $before")

  val newTime = FileTime.from(Instant.parse("2024-06-15T09:00:00Z"))
  Files.setLastModifiedTime(path, newTime)

  println(s"After:  ${Files.getLastModifiedTime(path)}")

  Files.deleteIfExists(path)
end main
```

`Files.size` returns the byte count without reading any content, making it  
a lightweight metadata query. `Files.getLastModifiedTime` returns a  
`FileTime` that converts cleanly to a `java.time.Instant`. Setting  
timestamps is commonly needed when replicating files while preserving  
their original modification times or when invalidating cached entries.  
Filesystem precision varies: some filesystems truncate sub-second values  
to the nearest second, so the stored time may differ slightly from the  
value passed to `setLastModifiedTime`.  

## Copying and moving files

Duplicate a file to a new location and relocate another file atomically  
using the NIO.2 API.  

```scala
import java.nio.file.{Files, StandardCopyOption}

@main def main() =
  val src   = Files.createTempFile("cpsrc", ".txt")
  val cpDst = Files.createTempFile("cpdst", ".txt")
  Files.writeString(src, "content to copy and move")

  Files.copy(src, cpDst, StandardCopyOption.REPLACE_EXISTING)
  println(s"Copy:      ${Files.readString(cpDst)}")
  println(s"Src still: ${Files.exists(src)}")

  val mvDst = src.resolveSibling("moved_demo.txt")
  Files.move(src, mvDst, StandardCopyOption.REPLACE_EXISTING)
  println(s"Src gone:  ${Files.exists(src)}")
  println(s"Moved:     ${Files.readString(mvDst)}")

  Files.deleteIfExists(cpDst)
  Files.deleteIfExists(mvDst)
end main
```

`Files.copy` duplicates the source bytes to the destination and leaves the  
source intact. `Files.move` removes the source after transferring the  
content. On the same filesystem partition, `Files.move` with  
`StandardCopyOption.ATOMIC_MOVE` asks the OS to perform the relocation  
atomically, so no reader can ever observe a partially moved file. Use  
`REPLACE_EXISTING` to overwrite an already-existing destination without  
a separate existence check.  

## Renaming and deleting files

Rename a file in place using `Files.move` and permanently remove it with  
`Files.delete`.  

```scala
import java.nio.file.{Files, StandardCopyOption}

@main def main() =
  val original = Files.createTempFile("rename_me", ".txt")
  Files.writeString(original, "rename test content")

  val renamed = original.resolveSibling("renamed_result.txt")
  Files.move(original, renamed, StandardCopyOption.REPLACE_EXISTING)

  println(s"Original exists: ${Files.exists(original)}")
  println(s"Renamed exists:  ${Files.exists(renamed)}")

  Files.delete(renamed)
  println(s"After delete:    ${Files.exists(renamed)}")
end main
```

`Files.move` within the same directory is the canonical rename operation  
on NIO.2: it atomically updates the directory entry without copying bytes.  
`Files.delete` throws `NoSuchFileException` if the path does not exist,  
making the failure explicit. Use `Files.deleteIfExists` instead when  
absence should be silently ignored, such as during cleanup in tests. Both  
methods throw `DirectoryNotEmptyException` if used on a non-empty  
directory.  

## Creating temporary files and directories

Create temporary files and directories using the platform's system  
temporary directory.  

```scala
import java.nio.file.Files

@main def main() =
  val tmpFile = Files.createTempFile("app-", ".tmp")
  val tmpDir  = Files.createTempDirectory("appwork-")

  println(s"Temp file: $tmpFile")
  println(s"Temp dir:  $tmpDir")

  Files.writeString(tmpFile, "temporary payload")
  println(s"Content:   ${Files.readString(tmpFile)}")

  val nested = Files.createTempFile(tmpDir, "task-", ".dat")
  println(s"Nested:    $nested")

  Files.deleteIfExists(nested)
  Files.deleteIfExists(tmpFile)
  Files.deleteIfExists(tmpDir)
end main
```

`Files.createTempFile` generates a unique file name from the given prefix  
and suffix in the system's default temporary directory, or in the  
directory supplied as the first argument. `Files.createTempDirectory`  
does the same for a directory entry. Temporary resources are not  
automatically deleted when the JVM exits; the application is responsible  
for removing them. A shutdown hook or `deleteOnExit` on the `File` object  
can automate deletion when the process terminates normally.  

---

## Creating and working with paths

Build `Path` objects from strings and URI values, then distinguish between  
absolute and relative paths.  

```scala
import java.nio.file.{Path, Paths}
import java.net.URI

@main def main() =
  val p1 = Paths.get("/tmp/data.txt")
  val p2 = Path.of("/tmp", "config", "app.conf")
  val p3 = Path.of(URI.create("file:///tmp/data.txt"))

  println(s"p1: $p1")
  println(s"p2: $p2")
  println(s"p1 equals p3: ${p1 == p3}")

  val abs = Path.of("/home/user/docs/report.txt")
  val rel = Path.of("docs/report.txt")

  println(s"abs isAbsolute: ${abs.isAbsolute}")
  println(s"rel isAbsolute: ${rel.isAbsolute}")

  val resolved    = Path.of("/home/user").resolve(rel)
  val relFromHome = Path.of("/home/user").relativize(abs)
  println(s"Resolved:    $resolved")
  println(s"Relativized: $relFromHome")
end main
```

`Path.of` is the factory method introduced in Java 11 and is preferred  
over the older `Paths.get`, though both produce equivalent results. Paths  
built from the same string are equal regardless of which factory was used.  
An absolute path begins with the filesystem root; a relative path is  
evaluated against a working directory. `resolve` appends a relative path  
to a base without touching the disk. `relativize` computes the shortest  
relative path navigating from the base to the target, which is useful  
for generating portable cross-directory references.  

## Normalizing paths

Remove redundant segments such as `.` and `..` from a path using  
`normalize`.  

```scala
import java.nio.file.Path

@main def main() =
  val messy = Path.of("/home/user/../user/./docs//report.txt")
  val clean = messy.normalize()

  println(s"Original:   $messy")
  println(s"Normalized: $clean")

  val rel = Path.of("a/b/../../c/../d")
  println(s"Rel before: $rel")
  println(s"Rel after:  ${rel.normalize()}")
end main
```

`normalize` processes the path string lexically without consulting the  
filesystem: it collapses `.` components, resolves `..` by removing the  
preceding segment, and eliminates duplicate separators. This means it  
does not resolve symbolic links and may produce a path that differs from  
what `toRealPath` would return. Always normalise paths that arrive from  
user input or configuration files before constructing derived paths from  
them to avoid unexpected traversal behaviour.  

## Resolving paths

Combine a base path with a relative segment to form a complete path.  

```scala
import java.nio.file.Path

@main def main() =
  val base = Path.of("/var/app")

  val config  = base.resolve("config/settings.yml")
  val logs    = base.resolve("logs/app.log")
  val sibling = base.resolveSibling("other")
  val abs     = base.resolve(Path.of("/etc/hosts"))

  println(s"Config:  $config")
  println(s"Logs:    $logs")
  println(s"Sibling: $sibling")
  println(s"Abs win: $abs")
end main
```

`resolve` is the primary way to construct sub-paths without string  
concatenation. When the argument is an absolute path, `resolve` simply  
returns it, which is intentional: callers can pass either a relative  
sub-path or an override absolute path and the method does the right thing.  
`resolveSibling` navigates to the parent first, making it convenient for  
replacing the last path component, such as changing a file's extension or  
redirecting output to a parallel directory.  

## Extracting filename, parent, and extension

Decompose a Path into its constituent parts: file name, parent directory,  
and extension string.  

```scala
import java.nio.file.Path

@main def main() =
  val path = Path.of("/var/log/app/server.log.gz")

  println(s"File name:  ${path.getFileName}")
  println(s"Parent:     ${path.getParent}")
  println(s"Root:       ${path.getRoot}")
  println(s"Name count: ${path.getNameCount}")

  val name = path.getFileName.toString
  val dot  = name.indexOf('.')
  val base = if dot >= 0 then name.substring(0, dot) else name
  val ext  = if dot >= 0 then name.substring(dot + 1) else ""

  println(s"Base name:  $base")
  println(s"Extension:  $ext")
end main
```

`getFileName` returns only the last component of the path as a new `Path`  
object. `getParent` returns everything except the last component, or `null`  
for a single-component path. The NIO.2 API does not provide a built-in  
extension method, so extension extraction requires simple string operations  
on the file name. Note that `server.log.gz` has a compound extension; the  
code above returns `log.gz` as the extension, which is often the most  
useful interpretation for compressed files.  

## Reading a file with Files.readString

Read a text file as a single `String` in one method call using the NIO.2  
high-level API.  

```scala
import java.nio.file.Files
import java.nio.charset.StandardCharsets

@main def main() =
  val path = Files.createTempFile("readstr", ".txt")
  Files.writeString(path, "Hello from NIO.2\nLine two\nLine three")

  val content = Files.readString(path, StandardCharsets.UTF_8)
  println(content)
  println(s"Length: ${content.length} chars")
  println(s"Lines:  ${content.lines().count()}")

  Files.deleteIfExists(path)
end main
```

`Files.readString` was introduced in Java 11 as a convenience method that  
opens the file, reads all bytes, decodes them with the specified charset,  
and closes the file in a single call. It is the NIO.2 equivalent of  
`Source.fromFile(...).mkString` but without the resource-management  
ceremony. Because it reads the entire file into a single heap object, it  
is not appropriate for files larger than available heap memory; prefer  
`Files.readAllLines` or a `BufferedReader` for large text files.  

## Writing a file with Files.writeString

Write or overwrite a text file in one call and optionally append to it  
using `Files.writeString`.  

```scala
import java.nio.file.{Files, StandardOpenOption}
import java.nio.charset.StandardCharsets

@main def main() =
  val path = Files.createTempFile("wstr", ".txt")

  Files.writeString(path, "Initial content.\n", StandardCharsets.UTF_8)
  println(Files.readString(path))

  Files.writeString(
    path,
    "Appended line.\n",
    StandardCharsets.UTF_8,
    StandardOpenOption.APPEND
  )
  println(Files.readString(path))

  Files.deleteIfExists(path)
end main
```

`Files.writeString` truncates the file and writes the given string in a  
single call by default. Passing `StandardOpenOption.APPEND` switches to  
append mode, making it functionally equivalent to `FileWriter(path, true)`  
but with a cleaner API and explicit charset control. `StandardOpenOption`  
also provides `CREATE_NEW`, which fails if the file already exists, and  
`SYNC`, which forces every write to reach the physical device before  
returning.  

## Reading all lines with Files.readAllLines

Load every line of a text file into a Scala `List` using  
`Files.readAllLines`.  

```scala
import java.nio.file.Files
import java.nio.charset.StandardCharsets
import scala.jdk.CollectionConverters.*

@main def main() =
  val path = Files.createTempFile("alllines", ".txt")
  Files.writeString(path, "alpha\nbeta\ngamma\ndelta")

  val lines = Files.readAllLines(path, StandardCharsets.UTF_8).asScala
  println(s"Total lines: ${lines.size}")
  lines.zipWithIndex.foreach { (line, i) =>
    println(s"  ${i + 1}: $line")
  }

  Files.deleteIfExists(path)
end main
```

`Files.readAllLines` returns a `java.util.List[String]`, which the  
`asScala` extension converts to a `mutable.Buffer` that behaves like a  
`Seq`. Each line has its line terminator stripped, consistent with  
`Source.getLines`. The method allocates all lines in memory at once, so  
it is unsuitable for very large files. For large files, use a  
`BufferedReader` and iterate line by line instead.  

## Writing bytes with Files.write

Persist a raw byte array to a file with a single `Files.write` call.  

```scala
import java.nio.file.Files

@main def main() =
  val path = Files.createTempFile("bytes", ".bin")
  val data = "binary\u0000content\u0000here".getBytes("UTF-8")

  Files.write(path, data)
  println(s"Written: ${data.length} bytes")

  val read = Files.readAllBytes(path)
  println(s"Round-trip: ${java.util.Arrays.equals(data, read)}")
  println(s"File size:  ${Files.size(path)} bytes")

  Files.deleteIfExists(path)
end main
```

`Files.write` creates or truncates the file and writes the byte array  
atomically with respect to the method call. It does not require an  
explicit stream or channel, which makes it ideal for small to  
medium-sized binary payloads. `Files.readAllBytes` is the symmetric read  
operation. Both methods open and close the underlying channel internally,  
so there is no resource to manage on the call site.  

## Copying and moving with Files.copy and Files.move

Duplicate or relocate a file atomically using NIO.2 high-level methods  
with copy options.  

```scala
import java.nio.file.{Files, StandardCopyOption}

@main def main() =
  val src   = Files.createTempFile("nio2src", ".txt")
  val cpDst = Files.createTempFile("nio2cp",  ".txt")
  Files.writeString(src, "NIO.2 copy and move demo")

  Files.copy(src, cpDst, StandardCopyOption.REPLACE_EXISTING,
             StandardCopyOption.COPY_ATTRIBUTES)
  println(s"Copied:     ${Files.readString(cpDst)}")

  val mvDst = src.resolveSibling("nio2_moved.txt")
  Files.move(src, mvDst, StandardCopyOption.REPLACE_EXISTING)
  println(s"Source gone: ${Files.exists(src)}")
  println(s"At target:   ${Files.readString(mvDst)}")

  Files.deleteIfExists(cpDst)
  Files.deleteIfExists(mvDst)
end main
```

`COPY_ATTRIBUTES` instructs the OS to copy file metadata such as  
timestamps and permissions alongside the content. Without it, the  
destination file gets the current time as its creation timestamp.  
`StandardCopyOption.ATOMIC_MOVE` asks the OS to rename the file atomically  
within a single filesystem partition; it is the safest option for log  
rotation and configuration hot-swap scenarios where partial writes must  
never be visible.  

## Reading BasicFileAttributes

Access creation time, size, and type information via `BasicFileAttributes`  
in a single metadata read.  

```scala
import java.nio.file.Files
import java.nio.file.attribute.BasicFileAttributes

@main def main() =
  val path = Files.createTempFile("attrs", ".txt")
  Files.writeString(path, "attribute demo content")

  val attrs = Files.readAttributes(path, classOf[BasicFileAttributes])

  println(s"Size:          ${attrs.size()} bytes")
  println(s"Created:       ${attrs.creationTime()}")
  println(s"Last modified: ${attrs.lastModifiedTime()}")
  println(s"Last accessed: ${attrs.lastAccessTime()}")
  println(s"Is file:       ${attrs.isRegularFile}")
  println(s"Is directory:  ${attrs.isDirectory}")

  Files.deleteIfExists(path)
end main
```

`Files.readAttributes` retrieves all attributes of the requested type in  
one native call, which is more efficient than calling individual  
`Files.size`, `Files.getLastModifiedTime`, and `Files.isRegularFile`  
methods separately. The `BasicFileAttributes` interface is available on  
every platform. More specific views such as `DosFileAttributes` and  
`PosixFileAttributes` extend it with platform-specific fields and are  
used in the following example.  

## Reading POSIX attributes and setting permissions

Read Unix file permissions and modify them on POSIX-compliant systems.  

```scala
import java.nio.file.Files
import java.nio.file.attribute.{PosixFileAttributes,
                                PosixFilePermissions}
import scala.util.Try

@main def main() =
  val path = Files.createTempFile("posix", ".txt")

  Try {
    val attrs = Files.readAttributes(path, classOf[PosixFileAttributes])
    val perms = PosixFilePermissions.toString(attrs.permissions())
    println(s"Permissions: $perms")
    println(s"Owner:       ${attrs.owner().getName}")
    println(s"Group:       ${attrs.group().getName}")

    val newPerms = PosixFilePermissions.fromString("rw-r-----")
    Files.setPosixFilePermissions(path, newPerms)
    println("Permissions updated.")
  }.recover {
    case _: UnsupportedOperationException =>
      println("POSIX attributes not supported on this OS.")
  }

  Files.deleteIfExists(path)
end main
```

`PosixFilePermissions.toString` converts the `Set[PosixFilePermission]`  
to the familiar nine-character string such as `rw-r--r--`. The symmetric  
`fromString` parses that format back into the set. POSIX attributes are  
available on Linux and macOS but not on Windows, so the example wraps the  
block in `Try` and recovers from `UnsupportedOperationException` to run  
portably. Setting permissions is commonly needed for secrets files such  
as private keys and configuration files that must not be world-readable.  

---

## Using DirectoryStream

Lazily iterate directory entries with `DirectoryStream` for explicit,  
resource-safe traversal.  

```scala
import java.nio.file.{Files, DirectoryStream, Path}
import scala.util.Using

@main def main() =
  val dir = Files.createTempDirectory("dstream")
  Files.createTempFile(dir, "a", ".txt")
  Files.createTempFile(dir, "b", ".txt")
  Files.createTempFile(dir, "c", ".log")

  Using(Files.newDirectoryStream(dir)) { stream =>
    stream.forEach(p => println(p.getFileName))
  }

  Using(Files.list(dir))(_.forEach(Files.delete))
  Files.deleteIfExists(dir)
end main
```

`Files.newDirectoryStream` returns a `DirectoryStream[Path]` that  
implements `Closeable`, making it compatible with `scala.util.Using`.  
Entries are produced lazily from the OS directory buffer rather than  
being loaded into a list all at once, which keeps memory usage constant  
regardless of directory size. Unlike `Files.list`, the order of entries  
is not guaranteed and depends entirely on the filesystem and OS. Always  
close the stream after iteration to release the underlying directory  
handle.  

## Filtering with glob patterns

Select filesystem entries by name pattern using a glob string passed  
directly to `newDirectoryStream`.  

```scala
import java.nio.file.Files
import scala.util.Using

@main def main() =
  val dir = Files.createTempDirectory("glob")
  Files.createTempFile(dir, "report", ".txt")
  Files.createTempFile(dir, "data",   ".csv")
  Files.createTempFile(dir, "notes",  ".txt")
  Files.createTempFile(dir, "audit",  ".log")

  println("Matching *.txt:")
  Using(Files.newDirectoryStream(dir, "*.txt")) { stream =>
    stream.forEach(p => println(s"  ${p.getFileName}"))
  }

  Using(Files.list(dir))(_.forEach(Files.delete))
  Files.deleteIfExists(dir)
end main
```

The glob syntax supported by `newDirectoryStream` uses `*` to match any  
sequence of characters within a single path component, `**` to match  
across components, and `?` to match a single character. Character classes  
such as `[a-z]` and brace groups such as `{*.txt,*.csv}` are also  
supported. The pattern is matched only against the file name component,  
not the full path, so `/tmp/glob/*.txt` is specified as just `*.txt`  
when the directory is already provided.  

## Recursive directory traversal

Walk an entire directory tree with `Files.walk` and process every entry  
at any depth.  

```scala
import java.nio.file.Files
import java.util.Comparator
import scala.util.Using

@main def main() =
  val root = Files.createTempDirectory("walk")
  val sub  = Files.createDirectories(root.resolve("sub"))
  Files.createTempFile(root, "root1", ".txt")
  Files.createTempFile(root, "root2", ".csv")
  Files.createTempFile(sub,  "sub1",  ".txt")

  println("All entries:")
  Using(Files.walk(root)) { stream =>
    stream.forEach(p => println(s"  $p"))
  }

  // Delete deepest entries first to allow directory removal
  Using(Files.walk(root)) { stream =>
    stream.sorted(Comparator.reverseOrder()).forEach(Files.delete)
  }
end main
```

`Files.walk` performs a depth-first traversal and yields the root path  
itself as the first entry. The depth is unbounded by default; pass a  
maximum depth as a second argument to limit traversal. Deleting with a  
reverse-sorted walk ensures that files are deleted before their parent  
directories, which is required because `Files.delete` refuses to remove  
a non-empty directory. For very deep trees, consider `Files.walkFileTree`  
with a `FileVisitor` to handle errors per-entry.  

---

## Reading a file with FileChannel

Open a `FileChannel` and drain all bytes into a `ByteBuffer` for  
processing.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Files, StandardOpenOption}

@main def main() =
  val path = Files.createTempFile("fch_read", ".txt")
  Files.writeString(path, "FileChannel read example")

  val channel = FileChannel.open(path, StandardOpenOption.READ)
  try
    val buf = ByteBuffer.allocate(channel.size().toInt)
    channel.read(buf)
    buf.flip()
    val text = new String(buf.array(), 0, buf.limit(), "UTF-8")
    println(text)
  finally
    channel.close()

  Files.deleteIfExists(path)
end main
```

`FileChannel.open` accepts one or more `OpenOption` values that control  
whether the channel is opened for reading, writing, or both. The channel  
size is read first to allocate an exact-fit `ByteBuffer`. After `read`,  
the buffer's position points just past the last byte written; `flip` resets  
the position to zero and sets the limit to the number of bytes read,  
making the buffer ready to consume. Always use `try`/`finally` or  
`Using` to close channels since they are `AutoCloseable`.  

## Writing a file with FileChannel

Encode a string into a `ByteBuffer` and write it to a file through a  
`FileChannel`.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Files, StandardOpenOption}

@main def main() =
  val path = Files.createTempFile("fch_write", ".txt")

  val channel = FileChannel.open(
    path,
    StandardOpenOption.WRITE,
    StandardOpenOption.TRUNCATE_EXISTING
  )
  try
    val data = "Written via FileChannel".getBytes("UTF-8")
    val buf  = ByteBuffer.wrap(data)
    while buf.hasRemaining do
      channel.write(buf)
  finally
    channel.close()

  println(Files.readString(path))
  Files.deleteIfExists(path)
end main
```

`ByteBuffer.wrap` creates a buffer that shares the underlying byte array,  
so no copying occurs. `channel.write` may write fewer bytes than requested  
in a single call — this is rare for file channels but common for socket  
channels — so the `while buf.hasRemaining` loop ensures the entire buffer  
is written before returning. `TRUNCATE_EXISTING` clears the file to zero  
length before writing, which is the equivalent of opening a file in  
write mode.  

## Allocating and using ByteBuffer

Create a `ByteBuffer`, write typed values into it, then flip and read them  
back to understand the position-limit-capacity model.  

```scala
import java.nio.ByteBuffer

@main def main() =
  val buf = ByteBuffer.allocate(32)

  println(s"capacity:  ${buf.capacity}")
  println(s"position:  ${buf.position}")
  println(s"limit:     ${buf.limit}")

  buf.putInt(2024)
  buf.putDouble(3.14159)
  buf.putLong(System.currentTimeMillis())

  println(s"After writes — position: ${buf.position}")

  buf.flip()
  println(s"After flip   — position: ${buf.position}, limit: ${buf.limit}")

  println(s"Int:    ${buf.getInt()}")
  println(s"Double: ${buf.getDouble()}")
  println(s"Long:   ${buf.getLong()}")
end main
```

A newly allocated `ByteBuffer` starts with `position = 0` and  
`limit = capacity`. Each `put` advances `position` by the type width in  
bytes. `flip` prepares the buffer for reading by setting `limit` to the  
current `position` and resetting `position` to zero, so subsequent `get`  
calls traverse exactly the bytes that were written. `clear` does the  
reverse — it resets both `position` and `limit` to their initial values —  
making the buffer ready for another write cycle. The direct counterpart  
`ByteBuffer.allocateDirect` bypasses the JVM heap for maximum IO  
throughput with native channels.  

## Reading into a ByteBuffer in a loop

Fill a reusable `ByteBuffer` from a `FileChannel` repeatedly to handle  
partial reads and large files.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Files, StandardOpenOption}
import java.io.ByteArrayOutputStream

@main def main() =
  val path = Files.createTempFile("bufloop", ".txt")
  Files.writeString(path, "0123456789ABCDEF" * 4)  // 64 chars

  val ch  = FileChannel.open(path, StandardOpenOption.READ)
  val buf = ByteBuffer.allocate(16)
  val out = ByteArrayOutputStream()
  try
    while ch.read(buf) != -1 do
      buf.flip()
      val tmp = Array.ofDim[Byte](buf.remaining)
      buf.get(tmp)
      out.write(tmp)
      buf.clear()
  finally
    ch.close()

  println(s"Total bytes collected: ${out.size()}")
  println(out.toString("UTF-8").take(32))
  Files.deleteIfExists(path)
end main
```

The `clear` call at the end of each iteration resets `position` to zero  
and `limit` to `capacity` so the buffer is ready to be filled again. The  
inner `Array.ofDim[Byte](buf.remaining)` copy is needed because  
`ByteArrayOutputStream.write` does not understand `ByteBuffer` directly.  
In performance-sensitive code, keep a single `byte[]` scratch array across  
iterations rather than allocating a new one per chunk.  

## Writing from a ByteBuffer in a loop

Encode multiple strings into `ByteBuffer` instances and write them to a  
`FileChannel` in sequence.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Files, StandardOpenOption}

@main def main() =
  val path  = Files.createTempFile("bufwloop", ".txt")
  val lines = List("first line", "second line", "third line")

  val ch = FileChannel.open(
    path,
    StandardOpenOption.WRITE,
    StandardOpenOption.TRUNCATE_EXISTING
  )
  try
    for line <- lines do
      val bytes = (line + "\n").getBytes("UTF-8")
      val buf   = ByteBuffer.wrap(bytes)
      while buf.hasRemaining do
        ch.write(buf)
  finally
    ch.close()

  println(Files.readString(path))
  Files.deleteIfExists(path)
end main
```

Each line is individually encoded and wrapped in a buffer. The inner  
`while buf.hasRemaining` loop handles partial writes, which are unusual  
for local file channels but essential for correctness when the same  
pattern is applied to socket channels. For better performance, encode all  
lines into a single large buffer or use `channel.write(ByteBuffer[])` to  
scatter multiple buffers in one call.  

## Memory-mapped files with FileChannel.map

Map a file region directly into process address space for zero-copy  
access.  

```scala
import java.nio.channels.FileChannel
import java.nio.file.{Files, StandardOpenOption}

@main def main() =
  val path = Files.createTempFile("mmap", ".dat")
  Files.write(path, (0 until 256).map(_.toByte).toArray)

  val ch = FileChannel.open(
    path,
    StandardOpenOption.READ,
    StandardOpenOption.WRITE
  )
  try
    val mapped = ch.map(FileChannel.MapMode.READ_WRITE, 0, ch.size())

    println(s"First byte:  ${mapped.get(0) & 0xff}")
    mapped.put(0, 0xff.toByte)
    println(s"After write: ${mapped.get(0) & 0xff}")
    mapped.force()
  finally
    ch.close()

  Files.deleteIfExists(path)
end main
```

`FileChannel.map` asks the OS to create a virtual memory mapping so that  
reads and writes on the returned `MappedByteBuffer` go directly to the  
page cache without an explicit read or write system call. This eliminates  
one copy compared with channel-based IO and is particularly effective for  
random access across large files. `force` flushes any dirty pages back to  
the storage device. Note that mapped regions cannot be unmapped explicitly  
in Java; the mapping persists until the buffer is garbage-collected.  

## Random access IO with FileChannel

Seek to an arbitrary byte offset and read or overwrite data at that  
position.  

```scala
import java.nio.ByteBuffer
import java.nio.channels.FileChannel
import java.nio.file.{Files, StandardOpenOption}

@main def main() =
  val path = Files.createTempFile("random", ".dat")
  Files.write(path, "ABCDEFGHIJ".getBytes("UTF-8"))

  val ch = FileChannel.open(
    path,
    StandardOpenOption.READ,
    StandardOpenOption.WRITE
  )
  try
    val buf = ByteBuffer.allocate(3)

    ch.position(5)
    ch.read(buf)
    buf.flip()
    val slice = new String(buf.array(), 0, buf.limit(), "UTF-8")
    println(s"Read at pos 5: $slice")   // FGH

    buf.clear()
    buf.put("XYZ".getBytes("UTF-8"))
    buf.flip()
    ch.position(2)
    ch.write(buf)
  finally
    ch.close()

  println(s"After overwrite: ${Files.readString(path)}") // ABXYZFGHIJ
  Files.deleteIfExists(path)
end main
```

`channel.position(n)` moves the file pointer to byte offset `n` without  
reading or writing any data. Subsequent reads and writes occur at that  
offset and automatically advance the position. This enables in-place  
record updates and binary patch operations that are impossible with  
sequential stream APIs. When combining reads and writes at different  
positions, always call `position` explicitly before each operation to  
avoid reading data written in the previous step.  

---

## Reading from a URL

Fetch the text content of a remote URL using `scala.io.Source`.  

```scala
import scala.io.Source

@main def main() =
  val url = "https://httpbin.org/get"
  try
    val source = Source.fromURL(url, "UTF-8")
    try
      val content = source.mkString
      println(content.take(300))
    finally
      source.close()
  catch
    case e: java.io.IOException =>
      println(s"Network error: ${e.getMessage}")
end main
```

`Source.fromURL` opens an HTTP connection, reads the response body, and  
returns a `Source` whose `mkString` collapses everything into a single  
string. The outer `try`/`catch` handles connectivity failures such as DNS  
resolution errors and refused connections. The inner `try`/`finally`  
closes the source regardless of whether reading succeeds. This approach  
is convenient for small, text-oriented responses; for large downloads or  
binary content, use `URL.openConnection` directly and stream the response  
body.  

## Downloading a file

Stream bytes from a remote URL and save them directly to a local file  
using `Files.copy`.  

```scala
import java.net.URL
import java.nio.file.{Files, StandardCopyOption}

@main def main() =
  val url  = new URL("https://httpbin.org/bytes/1024")
  val dest = Files.createTempFile("download", ".bin")
  try
    val conn = url.openConnection()
    conn.setConnectTimeout(5000)
    conn.setReadTimeout(10000)
    val in = conn.getInputStream
    try
      Files.copy(in, dest, StandardCopyOption.REPLACE_EXISTING)
      println(s"Downloaded ${Files.size(dest)} bytes to $dest")
    finally
      in.close()
  catch
    case e: java.io.IOException =>
      println(s"Download failed: ${e.getMessage}")
  finally
    Files.deleteIfExists(dest)
end main
```

`Files.copy(InputStream, Path)` streams bytes from the connection directly  
to the file without loading them all into heap memory first, making this  
approach safe for arbitrarily large downloads. Setting both  
`connectTimeout` and `readTimeout` before calling `getInputStream` ensures  
the download cannot hang indefinitely. `REPLACE_EXISTING` overwrites any  
partial file left by a previous failed attempt, which simplifies retry  
logic.  

## Handling connection timeouts

Configure connect and read timeouts on an `HttpURLConnection` to avoid  
hanging threads.  

```scala
import java.net.{URL, HttpURLConnection}

@main def main() =
  val url  = new URL("https://httpbin.org/delay/0")
  val conn = url.openConnection().asInstanceOf[HttpURLConnection]
  conn.setConnectTimeout(3000)
  conn.setReadTimeout(5000)
  try
    conn.connect()
    val code = conn.getResponseCode
    println(s"HTTP status: $code")
    println(s"Content-Type: ${conn.getContentType}")
  catch
    case e: java.net.SocketTimeoutException =>
      println(s"Timed out: ${e.getMessage}")
    case e: java.io.IOException =>
      println(s"IO error: ${e.getMessage}")
  finally
    conn.disconnect()
end main
```

`setConnectTimeout` limits how long the JVM waits to establish the TCP  
connection. `setReadTimeout` limits how long it waits for data after the  
connection is open. Both values are in milliseconds, and a value of zero  
means wait indefinitely — which is the unsafe default. In production  
services, always set both timeouts before calling `connect` or  
`getInputStream`. `SocketTimeoutException` extends `IOException`, so both  
can be caught by a single `IOException` handler if distinguishing them is  
not necessary.  

## Following redirects

Configure `HttpURLConnection` to follow HTTP 3xx redirects automatically.  

```scala
import java.net.{URL, HttpURLConnection}

@main def main() =
  val url  = new URL("https://httpbin.org/redirect/1")
  val conn = url.openConnection().asInstanceOf[HttpURLConnection]
  conn.setInstanceFollowRedirects(true)
  conn.setConnectTimeout(5000)
  conn.setReadTimeout(5000)
  try
    val code = conn.getResponseCode
    println(s"Final status: $code")
    println(s"Final URL:    ${conn.getURL}")
  catch
    case e: java.io.IOException =>
      println(s"Error: ${e.getMessage}")
  finally
    conn.disconnect()
end main
```

`HttpURLConnection` follows redirects by default at the class level via  
`HttpURLConnection.setFollowRedirects(true)`, but that static setting  
affects all connections in the JVM. `setInstanceFollowRedirects` overrides  
the behaviour for one connection only, which is the safer choice in  
multi-threaded applications. The JVM follows up to a fixed number of  
redirects (typically five) before throwing an exception, preventing  
infinite redirect loops. After following redirects, `conn.getURL` returns  
the final URL, not the original one.  

---

## Reading CSV manually

Parse a comma-separated values file line by line without an external  
library.  

```scala
import java.nio.file.Files
import scala.jdk.CollectionConverters.*

@main def main() =
  val path = Files.createTempFile("data", ".csv")
  Files.writeString(path,
    "name,age,city\nAlice,30,London\nBob,25,Paris\nCarol,35,Berlin")

  val lines  = Files.readAllLines(path).asScala
  val header = lines.head.split(",").toList

  lines.tail.foreach { line =>
    val fields = line.split(",").toList
    val record = header.zip(fields).toMap
    println(record)
  }

  Files.deleteIfExists(path)
end main
```

The header row is split first to establish the field names. Each  
subsequent line is split on commas and zipped with the header list to  
produce a `Map[String, String]`. This simple approach works well for  
CSV files with no quoted fields, no embedded commas, and no embedded  
newlines. Real CSV files from spreadsheet applications often contain  
quoted values with embedded commas; in those cases, a purpose-built CSV  
library provides correct parsing with far less code.  

## Writing CSV

Serialise a list of case class instances to a CSV file with a header row.  

```scala
import java.nio.file.Files

@main def main() =
  case class Person(name: String, age: Int, city: String)

  val people = List(
    Person("Alice", 30, "London"),
    Person("Bob",   25, "Paris"),
    Person("Carol", 35, "Berlin")
  )

  val path   = Files.createTempFile("out", ".csv")
  val header = "name,age,city"
  val rows   = people.map(p => s"${p.name},${p.age},${p.city}")
  val csv    = (header :: rows).mkString("\n")

  Files.writeString(path, csv)
  println(Files.readString(path))
  Files.deleteIfExists(path)
end main
```

Building each row as a string with `s"${p.name},${p.age},${p.city}"` is  
safe only when the field values are guaranteed not to contain commas,  
double-quote characters, or newlines. When those characters may appear,  
surround each field with double quotes and escape any embedded quotes by  
doubling them. For anything beyond trivial serialisation, a CSV library  
handles these edge cases reliably and produces standard-compliant output.  

## Parsing TSV

Read a tab-separated values file and split each row on the tab character.  

```scala
import java.nio.file.Files
import scala.jdk.CollectionConverters.*

@main def main() =
  val path = Files.createTempFile("data", ".tsv")
  Files.writeString(path,
    "product\tprice\tstock\nWidget\t9.99\t100\nGadget\t24.50\t50")

  val lines  = Files.readAllLines(path).asScala
  val header = lines.head.split("\t").toList

  lines.tail.foreach { row =>
    val fields = row.split("\t").toList
    val record = header.zip(fields).toMap
    println(record)
  }

  Files.deleteIfExists(path)
end main
```

TSV files use the tab character as a field separator, which avoids the  
ambiguity of commas inside quoted strings and makes the format simpler  
to parse manually. The pattern is identical to CSV parsing with the  
separator character changed to `"\t"`. Tab-separated formats are common  
in scientific data exports, database dumps, and spreadsheet applications  
that offer TSV as an alternative to CSV. Like CSV, TSV has no universal  
standard, so real-world files may embed escaped tabs or use different  
newline conventions.  

## Reading JSON from a file

Load raw JSON text from a file and extract field values using simple  
string operations without an external library.  

```scala
import java.nio.file.Files

@main def main() =
  val json = """{"name":"Alice","age":30,"active":true}"""
  val path = Files.createTempFile("data", ".json")
  Files.writeString(path, json)

  val content = Files.readString(path)

  val name = content
    .split("\"name\":\"").last
    .takeWhile(_ != '"')

  val age = content
    .split("\"age\":").last
    .takeWhile(_.isDigit)

  println(s"Name: $name")
  println(s"Age:  $age")

  Files.deleteIfExists(path)
end main
```

String-splitting is useful for quickly extracting a known field from a  
simple, predictable JSON structure in scripts and prototypes. It breaks  
immediately on any JSON with escaped characters, nested objects, or arrays.  
For production code, use a JSON library such as circe, ujson, or the  
built-in `javax.json` API to parse the document into a proper AST.  
The example is intentionally minimal to illustrate the file-reading  
mechanics rather than JSON parsing.  

## Writing JSON to a file

Serialise a case class to a JSON string using string interpolation and  
write it to a file.  

```scala
import java.nio.file.Files

@main def main() =
  case class Config(host: String, port: Int, debug: Boolean)

  val cfg = Config("localhost", 8080, false)

  val json =
    s"""|{
        |  "host":  "${cfg.host}",
        |  "port":  ${cfg.port},
        |  "debug": ${cfg.debug}
        |}""".stripMargin

  val path = Files.createTempFile("config", ".json")
  Files.writeString(path, json)

  println(Files.readString(path))
  Files.deleteIfExists(path)
end main
```

`stripMargin` removes the leading pipe characters and any whitespace  
before them, producing a cleanly indented JSON document. This hand-rolled  
approach is appropriate for fixed-schema configuration output where the  
structure is known at compile time and no dynamic key generation is  
needed. String interpolation handles `Boolean` and numeric values  
without quotes automatically. For dynamically structured data, a proper  
JSON library provides escaping, nesting, and serialisation safety.  

## Simple JSON validation

Check whether a string contains balanced JSON delimiters as a basic  
structural validity test.  

```scala
import java.nio.file.Files

@main def main() =
  def isBalanced(s: String): Boolean =
    val stack = scala.collection.mutable.Stack[Char]()
    val pairs = Map('}' -> '{', ']' -> '[', ')' -> '(')
    val ok = s.forall {
      case c @ ('{' | '[' | '(') => stack.push(c); true
      case c @ ('}' | ']' | ')') =>
        stack.nonEmpty && stack.pop() == pairs(c)
      case _ => true
    }
    ok && stack.isEmpty

  val valid   = """{"key": [1, 2, 3], "nested": {"ok": true}}"""
  val invalid = """{"key": [1, 2, 3"""

  println(s"Valid:   ${isBalanced(valid)}")
  println(s"Invalid: ${isBalanced(invalid)}")

  val path = Files.createTempFile("check", ".json")
  Files.writeString(path, valid)
  println(s"From file: ${isBalanced(Files.readString(path))}")
  Files.deleteIfExists(path)
end main
```

The function iterates over every character, pushing opening delimiters  
onto a stack and checking that each closing delimiter matches the top of  
the stack. If the stack is empty at the end and no mismatch was found,  
the delimiters are balanced. This check does not validate JSON value  
syntax, string escaping, or number formats; it only confirms that braces  
and brackets are properly nested. A real validator such as  
`javax.json.Json.createReader` parses the full grammar and reports  
meaningful error locations.  

---

## Choosing buffer sizes

Measure the effect of different IO buffer sizes on file copy throughput.  

```scala
import java.io.{FileInputStream, FileOutputStream}
import java.nio.file.Files

@main def main() =
  val src = Files.createTempFile("perf", ".dat")
  Files.write(src, Array.fill(1 * 1024 * 1024)(0x41.toByte))  // 1 MB

  def copyWithBuffer(bufSize: Int): Long =
    val dst = Files.createTempFile("pdst", ".dat")
    val t0  = System.nanoTime()
    val in  = FileInputStream(src.toFile)
    val out = FileOutputStream(dst.toFile)
    val buf = Array.ofDim[Byte](bufSize)
    try
      var n = in.read(buf)
      while n != -1 do
        out.write(buf, 0, n)
        n = in.read(buf)
    finally
      in.close()
      out.close()
    Files.deleteIfExists(dst)
    System.nanoTime() - t0

  List(512, 8192, 65536).foreach { size =>
    val ms = copyWithBuffer(size) / 1_000_000.0
    println(f"Buffer $size%7d bytes: $ms%.2f ms")
  }

  Files.deleteIfExists(src)
end main
```

A 512-byte buffer forces many more system calls than a 64 KiB buffer for  
the same data, and the timing difference is measurable even at one  
megabyte. Buffers larger than 64 KiB yield diminishing returns because  
the OS page cache and kernel socket buffers operate in similar size  
ranges. A good default for general file IO is 8 KiB, which matches the  
typical filesystem block size and the internal buffer of  
`BufferedInputStream` and `BufferedWriter`.  

## Avoiding small reads and writes

Demonstrate why single-byte IO is expensive and how a wrapper buffer  
eliminates the overhead.  

```scala
import java.io.{BufferedOutputStream, FileOutputStream}
import java.nio.file.Files

@main def main() =
  val data = Array.fill(100_000)(0x61.toByte)  // 100 KB of 'a'

  val pathRaw = Files.createTempFile("raw", ".dat")
  val t0 = System.nanoTime()
  val raw = FileOutputStream(pathRaw.toFile)
  try  data.foreach(b => raw.write(b.toInt))
  finally raw.close()
  val rawMs = (System.nanoTime() - t0) / 1_000_000.0

  val pathBuf = Files.createTempFile("buf", ".dat")
  val t1 = System.nanoTime()
  val buf = BufferedOutputStream(FileOutputStream(pathBuf.toFile), 8192)
  try  data.foreach(b => buf.write(b.toInt))
  finally buf.close()
  val bufMs = (System.nanoTime() - t1) / 1_000_000.0

  println(f"Unbuffered: $rawMs%.2f ms")
  println(f"Buffered:   $bufMs%.2f ms")

  Files.deleteIfExists(pathRaw)
  Files.deleteIfExists(pathBuf)
end main
```

Each single-byte `write` on an unbuffered `FileOutputStream` eventually  
calls through the JVM native layer. The `BufferedOutputStream` wrapper  
collects individual bytes until its internal 8 KiB buffer is full and  
then flushes them in a single underlying write. The resulting reduction  
in system calls produces a substantial speedup even on modern hardware  
with fast SSDs. The same principle applies to reads: `BufferedInputStream`  
wrapping `FileInputStream` can reduce read system calls by orders of  
magnitude for character-by-character processing.  

## Avoiding Source.fromFile in servers

Show why `scala.io.Source.fromFile` is unsuitable for repeated calls in  
long-running services, and how `Files.readString` replaces it safely.  

```scala
import scala.io.Source
import java.nio.file.Files
import java.nio.charset.StandardCharsets

@main def main() =
  val path = Files.createTempFile("config", ".txt")
  Files.writeString(path, "key=value\nmode=production")

  // Problematic: source opened but never explicitly closed
  def readConfigLeaky(): String =
    Source.fromFile(path.toFile, "UTF-8").mkString

  // Safe alternative using Using
  def readConfigSafe(): String =
    scala.util.Using.resource(
      Source.fromFile(path.toFile, "UTF-8")
    )(_.mkString)

  // Even simpler: NIO.2 one-liner that manages resources internally
  def readConfigNio(): String =
    Files.readString(path, StandardCharsets.UTF_8)

  println(readConfigSafe())
  println(readConfigNio())

  Files.deleteIfExists(path)
end main
```

`Source.fromFile` opens a file descriptor on every call. If the return  
value is used once and discarded without calling `close`, the descriptor  
leaks. A server handling thousands of requests per minute can exhaust the  
OS per-process file descriptor limit within minutes, causing  
`IOException: too many open files`. `Using.resource` closes the source  
immediately after the block regardless of exceptions. `Files.readString`  
is the simplest option: it manages the underlying channel internally and  
returns the string directly.  

## Using buffered readers and writers

Wrap file readers and writers in buffered adapters and choose the  
appropriate buffer size for the workload.  

```scala
import java.io.{BufferedReader, BufferedWriter, FileReader, FileWriter}
import java.nio.charset.StandardCharsets
import java.nio.file.Files

@main def main() =
  val path = Files.createTempFile("rw", ".txt")

  // Buffered writing
  val writer = BufferedWriter(FileWriter(path.toFile), 8192)
  try
    for i <- 1 to 500 do
      writer.write(s"record-$i\n")
  finally
    writer.close()

  // Buffered reading
  val reader = BufferedReader(
    FileReader(path.toFile, StandardCharsets.UTF_8), 8192)
  var count = 0
  try
    var line = reader.readLine()
    while line != null do
      count += 1
      line = reader.readLine()
  finally
    reader.close()

  println(s"Wrote and read $count lines")
  Files.deleteIfExists(path)
end main
```

`BufferedWriter` and `BufferedReader` are the standard wrappers for  
`FileWriter` and `FileReader` respectively. The optional second integer  
argument sets the buffer capacity in characters; the default of 8192  
characters is a sensible starting point for most workloads. Always  
specify the charset when constructing `FileReader` and `FileWriter`  
rather than relying on the platform default. In Java 11 and later, the  
constructors that accept a `Charset` directly make this more convenient  
than wrapping in `InputStreamReader`.  

## Streaming large files safely

Process a file too large to load into memory by reading it one line at a  
time with a `BufferedReader`.  

```scala
import java.io.{BufferedReader, FileReader}
import java.nio.charset.StandardCharsets
import java.nio.file.Files
import scala.util.Using

@main def main() =
  val path = Files.createTempFile("large", ".txt")
  val out  = Files.newBufferedWriter(path)
  try
    for i <- 1 to 100_000 do
      out.write(s"Line number $i\n")
  finally
    out.close()

  var lineCount = 0L
  var charCount = 0L

  Using(BufferedReader(FileReader(path.toFile, StandardCharsets.UTF_8))) { br =>
    var line = br.readLine()
    while line != null do
      lineCount += 1
      charCount += line.length
      line = br.readLine()
  }

  println(s"Lines: $lineCount")
  println(s"Chars: $charCount")
  Files.deleteIfExists(path)
end main
```

`BufferedReader.readLine` reads one line at a time and keeps only that  
line in heap memory, so the maximum memory used stays proportional to  
the longest line, not the file size. This is the correct approach for  
log analysis, ETL pipelines, and any processing step that does not need  
random access. `Files.newBufferedWriter` is the NIO.2 equivalent of  
`BufferedWriter(FileWriter(...))` and uses the system default charset  
unless a charset argument is provided. `Using` closes the reader  
automatically when the block exits, even if an exception occurs.  
