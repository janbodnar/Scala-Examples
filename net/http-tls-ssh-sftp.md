# HTTP, TLS, SSH, and SFTP in Scala 3

Modern applications rarely run in isolation. They send HTTP requests to  
REST APIs, serve HTTPS traffic to browsers, automate remote servers over  
SSH, and transfer files reliably through SFTP. Mastering these protocols  
directly from Scala 3 — using the JDK's built-in `java.net.http` client,  
the `javax.net.ssl` TLS infrastructure, and the SSHJ library — gives you  
precise control without heavy frameworks.  

## HTTP versus raw TCP

TCP provides a reliable, ordered byte stream between two endpoints. It  
guarantees that every byte arrives in sequence and without corruption, but  
it says nothing about what those bytes mean. HTTP is an application-layer  
protocol built on top of that byte stream. It defines a text-based  
message format: a request line, a set of headers, a blank line, and an  
optional body going from client to server, followed by a status line,  
response headers, and a body flowing back. Without HTTP, two programs  
communicating over TCP must invent their own framing, versioning, and  
error conventions. HTTP provides all of that out of the box and is  
understood by every web server, proxy, and load balancer in existence.  

When you write raw TCP code with `java.net.Socket`, you handle framing  
manually. HTTP is the better choice for any interaction that follows the  
request-response model, especially when the server is a standard web  
service. Raw sockets are appropriate for custom binary protocols, game  
engines, financial tick feeds, or situations where every microsecond  
matters and HTTP's overhead is too large.  

## HTTP/1.1 versus HTTP/2

HTTP/1.1 is a text-based protocol that sends one request at a time per  
TCP connection. Pipelining was specified but is poorly supported in  
practice, so clients typically open several parallel connections to  
achieve concurrency. Each request carries its full set of headers in  
plain ASCII, which can amount to hundreds of bytes of repeated metadata  
on every round trip.  

HTTP/2 switches to a binary framing layer and introduces multiplexing.  
Multiple logical request/response streams share a single TCP connection  
simultaneously. HTTP/2 also compresses headers with the HPACK algorithm  
and allows the server to push resources to the client before they are  
requested. The JDK 11 `HttpClient` supports both versions and negotiates  
HTTP/2 automatically when the server advertises it via ALPN during the  
TLS handshake.  

## Request/response lifecycle

A single HTTP GET follows these steps: the client performs a DNS lookup  
to convert the hostname into an IP address, then opens a TCP connection  
to that address on port 80 (or 443 for HTTPS). If TLS is involved, the  
handshake happens next. The client then writes the HTTP request line and  
headers, sends an optional body, and waits. The server reads the request,  
executes the handler, and writes back the status line, response headers,  
and body. The connection may be kept alive for future requests or closed  
depending on the `Connection` header.  

Connection pooling, which reuses open TCP connections across multiple  
requests to the same host, dramatically reduces latency by skipping the  
TCP and TLS setup on subsequent calls. The JDK `HttpClient` pools  
connections automatically.  

## TLS handshake basics

TLS (Transport Layer Security) wraps TCP traffic in an encrypted channel.  
The handshake proceeds as follows: the client sends a `ClientHello`  
message listing supported cipher suites and TLS versions. The server  
responds with a `ServerHello`, chooses a cipher suite, and sends its  
X.509 certificate. The client verifies the certificate chain against its  
trust store. Both sides then perform a key exchange (ECDHE or RSA) to  
derive a shared session key. Once both sides confirm the handshake with  
`Finished` messages, all subsequent data flows under symmetric encryption  
using that session key.  

TLS 1.3, the current version, removes several older, weaker cipher suites  
and simplifies the handshake to one round-trip, reducing latency compared  
to TLS 1.2.  

## Certificates, trust stores, and key stores

An X.509 certificate binds a public key to an identity (such as a domain  
name). It is signed by a Certificate Authority (CA) whose own certificate  
is signed by a root CA. Browsers and JVMs ship with a list of trusted  
root CAs. When your code opens an HTTPS connection, the JVM verifies that  
the server's certificate chain leads back to one of those roots.  

A trust store holds the CA certificates you trust. On the JVM it is  
typically a JKS or PKCS12 file. A key store holds your own private key  
and certificate, which you present for mutual TLS (mTLS). You load them  
into `SSLContext` via `TrustManagerFactory` and `KeyManagerFactory`.  

## SSH versus SFTP versus SCP

SSH (Secure Shell) is a cryptographic protocol for remote login and  
command execution. Once a session is established, you can run arbitrary  
commands on the remote host. SFTP (SSH File Transfer Protocol) is a  
separate subsystem that runs inside an SSH channel and provides a  
full-featured file-system API: upload, download, list, rename, stat, and  
delete. SCP (Secure Copy Protocol) is an older, simpler utility also  
layered over SSH that copies files but offers no browsing or attribute  
queries. Prefer SFTP for new code because it is more robust, supports  
resume, and handles errors more gracefully.  

The examples in this document use the SSHJ library  
(`com.hierynomus:sshj:0.38.0`). Add the following to your `build.sbt`:  

```
libraryDependencies += "com.hierynomus" % "sshj" % "0.38.0"
```

## When to use high-level HTTP versus raw sockets

Use the JDK `HttpClient` whenever you are talking to a REST API, a  
webhook, a health-check endpoint, or any service that speaks HTTP. The  
client handles connection pooling, TLS, redirects, compression, and  
content negotiation for you.  

Use raw sockets (`java.net.Socket` or NIO channels) when you are  
implementing a custom binary protocol, when you need sub-millisecond  
latency control, or when the remote service does not speak HTTP at all.  
Raw sockets give you full control but require you to handle every detail  
yourself.  

## Common pitfalls

**Timeouts**: Always set both a connect timeout and a per-request timeout.  
A missing timeout can cause threads to block indefinitely when a server  
is slow or unreachable.  

**Redirects**: By default the JDK `HttpClient` does not follow redirects.  
Many APIs redirect HTTP to HTTPS or move endpoints. Set  
`HttpClient.Redirect.NORMAL` or `ALWAYS` explicitly.  

**Certificate issues**: Self-signed certificates fail validation unless  
you load them into a custom trust store. Never disable certificate  
validation in production code; it exposes you to man-in-the-middle  
attacks.  

**SNI**: Some servers host multiple domains on one IP and use Server Name  
Indication to choose the right certificate. The JDK client sends SNI  
automatically; raw socket code must set it manually via `SSLParameters`.  

**Character encoding**: Always specify `StandardCharsets.UTF_8` explicitly  
when converting between bytes and strings to avoid platform-dependent  
behavior.  

---

## Simple HTTP GET

Perform a basic HTTP GET request and print the response body.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI

@main def main() =
  val client = HttpClient.newHttpClient()
  val request = HttpRequest.newBuilder()
    .uri(URI.create("https://httpbin.org/get"))
    .GET()
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofString())
  println(s"Status: ${response.statusCode()}")
  println(response.body())
end main
```

`HttpClient.newHttpClient` creates a client with default settings,  
including HTTP/1.1 and HTTP/2 support and a shared connection pool. The  
request is built with the fluent `HttpRequest.newBuilder` API: you set  
the URI, choose the GET method, and call `build` to produce an immutable  
`HttpRequest` instance. `client.send` blocks the calling thread until the  
full response body has been received, then returns an `HttpResponse[String]`  
whose `body()` method gives you the decoded text.  

## HTTP GET with headers

Add custom request headers to a GET request.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI

@main def main() =
  val client = HttpClient.newHttpClient()
  val request = HttpRequest.newBuilder()
    .uri(URI.create("https://httpbin.org/headers"))
    .header("Accept", "application/json")
    .header("X-Custom-Header", "ScalaExample")
    .GET()
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofString())
  println(s"Status: ${response.statusCode()}")
  println(response.body())
end main
```

Each call to `.header(name, value)` appends one header to the request.  
You can chain as many `.header` calls as needed before calling `.build()`.  
The `Accept` header tells the server which content type the client  
prefers. The echo endpoint at `httpbin.org/headers` reflects every  
incoming header back in the response body, making it easy to verify that  
your headers were sent correctly.  

## HTTP GET with query parameters

Build a URL with encoded query parameters before sending the request.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.{URI, URLEncoder}
import java.nio.charset.StandardCharsets

@main def main() =
  val base = "https://httpbin.org/get"
  val params = Map("name" -> "Scala", "version" -> "3")
  val query = params.map { (k, v) =>
    val ek = URLEncoder.encode(k, StandardCharsets.UTF_8)
    val ev = URLEncoder.encode(v, StandardCharsets.UTF_8)
    s"$ek=$ev"
  }.mkString("&")
  val client  = HttpClient.newHttpClient()
  val request = HttpRequest.newBuilder()
    .uri(URI.create(s"$base?$query"))
    .GET()
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofString())
  println(response.body())
end main
```

`URLEncoder.encode` percent-encodes any characters that are not safe in a  
URL query string, such as spaces (encoded as `%20` or `+`) and special  
characters. Building the query string manually gives you full control  
over parameter ordering and encoding. For production code, consider a  
URI builder from a library such as Apache HttpComponents to handle edge  
cases cleanly. The `httpbin.org/get` endpoint echoes the decoded query  
parameters back in the JSON response body.  

## HTTP POST with JSON body

Send a JSON payload to a server using the POST method.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI

@main def main() =
  val json   = """{"name": "Alice", "age": 30}"""
  val client = HttpClient.newHttpClient()
  val request = HttpRequest.newBuilder()
    .uri(URI.create("https://httpbin.org/post"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString(json))
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofString())
  println(s"Status: ${response.statusCode()}")
  println(response.body())
end main
```

