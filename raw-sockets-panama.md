# Raw Sockets and Project Panama in Scala 3  

Raw sockets let a program inspect and craft packets below normal socket APIs.  
Unlike TCP or UDP sockets, they expose link-layer or IP-layer details directly.  
A regular TCP socket gives you a byte stream after the kernel builds packets.  
A raw socket can show the packet headers that the kernel usually hides.  
That makes it useful for learning how network protocols fit together.  
It also makes mistakes easier, because every field now matters.  

On Linux, AF_PACKET raw sockets operate at the Ethernet frame layer.  
They can send or receive complete frames, including MAC addresses and types.  
This is different from AF_INET raw sockets, which start at the IP header.  
Examples in this document focus on Linux-oriented packet work.  
Many snippets are conceptual, because real packet injection needs privileges.  
Root access or specific capabilities are commonly required.  

A raw frame often starts with an Ethernet header.  
That header is followed by an IP packet when the EtherType says IPv4 or IPv6.  
Inside the IP packet, the protocol field points to TCP, UDP, or ICMP.  
Reading packets means peeling these layers one header at a time.  
Crafting packets means writing those layers in the correct byte order.  
Even small errors in lengths or checksums can invalidate a packet.  

Project Panama gives Scala access to native networking functions through Java.  
The Foreign Function and Memory API can look up symbols such as socket.  
It can then create downcall handles for bind, sendto, recvfrom, or close.  
This is useful when the JVM does not expose a low-level system call directly.  
Panama also helps keep native calls explicit and structured.  
The code stays in Scala while still reaching libc and kernel-facing APIs.  

MemorySegment represents off-heap memory with bounds and lifetime tracking.  
MemoryLayout describes how bytes are arranged inside native structures.  
Together they let you model headers like Ethernet, IPv4, or epoll_event.  
A layout can also produce VarHandles for named fields such as ttl.  
This makes packet parsing clearer than ad hoc pointer arithmetic.  
It also makes intent visible when a buffer mirrors a C struct.  

Safety matters when you craft packets by hand.  
You must respect network byte order, legal field sizes, and checksum rules.  
You should also avoid sending arbitrary traffic on networks you do not own.  
Raw sockets are best used for diagnostics, research, testing, and education.  
They help explain what protocol analyzers display and what kernels transmit.  
They also teach how abstractions like TCP rely on lower packet layers.  

---

## Opening a raw socket  
This example shows the constants used to open an AF_PACKET raw socket.  

```scala
@main def main() =
  val afPacket = 17
  val sockRaw = 3
  val ethPAll = 0x0003
  val isLinux = System.getProperty("os.name").toLowerCase.contains("linux")

  val fd =
    if isLinux then
      // A real program would call socket(AF_PACKET, SOCK_RAW, ethPAll)
      -1
    else
      -1

  println(s"AF_PACKET = $afPacket")
  println(s"SOCK_RAW = $sockRaw")
  println(s"ETH_P_ALL = 0x${ethPAll.toHexString}")
  println(s"conceptual fd = $fd")
end main
``` 

AF_PACKET works at the Linux link layer instead of the TCP or UDP layer.  
The snippet prints the constant values and shows the real call as a comment.  

## Binding to a network interface  
This example builds a conceptual sockaddr_ll buffer for the eth0 interface.  

```scala
import java.nio.ByteBuffer
import java.nio.ByteOrder

@main def main() =
  val protocol = 0x0800.toShort
  val ifindex = 2 // pretend this is eth0
  val sockaddrLl = ByteBuffer.allocate(20).order(ByteOrder.nativeOrder())

  sockaddrLl.putShort(17.toShort)  // sll_family = AF_PACKET
  sockaddrLl.putShort(protocol)    // sll_protocol = ETH_P_IP
  sockaddrLl.putInt(ifindex)       // sll_ifindex
  sockaddrLl.putShort(0.toShort)   // sll_hatype
  sockaddrLl.put(0.toByte)         // sll_pkttype
  sockaddrLl.put(6.toByte)         // sll_halen
  sockaddrLl.put(Array.fill[Byte](8)(0))

  println(s"sll_family = ${sockaddrLl.getShort(0)}")
  println(f"sll_protocol = 0x${sockaddrLl.getShort(2) & 0xffff}%04x")
  println(s"sll_ifindex = ${sockaddrLl.getInt(4)}")
end main
``` 

A bind call would attach the packet socket to the interface index for eth0.  
That limits capture or transmission to one link-layer device.  

## Setting promiscuous mode  
This example builds a packet_mreq structure for PACKET_ADD_MEMBERSHIP.  

```scala
import java.nio.ByteBuffer
import java.nio.ByteOrder

@main def main() =
  val packetAddMembership = 1
  val packetMrPromisc = 1
  val ifindex = 2
  val mreq = ByteBuffer.allocate(16).order(ByteOrder.nativeOrder())

  mreq.putInt(ifindex)
  mreq.putShort(packetMrPromisc.toShort)
  mreq.putShort(0.toShort)
  mreq.put(Array.fill[Byte](8)(0))

  println(s"opt = $packetAddMembership")
  println(s"mr_ifindex = ${mreq.getInt(0)}")
  println(s"mr_type = ${mreq.getShort(4)}")
end main
``` 

Promiscuous mode asks the interface to deliver frames not addressed to it.  
The buffer mirrors packet_mreq fields that setsockopt would consume.  

## Reading raw Ethernet frames  
This example simulates receiving a frame into a 2048-byte buffer.  

```scala
import java.nio.ByteBuffer

@main def main() =
  val frameBuffer = ByteBuffer.allocate(2048)
  val fakeFrame = Array.tabulate[Byte](60)(i => i.toByte)

  frameBuffer.put(fakeFrame)
  val bytesRead = fakeFrame.length

  println(s"bytesRead = $bytesRead")
  println(s"buffer position = ${frameBuffer.position()}")
end main
``` 

Real code would pass the buffer to recvfrom or a channel-like wrapper.  
The example keeps the focus on buffer sizing and the reported frame length.  

## Writing raw Ethernet frames  
This example builds a 14-byte Ethernet header and prints it as hex.  

```scala
import java.nio.ByteBuffer

@main def main() =
  def hex(data: Array[Byte]): String =
    data.map(b => f"${b & 0xff}%02x").mkString(" ")

  val dstMac = Array[Byte](0, 17, 34, 51, 68, 85)
  val srcMac = Array[Byte](102, 119, -120, -103, -86, -69)
  val header = ByteBuffer.allocate(14)

  header.put(dstMac)
  header.put(srcMac)
  header.putShort(0x0800.toShort)

  println(hex(header.array()))
end main
``` 

The first six bytes are the destination MAC and the next six are the source.  
The final two bytes hold the EtherType for IPv4 in network byte order.  

## Filtering frames by EtherType  
This example reads bytes 12 and 13 to identify the Ethernet payload type.  

```scala
@main def main() =
  val frame = Array.fill[Byte](64)(0)
  frame(12) = 0x08
  frame(13) = 0x00

  val etherType = ((frame(12) & 0xff) << 8) | (frame(13) & 0xff)
  val protocol = etherType match
    case 0x0800 => "IPv4"
    case 0x0806 => "ARP"
    case 0x86dd => "IPv6"
    case other  => f"Unknown 0x$other%04x"

  println(s"EtherType = $protocol")
end main
``` 

EtherType is the link-layer dispatch field for the next protocol.  
Reading it early lets a sniffer send each frame to the right parser.  

## Parsing Ethernet header fields  
This example extracts MAC addresses and the EtherType from a raw frame.  

```scala
@main def main() =
  def formatMac(bytes: Array[Byte]): String =
    bytes.map(b => f"${b & 0xff}%02x").mkString(":")

  val frame = Array[Byte](
    0, 17, 34, 51, 68, 85,
    102, 119, -120, -103, -86, -69,
    0x08, 0x00, 1, 2
  )

  val dstMac = frame.slice(0, 6)
  val srcMac = frame.slice(6, 12)
  val etherType = ((frame(12) & 0xff) << 8) | (frame(13) & 0xff)

  println(s"dstMac = ${formatMac(dstMac)}")
  println(s"srcMac = ${formatMac(srcMac)}")
  println(f"etherType = 0x$etherType%04x")
end main
``` 

Ethernet parsing usually starts with fixed offsets because the header is flat.  
Formatting MAC bytes makes captured frames easier to inspect by eye.  

## Computing Ethernet CRC (conceptual)  
This example uses CRC32 to illustrate the idea behind Ethernet frame checks.  

```scala
import java.util.zip.CRC32

@main def main() =
  val frame = Array[Byte](1, 2, 3, 4, 5, 6, 7, 8)
  val crc = CRC32()
  crc.update(frame)

  println(f"crc32 = 0x${crc.getValue}%08x")
end main
``` 

Ethernet hardware usually appends the frame check sequence automatically.  
CRC32 here is educational, because it is not a full replacement for NIC FCS.  


## Crafting an IPv4 header  
This example writes a basic 20-byte IPv4 header into a byte buffer.  

```scala
import java.nio.ByteBuffer
import java.nio.ByteOrder

@main def main() =
  val header = ByteBuffer.allocate(20).order(ByteOrder.BIG_ENDIAN)
  val srcIp = Array[Byte](192.toByte, 168.toByte, 1, 10)
  val dstIp = Array[Byte](192.toByte, 168.toByte, 1, 20)

  header.put(0x45.toByte)
  header.put(0.toByte)
  header.putShort(28.toShort)
  header.putShort(0x1234.toShort)
  header.putShort(0x4000.toShort)
  header.put(64.toByte)
  header.put(17.toByte)
  header.putShort(0.toShort)
  header.put(srcIp)
  header.put(dstIp)

  println(header.array().map(b => f"${b & 0xff}%02x").mkString(" "))
end main
``` 

