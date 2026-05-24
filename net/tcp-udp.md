# Classic TCP/UDP Networking in Scala 3

Networking at the socket level means writing code that opens connections,  
sends and receives bytes, and interprets raw byte streams as structured  
messages. A socket is the operating-system abstraction that gives a program  
a handle to a network endpoint. Each socket is bound to an IP address and  
a port number; together they uniquely identify where bytes enter and leave  
the process on a given machine.  

TCP and UDP are the two transport-layer protocols used in most applications.  
TCP (Transmission Control Protocol) is connection-oriented: before any data  
travels, a three-way handshake (SYN, SYN-ACK, ACK) establishes the session.  
The kernel automatically retransmits lost segments and delivers them to the  
application in exact send order. UDP (User Datagram Protocol) is  
connectionless: each datagram is dispatched independently, may arrive out  
of order, and may be silently dropped. UDP trades reliability for lower  
latency and simpler code, which is why DNS queries, video streaming, and  
online games often prefer it.  

An IPv4 address is a 32-bit number written in dotted-decimal notation such  
as 93.184.216.34. IPv6 addresses are 128 bits written as eight groups of  
four hexadecimal digits. Port numbers range from 0 to 65535; ports below  
1024 are called well-known ports and require elevated privileges to bind on  
most operating systems. DNS (Domain Name System) translates hostnames like  
example.com into IP addresses. Routing determines the path from source to  
destination by consulting the routing table in each network device. The TTL  
(time to live) field in an IP header limits how many router hops a packet  
may traverse before being discarded.  

In blocking I/O mode, a thread calling `accept`, `read`, or `write` suspends  
until the operation completes. Each connection therefore ties up one thread,  
which limits scalability to roughly the number of threads the JVM can  
sustain. In non-blocking mode (Java NIO), operations return immediately with  
a status code; a `Selector` monitors many channels for readiness and  
dispatches events to a single thread. The blocking `java.net` API covered  
in this chapter is simpler to understand and appropriate for moderate  
concurrency. NIO and AIO are covered in later chapters.  

TCP is a byte stream, not a message stream. The kernel may merge two  
consecutive writes into one segment or split one write across many packets.  
A single `read` on the receiving end may therefore return an arbitrary  
number of bytes, not aligned to any write boundary. Application code must  
impose its own message framing. Common strategies include terminating  
messages with a newline, using a unique delimiter byte sequence, sending  
fixed-length records, or prefixing each message with a four-byte integer  
that encodes the payload length. UDP, by contrast, preserves datagram  
boundaries: one `send` equals one `receive` of the same packet, or no  
receive at all if the packet is dropped.  

The most common pitfall when reading from a TCP socket is assuming that a  
single `read` call delivers a complete message. Always loop until the  
required number of bytes has arrived or end-of-stream is detected. Timeouts  
prevent threads from blocking indefinitely when a peer stops responding.  
Wrapping streams in `BufferedInputStream` and `BufferedOutputStream` reduces  
system-call overhead for small transfers. Forgetting to flush a buffered  
output stream leaves data trapped in the buffer. Always close sockets in a  
`finally` block to prevent file-descriptor leaks even when exceptions occur.  

The examples in this chapter use the blocking `java.net` API exclusively.  
They progress from raw socket creation through complete client-server pairs,  
message-framing patterns, UDP, DNS resolution, network diagnostics, and  
error-handling strategies. Mastering this layer provides the mental model  
that makes the NIO Selector and AIO chapters easier to grasp later.  

---

## Creating a TCP client

Connect to a remote host and port using the two-argument `Socket` constructor.  

```scala
@main def main() =
    import java.net.Socket

    val socket = Socket("example.com", 80)
    println(s"Connected:  ${socket.isConnected}")
    println(s"Remote:     ${socket.getRemoteSocketAddress}")
    println(s"Local port: ${socket.getLocalPort}")
    socket.close()
end main
```

The `Socket` constructor resolves the hostname using the system DNS resolver  
and performs a TCP three-way handshake before returning. The OS assigns an  
ephemeral local port from its available pool automatically. `isConnected`  
confirms the connection is live. Calling `close` sends a TCP FIN segment to  
the remote end and releases the OS file descriptor. This two-argument form  
is the most concise way to open a TCP connection in Java-based code.  

## Connecting to a server

Use a no-argument `Socket` and `connect` to set an explicit connection timeout.  

```scala
@main def main() =
    import java.net.{Socket, InetSocketAddress}

    val socket  = Socket()
    val address = InetSocketAddress("example.com", 80)
    socket.connect(address, 5000)   // 5-second timeout
    println(s"Connected to ${socket.getRemoteSocketAddress}")
    socket.close()
end main
```

Creating `Socket()` with no arguments produces an unconnected socket.  
Calling `connect` with an `InetSocketAddress` and a timeout in milliseconds  
gives fine-grained control: if the handshake does not complete within the  
allowed time, a `SocketTimeoutException` is thrown and the socket remains  
closeable. This two-step approach is preferred over the single-argument  
constructor whenever connection timeouts are required. `InetSocketAddress`  
can also wrap a pre-resolved `InetAddress`, avoiding DNS overhead on  
repeated connections to the same host.  

## Reading bytes from a socket

Issue an HTTP request and read the raw response bytes from the socket input stream.  

```scala
@main def main() =
    import java.net.Socket

    val socket = Socket("example.com", 80)
    val out    = socket.getOutputStream
    val req    = "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n"
    out.write(req.getBytes("UTF-8"))
    out.flush()

    val in  = socket.getInputStream
    val buf = Array.ofDim[Byte](4096)
    val n   = in.read(buf)
    println(s"Read $n bytes")
    println(new String(buf, 0, n, "UTF-8"))
    socket.close()
end main
```

`getInputStream` returns the raw `InputStream` associated with the socket.  
The `read` call fills the byte array and returns the actual count of bytes  
placed into it, which may be fewer than the buffer capacity. The example  
sends a minimal HTTP/1.0 GET request first so the server has reason to  
respond. Specifying the charset explicitly in `new String` avoids  
platform-default encoding surprises. A production HTTP client would loop  
until all expected bytes arrive, because a single `read` rarely delivers  
the full response.  

## Writing bytes to a socket

Send raw bytes from a client to a server using the socket output stream.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}

    val server = ServerSocket(9001)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val buf    = Array.ofDim[Byte](256)
            val n      = client.getInputStream.read(buf)
            println(s"Server received: ${new String(buf, 0, n, "UTF-8")}")
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket  = Socket("localhost", 9001)
    val payload = "Hello from client".getBytes("UTF-8")
    socket.getOutputStream.write(payload)
    socket.getOutputStream.flush()
    socket.close()
    serverThread.join()
end main
```

This self-contained example starts a minimal server in a background thread  
before the client connects. The client obtains its `OutputStream` with  
`getOutputStream`, writes a byte array, and calls `flush` to ensure the  
bytes reach the OS network buffer immediately. The server reads the bytes  
and prints them after decoding. `serverThread.join` keeps the main thread  
alive until the server has finished processing before the program exits.  

## Flushing output streams

Wrap a socket output stream in `BufferedOutputStream` and flush explicitly  
after writing.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.BufferedOutputStream

    val server = ServerSocket(9002)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val buf    = Array.ofDim[Byte](256)
            val n      = client.getInputStream.read(buf)
            println(s"Received: ${new String(buf, 0, n, "UTF-8")}")
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9002)
    val out    = BufferedOutputStream(socket.getOutputStream)
    out.write("Buffered data".getBytes("UTF-8"))
    out.flush()   // Without this, data stays in the buffer
    socket.close()
    serverThread.join()
end main
```

A `BufferedOutputStream` accumulates bytes in an in-process buffer and  
delivers them to the OS in larger, more efficient batches. The trade-off  
is that data written to the buffer is not transmitted until the buffer  
fills or `flush` is called explicitly. Forgetting to flush is one of the  
most common networking bugs: the program appears to hang because the remote  
peer never receives the message. Always call `flush` after writing a  
complete message or command, and close streams in a `finally` block so the  
JVM flushes on exit.  

## Handling end-of-stream

Loop until `read` returns −1 to consume the entire incoming TCP stream.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}

    val server = ServerSocket(9003)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            client.getOutputStream.write("Hello\nWorld\n".getBytes("UTF-8"))
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9003)
    val in     = socket.getInputStream
    val buf    = Array.ofDim[Byte](128)
    val sb     = StringBuilder()
    var n      = in.read(buf)
    while n != -1 do
        sb.append(new String(buf, 0, n, "UTF-8"))
        n = in.read(buf)
    socket.close()
    println(sb.toString)
    serverThread.join()
end main
```

When the remote end closes its sending side, subsequent `read` calls return  
−1 to signal end-of-stream. A while loop that accumulates each batch of  
bytes into a `StringBuilder` is the standard pattern for reading an entire  
stream. End-of-stream does not necessarily mean the connection is broken;  
it simply means the peer has finished sending data. The client may still  
write back through its own output stream if the protocol requires a reply  
before closing.  

## Closing sockets safely

Use `try`/`finally` to guarantee a socket is closed even when exceptions occur.  

```scala
@main def main() =
    import java.net.Socket

    val socket = Socket("example.com", 80)
    try
        val out = socket.getOutputStream
        val req = "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n"
        out.write(req.getBytes("UTF-8"))
        out.flush()
        val buf = Array.ofDim[Byte](1024)
        val n   = socket.getInputStream.read(buf)
        println(s"Read $n bytes")
    finally
        socket.close()
end main
```

Placing the socket inside a `try`/`finally` block guarantees that `close`  
is called whether the code succeeds or throws an exception. Failing to  
close a socket leaks the underlying OS file descriptor, and on systems with  
low descriptor limits this eventually causes "too many open files" errors.  
The `close` method on `Socket` also closes both the input and output streams  
associated with it. When multiple resources need releasing, close them in  
reverse order of creation.  

## Setting socket timeouts

Configure `setSoTimeout` to prevent a blocking `read` from waiting indefinitely.  

```scala
@main def main() =
    import java.net.{Socket, SocketTimeoutException}

    val socket = Socket("example.com", 80)
    socket.setSoTimeout(3000)   // 3-second read timeout
    try
        val out = socket.getOutputStream
        out.write("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n".getBytes("UTF-8"))
        out.flush()
        val buf = Array.ofDim[Byte](4096)
        val n   = socket.getInputStream.read(buf)
        println(s"Read $n bytes")
    catch
        case _: SocketTimeoutException =>
            println("Read timed out after 3 seconds")
    finally
        socket.close()
end main
```

`setSoTimeout` sets the maximum milliseconds a blocking `read` will wait  
before throwing `SocketTimeoutException`. The socket itself remains open  
after a timeout; the caller can retry the read, apply fallback logic, or  
close the connection. A timeout of zero (the default) means wait forever.  
Setting a sensible timeout is essential in production because a misbehaving  
peer that stops sending without closing the connection would otherwise block  
the reading thread permanently.  

## Enabling SO_REUSEADDR

Set `SO_REUSEADDR` on a server socket before binding to allow port reuse  
after restart.  

```scala
@main def main() =
    import java.net.ServerSocket

    val server = ServerSocket()
    server.setReuseAddress(true)
    server.bind(java.net.InetSocketAddress(9004))
    println(s"Bound to ${server.getLocalSocketAddress}")
    println(s"SO_REUSEADDR: ${server.getReuseAddress}")
    server.close()