`HttpRequest.BodyPublishers.ofString` wraps a Scala `String` in a  
`BodyPublisher` that the HTTP client reads when sending the request body.  
The `Content-Type: application/json` header tells the server how to  
interpret the bytes. The status code `200` confirms the server accepted  
the payload. The httpbin echo service reflects the parsed JSON object  
back in its response, which is a convenient way to confirm the body was  
transmitted correctly.  

## HTTP PUT with JSON body

Update a remote resource by sending a JSON body with the PUT method.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI

@main def main() =
  val json   = """{"id": 1, "name": "Bob", "active": true}"""
  val client = HttpClient.newHttpClient()
  val request = HttpRequest.newBuilder()
    .uri(URI.create("https://httpbin.org/put"))
    .header("Content-Type", "application/json")
    .PUT(HttpRequest.BodyPublishers.ofString(json))
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofString())
  println(s"Status: ${response.statusCode()}")
  println(response.body())
end main
```

PUT is semantically different from POST: PUT is idempotent and is  
intended to fully replace the identified resource with the provided  
representation. The builder's `.PUT(publisher)` method sets the method  
to `PUT` and attaches the body publisher in one step. The response body  
from httpbin echoes the submitted JSON, confirming the request was  
received with the correct content.  

## HTTP DELETE request

Delete a remote resource with the HTTP DELETE method.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI

@main def main() =
  val client = HttpClient.newHttpClient()
  val request = HttpRequest.newBuilder()
    .uri(URI.create("https://httpbin.org/delete"))
    .DELETE()
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofString())
  println(s"Status: ${response.statusCode()}")
  println(response.body())
end main
```

The `.DELETE()` builder method produces a request with no body, as  
expected by the HTTP specification for the DELETE method. A successful  
response with status `200` or `204` indicates the server processed the  
deletion. In real APIs the URI typically includes a resource identifier,  
such as `/users/42`, to specify which record to remove.  

## Sending form data

Submit HTML form data encoded as `application/x-www-form-urlencoded`.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.{URI, URLEncoder}
import java.nio.charset.StandardCharsets

@main def main() =
  val params = Map("username" -> "alice", "password" -> "s3cret")
  val body = params.map { (k, v) =>
    val ek = URLEncoder.encode(k, StandardCharsets.UTF_8)
    val ev = URLEncoder.encode(v, StandardCharsets.UTF_8)
    s"$ek=$ev"
  }.mkString("&")
  val client = HttpClient.newHttpClient()
  val request = HttpRequest.newBuilder()
    .uri(URI.create("https://httpbin.org/post"))
    .header("Content-Type",
      "application/x-www-form-urlencoded")
    .POST(HttpRequest.BodyPublishers.ofString(body))
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofString())
  println(s"Status: ${response.statusCode()}")
  println(response.body())
end main
```

Form-encoded data is the format browsers use when submitting basic HTML  
forms. Keys and values are percent-encoded separately and joined with `&`.  
The `Content-Type` header must be set to `application/x-www-form-urlencoded`  
so the server knows how to parse the body. This encoding is simpler than  
multipart and is appropriate when you do not need to upload binary files.  

## Sending multipart data

Build a `multipart/form-data` request body manually for mixed field and  
binary uploads.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI
import java.nio.charset.StandardCharsets

@main def main() =
  val boundary = "----ScalaBoundary7A3F"
  val CRLF     = "\r\n"
  val body =
    s"--$boundary$CRLF" +
    s"Content-Disposition: form-data; name=\"field1\"$CRLF$CRLF" +
    s"hello$CRLF" +
    s"--$boundary$CRLF" +
    s"Content-Disposition: form-data; name=\"field2\"$CRLF$CRLF" +
    s"world$CRLF" +
    s"--$boundary--$CRLF"
  val client = HttpClient.newHttpClient()
  val request = HttpRequest.newBuilder()
    .uri(URI.create("https://httpbin.org/post"))
    .header("Content-Type",
      s"multipart/form-data; boundary=$boundary")
    .POST(HttpRequest.BodyPublishers.ofString(
      body, StandardCharsets.UTF_8))
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofString())
  println(s"Status: ${response.statusCode()}")
  println(response.body())
end main
```

Multipart encoding separates each part with a boundary string that must  
not appear in any part's content. Each part opens with a boundary line,  
followed by `Content-Disposition` and optional headers, a blank line, and  
the part data. A final boundary with a trailing `--` signals the end of  
the request body. The JDK HTTP client does not generate multipart bodies  
automatically, so you build the raw text yourself. For real file uploads,  
read file bytes as a `byte[]` and combine them with the text parts before  
constructing the publisher.  

## Streaming request body

Stream large request body data from an `InputStream` rather than loading  
the entire content into memory.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI
import java.io.ByteArrayInputStream

@main def main() =
  val data = ("x" * 100_000).getBytes("UTF-8")
  val client = HttpClient.newHttpClient()
  val request = HttpRequest.newBuilder()
    .uri(URI.create("https://httpbin.org/post"))
    .header("Content-Type", "text/plain")
    .POST(HttpRequest.BodyPublishers.ofInputStream(
      () => new ByteArrayInputStream(data)))
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofString())
  println(s"Status: ${response.statusCode()}")
  println(s"Sent ${data.length} bytes")
end main
```

`BodyPublishers.ofInputStream` accepts a `Supplier[InputStream]`, not a  
raw stream, so the client can retry the request if a redirect requires  
re-sending the body. The supplier is called once per attempt. Using a  
stream prevents loading the entire payload into a single `String` or  
`byte[]`, which matters for uploads of files larger than available heap.  
In production, supply a `FileInputStream` or a piped stream from a  
processing pipeline.  

## Streaming response body

Process a streaming HTTP response line by line without buffering the  
entire body.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI

@main def main() =
  val client = HttpClient.newHttpClient()
  val request = HttpRequest.newBuilder()
    .uri(URI.create("https://httpbin.org/stream/5"))
    .GET()
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofLines())
  println(s"Status: ${response.statusCode()}")
  response.body().forEach(line => println(line))
end main
```

`HttpResponse.BodyHandlers.ofLines()` returns a `Stream[String]` that  
emits lines as they arrive from the network. This approach is ideal for  
server-sent events, newline-delimited JSON (NDJSON), and any API that  
pushes incremental results over a long-lived connection. The  
`Stream` is lazy; it does not buffer the whole response before your  
handler sees the first line, which keeps memory usage constant regardless  
of response size.  

## Handling redirects

Configure the HTTP client to follow redirects automatically.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI

@main def main() =
  val client = HttpClient.newBuilder()
    .followRedirects(HttpClient.Redirect.NORMAL)
    .build()
  val request = HttpRequest.newBuilder()
    .uri(URI.create("http://httpbin.org/redirect/3"))
    .GET()
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofString())
  println(s"Final status: ${response.statusCode()}")
  println(s"Final URI:    ${response.uri()}")
end main
```

`HttpClient.Redirect.NORMAL` follows redirects for safe methods (GET,  
HEAD) but not for POST or PUT, matching the behaviour users expect from  
a browser. `Redirect.ALWAYS` follows redirects for all methods including  
POST, which may cause unintended re-submission of form data. `Redirect.NEVER`  
is the default, meaning you must handle 3xx responses yourself.  
`response.uri()` shows the final URL after all redirects have been  
resolved, which is useful for audit logging and link deduplication.  

## Handling timeouts

Set connect and per-request timeouts to avoid blocking threads  
indefinitely.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse,
  HttpTimeoutException}
import java.net.URI
import java.time.Duration

@main def main() =
  val client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(5))
    .build()
  val request = HttpRequest.newBuilder()
    .uri(URI.create("https://httpbin.org/delay/2"))
    .timeout(Duration.ofSeconds(10))
    .GET()
    .build()
  try
    val response = client.send(
      request, HttpResponse.BodyHandlers.ofString())
    println(s"Status: ${response.statusCode()}")
  catch
    case e: HttpTimeoutException =>
      println(s"Timed out: ${e.getMessage}")
    case e: Exception =>
      println(s"Error: ${e.getMessage}")
end main
```

The connect timeout governs how long the client waits for the TCP  
handshake to complete. The request timeout set on the builder governs  
how long the client waits for the complete response after the connection  
is established. If either limit is exceeded, a `HttpTimeoutException` is  
thrown. Always handle this exception explicitly in production code. The  
`/delay/2` endpoint at httpbin sleeps for two seconds before responding,  
giving you an easy way to test timeout behaviour without a real slow  
server.  

## Handling HTTP errors

Inspect the status code and react appropriately to server-side errors.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI

@main def main() =
  val client = HttpClient.newHttpClient()
  val request = HttpRequest.newBuilder()
    .uri(URI.create("https://httpbin.org/status/404"))
    .GET()
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofString())
  val status = response.statusCode()
  status match
    case s if s >= 500 =>
      println(s"Server error: $s")
    case 404 =>
      println("Resource not found (404)")
    case 403 =>
      println("Forbidden (403)")
    case s if s >= 400 =>
      println(s"Client error: $s")
    case _ =>
      println(s"OK: $status")
      println(response.body())
end main
```

The JDK HTTP client does not throw an exception on 4xx or 5xx responses.  
It is your responsibility to check `response.statusCode()` and decide  
what to do. This design makes it easy to log error bodies or retry  
selectively based on the specific code. A match expression is a clean way  
to handle several ranges and specific codes in priority order, with a  
catch-all for the success path at the end.  

## Reading response headers

Inspect the response headers returned by the server.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI

@main def main() =
  val client = HttpClient.newHttpClient()
  val request = HttpRequest.newBuilder()
    .uri(URI.create(
      "https://httpbin.org/response-headers?X-Echo=hello"))
    .GET()
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofString())
  val headers = response.headers()
  headers.map().forEach { (name, values) =>
    println(s"$name: ${values.mkString(", ")}")
  }
end main
```