Version and IHL share the first byte, so 0x45 means IPv4 with a 20-byte header.  
The checksum field stays zero until a checksum routine fills it in.  

## Computing IPv4 header checksum  
This example computes the RFC 791 one's-complement checksum for IPv4.  

```scala
@main def main() =
  def ipv4Checksum(data: Array[Byte]): Int =
    val padded = if data.length % 2 == 0 then data else data :+ 0.toByte
    var sum = 0

    for i <- 0 until padded.length by 2 do
      val word = ((padded(i) & 0xff) << 8) | (padded(i + 1) & 0xff)
      sum += word
      while sum > 0xffff do
        sum = (sum & 0xffff) + (sum >>> 16)

    (~sum) & 0xffff

  val header = Array[Byte](
    0x45, 0, 0, 28, 0x12, 0x34, 0x40, 0,
    64, 17, 0, 0,
    192.toByte, 168.toByte, 1, 10,
    192.toByte, 168.toByte, 1, 20
  )

  println(f"checksum = 0x${ipv4Checksum(header)}%04x")
end main
``` 

IPv4 header checksum covers only the header, not the payload.  
Routers may update TTL, so they also update this checksum in transit.  

## Crafting an IPv4 packet with payload  
This example combines a 20-byte IPv4 header with an 8-byte payload.  

```scala
import java.nio.ByteBuffer
import java.nio.ByteOrder

@main def main() =
  val payload = Array[Byte](1, 2, 3, 4, 5, 6, 7, 8)
  val packet = ByteBuffer.allocate(20 + payload.length).order(ByteOrder.BIG_ENDIAN)

  packet.put(0x45.toByte)
  packet.put(0.toByte)
  packet.putShort((20 + payload.length).toShort)
  packet.putShort(1.toShort)
  packet.putShort(0x4000.toShort)
  packet.put(64.toByte)
  packet.put(17.toByte)
  packet.putShort(0.toShort)
  packet.put(Array[Byte](10, 0, 0, 1))
  packet.put(Array[Byte](10, 0, 0, 2))
  packet.put(payload)

  println(s"total length = ${packet.array().length}")
end main
``` 

The total length field must match the full IPv4 packet size in bytes.  
Placing header and payload together makes later checksum work easier.  

## Parsing IPv4 header fields  
This example extracts common IPv4 fields from a raw byte array.  

```scala
@main def main() =
  def formatIp(bytes: Array[Byte]): String =
    bytes.map(_ & 0xff).mkString(".")

  val packet = Array[Byte](
    0x45, 0, 0, 28, 0x12, 0x34, 0x40, 0,
    64, 17, 0, 0,
    192.toByte, 168.toByte, 1, 10,
    192.toByte, 168.toByte, 1, 20
  )

  val version = (packet(0) >>> 4) & 0x0f
  val ihl = packet(0) & 0x0f
  val ttl = packet(8) & 0xff
  val protocol = packet(9) & 0xff
  val srcIp = formatIp(packet.slice(12, 16))
  val dstIp = formatIp(packet.slice(16, 20))

  println(s"version = $version")
  println(s"ihl = $ihl")
  println(s"ttl = $ttl")
  println(s"protocol = $protocol")
  println(s"src = $srcIp")
  println(s"dst = $dstIp")
end main
``` 

IPv4 has fixed offsets for these base header fields when no options are used.  
The IHL value tells you where payload parsing should begin.  

## Crafting an IPv6 header (conceptual)  
This example builds a 40-byte IPv6 header with source and destination data.  

```scala
import java.nio.ByteBuffer
import java.nio.ByteOrder

@main def main() =
  val src = Array.fill[Byte](16)(0)
  val dst = Array.fill[Byte](16)(0)
  src(15) = 1
  dst(15) = 2

  val header = ByteBuffer.allocate(40).order(ByteOrder.BIG_ENDIAN)
  header.putInt(0x60000000)
  header.putShort(8.toShort)
  header.put(17.toByte)
  header.put(64.toByte)
  header.put(src)
  header.put(dst)

  println(s"ipv6 header bytes = ${header.array().length}")
end main
``` 

IPv6 keeps a fixed 40-byte base header and moves optional data to extensions.  
The first word packs the version, traffic class, and flow label.  

## Parsing IPv6 header fields  
This example extracts the next header, hop limit, and two IPv6 addresses.  

```scala
@main def main() =
  def formatIpv6(bytes: Array[Byte]): String =
    bytes.grouped(2)
      .map(group => f"${((group(0) & 0xff) << 8) | (group(1) & 0xff)}%x")
      .mkString(":")

  val header = Array.fill[Byte](40)(0)
  header(0) = 0x60
  header(6) = 17
  header(7) = 64
  header(23) = 1
  header(39) = 2

  val nextHeader = header(6) & 0xff
  val hopLimit = header(7) & 0xff
  val src = formatIpv6(header.slice(8, 24))
  val dst = formatIpv6(header.slice(24, 40))

  println(s"nextHeader = $nextHeader")
  println(s"hopLimit = $hopLimit")
  println(s"src = $src")
  println(s"dst = $dst")
end main
``` 

IPv6 addresses occupy 16 bytes each, so formatting helps readability quickly.  
The next header field is the IPv6 counterpart of the IPv4 protocol field.  

## Handling fragmentation fields (conceptual)  
This example sets and reads the IPv4 flags and fragment offset bits.  

```scala
@main def main() =
  val header = Array.fill[Byte](20)(0)
  val df = 0x4000
  val mf = 0x2000
  val offset = 0x0010
  val flagsAndOffset = df | mf | offset

  header(6) = ((flagsAndOffset >>> 8) & 0xff).toByte
  header(7) = (flagsAndOffset & 0xff).toByte

  val parsed = ((header(6) & 0xff) << 8) | (header(7) & 0xff)
  val parsedDf = (parsed & 0x4000) != 0
  val parsedMf = (parsed & 0x2000) != 0
  val parsedOffset = parsed & 0x1fff

  println(s"DF = $parsedDf")
  println(s"MF = $parsedMf")
  println(s"offset = $parsedOffset")
end main
``` 

The top three bits hold flags and the lower thirteen bits hold the offset.  
These values matter only when a packet is large enough to fragment.  

## IPv4 TTL and protocol field extraction  
This example pretty-prints the IPv4 TTL and protocol name fields.  

```scala
@main def main() =
  val header = Array[Byte](
    0x45, 0, 0, 28, 0, 1, 0x40, 0,
    32, 6, 0, 0,
    10, 0, 0, 1,
    10, 0, 0, 2
  )

  val ttl = header(8) & 0xff
  val protocolNumber = header(9) & 0xff
  val protocolName = protocolNumber match
    case 6  => "TCP"
    case 17 => "UDP"
    case 1  => "ICMP"
    case _  => "Other"

  println(s"TTL = $ttl")
  println(s"protocol = $protocolName")
end main
``` 

TTL limits how many routed hops a packet can cross before expiry.  
The protocol field decides which transport or control parser runs next.  


## Crafting a UDP header  
This example writes the four fields of an 8-byte UDP header.  

```scala
import java.nio.ByteBuffer
import java.nio.ByteOrder

@main def main() =
  val udp = ByteBuffer.allocate(8).order(ByteOrder.BIG_ENDIAN)
  udp.putShort(50000.toShort)
  udp.putShort(53.toShort)
  udp.putShort(8.toShort)
  udp.putShort(0.toShort)

  println(udp.array().map(b => f"${b & 0xff}%02x").mkString(" "))
end main
``` 

UDP keeps its header small, so payload parsing starts immediately after byte 8.  
The checksum may be zero in IPv4, but many tools still compute it.  

## Computing UDP checksum  
This example computes the UDP checksum with a pseudo-header and payload.  

```scala
@main def main() =
  def internetChecksum(data: Array[Byte]): Int =
    val padded = if data.length % 2 == 0 then data else data :+ 0.toByte
    var sum = 0

    for i <- 0 until padded.length by 2 do
      val word = ((padded(i) & 0xff) << 8) | (padded(i + 1) & 0xff)
      sum += word
      while sum > 0xffff do
        sum = (sum & 0xffff) + (sum >>> 16)

    (~sum) & 0xffff

  val srcIp = Array[Byte](10, 0, 0, 1)
  val dstIp = Array[Byte](10, 0, 0, 2)
  val udp = Array[Byte](0x13, -120, 0, 53, 0, 13, 0, 0, 72, 101, 108, 108, 111)
  val pseudo = srcIp ++ dstIp ++ Array[Byte](0, 17, 0, udp.length.toByte)

  println(f"udp checksum = 0x${internetChecksum(pseudo ++ udp)}%04x")
end main
``` 

UDP checksum covers a pseudo-header so that source and destination matter too.  
That extra context helps detect misdelivery as well as payload corruption.  

## Building a full UDP packet (IP + UDP + payload)  
This example combines an IPv4 header, UDP header, and Hello payload.  

```scala
@main def main() =
  def hex(data: Array[Byte], n: Int): String =
    data.take(n).map(b => f"${b & 0xff}%02x").mkString(" ")

  val ip = Array[Byte](
    0x45, 0, 0, 33, 0, 1, 0x40, 0,
    64, 17, 0, 0,
    10, 0, 0, 1,
    10, 0, 0, 2
  )
  val udp = Array[Byte](0x13, -120, 0, 53, 0, 13, 0, 0)
  val payload = "Hello".getBytes("UTF-8")
  val packet = ip ++ udp ++ payload

  println(s"total size = ${packet.length}")
  println(hex(packet, 30))
end main
``` 

Joining the three pieces into one array makes raw transmission straightforward.  
The first 20 bytes are IPv4 and the next 8 bytes are UDP.  

## Parsing UDP header fields  
This example reads a UDP segment that starts at byte 20 of an IP packet.  