end main
```

After a server socket is closed, the OS keeps the local port in a TIME_WAIT  
state for up to two minutes to handle late-arriving packets. Attempting to  
rebind the same port during this period normally raises `BindException` with  
the message "Address already in use". Enabling `SO_REUSEADDR` before calling  
`bind` allows the port to be reused immediately. This is a standard setting  
for any server that may need to restart quickly. The option must be set  
before `bind`, which is why the no-argument constructor is used first.  

## Reading text lines from a socket

Wrap a socket input stream in `BufferedReader` to receive complete lines.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{BufferedReader, InputStreamReader, PrintWriter}

    val server = ServerSocket(9005)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val w = PrintWriter(client.getOutputStream, true)
            w.println("Line one from server")
            w.println("Line two from server")
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9005)
    val reader = BufferedReader(InputStreamReader(socket.getInputStream, "UTF-8"))
    var line   = reader.readLine()
    while line != null do
        println(s"Client got: $line")
        line = reader.readLine()
    socket.close()
    serverThread.join()
end main
```

`BufferedReader.readLine` reads characters until it encounters `\n`, `\r`,  
or `\r\n` and returns the line without the terminator. When the stream ends  
it returns `null`, which the while loop uses as its exit condition. Wrapping  
the raw `InputStream` in an `InputStreamReader` with an explicit charset  
converts bytes to characters before `BufferedReader` sees them. This layered  
wrapping (socket → InputStream → InputStreamReader → BufferedReader) is the  
canonical Java pattern for text-line protocols.  

## Writing text lines to a socket

Use `PrintWriter` with auto-flush to send newline-terminated text lines.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{BufferedReader, InputStreamReader, PrintWriter}

    val server = ServerSocket(9006)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val reader = BufferedReader(
                InputStreamReader(client.getInputStream, "UTF-8"))
            var line = reader.readLine()
            while line != null do
                println(s"Server got: $line")
                line = reader.readLine()
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9006)
    val writer = PrintWriter(socket.getOutputStream, true)   // auto-flush
    writer.println("First message")
    writer.println("Second message")
    socket.close()
    serverThread.join()
end main
```

`PrintWriter` with `autoFlush = true` flushes the internal buffer after  
every call to `println`, which appends a `\n` to the text. The server sees  
each message as a complete line without the client needing an explicit  
`flush` call. The auto-flush parameter applies only to `println`, `printf`,  
and `format`; plain `print` and `write` calls do not trigger it. For  
protocols where the client sends many small messages rapidly, disabling  
auto-flush and calling `flush` manually after a batch reduces the number  
of system calls.  

---

## Creating a TCP server socket

Bind a `ServerSocket` to a local port and prepare it to accept connections.  

```scala
@main def main() =
    import java.net.ServerSocket

    val server = ServerSocket(9007)
    println(s"Listening on port ${server.getLocalPort}")
    println(s"Bound:  ${server.isBound}")
    println(s"Closed: ${server.isClosed}")
    server.close()
end main
```

`ServerSocket(port)` binds to all local interfaces on the given port and  
sets a default listen backlog, which controls how many pending connections  
the OS queues before refusing new ones. The server is ready to accept  
connections immediately after construction. `isBound` confirms the port  
assignment succeeded and `isClosed` shows the socket is still active. No  
client connections are accepted yet; that requires calling `accept`.  

## Accepting a single client

Call `accept` once to handle exactly one incoming TCP connection.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{BufferedReader, InputStreamReader, PrintWriter}

    val server = ServerSocket(9008)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            println(s"Client from: ${client.getRemoteSocketAddress}")
            PrintWriter(client.getOutputStream, true).println("Welcome")
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9008)
    val line = BufferedReader(
        InputStreamReader(socket.getInputStream, "UTF-8")).readLine()
    println(s"Received: $line")
    socket.close()
    serverThread.join()
end main
```

`accept` blocks the calling thread until a client completes the TCP  
handshake. It then returns a new `Socket` representing that specific  
connection, leaving the `ServerSocket` ready to accept further connections.  
The original `ServerSocket` and the returned per-client `Socket` are  
independent objects; closing one does not close the other. Printing the  
remote socket address at acceptance gives useful diagnostic information  
about where clients are connecting from.  

## Echo server (single client)

Accept one connection and echo every line back to the sender.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{BufferedReader, InputStreamReader, PrintWriter}

    val server = ServerSocket(9009)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val reader = BufferedReader(
                InputStreamReader(client.getInputStream, "UTF-8"))
            val writer = PrintWriter(client.getOutputStream, true)
            var line   = reader.readLine()
            while line != null do
                writer.println(line)
                line = reader.readLine()
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9009)
    val writer = PrintWriter(socket.getOutputStream, true)
    val reader = BufferedReader(InputStreamReader(socket.getInputStream, "UTF-8"))
    writer.println("Hello")
    println(s"Echo: ${reader.readLine()}")
    writer.println("World")
    println(s"Echo: ${reader.readLine()}")
    socket.close()
    serverThread.join()
end main
```

An echo server is the network equivalent of "Hello, World". The server reads  
each line and immediately writes it back unchanged. The client sends two  
lines and prints the echoed responses. Both sides use auto-flush `PrintWriter`  
so neither needs explicit flush calls. This minimal pattern verifies that  
both the send and receive paths work correctly before more complex  
application-level logic is built on top.  

## Echo server (multiple clients)

Loop over `accept` to serve each incoming client sequentially.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{BufferedReader, InputStreamReader, PrintWriter}

    val server = ServerSocket(9010)
    val serverThread = new Thread:
        override def run() =
            var remaining = 2
            while remaining > 0 do
                val client = server.accept()
                val reader = BufferedReader(
                    InputStreamReader(client.getInputStream, "UTF-8"))
                val writer = PrintWriter(client.getOutputStream, true)
                var line   = reader.readLine()
                while line != null do
                    writer.println(line)
                    line = reader.readLine()
                client.close()
                remaining -= 1
            server.close()
    serverThread.start()
    Thread.sleep(50)

    for i <- 1 to 2 do
        val socket = Socket("localhost", 9010)
        val writer = PrintWriter(socket.getOutputStream, true)
        val reader = BufferedReader(
            InputStreamReader(socket.getInputStream, "UTF-8"))
        writer.println(s"Client $i says hello")
        println(s"Client $i echo: ${reader.readLine()}")
        socket.close()
    serverThread.join()
end main
```

Wrapping `accept` in a loop turns the server into one that serves multiple  
clients one at a time. The server blocks on the inner while loop until each  
client disconnects, then loops back to `accept` the next one. Sequential  
handling is simple and safe but means a slow client blocks all waiting  
clients. For moderate loads this is perfectly adequate; for high concurrency  
a thread-per-connection or non-blocking design is needed.  

## Thread-per-connection server

Spawn a new thread for each accepted connection to serve clients concurrently.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{BufferedReader, InputStreamReader, PrintWriter}

    def handleClient(client: Socket): Unit =
        val reader = BufferedReader(
            InputStreamReader(client.getInputStream, "UTF-8"))
        val writer = PrintWriter(client.getOutputStream, true)
        var line   = reader.readLine()
        while line != null do
            writer.println(s"Echo: $line")
            line = reader.readLine()
        client.close()

    val server = ServerSocket(9011)
    val serverThread = new Thread:
        override def run() =
            var count = 2
            while count > 0 do
                val client = server.accept()
                val t = new Thread:
                    override def run() = handleClient(client)
                t.start()
                count -= 1
            server.close()
    serverThread.start()
    Thread.sleep(50)

    def makeClient(msg: String): Thread =
        val t = new Thread:
            override def run() =
                val socket = Socket("localhost", 9011)
                val writer = PrintWriter(socket.getOutputStream, true)
                val reader = BufferedReader(
                    InputStreamReader(socket.getInputStream, "UTF-8"))
                writer.println(msg)
                println(reader.readLine())
                socket.close()
        t.start()
        t
    val t1 = makeClient("Hello from thread 1")
    val t2 = makeClient("Hello from thread 2")
    t1.join(); t2.join(); serverThread.join()
end main
```

Each accepted connection is handed to a dedicated thread, so all clients  
are served in parallel. The server loop calls `accept` and immediately  
delegates the returned `Socket` to a new thread before looping back. The  
top-level `handleClient` function keeps the server loop concise. This design  
scales to hundreds of concurrent connections but not to thousands, because  
each thread consumes hundreds of kilobytes of stack memory. For higher  
concurrency, a thread pool or NIO selector is the appropriate upgrade.  

## Handling disconnects

Catch `IOException` in the read loop to detect unexpected client disconnections.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{BufferedReader, InputStreamReader, IOException}

    val server = ServerSocket(9012)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val reader = BufferedReader(
                InputStreamReader(client.getInputStream, "UTF-8"))
            try
                var line = reader.readLine()
                while line != null do
                    println(s"Server: $line")
                    line = reader.readLine()
                println("Client closed connection normally")
            catch
                case e: IOException =>
                    println(s"Client disconnected abruptly: ${e.getMessage}")
            finally
                client.close()
                server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9012)
    socket.getOutputStream.write("Hello\n".getBytes("UTF-8"))
    socket.getOutputStream.flush()
    socket.close()
    serverThread.join()
end main
```

A normal disconnect happens when the client closes the socket cleanly:  
`readLine` returns `null` and the while loop exits gracefully. An  
unexpected disconnect — caused by a crash, network failure, or OS signal —  
raises an `IOException` during the next read operation. Catching both  
conditions and logging them helps operators diagnose connection issues.  
The `finally` block ensures the per-client socket and server socket are  
released regardless of how the loop exits.  

## Detecting client closure

Check for `null` from `readLine` or −1 from `read` to detect that the client  
has closed its sending side.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{BufferedReader, InputStreamReader}

    val server = ServerSocket(9013)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val reader = BufferedReader(
                InputStreamReader(client.getInputStream, "UTF-8"))
            var line = reader.readLine()
            while line != null do
                println(s"Server received: '$line'")
                line = reader.readLine()
            println(s"EOF: ${client.getRemoteSocketAddress} closed")
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9013)
    val out    = socket.getOutputStream
    out.write("Line A\n".getBytes("UTF-8"))
    out.write("Line B\n".getBytes("UTF-8"))
    out.flush()
    socket.close()
    serverThread.join()
end main
```

When a client calls `close()`, the TCP stack sends a FIN segment to the  
server. The server's `readLine` or `read` returns `null` or −1 as a result  
of receiving that FIN, signalling end-of-stream. The server should then  
close its own side of the connection and release resources. Detecting  
closure reliably requires always checking the return value of every read  
operation; relying on exceptions alone is insufficient because a clean close  
does not throw.  

## Graceful shutdown of server

Close the `ServerSocket` from another thread to unblock a waiting `accept`.  

```scala
@main def main() =
    import java.net.{ServerSocket, SocketException}

    val server = ServerSocket(9014)
    println(s"Server listening on ${server.getLocalPort}")

    val acceptThread = new Thread:
        override def run() =
            try
                while !server.isClosed do
                    val client = server.accept()
                    println(s"Accepted: ${client.getRemoteSocketAddress}")
                    client.close()
            catch
                case _: SocketException =>
                    println("Accept loop terminated by socket close")
    acceptThread.start()

    Thread.sleep(200)
    println("Initiating graceful shutdown …")
    server.close()
    acceptThread.join()
    println("Server stopped")