`response.headers()` returns an `HttpHeaders` object whose `map()` method  
gives you a `java.util.Map[String, List[String]]`. Header names are  
lower-cased by the JDK client in accordance with the HTTP/2 specification.  
A single header name can appear multiple times; that is why each value is  
a `List[String]`. You can also retrieve a specific header by name with  
`headers.firstValue("content-type")` which returns an `Optional[String]`.  

## Reading cookies

Capture cookies from a response using a `CookieManager`.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.{URI, CookieManager, CookiePolicy}

@main def main() =
  val cookieManager =
    new CookieManager(null, CookiePolicy.ACCEPT_ALL)
  val client = HttpClient.newBuilder()
    .cookieHandler(cookieManager)
    .build()
  val request = HttpRequest.newBuilder()
    .uri(URI.create(
      "https://httpbin.org/cookies/set?lang=scala&ver=3"))
    .GET()
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofString())
  println(s"Status: ${response.statusCode()}")
  val store = cookieManager.getCookieStore
  store.getCookies.forEach { c =>
    println(s"${c.getName} = ${c.getValue}")
  }
end main
```

Attaching a `CookieManager` to the client instructs it to store  
`Set-Cookie` headers from responses and send matching `Cookie` headers  
on subsequent requests to the same domain. `CookiePolicy.ACCEPT_ALL`  
accepts cookies from any server; production code should use  
`CookiePolicy.ACCEPT_ORIGINAL_SERVER` to prevent cross-site cookie  
leakage. The `CookieStore` allows you to enumerate, add, or remove  
cookies programmatically.  

## Setting custom User-Agent

Override the default `User-Agent` header to identify your application.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI

@main def main() =
  val client = HttpClient.newHttpClient()
  val request = HttpRequest.newBuilder()
    .uri(URI.create("https://httpbin.org/user-agent"))
    .header("User-Agent", "ScalaHTTPClient/3.0 (JVM)")
    .GET()
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofString())
  println(response.body())
end main
```

The `User-Agent` header identifies the client software to the server.  
Some APIs require a specific format or reject requests with a missing or  
generic agent string. The JDK client does not set a `User-Agent` by  
default, so you always have full control over this header. The  
`httpbin.org/user-agent` endpoint echoes the received agent string back,  
making it easy to confirm that your value was transmitted correctly.  

## Downloading a file over HTTP

Download a binary file from a URL and save it directly to disk.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI
import java.nio.file.Paths

@main def main() =
  val client = HttpClient.newHttpClient()
  val url    = "https://www.scala-lang.org/favicon.ico"
  val dest   = Paths.get("/tmp/scala-favicon.ico")
  val request = HttpRequest.newBuilder()
    .uri(URI.create(url))
    .GET()
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofFile(dest))
  println(s"Status:    ${response.statusCode()}")
  println(s"Saved to:  ${response.body()}")
  println(s"File size: ${dest.toFile.length} bytes")
end main
```

`BodyHandlers.ofFile(path)` writes the response body bytes directly to  
the specified path, bypassing an in-memory buffer. The handler creates  
or truncates the file at the given path before writing. This is the most  
efficient approach for large downloads because the data flows from the  
network straight to disk. The `response.body()` returns the `Path` that  
was written, confirming the destination.  

## Uploading a file over HTTP

Read a local file and POST its contents as the request body.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI
import java.nio.file.{Files, Paths}

@main def main() =
  val filePath = Paths.get("/tmp/upload-demo.txt")
  Files.writeString(filePath, "Hello from Scala 3 upload!")
  val client = HttpClient.newHttpClient()
  val request = HttpRequest.newBuilder()
    .uri(URI.create("https://httpbin.org/post"))
    .header("Content-Type", "text/plain")
    .POST(HttpRequest.BodyPublishers.ofFile(filePath))
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofString())
  println(s"Status: ${response.statusCode()}")
  println(response.body().take(300))
end main
```

`BodyPublishers.ofFile(path)` creates a publisher that reads the file  
lazily during transmission. The JDK determines the `Content-Length`  
automatically from the file size and sets it on the request. If the file  
is very large, the data is read in chunks and never fully loaded into  
heap. For multi-file uploads, combine `ofFile` publishers with a  
multipart body as shown in the multipart example.  

## HTTP/2 request

Request HTTP/2 explicitly and confirm the negotiated protocol version.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI

@main def main() =
  val client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)
    .build()
  val request = HttpRequest.newBuilder()
    .uri(URI.create("https://nghttp2.org/httpbin/get"))
    .GET()
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofString())
  println(s"Protocol: ${response.version()}")
  println(s"Status:   ${response.statusCode()}")
  println(response.body().take(200))
end main
```

Setting `HttpClient.Version.HTTP_2` tells the client to prefer HTTP/2.  
The actual version used depends on TLS ALPN negotiation: if the server  
advertises `h2` during the TLS handshake, the connection uses HTTP/2;  
otherwise the client falls back to HTTP/1.1. `response.version()` reports  
the protocol that was actually used. The `nghttp2.org` test server  
supports HTTP/2, making it a reliable target for verifying your client  
configuration.  

## Asynchronous HTTP request

Send an HTTP request asynchronously and process the response without  
blocking the current thread.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI
import java.util.concurrent.CompletableFuture

@main def main() =
  val client = HttpClient.newHttpClient()
  val requests = List(
    "https://httpbin.org/get",
    "https://httpbin.org/ip",
    "https://httpbin.org/uuid"
  ).map { url =>
    HttpRequest.newBuilder()
      .uri(URI.create(url))
      .GET()
      .build()
  }
  val futures = requests.map { req =>
    client.sendAsync(req, HttpResponse.BodyHandlers.ofString())
      .thenApply(r => s"${r.statusCode()} ${r.uri()}")
  }
  val all = CompletableFuture.allOf(futures*).thenRun {
    () => futures.foreach(f => println(f.join()))
  }
  all.join()
end main
```

`sendAsync` returns a `CompletableFuture[HttpResponse[T]]` immediately  
without blocking the thread. Chaining `.thenApply` transforms the result  
once it arrives. `CompletableFuture.allOf` creates a future that  
completes when all supplied futures are done, letting you wait for a  
batch of parallel requests efficiently. The `join()` at the end blocks  
only until all responses have been processed, keeping the concurrency  
model clean. This pattern is well-suited for fan-out scenarios such as  
calling multiple microservices in parallel.  

---

## Simple HTTPS request

Connect to an HTTPS endpoint using the default JVM trust store.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI

@main def main() =
  val client = HttpClient.newHttpClient()
  val request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.github.com"))
    .header("Accept", "application/json")
    .GET()
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofString())
  println(s"Status: ${response.statusCode()}")
  println(response.body().take(300))
end main
```

The JDK `HttpClient` performs TLS automatically when the URI scheme is  
`https`. It uses the default `SSLContext`, which trusts the same CA  
certificates as the JVM's default trust store, typically located in  
`$JAVA_HOME/lib/security/cacerts`. No TLS configuration is needed for  
publicly trusted certificates. The connection upgrade to TLS happens  
transparently during the TCP connect phase before any HTTP bytes are  
exchanged.  

## Loading a custom trust store

Configure the HTTP client to trust a specific CA from a PKCS12 trust  
store file.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI
import java.security.KeyStore
import javax.net.ssl.{SSLContext, TrustManagerFactory}
import java.io.FileInputStream
import scala.util.Using

@main def main() =
  val tsPath = "/path/to/truststore.p12"
  val tsPass = "changeit".toCharArray
  val ts     = KeyStore.getInstance("PKCS12")
  Using.resource(new FileInputStream(tsPath)) { fis =>
    ts.load(fis, tsPass)
  }
  val tmf = TrustManagerFactory.getInstance(
    TrustManagerFactory.getDefaultAlgorithm)
  tmf.init(ts)
  val sslCtx = SSLContext.getInstance("TLS")
  sslCtx.init(null, tmf.getTrustManagers, null)
  val client = HttpClient.newBuilder()
    .sslContext(sslCtx)
    .build()
  val request = HttpRequest.newBuilder()
    .uri(URI.create("https://internal.example.com"))
    .GET()
    .build()
  println("Custom trust store loaded successfully")
end main
```

A custom trust store is required when your server uses a certificate  
signed by an internal or private CA that is not included in the JVM's  
default bundle. Loading it into a `KeyStore` and initialising a  
`TrustManagerFactory` with that store creates trust managers that accept  
only the CAs in your custom store. Passing those managers to `SSLContext`  
and then to the `HttpClient.Builder` confines trust to your chosen  
authority. `scala.util.Using.resource` ensures the `FileInputStream` is  
closed even if the `load` call throws.  

## Loading a custom key store

Configure the client to present a client certificate during the TLS  
handshake.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI
import java.security.KeyStore
import javax.net.ssl.{SSLContext, KeyManagerFactory,
  TrustManagerFactory}
import java.io.FileInputStream
import scala.util.Using