```scala
@main def main() =
  val packet = Array[Byte](
    0x45, 0, 0, 33, 0, 1, 0x40, 0,
    64, 17, 0, 0,
    10, 0, 0, 1,
    10, 0, 0, 2,
    0x13, -120, 0, 53, 0, 13, 0x12, 0x34
  )

  val base = 20
  val srcPort = ((packet(base) & 0xff) << 8) | (packet(base + 1) & 0xff)
  val dstPort = ((packet(base + 2) & 0xff) << 8) | (packet(base + 3) & 0xff)
  val length = ((packet(base + 4) & 0xff) << 8) | (packet(base + 5) & 0xff)
  val checksum = ((packet(base + 6) & 0xff) << 8) | (packet(base + 7) & 0xff)

  println(s"srcPort = $srcPort")
  println(s"dstPort = $dstPort")
  println(s"length = $length")
  println(f"checksum = 0x$checksum%04x")
end main
``` 

Offsets are fixed for the UDP header, so extraction is simple and fast.  
You still need the IP header length to know where UDP begins.  

## Sending a UDP packet via raw socket  
This example outlines the sendto flow for a packet socket transmission.  

```scala
@main def main() =
  val afPacket = 17
  val sockRaw = 3
  val ifindex = 2
  val packet = Array[Byte](1, 2, 3, 4, 5)

  println(s"socket($afPacket, $sockRaw, 0x0800)")
  println(s"prepare sockaddr_ll for ifindex $ifindex")
  // A real program would call sendto with root or CAP_NET_RAW privileges.
  println(s"would send ${packet.length} bytes")
end main
``` 

Raw transmission needs a link-layer destination as well as packet bytes.  
The privileged syscall is omitted so the example stays safe and portable.  

## Validating UDP checksum on receive  
This example recomputes the checksum and reports whether the datagram is valid.  

```scala
@main def main() =
  def internetChecksum(data: Array[Byte]): Int =
    val padded = if data.length % 2 == 0 then data else data :+ 0.toByte
    var sum = 0

    for i <- 0 until padded.length by 2 do
      val word = ((padded(i) & 0xff) << 8) | (padded(i + 1) & 0xff)
      sum += word
      while sum > 0xffff do
        sum = (sum & 0xffff) + (sum >>> 16)

    (~sum) & 0xffff

  val srcIp = Array[Byte](10, 0, 0, 1)
  val dstIp = Array[Byte](10, 0, 0, 2)
  val udp = Array[Byte](0x13, -120, 0, 53, 0, 13, 0, 0, 72, 101, 108, 108, 111)
  val pseudo = srcIp ++ dstIp ++ Array[Byte](0, 17, 0, udp.length.toByte)
  val expected = internetChecksum(pseudo ++ udp)

  println(if expected == 0 then "valid" else "invalid")
end main
``` 

Receivers verify the same pseudo-header fields that senders used to compute it.  
A correct result folds to zero when the checksum is included in the segment.  


## Crafting a TCP header  
This example writes a minimal 20-byte TCP header into a byte buffer.  

```scala
import java.nio.ByteBuffer
import java.nio.ByteOrder

@main def main() =
  val tcp = ByteBuffer.allocate(20).order(ByteOrder.BIG_ENDIAN)
  tcp.putShort(50000.toShort)
  tcp.putShort(80.toShort)
  tcp.putInt(1)
  tcp.putInt(0)
  tcp.put((5 << 4).toByte)
  tcp.put(0x02.toByte)
  tcp.putShort(65535.toShort)
  tcp.putShort(0.toShort)
  tcp.putShort(0.toShort)

  println(tcp.array().map(b => f"${b & 0xff}%02x").mkString(" "))
end main
``` 

The data offset says the TCP header is 20 bytes with no options.  
Flags, window size, checksum, and urgent pointer follow the sequence fields.  

## Computing TCP pseudo-header checksum  
This example computes a TCP checksum across pseudo-header and segment bytes.  

```scala
@main def main() =
  def internetChecksum(data: Array[Byte]): Int =
    val padded = if data.length % 2 == 0 then data else data :+ 0.toByte
    var sum = 0

    for i <- 0 until padded.length by 2 do
      val word = ((padded(i) & 0xff) << 8) | (padded(i + 1) & 0xff)
      sum += word
      while sum > 0xffff do
        sum = (sum & 0xffff) + (sum >>> 16)

    (~sum) & 0xffff

  val srcIp = Array[Byte](10, 0, 0, 1)
  val dstIp = Array[Byte](10, 0, 0, 2)
  val tcp = Array.fill[Byte](20)(0)
  tcp(13) = 0x02
  val pseudo = srcIp ++ dstIp ++ Array[Byte](0, 6, 0, tcp.length.toByte)

  println(f"tcp checksum = 0x${internetChecksum(pseudo ++ tcp)}%04x")
end main
``` 

TCP uses the same Internet checksum style as UDP and ICMP.  
Including IP addresses helps catch misrouted or corrupted segments.  

## Crafting a TCP SYN packet  
This example builds a SYN segment and wraps it in a simple IPv4 packet.  

```scala
import scala.util.Random

@main def main() =
  val seqNum = Random.nextInt(Int.MaxValue)
  val tcp = Array.fill[Byte](20)(0)
  tcp(0) = 0x13
  tcp(1) = -120
  tcp(2) = 0
  tcp(3) = 80
  tcp(12) = (5 << 4).toByte
  tcp(13) = 0x02

  val ip = Array[Byte](
    0x45, 0, 0, 40, 0, 1, 0x40, 0,
    64, 6, 0, 0,
    10, 0, 0, 1,
    10, 0, 0, 2
  )
  val packet = ip ++ tcp

  println(s"seqNum = $seqNum")
  println(s"packet size = ${packet.length}")
end main
``` 

A SYN starts a TCP handshake by advertising an initial sequence number.  
The packet shown here is minimal and omits checksum and option handling.  

## Crafting a TCP RST packet  
This example sets the RST flag to model an abrupt TCP reset packet.  

```scala
@main def main() =
  val tcp = Array.fill[Byte](20)(0)
  tcp(12) = (5 << 4).toByte
  tcp(13) = 0x04

  println(f"flags = 0x${tcp(13) & 0xff}%02x")
  println("RST is used to reject or abort a connection immediately")
end main
``` 

RST tells the peer that the current connection state is invalid or unwanted.  
Unlike FIN, it does not perform an orderly half-close sequence.  

## Parsing TCP header fields  
This example extracts core TCP fields and decodes the flag bits.  

```scala
@main def main() =
  def decodeFlags(flags: Int): String =
    val parts = List(
      "FIN" -> 0x01,
      "SYN" -> 0x02,
      "RST" -> 0x04,
      "PSH" -> 0x08,
      "ACK" -> 0x10,
      "URG" -> 0x20
    ).collect:
      case (name, bit) if (flags & bit) != 0 => name
    parts.mkString(",")

  val tcp = Array[Byte](
    0x13, -120, 0, 80,
    0, 0, 0, 10,
    0, 0, 0, 20,
    0x50, 0x12, 0x20, 0,
    0x12, 0x34, 0, 0
  )

  val srcPort = ((tcp(0) & 0xff) << 8) | (tcp(1) & 0xff)
  val dstPort = ((tcp(2) & 0xff) << 8) | (tcp(3) & 0xff)
  val seqNum = java.nio.ByteBuffer.wrap(tcp.slice(4, 8)).getInt
  val ackNum = java.nio.ByteBuffer.wrap(tcp.slice(8, 12)).getInt
  val flags = tcp(13) & 0xff
  val windowSize = ((tcp(14) & 0xff) << 8) | (tcp(15) & 0xff)

  println(s"srcPort = $srcPort")
  println(s"dstPort = $dstPort")
  println(s"seqNum = $seqNum")
  println(s"ackNum = $ackNum")
  println(s"flags = ${decodeFlags(flags)}")
  println(s"windowSize = $windowSize")
end main
``` 

TCP parsing often begins with ports, sequence numbers, and the flag byte.  
The decoded flags explain whether the segment carries control semantics.  

## Conceptual 3-way handshake  
This example prints the SYN, SYN-ACK, and ACK exchange at byte level.  

```scala
@main def main() =
  val syn = 0x02
  val synAck = 0x12
  val ack = 0x10

  println(f"client -> server: SYN     flags=0x$syn%02x")
  println(f"server -> client: SYN-ACK flags=0x$synAck%02x")
  println(f"client -> server: ACK     flags=0x$ack%02x")
end main
``` 

The handshake synchronizes both sides before application data can flow.  
Each step is just a normal TCP segment with different flag combinations.  

## TCP flag bitmask operations  
This example demonstrates setting, testing, and clearing TCP flag bits.  

```scala
@main def main() =
  val fin = 0x01
  val syn = 0x02
  val rst = 0x04
  val psh = 0x08
  val ack = 0x10
  val urg = 0x20

  var flags = syn | ack
  println(s"has SYN = ${(flags & syn) != 0}")
  flags |= psh
  println(f"after set PSH = 0x$flags%02x")
  flags &= ~ack
  println(f"after clear ACK = 0x$flags%02x")
  println(f"all constants = 0x${fin | syn | rst | psh | ack | urg}%02x")
end main
``` 

Bitmasks make it cheap to combine multiple control flags in one byte.  
The same pattern appears when decoding capture output from sniffers.  


## Crafting ICMP echo request  
This example builds an ICMP echo request with a 56-byte payload.  