end main
```

Closing a `ServerSocket` from a separate thread causes any blocking `accept`  
call on it to throw `SocketException` with the message "Socket closed". This  
is the idiomatic Java pattern for stopping a server from outside the accept  
loop. Checking `server.isClosed` in the loop condition avoids one spurious  
accept attempt after the close. The shutdown thread joins the accept thread  
to confirm it has finished before declaring the server stopped.  

## Logging client connections

Print connection metadata — remote address, port, and timestamp — at accept.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.time.LocalDateTime

    val server = ServerSocket(9015)
    val serverThread = new Thread:
        override def run() =
            var count = 2
            while count > 0 do
                val client = server.accept()
                val remote = client.getRemoteSocketAddress
                val local  = client.getLocalSocketAddress
                val time   = LocalDateTime.now()
                println(s"[$time] Connection from $remote to $local")
                client.close()
                count -= 1
            server.close()
    serverThread.start()
    Thread.sleep(50)

    for _ <- 1 to 2 do
        val s = Socket("localhost", 9015)
        s.close()
    serverThread.join()
end main
```

Recording the remote and local socket addresses together with a timestamp  
at the moment of acceptance gives an audit trail of incoming connections.  
`getRemoteSocketAddress` returns an `InetSocketAddress` whose `toString`  
includes both the IP and the ephemeral port, uniquely identifying each  
session. This logging pattern is the foundation of network access logs in  
production servers. In a real server the entries would be written to a file,  
a database, or a structured logging framework.  

## Sending structured messages

Define a simple key=value line protocol and exchange typed records over TCP.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{BufferedReader, InputStreamReader, PrintWriter}

    case class Record(name: String, age: Int)

    def encode(r: Record): String = s"name=${r.name};age=${r.age}"
    def decode(s: String): Record =
        val m = s.split(";").map(_.split("=", 2)).map(a => a(0) -> a(1)).toMap
        Record(m("name"), m("age").toInt)

    val server = ServerSocket(9016)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val reader = BufferedReader(
                InputStreamReader(client.getInputStream, "UTF-8"))
            val rec = decode(reader.readLine())
            println(s"Server decoded: $rec")
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9016)
    PrintWriter(socket.getOutputStream, true).println(encode(Record("Alice", 30)))
    socket.close()
    serverThread.join()
end main
```

A key=value text protocol is simple to implement and easy to debug with  
standard tools like `nc` or `telnet`. The `encode` function converts a case  
class to a single line and `decode` parses it back. Each record is terminated  
by a newline so `readLine` can delimit messages without an explicit length  
prefix. This naive encoding is fragile if field values can contain semicolons  
or equals signs; in production, URI encoding or a proper serialisation  
library would handle escaping safely.  

---

## Line-based protocol

Frame messages with a newline character so each `readLine` delivers exactly  
one message.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{BufferedReader, InputStreamReader, PrintWriter}

    val server = ServerSocket(9017)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val reader = BufferedReader(
                InputStreamReader(client.getInputStream, "UTF-8"))
            var line = reader.readLine()
            while line != null do
                println(s"MSG: [$line]")
                line = reader.readLine()
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9017)
    val writer = PrintWriter(socket.getOutputStream, true)
    List("PING", "GET /index", "QUIT").foreach(writer.println)
    socket.close()
    serverThread.join()
end main
```

The newline character is the simplest and most universal message delimiter.  
HTTP/1.x, SMTP, FTP, and Redis all use line-based protocols for their  
command channels. `PrintWriter.println` appends `\n` automatically, and  
`BufferedReader.readLine` strips it on the other end. The main limitation is  
that message content must not contain bare newlines; binary data or  
multi-line payloads require an additional encoding step such as base64 or  
a two-part header/body structure.  

## Delimiter-based protocol

Use a custom multi-byte sentinel string to mark the end of each message.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.InputStream

    val DELIM = "||END||"

    def readUntilDelim(in: InputStream): String =
        val sb      = StringBuilder()
        val oneByte = Array.ofDim[Byte](1)
        var reading = true
        while reading && in.read(oneByte) != -1 do
            sb.append(new String(oneByte, "UTF-8"))
            if sb.toString.endsWith(DELIM) then reading = false
        sb.toString.stripSuffix(DELIM)

    val server = ServerSocket(9018)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val msg    = readUntilDelim(client.getInputStream)
            println(s"Server got: [$msg]")
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9018)
    val out    = socket.getOutputStream
    val text   = "Hello\nWorld" + DELIM   // payload contains a newline
    out.write(text.getBytes("UTF-8"))
    out.flush()
    socket.close()
    serverThread.join()
end main
```

A custom delimiter such as `||END||` can delimit messages that contain bare  
newlines, making it suitable for payloads like JSON snippets or log entries.  
The reader accumulates bytes until the accumulated string ends with the  
delimiter. This byte-by-byte approach is simple but slow for large messages;  
in practice a sliding-window search over a larger read buffer is more  
efficient. The delimiter itself must not appear in the payload without  
escaping.  

## Fixed-length frames

Send and receive messages of a fixed, pre-agreed byte length.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{InputStream, EOFException}

    val FRAME_SIZE = 32

    def readExact(in: InputStream, n: Int): Array[Byte] =
        val buf    = Array.ofDim[Byte](n)
        var offset = 0
        while offset < n do
            val got = in.read(buf, offset, n - offset)
            if got == -1 then throw EOFException(s"Expected $n bytes, got $offset")
            offset += got
        buf

    val server = ServerSocket(9019)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val frame  = readExact(client.getInputStream, FRAME_SIZE)
            println(s"Frame: '${new String(frame, "UTF-8").trim}'")
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9019)
    val raw    = "HELLO".getBytes("UTF-8")
    val frame  = Array.fill[Byte](FRAME_SIZE)(0)
    System.arraycopy(raw, 0, frame, 0, raw.length)
    socket.getOutputStream.write(frame)
    socket.getOutputStream.flush()
    socket.close()
    serverThread.join()
end main
```

Fixed-length framing is the simplest framing scheme: both sides agree on  
the frame size at design time. The sender pads short payloads with zero  
bytes and the receiver always reads exactly that many bytes. The `readExact`  
helper loops until the full frame arrives, accumulating partial reads  
correctly. This approach works well for binary protocols with fixed record  
structures such as sensor readings or financial ticks. Variable-length  
content requires a different strategy such as length-prefix framing.  

## Length-prefix frames

Prefix each message with a 4-byte integer that encodes the payload size.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{DataInputStream, DataOutputStream}

    def sendFrame(out: DataOutputStream, msg: String): Unit =
        val bytes = msg.getBytes("UTF-8")
        out.writeInt(bytes.length)
        out.write(bytes)
        out.flush()

    def recvFrame(in: DataInputStream): String =
        val len = in.readInt()
        val buf = Array.ofDim[Byte](len)
        in.readFully(buf)
        new String(buf, "UTF-8")

    val server = ServerSocket(9020)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val in     = DataInputStream(client.getInputStream)
            println(s"Server received: ${recvFrame(in)}")
            println(s"Server received: ${recvFrame(in)}")
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9020)
    val out    = DataOutputStream(socket.getOutputStream)
    sendFrame(out, "Short message")
    sendFrame(out, "A longer second message with more content here")
    socket.close()
    serverThread.join()
end main
```

A 4-byte big-endian integer length prefix is the most common binary framing  
technique. The receiver reads the 4-byte header first, learns the exact  
payload size, and then reads precisely that many bytes using  
`DataInputStream.readFully`, which loops internally until the count is  
satisfied or an EOF occurs. `DataOutputStream.writeInt` uses network byte  
order (big-endian) by default, which matches the TCP/IP standards. This  
scheme handles variable-length messages of up to 2 GB.  

## JSON messages over TCP

Encode messages as JSON strings with a length prefix and decode them on arrival.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{DataInputStream, DataOutputStream}

    // Minimal JSON helpers — no external dependencies required
    def toJson(m: Map[String, String]): String =
        m.map { case (k, v) => s""""$k":"$v"""" }.mkString("{", ",", "}")

    def fromJson(s: String): Map[String, String] =
        s.stripPrefix("{").stripSuffix("}")
         .split(",")
         .map(_.split(":").map(_.trim.stripPrefix("\"").stripSuffix("\"")))
         .collect { case Array(k, v) => k -> v }
         .toMap

    def send(out: DataOutputStream, json: String): Unit =
        val b = json.getBytes("UTF-8")
        out.writeInt(b.length); out.write(b); out.flush()

    def recv(in: DataInputStream): String =
        val b = Array.ofDim[Byte](in.readInt())
        in.readFully(b)
        new String(b, "UTF-8")

    val server = ServerSocket(9021)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val msg    = fromJson(recv(DataInputStream(client.getInputStream)))
            println(s"action=${msg("action")} user=${msg("user")}")
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9021)
    send(DataOutputStream(socket.getOutputStream),
         toJson(Map("action" -> "login", "user" -> "alice")))
    socket.close()
    serverThread.join()
end main
```

JSON over TCP combines human-readable data with a binary length prefix so  
the receiver knows exactly how many bytes to read before parsing. The  
example avoids external library dependencies by using small manual helpers  
suitable for simple maps. In production Scala code, a library such as circe  
or upickle would handle the encoding, and the same length-prefix transport  
layer would carry its bytes unchanged. The separation between transport  
(length prefix) and encoding (JSON) is a clean architectural boundary.  

## Binary messages over TCP

Exchange typed binary fields using `DataOutputStream` and `DataInputStream`.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{DataInputStream, DataOutputStream}

    val server = ServerSocket(9022)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val in     = DataInputStream(client.getInputStream)
            val id     = in.readInt()
            val score  = in.readDouble()
            val name   = in.readUTF()
            println(s"id=$id  score=$score  name=$name")
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9022)
    val out    = DataOutputStream(socket.getOutputStream)
    out.writeInt(42)
    out.writeDouble(98.6)
    out.writeUTF("Alice")
    out.flush()
    socket.close()
    serverThread.join()
end main
```

`DataOutputStream` writes primitive Java types as fixed-size big-endian  
binary values: `writeInt` uses 4 bytes and `writeDouble` uses 8. `writeUTF`  
is a special case that writes a 2-byte length followed by modified UTF-8  
bytes; it is only compatible with `DataInputStream.readUTF` on the other  
end. Binary encoding is more compact and faster to parse than text, making  
it suitable for high-throughput internal protocols. Both sides must agree  
on the exact field order and types, usually by sharing a protocol  
specification or a shared schema file.  

## Handling partial reads

Loop until the required number of bytes has been accumulated from the stream.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{InputStream, EOFException}

    def readFully(in: InputStream, n: Int): Array[Byte] =
        val buf = Array.ofDim[Byte](n)
        var pos = 0
        while pos < n do
            val got = in.read(buf, pos, n - pos)
            if got < 0 then throw EOFException(s"Expected $n bytes, got $pos")
            pos += got
        buf

    val server = ServerSocket(9023)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val data   = readFully(client.getInputStream, 16)
            println(s"Got 16 bytes: '${new String(data, "UTF-8")}'")
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9023)
    socket.getOutputStream.write("Exactly16Chars!!".getBytes("UTF-8"))
    socket.getOutputStream.flush()
    socket.close()
    serverThread.join()
end main
```