@main def main() =
  val ksPath = "/path/to/client-keystore.p12"
  val ksPass = "keypass".toCharArray
  val ks     = KeyStore.getInstance("PKCS12")
  Using.resource(new FileInputStream(ksPath)) { fis =>
    ks.load(fis, ksPass)
  }
  val kmf = KeyManagerFactory.getInstance(
    KeyManagerFactory.getDefaultAlgorithm)
  kmf.init(ks, ksPass)
  val tmf = TrustManagerFactory.getInstance(
    TrustManagerFactory.getDefaultAlgorithm)
  tmf.init(null: KeyStore)
  val sslCtx = SSLContext.getInstance("TLS")
  sslCtx.init(kmf.getKeyManagers, tmf.getTrustManagers, null)
  val client = HttpClient.newBuilder()
    .sslContext(sslCtx)
    .build()
  println("Key store loaded; client certificate ready")
end main
```

A key store holds your private key and the corresponding certificate  
chain that identifies your client to the server. Passing a  
`KeyManagerFactory` initialised from this store into `SSLContext.init`  
enables the JVM to select and present the right certificate when the  
server requests one. `TrustManagerFactory.init(null)` falls back to the  
default JVM trust store for server certificate verification. This pattern  
is the foundation for mutual TLS, where both sides authenticate.  

## Disabling certificate validation

Bypass all certificate checks for testing against internal environments.  
**Never use this in production.**  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI
import java.security.cert.X509Certificate
import javax.net.ssl.{SSLContext, TrustManager, X509TrustManager}

@main def main() =
  val trustAll = Array[TrustManager](
    new X509TrustManager:
      def getAcceptedIssuers = Array.empty[X509Certificate]
      def checkClientTrusted(
        certs: Array[X509Certificate], t: String) = ()
      def checkServerTrusted(
        certs: Array[X509Certificate], t: String) = ()
  )
  val sslCtx = SSLContext.getInstance("TLS")
  sslCtx.init(null, trustAll, new java.security.SecureRandom())
  val client = HttpClient.newBuilder()
    .sslContext(sslCtx)
    .build()
  // WARNING: trust-all context — for local testing only
  println("Trust-all SSL context created (NEVER use in prod)")
end main
```

This example creates a `TrustManager` that accepts every certificate  
unconditionally. It is sometimes used in automated test pipelines that  
spin up local services with self-signed certificates and where installing  
those certificates into a trust store is impractical. The danger is that  
it makes your code completely vulnerable to man-in-the-middle attacks: an  
attacker can intercept and read or modify all traffic. Always load the  
actual certificate into a custom trust store for production and staging  
environments instead.  

## Hostname verification

Verify that the server's TLS certificate matches the hostname you  
connected to.  

```scala
import javax.net.ssl.{SSLSocket, SSLParameters,
  SSLSocketFactory}

@main def main() =
  val factory = SSLSocketFactory.getDefault
    .asInstanceOf[SSLSocketFactory]
  val socket = factory.createSocket("www.example.com", 443)
    .asInstanceOf[SSLSocket]
  val params = new SSLParameters()
  params.setEndpointIdentificationAlgorithm("HTTPS")
  socket.setSSLParameters(params)
  try
    socket.startHandshake()
    val session = socket.getSession
    println(s"Peer host:    ${session.getPeerHost}")
    println(s"Protocol:     ${session.getProtocol}")
    println(s"Cipher suite: ${session.getCipherSuite}")
  finally
    socket.close()
end main
```

Setting `setEndpointIdentificationAlgorithm("HTTPS")` on `SSLParameters`  
instructs the JVM's TLS implementation to verify that the certificate's  
Subject Alternative Names or Common Name matches the hostname in your  
connection request. Without this check, an attacker with a valid  
certificate for a different domain could impersonate your target server.  
The JDK `HttpClient` enables this automatically; when using raw  
`SSLSocket`, you must set it manually as shown here.  

## Mutual TLS

Authenticate both client and server during the TLS handshake (mTLS).  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI
import java.security.KeyStore
import javax.net.ssl.{SSLContext, KeyManagerFactory,
  TrustManagerFactory}
import java.io.FileInputStream
import scala.util.Using

@main def main() =
  val ksPath = "/path/to/client-keystore.p12"
  val tsPath = "/path/to/server-truststore.p12"
  val ksPass = "kspass".toCharArray
  val tsPass = "tspass".toCharArray
  val ks = KeyStore.getInstance("PKCS12")
  Using.resource(new FileInputStream(ksPath))(ks.load(_, ksPass))
  val ts = KeyStore.getInstance("PKCS12")
  Using.resource(new FileInputStream(tsPath))(ts.load(_, tsPass))
  val kmf = KeyManagerFactory.getInstance(
    KeyManagerFactory.getDefaultAlgorithm)
  kmf.init(ks, ksPass)
  val tmf = TrustManagerFactory.getInstance(
    TrustManagerFactory.getDefaultAlgorithm)
  tmf.init(ts)
  val sslCtx = SSLContext.getInstance("TLS")
  sslCtx.init(kmf.getKeyManagers, tmf.getTrustManagers, null)
  val client = HttpClient.newBuilder()
    .sslContext(sslCtx)
    .build()
  println("mTLS client configured successfully")
end main
```

In mutual TLS the server requires the client to present a valid  
certificate, not just the other way around. This eliminates the risk of  
an unauthorised client connecting even if they know the server's address.  
`KeyManagerFactory` provides the client certificate during the handshake,  
and `TrustManagerFactory` verifies the server's certificate. Both stores  
are loaded from PKCS12 files here, which is the format recommended for  
new deployments over the older JKS format.  

## Debugging TLS handshake

Enable verbose TLS logging to diagnose handshake failures.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI

@main def main() =
  System.setProperty("javax.net.debug", "ssl:handshake")
  val client = HttpClient.newHttpClient()
  val request = HttpRequest.newBuilder()
    .uri(URI.create("https://www.scala-lang.org"))
    .GET()
    .build()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofString())
  println(s"Status: ${response.statusCode()}")
  // Detailed TLS output appears on stderr
end main
```

The JVM property `javax.net.debug` enables the built-in TLS diagnostics  
engine. Setting it to `ssl:handshake` prints every step of the handshake:  
the cipher suites exchanged in `ClientHello`, the certificate chain  
provided by the server, the key exchange parameters, and the final  
`Finished` confirmation. Setting it to `all` produces even more output,  
including every record sent and received. This is the first tool to reach  
for when a connection fails with `SSLHandshakeException`. The output goes  
to `stderr` so it does not interfere with your application's normal  
output on `stdout`.  

## Extracting certificate information

Read the X.509 certificate details from an HTTPS connection.  

```scala
import java.net.URL
import javax.net.ssl.HttpsURLConnection
import java.security.cert.X509Certificate

@main def main() =
  val url  = new URL("https://www.scala-lang.org")
  val conn = url.openConnection()
    .asInstanceOf[HttpsURLConnection]
  conn.connect()
  try
    conn.getServerCertificates.foreach { cert =>
      val x509 = cert.asInstanceOf[X509Certificate]
      println(s"Subject: ${x509.getSubjectX500Principal.getName}")
      println(s"Issuer:  ${x509.getIssuerX500Principal.getName}")
      println(s"Valid from: ${x509.getNotBefore}")
      println(s"Expires:    ${x509.getNotAfter}")
      println(s"SAN: ${
        Option(x509.getSubjectAlternativeNames)
          .map(_.size).getOrElse(0)} names")
      println("---")
    }
  finally
    conn.disconnect()
end main
```

`HttpsURLConnection.getServerCertificates` returns the full certificate  
chain as the server presented it during the TLS handshake. Casting each  
`Certificate` to `X509Certificate` unlocks the full X.509 API: issuer,  
subject, validity period, serial number, signature algorithm, and  
extensions such as Subject Alternative Names. This information is useful  
for compliance audits, monitoring dashboards, and automated certificate  
management tools that need to track what certificates are in use.  

## Verifying certificate expiration

Check whether the server's certificate will expire within 30 days.  

```scala
import java.net.URL
import javax.net.ssl.HttpsURLConnection
import java.security.cert.X509Certificate
import java.time.ZoneId
import java.time.format.DateTimeFormatter
import java.util.Date

@main def main() =
  val hosts = List(
    "www.scala-lang.org",
    "www.github.com"
  )
  val fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd")
    .withZone(ZoneId.systemDefault())
  hosts.foreach { host =>
    val conn = new URL(s"https://$host")
      .openConnection().asInstanceOf[HttpsURLConnection]
    conn.setConnectTimeout(5000)
    conn.connect()
    try
      val cert = conn.getServerCertificates.head
        .asInstanceOf[X509Certificate]
      val now      = new Date()
      val expiry   = cert.getNotAfter
      val daysLeft =
        (expiry.getTime - now.getTime) / 86_400_000L
      val warn = if daysLeft < 30 then " *** EXPIRING SOON"
                 else ""
      println(
        s"$host | ${fmt.format(expiry.toInstant)} | " +
        s"$daysLeft days$warn")
    finally
      conn.disconnect()
  }
end main
```

Certificate expiration is one of the most common causes of unexpected  
outages in production systems. Running this check periodically in a  
monitoring job and alerting when a certificate has fewer than 30 days  
remaining gives you time to renew before the site goes down. The example  
calculates `daysLeft` by subtracting the current epoch milliseconds from  
the certificate's `notAfter` timestamp and dividing by the number of  
milliseconds per day. Extending this to a full monitoring tool requires  
only a scheduler and a notification mechanism.  

## Connecting with a self-signed certificate

Trust a single self-signed certificate without disabling all validation.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI
import java.security.KeyStore
import java.security.cert.{CertificateFactory, X509Certificate}
import javax.net.ssl.{SSLContext, TrustManagerFactory}
import java.io.FileInputStream
import scala.util.Using