```scala
@main def main() =
  def icmpChecksum(data: Array[Byte]): Int =
    val padded = if data.length % 2 == 0 then data else data :+ 0.toByte
    var sum = 0

    for i <- 0 until padded.length by 2 do
      val word = ((padded(i) & 0xff) << 8) | (padded(i + 1) & 0xff)
      sum += word
      while sum > 0xffff do
        sum = (sum & 0xffff) + (sum >>> 16)

    (~sum) & 0xffff

  val payload = Array.fill[Byte](56)('A'.toByte)
  val packet = Array.fill[Byte](8 + payload.length)(0)
  packet(0) = 8
  packet(1) = 0
  packet(4) = 0x12
  packet(5) = 0x34
  packet(6) = 0
  packet(7) = 1
  Array.copy(payload, 0, packet, 8, payload.length)
  val checksum = icmpChecksum(packet)
  packet(2) = ((checksum >>> 8) & 0xff).toByte
  packet(3) = (checksum & 0xff).toByte

  println(s"icmp bytes = ${packet.length}")
end main
``` 

Echo request is the packet type behind the classic ping utility.  
The checksum covers both the header and the payload bytes.  

## Crafting ICMP echo reply  
This example builds an echo reply by copying identifier and sequence values.  

```scala
@main def main() =
  def icmpChecksum(data: Array[Byte]): Int =
    val padded = if data.length % 2 == 0 then data else data :+ 0.toByte
    var sum = 0

    for i <- 0 until padded.length by 2 do
      val word = ((padded(i) & 0xff) << 8) | (padded(i + 1) & 0xff)
      sum += word
      while sum > 0xffff do
        sum = (sum & 0xffff) + (sum >>> 16)

    (~sum) & 0xffff

  val request = Array.fill[Byte](64)(0)
  request(0) = 8
  request(4) = 0x12
  request(5) = 0x34
  request(6) = 0
  request(7) = 1

  val reply = request.clone()
  reply(0) = 0
  reply(1) = 0
  reply(2) = 0
  reply(3) = 0
  val checksum = icmpChecksum(reply)
  reply(2) = ((checksum >>> 8) & 0xff).toByte
  reply(3) = (checksum & 0xff).toByte

  println(f"reply checksum = 0x$checksum%04x")
end main
``` 

An echo reply mirrors the request so the sender can match the response easily.  
Only the type and checksum must change for this simple transformation.  

## Computing ICMP checksum  
This example defines a reusable ICMP checksum function and verifies two packets.  

```scala
@main def main() =
  def icmpChecksum(data: Array[Byte]): Int =
    val padded = if data.length % 2 == 0 then data else data :+ 0.toByte
    var sum = 0

    for i <- 0 until padded.length by 2 do
      val word = ((padded(i) & 0xff) << 8) | (padded(i + 1) & 0xff)
      sum += word
      while sum > 0xffff do
        sum = (sum & 0xffff) + (sum >>> 16)

    (~sum) & 0xffff

  val request = Array[Byte](8, 0, 0, 0, 0x12, 0x34, 0, 1)
  val reply = Array[Byte](0, 0, 0, 0, 0x12, 0x34, 0, 1)

  println(f"request = 0x${icmpChecksum(request)}%04x")
  println(f"reply   = 0x${icmpChecksum(reply)}%04x")
end main
``` 

ICMP reuses the same one's-complement checksum style used in other IP tools.  
A standalone helper keeps packet-building examples shorter and easier to read.  

## Parsing ICMP header fields  
This example extracts the ICMP type, code, checksum, id, and sequence fields.  

```scala
@main def main() =
  val packet = Array[Byte](8, 0, 0x1a, 0x2b, 0x12, 0x34, 0, 1)

  val packetType = packet(0) & 0xff
  val code = packet(1) & 0xff
  val checksum = ((packet(2) & 0xff) << 8) | (packet(3) & 0xff)
  val id = ((packet(4) & 0xff) << 8) | (packet(5) & 0xff)
  val seqNum = ((packet(6) & 0xff) << 8) | (packet(7) & 0xff)
  val description = packetType match
    case 0 => "echo reply"
    case 8 => "echo request"
    case 3 => "destination unreachable"
    case _ => "other"

  println(s"type = $packetType ($description)")
  println(s"code = $code")
  println(f"checksum = 0x$checksum%04x")
  println(s"id = $id, seq = $seqNum")
end main
``` 

Most common ICMP messages share the same first four bytes of type and code.  
The remaining bytes depend on the specific ICMP message format.  

## Implementing a minimal "ping" (raw socket)  
This example prints the conceptual send, receive, parse, and RTT loop.  

```scala
@main def main() =
  val start = System.nanoTime()

  println("craft ICMP echo request")
  // A real program would call sendto on a raw socket here.
  println("sendto(fd, request, ...)")
  // A real program would call recvfrom and parse the reply here.
  println("recvfrom(fd, buffer, ...)")

  val end = System.nanoTime()
  val rttMs = (end - start) / 1_000_000.0
  println(f"reply matched, rtt = $rttMs%.3f ms")
end main
``` 

Ping is a tight loop around an echo request, a response wait, and a match step.  
Timestamping around that loop gives an approximate round-trip time.  

## ICMP unreachable message structure  
This example builds a destination unreachable packet with embedded IP bytes.  

```scala
@main def main() =
  val originalIp = Array[Byte](
    0x45, 0, 0, 28, 0, 1, 0x40, 0,
    64, 17, 0, 0,
    10, 0, 0, 1,
    10, 0, 0, 2
  )
  val icmp = Array.fill[Byte](8 + originalIp.length)(0)
  icmp(0) = 3
  icmp(1) = 1
  Array.copy(originalIp, 0, icmp, 8, originalIp.length)

  println(s"icmp type = ${icmp(0)}")
  println(s"embedded ip bytes = ${icmp.length - 8}")
end main
``` 

Destination unreachable messages include unused bytes and part of the trigger.  
That embedded original header helps the sender identify which flow failed.  


## Capturing raw packets  
This example simulates a capture loop that stores a fixed number of frames.  

```scala
import scala.collection.mutable.ListBuffer

@main def main() =
  val captured = ListBuffer.empty[Array[Byte]]
  val stream = List(
    Array[Byte](1, 2, 3),
    Array[Byte](4, 5, 6),
    Array[Byte](7, 8, 9)
  )
  val maxFrames = 2

  for frame <- stream.take(maxFrames) do
    captured += frame

  println(s"captured = ${captured.size}")
end main
``` 

A real sniffer would loop on recvfrom until a timeout or shutdown signal.  
The mutable buffer stands in for a short-lived capture batch.  

## Filtering by protocol  
This example counts TCP, UDP, and ICMP packets by the IPv4 protocol byte.  

```scala
@main def main() =
  val packets = List(
    Array[Byte](0x45, 0, 0, 0, 0, 0, 0, 0, 64, 6),
    Array[Byte](0x45, 0, 0, 0, 0, 0, 0, 0, 64, 17),
    Array[Byte](0x45, 0, 0, 0, 0, 0, 0, 0, 64, 1),
    Array[Byte](0x45, 0, 0, 0, 0, 0, 0, 0, 64, 6)
  )

  def proto(packet: Array[Byte]): Int = packet(9) & 0xff

  println(s"tcp = ${packets.count(p => proto(p) == 6)}")
  println(s"udp = ${packets.count(p => proto(p) == 17)}")
  println(s"icmp = ${packets.count(p => proto(p) == 1)}")
end main
``` 

Filtering by protocol is the first useful reduction for IP packet streams.  
It lets later code route TCP, UDP, and ICMP packets to separate analyzers.  

## Filtering by port  
This example retains only packets whose destination port equals 80.  

```scala
@main def main() =
  val packets = List(
    Array[Byte](0x45, 0, 0, 0, 0, 0, 0, 0, 64, 6,
      0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
      0x13, -120, 0, 80),
    Array[Byte](0x45, 0, 0, 0, 0, 0, 0, 0, 64, 17,
      0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
      0x13, -120, 0, 53)
  )

  def dstPort(packet: Array[Byte]): Int =
    val ihl = (packet(0) & 0x0f) * 4
    ((packet(ihl + 2) & 0xff) << 8) | (packet(ihl + 3) & 0xff)

  val matches = packets.filter(packet => dstPort(packet) == 80)
  println(s"dst port 80 packets = ${matches.size}")
end main
``` 

The transport header starts after the IPv4 header length, not always at byte 20.  
Using IHL first keeps the port parser safe when options are present.  

## Filtering by MAC address  
This example filters Ethernet frames by a destination MAC address.  

```scala
@main def main() =
  val target = Array[Byte](0, 17, 34, 51, 68, 85)
  val frames = List(
    target ++ Array.fill[Byte](54)(0),
    Array[Byte](1, 1, 1, 1, 1, 1) ++ Array.fill[Byte](54)(0)
  )

  val matches = frames.filter(frame => frame.take(6).sameElements(target))
  println(s"matching frames = ${matches.size}")
end main
``` 

MAC filtering is useful before any higher-level protocol work begins.  
It can narrow a capture to traffic meant for one host or one multicast group.  

## Extracting payloads  
This example slices the UDP payload from an IP plus UDP packet.  

```scala
@main def main() =
  val packet = Array[Byte](
    0x45, 0, 0, 33, 0, 1, 0x40, 0,
    64, 17, 0, 0,
    10, 0, 0, 1,
    10, 0, 0, 2,
    0x13, -120, 0, 53, 0, 13, 0, 0,
    72, 101, 108, 108, 111
  )

  val payload = packet.drop(28)
  println(new String(payload, "UTF-8"))
end main
``` 

For a simple IPv4 plus UDP packet, the payload begins after 20 plus 8 bytes.  
Real code should still read IHL in case the IPv4 header has options.  

## Building a simple packet sniffer  
This example wraps capture filtering inside a PacketSniffer object.  

```scala
object PacketSniffer:
  def capture(
    stream: List[Array[Byte]],
    predicate: Array[Byte] => Boolean
  ): List[Array[Byte]] =
    stream.filter(predicate)

@main def main() =
  val frames = List(
    Array[Byte](0x08, 0x00),
    Array[Byte](0x08, 0x06),
    Array[Byte](0x08, 0x00)
  )

  val matches = PacketSniffer.capture(frames, frame => frame.sameElements(Array[Byte](0x08, 0x00)))
  println(s"matches = ${matches.size}")
end main
``` 