TCP is a stream protocol: a single `write` of N bytes may be delivered to  
the receiver as multiple `read` calls each returning fewer bytes. This is  
called a partial read and is normal behaviour, not an error. The `readFully`  
function accumulates bytes by repeatedly calling `read` with an advancing  
offset until the required count is satisfied. Ignoring partial reads and  
assuming one `read` always fills the buffer is a source of subtle,  
hard-to-reproduce bugs that only manifest under load or on slow connections.  

## Handling partial writes

Mirror the read loop pattern for writes when adapting code for NIO channels.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.OutputStream

    // Blocking OutputStream.write always completes in one call.
    // This helper mirrors the loop needed for non-blocking NIO channels.
    def writeAll(out: OutputStream, data: Array[Byte]): Unit =
        var offset = 0
        while offset < data.length do
            // For OutputStream, write always sends all bytes; offset jumps to end.
            out.write(data, offset, data.length - offset)
            offset = data.length
        out.flush()

    val server = ServerSocket(9024)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val buf    = Array.ofDim[Byte](1024)
            val n      = client.getInputStream.read(buf)
            println(s"Server received $n bytes")
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9024)
    writeAll(socket.getOutputStream, ("X" * 512).getBytes("UTF-8"))
    socket.close()
    serverThread.join()
end main
```

Blocking `OutputStream.write` always transmits all requested bytes before  
returning, so partial writes do not occur in the blocking `java.net` API.  
However, when code is later adapted to non-blocking NIO, a channel's `write`  
may return a byte count smaller than the buffer remaining. Writing a  
`writeAll` helper from the start costs nothing on blocking streams and makes  
migration to NIO straightforward. The pattern loops on `write`, advancing  
the offset by the bytes actually sent each iteration.  

## Buffering strategies

Compare wrapping a socket stream with `BufferedInputStream`/  
`BufferedOutputStream` and a typed `DataOutputStream` layer on top.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{BufferedInputStream, BufferedOutputStream,
                    DataInputStream, DataOutputStream}

    val server = ServerSocket(9025)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val in     = DataInputStream(
                BufferedInputStream(client.getInputStream, 8192))
            val count   = in.readInt()
            val results = (1 to count).map(_ => in.readDouble())
            println(s"Received ${results.length} doubles, first=${results.head}")
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9025)
    val out    = DataOutputStream(
        BufferedOutputStream(socket.getOutputStream, 8192))
    val values = Array.tabulate(100)(i => i.toDouble * 1.1)
    out.writeInt(values.length)
    values.foreach(out.writeDouble)
    out.flush()
    socket.close()
    serverThread.join()
end main
```

Wrapping the raw socket stream in `BufferedOutputStream` reduces system-call  
frequency by batching small writes into a single kernel call. The 8192-byte  
buffer matches typical OS socket-buffer granularity. The `DataOutputStream`  
layer on top handles type serialisation. On the receive side,  
`BufferedInputStream` pre-reads a larger chunk from the OS buffer and  
satisfies subsequent small reads from memory rather than from the kernel.  
The correct wrapping order is always: raw socket stream → buffer → typed I/O  
layer, and the stream must be flushed through all layers at the end.  

---

## Sending a UDP datagram

Create a `DatagramSocket` and dispatch a single UDP packet to a remote address.  

```scala
@main def main() =
    import java.net.{DatagramSocket, DatagramPacket, InetAddress}

    val socket  = DatagramSocket()
    val address = InetAddress.getByName("localhost")
    val message = "Hello UDP".getBytes("UTF-8")
    val packet  = DatagramPacket(message, message.length, address, 9100)
    socket.send(packet)
    println(s"Sent ${message.length} bytes to $address:9100")
    socket.close()
end main
```

A `DatagramSocket` for sending does not require binding to a specific local  
port; the OS assigns an ephemeral port automatically. A `DatagramPacket`  
bundles the payload bytes with the destination address and port. Calling  
`send` dispatches the datagram to the OS network stack, which transmits it  
without waiting for acknowledgement. There is no guarantee the packet will  
arrive; the receiver must handle loss explicitly if reliability is required.  

## Receiving a UDP datagram

Bind a `DatagramSocket` to a port and block on `receive` until a packet arrives.  

```scala
@main def main() =
    import java.net.{DatagramSocket, DatagramPacket, InetAddress}

    val receiver = DatagramSocket(9101)
    val senderThread = new Thread:
        override def run() =
            Thread.sleep(50)
            val s   = DatagramSocket()
            val msg = "Ping from sender".getBytes("UTF-8")
            val pkt = DatagramPacket(
                msg, msg.length, InetAddress.getLoopbackAddress, 9101)
            s.send(pkt)
            s.close()
    senderThread.start()

    val buf    = Array.ofDim[Byte](1024)
    val packet = DatagramPacket(buf, buf.length)
    receiver.receive(packet)
    val text = new String(packet.getData, 0, packet.getLength, "UTF-8")
    println(s"From ${packet.getAddress}:${packet.getPort}: $text")
    receiver.close()
    senderThread.join()
end main
```

A `DatagramSocket` bound to a specific port waits for incoming datagrams by  
calling `receive`, which blocks until a packet arrives. The method fills the  
supplied `DatagramPacket` buffer and records the sender's address and port.  
The actual byte count in the payload is `packet.getLength`, not the buffer  
length. A receiver should always allocate a buffer large enough to hold the  
largest expected datagram, up to 65507 bytes for IPv4 UDP.  

## Setting packet buffer sizes

Tune `SO_RCVBUF` and `SO_SNDBUF` to improve throughput for large UDP workloads.  

```scala
@main def main() =
    import java.net.DatagramSocket

    val socket = DatagramSocket()
    println(s"Default recv buf: ${socket.getReceiveBufferSize} bytes")
    println(s"Default send buf: ${socket.getSendBufferSize} bytes")
    socket.setReceiveBufferSize(65536)
    socket.setSendBufferSize(65536)
    println(s"New recv buf: ${socket.getReceiveBufferSize} bytes")
    println(s"New send buf: ${socket.getSendBufferSize} bytes")
    socket.close()
end main
```

The OS kernel maintains per-socket send and receive buffers. For UDP, if the  
receive buffer is too small, the kernel silently drops incoming datagrams  
when the buffer fills faster than the application reads them.  
`setReceiveBufferSize` requests a minimum buffer size from the kernel; the  
actual size may be larger due to OS rounding or system limits. Increasing  
the send buffer allows the application to submit more data before the kernel  
blocks the `send` call. These settings matter most for high-throughput  
applications such as video streaming or telemetry collection.  

## Handling packet loss

Set a socket timeout and retry the `receive` call when a UDP reply does not arrive.  

```scala
@main def main() =
    import java.net.{DatagramSocket, DatagramPacket, InetAddress,
                     SocketTimeoutException}

    val socket = DatagramSocket()
    socket.setSoTimeout(500)

    val addr     = InetAddress.getLoopbackAddress
    val question = "ARE_YOU_THERE?".getBytes("UTF-8")
    val pkt      = DatagramPacket(question, question.length, addr, 9102)
    val buf      = Array.ofDim[Byte](256)
    val reply    = DatagramPacket(buf, buf.length)

    var attempts = 3
    var received = false
    while attempts > 0 && !received do
        socket.send(pkt)
        try
            socket.receive(reply)
            println(s"Reply: ${new String(reply.getData, 0, reply.getLength, "UTF-8")}")
            received = true
        catch
            case _: SocketTimeoutException =>
                attempts -= 1
                println(s"No reply, retries left: $attempts")
    if !received then println("Gave up after all attempts")
    socket.close()
end main
```

Because UDP provides no delivery guarantee, a client must implement its own  
reliability mechanism when an answer is expected. The standard approach is  
to set a receive timeout with `setSoTimeout`, send the request, and catch  
`SocketTimeoutException` if no reply arrives within the window. The example  
retries up to three times before giving up. More sophisticated  
implementations use exponential backoff between retries to avoid flooding  
the network. This is exactly the mechanism used by DNS resolvers when  
querying recursive name servers.  

## UDP echo server

Receive a datagram and send its bytes back to the originating address and port.  

```scala
@main def main() =
    import java.net.{DatagramSocket, DatagramPacket}

    val server = DatagramSocket(9103)
    val serverThread = new Thread:
        override def run() =
            val buf = Array.ofDim[Byte](1024)
            val pkt = DatagramPacket(buf, buf.length)
            server.receive(pkt)
            val echo = DatagramPacket(
                pkt.getData, pkt.getLength, pkt.getAddress, pkt.getPort)
            server.send(echo)
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val client = DatagramSocket()
    client.setSoTimeout(2000)
    val msg = "Echo me!".getBytes("UTF-8")
    client.send(DatagramPacket(msg, msg.length,
        java.net.InetAddress.getLoopbackAddress, 9103))
    val rbuf  = Array.ofDim[Byte](1024)
    val reply = DatagramPacket(rbuf, rbuf.length)
    client.receive(reply)
    println(s"Echo: ${new String(reply.getData, 0, reply.getLength, "UTF-8")}")
    client.close()
    serverThread.join()
end main
```

The UDP echo server demonstrates the fundamental receive-and-send pattern.  
After `receive` fills the packet, the server constructs a new  
`DatagramPacket` using the same data bytes but targeting the sender's  
address and port from `pkt.getAddress` and `pkt.getPort`. The server  
maintains no connection state; each exchange is entirely independent. This  
stateless quality makes UDP servers easy to scale horizontally but requires  
the application layer to manage sessions when needed.  

## UDP echo client

Send a UDP message and wait for the echo reply from a running server.  

```scala
@main def main() =
    import java.net.{DatagramSocket, DatagramPacket, InetAddress,
                     SocketTimeoutException}

    val port   = 9104
    val server = DatagramSocket(port)
    val serverThread = new Thread:
        override def run() =
            val buf = Array.ofDim[Byte](512)
            val pkt = DatagramPacket(buf, buf.length)
            server.receive(pkt)
            server.send(DatagramPacket(
                pkt.getData, pkt.getLength, pkt.getAddress, pkt.getPort))
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val client = DatagramSocket()
    client.setSoTimeout(3000)
    val host = InetAddress.getLoopbackAddress
    val data = "Hello UDP client".getBytes("UTF-8")
    client.send(DatagramPacket(data, data.length, host, port))

    val rbuf  = Array.ofDim[Byte](512)
    val reply = DatagramPacket(rbuf, rbuf.length)
    try
        client.receive(reply)
        println(s"Reply: ${new String(reply.getData, 0, reply.getLength, "UTF-8")}")
    catch
        case _: SocketTimeoutException => println("No reply received")
    client.close()
    serverThread.join()
end main
```

The client does not need to bind to a specific port before sending; the OS  
picks an ephemeral port and the server echoes back to it. Each call to  
`receive` after a send must have a timeout in case the server has already  
closed or the packet was dropped, which the `SocketTimeoutException` catch  
handles gracefully. Notice that the client and server exchange a complete  
request-reply pair without any prior handshake, which is characteristic of  
UDP communication.  

## Broadcast UDP messages

Enable `SO_BROADCAST` and send a datagram to the subnet broadcast address.  