@main def main() =
  val certPath = "/path/to/selfsigned.crt"
  val cf       = CertificateFactory.getInstance("X.509")
  val cert = Using.resource(new FileInputStream(certPath)) { fis =>
    cf.generateCertificate(fis).asInstanceOf[X509Certificate]
  }
  val ts = KeyStore.getInstance(KeyStore.getDefaultType)
  ts.load(null, null)
  ts.setCertificateEntry("self-signed", cert)
  val tmf = TrustManagerFactory.getInstance(
    TrustManagerFactory.getDefaultAlgorithm)
  tmf.init(ts)
  val sslCtx = SSLContext.getInstance("TLS")
  sslCtx.init(null, tmf.getTrustManagers, null)
  val client = HttpClient.newBuilder()
    .sslContext(sslCtx)
    .build()
  println("Self-signed cert trusted; client ready")
end main
```

This approach is safer than disabling all validation because it trusts  
only one specific certificate. The `CertificateFactory` loads the PEM or  
DER encoded certificate directly from a file and inserts it into an  
empty in-memory `KeyStore` under an alias. The `TrustManagerFactory`  
built from that store accepts any server that presents exactly this  
certificate. This pattern is ideal for development environments, Docker  
Compose setups, and internal services that rotate certificates  
infrequently.  

---

## Connecting via SSH with password

Open an SSH session to a remote host using password authentication.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier

@main def main() =
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    println(s"Connected:     ${ssh.isConnected}")
    println(s"Authenticated: ${ssh.isAuthenticated}")
    println(s"Remote version: ${ssh.getTransport.getServerVersion}")
  finally
    ssh.disconnect()
end main
```

The SSHJ library (`com.hierynomus:sshj:0.38.0`) provides a clean Scala-  
friendly API for all SSH operations. `SSHClient.connect` performs the TCP  
connection and SSH protocol negotiation. `authPassword` sends the  
credentials. `PromiscuousVerifier` accepts any host key — useful during  
development but dangerous in production. In production, replace it with  
`OpenSSHKnownHosts` to verify the server's identity against `~/.ssh/known_hosts`.  
Always disconnect in a `finally` block to release the underlying socket.  

## Connecting via SSH with private key

Authenticate with an RSA or Ed25519 private key file.  

```scala
import net.schmizz.sshj.SSHClient

@main def main() =
  val ssh = new SSHClient()
  ssh.loadKnownHosts()
  ssh.connect("192.168.1.100")
  try
    ssh.authPublickey("alice", "/home/alice/.ssh/id_ed25519")
    println(s"Authenticated: ${ssh.isAuthenticated}")
    println(s"Remote host:   ${ssh.getRemoteHostname}")
  finally
    ssh.disconnect()
end main
```

`loadKnownHosts()` reads `~/.ssh/known_hosts` and configures the client  
to reject any server whose host key does not match the stored fingerprint.  
This is the correct production setting: it prevents connecting to an  
impersonated server. `authPublickey` reads the private key file, signs a  
challenge from the server, and sends the signature. The private key may  
be password-protected; pass the passphrase as a third argument if needed.  
Ed25519 keys are recommended for new deployments because they are shorter  
and faster than RSA while offering equivalent security.  

## Executing a remote command

Run a single command on the remote server and collect the output.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier

@main def main() =
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    val session = ssh.startSession()
    try
      val cmd = session.exec("hostname && uname -r")
      val output = new String(
        cmd.getInputStream.readAllBytes()).trim
      cmd.join()
      println(output)
    finally
      session.close()
  finally
    ssh.disconnect()
end main
```

`ssh.startSession()` opens an SSH session channel. `session.exec(command)`  
requests that the server execute the shell command and returns a `Command`  
object with input, output, and error streams. `readAllBytes()` blocks  
until the command closes its output stream, which happens when it exits.  
Calling `cmd.join()` waits for the command to fully terminate and its  
exit status to be reported. Always close the session inside a `finally`  
block to release the SSH channel.  

## Reading remote command output

Stream standard output and standard error from a remote command  
separately.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier
import java.io.{BufferedReader, InputStreamReader}

@main def main() =
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    val session = ssh.startSession()
    try
      val cmd = session.exec("ls -la /home/alice")
      val stdout = new BufferedReader(
        new InputStreamReader(cmd.getInputStream))
      val stderr = new BufferedReader(
        new InputStreamReader(cmd.getErrorStream))
      println("=== stdout ===")
      Iterator.continually(stdout.readLine())
        .takeWhile(_ != null)
        .foreach(println)
      println("=== stderr ===")
      Iterator.continually(stderr.readLine())
        .takeWhile(_ != null)
        .foreach(println)
      cmd.join()
    finally
      session.close()
  finally
    ssh.disconnect()
end main
```

Reading stdout and stderr independently prevents a deadlock that can  
occur if one stream fills its buffer while the other is not being read.  
Wrapping each stream in a `BufferedReader` and using an  
`Iterator.continually` / `takeWhile(_ != null)` pattern gives a clean  
Scala idiom for consuming Java `InputStream` line by line. For commands  
that produce large output, consider reading both streams in separate  
threads or using a `StreamCopier` from the SSHJ utility package.  

## Handling command exit codes

Check the remote process exit code to determine success or failure.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier

@main def main() =
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    val session = ssh.startSession()
    try
      val cmd = session.exec(
        "test -f /etc/hosts && echo 'found' || echo 'missing'")
      val out = new String(
        cmd.getInputStream.readAllBytes()).trim
      val err = new String(
        cmd.getErrorStream.readAllBytes()).trim
      cmd.join()
      val code = cmd.getExitStatus
      println(s"stdout:    $out")
      println(s"stderr:    $err")
      println(s"exit code: $code")
      if code == 0 then println("Command succeeded")
      else println(s"Command failed (exit $code)")
    finally
      session.close()
  finally
    ssh.disconnect()
end main
```

`cmd.getExitStatus` returns the POSIX exit code as an `Integer`. A value  
of zero indicates success; any non-zero value indicates a failure whose  
meaning depends on the command. It is critical to call `cmd.join()` before  
reading the exit status, because the status is sent from the server as a  
separate SSH message after the command process terminates. Skipping  
`join()` may return `null` if the status message has not yet arrived.  

## Port forwarding — local

Forward a local port to a remote service through an SSH tunnel.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier
import net.schmizz.sshj.connection.channel.direct.LocalPortForwarder
import java.net.ServerSocket

@main def main() =
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    val serverSocket = new ServerSocket(18080)
    val params = new LocalPortForwarder.Parameters(
      "127.0.0.1", 18080,
      "db.internal.example.com", 5432)
    val forwarder = ssh.newLocalPortForwarder(
      params, serverSocket)
    println("Forwarding localhost:18080 → db.internal:5432")
    println("Press Ctrl+C to stop forwarding")
    forwarder.listen()
  finally
    ssh.disconnect()
end main
```

Local port forwarding makes a remote service appear on a local port.  
Your application connects to `localhost:18080` as if it were a local  
database, while SSHJ transparently tunnels the TCP data through the SSH  
connection to `db.internal.example.com:5432`. This is useful for  
accessing databases, internal dashboards, or any service that is only  
reachable from within a private network. The `forwarder.listen()` call  
blocks and handles incoming connections until the program is interrupted.  

## Port forwarding — remote

Ask the SSH server to forward connections from a remote port to a local  
service.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier
import net.schmizz.sshj.connection.channel.forwarded.{
  RemotePortForwarder, SocketForwardingConnectListener}
import java.net.InetSocketAddress

@main def main() =
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    val rpf = ssh.getRemotePortForwarder
    val binding = rpf.bind(
      RemotePortForwarder.Forward.toRemoteAddress(
        new InetSocketAddress("0.0.0.0", 19090)),
      new SocketForwardingConnectListener(
        new InetSocketAddress("localhost", 3000)))
    println(
      s"Remote port ${binding.getPort} → local 3000")
    ssh.getTransport.join()
  finally
    ssh.disconnect()
end main
```

Remote port forwarding is the reverse of local forwarding. The SSH server  
listens on a port on the remote machine and forwards incoming connections  
back through the tunnel to a service running on the machine where this  
code is running. This is often called a "reverse tunnel" and is useful  
for exposing a local development server to a public host or for letting  
a server in a private network reach back into your network. The  
`transport.join()` keeps the program alive while the tunnel is active.  

## SSH keep-alive

Configure periodic keep-alive messages to prevent session timeouts.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier
import net.schmizz.sshj.connection.keepalive.KeepAliveProvider
import net.schmizz.sshj.DefaultConfig

@main def main() =
  val config = new DefaultConfig()
  config.setKeepAliveProvider(KeepAliveProvider.KEEP_ALIVE)
  val ssh = new SSHClient(config)
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    ssh.getConnection.getKeepAlive.setKeepAliveInterval(30)
    println("Keep-alive: 30-second interval")
    val session = ssh.startSession()
    try
      val cmd = session.exec("sleep 5 && echo done")
      println(new String(
        cmd.getInputStream.readAllBytes()).trim)
      cmd.join()
    finally
      session.close()
  finally
    ssh.disconnect()
end main
```

Network firewalls and NAT devices commonly close idle TCP connections  
after 60–300 seconds of inactivity. An SSH session that is waiting for  
a long-running command or holding a port-forward tunnel open may be  
silently killed. Keep-alive messages prevent this by sending a small SSH  
request every 30 seconds. The `KeepAliveProvider.KEEP_ALIVE` strategy  
sends these messages through the SSH connection layer, which causes the  
server to respond and keeps the underlying TCP connection alive through  
any intervening firewalls.  

## SSH session timeout handling