A sniffer usually separates capture mechanics from per-frame decision logic.  
A predicate is a compact way to express reusable capture rules.  

## Building a simple protocol analyzer  
This example walks from EtherType to IP protocol and then to port numbers.  

```scala
@main def main() =
  val frame = Array[Byte](
    0, 17, 34, 51, 68, 85,
    102, 119, -120, -103, -86, -69,
    0x08, 0x00,
    0x45, 0, 0, 40, 0, 1, 0x40, 0,
    64, 6, 0, 0,
    10, 0, 0, 1,
    10, 0, 0, 2,
    0x13, -120, 0, 80
  )

  val etherType = ((frame(12) & 0xff) << 8) | (frame(13) & 0xff)
  println(f"etherType = 0x$etherType%04x")

  if etherType == 0x0800 then
    val ipStart = 14
    val protocol = frame(ipStart + 9) & 0xff
    println(s"ip protocol = $protocol")
    if protocol == 6 || protocol == 17 then
      val ihl = (frame(ipStart) & 0x0f) * 4
      val transportStart = ipStart + ihl
      val srcPort = ((frame(transportStart) & 0xff) << 8) | (frame(transportStart + 1) & 0xff)
      val dstPort = ((frame(transportStart + 2) & 0xff) << 8) | (frame(transportStart + 3) & 0xff)
      println(s"ports = $srcPort -> $dstPort")
end main
``` 

Protocol analysis is mostly a disciplined walk through nested header offsets.  
Each earlier field tells the parser how to interpret the next region of bytes.  

## Timestamping captured packets  
This example records relative timestamps for each captured frame.  

```scala
@main def main() =
  val packets = List(Array[Byte](1), Array[Byte](2), Array[Byte](3))
  val start = System.nanoTime()

  for (packet, index) <- packets.zipWithIndex do
    val now = System.nanoTime()
    val elapsedMs = (now - start) / 1_000_000.0
    println(f"frame $index at $elapsedMs%.3f ms size=${packet.length}")
end main
``` 

Relative timing helps show bursts, idle gaps, and ordering during capture.  
The same timestamps often feed packet trace or rate calculations later.  


## Loading libc via Panama  
This example loads libc symbols and calls strlen through a downcall handle.  

```scala
import java.lang.foreign.*

@main def main() =
  val linker = Linker.nativeLinker()
  val lookup = linker.defaultLookup()
  val symbolOpt = lookup.find("strlen")

  if symbolOpt.isPresent then
    val descriptor = FunctionDescriptor.of(ValueLayout.JAVA_LONG, ValueLayout.ADDRESS)
    val strlen = linker.downcallHandle(symbolOpt.get(), descriptor)
    val arena = Arena.ofConfined()

    try
      val text = arena.allocateFrom("panama")
      try
        val length = strlen.invokeWithArguments(text).asInstanceOf[Long]
        println(s"strlen = $length")
      catch
        case e: Throwable => println(e.getMessage)
    finally
      arena.close()
  else
    println("strlen symbol not found")
end main
``` 

The lookup finds native symbols from the default process libraries.  
Calling strlen is a simple way to verify the FFM plumbing before socket work.  

## Calling socket() via Panama  
This example looks up socket and builds a matching function descriptor.  

```scala
import java.lang.foreign.*

@main def main() =
  val linker = Linker.nativeLinker()
  val lookup = linker.defaultLookup()
  val symbolOpt = lookup.find("socket")

  if symbolOpt.isPresent then
    val descriptor = FunctionDescriptor.of(
      ValueLayout.JAVA_INT,
      ValueLayout.JAVA_INT,
      ValueLayout.JAVA_INT,
      ValueLayout.JAVA_INT
    )
    val socketHandle = linker.downcallHandle(symbolOpt.get(), descriptor)

    try
      val fd = socketHandle.invokeWithArguments(
        Integer.valueOf(17),
        Integer.valueOf(3),
        Integer.valueOf(0)
      ).asInstanceOf[Int]
      println(s"fd = $fd")
    catch
      case e: Throwable => println(e.getMessage)
  else
    println("socket symbol not found")
end main
``` 

The descriptor matches the C signature of socket for integer arguments.  
Opening AF_PACKET raw sockets may fail without Linux privileges.  

## Calling bind() via Panama  
This example passes a sockaddr MemorySegment to a bind downcall handle.  

```scala
import java.lang.foreign.*
import java.nio.ByteOrder

@main def main() =
  val linker = Linker.nativeLinker()
  val lookup = linker.defaultLookup()
  val symbolOpt = lookup.find("bind")

  if symbolOpt.isPresent then
    val descriptor = FunctionDescriptor.of(
      ValueLayout.JAVA_INT,
      ValueLayout.JAVA_INT,
      ValueLayout.ADDRESS,
      ValueLayout.JAVA_INT
    )
    val bindHandle = linker.downcallHandle(symbolOpt.get(), descriptor)
    val arena = Arena.ofConfined()

    try
      val sockaddr = arena.allocate(16, 2)
      val shortLayout = ValueLayout.JAVA_SHORT_UNALIGNED.withOrder(ByteOrder.nativeOrder())
      sockaddr.set(shortLayout, 0, 2.toShort)
      sockaddr.set(shortLayout, 2, 8080.toShort)

      try
        val rc = bindHandle.invokeWithArguments(
          Integer.valueOf(3),
          sockaddr,
          Integer.valueOf(16)
        ).asInstanceOf[Int]
        println(s"bind rc = $rc")
      catch
        case e: Throwable => println(e.getMessage)
    finally
      arena.close()
  else
    println("bind symbol not found")
end main
``` 

The signature is int fd, address pointer, and address length in bytes.  
The MemorySegment stands in for a native sockaddr passed to libc.  

## Calling sendto() via Panama  
This example creates a sendto handle and passes a payload MemorySegment.  

```scala
import java.lang.foreign.*

@main def main() =
  val linker = Linker.nativeLinker()
  val lookup = linker.defaultLookup()
  val symbolOpt = lookup.find("sendto")

  if symbolOpt.isPresent then
    val descriptor = FunctionDescriptor.of(
      ValueLayout.JAVA_LONG,
      ValueLayout.JAVA_INT,
      ValueLayout.ADDRESS,
      ValueLayout.JAVA_LONG,
      ValueLayout.JAVA_INT,
      ValueLayout.ADDRESS,
      ValueLayout.JAVA_INT
    )
    val sendtoHandle = linker.downcallHandle(symbolOpt.get(), descriptor)
    val arena = Arena.ofConfined()

    try
      val payload = arena.allocateFrom(Array[Byte](1, 2, 3, 4))
      val sockaddr = arena.allocate(16, 1)
      try
        val sent = sendtoHandle.invokeWithArguments(
          Integer.valueOf(3),
          payload,
          java.lang.Long.valueOf(payload.byteSize()),
          Integer.valueOf(0),
          sockaddr,
          Integer.valueOf(16)
        ).asInstanceOf[Long]
        println(s"sendto bytes = $sent")
      catch
        case e: Throwable => println(e.getMessage)
    finally
      arena.close()
  else
    println("sendto symbol not found")
end main
``` 

The payload is an off-heap segment so native code can read it directly.  
sendto also needs a destination address segment and its length.  

## Calling recvfrom() via Panama  
This example allocates a receive buffer MemorySegment for recvfrom.  

```scala
import java.lang.foreign.*

@main def main() =
  val linker = Linker.nativeLinker()
  val lookup = linker.defaultLookup()
  val symbolOpt = lookup.find("recvfrom")

  if symbolOpt.isPresent then
    val descriptor = FunctionDescriptor.of(
      ValueLayout.JAVA_LONG,
      ValueLayout.JAVA_INT,
      ValueLayout.ADDRESS,
      ValueLayout.JAVA_LONG,
      ValueLayout.JAVA_INT,
      ValueLayout.ADDRESS,
      ValueLayout.ADDRESS
    )
    val recvfromHandle = linker.downcallHandle(symbolOpt.get(), descriptor)
    val arena = Arena.ofConfined()

    try
      val buffer = arena.allocate(2048, 1)
      val from = arena.allocate(32, 1)
      val fromLen = arena.allocate(ValueLayout.JAVA_INT)
      fromLen.set(ValueLayout.JAVA_INT, 0, 32)

      try
        val n = recvfromHandle.invokeWithArguments(
          Integer.valueOf(3),
          buffer,
          java.lang.Long.valueOf(buffer.byteSize()),
          Integer.valueOf(0),
          from,
          fromLen
        ).asInstanceOf[Long]
        println(s"recvfrom bytes = $n")
      catch
        case e: Throwable => println(e.getMessage)
    finally
      arena.close()
  else
    println("recvfrom symbol not found")
end main
``` 

recvfrom fills both the data buffer and the caller-provided source address.  
The length pointer tells native code how much address storage is available.  

## Calling setsockopt() via Panama  
This example sets SO_REUSEADDR with a 4-byte integer option segment.  

```scala
import java.lang.foreign.*

@main def main() =
  val linker = Linker.nativeLinker()
  val lookup = linker.defaultLookup()
  val symbolOpt = lookup.find("setsockopt")

  if symbolOpt.isPresent then
    val descriptor = FunctionDescriptor.of(
      ValueLayout.JAVA_INT,
      ValueLayout.JAVA_INT,
      ValueLayout.JAVA_INT,
      ValueLayout.JAVA_INT,
      ValueLayout.ADDRESS,
      ValueLayout.JAVA_INT
    )
    val setsockoptHandle = linker.downcallHandle(symbolOpt.get(), descriptor)
    val arena = Arena.ofConfined()

    try
      val opt = arena.allocate(ValueLayout.JAVA_INT)
      opt.set(ValueLayout.JAVA_INT, 0, 1)

      try
        val rc = setsockoptHandle.invokeWithArguments(
          Integer.valueOf(3),
          Integer.valueOf(1),
          Integer.valueOf(2),
          opt,
          Integer.valueOf(4)
        ).asInstanceOf[Int]
        println(s"setsockopt rc = $rc")
      catch
        case e: Throwable => println(e.getMessage)
    finally
      arena.close()
  else
    println("setsockopt symbol not found")
end main
``` 