```scala
@main def main() =
    import java.net.{DatagramSocket, DatagramPacket, InetAddress,
                     SocketTimeoutException}

    val port = 9105

    val receiver = DatagramSocket(port)
    val receiverThread = new Thread:
        override def run() =
            receiver.setBroadcast(true)
            receiver.setSoTimeout(2000)
            val buf = Array.ofDim[Byte](256)
            val pkt = DatagramPacket(buf, buf.length)
            try
                receiver.receive(pkt)
                println(s"Receiver got: ${new String(pkt.getData, 0, pkt.getLength, "UTF-8")}")
            catch
                case _: SocketTimeoutException => println("Receiver timed out")
            finally
                receiver.close()
    receiverThread.start()
    Thread.sleep(50)

    val sender = DatagramSocket()
    sender.setBroadcast(true)
    val msg   = "DISCOVER".getBytes("UTF-8")
    val bcast = InetAddress.getByName("255.255.255.255")
    sender.send(DatagramPacket(msg, msg.length, bcast, port))
    println("Broadcast sent")
    sender.close()
    receiverThread.join()
end main
```

UDP broadcast sends one datagram that the OS delivers to every host on the  
local subnet. Both sender and receiver must enable `SO_BROADCAST` by calling  
`setBroadcast(true)`. The address `255.255.255.255` is the limited broadcast  
address, restricted to the local network and never forwarded by routers.  
Broadcast is useful for service-discovery protocols such as DHCP and mDNS.  
Many operating systems allow unprivileged code to send broadcasts, but  
firewalls and managed switches often block them.  

## Multicast UDP messages

Join a multicast group with `MulticastSocket` to receive packets sent to a  
group address.  

```scala
@main def main() =
    import java.net.{MulticastSocket, DatagramPacket, InetAddress,
                     InetSocketAddress, NetworkInterface, SocketTimeoutException}

    val GROUP = InetAddress.getByName("239.1.2.3")
    val PORT  = 9106

    val receiver = MulticastSocket(PORT)
    val receiverThread = new Thread:
        override def run() =
            receiver.setSoTimeout(2000)
            val group = InetSocketAddress(GROUP, PORT)
            val iface = NetworkInterface.getByInetAddress(
                InetAddress.getLoopbackAddress)
            try
                receiver.joinGroup(group, iface)
                val buf = Array.ofDim[Byte](256)
                val pkt = DatagramPacket(buf, buf.length)
                receiver.receive(pkt)
                println(s"Multicast: ${new String(pkt.getData, 0, pkt.getLength, "UTF-8")}")
                receiver.leaveGroup(group, iface)
            catch
                case _: SocketTimeoutException => println("Multicast receive timed out")
                case e: Exception => println(s"Receiver: ${e.getMessage}")
            finally
                receiver.close()
    receiverThread.start()
    Thread.sleep(100)

    val sender = MulticastSocket()
    val msg    = "MULTICAST_HELLO".getBytes("UTF-8")
    try sender.send(DatagramPacket(msg, msg.length, GROUP, PORT))
    catch case e: Exception => println(s"Send: ${e.getMessage}")
    finally sender.close()
    receiverThread.join()
end main
```

Multicast allows a single sender to reach multiple receivers that have  
joined a group without sending individual copies to each. IPv4 multicast  
addresses occupy the range 224.0.0.0–239.255.255.255. The receiver calls  
`joinGroup` on a specific network interface to register interest; the kernel  
then delivers matching packets. Multicast is widely used for streaming,  
cluster heartbeat protocols, and routing protocols such as OSPF. Sending  
multicast requires no special privileges; receiving requires joining the  
group, which may involve IGMP traffic on the local network.  

## Setting TTL for UDP packets

Use `setTimeToLive` on a `MulticastSocket` to control how many router hops  
a packet may traverse.  

```scala
@main def main() =
    import java.net.{MulticastSocket, DatagramPacket, InetAddress}

    val socket = MulticastSocket()
    println(s"Default TTL: ${socket.getTimeToLive}")
    socket.setTimeToLive(4)
    println(s"New TTL:     ${socket.getTimeToLive}")

    val target = InetAddress.getByName("239.1.2.4")
    val msg    = "TTL-demo".getBytes("UTF-8")
    val pkt    = DatagramPacket(msg, msg.length, target, 9107)
    try socket.send(pkt)
    catch case e: Exception => println(s"Send: ${e.getMessage}")
    finally socket.close()
end main
```

The TTL (time to live) field in an IP packet decrements by one at each  
router hop. When it reaches zero, the router discards the packet and may  
send an ICMP Time Exceeded message back to the sender. `MulticastSocket`  
exposes TTL control via `setTimeToLive`, which accepts values from 0 to 255.  
A TTL of 1 (the default for multicast) confines packets to the local subnet.  
Increasing the TTL allows packets to cross routers into wider network  
segments. TTL exhaustion is the mechanism that traceroute uses to map network  
paths.  

---

## DNS lookup using InetAddress

Translate a hostname to its IP address using `InetAddress.getByName`.  

```scala
@main def main() =
    import java.net.InetAddress

    val addr = InetAddress.getByName("example.com")
    println(s"Host:        ${addr.getHostName}")
    println(s"IP:          ${addr.getHostAddress}")
    println(s"Loopback:    ${addr.isLoopbackAddress}")
    println(s"Site-local:  ${addr.isSiteLocalAddress}")
    println(s"Bytes:       ${addr.getAddress.map(b => f"${b & 0xFF}%02X").mkString(" ")}")
end main
```

`InetAddress.getByName` performs a blocking DNS lookup using the system  
resolver configured in the operating system. It returns the first IP address  
the resolver provides, which may be IPv4 or IPv6 depending on the DNS records  
and the host's network configuration. `getHostAddress` returns the  
dotted-decimal IP string without performing a reverse lookup. `getAddress`  
returns the raw bytes, which are useful when constructing low-level network  
structures.  

## Reverse DNS lookup

Look up the hostname for a known IP address using `InetAddress.getByAddress`.  

```scala
@main def main() =
    import java.net.InetAddress

    // Construct InetAddress from raw bytes to skip a forward DNS lookup
    val ipBytes = Array[Byte](93.toByte, 184.toByte, 216.toByte, 34.toByte)
    val addr    = InetAddress.getByAddress(ipBytes)
    println(s"Address:  ${addr.getHostAddress}")
    println(s"Hostname: ${addr.getCanonicalHostName}")
end main
```

A reverse DNS lookup resolves an IP address back to a hostname by querying  
the `in-addr.arpa` DNS zone. `InetAddress.getByAddress` creates an  
`InetAddress` from a raw byte array without triggering a forward DNS query.  
Calling `getCanonicalHostName` then performs the reverse lookup. If no PTR  
record exists, `getCanonicalHostName` returns the dotted-decimal IP string  
instead of a hostname. Reverse lookups are used in server logs, spam filters,  
and diagnostics to associate connections with domain names.  

## Resolving multiple addresses

Retrieve all IP addresses for a hostname with `InetAddress.getAllByName`.  

```scala
@main def main() =
    import java.net.InetAddress

    val addresses = InetAddress.getAllByName("google.com")
    println(s"${addresses.length} address(es) for google.com:")
    for addr <- addresses do
        val kind = if addr.isInstanceOf[java.net.Inet6Address] then "IPv6" else "IPv4"
        println(s"  [$kind] ${addr.getHostAddress}")
end main
```

Many services publish multiple A or AAAA records for load balancing and  
fault tolerance. `getAllByName` returns all addresses the resolver found,  
not just the first one. A robust client can try each address in turn until  
a connection succeeds. IPv4 addresses arrive as 4-byte arrays and IPv6 as  
16-byte arrays; both are returned as `InetAddress` objects, and checking  
`instanceof Inet6Address` distinguishes them when protocol-specific handling  
is needed. The JVM's DNS cache retains results for a configurable period  
controlled by the `networkaddress.cache.ttl` security property.  

## Handling DNS timeouts

Catch `UnknownHostException` and apply a deadline to bound DNS resolution time.  

```scala
@main def main() =
    import java.net.{InetAddress, UnknownHostException}
    import java.util.concurrent.{Callable, Executors, TimeUnit, TimeoutException}

    def resolveWithTimeout(host: String, ms: Long): Option[InetAddress] =
        val exec   = Executors.newSingleThreadExecutor()
        val future = exec.submit((() => InetAddress.getByName(host)): Callable[InetAddress])
        exec.shutdown()
        try Some(future.get(ms, TimeUnit.MILLISECONDS))
        catch
            case _: TimeoutException =>
                future.cancel(true)
                println(s"DNS lookup for '$host' timed out after ${ms}ms")
                None
            case _: java.util.concurrent.ExecutionException =>
                println(s"Could not resolve '$host'")
                None

    resolveWithTimeout("example.com", 3000).foreach(a => println(s"Resolved: $a"))
    resolveWithTimeout("does.not.exist.invalid", 2000)
end main
```

Java's `InetAddress.getByName` uses the JVM's built-in DNS implementation  
and does not accept a timeout parameter directly. A standard workaround  
submits the resolution to a single-threaded executor and calls `Future.get`  
with a timeout. If the lookup exceeds the deadline, `TimeoutException` is  
caught and the future is cancelled. The `ExecutionException` wrapper catches  
`UnknownHostException` that the callable throws. This executor pattern is  
the most portable solution across JVM versions and operating systems.  

## Manual DNS query (raw UDP packet)

Build a minimal DNS question packet and send it directly to a public resolver.  

```scala
@main def main() =
    import java.net.{DatagramSocket, DatagramPacket, InetAddress}
    import java.io.{ByteArrayOutputStream, DataOutputStream}

    def buildQuery(txId: Short, name: String): Array[Byte] =
        val payload = ByteArrayOutputStream()
        val out     = DataOutputStream(payload)
        out.writeShort(txId)     // Transaction ID
        out.writeShort(0x0100)   // Flags: standard query, recursion desired
        out.writeShort(1)        // QDCOUNT = 1
        out.writeShort(0)        // ANCOUNT = 0
        out.writeShort(0)        // NSCOUNT = 0
        out.writeShort(0)        // ARCOUNT = 0
        for label <- name.split("\\.") do
            out.writeByte(label.length)
            out.write(label.getBytes("UTF-8"))
        out.writeByte(0)         // Root label terminator
        out.writeShort(1)        // QTYPE = A
        out.writeShort(1)        // QCLASS = IN
        payload.toByteArray

    val socket   = DatagramSocket()
    socket.setSoTimeout(3000)
    val query    = buildQuery(0x1234.toShort, "example.com")
    val resolver = InetAddress.getByName("8.8.8.8")
    socket.send(DatagramPacket(query, query.length, resolver, 53))
    val buf      = Array.ofDim[Byte](512)
    val response = DatagramPacket(buf, buf.length)
    try
        socket.receive(response)
        println(s"Received ${response.getLength} bytes from ${response.getAddress}")
    catch
        case e: Exception => println(s"Error: ${e.getMessage}")
    finally socket.close()
end main
```

A DNS message begins with a 12-byte header followed by question records and  
optional answer records. The header contains a 16-bit transaction ID, flags,  
and four counters. A question record encodes the hostname as length-prefixed  
labels (for example, `example.com` becomes `\x07example\x03com\x00`).  
Sending this packet to port 53 of a recursive resolver (8.8.8.8 is Google's  
public DNS) triggers a lookup and returns a binary response. This raw  
approach exposes the wire format that higher-level APIs hide.  

## Parsing DNS response fields

Extract the transaction ID, flags, and answer count from a raw DNS response.  