Set connection and socket timeouts and handle transport errors gracefully.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier
import net.schmizz.sshj.transport.TransportException
import java.util.concurrent.TimeUnit
import java.net.ConnectException

@main def main() =
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.setConnectTimeout(5000)
  ssh.setTimeout(30000)
  try
    ssh.connect("192.168.1.100")
    ssh.authPassword("alice", "s3cret")
    val session = ssh.startSession()
    try
      val cmd = session.exec("echo hello")
      cmd.join(10, TimeUnit.SECONDS)
      println(new String(
        cmd.getInputStream.readAllBytes()).trim)
      println(s"Exit: ${cmd.getExitStatus}")
    finally
      session.close()
  catch
    case e: TransportException =>
      println(s"SSH transport error: ${e.getMessage}")
    case e: ConnectException =>
      println(s"Cannot connect: ${e.getMessage}")
    case e: Exception =>
      println(s"Unexpected: ${e.getMessage}")
  finally
    if ssh.isConnected then ssh.disconnect()
end main
```

`setConnectTimeout` controls how long the client waits for the TCP  
connection to be established. `setTimeout` controls the read timeout on  
the SSH transport socket, after which an idle connection is closed and a  
`TransportException` is thrown. `cmd.join(timeout, unit)` prevents a  
runaway remote command from blocking your thread forever. Checking  
`ssh.isConnected` in `finally` avoids a `NullPointerException` if the  
connection failed before the `connect()` call returned.  

## Listing remote directory via SSH

Execute an `ls` command and print the remote directory contents.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier
import java.io.{BufferedReader, InputStreamReader}

@main def main() =
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    val session = ssh.startSession()
    try
      val cmd = session.exec("ls -la /home/alice/")
      val reader = new BufferedReader(
        new InputStreamReader(cmd.getInputStream))
      println("Remote directory listing:")
      Iterator.continually(reader.readLine())
        .takeWhile(_ != null)
        .foreach(line => println(s"  $line"))
      cmd.join()
    finally
      session.close()
  finally
    ssh.disconnect()
end main
```

Running `ls` via an SSH command is a quick way to inspect a remote  
directory when all you need is a simple listing and you do not want to  
open a full SFTP subsystem. It is also useful for checking whether a  
specific file exists with `test -e filename && echo yes`. For structured  
metadata such as file sizes, permissions, and timestamps that you need  
to process programmatically, prefer the SFTP `stat` and `ls` operations  
demonstrated in the SFTP section below.  

---

## Connecting via SFTP

Open an SFTP client and inspect a remote path attribute.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier

@main def main() =
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    val sftp = ssh.newSFTPClient()
    try
      val attrs = sftp.stat("/home/alice")
      println(s"Home dir UID: ${attrs.getUID}")
      println(s"Home dir GID: ${attrs.getGID}")
      println(s"Mode:         ${attrs.getMode}")
    finally
      sftp.close()
  finally
    ssh.disconnect()
end main
```

`ssh.newSFTPClient()` negotiates the SFTP subsystem over the existing SSH  
connection. The subsystem runs as a separate channel inside the SSH  
session, so you do not need a separate `startSession()` call. `sftp.stat`  
performs a remote `stat` operation equivalent to the Unix `stat` command  
and returns a `FileAttributes` object with size, ownership, permissions,  
and timestamps. Always close the `SFTPClient` before disconnecting SSH;  
this sends the proper SFTP close message to the server.  

## Listing remote directory

List the contents of a remote directory with file names and permissions.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier

@main def main() =
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    val sftp = ssh.newSFTPClient()
    try
      val entries = sftp.ls("/home/alice")
      entries.forEach { entry =>
        val attrs = entry.getAttributes
        println(
          f"${attrs.getMode}%-12s  ${attrs.getSize}%10d  " +
          s"${entry.getName}")
      }
    finally
      sftp.close()
  finally
    ssh.disconnect()
end main
```

`sftp.ls(path)` returns a `java.util.List[RemoteResourceInfo]`. Each  
entry carries the file name and a `FileAttributes` record with the same  
fields as `stat`. The formatted output mimics `ls -l`: permissions on  
the left, size in the middle, and name on the right. Unlike the SSH `ls`  
command approach, this gives you structured data that you can filter,  
sort, and process without parsing text.  

## Uploading a file via SFTP

Transfer a local file to the remote server over SFTP.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier
import net.schmizz.sshj.xfer.FileSystemFile
import java.nio.file.{Files, Paths}

@main def main() =
  val localPath = Paths.get("/tmp/sftp-upload.txt")
  Files.writeString(localPath, "Hello from Scala 3 SFTP!")
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    val sftp = ssh.newSFTPClient()
    try
      sftp.put(
        new FileSystemFile(localPath.toFile),
        "/home/alice/sftp-upload.txt")
      println("Upload complete")
    finally
      sftp.close()
  finally
    ssh.disconnect()
end main
```

`sftp.put(source, remotePath)` transfers the file atomically by writing  
to a temporary file on the server and renaming it on completion.  
`FileSystemFile` wraps a `java.io.File` and implements SSHJ's  
`LocalSourceFile` interface, which provides metadata (name, size,  
permissions) that the SFTP protocol requires alongside the bytes.  
The remote path can be either a full file path or a directory; when it  
is a directory, the local file name is preserved.  

## Downloading a file via SFTP

Transfer a remote file to the local filesystem over SFTP.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier
import net.schmizz.sshj.xfer.FileSystemFile

@main def main() =
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    val sftp = ssh.newSFTPClient()
    try
      sftp.get(
        "/home/alice/report.csv",
        new FileSystemFile("/tmp/report.csv"))
      println("Download complete: /tmp/report.csv")
    finally
      sftp.close()
  finally
    ssh.disconnect()
end main
```

`sftp.get(remotePath, localDest)` downloads the remote file to the  
supplied `LocalDestFile`. `FileSystemFile` created from a path string  
writes the bytes directly to that path, creating the file if it does not  
exist or overwriting it if it does. For downloading into a specific  
directory while keeping the original file name, pass a  
`FileSystemFile` wrapping the directory path; SSHJ appends the remote  
file name automatically.  

## Resuming interrupted transfers

Resume a partially downloaded file by starting from the byte offset  
already received.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier
import net.schmizz.sshj.sftp.OpenMode
import java.nio.file.{Files, Paths, StandardOpenOption}
import java.util.EnumSet

@main def main() =
  val localPath  = Paths.get("/tmp/resume-target.bin")
  val remotePath = "/home/alice/bigfile.bin"
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    val sftp = ssh.newSFTPClient()
    try
      val offset =
        if Files.exists(localPath) then Files.size(localPath)
        else 0L
      println(s"Resuming from offset: $offset")
      val rf = sftp.open(
        remotePath, EnumSet.of(OpenMode.READ))
      try
        val out = Files.newOutputStream(localPath,
          StandardOpenOption.APPEND,
          StandardOpenOption.CREATE)
        try
          val buf = new Array[Byte](8192)
          var pos = offset
          var n   = rf.read(pos, buf, 0, buf.length)
          while n > 0 do
            out.write(buf, 0, n)
            pos += n
            n = rf.read(pos, buf, 0, buf.length)
          println(s"Transfer complete. Total: $pos bytes")
        finally
          out.close()
      finally
        rf.close()
    finally
      sftp.close()
  finally
    ssh.disconnect()
end main
```

SFTP's `RemoteFile.read(offset, buffer, start, len)` supports random  
access: you specify the byte offset at which to start reading. By  
measuring the size of the already-downloaded file and beginning the read  
there, you continue from where a previous transfer was interrupted.  
Opening the local file with `StandardOpenOption.APPEND` ensures new  
bytes are added at the end rather than overwriting the existing data.  
This manual resume loop is useful when a high-level `get` call does not  
support restart and the files are large enough that restarting from zero  
would be expensive.  

## Reading file attributes

Retrieve metadata for a remote file, including size, ownership, and  
timestamps.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier

@main def main() =
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    val sftp = ssh.newSFTPClient()
    try
      val attrs = sftp.stat("/home/alice/.bashrc")
      println(s"Size:  ${attrs.getSize} bytes")
      println(s"UID:   ${attrs.getUID}")
      println(s"GID:   ${attrs.getGID}")
      println(s"Mode:  ${attrs.getMode}")
      println(s"Mtime: ${attrs.getMtime}")
      println(s"Atime: ${attrs.getAtime}")
      println(s"Type:  ${attrs.getType}")
    finally
      sftp.close()
  finally
    ssh.disconnect()
end main
```

`sftp.stat` follows symbolic links; use `sftp.lstat` if you want the  
attributes of the link itself rather than its target. The `FileAttributes`  
object mirrors the SFTP protocol's file attributes block: it always  
contains size, permissions (mode), owner IDs, and modification times for  
regular SFTP servers. Timestamps are returned as Unix epoch seconds.  
These attributes are the same ones returned by each entry in `sftp.ls`,  
so you can call `stat` for single-file lookups or use `ls` for batch  
queries.  

## Creating directories

Create a directory hierarchy on the remote server.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier

@main def main() =
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    val sftp = ssh.newSFTPClient()
    try
      val newDir = "/home/alice/backups/2025/june"
      sftp.mkdirs(newDir)
      println(s"Created: $newDir")
      val entries = sftp.ls("/home/alice/backups")
      entries.forEach(e => println(s"  ${e.getName}"))
    finally
      sftp.close()
  finally
    ssh.disconnect()