The option value is just native memory holding an integer equal to one.  
This pattern also works for many other socket options and struct values.  

## Calling select() via Panama  
This example prepares an fd_set and timeval segment for a select call.  

```scala
import java.lang.foreign.*
import java.nio.ByteOrder

@main def main() =
  def setFd(fdSet: MemorySegment, fd: Int): Unit =
    val byteIndex = fd / 8
    val bit = 1 << (fd % 8)
    val current = fdSet.get(ValueLayout.JAVA_BYTE, byteIndex.toLong)
    fdSet.set(ValueLayout.JAVA_BYTE, byteIndex.toLong, (current | bit).toByte)

  val linker = Linker.nativeLinker()
  val lookup = linker.defaultLookup()
  val symbolOpt = lookup.find("select")

  if symbolOpt.isPresent then
    val descriptor = FunctionDescriptor.of(
      ValueLayout.JAVA_INT,
      ValueLayout.JAVA_INT,
      ValueLayout.ADDRESS,
      ValueLayout.ADDRESS,
      ValueLayout.ADDRESS,
      ValueLayout.ADDRESS
    )
    val selectHandle = linker.downcallHandle(symbolOpt.get(), descriptor)
    val arena = Arena.ofConfined()

    try
      val fdSet = arena.allocate(128, 1)
      val timeval = arena.allocate(16, 8)
      val longLayout = ValueLayout.JAVA_LONG_UNALIGNED.withOrder(ByteOrder.nativeOrder())
      setFd(fdSet, 3)
      timeval.set(longLayout, 0, 1L)
      timeval.set(longLayout, 8, 0L)

      try
        val ready = selectHandle.invokeWithArguments(
          Integer.valueOf(4),
          fdSet,
          MemorySegment.NULL,
          MemorySegment.NULL,
          timeval
        ).asInstanceOf[Int]
        println(s"ready = $ready")
      catch
        case e: Throwable => println(e.getMessage)
    finally
      arena.close()
  else
    println("select symbol not found")
end main
``` 

select expects bitsets that name the watched file descriptors.  
The 128-byte fd_set here is a simple educational stand-in for the C layout.  

## Calling epoll_create1() via Panama  
This example calls epoll_create1 and prints the resulting epoll descriptor.  

```scala
import java.lang.foreign.*

@main def main() =
  val linker = Linker.nativeLinker()
  val lookup = linker.defaultLookup()
  val symbolOpt = lookup.find("epoll_create1")

  if symbolOpt.isPresent then
    val descriptor = FunctionDescriptor.of(ValueLayout.JAVA_INT, ValueLayout.JAVA_INT)
    val handle = linker.downcallHandle(symbolOpt.get(), descriptor)

    try
      val fd = handle.invokeWithArguments(Integer.valueOf(0)).asInstanceOf[Int]
      println(s"epoll fd = $fd")
    catch
      case e: Throwable => println(e.getMessage)
  else
    println("epoll_create1 symbol not found")
end main
``` 

Linux epoll scales better than select when many descriptors are active.  
The create call returns a new descriptor used for later epoll operations.  

## Calling epoll_ctl() via Panama  
This example defines an epoll_event layout and calls EPOLL_CTL_ADD.  

```scala
import java.lang.foreign.*

@main def main() =
  val eventLayout = MemoryLayout.structLayout(
    ValueLayout.JAVA_INT.withName("events"),
    ValueLayout.JAVA_LONG_UNALIGNED.withName("data")
  )
  val linker = Linker.nativeLinker()
  val lookup = linker.defaultLookup()
  val symbolOpt = lookup.find("epoll_ctl")

  if symbolOpt.isPresent then
    val descriptor = FunctionDescriptor.of(
      ValueLayout.JAVA_INT,
      ValueLayout.JAVA_INT,
      ValueLayout.JAVA_INT,
      ValueLayout.JAVA_INT,
      ValueLayout.ADDRESS
    )
    val handle = linker.downcallHandle(symbolOpt.get(), descriptor)
    val arena = Arena.ofConfined()

    try
      val event = arena.allocate(eventLayout)
      event.set(ValueLayout.JAVA_INT, 0, 1)
      event.set(ValueLayout.JAVA_LONG_UNALIGNED, 4, 3L)

      try
        val rc = handle.invokeWithArguments(
          Integer.valueOf(5),
          Integer.valueOf(1),
          Integer.valueOf(3),
          event
        ).asInstanceOf[Int]
        println(s"epoll_ctl rc = $rc")
      catch
        case e: Throwable => println(e.getMessage)
    finally
      arena.close()
  else
    println("epoll_ctl symbol not found")
end main
``` 

The layout models events as a 32-bit mask and data as a 64-bit payload.  
That mirrors the compact epoll_event structure many packet tools rely on.  

## Calling epoll_wait() via Panama  
This example allocates an array of epoll_event structures for epoll_wait.  

```scala
import java.lang.foreign.*

@main def main() =
  val eventLayout = MemoryLayout.structLayout(
    ValueLayout.JAVA_INT.withName("events"),
    ValueLayout.JAVA_LONG_UNALIGNED.withName("data")
  )
  val linker = Linker.nativeLinker()
  val lookup = linker.defaultLookup()
  val symbolOpt = lookup.find("epoll_wait")

  if symbolOpt.isPresent then
    val descriptor = FunctionDescriptor.of(
      ValueLayout.JAVA_INT,
      ValueLayout.JAVA_INT,
      ValueLayout.ADDRESS,
      ValueLayout.JAVA_INT,
      ValueLayout.JAVA_INT
    )
    val handle = linker.downcallHandle(symbolOpt.get(), descriptor)
    val arena = Arena.ofConfined()

    try
      val maxEvents = 64
      val events = arena.allocate(eventLayout.byteSize() * maxEvents, eventLayout.byteAlignment())
      try
        val ready = handle.invokeWithArguments(
          Integer.valueOf(5),
          events,
          Integer.valueOf(maxEvents),
          Integer.valueOf(1000)
        ).asInstanceOf[Int]
        println(s"ready = $ready")
      catch
        case e: Throwable => println(e.getMessage)
    finally
      arena.close()
  else
    println("epoll_wait symbol not found")
end main
``` 

epoll_wait fills the array with ready event records up to maxEvents.  
A timeout of one second keeps the wait bounded for polling-style tools.  

## Calling close() via Panama  
This example calls close on a descriptor and checks the return code.  

```scala
import java.lang.foreign.*

@main def main() =
  val linker = Linker.nativeLinker()
  val lookup = linker.defaultLookup()
  val symbolOpt = lookup.find("close")

  if symbolOpt.isPresent then
    val descriptor = FunctionDescriptor.of(ValueLayout.JAVA_INT, ValueLayout.JAVA_INT)
    val closeHandle = linker.downcallHandle(symbolOpt.get(), descriptor)

    try
      val rc = closeHandle.invokeWithArguments(Integer.valueOf(3)).asInstanceOf[Int]
      println(s"close rc = $rc")
    catch
      case e: Throwable => println(e.getMessage)
  else
    println("close symbol not found")
end main
``` 

Native resources must still be closed even when memory management is safer.  
Calling close explicitly keeps descriptor lifetimes predictable in examples.  


## Allocating off-heap MemorySegment  
This example allocates 1024 bytes of confined off-heap memory.  

```scala
import java.lang.foreign.*

@main def main() =
  val arena = Arena.ofConfined()

  try
    val segment = arena.allocate(ValueLayout.JAVA_BYTE, 1024)
    println(s"byteSize = ${segment.byteSize()}")
  finally
    arena.close()
end main
``` 

Off-heap memory is useful when native APIs need direct access to bytes.  
A confined arena gives the segment a clear lifetime inside one thread.  

## Writing bytes into MemorySegment  
This example writes four IPv4 address bytes into an off-heap segment.  

```scala
import java.lang.foreign.*

@main def main() =
  val arena = Arena.ofConfined()

  try
    val seg = arena.allocate(ValueLayout.JAVA_BYTE, 4)
    val ip = Array[Byte](192.toByte, 168.toByte, 1, 10)

    for (value, index) <- ip.zipWithIndex do
      seg.set(ValueLayout.JAVA_BYTE, index.toLong, value)

    println("wrote 4 bytes")
  finally
    arena.close()
end main
``` 

set writes directly to the target segment at the given byte offset.  
This is a natural way to build native structs one field at a time.  

## Reading bytes from MemorySegment  
This example reads four bytes back and formats them as a dotted IPv4 string.  

```scala
import java.lang.foreign.*

@main def main() =
  val arena = Arena.ofConfined()

  try
    val seg = arena.allocate(ValueLayout.JAVA_BYTE, 4)
    val ip = Array[Byte](192.toByte, 168.toByte, 1, 10)

    for (value, index) <- ip.zipWithIndex do
      seg.set(ValueLayout.JAVA_BYTE, index.toLong, value)

    val formatted =
      (0 until 4)
        .map(i => seg.get(ValueLayout.JAVA_BYTE, i.toLong) & 0xff)
        .mkString(".")

    println(formatted)
  finally
    arena.close()
end main
``` 

get performs the inverse operation and returns the typed value at an offset.  
Formatting the result verifies that the bytes were stored correctly.  

## Defining Ethernet header struct layout  
This example models an Ethernet header with MemoryLayout utilities.  