```scala
@main def main() =
    import java.net.{DatagramSocket, DatagramPacket, InetAddress}
    import java.io.{ByteArrayInputStream, ByteArrayOutputStream,
                    DataInputStream, DataOutputStream}

    def buildQuery(txId: Short, name: String): Array[Byte] =
        val out = ByteArrayOutputStream()
        val d   = DataOutputStream(out)
        d.writeShort(txId); d.writeShort(0x0100)
        d.writeShort(1); d.writeShort(0); d.writeShort(0); d.writeShort(0)
        for label <- name.split("\\.") do
            d.writeByte(label.length); d.write(label.getBytes("UTF-8"))
        d.writeByte(0); d.writeShort(1); d.writeShort(1)
        out.toByteArray

    val socket = DatagramSocket()
    socket.setSoTimeout(3000)
    val query  = buildQuery(0xABCD.toShort, "example.com")
    socket.send(DatagramPacket(query, query.length,
        InetAddress.getByName("8.8.8.8"), 53))
    val buf = Array.ofDim[Byte](512)
    val pkt = DatagramPacket(buf, buf.length)
    try
        socket.receive(pkt)
        val in      = DataInputStream(ByteArrayInputStream(pkt.getData, 0, pkt.getLength))
        val txId    = in.readShort()
        val flags   = in.readShort()
        val qdcount = in.readShort()
        val ancount = in.readShort()
        println(f"TX ID:    0x${txId & 0xFFFF}%04X")
        println(f"Flags:    0x${flags & 0xFFFF}%04X")
        println(s"QD count: $qdcount")
        println(s"AN count: $ancount")
    catch case e: Exception => println(s"Error: ${e.getMessage}")
    finally socket.close()
end main
```

The first two bytes of a DNS response echo the transaction ID from the  
query, allowing the client to match responses to outstanding requests.  
Bytes 2–3 are flag bits: bit 15 is set (QR=1) to mark a response, bit 7  
is the recursion-available flag, and bits 0–3 carry the RCODE where zero  
means no error. The ANCOUNT field at bytes 6–7 tells the parser how many  
answer resource records follow the question section. Parsing a complete DNS  
response involves skipping the question section and decoding each answer  
record's type, class, TTL, and RDATA fields.  

---

## Using InetAddress.isReachable

Call `isReachable` as a portable, privilege-aware connectivity probe.  

```scala
@main def main() =
    import java.net.InetAddress

    val hosts = List("localhost", "8.8.8.8", "192.0.2.1")   // last is TEST-NET
    for host <- hosts do
        try
            val addr      = InetAddress.getByName(host)
            val timeout   = 2000
            val reachable = addr.isReachable(timeout)
            val status    = if reachable then "reachable" else "unreachable"
            println(s"$host (${addr.getHostAddress}): $status")
        catch
            case e: Exception => println(s"$host: error – ${e.getMessage}")
end main
```

`InetAddress.isReachable` tries to determine whether a host is reachable  
within the given timeout in milliseconds. On Unix systems with sufficient  
privileges, it sends an ICMP Echo Request (ping). Without privileges, it  
falls back to attempting a TCP connection on port 7 (the echo service). The  
result is a best-effort check; a `false` return might indicate a firewall  
blocking ICMP rather than an offline host. Addresses in the 192.0.2.0/24  
range (TEST-NET-1) are reserved for documentation and are never routed on  
the public internet.  

## ICMP echo request and reply

Explain why direct ICMP sockets require privileges and demonstrate the  
`isReachable` portable alternative.  

```scala
@main def main() =
    import java.net.InetAddress

    // Java provides no raw ICMP socket API without JNI or elevated privileges.
    // isReachable uses ICMP Echo when run as root, otherwise TCP port 7.
    val loopback = InetAddress.getLoopbackAddress
    val start    = System.currentTimeMillis()
    val ok       = loopback.isReachable(1000)
    val rtt      = System.currentTimeMillis() - start
    println(s"Check on ${loopback.getHostAddress}: ${if ok then "up" else "down"}")
    println(s"Round-trip time: ${rtt}ms")
    println()
    println("Note: true ICMP requires raw sockets (root or CAP_NET_RAW).")
    println("isReachable abstracts this behind an OS privilege check.")
end main
```

The ICMP protocol operates at the network layer, below TCP and UDP. An ICMP  
Echo Request (type 8) sent to a host elicits an Echo Reply (type 0) when  
the host is alive and not filtering such packets. Java cannot open raw  
sockets without native code or elevated OS privileges because the JVM runs  
in user space. The `isReachable` abstraction provides a useful  
approximation. Understanding ICMP is essential for interpreting traceroute  
output and for reasoning about network-layer diagnostic tools.  

## TTL and hop limits

Demonstrate how the IP TTL field controls the maximum number of routers a  
packet may traverse.  

```scala
@main def main() =
    import java.net.{MulticastSocket, DatagramPacket, InetAddress}

    // Each router decrements TTL by one and discards the packet when it
    // reaches zero, optionally sending ICMP Time Exceeded back to the sender.
    println("Sending probes with decreasing TTL values:")
    for ttl <- List(64, 32, 16, 8, 4, 1) do
        val ms  = MulticastSocket()
        ms.setTimeToLive(ttl)
        val msg = s"TTL=$ttl".getBytes("UTF-8")
        val pkt = DatagramPacket(msg, msg.length,
            InetAddress.getByName("8.8.8.8"), 53)
        try
            ms.send(pkt)
            println(s"  Sent with TTL $ttl")
        catch
            case e: Exception => println(s"  TTL $ttl: ${e.getMessage}")
        finally ms.close()
end main
```

The IP TTL field is an integer set by the sender and decremented by each  
router. When it reaches zero, the router discards the packet and, if the  
protocol permits, sends an ICMP Time Exceeded message back to the sender.  
This mechanism prevents packets from looping indefinitely in case of routing  
loops. Traceroute exploits TTL exhaustion deliberately: it sends probes with  
TTL=1, then TTL=2, and so on, collecting ICMP Time Exceeded replies from  
each successive router to map the path to the target host.  

## Conceptual traceroute logic

Illustrate how traceroute works by sending probes with incrementing TTL and  
collecting ICMP replies.  

```scala
@main def main() =
    import java.net.InetAddress

    // Java cannot receive ICMP Time Exceeded replies without raw sockets.
    // This example approximates the logic using isReachable as the probe.
    val target  = "8.8.8.8"
    val maxHops = 10
    println(s"Conceptual traceroute to $target (max $maxHops hops):")

    var hop   = 1
    var found = false
    while hop <= maxHops && !found do
        // A real implementation sends UDP/ICMP with TTL=hop and awaits
        // ICMP Time Exceeded from the router at that hop distance.
        val addr = InetAddress.getByName(target)
        if addr.isReachable(1000) then
            println(s"  Hop $hop: $target (${addr.getHostAddress}) — reached")
            found = true
        else
            println(s"  Hop $hop: no ICMP reply (may be filtered)")
        hop += 1
    if !found then println("Max hops exceeded without reaching target")
end main
```

A real traceroute sends UDP or ICMP Echo packets with TTL=1 to the target.  
The first router decrements TTL to zero and returns an ICMP Time Exceeded  
response from its own IP address. The tool records that IP as hop 1, then  
retries with TTL=2, and so on until the target itself responds with an ICMP  
Echo Reply or a port-unreachable message. Java's standard library cannot  
receive ICMP Time Exceeded messages without raw-socket privileges. A  
complete implementation requires JNI, a native helper process, or a  
third-party library such as RockSaw.  

## Reading local routing information

Enumerate local network interfaces and their bound addresses using  
`NetworkInterface`.  

```scala
@main def main() =
    import java.net.NetworkInterface
    import scala.jdk.CollectionConverters.*

    val interfaces = NetworkInterface.getNetworkInterfaces.asScala.toList
    for iface <- interfaces do
        val addrs   = iface.getInetAddresses.asScala.toList
        val addrStr =
            if addrs.isEmpty then "no addresses"
            else addrs.map(_.getHostAddress).mkString(", ")
        println(s"${iface.getName} (${iface.getDisplayName}): $addrStr")
end main
```

`NetworkInterface.getNetworkInterfaces` returns all interfaces present on  
the machine, including loopback, Ethernet, Wi-Fi, and virtual interfaces.  
Each interface exposes an enumeration of its `InetAddress` bindings.  
`scala.jdk.CollectionConverters.*` bridges Java `Enumeration` to a Scala  
`Iterator` for idiomatic traversal. This output is the same information  
shown by `ip addr` on Linux or `ifconfig` on macOS, without spawning an  
external process.  

## Determining default gateway

Identify the default gateway by running the appropriate system routing command.  

```scala
@main def main() =
    import scala.jdk.CollectionConverters.*

    // Java has no portable API for reading the kernel routing table.
    // The most reliable approach is to invoke the platform-specific command.
    val os = System.getProperty("os.name").toLowerCase
    val command =
        if os.contains("win") then
            List("cmd", "/c", "route", "print", "0.0.0.0")
        else if os.contains("mac") then
            List("netstat", "-rn")
        else
            List("ip", "route", "show", "default")

    try
        val proc   = ProcessBuilder(command.asJava)
            .redirectErrorStream(true)
            .start()
        val output = scala.io.Source.fromInputStream(proc.getInputStream).mkString
        proc.waitFor()
        output.linesIterator.take(10).foreach(println)
    catch
        case e: Exception => println(s"Could not read routes: ${e.getMessage}")
end main
```

Java and Scala have no portable standard-library API for reading the kernel  
routing table. The practical workaround is to invoke the platform-appropriate  
system command. On Linux, `ip route show default` prints the default route  
and gateway address directly. On macOS, `netstat -rn` lists the full routing  
table. On Windows, `route print 0.0.0.0` shows the default route entries.  
This approach is adequate for diagnostic utilities and administration  
scripts, though not suitable for portable production libraries.  

---

## Simple port scanner

Attempt a TCP connection to each port in a range to identify open services.  

```scala
@main def main() =
    import java.net.{Socket, InetSocketAddress}

    val host    = "localhost"
    val start   = 9000
    val end     = 9010
    val timeout = 200
    println(s"Scanning $host ports $start–$end …")
    for port <- start to end do
        val socket = Socket()
        try
            socket.connect(InetSocketAddress(host, port), timeout)
            println(s"  $port/tcp  OPEN")
        catch
            case _: Exception =>   // closed or filtered
        finally
            socket.close()
end main
```

A port scanner works by attempting TCP connections with a short timeout. If  
the handshake succeeds, the port is open and a service is listening. If the  
connection is refused immediately (TCP RST), the port is closed. If the  
timeout expires with no reply, the port is filtered by a firewall. This  
blocking implementation scans ports sequentially, which is slow for large  
ranges; a production scanner would use threads or NIO for concurrent probing.  
Always ensure you have authorisation before scanning hosts you do not own.  

## Banner grabber

Connect to a port and read the first line the service sends to identify itself.  

```scala
@main def main() =
    import java.net.{Socket, InetSocketAddress}
    import java.io.{BufferedReader, InputStreamReader}

    val targets = List(
        ("smtp.gmail.com", 587),
        ("localhost",      22)
    )
    for (host, port) <- targets do
        val socket = Socket()
        try
            socket.connect(InetSocketAddress(host, port), 3000)
            socket.setSoTimeout(3000)
            val reader = BufferedReader(
                InputStreamReader(socket.getInputStream, "UTF-8"))
            val banner = reader.readLine()
            println(s"$host:$port → $banner")
        catch
            case e: Exception =>
                println(s"$host:$port → ${e.getClass.getSimpleName}: ${e.getMessage}")
        finally
            socket.close()
end main
```