end main
```

`sftp.mkdirs(path)` creates all missing intermediate directories in the  
path, similar to the Unix `mkdir -p` command. It does not fail if the  
final directory already exists. Using `mkdirs` before uploading ensures  
the target path exists without needing a separate check-and-create  
sequence. The subsequent `sftp.ls` call on the parent directory confirms  
the new directory appears in the listing.  

## Deleting files via SFTP

Remove a file from the remote server.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier
import net.schmizz.sshj.sftp.SFTPException

@main def main() =
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    val sftp = ssh.newSFTPClient()
    try
      val remoteFile = "/home/alice/obsolete.log"
      try
        sftp.rm(remoteFile)
        println(s"Deleted: $remoteFile")
      catch
        case e: SFTPException =>
          println(s"Could not delete: ${e.getMessage}")
    finally
      sftp.close()
  finally
    ssh.disconnect()
end main
```

`sftp.rm(path)` sends an SFTP `REMOVE` request to the server. The server  
returns an error if the path does not exist, if it is a directory (use  
`rmdir` for directories), or if the authenticated user lacks write  
permission on the parent directory. The `SFTPException` carries a  
`StatusCode` that identifies the specific reason for the failure, letting  
you distinguish between "not found" and "permission denied" in your error  
handling logic.  

## Renaming files via SFTP

Rename or move a file to a new path on the remote server.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier
import net.schmizz.sshj.sftp.SFTPException

@main def main() =
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    val sftp = ssh.newSFTPClient()
    try
      val oldPath = "/home/alice/draft.txt"
      val newPath = "/home/alice/final.txt"
      try
        sftp.rename(oldPath, newPath)
        println(s"Renamed: $oldPath → $newPath")
      catch
        case e: SFTPException =>
          println(s"Rename failed: ${e.getMessage}")
    finally
      sftp.close()
  finally
    ssh.disconnect()
end main
```

`sftp.rename(oldPath, newPath)` sends an SFTP `RENAME` request. If the  
source and destination are on the same filesystem on the server, this is  
an atomic operation backed by the kernel's `rename` system call. If the  
destination path already exists, the behaviour depends on the server  
implementation: some servers overwrite silently while others return an  
error. The SFTP v5 and later protocols include an `OVERWRITE` flag; SSHJ  
targets SFTP v3 by default, which is the widest-compatible choice.  

## Handling SFTP errors

Inspect SFTP error codes to distinguish between different failure  
conditions.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier
import net.schmizz.sshj.sftp.{SFTPException, Response}

@main def main() =
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    val sftp = ssh.newSFTPClient()
    try
      val paths = List(
        "/home/alice/.bashrc",
        "/nonexistent/file.txt",
        "/root/secret.txt"
      )
      paths.foreach { p =>
        try
          val attrs = sftp.stat(p)
          println(s"OK   $p (${attrs.getSize} bytes)")
        catch
          case e: SFTPException =>
            val code = e.getStatusCode
            val msg = code match
              case Response.StatusCode.NO_SUCH_FILE =>
                "not found"
              case Response.StatusCode.PERMISSION_DENIED =>
                "permission denied"
              case _ =>
                s"SFTP error [$code]"
            println(s"FAIL $p — $msg")
      }
    finally
      sftp.close()
  finally
    ssh.disconnect()
end main
```

`SFTPException.getStatusCode()` returns a `Response.StatusCode` enum  
value that maps directly to the SFTP protocol error codes defined in the  
specification. Matching on `NO_SUCH_FILE` and `PERMISSION_DENIED`  
separately lets you give users precise diagnostic messages instead of a  
generic error. Other useful codes include `FAILURE` (catch-all server  
error), `BAD_MESSAGE` (protocol mismatch), and `OP_UNSUPPORTED` (the  
server does not implement the requested operation). Wrapping each  
operation individually lets you continue processing the rest of the list  
even when one item fails.  

---

## Uploading a file via SCP

Copy a local file to a remote server using the SCP protocol.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier
import net.schmizz.sshj.xfer.FileSystemFile
import java.nio.file.{Files, Paths}

@main def main() =
  val localPath = Paths.get("/tmp/scp-upload.txt")
  Files.writeString(localPath, "Hello via SCP from Scala 3!")
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    ssh.useCompression()
    ssh.newSCPFileTransfer()
      .upload(
        new FileSystemFile(localPath.toFile),
        "/home/alice/")
    println("SCP upload complete")
  finally
    ssh.disconnect()
end main
```

SCP runs as a remote command (`scp -t /remote/path`) on the server side.  
SSHJ's `SCPFileTransfer` handles the SCP wire protocol, sending the file  
size, permissions, and bytes in the format the server expects. Calling  
`ssh.useCompression()` negotiates zlib compression on the SSH transport,  
which can halve transfer time for compressible data such as text logs.  
Unlike SFTP, SCP cannot resume interrupted transfers or query remote  
attributes, so prefer SFTP for production bulk transfer jobs.  

## Downloading a file via SCP

Copy a remote file to the local filesystem using SCP.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier
import net.schmizz.sshj.xfer.FileSystemFile

@main def main() =
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    ssh.newSCPFileTransfer()
      .download(
        "/home/alice/data.csv",
        new FileSystemFile("/tmp/scp-data.csv"))
    println("SCP download complete: /tmp/scp-data.csv")
  finally
    ssh.disconnect()
end main
```

SCP downloads work by running `scp -f /remote/path` on the server, which  
streams the file content back over the SSH channel. SSHJ parses the SCP  
framing and writes the bytes to the local `FileSystemFile` destination.  
The local file is created or overwritten atomically once the transfer  
finishes. SCP preserves file permissions and timestamps sent by the  
server, which is useful when you need the modification time of the  
original file to match on the local copy.  

## Verifying file integrity after SCP download

Download a file via SCP and verify its SHA-256 digest.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier
import net.schmizz.sshj.xfer.FileSystemFile
import java.nio.file.{Files, Paths}
import java.security.MessageDigest

@main def main() =
  val localPath = Paths.get("/tmp/verified.bin")
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    ssh.newSCPFileTransfer()
      .download(
        "/home/alice/package.tar.gz",
        new FileSystemFile(localPath.toFile))
    val bytes  = Files.readAllBytes(localPath)
    val md     = MessageDigest.getInstance("SHA-256")
    val digest = md.digest(bytes)
    val hex    = digest.map(b => f"$b%02x").mkString
    println(s"Downloaded: $localPath")
    println(s"Size:       ${bytes.length} bytes")
    println(s"SHA-256:    $hex")
  finally
    ssh.disconnect()
end main
```

Computing a SHA-256 digest after downloading is the standard way to  
verify that the file was not corrupted during transfer. In a real  
workflow, the expected digest is published alongside the file (in a  
`.sha256` sidecar file or on a release page), and you compare the  
computed value against the published one. If they differ, the file is  
discarded and the download is retried. `MessageDigest.getInstance("SHA-256")`  
uses the JDK's built-in implementation; no third-party library is needed.  

---

## HTTP JSON client helper

A reusable helper object that wraps `HttpClient` for JSON GET and POST  
calls.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI
import java.time.Duration

object JsonHttp:
  private val client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(10))
    .followRedirects(HttpClient.Redirect.NORMAL)
    .build()

  def get(url: String,
          headers: Map[String, String] = Map.empty): String =
    val builder = HttpRequest.newBuilder(URI.create(url))
      .header("Accept", "application/json")
    val builderWithHeaders = headers.foldLeft(builder) {
      (b, kv) => b.header(kv._1, kv._2)
    }
    val request  = builderWithHeaders.GET().build()
    val response = client.send(
      request, HttpResponse.BodyHandlers.ofString())
    if response.statusCode() >= 400 then
      throw RuntimeException(
        s"HTTP ${response.statusCode()}: $url")
    response.body()

  def post(url: String, jsonBody: String): String =
    val request = HttpRequest.newBuilder(URI.create(url))
      .header("Content-Type", "application/json")
      .header("Accept", "application/json")
      .POST(HttpRequest.BodyPublishers.ofString(jsonBody))
      .build()
    val response = client.send(
      request, HttpResponse.BodyHandlers.ofString())
    if response.statusCode() >= 400 then
      throw RuntimeException(
        s"HTTP ${response.statusCode()}: $url")
    response.body()

@main def main() =
  val body = JsonHttp.get("https://httpbin.org/json")
  println(body.take(200))
  val result = JsonHttp.post(
    "https://httpbin.org/post",
    """{"key": "value", "lang": "scala"}""")
  println(result.take(200))
end main
```

Centralising the `HttpClient` in a singleton object ensures connection  
pooling is shared across all calls in the application, avoiding the  
overhead of creating a new client for every request. The `get` method  
accepts optional extra headers through a `Map`, making it easy to add  
`Authorization` or `X-Api-Key` headers without changing the call site.  
Throwing a `RuntimeException` on error status codes keeps the API  
simple: callers see either a result or an exception, not a raw  
`HttpResponse` that they must check every time.  

## Simple HTTP API wrapper

Wrap a REST API in a typed Scala class with dedicated methods per  
endpoint.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI

class GitHubApi(token: String):
  private val base   = "https://api.github.com"
  private val client = HttpClient.newHttpClient()

  private def req(path: String): HttpRequest =
    HttpRequest.newBuilder()
      .uri(URI.create(s"$base$path"))
      .header("Authorization", s"Bearer $token")
      .header("Accept", "application/vnd.github+json")
      .header("X-GitHub-Api-Version", "2022-11-28")
      .GET()
      .build()

  def getUser(login: String): String =
    client.send(req(s"/users/$login"),
      HttpResponse.BodyHandlers.ofString()).body()

  def listRepos(login: String, perPage: Int = 5): String =
    client.send(req(s"/users/$login/repos?per_page=$perPage"),
      HttpResponse.BodyHandlers.ofString()).body()