```scala
import java.lang.foreign.*

@main def main() =
  val ethernetLayout = MemoryLayout.structLayout(
    MemoryLayout.sequenceLayout(6, ValueLayout.JAVA_BYTE).withName("dst"),
    MemoryLayout.sequenceLayout(6, ValueLayout.JAVA_BYTE).withName("src"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("etherType")
  )

  println(s"ethernet bytes = ${ethernetLayout.byteSize()}")
end main
``` 

Sequence layouts are convenient for repeated byte fields such as MAC addresses.  
The final unaligned short keeps the total header size at 14 bytes.  

## Defining IPv4 header struct layout  
This example defines a compact 20-byte MemoryLayout for an IPv4 header.  

```scala
import java.lang.foreign.*

@main def main() =
  val ipv4Layout = MemoryLayout.structLayout(
    ValueLayout.JAVA_BYTE.withName("version_ihl"),
    ValueLayout.JAVA_BYTE.withName("dscp"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("totalLen"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("id"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("flagsFragment"),
    ValueLayout.JAVA_BYTE.withName("ttl"),
    ValueLayout.JAVA_BYTE.withName("protocol"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("checksum"),
    ValueLayout.JAVA_INT_UNALIGNED.withName("srcIp"),
    ValueLayout.JAVA_INT_UNALIGNED.withName("dstIp")
  )

  println(s"ipv4 bytes = ${ipv4Layout.byteSize()}")
end main
``` 

Named fields make later VarHandle access much easier to read.  
Unaligned integer layouts preserve the packed network header size.  

## Defining UDP header struct layout  
This example defines the four fields of a UDP header as a MemoryLayout.  

```scala
import java.lang.foreign.*

@main def main() =
  val udpLayout = MemoryLayout.structLayout(
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("srcPort"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("dstPort"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("length"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("checksum")
  )

  println(s"udp bytes = ${udpLayout.byteSize()}")
end main
``` 

UDP is a good first layout because it is small and strictly fixed width.  
This kind of layout can back either parsing or packet construction.  

## Defining TCP header struct layout  
This example defines a compact 20-byte layout for a basic TCP header.  

```scala
import java.lang.foreign.*

@main def main() =
  val tcpLayout = MemoryLayout.structLayout(
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("srcPort"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("dstPort"),
    ValueLayout.JAVA_INT_UNALIGNED.withName("seqNum"),
    ValueLayout.JAVA_INT_UNALIGNED.withName("ackNum"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("dataOffsetFlags"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("window"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("checksum"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("urgent")
  )

  println(s"tcp bytes = ${tcpLayout.byteSize()}")
end main
``` 

The combined dataOffsetFlags field keeps the classic 20-byte base header shape.  
Options would live beyond this base layout when they are present.  

## Defining ICMP header struct layout  
This example models the common 8-byte ICMP echo header fields.  

```scala
import java.lang.foreign.*

@main def main() =
  val icmpLayout = MemoryLayout.structLayout(
    ValueLayout.JAVA_BYTE.withName("type"),
    ValueLayout.JAVA_BYTE.withName("code"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("checksum"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("id"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("seqNum")
  )

  println(s"icmp bytes = ${icmpLayout.byteSize()}")
end main
``` 

ICMP echo uses a small header, so a direct layout mirrors it nicely.  
The same base pattern appears in many other ICMP message forms.  

## Mapping struct fields to MemoryLayout  
This example gets a VarHandle for the ttl field and updates it in place.  

```scala
import java.lang.foreign.*

@main def main() =
  val ipv4Layout = MemoryLayout.structLayout(
    ValueLayout.JAVA_BYTE.withName("version_ihl"),
    ValueLayout.JAVA_BYTE.withName("dscp"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("totalLen"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("id"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("flagsFragment"),
    ValueLayout.JAVA_BYTE.withName("ttl"),
    ValueLayout.JAVA_BYTE.withName("protocol"),
    ValueLayout.JAVA_SHORT_UNALIGNED.withName("checksum"),
    ValueLayout.JAVA_INT_UNALIGNED.withName("srcIp"),
    ValueLayout.JAVA_INT_UNALIGNED.withName("dstIp")
  )
  val ttlHandle = ipv4Layout.varHandle(MemoryLayout.PathElement.groupElement("ttl"))
  val arena = Arena.ofConfined()

  try
    val header = arena.allocate(ipv4Layout)
    ttlHandle.set(header, 64.toByte)
    println(s"ttl = ${ttlHandle.get(header)}")
  finally
    arena.close()
end main
``` 

A VarHandle lets code address a named field without manual offset math.  
That improves clarity when native layouts become larger or more nested.  

## Parsing packets using MemorySegment slices  
This example splits one MemorySegment into IP and UDP header slices.  

```scala
import java.lang.foreign.*

@main def main() =
  val arena = Arena.ofConfined()

  try
    val packet = arena.allocate(28, 1)
    val ipHeader = packet.asSlice(0, 20)
    val udpHeader = packet.asSlice(20, 8)

    println(s"ipHeader bytes = ${ipHeader.byteSize()}")
    println(s"udpHeader bytes = ${udpHeader.byteSize()}")
  finally
    arena.close()
end main
``` 

Slices are cheap views onto the same off-heap storage region.  
They make header boundaries explicit without copying any bytes.  

## Copying data between MemorySegments  
This example copies a byte pattern from one segment to another.  

```scala
import java.lang.foreign.*

@main def main() =
  val arena = Arena.ofConfined()

  try
    val src = arena.allocate(8, 1)
    val dst = arena.allocate(8, 1)

    for i <- 0 until 8 do
      src.set(ValueLayout.JAVA_BYTE, i.toLong, (i + 1).toByte)

    MemorySegment.copy(src, 0, dst, 0, 8)

    val copied = (0 until 8).map(i => dst.get(ValueLayout.JAVA_BYTE, i.toLong) & 0xff)
    println(copied.mkString(" "))
  finally
    arena.close()
end main
``` 

Direct segment copies are useful when packets move between staging buffers.  
The destination stays off-heap, so native code can read it immediately.  

## MemorySegment byte order handling  
This example writes the same port value in big-endian and little-endian form.  

```scala
import java.lang.foreign.*
import java.nio.ByteOrder

@main def main() =
  val arena = Arena.ofConfined()

  try
    val seg = arena.allocate(4, 1)
    val big = ValueLayout.JAVA_SHORT_UNALIGNED.withOrder(ByteOrder.BIG_ENDIAN)
    val little = ValueLayout.JAVA_SHORT_UNALIGNED.withOrder(ByteOrder.LITTLE_ENDIAN)

    seg.set(big, 0, 8080.toShort)
    seg.set(little, 2, 8080.toShort)

    println(f"big-endian    = 0x${seg.get(big, 0) & 0xffff}%04x")
    println(f"little-endian = 0x${seg.get(little, 2) & 0xffff}%04x")
  finally
    arena.close()
end main
``` 

Network protocols normally use big-endian byte order, also called network order.  
Seeing both layouts side by side makes endianness bugs easier to spot.  


## Hex dump utility  
This example implements a tcpdump-style hex and ASCII formatter.  

```scala
@main def main() =
  def hexDump(data: Array[Byte]): String =
    data.grouped(16).zipWithIndex.map:
      case (row, rowIndex) =>
        val hexPart = row.map(b => f"${b & 0xff}%02x").padTo(16, "  ").mkString(" ")
        val asciiPart = row.map: b =>
          val c = b & 0xff
          if c >= 32 && c <= 126 then c.toChar else '.'
        f"${rowIndex * 16}%04x  $hexPart  $asciiPart"
    .mkString("\n")

  val data = "Scala raw packets".getBytes("UTF-8")
  println(hexDump(data))
end main
``` 

Hex dumps are the fastest way to inspect unknown packet bytes during debugging.  
Showing ASCII beside hex helps spot payload text immediately.  

## Packet builder helper  
This example defines a chainable PacketBuilder for layered packet assembly.  

```scala
import java.nio.ByteBuffer
import scala.collection.mutable.ArrayBuffer

class PacketBuilder:
  private val parts = ArrayBuffer.empty[Array[Byte]]

  def ethernetHeader(dst: Array[Byte], src: Array[Byte], etherType: Int): this.type =
    val bb = ByteBuffer.allocate(14)
    bb.put(dst)
    bb.put(src)
    bb.putShort(etherType.toShort)
    parts += bb.array()
    this

  def ipHeader(src: Array[Byte], dst: Array[Byte], protocol: Int, totalLen: Int): this.type =
    val bb = ByteBuffer.allocate(20)
    bb.put(0x45.toByte)
    bb.put(0.toByte)
    bb.putShort(totalLen.toShort)
    bb.putShort(1.toShort)
    bb.putShort(0x4000.toShort)
    bb.put(64.toByte)
    bb.put(protocol.toByte)
    bb.putShort(0.toShort)
    bb.put(src)
    bb.put(dst)
    parts += bb.array()
    this

  def udpHeader(srcPort: Int, dstPort: Int, length: Int): this.type =
    val bb = ByteBuffer.allocate(8)
    bb.putShort(srcPort.toShort)
    bb.putShort(dstPort.toShort)
    bb.putShort(length.toShort)
    bb.putShort(0.toShort)
    parts += bb.array()
    this

  def payload(data: Array[Byte]): this.type =
    parts += data
    this

  def build(): Array[Byte] =
    parts.flatten.toArray

@main def main() =
  val packet = PacketBuilder()
    .ethernetHeader(Array.fill[Byte](6)(1), Array.fill[Byte](6)(2), 0x0800)
    .ipHeader(Array[Byte](10, 0, 0, 1), Array[Byte](10, 0, 0, 2), 17, 33)
    .udpHeader(50000, 53, 13)
    .payload("Hello".getBytes("UTF-8"))
    .build()

  println(s"packet bytes = ${packet.length}")
end main
``` 