Many network services send an identification banner immediately upon  
connection before any client request. SMTP servers send `220 smtp.example.com  
ESMTP`, SSH servers send their version string, and HTTP/1.x proxies may send  
a greeting. Reading the banner requires only a connection and a single  
`readLine` call. Banner grabbing is used in security audits to identify  
software versions and in monitoring scripts to verify that expected services  
are running. A three-second timeout on both connect and read prevents the  
scanner from hanging on non-responsive ports.  

## TCP client with reconnect logic

Retry the TCP connection with a delay when the server is temporarily unavailable.  

```scala
@main def main() =
    import java.net.{Socket, InetSocketAddress, ConnectException}

    def connectWithRetry(host: String, port: Int, maxAttempts: Int): Option[Socket] =
        var attempt = 0
        var result: Option[Socket] = None
        while attempt < maxAttempts && result.isEmpty do
            attempt += 1
            try
                val s = Socket()
                s.connect(InetSocketAddress(host, port), 2000)
                println(s"Connected on attempt $attempt")
                result = Some(s)
            catch
                case _: ConnectException =>
                    println(s"Attempt $attempt failed, retrying in 500ms …")
                    Thread.sleep(500)
                case e: Exception =>
                    println(s"Unexpected: ${e.getMessage}")
                    attempt = maxAttempts
        result

    connectWithRetry("localhost", 19999, 3) match
        case Some(s) =>
            println(s"Connected: ${s.getRemoteSocketAddress}")
            s.close()
        case None =>
            println("Could not connect after all attempts")
end main
```

Network services are often temporarily unavailable due to restarts, rolling  
deployments, or transient failures. A robust client retries connections  
rather than failing immediately. The `connectWithRetry` function attempts  
the connection up to `maxAttempts` times, sleeping 500 ms between failures.  
Only `ConnectException` (connection refused) triggers a retry; other  
exceptions such as `SocketTimeoutException` or `UnknownHostException` may  
indicate permanent problems and warrant different handling. In production,  
exponential backoff increases the sleep duration on each successive failure.  

## UDP client with retry logic

Resend a UDP request if no reply arrives within the timeout window.  

```scala
@main def main() =
    import java.net.{DatagramSocket, DatagramPacket, InetAddress,
                     SocketTimeoutException}

    val port   = 9200
    val server = DatagramSocket(port)
    val serverThread = new Thread:
        override def run() =
            val buf = Array.ofDim[Byte](256)
            val pkt = DatagramPacket(buf, buf.length)
            server.receive(pkt)
            server.send(DatagramPacket(
                pkt.getData, pkt.getLength, pkt.getAddress, pkt.getPort))
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val client    = DatagramSocket()
    client.setSoTimeout(500)
    val host      = InetAddress.getLoopbackAddress
    val msg       = "HELLO".getBytes("UTF-8")
    val rbuf      = Array.ofDim[Byte](256)
    var tries     = 0
    var done      = false
    while tries < 3 && !done do
        tries += 1
        client.send(DatagramPacket(msg, msg.length, host, port))
        try
            val reply = DatagramPacket(rbuf, rbuf.length)
            client.receive(reply)
            println(s"Reply (try $tries): ${new String(reply.getData, 0, reply.getLength, "UTF-8")}")
            done = true
        catch
            case _: SocketTimeoutException => println(s"Timeout on try $tries")
    client.close()
    serverThread.join()
end main
```

A UDP request-reply pattern needs application-level retry because UDP  
provides no automatic retransmission. The client sets a receive timeout,  
sends the request, and waits for a reply. If `SocketTimeoutException` fires,  
the loop retries the send before listening again. In real DNS or TFTP  
clients, the retry count and backoff interval are configurable parameters.  
Care must be taken to distinguish a genuine timeout from a misrouted reply  
by checking the sender address of the received packet.  

## Hex dump of received data

Format received bytes as a hex dump showing both hexadecimal values and  
printable ASCII characters.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}

    def hexDump(data: Array[Byte], length: Int): String =
        val sb = StringBuilder()
        var i  = 0
        while i < length do
            val chunk = data.slice(i, math.min(i + 16, length))
            val hex   = chunk.map(b => f"${b & 0xFF}%02X").mkString(" ")
            val ascii = chunk.map(b => if b >= 32 && b < 127 then b.toChar else '.').mkString
            sb.append(f"${i}%04X  ${hex.padTo(47, ' ')}  $ascii\n")
            i += 16
        sb.toString

    val server = ServerSocket(9201)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val buf    = Array.ofDim[Byte](256)
            val n      = client.getInputStream.read(buf)
            print(hexDump(buf, n))
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9201)
    socket.getOutputStream.write("Hello\u0000\u0001\u0002 Networking!".getBytes("UTF-8"))
    socket.getOutputStream.flush()
    socket.close()
    serverThread.join()
end main
```

A hex dump displays raw bytes in two columns: hexadecimal values for exact  
byte representation, and ASCII for readability where printable characters  
appear. Each row covers 16 bytes; non-printable bytes are shown as `.` in  
the ASCII column. The offset on the left indicates the byte position within  
the buffer. This format is the standard output of tools like `xxd` and  
Wireshark's data pane, making it instantly familiar to network programmers.  
Hex dumps are invaluable when debugging binary protocols where unexpected  
bytes cause framing failures.  

## Logging network traffic

Wrap input and output streams to record every byte that passes through the  
connection.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{InputStream, OutputStream}

    class LoggingInputStream(in: InputStream, tag: String) extends InputStream:
        def read(): Int =
            val b = in.read()
            if b != -1 then print(f"[$tag RX 0x${b}%02X] ")
            b

    class LoggingOutputStream(out: OutputStream, tag: String) extends OutputStream:
        def write(b: Int): Unit =
            print(f"[$tag TX 0x${b}%02X] ")
            out.write(b)

    val server = ServerSocket(9202)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val data   = Array.ofDim[Byte](3)
            client.getInputStream.read(data)
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9202)
    val out    = LoggingOutputStream(socket.getOutputStream, "C")
    List(0x41, 0x42, 0x43).foreach(out.write)
    out.flush()
    socket.close()
    serverThread.join()
    println()
end main
```

Wrapping raw socket streams in decorator classes that log every byte is a  
non-invasive way to add traffic capture without modifying existing protocol  
code. `LoggingInputStream` and `LoggingOutputStream` extend the standard  
Java base classes and override only the single-byte methods; multi-byte  
overloads in the base class delegate to the single-byte version. In  
production, logging every byte is too verbose; a ring-buffer capture that  
records only the last N kilobytes is more practical for post-mortem  
debugging.  

## Simple chat client and server

Build a blocking bidirectional chat pair using two threads per connection.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{BufferedReader, InputStreamReader, PrintWriter}

    def chatSession(socket: Socket, name: String, messages: List[String]): Unit =
        val reader = BufferedReader(InputStreamReader(socket.getInputStream, "UTF-8"))
        val writer = PrintWriter(socket.getOutputStream, true)
        val readThread = new Thread:
            override def run() =
                var line = reader.readLine()
                while line != null do
                    println(s"[$name received] $line")
                    line = reader.readLine()
        readThread.start()
        messages.foreach(writer.println)
        socket.shutdownOutput()
        readThread.join()
        socket.close()

    val server = ServerSocket(9203)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            chatSession(client, "Server", List("Hi client!", "How are you?"))
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9203)
    chatSession(socket, "Client", List("Hello server!", "I am fine."))
    serverThread.join()
end main
```

Bidirectional communication requires the ability to send and receive  
simultaneously. `chatSession` starts a reader thread that continuously prints  
incoming messages while the caller's thread sends outgoing ones. After  
sending, `shutdownOutput` sends a TCP FIN to signal the end of outgoing data  
without closing the socket entirely; the reader thread can still receive the  
peer's remaining messages. Joining the reader thread ensures all messages  
are printed before the socket is closed and the session ends.  

## File transfer over TCP

Read a file from disk and stream its bytes over a TCP connection to a receiver.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{FileInputStream, FileOutputStream}
    import java.nio.file.Files

    val srcPath = Files.createTempFile("transfer-src", ".bin")
    Files.write(srcPath, Array.tabulate[Byte](1024)(i => (i % 256).toByte))
    val dstPath = Files.createTempFile("transfer-dst", ".bin")

    val server = ServerSocket(9204)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val out    = FileOutputStream(dstPath.toFile)
            val buf    = Array.ofDim[Byte](4096)
            val in     = client.getInputStream
            var n      = in.read(buf)
            while n != -1 do
                out.write(buf, 0, n)
                n = in.read(buf)
            out.close(); client.close(); server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9204)
    val in     = FileInputStream(srcPath.toFile)
    val out    = socket.getOutputStream
    val buf    = Array.ofDim[Byte](4096)
    var n      = in.read(buf)
    while n != -1 do
        out.write(buf, 0, n)
        n = in.read(buf)
    in.close(); socket.close()
    serverThread.join()

    println(s"Sent:     ${Files.size(srcPath)} bytes")
    println(s"Received: ${Files.size(dstPath)} bytes")
    println(s"Match:    ${java.util.Arrays.equals(
        Files.readAllBytes(srcPath), Files.readAllBytes(dstPath))}")
end main
```

File transfer over TCP is a direct application of the read-write loop: the  
sender reads a block from disk into a buffer and writes it to the socket;  
the receiver reads from the socket and writes to disk. The loop continues  
until `read` returns −1, signalling that the sender closed its output stream  
after the last byte. Using a 4096-byte buffer balances memory use and  
system-call overhead. In production, `FileChannel.transferTo` offers a more  
efficient kernel-level copy that avoids the intermediate user-space buffer.  

## Measuring round-trip time (TCP)

Record timestamps before and after a full TCP request-reply exchange to  
compute latency.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{BufferedReader, InputStreamReader, PrintWriter}

    val server = ServerSocket(9205)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val reader = BufferedReader(InputStreamReader(client.getInputStream, "UTF-8"))
            val writer = PrintWriter(client.getOutputStream, true)
            var line   = reader.readLine()
            while line != null do
                writer.println(line)
                line = reader.readLine()
            client.close(); server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9205)
    val writer = PrintWriter(socket.getOutputStream, true)
    val reader = BufferedReader(InputStreamReader(socket.getInputStream, "UTF-8"))

    var totalNs = 0L
    val runs    = 5
    for _ <- 1 to runs do
        val t0  = System.nanoTime()
        writer.println("PING")
        reader.readLine()
        val rtt = System.nanoTime() - t0
        totalNs += rtt
        println(s"RTT: ${rtt / 1_000_000}ms  (${rtt / 1_000}µs)")
    println(s"Average: ${totalNs / runs / 1_000_000}ms")
    socket.close()
    serverThread.join()