@main def main() =
  val token = sys.env.getOrElse("GITHUB_TOKEN", "")
  val api   = new GitHubApi(token)
  println(api.getUser("scala").take(300))
  println(api.listRepos("scala").take(300))
end main
```

Encapsulating the base URL, authentication header, and API version  
header in a private `req` helper method avoids repetition across all  
endpoint methods. Every public method in `GitHubApi` expresses the  
business intent clearly — `getUser`, `listRepos` — rather than exposing  
HTTP internals to the caller. Reading the token from an environment  
variable keeps credentials out of source code. This pattern scales well  
as you add more endpoints: add one method and one test, and the shared  
infrastructure stays untouched.  

## SSH command runner utility

A reusable utility that runs a command on a remote host and returns  
structured output.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier

case class CmdResult(
  stdout: String,
  stderr: String,
  exitCode: Int
)

object SshRunner:
  def run(host: String, user: String,
          password: String, command: String): CmdResult =
    val ssh = new SSHClient()
    ssh.addHostKeyVerifier(new PromiscuousVerifier())
    ssh.connect(host)
    try
      ssh.authPassword(user, password)
      val session = ssh.startSession()
      try
        val cmd = session.exec(command)
        val out = new String(
          cmd.getInputStream.readAllBytes()).trim
        val err = new String(
          cmd.getErrorStream.readAllBytes()).trim
        cmd.join()
        CmdResult(out, err,
          Option(cmd.getExitStatus).map(_.intValue).getOrElse(-1))
      finally
        session.close()
    finally
      ssh.disconnect()

@main def main() =
  val r = SshRunner.run(
    "192.168.1.100", "alice", "s3cret", "uname -a")
  println(s"stdout:    ${r.stdout}")
  println(s"stderr:    ${r.stderr}")
  println(s"exit code: ${r.exitCode}")
end main
```

Returning a `case class` with typed fields is cleaner than returning a  
raw tuple or string: callers can access `.stdout`, `.stderr`, and  
`.exitCode` by name. `Option(cmd.getExitStatus).map(_.intValue).getOrElse(-1)`  
safely unboxes the nullable Java `Integer` and returns `-1` when the  
server does not send an exit status — which happens if the SSH channel  
was closed without a normal process exit. This utility is the foundation  
for deployment scripts, health check runners, and configuration management  
tools built in Scala.  

## SFTP sync tool

Upload all files from a local directory to a remote directory, creating  
the target path if needed.  

```scala
import net.schmizz.sshj.SSHClient
import net.schmizz.sshj.transport.verification.PromiscuousVerifier
import net.schmizz.sshj.xfer.FileSystemFile
import java.nio.file.{Files, Paths}

@main def main() =
  val localDir  = Paths.get("/tmp/sync-source")
  val remoteDir = "/home/alice/sync-target"
  Files.createDirectories(localDir)
  Seq("a.txt", "b.txt", "c.txt").foreach { name =>
    Files.writeString(localDir.resolve(name), s"Content: $name")
  }
  val ssh = new SSHClient()
  ssh.addHostKeyVerifier(new PromiscuousVerifier())
  ssh.connect("192.168.1.100")
  try
    ssh.authPassword("alice", "s3cret")
    val sftp = ssh.newSFTPClient()
    try
      sftp.mkdirs(remoteDir)
      Files.list(localDir)
        .filter(Files.isRegularFile(_))
        .forEach { local =>
          val remote =
            s"$remoteDir/${local.getFileName}"
          sftp.put(
            new FileSystemFile(local.toFile), remote)
          println(s"Synced: ${local.getFileName}")
        }
    finally
      sftp.close()
  finally
    ssh.disconnect()
end main
```

This one-way sync iterates over regular files in the local directory  
using `Files.list` and uploads each one via `sftp.put`. Calling  
`sftp.mkdirs(remoteDir)` first ensures the destination directory tree  
exists without a separate check. A production-grade sync tool would  
compare file sizes or checksums before uploading and skip files that are  
already identical on the remote side, reducing unnecessary network  
transfers. It would also handle subdirectories recursively and log each  
operation to a structured log.  

## Certificate expiration checker

Check the TLS certificate expiration dates for a list of hostnames and  
warn about certificates expiring within 30 days.  

```scala
import java.net.URL
import javax.net.ssl.HttpsURLConnection
import java.security.cert.X509Certificate
import java.time.ZoneId
import java.time.format.DateTimeFormatter
import java.util.Date

def checkExpiry(host: String): Unit =
  val conn = new URL(s"https://$host")
    .openConnection().asInstanceOf[HttpsURLConnection]
  conn.setConnectTimeout(5000)
  conn.setReadTimeout(5000)
  conn.connect()
  try
    val now = new Date()
    val fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd")
      .withZone(ZoneId.systemDefault())
    conn.getServerCertificates.headOption.foreach { cert =>
      val x509     = cert.asInstanceOf[X509Certificate]
      val expiry   = x509.getNotAfter
      val daysLeft =
        (expiry.getTime - now.getTime) / 86_400_000L
      val tag  = if daysLeft < 30 then " *** EXPIRING SOON"
                 else " OK"
      val date = fmt.format(expiry.toInstant)
      println(f"$host%-30s $date  $daysLeft%4d days$tag")
    }
  finally
    conn.disconnect()

@main def main() =
  val hosts = List(
    "www.scala-lang.org",
    "www.github.com",
    "www.google.com"
  )
  println(
    f"${"HOST"}%-30s ${"EXPIRY"}%-10s  ${"DAYS"}%4s")
  println("-" * 60)
  hosts.foreach(checkExpiry)
end main
```

Extracting the logic into a standalone `checkExpiry` function keeps the  
`main` function focused on orchestration. Using `headOption` on the  
certificate array retrieves only the leaf (end-entity) certificate, which  
is the one with the shortest validity period and the one your monitoring  
should track. The formatted tabular output makes it easy to scan for  
problems at a glance. In a production monitoring system, you would run  
this check daily via a scheduler and emit metrics or alerts when the  
threshold is crossed.  

## HTTP latency measurement

Measure the round-trip latency of HTTP requests to multiple URLs and  
report the results in a table.  

```scala
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.net.URI
import java.time.{Duration, Instant}

case class LatencyResult(
  url: String, status: Int, ms: Long, bytes: Int)

def measure(client: HttpClient, url: String): LatencyResult =
  val request = HttpRequest.newBuilder(URI.create(url))
    .timeout(Duration.ofSeconds(15))
    .GET()
    .build()
  val t0       = Instant.now()
  val response = client.send(
    request, HttpResponse.BodyHandlers.ofString())
  val elapsed  =
    Duration.between(t0, Instant.now()).toMillis
  LatencyResult(
    url, response.statusCode(),
    elapsed, response.body().length)

@main def main() =
  val client = HttpClient.newBuilder()
    .followRedirects(HttpClient.Redirect.NORMAL)
    .build()
  val urls = List(
    "https://www.scala-lang.org",
    "https://www.github.com",
    "https://api.github.com"
  )
  val results = urls.map(measure(client, _))
  println(
    f"${"URL"}%-35s ${"ST"}%4s ${"ms"}%6s ${"bytes"}%8s")
  println("-" * 58)
  results.foreach { r =>
    println(
      f"${r.url}%-35s ${r.status}%4d ${r.ms}%6d ${r.bytes}%8d")
  }
end main
```

Sharing a single `HttpClient` across all measurements reuses the  
underlying connection pool, which means the first request to each host  
pays the TCP and TLS setup cost while subsequent requests to the same  
host may reuse the existing connection. This matches real-world  
application behaviour more accurately than creating a new client per  
measurement. The `Instant`-based timing wraps the synchronous `send`  
call and measures wall-clock time including DNS, TCP, TLS, and HTTP  
processing, giving end-to-end latency from the client's perspective.  

## HTTPS cipher suite inspection

Connect to a server with a raw `SSLSocket` and print the negotiated  
protocol, cipher suite, and available configuration.  

```scala
import javax.net.ssl.{SSLSocket, SSLParameters,
  SSLSocketFactory}

@main def main() =
  val factory = SSLSocketFactory.getDefault
    .asInstanceOf[SSLSocketFactory]
  val socket = factory.createSocket("www.google.com", 443)
    .asInstanceOf[SSLSocket]
  val params = new SSLParameters()
  params.setEndpointIdentificationAlgorithm("HTTPS")
  socket.setSSLParameters(params)
  try
    socket.startHandshake()
    val session = socket.getSession
    println(s"Host:         ${session.getPeerHost}")
    println(s"Protocol:     ${session.getProtocol}")
    println(s"Cipher suite: ${session.getCipherSuite}")
    println(s"Peer certs:   " +
      s"${session.getPeerCertificates.length}")
    println()
    println("Enabled cipher suites (first 8):")
    socket.getEnabledCipherSuites
      .take(8).foreach(cs => println(s"  $cs"))
    println()
    println("Supported protocols:")
    socket.getSupportedProtocols
      .foreach(p => println(s"  $p"))
  finally
    socket.close()
end main
```

A raw `SSLSocket` gives you lower-level access to TLS internals than the  
`HttpClient` API. After the handshake completes, `getSession` exposes  
the negotiated protocol version and cipher suite that both sides agreed  
on. `getEnabledCipherSuites` lists all suites the JVM is willing to use;  
a server that only supports weak suites will negotiate one of them  
unless you call `setEnabledCipherSuites` to restrict the list. This  
technique is useful for security audits, compliance checks, and  
debugging TLS configuration mismatches between a Scala client and a  
server that enforces specific cipher policies.  