A builder hides repetitive header allocation and concatenation logic.  
Chainable calls also make the packet structure easier to read top to bottom.  

## Checksum helper functions  
This example reuses one Internet checksum routine across packet types.  

```scala
@main def main() =
  def internetChecksum(data: Array[Byte]): Int =
    val padded = if data.length % 2 == 0 then data else data :+ 0.toByte
    var sum = 0

    for i <- 0 until padded.length by 2 do
      val word = ((padded(i) & 0xff) << 8) | (padded(i + 1) & 0xff)
      sum += word
      while sum > 0xffff do
        sum = (sum & 0xffff) + (sum >>> 16)

    (~sum) & 0xffff

  val sample = Array[Byte](0x45, 0, 0, 28, 0, 1, 0x40, 0, 64, 17, 0, 0)
  println(f"checksum = 0x${internetChecksum(sample)}%04x")
end main
``` 

The same helper can serve IPv4, UDP, TCP, and ICMP with small wrappers.  
Only the covered bytes differ between those protocol cases.  

## Protocol dispatcher (EtherType → handler)  
This example maps EtherType values to frame handler functions.  

```scala
@main def main() =
  def handleIp(frame: Array[Byte]): Unit = println(s"IPv4 frame ${frame.length} bytes")
  def handleArp(frame: Array[Byte]): Unit = println(s"ARP frame ${frame.length} bytes")
  def handleIpv6(frame: Array[Byte]): Unit = println(s"IPv6 frame ${frame.length} bytes")

  val handlers = Map[Int, Array[Byte] => Unit](
    0x0800 -> handleIp,
    0x0806 -> handleArp,
    0x86dd -> handleIpv6
  )
  val frame = Array.fill[Byte](14)(0)
  val etherType = 0x0800

  handlers.get(etherType) match
    case Some(handler) => handler(frame)
    case None          => println("no handler")
end main
``` 

Dispatch tables keep protocol branching small and easy to extend.  
They also separate frame classification from protocol-specific parsing.  

## Simple intrusion-detection-style pattern matching  
This example scans packet payloads for the byte sequence ATTACK.  

```scala
@main def main() =
  def containsSignature(data: Array[Byte], signature: Array[Byte]): Boolean =
    data.sliding(signature.length).exists(window => window.sameElements(signature))

  val packet = "normal ATTACK marker".getBytes("UTF-8")
  val signature = "ATTACK".getBytes("UTF-8")

  if containsSignature(packet, signature) then
    println("alert: signature detected")
  else
    println("clean")
end main
``` 

This is intentionally simple, but it shows the core of byte-pattern scanning.  
Real detectors add offsets, context, and many optimized signatures.  

## Packet rate measurement  
This example counts packets inside a sliding one-second window.  

```scala
import scala.collection.mutable.Queue

@main def main() =
  val times = Queue.empty[Long]
  val arrivals = List(0L, 200L, 450L, 900L, 1200L, 1500L)

  for offset <- arrivals do
    val now = offset
    times.enqueue(now)
    while times.nonEmpty && now - times.head > 1000 do
      times.dequeue()
    println(s"pps window = ${times.size}")
end main
``` 

Packet rate windows show bursts more clearly than a cumulative total.  
The same idea can back throughput charts or alert thresholds.  

## Interface statistics via raw socket  
This example reads /proc/net/dev and prints RX and TX counters.  

```scala
import scala.io.Source

@main def main() =
  val interfaceName = "lo"
  val lines = Source.fromFile("/proc/net/dev").getLines().drop(2).toList

  val maybeStats = lines.find(_.trim.startsWith(s"$interfaceName:"))
  maybeStats match
    case Some(line) =>
      val parts = line.split(":")(1).trim.split("\\s+")
      val rxBytes = parts(0)
      val rxPackets = parts(1)
      val txBytes = parts(8)
      val txPackets = parts(9)
      println(s"$interfaceName rxBytes=$rxBytes rxPackets=$rxPackets")
      println(s"$interfaceName txBytes=$txBytes txPackets=$txPackets")
    case None =>
      println("interface not found")
end main
``` 

Kernel interface counters help correlate crafted traffic with actual device use.  
They are often easier to read than issuing extra control syscalls.  

## IP address formatting utility  
This example formats IPv4 and IPv6 byte arrays for display.  

```scala
@main def main() =
  def formatIPv4(bytes: Array[Byte]): String =
    bytes.map(_ & 0xff).mkString(".")

  def formatIPv6(bytes: Array[Byte]): String =
    bytes.grouped(2)
      .map(group => f"${((group(0) & 0xff) << 8) | (group(1) & 0xff)}%x")
      .mkString(":")

  val ipv4 = Array[Byte](192.toByte, 168.toByte, 1, 10)
  val ipv6 = Array.fill[Byte](16)(0)
  ipv6(15) = 1

  println(formatIPv4(ipv4))
  println(formatIPv6(ipv6))
end main
``` 

Readable address formatting is essential when printing parser output.  
The helper functions keep that logic out of packet-decoding code.  

## MAC address parsing and formatting  
This example converts MAC addresses between colon text and raw bytes.  

```scala
@main def main() =
  def parseMac(s: String): Array[Byte] =
    s.split(":").map(part => Integer.parseInt(part, 16).toByte)

  def formatMac(bytes: Array[Byte]): String =
    bytes.map(b => f"${b & 0xff}%02x").mkString(":")

  val mac = parseMac("00:11:22:33:44:55")
  println(formatMac(mac))
end main
``` 

Parsing and formatting MAC addresses is common in sniffers and frame builders.  
Keeping both directions together avoids duplicated conversion code.  

## Port scanner skeleton (educational)  
This example iterates ports and prints the TCP SYN targets it would probe.  

```scala
@main def main() =
  for port <- 1 to 1024 do
    val tcpHeader = Array.fill[Byte](20)(0)
    tcpHeader(2) = ((port >>> 8) & 0xff).toByte
    tcpHeader(3) = (port & 0xff).toByte
    tcpHeader(12) = (5 << 4).toByte
    tcpHeader(13) = 0x02

    if port <= 5 || port >= 1020 then
      println(s"would send SYN to port $port")
end main
``` 

The example stays educational by avoiding any actual transmission.  
It still shows how a scanner varies only a few header bytes per probe.  

## ARP packet structure (conceptual)  
This example builds a 28-byte ARP request in a byte buffer.  

```scala
import java.nio.ByteBuffer
import java.nio.ByteOrder

@main def main() =
  val senderHw = Array[Byte](0, 17, 34, 51, 68, 85)
  val senderIp = Array[Byte](192.toByte, 168.toByte, 1, 10)
  val targetHw = Array.fill[Byte](6)(0)
  val targetIp = Array[Byte](192.toByte, 168.toByte, 1, 1)

  val arp = ByteBuffer.allocate(28).order(ByteOrder.BIG_ENDIAN)
  arp.putShort(1.toShort)
  arp.putShort(0x0800.toShort)
  arp.put(6.toByte)
  arp.put(4.toByte)
  arp.putShort(1.toShort)
  arp.put(senderHw)
  arp.put(senderIp)
  arp.put(targetHw)
  arp.put(targetIp)

  println(s"arp bytes = ${arp.array().length}")
end main
``` 

ARP links layer two and layer three by resolving MACs from IPv4 addresses.  
The zero target hardware field marks this packet as a request.  

## DNS query parsing skeleton  
This example parses the first 12 bytes of a DNS message header.  

```scala
@main def main() =
  val dns = Array[Byte](
    0x12, 0x34,
    0x01, 0x00,
    0x00, 0x01,
    0x00, 0x00,
    0x00, 0x00,
    0x00, 0x00
  )

  def u16(offset: Int): Int =
    ((dns(offset) & 0xff) << 8) | (dns(offset + 1) & 0xff)

  println(f"transactionId = 0x${u16(0)}%04x")
  println(f"flags = 0x${u16(2)}%04x")
  println(s"questions = ${u16(4)}")
  println(s"answers = ${u16(6)}")
  println(s"authority = ${u16(8)}")
  println(s"additional = ${u16(10)}")
end main
``` 

The DNS header is small but packed, so helper functions reduce repetition.  
Name parsing would continue after these fixed twelve bytes.  

## PCAP-style file writer  
This example writes a minimal PCAP stream with one packet record.  

```scala
import java.io.ByteArrayOutputStream
import java.nio.ByteBuffer
import java.nio.ByteOrder

@main def main() =
  val out = ByteArrayOutputStream()
  val global = ByteBuffer.allocate(24).order(ByteOrder.LITTLE_ENDIAN)
  global.putInt(0xa1b2c3d4)
  global.putShort(2.toShort)
  global.putShort(4.toShort)
  global.putInt(0)
  global.putInt(0)
  global.putInt(65535)
  global.putInt(1)
  out.write(global.array())

  val packet = Array[Byte](1, 2, 3, 4)
  val record = ByteBuffer.allocate(16).order(ByteOrder.LITTLE_ENDIAN)
  record.putInt(0)
  record.putInt(0)
  record.putInt(packet.length)
  record.putInt(packet.length)
  out.write(record.array())
  out.write(packet)

  println(s"pcap bytes = ${out.toByteArray.length}")
end main
``` 

PCAP stores a global header once and then one record header per packet.  
That makes it easy to export synthetic traffic for later inspection.  

## Network namespace detection  
This example reads the current network namespace inode from /proc/self/ns/net.  

```scala
import java.nio.file.Files
import java.nio.file.Path

@main def main() =
  val target = Files.readSymbolicLink(Path.of("/proc/self/ns/net")).toString
  println(s"namespace link = $target")
  println("namespace awareness matters because raw sockets see per-namespace state")
end main
``` 

Linux namespaces can isolate interfaces, routes, and visible packet traffic.  
Knowing the current namespace prevents confusion during low-level experiments.  