end main
```

TCP round-trip time is measured by recording a nanosecond timestamp  
immediately before sending a request and again immediately after receiving  
the reply. Running several iterations and averaging them smooths out  
scheduling jitter from the OS thread scheduler. RTT measured through  
loopback (localhost) is typically under 1 ms and reflects primarily  
thread-switching overhead. Over the network, propagation delay and kernel  
buffer latency dominate and RTT values are in the tens to hundreds of  
milliseconds range.  

## Measuring round-trip time (UDP)

Time a UDP request-reply exchange to measure datagram round-trip latency.  

```scala
@main def main() =
    import java.net.{DatagramSocket, DatagramPacket, InetAddress}

    val port   = 9206
    val server = DatagramSocket(port)
    val serverThread = new Thread:
        override def run() =
            val buf = Array.ofDim[Byte](64)
            for _ <- 1 to 5 do
                val pkt = DatagramPacket(buf, buf.length)
                server.receive(pkt)
                server.send(DatagramPacket(
                    pkt.getData, pkt.getLength, pkt.getAddress, pkt.getPort))
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val client = DatagramSocket()
    client.setSoTimeout(2000)
    val host  = InetAddress.getLoopbackAddress
    val msg   = "PING".getBytes("UTF-8")
    val rbuf  = Array.ofDim[Byte](64)
    var total = 0L
    val runs  = 5
    for i <- 1 to runs do
        val t0  = System.nanoTime()
        client.send(DatagramPacket(msg, msg.length, host, port))
        client.receive(DatagramPacket(rbuf, rbuf.length))
        val rtt = System.nanoTime() - t0
        total += rtt
        println(s"UDP RTT $i: ${rtt / 1_000_000}ms  (${rtt / 1_000}µs)")
    println(s"Average: ${total / runs / 1_000_000}ms")
    client.close()
    serverThread.join()
end main
```

UDP RTT measurement follows the same timestamp-before-send and  
timestamp-after-receive pattern as the TCP version. Because UDP does not  
retransmit, a lost packet would cause the `receive` call to time out and  
raise `SocketTimeoutException`, invalidating that measurement. In practice,  
loopback UDP RTT is similar to or slightly lower than TCP RTT because there  
is no connection-setup overhead and no ACK processing. Over the public  
internet, UDP RTT can be lower than TCP RTT when TCP's flow control or  
congestion window is limiting throughput.  

---

## Handling SocketTimeoutException

Respond gracefully when a read operation exceeds its configured timeout.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket, SocketTimeoutException}
    import java.io.{BufferedReader, InputStreamReader}

    // Server that accepts but deliberately delays before sending data
    val server = ServerSocket(9207)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            Thread.sleep(5000)   // Deliberately unresponsive
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9207)
    socket.setSoTimeout(500)
    val reader = BufferedReader(InputStreamReader(socket.getInputStream, "UTF-8"))
    try
        val line = reader.readLine()
        println(s"Received: $line")
    catch
        case _: SocketTimeoutException =>
            println("Read timed out — server is not responding")
            println("Possible actions: retry, reconnect, or abort")
    finally
        socket.close()
        server.close()
        serverThread.interrupt()
        serverThread.join(1000)
end main
```

`SocketTimeoutException` is a subclass of `java.io.InterruptedIOException`  
and is thrown when a blocking `read` or `accept` call exceeds the timeout  
set by `setSoTimeout`. Unlike most exceptions, it does not leave the socket  
in a broken state: the connection remains open and the caller may retry the  
read, apply fallback logic, or close the socket. Logging the event with a  
human-readable message helps operators diagnose slow services. Production  
code should distinguish a read timeout (connection alive but peer idle)  
from a connect timeout (peer unreachable).  

## Handling ConnectException

Catch `ConnectException` to identify refused or unreachable connection  
attempts.  

```scala
@main def main() =
    import java.net.{Socket, InetSocketAddress, ConnectException,
                     SocketTimeoutException}

    val targets = List(
        ("localhost", 19998),   // nothing listening
        ("192.0.2.1", 80),      // TEST-NET, unreachable
        ("localhost", 19997)
    )
    for (host, port) <- targets do
        val socket = Socket()
        try
            socket.connect(InetSocketAddress(host, port), 500)
            println(s"$host:$port — connected")
            socket.close()
        catch
            case _: ConnectException =>
                println(s"$host:$port — connection refused (no service listening)")
            case _: SocketTimeoutException =>
                println(s"$host:$port — timed out (host unreachable or filtered)")
            case e: Exception =>
                println(s"$host:$port — ${e.getClass.getSimpleName}: ${e.getMessage}")
        finally
            if !socket.isClosed then socket.close()
end main
```

`ConnectException` is thrown when the remote host is reachable but actively  
refuses the connection — typically because no process is listening on the  
target port. The OS sends a TCP RST in response and the JVM translates it  
to this exception. Distinguishing `ConnectException` from  
`SocketTimeoutException` is important: a refused connection means the host  
is up but the service is down or on a different port, while a timeout means  
the host is unreachable or a firewall is silently dropping packets. Both  
cases warrant different operational responses.  

## Handling EOFException

Catch `EOFException` from `DataInputStream` when the stream ends before all  
expected bytes arrive.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{DataInputStream, DataOutputStream, EOFException}

    val server = ServerSocket(9208)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val out    = DataOutputStream(client.getOutputStream)
            out.writeInt(42)     // Sends only 4 bytes then closes
            out.flush()
            // Deliberately omits writeDouble that the client expects
            client.close()
            server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9208)
    val in     = DataInputStream(socket.getInputStream)
    try
        val id    = in.readInt()
        println(s"id = $id")
        val score = in.readDouble()   // EOFException: only 4 bytes were sent
        println(s"score = $score")
    catch
        case _: EOFException =>
            println("Stream ended before all expected fields arrived")
            println("Causes: server bug, protocol mismatch, or premature close")
    finally
        socket.close()
        serverThread.join()
end main
```

`DataInputStream.readFully` and the typed read methods (`readInt`,  
`readDouble`, and so on) throw `EOFException` when the stream ends with  
fewer bytes than required. This is a specific subclass of `IOException` and  
should be caught separately for clear diagnostics. It typically indicates a  
protocol mismatch — the sender closed the connection before transmitting all  
fields of a message — or a server crash mid-response. The handler should  
log the partial state, close the socket, and if appropriate reconnect and  
request a retransmission.  

## Retry with exponential backoff

Increase the wait time between successive connection attempts to avoid  
overwhelming a recovering service.  

```scala
@main def main() =
    import java.net.{Socket, InetSocketAddress, ConnectException,
                     SocketTimeoutException}

    def connectWithBackoff(
        host: String, port: Int,
        maxAttempts: Int, baseDelayMs: Long
    ): Option[Socket] =
        var attempt = 0
        var delayMs = baseDelayMs
        var result: Option[Socket] = None
        while attempt < maxAttempts && result.isEmpty do
            attempt += 1
            try
                val s = Socket()
                s.connect(InetSocketAddress(host, port), 2000)
                println(s"Connected on attempt $attempt")
                result = Some(s)
            catch
                case _: ConnectException | _: SocketTimeoutException =>
                    println(s"Attempt $attempt failed, waiting ${delayMs}ms …")
                    if attempt < maxAttempts then Thread.sleep(delayMs)
                    delayMs = math.min(delayMs * 2, 30_000)   // cap at 30s
        result

    connectWithBackoff("localhost", 29999, 4, 100) match
        case Some(s) =>
            println(s"Success: ${s.getRemoteSocketAddress}")
            s.close()
        case None =>
            println("All attempts exhausted")
end main
```

Exponential backoff doubles the delay after each failed attempt, starting  
from a small base value and capping at a maximum to prevent excessively long  
waits. This strategy avoids the thundering herd effect: when a service  
restarts, all waiting clients do not retry simultaneously and flood it with  
connection requests. Adding a small random jitter (±10%) further spreads  
retries from multiple concurrent clients. This pattern is recommended by  
AWS, Google, and other cloud providers for building resilient service  
clients.  

## Validating incoming data

Check received messages against format, length, and value constraints before  
processing.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{BufferedReader, InputStreamReader, PrintWriter}

    def validate(line: String): Either[String, (String, Int)] =
        val parts = line.split(":", 2)
        if parts.length != 2 then Left(s"Bad format: '$line'")
        else
            val name   = parts(0).trim
            val ageStr = parts(1).trim
            if name.isEmpty then Left("Empty name")
            else if name.length > 64 then Left("Name too long")
            else
                try
                    val age = ageStr.toInt
                    if age < 0 || age > 150 then Left(s"Age out of range: $age")
                    else Right(name -> age)
                catch
                    case _: NumberFormatException => Left(s"Invalid age: '$ageStr'")

    val server = ServerSocket(9209)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val reader = BufferedReader(InputStreamReader(client.getInputStream, "UTF-8"))
            val writer = PrintWriter(client.getOutputStream, true)
            var line   = reader.readLine()
            while line != null do
                validate(line) match
                    case Right((name, age)) => writer.println(s"OK $name is $age")
                    case Left(err)          => writer.println(s"ERR $err")
                line = reader.readLine()
            client.close(); server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9209)
    val writer = PrintWriter(socket.getOutputStream, true)
    val reader = BufferedReader(InputStreamReader(socket.getInputStream, "UTF-8"))
    List("Alice:30", "Bob:", "Charlie:200", ":25", "Dave:42").foreach: msg =>
        writer.println(msg)
        println(s"$msg -> ${reader.readLine()}")
    socket.close()
    serverThread.join()
end main
```

Network input must never be trusted unconditionally. Even well-intentioned  
peers may send malformed data due to bugs, and malicious peers deliberately  
send invalid input. The `validate` function returns an `Either` so the  
server explicitly handles both success and failure without throwing for  
expected bad input. Checking length bounds prevents unbounded memory  
allocation; range checks prevent numeric overflow in downstream processing.  
Returning structured error responses (`ERR <message>`) gives clients  
actionable diagnostics rather than a silent close.  

## Defensive parsing of messages

Use `Try` and pattern matching to parse binary frames without crashing on  
malformed data.  

```scala
@main def main() =
    import java.net.{Socket, ServerSocket}
    import java.io.{DataInputStream, DataOutputStream}
    import scala.util.{Try, Success, Failure}

    case class Measurement(sensorId: Int, value: Double, unit: String)

    def parseFrame(in: DataInputStream): Try[Measurement] =
        Try:
            val id    = in.readInt()
            val value = in.readDouble()
            val unit  = in.readUTF()
            if unit.length > 8 then
                throw IllegalArgumentException(s"Unit too long: '$unit'")
            Measurement(id, value, unit)

    val server = ServerSocket(9210)
    val serverThread = new Thread:
        override def run() =
            val client = server.accept()
            val in     = DataInputStream(client.getInputStream)
            for _ <- 1 to 3 do
                parseFrame(in) match
                    case Success(m) =>
                        println(s"Sensor ${m.sensorId}: ${m.value} ${m.unit}")
                    case Failure(e) =>
                        println(s"Parse error: ${e.getMessage}")
            client.close(); server.close()
    serverThread.start()
    Thread.sleep(50)

    val socket = Socket("localhost", 9210)
    val out    = DataOutputStream(socket.getOutputStream)
    out.writeInt(1); out.writeDouble(23.5);   out.writeUTF("Celsius")
    out.writeInt(2); out.writeDouble(1013.0); out.writeUTF("hPa")
    out.writeInt(3); out.writeDouble(42.0);   out.writeUTF("Percent")
    out.flush()
    socket.close()
    serverThread.join()
end main
```

Wrapping the parsing logic in `Try { ... }` converts any exception thrown  
during reading or validation into a `Failure` value. The server can then  
pattern-match on `Success` and `Failure` and decide independently whether to  
log the error and continue or to close the connection. This is especially  
important for long-running connections that carry many messages: one corrupt  
frame should not terminate the entire session. The additional `unit.length`  
guard demonstrates application-level validation layered on top of the  
low-level binary parse.  
