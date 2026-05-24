# OS-Level Networking in Scala 3

Operating-system-level networking gives a program direct access to the  
data structures that the kernel uses to move packets, resolve addresses,  
and maintain connections. Instead of asking a command-line tool for the  
routing table and parsing its output, you read the same kernel-exported  
information directly, with full control over filtering, aggregation, and  
timing. This capability is essential for network monitors, configuration  
managers, security analyzers, and diagnostic utilities that must remain  
efficient and portable across Linux environments.  

## How Linux exposes networking state

Linux follows the UNIX philosophy of presenting everything as a file.  
The kernel exposes live networking state through two virtual file systems  
that are populated on demand, without any persistent storage behind them.  
`/proc/net` holds text files such as `dev`, `arp`, `route`, `tcp`, and  
`udp`. Each read returns a fresh snapshot of the current kernel state.  
`/sys/class/net` organises per-interface attributes in a directory  
hierarchy where each attribute is a separate file holding a single value.  
Reading these pseudo-files is cheap, always current, and requires no root  
privileges for the majority of entries.  

## Netlink sockets

Netlink is a Linux inter-process communication mechanism designed  
specifically for exchanging network configuration data between the kernel  
and user-space processes. Unlike `/proc` files, Netlink supports  
bidirectional communication: you can query current state and also request  
changes. The `NETLINK_ROUTE` protocol family exposes routes, links,  
addresses, and neighbor tables through a binary message protocol defined  
in `<linux/rtnetlink.h>`. Messages follow a fixed-size `nlmsghdr` header  
followed by a family-specific payload and a series of type-length-value  
attributes called `rtattr` records.  

## Routing tables, ARP tables, and interfaces

A routing table is a kernel data structure that maps destination IP  
prefixes to next-hop gateways and output interfaces. When forwarding a  
packet, the kernel performs a longest-prefix match to find the most  
specific matching entry. An ARP (Address Resolution Protocol) table maps  
layer-3 IP addresses to layer-2 MAC addresses so that packets can be  
delivered on Ethernet links. Network interfaces are the logical or  
physical attachment points for the network. Each interface carries a name,  
a hardware address, a maximum transmission unit, one or more IP addresses,  
and a set of operational flags.  

## Kernel space versus user space

The Linux kernel runs in a privileged execution mode called kernel space  
with unrestricted access to hardware and protected memory. User-space  
programs run with restricted privileges and access kernel data only  
through system calls or the virtual file systems described above. This  
boundary prevents user code from accidentally corrupting kernel data  
structures. When your Scala application reads `/proc/net/dev`, the kernel  
copies the current counter values into the virtual file on demand, without  
exposing any raw kernel memory pointers to the program.  

## Project Panama and native networking APIs

Project Panama (the `java.lang.foreign` package, finalised in Java 21)  
provides a Foreign Function and Memory API that lets Scala code call C  
functions directly, without writing JNI glue code. For networking, this  
means calling `socket()`, `bind()`, `sendto()`, and `recvfrom()` using  
`Linker.nativeLinker()` and representing C structs with `MemorySegment`.  
Panama also provides safe off-heap memory management through `Arena`,  
which automatically frees native memory when the arena is closed, even  
if an exception is thrown.  

## Safety considerations

Reading `/proc` and `/sys` is generally safe but requires care. Most  
entries under `/proc/net` are world-readable, but some may require  
elevated privileges on hardened or containerised systems. The content is  
ephemeral and changes between reads, so reading the entire file at once  
into a `List[String]` is safer than processing it incrementally across  
separate calls. Always handle `IOException` gracefully, because the  
underlying interface or route may disappear between the time you list it  
and the time you read its attributes.  

---

## Reading /proc/net/dev

Reads the raw content of `/proc/net/dev` and prints every line.  

```scala
import scala.io.Source

@main def main() =
    val source = Source.fromFile("/proc/net/dev")
    try
        source.getLines().foreach(println)
    finally
        source.close()
end main
```

The `/proc/net/dev` pseudo-file contains one row per network interface  
preceded by two header lines. The kernel writes the current counters into  
this file each time it is opened. Using `Source.fromFile` and calling  
`getLines()` yields a lazy iterator over each line. Wrapping the read in  
a `try/finally` block ensures that the underlying file descriptor is  
closed even when an exception occurs.  

## Parsing RX/TX bytes and packets

Extracts the received and transmitted byte counts for every interface.  

```scala
import scala.io.Source

@main def main() =
    val source = Source.fromFile("/proc/net/dev")
    try
        val lines = source.getLines().toList
        val data = lines.drop(2)
        for line <- data do
            val parts = line.trim.split("""\s+""")
            val iface = parts(0).stripSuffix(":")
            val rxBytes   = parts(1).toLong
            val rxPackets = parts(2).toLong
            val txBytes   = parts(9).toLong
            val txPackets = parts(10).toLong
            println(s"$iface  RX=$rxBytes bytes ($rxPackets pkts)" +
                    s"  TX=$txBytes bytes ($txPackets pkts)")
    finally
        source.close()
end main
```

The first two lines of `/proc/net/dev` are column headers and must be  
skipped with `drop(2)`. Each remaining line starts with the interface  
name followed by a colon, so `stripSuffix(":")` removes that punctuation.  
The columns are fixed: index 1 holds received bytes, index 2 holds  
received packets, index 9 holds transmitted bytes, and index 10 holds  
transmitted packets. Splitting on `\s+` after trimming handles the  
variable-width spacing that the kernel uses when printing numbers.  

## Reading /proc/net/arp

Reads the raw ARP table from the kernel and prints every entry.  

```scala
import scala.io.Source

@main def main() =
    val source = Source.fromFile("/proc/net/arp")
    try
        source.getLines().foreach(println)
    finally
        source.close()
end main
```

The `/proc/net/arp` file exposes the kernel's neighbor table for IPv4.  
It contains one line per ARP entry, with the IP address, hardware type,  
flags, MAC address, mask, and interface name as space-separated fields.  
The first line is a header row. The flags field distinguishes complete  
entries (`0x2`) from incomplete ones (`0x0`), which appear when the  
kernel is still waiting for an ARP reply.  

## Parsing ARP table entries

Parses the ARP table and prints each IP address with its MAC and device.  

```scala
import scala.io.Source

case class ArpEntry(ip: String, mac: String, flags: String, dev: String)

@main def main() =
    val source = Source.fromFile("/proc/net/arp")
    try
        val entries = source.getLines()
            .drop(1)
            .map(_.trim.split("""\s+"""))
            .filter(_.length >= 6)
            .map(p => ArpEntry(p(0), p(3), p(2), p(5)))
            .toList
        entries.foreach(e =>
            println(s"${e.ip}  ${e.mac}  flags=${e.flags}  dev=${e.dev}")
        )
    finally
        source.close()
end main
```

The `case class ArpEntry` models one row of the table and makes later  
processing type-safe. `drop(1)` skips the header, and filtering for  
lines with at least six fields guards against unexpected formatting.  
Column index 3 holds the hardware address while index 5 holds the  
interface name. The resulting list can be sorted, filtered, or exported  
to JSON without further parsing.  

## Reading /proc/net/route

Reads the raw IPv4 routing table exported by the kernel.  

```scala
import scala.io.Source

@main def main() =
    val source = Source.fromFile("/proc/net/route")
    try
        source.getLines().foreach(println)
    finally
        source.close()
end main
```

The `/proc/net/route` file lists the kernel's IPv4 Forwarding Information  
Base in a tab-separated format. Addresses and masks appear as eight-digit  
hexadecimal strings stored in little-endian byte order. The flags field  
is also hexadecimal; the bit `0x0001` marks usable routes, `0x0002` marks  
gateway routes, and `0x0004` marks host routes. All numeric values except  
the metric are in hex, which requires conversion before they can be  
displayed as dotted-decimal IP addresses.  

## Parsing routing table entries

Converts hex routing entries to human-readable dotted-decimal form.  

```scala
import scala.io.Source

def hexToIp(hex: String): String =
    val v = java.lang.Long.parseUnsignedLong(hex, 16)
    val b0 = (v & 0xff).toInt
    val b1 = ((v >> 8) & 0xff).toInt
    val b2 = ((v >> 16) & 0xff).toInt
    val b3 = ((v >> 24) & 0xff).toInt
    s"$b0.$b1.$b2.$b3"

@main def main() =
    val source = Source.fromFile("/proc/net/route")
    try
        val lines = source.getLines().toList.drop(1)
        for line <- lines do
            val f = line.trim.split("""\s+""")
            if f.length >= 8 then
                val iface  = f(0)
                val dest   = hexToIp(f(1))
                val gw     = hexToIp(f(2))
                val mask   = hexToIp(f(7))
                println(s"$iface  $dest  via $gw  mask $mask")
    finally
        source.close()
end main
```

Addresses in `/proc/net/route` are 32-bit little-endian integers printed  
as hex. `hexToIp` extracts each octet by shifting the parsed value and  
masking with `0xff`, producing the correct dotted-decimal result. Column  
indices 1, 2, and 7 hold destination, gateway, and mask respectively  
after splitting on whitespace. Skipping the first line removes the column  
header row.  

## Reading /proc/net/tcp

Reads the kernel's IPv4 TCP socket table and prints raw entries.  

```scala
import scala.io.Source

@main def main() =
    val source = Source.fromFile("/proc/net/tcp")
    try
        source.getLines().foreach(println)
    finally
        source.close()
end main
```

The `/proc/net/tcp` file lists every active IPv4 TCP socket maintained  
by the kernel, including sockets in the `LISTEN`, `ESTABLISHED`,  
`TIME_WAIT`, and `CLOSE_WAIT` states. Each row contains the local and  
remote address-port pairs as hexadecimal strings, a two-hex-digit state  
code, transmit and receive queue sizes, and the owning user ID. The state  
codes map directly to the standard TCP state machine defined in RFC 9293.  

## Reading /proc/net/udp

Reads the kernel's IPv4 UDP socket table and prints raw entries.  

```scala
import scala.io.Source

@main def main() =
    val source = Source.fromFile("/proc/net/udp")
    try
        source.getLines().foreach(println)
    finally
        source.close()
end main
```

The `/proc/net/udp` file follows the same column layout as `/proc/net/tcp`  
but lists UDP sockets. Because UDP is connectionless, the remote address  
field is always `00000000:0000` for unconnected sockets. The state column  
is always `07` (CLOSE) for typical UDP sockets since the kernel does not  
maintain a connection-oriented state machine for them. The transmit queue  
column is useful for detecting UDP sockets that are building up a backlog.  

## Parsing TCP connection states

Decodes the hex state codes in `/proc/net/tcp` to named TCP states.  

```scala
import scala.io.Source

val tcpStates = Map(
    "01" -> "ESTABLISHED", "02" -> "SYN_SENT",
    "03" -> "SYN_RECV",    "04" -> "FIN_WAIT1",
    "05" -> "FIN_WAIT2",   "06" -> "TIME_WAIT",
    "07" -> "CLOSE",       "08" -> "CLOSE_WAIT",
    "09" -> "LAST_ACK",    "0A" -> "LISTEN",
    "0B" -> "CLOSING"
)

def hexToIp(hex: String): String =
    val v = java.lang.Long.parseUnsignedLong(hex, 16)
    s"${v & 0xff}.${(v>>8)&0xff}.${(v>>16)&0xff}.${(v>>24)&0xff}"

@main def main() =
    val source = Source.fromFile("/proc/net/tcp")
    try
        source.getLines().drop(1).foreach: line =>
            val f = line.trim.split("""\s+""")
            if f.length >= 4 then
                val Array(localIpHex, localPortHex) = f(1).split(":")
                val localIp   = hexToIp(localIpHex)
                val localPort = Integer.parseInt(localPortHex, 16)
                val state     = tcpStates.getOrElse(f(3).toUpperCase, "UNKNOWN")
                println(s"$localIp:$localPort  $state")
    finally
        source.close()
end main
```

The state code occupies column index 3 in each data row. Upper-casing the  
field before the map lookup ensures that lowercase hex letters match the  
uppercase keys. The local address field is formatted as `IIPHEX:PORTHEX`  
where the IP uses the same little-endian encoding as `/proc/net/route` and  
the port is a big-endian two-byte value. Ports like `0016` decode to 22  
(SSH) and `0050` decode to 80 (HTTP).  

## Parsing UDP socket entries

Extracts local address, port, and queue depth from `/proc/net/udp`.  

```scala
import scala.io.Source

def hexToIp(hex: String): String =
    val v = java.lang.Long.parseUnsignedLong(hex, 16)
    s"${v & 0xff}.${(v>>8)&0xff}.${(v>>16)&0xff}.${(v>>24)&0xff}"

@main def main() =
    val source = Source.fromFile("/proc/net/udp")
    try
        source.getLines().drop(1).foreach: line =>
            val f = line.trim.split("""\s+""")
            if f.length >= 5 then
                val Array(ipHex, portHex) = f(1).split(":")
                val ip    = hexToIp(ipHex)
                val port  = Integer.parseInt(portHex, 16)
                val queues = f(4)  // tx_queue:rx_queue
                println(s"udp  $ip:$port  queues=$queues")
    finally
        source.close()
end main
```

Column index 4 holds a combined `tx_queue:rx_queue` field expressed in  
hex bytes. A non-zero receive queue (`rx_queue`) indicates that the  
application is not reading datagrams fast enough and that kernel buffer  
space is being consumed. Monitoring this value over time helps identify  
UDP receive overruns before they cause packet loss at the socket layer.  

## Reading /proc/net/snmp

Reads the SNMP MIB-II counters exported by the kernel.  

```scala
import scala.io.Source

@main def main() =
    val source = Source.fromFile("/proc/net/snmp")
    try
        source.getLines().foreach(println)
    finally
        source.close()
end main
```

The `/proc/net/snmp` file presents counters for the IP, ICMP, TCP, and  
UDP MIB groups defined in RFC 1213. Each group appears as two consecutive  
lines: the first lists column names and the second lists the corresponding  
values. This format makes it straightforward to parse by reading pairs of  
lines and zipping keys with values. The counters are monotonically  
increasing since the last kernel boot, so delta computations reveal rates.  

## Extracting IP statistics

Parses the IP group counters from `/proc/net/snmp` into a key-value map.  

```scala
import scala.io.Source

@main def main() =
    val source = Source.fromFile("/proc/net/snmp")
    try
        val lines = source.getLines().toList
        val ipIdx = lines.indexWhere(_.startsWith("Ip:"))
        if ipIdx >= 0 && ipIdx + 1 < lines.length then
            val keys   = lines(ipIdx).split("""\s+""").toList
            val values = lines(ipIdx + 1).split("""\s+""").toList
            val stats  = keys.zip(values).drop(1).toMap
            stats.foreach((k, v) => println(s"$k = $v"))
    finally
        source.close()
end main
```

`indexWhere` locates the first line beginning with `"Ip:"`, which is the  
key-names row for the IP group. The following line holds the matching  
values in the same column order. Zipping keys with values and dropping  
the prefix label `"Ip"` builds a clean `Map[String, String]`. Useful  
fields include `InReceives` (total inbound packets), `ForwDatagrams`  
(forwarded packets), and `InDiscards` (packets dropped due to no buffer  
space).  

## Listing network interfaces

Lists every network interface known to the kernel via `/sys/class/net`.  

```scala
import java.nio.file.{Files, Paths}
import scala.jdk.StreamConverters.*

@main def main() =
    val sysnet = Paths.get("/sys/class/net")
    val ifaces = Files.list(sysnet).toScala(List)
    ifaces
        .map(_.getFileName.toString)
        .sorted
        .foreach(println)
end main
```

`/sys/class/net` contains one symbolic link per network interface. Each  
link points to the interface's directory under `/sys/devices`. Calling  
`Files.list` returns a `Stream[Path]` of those links. `getFileName`  
extracts the interface name from the path. Sorting the result makes the  
output deterministic and easier to compare across multiple readings.  

## Reading interface MAC address

Reads the hardware MAC address of an interface from sysfs.  

```scala
import java.nio.file.{Files, Paths}

@main def main() =
    val iface = "eth0"
    val path  = Paths.get(s"/sys/class/net/$iface/address")
    if Files.exists(path) then
        val mac = Files.readString(path).trim
        println(s"$iface  MAC: $mac")
    else
        println(s"Interface $iface not found")
end main
```

Each interface directory under `/sys/class/net` contains an `address` file  
that holds the interface's hardware address as a colon-separated hex  
string such as `aa:bb:cc:dd:ee:ff`. `Files.readString` reads the entire  
file into a `String` in one call, and `trim` removes the trailing newline  
that the kernel appends. Virtual interfaces such as `lo` report all-zero  
addresses, while physical Ethernet interfaces report the burned-in address  
programmed into the NIC firmware.  

## Reading MTU

Reads the maximum transmission unit configured for an interface.  

```scala
import java.nio.file.{Files, Paths}

@main def main() =
    val sysnet = Paths.get("/sys/class/net")
    val dirs   = Files.list(sysnet).toArray.toList
    dirs.foreach: entry =>
        val p = entry.asInstanceOf[java.nio.file.Path]
        val mtuPath = p.resolve("mtu")
        if Files.exists(mtuPath) then
            val name = p.getFileName.toString
            val mtu  = Files.readString(mtuPath).trim
            println(s"$name  MTU=$mtu")
end main
```

The MTU value in `/sys/class/net/<iface>/mtu` is a decimal integer  
representing the largest payload, in bytes, that the interface will  
transmit without fragmentation. The standard Ethernet MTU is 1500. Jumbo  
frames use values up to 9000. The loopback interface typically reports  
65536. Comparing MTU values across interfaces helps diagnose  
fragmentation-induced performance problems on paths where interfaces  
have mismatched MTUs.  

## Reading operational state

Reads the `up` or `down` operational state for every interface.  

```scala
import java.nio.file.{Files, Paths}
import scala.jdk.StreamConverters.*

@main def main() =
    val sysnet = Paths.get("/sys/class/net")
    Files.list(sysnet).toScala(List).foreach: p =>
        val name      = p.getFileName.toString
        val statePath = p.resolve("operstate")
        if Files.exists(statePath) then
            val state = Files.readString(statePath).trim
            println(s"$name  $state")
end main
```

The `operstate` file contains a single word such as `up`, `down`,  
`unknown`, `dormant`, or `lowerlayerdown`. An `unknown` state typically  
indicates that the driver does not report RFC 2863 operational state and  
should be treated as potentially active. `lowerlayerdown` means a  
physical dependency (such as a bonding parent) is unavailable. Monitoring  
these values over time can reveal flapping interfaces that cycle  
repeatedly between `up` and `down`.  

## Reading speed and duplex

Reads the negotiated speed and duplex setting for a physical interface.  

```scala
import java.nio.file.{Files, Paths}

@main def main() =
    val iface  = "eth0"
    val base   = Paths.get(s"/sys/class/net/$iface")
    val sPath  = base.resolve("speed")
    val dPath  = base.resolve("duplex")
    val speed  = if Files.exists(sPath) then
        Files.readString(sPath).trim else "n/a"
    val duplex = if Files.exists(dPath) then
        Files.readString(dPath).trim else "n/a"
    println(s"$iface  speed=${speed}Mb/s  duplex=$duplex")
end main
```

The `speed` file contains an integer representing the current link speed  
in megabits per second, such as `1000` for gigabit Ethernet. The `duplex`  
file contains `full` or `half`. Both files may be absent or may return  
`-1` for virtual interfaces, tunnel devices, and wireless adapters that  
do not support ethtool-style attribute reporting. Checking for file  
existence before reading prevents spurious `IOException` for such cases.  

## Reading interface flags

Reads the hex flags word that describes interface capabilities.  

```scala
import java.nio.file.{Files, Paths}
import scala.jdk.StreamConverters.*

val IFF_UP        = 0x0001
val IFF_LOOPBACK  = 0x0008
val IFF_MULTICAST = 0x1000

@main def main() =
    val sysnet = Paths.get("/sys/class/net")
    Files.list(sysnet).toScala(List).foreach: p =>
        val name  = p.getFileName.toString
        val fPath = p.resolve("flags")
        if Files.exists(fPath) then
            val raw   = Files.readString(fPath).trim
            val flags = java.lang.Long.parseUnsignedLong(raw.stripPrefix("0x"), 16)
            val up    = if (flags & IFF_UP)        != 0 then "UP"        else "DOWN"
            val loop  = if (flags & IFF_LOOPBACK)  != 0 then "LOOPBACK"  else ""
            val mcast = if (flags & IFF_MULTICAST) != 0 then "MULTICAST" else ""
            println(s"$name  $up $loop $mcast".trim)
end main
```

The `flags` file exposes the `net_device` flags word as a `0x`-prefixed  
hexadecimal string. Each bit corresponds to a constant defined in  
`<linux/if.h>`. `IFF_UP` means the interface has been administratively  
enabled. `IFF_LOOPBACK` identifies the loopback device. `IFF_MULTICAST`  
indicates that the interface can send and receive multicast frames.  
Parsing the word directly avoids a dependency on `ioctl` and works  
from unprivileged user space.  

## Reading IPv4 addresses

Enumerates all IPv4 addresses assigned to every interface.  

```scala
import java.net.NetworkInterface
import scala.jdk.CollectionConverters.*

@main def main() =
    val ifaces = NetworkInterface.getNetworkInterfaces
    if ifaces != null then
        ifaces.asScala.foreach: iface =>
            iface.getInterfaceAddresses.asScala
                .filter(_.getAddress.isInstanceOf[java.net.Inet4Address])
                .foreach: ia =>
                    val addr   = ia.getAddress.getHostAddress
                    val prefix = ia.getNetworkPrefixLength
                    println(s"${iface.getName}  $addr/$prefix")
end main
```

`NetworkInterface.getNetworkInterfaces` returns an enumeration of all  
interfaces visible to the JVM. `getInterfaceAddresses` returns both IPv4  
and IPv6 bindings as `InterfaceAddress` objects, each carrying the IP  
address and its prefix length. Filtering for `Inet4Address` instances  
isolates the IPv4 bindings. The prefix length replaces the older  
dotted-decimal subnet mask notation and allows direct CIDR display.  

## Reading IPv6 addresses

Lists all IPv6 addresses by parsing `/proc/net/if_inet6`.  

```scala
import scala.io.Source

def parseIpv6(raw: String): String =
    raw.grouped(4).mkString(":")

@main def main() =
    val source = Source.fromFile("/proc/net/if_inet6")
    try
        source.getLines().foreach: line =>
            val f      = line.trim.split("""\s+""")
            if f.length >= 6 then
                val addr   = parseIpv6(f(0))
                val prefix = Integer.parseInt(f(2), 16)
                val iface  = f(5)
                println(s"$iface  $addr/$prefix")
    finally
        source.close()
end main
```

`/proc/net/if_inet6` lists every IPv6 address on every interface as a  
32-character hex string without separators, followed by the interface  
index, prefix length in hex, scope, flags, and interface name. Grouping  
the raw string into four-character chunks and joining them with colons  
produces standard colon-separated notation. The scope byte distinguishes  
link-local (`0x20`), site-local (`0x40`), and global (`0x00`) addresses.  

## Reading queue lengths

Reads the transmit queue length configured for every interface.  

```scala
import java.nio.file.{Files, Paths}
import scala.jdk.StreamConverters.*

@main def main() =
    val sysnet = Paths.get("/sys/class/net")
    Files.list(sysnet).toScala(List).foreach: p =>
        val name  = p.getFileName.toString
        val qPath = p.resolve("tx_queue_len")
        if Files.exists(qPath) then
            val qlen = Files.readString(qPath).trim
            println(s"$name  tx_queue_len=$qlen")
end main
```

The `tx_queue_len` attribute controls how many packets the kernel will  
queue in the transmit ring before dropping. The default for Ethernet  
interfaces is 1000, while loopback typically reports 1000 as well.  
Increasing the queue length can smooth bursty traffic but also increases  
the latency experienced by older packets in the queue. Monitoring this  
value alongside the `tx_dropped` counter reveals whether the queue is  
being exhausted under load.  

## Reading statistics from sysfs

Reads per-interface statistics from `/sys/class/net/<iface>/statistics/`.  

```scala
import java.nio.file.{Files, Paths}
import scala.jdk.StreamConverters.*

@main def main() =
    val iface   = "eth0"
    val statDir = Paths.get(s"/sys/class/net/$iface/statistics")
    if Files.isDirectory(statDir) then
        Files.list(statDir).toScala(List)
            .sortBy(_.getFileName.toString)
            .foreach: p =>
                val name  = p.getFileName.toString
                val value = Files.readString(p).trim
                println(s"$name = $value")
end main
```

The `statistics` subdirectory under each interface directory in sysfs  
exposes the same counters as `/proc/net/dev` but as individual files,  
one counter per file. Available counters include `rx_bytes`, `tx_bytes`,  
`rx_packets`, `tx_packets`, `rx_errors`, `tx_errors`, `rx_dropped`, and  
`tx_dropped`. Reading individual files is more efficient when only a  
subset of counters is needed, because the kernel does not need to  
serialise the full interface table.  

## Parsing the kernel routing table

Parses `/proc/net/route` into structured route records.  

```scala
import scala.io.Source

case class Route(
    iface: String, dest: String, gateway: String,
    flags: Int, metric: Int, mask: String)

def hexToIp(hex: String): String =
    val v = java.lang.Long.parseUnsignedLong(hex, 16)
    s"${v&0xff}.${(v>>8)&0xff}.${(v>>16)&0xff}.${(v>>24)&0xff}"

@main def main() =
    val source = Source.fromFile("/proc/net/route")
    try
        val routes = source.getLines().drop(1).flatMap: line =>
            val f = line.trim.split("""\s+""")
            if f.length >= 11 then
                Some(Route(
                    iface   = f(0),
                    dest    = hexToIp(f(1)),
                    gateway = hexToIp(f(2)),
                    flags   = Integer.parseInt(f(3), 16),
                    metric  = f(6).toInt,
                    mask    = hexToIp(f(7))
                ))
            else None
        .toList
        routes.foreach(r =>
            println(s"${r.dest}  via ${r.gateway}  dev ${r.iface}" +
                    s"  metric ${r.metric}  mask ${r.mask}"))
    finally
        source.close()
end main
```

Wrapping each row in a `case class` makes downstream manipulation  
type-safe and avoids repeated index lookups. `flatMap` with `Some/None`  
cleanly skips malformed lines. Column indices follow the kernel's fixed  
layout: 0 = interface, 1 = destination, 2 = gateway, 3 = flags,  
6 = metric, 7 = mask. The flags integer encodes usability, gateway  
presence, and host-route status as individual bits.  

## Determining the default gateway

Finds the default gateway by locating the route with destination `0.0.0.0`.  

```scala
import scala.io.Source

def hexToIp(hex: String): String =
    val v = java.lang.Long.parseUnsignedLong(hex, 16)
    s"${v&0xff}.${(v>>8)&0xff}.${(v>>16)&0xff}.${(v>>24)&0xff}"

@main def main() =
    val source = Source.fromFile("/proc/net/route")
    try
        val default = source.getLines().drop(1).flatMap: line =>
            val f = line.trim.split("""\s+""")
            if f.length >= 8 && f(1) == "00000000" then
                Some((hexToIp(f(2)), f(0)))
            else None
        .toList
        default match
            case (gw, iface) :: _ =>
                println(s"Default gateway: $gw  via $iface")
            case Nil =>
                println("No default gateway found")
    finally
        source.close()
end main
```

The default gateway route always has a destination of `00000000`  
(hexadecimal for `0.0.0.0`) and a mask of `00000000`. When multiple  
default routes exist, the kernel selects among them by metric; the  
example takes the first match. The gateway IP address printed is the  
next-hop router that the kernel will use for traffic not matching any  
more-specific prefix.  

## Determining the interface for a destination

Finds which interface would be used to reach a given IP address.  

```scala
import scala.io.Source

def hexToIp(hex: String): String =
    val v = java.lang.Long.parseUnsignedLong(hex, 16)
    s"${v&0xff}.${(v>>8)&0xff}.${(v>>16)&0xff}.${(v>>24)&0xff}"

def ipToLong(ip: String): Long =
    val parts = ip.split("\\.")
    parts.foldLeft(0L)((acc, p) => (acc << 8) | p.toLong)

@main def main() =
    val target = "192.168.1.50"
    val tLong  = ipToLong(target)
    val source = Source.fromFile("/proc/net/route")
    try
        val match_ = source.getLines().drop(1).flatMap: line =>
            val f = line.trim.split("""\s+""")
            if f.length >= 8 then
                val dest = java.lang.Long.parseUnsignedLong(f(1), 16)
                val mask = java.lang.Long.parseUnsignedLong(f(7), 16)
                val destIp = dest & mask
                val targIp = tLong & mask
                if destIp == targIp then Some((f(0), hexToIp(f(2))))
                else None
            else None
        .toList
        match_.headOption match
            case Some((iface, gw)) =>
                println(s"Route to $target: dev $iface via $gw")
            case None =>
                println(s"No route to $target")
    finally
        source.close()
end main
```

Determining the outgoing interface for a destination mirrors what the  
kernel does internally during routing. For each candidate route, the  
destination prefix is computed by AND-ing the route's destination with  
its mask. The same mask is applied to the target address. If both  
produce the same result, the route matches. Because the kernel stores  
values in little-endian byte order while `ipToLong` uses network byte  
order, both sides of the comparison must use the same representation,  
which here is the raw hex value from the file.  

## Computing CIDR masks

Converts a little-endian hex mask from `/proc/net/route` to CIDR length.  

```scala
@main def main() =
    val masks = List(
        "00000000",  // /0  default
        "000000FF",  // /8
        "0000FFFF",  // /16
        "00FFFFFF",  // /24
        "FFFFFFFF"   // /32
    )
    masks.foreach: hex =>
        val v    = java.lang.Long.parseUnsignedLong(hex, 16)
        val cidr = java.lang.Integer.bitCount(v.toInt)
        val b0   = (v & 0xff).toInt
        val b1   = ((v >> 8)  & 0xff).toInt
        val b2   = ((v >> 16) & 0xff).toInt
        val b3   = ((v >> 24) & 0xff).toInt
        println(s"$hex  =>  $b0.$b1.$b2.$b3  /$cidr")
end main
```

The CIDR prefix length equals the number of set bits in the subnet mask.  
`Integer.bitCount` counts these bits in a single instruction. The  
little-endian byte order used in `/proc/net/route` means that octet 0 of  
the dotted-decimal representation is held in the least significant byte  
of the integer, which is why `b0 = v & 0xff` gives the first octet  
correctly. The table above confirms that `00FFFFFF` decodes to  
`255.255.255.0` (a `/24`) as expected.  

## Matching routes by prefix

Tests whether a given IP address falls within a route's prefix.  

```scala
@main def main() =
    def inPrefix(addrDot: String, destHex: String, maskHex: String): Boolean =
        def dotToLong(ip: String): Long =
            val p = ip.split("\\.")
            ((p(0).toLong << 24) | (p(1).toLong << 16) |
             (p(2).toLong << 8)  | p(3).toLong)
        val dest = java.lang.Long.parseUnsignedLong(destHex, 16)
        val mask = java.lang.Long.parseUnsignedLong(maskHex, 16)
        val addr = dotToLong(addrDot)
        // Convert dest from little-endian to network byte order
        val destNbo = java.lang.Integer.reverseBytes(dest.toInt).toLong & 0xFFFFFFFFL
        val maskNbo = java.lang.Integer.reverseBytes(mask.toInt).toLong & 0xFFFFFFFFL
        (addr & maskNbo) == (destNbo & maskNbo)

    println(inPrefix("192.168.1.50",  "0001A8C0", "00FFFFFF"))  // true
    println(inPrefix("10.0.0.1",      "0001A8C0", "00FFFFFF"))  // false
    println(inPrefix("0.0.0.0",       "00000000", "00000000"))  // true (default)
end main
```

Prefix matching requires that the mask be applied to both the candidate  
address and the route destination before comparing them. Because  
`/proc/net/route` stores values in little-endian host byte order while  
dotted-decimal addresses are expressed in network byte order, calling  
`Integer.reverseBytes` realigns the byte ordering before the comparison.  
This function mirrors the kernel's longest-prefix-match logic and can be  
used to simulate routing decisions in user space.  

## Conceptual longest-prefix match

Finds the most specific matching route for a destination address.  

```scala
import scala.io.Source

case class Route(iface: String, destHex: String, gwHex: String,
                 maskHex: String, metric: Int)

def hexToIp(h: String): String =
    val v = java.lang.Long.parseUnsignedLong(h, 16)
    s"${v&0xff}.${(v>>8)&0xff}.${(v>>16)&0xff}.${(v>>24)&0xff}"

def prefixLen(maskHex: String): Int =
    java.lang.Integer.bitCount(
        java.lang.Long.parseUnsignedLong(maskHex, 16).toInt)

@main def main() =
    val target = 0xC0A80132L  // 192.168.1.50 in network byte order
    val source = Source.fromFile("/proc/net/route")
    try
        val routes = source.getLines().drop(1).flatMap: line =>
            val f = line.trim.split("""\s+""")
            if f.length >= 11 then Some(Route(f(0), f(1), f(2), f(7), f(6).toInt))
            else None
        .toList
        val best = routes
            .filter: r =>
                val d = java.lang.Integer.reverseBytes(
                    java.lang.Long.parseUnsignedLong(r.destHex,16).toInt
                ).toLong & 0xFFFFFFFFL
                val m = java.lang.Integer.reverseBytes(
                    java.lang.Long.parseUnsignedLong(r.maskHex,16).toInt
                ).toLong & 0xFFFFFFFFL
                (target & m) == (d & m)
            .maxByOption(r => (prefixLen(r.maskHex), -r.metric))
        best match
            case Some(r) =>
                println(s"Best route: ${hexToIp(r.destHex)}/${prefixLen(r.maskHex)}" +
                        s"  via ${hexToIp(r.gwHex)}  dev ${r.iface}")
            case None => println("No route found")
    finally
        source.close()
end main
```

Longest-prefix match selects the route whose mask has the highest CIDR  
value among all routes that match the target address. When two routes  
have equal prefix lengths, the one with the lower metric wins. The  
`maxByOption` call sorts first by prefix length (descending) and then  
by negated metric (also effectively descending), so the best route  
bubbles to the top. This replicates the kernel's routing algorithm in  
pure Scala without any system calls.  

## Reading IPv6 routing entries

Parses `/proc/net/ipv6_route` and prints each prefix with its interface.  

```scala
import scala.io.Source

def parseIpv6Addr(raw: String): String =
    raw.grouped(4).mkString(":")

@main def main() =
    val source = Source.fromFile("/proc/net/ipv6_route")
    try
        source.getLines().foreach: line =>
            val f = line.trim.split("""\s+""")
            if f.length >= 10 then
                val dest   = parseIpv6Addr(f(0))
                val plen   = Integer.parseInt(f(1), 16)
                val nexthop = parseIpv6Addr(f(4))
                val metric = Integer.parseInt(f(5), 16)
                val iface  = f(9)
                println(s"$dest/$plen  via $nexthop  dev $iface  metric $metric")
    finally
        source.close()
end main
```

`/proc/net/ipv6_route` encodes IPv6 addresses as 32-character hex strings  
without separators, and prefix lengths as two-character hex values.  
`parseIpv6Addr` groups characters into four-character blocks and joins  
them with colons, producing the standard colon-separated representation.  
A destination of all zeros with prefix length 0 represents the IPv6  
default route (`::/0`). The nexthop `00000000000000000000000000000000`  
indicates that the route is directly connected.  

## Detecting unreachable routes

Identifies routes marked with the reject flag in the routing table.  

```scala
import scala.io.Source

val RTF_REJECT = 0x0200

def hexToIp(h: String): String =
    val v = java.lang.Long.parseUnsignedLong(h, 16)
    s"${v&0xff}.${(v>>8)&0xff}.${(v>>16)&0xff}.${(v>>24)&0xff}"

@main def main() =
    val source = Source.fromFile("/proc/net/route")
    try
        source.getLines().drop(1).foreach: line =>
            val f = line.trim.split("""\s+""")
            if f.length >= 4 then
                val flags = Integer.parseInt(f(3), 16)
                if (flags & RTF_REJECT) != 0 then
                    println(s"UNREACHABLE: ${hexToIp(f(1))}  dev ${f(0)}")
    finally
        source.close()
end main
```

The `RTF_REJECT` flag (`0x0200`) marks routes that should cause the  
kernel to send an ICMP "destination unreachable" message rather than  
forwarding the packet. Such routes are typically added intentionally to  
create policy-based black holes or to prevent traffic from leaking to  
an unintended interface. Detecting these routes programmatically is  
useful for auditing routing policy or confirming that black-hole routes  
are still installed after a configuration change.  

## Parsing ARP entries

Reads and structures every entry in the kernel's ARP table.  

```scala
import scala.io.Source

case class ArpEntry(ip: String, hwType: String, flags: String,
                    mac: String, mask: String, dev: String)

@main def main() =
    val source = Source.fromFile("/proc/net/arp")
    try
        val entries = source.getLines().drop(1).flatMap: line =>
            val f = line.trim.split("""\s+""")
            if f.length >= 6 then
                Some(ArpEntry(f(0), f(1), f(2), f(3), f(4), f(5)))
            else None
        .toList
        entries.foreach: e =>
            println(s"${e.ip}  ${e.mac}  flags=${e.flags}  dev=${e.dev}")
    finally
        source.close()
end main
```

The ARP table in `/proc/net/arp` has six columns: IP address, hardware  
type (`0x1` for Ethernet), flags, MAC address, mask (always `*`), and  
device name. Modelling the row as a `case class` enables sorting by IP,  
grouping by device, and filtering by flags without repeated index  
lookups. The flags field is the most important for diagnostics: `0x2`  
indicates a complete entry while `0x0` means the kernel has not yet  
received a reply.  

## Detecting incomplete ARP entries

Finds ARP entries whose resolution is still pending.  

```scala
import scala.io.Source

@main def main() =
    val source = Source.fromFile("/proc/net/arp")
    try
        val incomplete = source.getLines().drop(1).filter: line =>
            val f = line.trim.split("""\s+""")
            f.length >= 3 && f(2) == "0x0"
        .toList
        if incomplete.isEmpty then
            println("No incomplete ARP entries")
        else
            println("Incomplete ARP entries:")
            incomplete.foreach(println)
    finally
        source.close()
end main
```

An incomplete ARP entry appears when the kernel has sent an ARP request  
but has not yet received a reply, or when the entry has expired and is  
being refreshed. The flags field is `0x0` in these cases. Incomplete  
entries are normal in transient situations but may indicate unreachable  
hosts, VLAN misconfigurations, or blocked ARP traffic when they persist  
for more than a few seconds. Automated detection of such entries is  
valuable in network health monitoring scripts.  

## Mapping IP to MAC

Builds a lookup map from IP address to MAC address using the ARP table.  

```scala
import scala.io.Source

@main def main() =
    val source = Source.fromFile("/proc/net/arp")
    try
        val ipToMac: Map[String, String] =
            source.getLines().drop(1).flatMap: line =>
                val f = line.trim.split("""\s+""")
                if f.length >= 4 && f(2) == "0x2" then Some(f(0) -> f(3))
                else None
            .toMap
        ipToMac.foreach: (ip, mac) =>
            println(s"$ip  =>  $mac")
    finally
        source.close()
end main
```

Filtering for entries with flags `0x2` ensures that only fully resolved  
entries appear in the map. Attempting to include incomplete entries would  
associate the placeholder MAC `00:00:00:00:00:00` with real IP addresses,  
which would cause incorrect results in any code that consumes the map.  
This map can be used to correlate TCP connections (which show only IPs)  
with their layer-2 hardware addresses for network forensics or auditing.  

## Detecting duplicate MACs

Groups ARP entries by MAC address and reports any that map to multiple IPs.  

```scala
import scala.io.Source

@main def main() =
    val source = Source.fromFile("/proc/net/arp")
    try
        val macToIps: Map[String, List[String]] =
            source.getLines().drop(1).flatMap: line =>
                val f = line.trim.split("""\s+""")
                if f.length >= 4 && f(2) == "0x2" then Some(f(3) -> f(0))
                else None
            .toList
            .groupMap(_._1)(_._2)
        macToIps
            .filter(_._2.size > 1)
            .foreach: (mac, ips) =>
                println(s"Duplicate MAC $mac seen on: ${ips.mkString(", ")}")
    finally
        source.close()
end main
```

A MAC address that appears for more than one IP address is a strong  
indicator of ARP spoofing, IP address conflicts, or misconfigured  
virtual machines sharing a hardware address. `groupMap` is the idiomatic  
Scala 2.13+/3 way to build a `Map[K, List[V]]` from a list of pairs in  
a single pass. The filter step removes MACs that appear only once,  
leaving only the suspicious duplicates.  

## Refreshing the ARP cache

Demonstrates how to trigger ARP resolution by opening a TCP connection.  

```scala
import java.net.{InetAddress, Socket}
import java.io.IOException

@main def main() =
    val host = "192.168.1.1"
    println(s"Attempting ARP refresh for $host ...")
    try
        // Opening a socket causes the kernel to resolve the MAC via ARP
        val sock = new Socket()
        sock.connect(
            new java.net.InetSocketAddress(host, 80),
            500  // 500 ms timeout
        )
        sock.close()
        println(s"Connection succeeded; ARP entry should now be complete")
    catch
        case _: IOException =>
            println(s"Connection failed, but kernel may still have updated ARP")
    // The ARP table is now updated; read it to confirm
    val source = scala.io.Source.fromFile("/proc/net/arp")
    try
        source.getLines().filter(_.startsWith(host)).foreach(println)
    finally
        source.close()
end main
```

The Linux kernel automatically resolves layer-2 addresses when a socket  
attempts to reach a destination. Even if the TCP connection is refused  
or times out, the ARP exchange happens transparently at the kernel level  
before the TCP SYN is sent. This technique forces an ARP entry to become  
complete without requiring raw socket privileges or calling `arping`.  
Reading `/proc/net/arp` immediately after confirms the refreshed entry.  

## Checking if an interface is up

Uses the Java `NetworkInterface` API to test administrative status.  

```scala
import java.net.NetworkInterface

@main def main() =
    val ifaces = NetworkInterface.getNetworkInterfaces
    if ifaces != null then
        val it = ifaces.asInstanceOf[java.util.Enumeration[NetworkInterface]]
        import scala.jdk.CollectionConverters.*
        NetworkInterface.getNetworkInterfaces.asScala.foreach: iface =>
            val up   = iface.isUp
            val name = iface.getName
            println(s"$name  ${if up then "UP" else "DOWN"}")
end main
```

`NetworkInterface.isUp()` delegates to the `SIOCGIFFLAGS` ioctl  
internally and checks the `IFF_UP` flag without requiring file  
operations. An interface that is administratively up but has no carrier  
(unplugged cable) is still reported as `up` by this method. For a more  
accurate physical link state, reading `operstate` from sysfs is  
preferable, since it distinguishes between administrative and operational  
states according to the RFC 2863 model.  

## Checking multicast support

Determines which interfaces support multicast transmission.  

```scala
import java.net.NetworkInterface
import scala.jdk.CollectionConverters.*

@main def main() =
    NetworkInterface.getNetworkInterfaces.asScala.foreach: iface =>
        val mcast = iface.supportsMulticast
        val name  = iface.getName
        if mcast then
            println(s"$name  MULTICAST")
end main
```

`supportsMulticast()` checks the `IFF_MULTICAST` flag of the interface.  
Multicast-capable interfaces can join multicast groups and send or  
receive multicast datagrams. Loopback, Ethernet, and Wi-Fi interfaces  
typically support multicast, while point-to-point tunnel devices may  
not. Applications that use IP multicast for service discovery (such as  
mDNS or SSDP) must enumerate only multicast-capable interfaces to avoid  
binding errors.  

## Reading interface addresses via NetworkInterface

Lists all addresses attached to every interface using the Java API.  

```scala
import java.net.NetworkInterface
import scala.jdk.CollectionConverters.*

@main def main() =
    NetworkInterface.getNetworkInterfaces.asScala.foreach: iface =>
        val addrs = iface.getInterfaceAddresses.asScala
        if addrs.nonEmpty then
            println(s"${iface.getName}:")
            addrs.foreach: ia =>
                val ip     = ia.getAddress.getHostAddress
                val prefix = ia.getNetworkPrefixLength
                val bcast  = Option(ia.getBroadcast)
                    .map(_.getHostAddress).getOrElse("n/a")
                println(s"  inet $ip/$prefix  brd $bcast")
end main
```

`InterfaceAddress` bundles the IP address, its network prefix length, and  
the broadcast address (for IPv4) into a single object. IPv6 addresses do  
not have a broadcast address, so `getBroadcast` returns `null` for them.  
Using `Option` wraps the nullable Java method result safely. This example  
mirrors the output of `ip addr show` and provides an accurate view of  
the host's addressing configuration without parsing any text files.  

## Reading hardware address via NetworkInterface

Reads the raw hardware address bytes and formats them as a MAC string.  

```scala
import java.net.NetworkInterface
import scala.jdk.CollectionConverters.*

@main def main() =
    NetworkInterface.getNetworkInterfaces.asScala.foreach: iface =>
        val hw = iface.getHardwareAddress
        if hw != null && hw.nonEmpty then
            val mac = hw.map(b => f"${b & 0xff}%02x").mkString(":")
            println(s"${iface.getName}  $mac")
end main
```

`getHardwareAddress()` returns a `Array[Byte]` of the raw MAC address  
bytes. Each byte is masked with `0xff` to convert the signed Java byte  
to an unsigned integer before formatting it with the `%02x` format  
specifier, which zero-pads single-digit hex values. The call returns  
`null` for virtual interfaces such as loopback, `tun`, and `veth` devices  
that do not have a hardware address. Null-checking before use prevents  
`NullPointerException` on those interfaces.  

## Enumerating all interfaces

Collects all network interfaces into a structured list for further use.  

```scala
import java.net.NetworkInterface
import scala.jdk.CollectionConverters.*

case class IfaceInfo(name: String, index: Int, up: Boolean,
                     loopback: Boolean, virtual: Boolean)

@main def main() =
    val ifaces = NetworkInterface.getNetworkInterfaces.asScala.map: iface =>
        IfaceInfo(
            name     = iface.getName,
            index    = iface.getIndex,
            up       = iface.isUp,
            loopback = iface.isLoopback,
            virtual  = iface.isVirtual
        )
    .toList
    ifaces.sortBy(_.index).foreach: i =>
        println(s"${i.index}  ${i.name.padTo(12, ' ')}  " +
                s"up=${i.up}  lo=${i.loopback}  virt=${i.virtual}")
end main
```

Collecting interface metadata into a `case class` separates the  
enumeration step from the analysis step and makes the data easy to  
serialize, filter, or pass to other functions. `getIndex` returns the  
numeric interface index as used in socket binding and Netlink messages.  
Sorting by index ensures a stable, consistent ordering that matches the  
order reported by `ip link show`. This snapshot can be compared against  
a previous snapshot to detect interface additions or removals.  

## Filtering interfaces by type

Separates loopback, virtual, and physical interfaces into distinct groups.  

```scala
import java.net.NetworkInterface
import scala.jdk.CollectionConverters.*

@main def main() =
    val all = NetworkInterface.getNetworkInterfaces.asScala.toList

    val loopbacks = all.filter(_.isLoopback)
    val virtuals  = all.filter(i => !i.isLoopback && i.isVirtual)
    val physical  = all.filter(i => !i.isLoopback && !i.isVirtual)

    println("Loopback:")
    loopbacks.foreach(i => println(s"  ${i.getName}"))
    println("Virtual:")
    virtuals.foreach(i => println(s"  ${i.getName}"))
    println("Physical:")
    physical.foreach(i => println(s"  ${i.getName}"))
end main
```

`isLoopback` identifies the loopback interface (`lo`). `isVirtual`  
identifies logical sub-interfaces and VLAN-tagged interfaces whose names  
typically contain a colon or a dot, such as `eth0:1` or `eth0.100`.  
Physical interfaces are those that are neither loopback nor virtual, and  
they correspond to real network adapters. This classification is useful  
when applying routing rules, firewall policies, or monitoring thresholds  
that should differ by interface category.  

## Opening a Netlink socket

Uses Project Panama to call the C `socket()` function and open a  
raw Netlink socket.  

```scala
import java.lang.foreign.*
import java.lang.foreign.ValueLayout.*

val AF_NETLINK   = 16
val SOCK_RAW     = 3
val NETLINK_ROUTE = 0

@main def main() =
    val linker  = Linker.nativeLinker()
    val lookup  = linker.defaultLookup()
    val socketH = linker.downcallHandle(
        lookup.find("socket").get(),
        FunctionDescriptor.of(JAVA_INT, JAVA_INT, JAVA_INT, JAVA_INT)
    )
    val closeH = linker.downcallHandle(
        lookup.find("close").get(),
        FunctionDescriptor.ofVoid(JAVA_INT)
    )
    val fd = socketH.invoke(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE)
        .asInstanceOf[Int]
    if fd < 0 then
        println("Failed to open Netlink socket")
    else
        println(s"Netlink socket fd=$fd")
        closeH.invoke(fd)
        println("Socket closed")
end main
```

`Linker.nativeLinker()` resolves symbols from the process's C runtime.  
`downcallHandle` binds a specific C function to a `MethodHandle` by  
describing its signature through a `FunctionDescriptor`. Constants  
`AF_NETLINK=16`, `SOCK_RAW=3`, and `NETLINK_ROUTE=0` match the values  
defined in `<linux/netlink.h>` and `<sys/socket.h>`. Running this code  
requires JVM flags `--enable-native-access=ALL-UNNAMED` and Java 21 or  
later. The file descriptor must always be closed to prevent resource  
leaks.  

## Sending a Netlink request

Constructs an `RTM_GETROUTE` request and sends it over a Netlink socket.  

```scala
import java.lang.foreign.*
import java.lang.foreign.ValueLayout.*

@main def main() =
    val AF_NETLINK    = 16;  val SOCK_RAW      = 3
    val NETLINK_ROUTE = 0;   val RTM_GETROUTE  = 26
    val NLM_F_REQUEST = 0x01; val NLM_F_DUMP   = 0x300
    val AF_UNSPEC     = 0

    val linker  = Linker.nativeLinker()
    val lookup  = linker.defaultLookup()
    val socketH = linker.downcallHandle(lookup.find("socket").get(),
        FunctionDescriptor.of(JAVA_INT, JAVA_INT, JAVA_INT, JAVA_INT))
    val sendH   = linker.downcallHandle(lookup.find("send").get(),
        FunctionDescriptor.of(JAVA_LONG, JAVA_INT, ADDRESS, JAVA_LONG, JAVA_INT))
    val closeH  = linker.downcallHandle(lookup.find("close").get(),
        FunctionDescriptor.ofVoid(JAVA_INT))

    val fd = socketH.invoke(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE).asInstanceOf[Int]
    Arena.ofConfined().use: arena =>
        // nlmsghdr (16 bytes) + rtgenmsg (1 byte) padded to 20
        val buf = arena.allocate(20)
        buf.set(JAVA_INT,   0,  20)                          // nlmsg_len
        buf.set(JAVA_SHORT, 4,  RTM_GETROUTE.toShort)        // nlmsg_type
        buf.set(JAVA_SHORT, 6,  (NLM_F_REQUEST | NLM_F_DUMP).toShort) // flags
        buf.set(JAVA_INT,   8,  1)                           // nlmsg_seq
        buf.set(JAVA_INT,   12, 0)                           // nlmsg_pid
        buf.set(JAVA_BYTE,  16, AF_UNSPEC.toByte)            // rtgen_family
        val sent = sendH.invoke(fd, buf, 20L, 0).asInstanceOf[Long]
        println(s"Sent $sent bytes to kernel")
    closeH.invoke(fd)
end main
```

The Netlink message begins with a fixed 16-byte `nlmsghdr` followed by  
a protocol-specific payload. For `RTM_GETROUTE`, the payload is a single  
`rtgenmsg` byte specifying the address family. `NLM_F_DUMP` requests all  
entries rather than a single lookup. `Arena.ofConfined()` allocates  
20 bytes of native memory that is automatically freed when the arena  
scope exits. Byte offsets are used to write each field directly into  
the `MemorySegment` buffer, avoiding the need for a generated struct  
accessor class.  

## Receiving a Netlink response

Reads the kernel's multipart response from a Netlink socket.  

```scala
import java.lang.foreign.*
import java.lang.foreign.ValueLayout.*

@main def main() =
    val AF_NETLINK = 16; val SOCK_RAW = 3; val NETLINK_ROUTE = 0
    val NLMSG_DONE = 3;  val NLMSG_ERROR = 2

    val linker = Linker.nativeLinker()
    val lookup = linker.defaultLookup()
    val socketH = linker.downcallHandle(lookup.find("socket").get(),
        FunctionDescriptor.of(JAVA_INT, JAVA_INT, JAVA_INT, JAVA_INT))
    val recvH = linker.downcallHandle(lookup.find("recv").get(),
        FunctionDescriptor.of(JAVA_LONG, JAVA_INT, ADDRESS, JAVA_LONG, JAVA_INT))
    val closeH = linker.downcallHandle(lookup.find("close").get(),
        FunctionDescriptor.ofVoid(JAVA_INT))

    val fd = socketH.invoke(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE).asInstanceOf[Int]
    Arena.ofConfined().use: arena =>
        val buf  = arena.allocate(32768)
        var done = false
        while !done do
            val n = recvH.invoke(fd, buf, 32768L, 0).asInstanceOf[Long]
            if n <= 0 then done = true
            else
                var offset = 0L
                while offset + 16 <= n do
                    val msgLen  = buf.get(JAVA_INT,   offset)
                    val msgType = buf.get(JAVA_SHORT, offset + 4)
                    println(s"nlmsg: len=$msgLen type=$msgType")
                    if msgType == NLMSG_DONE || msgType == NLMSG_ERROR then
                        done = true
                    offset += Math.max(msgLen.toLong, 16L)
    closeH.invoke(fd)
end main
```

Netlink responses arrive as multipart messages, each with its own  
`nlmsghdr`. The loop processes as many headers as fit in the received  
buffer, advancing `offset` by the aligned length of each message.  
A `NLMSG_DONE` type terminates the multipart sequence, while  
`NLMSG_ERROR` signals that the kernel encountered a problem. The 32 KiB  
receive buffer is large enough for typical routing and link table  
dumps. In production code, the `send` step from the previous example  
would precede this receive loop.  

## Parsing Netlink message headers

Extracts fields from raw `nlmsghdr` bytes in a response buffer.  

```scala
import java.lang.foreign.*
import java.lang.foreign.ValueLayout.*

case class NlMsgHdr(len: Int, msgType: Short, flags: Short,
                    seq: Int, pid: Int)

def parseNlMsgHdr(seg: MemorySegment, offset: Long): NlMsgHdr =
    NlMsgHdr(
        len     = seg.get(JAVA_INT,   offset),
        msgType = seg.get(JAVA_SHORT, offset + 4),
        flags   = seg.get(JAVA_SHORT, offset + 6),
        seq     = seg.get(JAVA_INT,   offset + 8),
        pid     = seg.get(JAVA_INT,   offset + 12)
    )

@main def main() =
    Arena.ofConfined().use: arena =>
        val buf = arena.allocate(16)
        buf.set(JAVA_INT,   0,  16.toInt)      // nlmsg_len
        buf.set(JAVA_SHORT, 4,  3.toShort)     // NLMSG_DONE
        buf.set(JAVA_SHORT, 6,  0x0300.toShort) // NLM_F_MULTI
        buf.set(JAVA_INT,   8,  42)            // seq
        buf.set(JAVA_INT,   12, 0)             // pid = kernel
        val hdr = parseNlMsgHdr(buf, 0L)
        println(s"len=${hdr.len}  type=${hdr.msgType}" +
                s"  flags=${hdr.flags}  seq=${hdr.seq}  pid=${hdr.pid}")
end main
```

Encapsulating the header parse into `parseNlMsgHdr` isolates the  
low-level `MemorySegment` offset arithmetic from the higher-level parsing  
logic. The `case class NlMsgHdr` gives each field a meaningful name and  
makes pattern-matching on message types straightforward. The `offset`  
parameter allows the function to parse headers at any position within a  
larger buffer, which is essential when processing multipart responses  
that contain several consecutive headers.  

## Reading the routing table via Netlink

Sends an `RTM_GETROUTE` dump and prints the raw message types received.  

```scala
import java.lang.foreign.*
import java.lang.foreign.ValueLayout.*

@main def main() =
    val AF_NETLINK = 16; val SOCK_RAW = 3; val NETLINK_ROUTE = 0
    val RTM_GETROUTE = 26; val RTM_NEWROUTE = 24
    val NLM_F_REQUEST = 0x01; val NLM_F_DUMP = 0x300
    val NLMSG_DONE = 3

    val linker  = Linker.nativeLinker()
    val lookup  = linker.defaultLookup()
    val socketH = linker.downcallHandle(lookup.find("socket").get(),
        FunctionDescriptor.of(JAVA_INT, JAVA_INT, JAVA_INT, JAVA_INT))
    val sendH   = linker.downcallHandle(lookup.find("send").get(),
        FunctionDescriptor.of(JAVA_LONG, JAVA_INT, ADDRESS, JAVA_LONG, JAVA_INT))
    val recvH   = linker.downcallHandle(lookup.find("recv").get(),
        FunctionDescriptor.of(JAVA_LONG, JAVA_INT, ADDRESS, JAVA_LONG, JAVA_INT))
    val closeH  = linker.downcallHandle(lookup.find("close").get(),
        FunctionDescriptor.ofVoid(JAVA_INT))

    val fd = socketH.invoke(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE).asInstanceOf[Int]
    Arena.ofConfined().use: arena =>
        val req = arena.allocate(20)
        req.set(JAVA_INT,   0,  20)
        req.set(JAVA_SHORT, 4,  RTM_GETROUTE.toShort)
        req.set(JAVA_SHORT, 6,  (NLM_F_REQUEST | NLM_F_DUMP).toShort)
        req.set(JAVA_INT,   8,  1)
        req.set(JAVA_INT,   12, 0)
        req.set(JAVA_BYTE,  16, 0.toByte)
        sendH.invoke(fd, req, 20L, 0)

        val buf  = arena.allocate(65536)
        var done = false
        var routeCount = 0
        while !done do
            val n = recvH.invoke(fd, buf, 65536L, 0).asInstanceOf[Long]
            var off = 0L
            while off + 16 <= n do
                val msgLen  = buf.get(JAVA_INT,   off).toLong
                val msgType = buf.get(JAVA_SHORT, off + 4).toInt & 0xffff
                if msgType == NLMSG_DONE then done = true
                else if msgType == RTM_NEWROUTE then routeCount += 1
                off += Math.max(msgLen, 16L)
            if n <= 0 then done = true
        println(s"Received $routeCount RTM_NEWROUTE messages")
    closeH.invoke(fd)
end main
```

Each `RTM_NEWROUTE` message in the dump represents one routing entry.  
The payload after the 16-byte `nlmsghdr` begins with a `rtmsg` struct  
(12 bytes) followed by a series of `rtattr` type-length-value records  
that carry the destination prefix, gateway, and output interface index.  
This example counts the raw messages; the following examples show how  
to decode the embedded attributes.  

## Reading link information via Netlink

Requests link (interface) information using `RTM_GETLINK`.  

```scala
import java.lang.foreign.*
import java.lang.foreign.ValueLayout.*

@main def main() =
    val AF_NETLINK = 16; val SOCK_RAW = 3; val NETLINK_ROUTE = 0
    val RTM_GETLINK = 18; val RTM_NEWLINK = 16; val NLMSG_DONE = 3
    val NLM_F_REQUEST = 0x01; val NLM_F_DUMP = 0x300

    val linker  = Linker.nativeLinker()
    val lookup  = linker.defaultLookup()
    val socketH = linker.downcallHandle(lookup.find("socket").get(),
        FunctionDescriptor.of(JAVA_INT, JAVA_INT, JAVA_INT, JAVA_INT))
    val sendH   = linker.downcallHandle(lookup.find("send").get(),
        FunctionDescriptor.of(JAVA_LONG, JAVA_INT, ADDRESS, JAVA_LONG, JAVA_INT))
    val recvH   = linker.downcallHandle(lookup.find("recv").get(),
        FunctionDescriptor.of(JAVA_LONG, JAVA_INT, ADDRESS, JAVA_LONG, JAVA_INT))
    val closeH  = linker.downcallHandle(lookup.find("close").get(),
        FunctionDescriptor.ofVoid(JAVA_INT))

    val fd = socketH.invoke(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE).asInstanceOf[Int]
    Arena.ofConfined().use: arena =>
        // nlmsghdr (16) + ifinfomsg family+pad+type+index+flags+change (16)
        val req = arena.allocate(32)
        req.set(JAVA_INT,   0,  32)
        req.set(JAVA_SHORT, 4,  RTM_GETLINK.toShort)
        req.set(JAVA_SHORT, 6,  (NLM_F_REQUEST | NLM_F_DUMP).toShort)
        req.set(JAVA_INT,   8,  1)
        req.set(JAVA_INT,   12, 0)
        // ifinfomsg: all zeros requests all interfaces
        sendH.invoke(fd, req, 32L, 0)

        val buf  = arena.allocate(65536)
        var done = false
        var linkCount = 0
        while !done do
            val n = recvH.invoke(fd, buf, 65536L, 0).asInstanceOf[Long]
            var off = 0L
            while off + 16 <= n do
                val msgLen  = buf.get(JAVA_INT,   off).toLong
                val msgType = buf.get(JAVA_SHORT, off + 4).toInt & 0xffff
                if msgType == NLMSG_DONE then done = true
                else if msgType == RTM_NEWLINK then linkCount += 1
                off += Math.max(msgLen, 16L)
            if n <= 0 then done = true
        println(s"Received $linkCount RTM_NEWLINK messages")
    closeH.invoke(fd)
end main
```

`RTM_GETLINK` with `NLM_F_DUMP` returns one `RTM_NEWLINK` message per  
network interface. Each message carries an `ifinfomsg` struct followed  
by `rtattr` records that describe the interface name (`IFLA_IFNAME`),  
MAC address (`IFLA_ADDRESS`), MTU (`IFLA_MTU`), and operational flags.  
The payload after the 16-byte `nlmsghdr` contains the `ifinfomsg` family  
byte, padding, type, index, flags, and change mask before the attributes  
begin at offset 16 + 16 = 32.  

## Reading address information via Netlink

Queries assigned IP addresses using `RTM_GETADDR`.  

```scala
import java.lang.foreign.*
import java.lang.foreign.ValueLayout.*

@main def main() =
    val AF_NETLINK = 16; val SOCK_RAW = 3; val NETLINK_ROUTE = 0
    val RTM_GETADDR = 22; val RTM_NEWADDR = 20; val NLMSG_DONE = 3
    val NLM_F_REQUEST = 0x01; val NLM_F_DUMP = 0x300

    val linker  = Linker.nativeLinker()
    val lookup  = linker.defaultLookup()
    val socketH = linker.downcallHandle(lookup.find("socket").get(),
        FunctionDescriptor.of(JAVA_INT, JAVA_INT, JAVA_INT, JAVA_INT))
    val sendH   = linker.downcallHandle(lookup.find("send").get(),
        FunctionDescriptor.of(JAVA_LONG, JAVA_INT, ADDRESS, JAVA_LONG, JAVA_INT))
    val recvH   = linker.downcallHandle(lookup.find("recv").get(),
        FunctionDescriptor.of(JAVA_LONG, JAVA_INT, ADDRESS, JAVA_LONG, JAVA_INT))
    val closeH  = linker.downcallHandle(lookup.find("close").get(),
        FunctionDescriptor.ofVoid(JAVA_INT))

    val fd = socketH.invoke(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE).asInstanceOf[Int]
    Arena.ofConfined().use: arena =>
        val req = arena.allocate(24)
        req.set(JAVA_INT,   0,  24)
        req.set(JAVA_SHORT, 4,  RTM_GETADDR.toShort)
        req.set(JAVA_SHORT, 6,  (NLM_F_REQUEST | NLM_F_DUMP).toShort)
        req.set(JAVA_INT,   8,  1)
        req.set(JAVA_INT,   12, 0)
        // ifaddrmsg: family=0 (AF_UNSPEC), rest zero
        sendH.invoke(fd, req, 24L, 0)

        val buf  = arena.allocate(65536)
        var done = false
        var addrCount = 0
        while !done do
            val n = recvH.invoke(fd, buf, 65536L, 0).asInstanceOf[Long]
            var off = 0L
            while off + 16 <= n do
                val msgLen  = buf.get(JAVA_INT,   off).toLong
                val msgType = buf.get(JAVA_SHORT, off + 4).toInt & 0xffff
                if msgType == NLMSG_DONE then done = true
                else if msgType == RTM_NEWADDR then addrCount += 1
                off += Math.max(msgLen, 16L)
            if n <= 0 then done = true
        println(s"Received $addrCount RTM_NEWADDR messages")
    closeH.invoke(fd)
end main
```

`RTM_GETADDR` returns one `RTM_NEWADDR` message per IP address assigned  
to any interface, covering both IPv4 and IPv6. Each message begins with  
an `ifaddrmsg` struct (8 bytes) that carries the address family, prefix  
length, flags, scope, and interface index, followed by `rtattr` records  
for `IFA_LOCAL`, `IFA_ADDRESS`, and `IFA_LABEL`. Parsing the attributes  
at the correct byte offset after the 16 + 8 = 24 header bytes reveals  
the actual address bytes.  

## Reading neighbor information via Netlink

Queries the ARP/NDP neighbor table using `RTM_GETNEIGH`.  

```scala
import java.lang.foreign.*
import java.lang.foreign.ValueLayout.*

@main def main() =
    val AF_NETLINK = 16; val SOCK_RAW = 3; val NETLINK_ROUTE = 0
    val RTM_GETNEIGH = 30; val RTM_NEWNEIGH = 28; val NLMSG_DONE = 3
    val NLM_F_REQUEST = 0x01; val NLM_F_DUMP = 0x300

    val linker  = Linker.nativeLinker()
    val lookup  = linker.defaultLookup()
    val socketH = linker.downcallHandle(lookup.find("socket").get(),
        FunctionDescriptor.of(JAVA_INT, JAVA_INT, JAVA_INT, JAVA_INT))
    val sendH   = linker.downcallHandle(lookup.find("send").get(),
        FunctionDescriptor.of(JAVA_LONG, JAVA_INT, ADDRESS, JAVA_LONG, JAVA_INT))
    val recvH   = linker.downcallHandle(lookup.find("recv").get(),
        FunctionDescriptor.of(JAVA_LONG, JAVA_INT, ADDRESS, JAVA_LONG, JAVA_INT))
    val closeH  = linker.downcallHandle(lookup.find("close").get(),
        FunctionDescriptor.ofVoid(JAVA_INT))

    val fd = socketH.invoke(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE).asInstanceOf[Int]
    Arena.ofConfined().use: arena =>
        val req = arena.allocate(24)
        req.set(JAVA_INT,   0,  24)
        req.set(JAVA_SHORT, 4,  RTM_GETNEIGH.toShort)
        req.set(JAVA_SHORT, 6,  (NLM_F_REQUEST | NLM_F_DUMP).toShort)
        req.set(JAVA_INT,   8,  1)
        req.set(JAVA_INT,   12, 0)
        sendH.invoke(fd, req, 24L, 0)

        val buf  = arena.allocate(65536)
        var done = false
        var neighCount = 0
        while !done do
            val n = recvH.invoke(fd, buf, 65536L, 0).asInstanceOf[Long]
            var off = 0L
            while off + 16 <= n do
                val msgLen  = buf.get(JAVA_INT,   off).toLong
                val msgType = buf.get(JAVA_SHORT, off + 4).toInt & 0xffff
                if msgType == NLMSG_DONE then done = true
                else if msgType == RTM_NEWNEIGH then neighCount += 1
                off += Math.max(msgLen, 16L)
            if n <= 0 then done = true
        println(s"Received $neighCount RTM_NEWNEIGH messages")
    closeH.invoke(fd)
end main
```

`RTM_GETNEIGH` gives access to the same data as `/proc/net/arp` but also  
includes IPv6 NDP entries and carries richer state information. Each  
`RTM_NEWNEIGH` message contains a `ndmsg` struct with the interface  
index, state flags (`NUD_REACHABLE`, `NUD_STALE`, `NUD_FAILED`, etc.),  
and neighbor flags. The `NDA_DST` attribute holds the IP address and  
`NDA_LLADDR` holds the hardware address, both as raw bytes in the  
attribute payload.  

## Parsing rtnetlink attributes

Iterates over the `rtattr` records following a Netlink message header.  

```scala
import java.lang.foreign.*
import java.lang.foreign.ValueLayout.*

case class RtAttr(attrType: Int, data: Array[Byte])

def parseAttrs(seg: MemorySegment, start: Long, end: Long): List[RtAttr] =
    var off    = start
    val result = scala.collection.mutable.ListBuffer[RtAttr]()
    while off + 4 <= end do
        val rta_len  = seg.get(JAVA_SHORT, off).toInt & 0xffff
        val rta_type = seg.get(JAVA_SHORT, off + 2).toInt & 0xffff
        if rta_len < 4 then off = end  // guard against corrupt data
        else
            val dataLen  = rta_len - 4
            val data     = new Array[Byte](dataLen)
            MemorySegment.copy(seg, off + 4, MemorySegment.ofArray(data), 0, dataLen)
            result += RtAttr(rta_type, data)
            off += ((rta_len + 3) & ~3).toLong  // align to 4 bytes
    result.toList

@main def main() =
    Arena.ofConfined().use: arena =>
        // Simulate an rtattr: type=1 (IFA_ADDRESS), data = 192.168.1.1
        val seg = arena.allocate(12)
        seg.set(JAVA_SHORT, 0, 8.toShort)   // rta_len = 4 header + 4 data
        seg.set(JAVA_SHORT, 2, 1.toShort)   // rta_type = IFA_ADDRESS
        seg.set(JAVA_BYTE,  4, 192.toByte)
        seg.set(JAVA_BYTE,  5, 168.toByte)
        seg.set(JAVA_BYTE,  6, 1.toByte)
        seg.set(JAVA_BYTE,  7, 1.toByte)
        val attrs = parseAttrs(seg, 0L, 8L)
        attrs.foreach: a =>
            val ip = a.data.map(b => (b & 0xff).toString).mkString(".")
            println(s"type=${a.attrType}  data=$ip")
end main
```

`rtattr` records are the fundamental building block of Netlink payloads.  
Each record begins with a two-byte total length and a two-byte type code,  
followed by the payload bytes. After reading each attribute, the offset  
advances by the length rounded up to a four-byte boundary, which is the  
standard Netlink alignment rule. `MemorySegment.copy` transfers raw bytes  
into a `Array[Byte]` for safe manipulation in JVM code. The `parseAttrs`  
helper can be reused for `RTM_NEWROUTE`, `RTM_NEWLINK`, `RTM_NEWADDR`,  
and `RTM_NEWNEIGH` messages.  

## Converting C structs to MemorySegment

Uses `MemoryLayout` to describe a C struct and access its fields.  

```scala
import java.lang.foreign.*
import java.lang.foreign.ValueLayout.*

@main def main() =
    // Describe sockaddr_nl: family(2) + pad(2) + pid(4) + groups(4) = 12 bytes
    val sockaddrNlLayout = MemoryLayout.structLayout(
        JAVA_SHORT.withName("nl_family"),
        JAVA_SHORT.withName("nl_pad"),
        JAVA_INT.withName("nl_pid"),
        JAVA_INT.withName("nl_groups")
    ).withName("sockaddr_nl")

    val familyVH = sockaddrNlLayout.varHandle(
        MemoryLayout.PathElement.groupElement("nl_family"))
    val pidVH    = sockaddrNlLayout.varHandle(
        MemoryLayout.PathElement.groupElement("nl_pid"))
    val groupsVH = sockaddrNlLayout.varHandle(
        MemoryLayout.PathElement.groupElement("nl_groups"))

    Arena.ofConfined().use: arena =>
        val seg = arena.allocate(sockaddrNlLayout)
        familyVH.set(seg, 0L, 16.toShort)  // AF_NETLINK
        pidVH.set(seg, 0L, 0)              // 0 = kernel
        groupsVH.set(seg, 0L, 0)

        val fam  = familyVH.get(seg, 0L).asInstanceOf[Short]
        val pid  = pidVH.get(seg, 0L).asInstanceOf[Int]
        val grps = groupsVH.get(seg, 0L).asInstanceOf[Int]
        println(s"family=$fam  pid=$pid  groups=$grps")
end main
```

`MemoryLayout.structLayout` creates a layout that mirrors the C struct  
definition and provides named access paths. `varHandle` with a  
`groupElement` path generates a `VarHandle` that reads and writes a  
specific field at the correct offset within the segment, eliminating  
manual byte-offset arithmetic. `arena.allocate(layout)` allocates exactly  
as many bytes as the layout requires, preventing buffer overruns. This  
approach scales to complex nested structs without sacrificing type safety.  

## Safe off-heap memory handling

Demonstrates the `Arena` API for deterministic off-heap memory cleanup.  

```scala
import java.lang.foreign.*
import java.lang.foreign.ValueLayout.*

@main def main() =
    println("=== Confined arena (single-threaded, deterministic) ===")
    try
        Arena.ofConfined().use: arena =>
            val buf = arena.allocate(256)
            buf.set(JAVA_INT, 0, 0xDEADBEEF.toInt)
            val v = buf.get(JAVA_INT, 0)
            println(f"Wrote and read: 0x$v%08X")
            println(s"Segment alive: ${buf.scope.isAlive}")
        // arena is closed here; buf is no longer accessible
        println("Arena closed successfully")
    catch
        case ex: Exception => println(s"Error: $ex")

    println("\n=== Shared arena (multi-threaded) ===")
    val shared = Arena.ofShared()
    try
        val seg = shared.allocate(64)
        seg.set(JAVA_LONG, 0, System.nanoTime())
        println(s"Timestamp written to shared segment")
    finally
        shared.close()
        println("Shared arena closed")
end main
```

`Arena.ofConfined()` creates a memory scope that is accessible only from  
the thread that created it, which enables the runtime to perform cheap,  
non-synchronised cleanup. `Arena.ofShared()` allows access from multiple  
threads but requires explicit `close()`, which is why it is placed in a  
`finally` block. After an arena is closed, any attempt to access memory  
allocated from it throws `IllegalStateException`, providing a safe  
boundary that prevents use-after-free bugs that would be silent in C.  

## Reading nftables ruleset

Runs `nft list ruleset` and parses the table and chain names from output.  

```scala
import sys.process.*
import scala.util.Try

@main def main() =
    val result = Try("nft list ruleset".!!)
    result match
        case scala.util.Failure(ex) =>
            println(s"nft not available or permission denied: ${ex.getMessage}")
        case scala.util.Success(output) =>
            val lines = output.linesIterator.toList
            val tables = lines.filter(_.trim.startsWith("table "))
                .map(_.trim)
            val chains = lines.filter(_.trim.startsWith("chain "))
                .map(_.trim)
            println(s"Tables (${tables.size}):")
            tables.foreach(t => println(s"  $t"))
            println(s"Chains (${chains.size}):")
            chains.foreach(c => println(s"  $c"))
end main
```

`sys.process.*` provides the `!!` operator, which captures standard  
output as a `String`. Wrapping the call in `Try` handles the case where  
`nft` is not installed or the process returns a non-zero exit code  
due to insufficient privileges. Reading nftables state does not modify  
any firewall rules; it is a purely read-only operation that mirrors  
the `nft list ruleset` command. On systems that use `iptables` instead  
of `nftables`, the output will be empty or the command will fail.  

## Reading iptables counters

Parses packet and byte counters from `iptables -L -n -v` output.  

```scala
import sys.process.*
import scala.util.Try

case class IptRule(chain: String, pkts: String, bytes: String, target: String)

@main def main() =
    val result = Try("iptables -L -n -v".!!)
    result match
        case scala.util.Failure(ex) =>
            println(s"iptables not available: ${ex.getMessage}")
        case scala.util.Success(output) =>
            var currentChain = "UNKNOWN"
            val rules = scala.collection.mutable.ListBuffer[IptRule]()
            output.linesIterator.foreach: line =>
                if line.startsWith("Chain ") then
                    currentChain = line.split(" ")(1)
                else
                    val f = line.trim.split("""\s+""")
                    if f.length >= 4 && f(0).forall(_.isDigit) then
                        rules += IptRule(currentChain, f(0), f(1), f(2))
            rules.foreach: r =>
                println(s"${r.chain.padTo(12,' ')}  pkts=${r.pkts.padTo(8,' ')}" +
                        s"  bytes=${r.bytes.padTo(10,' ')}  target=${r.target}")
end main
```

The verbose iptables listing begins each chain section with a `Chain`  
header followed by per-rule rows. Each rule row starts with a packet  
count and a byte count before the target and protocol fields. Tracking  
which chain each rule belongs to requires maintaining a mutable  
`currentChain` variable as the parser advances through the lines.  
Packet and byte counters reset when the chain is flushed or the system  
reboots, so they represent cumulative totals since the last reset.  

## Reading connection tracking entries

Reads active connection tracking entries from `/proc/net/nf_conntrack`.  

```scala
import scala.io.Source
import scala.util.Try

@main def main() =
    val result = Try(Source.fromFile("/proc/net/nf_conntrack"))
    result match
        case scala.util.Failure(_) =>
            println("nf_conntrack not available (module not loaded?)")
        case scala.util.Success(source) =>
            try
                val lines = source.getLines().toList
                println(s"Total tracked connections: ${lines.size}")
                lines.take(5).foreach: line =>
                    val fields = line.trim.split("""\s+""")
                    val proto  = if fields.length > 2 then fields(2) else "?"
                    val state  = fields.find(_.startsWith("state="))
                        .getOrElse("state=?")
                    val src    = fields.find(_.startsWith("src="))
                        .getOrElse("src=?")
                    val dst    = fields.find(_.startsWith("dst="))
                        .getOrElse("dst=?")
                    println(s"$proto  $src  $dst  $state")
            finally
                source.close()
end main
```

`/proc/net/nf_conntrack` exposes the netfilter connection tracking table,  
which records every tracked flow traversing the firewall. Each line  
contains the protocol family, the L4 protocol, the timeout, state for  
stateful protocols, and key=value pairs for source/destination addresses  
and ports. This file is only present when the `nf_conntrack` kernel  
module is loaded. Reading it does not affect firewall state, making it  
safe for passive monitoring applications.  

## Conceptual packet filtering pipeline

Demonstrates how packet filtering decisions can be modelled in Scala 3.  

```scala
enum Verdict:
    case Accept, Drop, Reject

case class Packet(srcIp: String, dstIp: String, proto: String, dstPort: Int)

type Rule = Packet => Option[Verdict]

def chain(rules: List[Rule], pkt: Packet): Verdict =
    rules.flatMap(_(pkt)).headOption.getOrElse(Verdict.Accept)

@main def main() =
    val blockSsh: Rule = pkt =>
        if pkt.dstPort == 22 then Some(Verdict.Drop) else None

    val allowHttp: Rule = pkt =>
        if pkt.dstPort == 80 || pkt.dstPort == 443 then Some(Verdict.Accept)
        else None

    val rejectAll: Rule = _ => Some(Verdict.Reject)

    val filterChain = List(blockSsh, allowHttp, rejectAll)

    val packets = List(
        Packet("10.0.0.1", "10.0.0.2", "tcp", 22),
        Packet("10.0.0.1", "10.0.0.2", "tcp", 80),
        Packet("10.0.0.1", "10.0.0.2", "udp", 53)
    )
    packets.foreach: p =>
        val v = chain(filterChain, p)
        println(s"${p.srcIp}:${p.dstPort}  =>  $v")
end main
```

This model mirrors how Linux netfilter organises rules into chains, where  
each rule either returns a terminal verdict or passes the packet to the  
next rule. `Option[Verdict]` represents the rule's decision: `None` means  
"pass to next rule" and `Some(v)` terminates the chain. The functional  
approach makes it straightforward to test individual rules in isolation,  
compose chains from rule lists, and add logging or metrics at any point  
in the pipeline without modifying the rule implementations.  

## Computing interface throughput

Samples RX/TX byte counters twice and computes bytes-per-second rates.  

```scala
import scala.io.Source

def readBytes(iface: String): Option[(Long, Long)] =
    val path = s"/sys/class/net/$iface/statistics"
    try
        val rx = Source.fromFile(s"$path/rx_bytes").mkString.trim.toLong
        val tx = Source.fromFile(s"$path/tx_bytes").mkString.trim.toLong
        Some((rx, tx))
    catch case _: Exception => None

@main def main() =
    val iface    = "eth0"
    val interval = 1000L  // milliseconds

    readBytes(iface) match
        case None => println(s"Cannot read stats for $iface")
        case Some((rx0, tx0)) =>
            Thread.sleep(interval)
            readBytes(iface) match
                case None => println("Stats disappeared during measurement")
                case Some((rx1, tx1)) =>
                    val rxRate = (rx1 - rx0) * 1000 / interval
                    val txRate = (tx1 - tx0) * 1000 / interval
                    println(s"$iface  RX=${rxRate} B/s  TX=${txRate} B/s")
end main
```

Two readings separated by a known interval allow a simple delta  
calculation. Dividing the byte delta by the interval in seconds yields  
bytes per second. The `* 1000 / interval` pattern avoids floating-point  
arithmetic while maintaining accuracy for typical traffic rates.  
Wrapping the file reads in `try/catch` handles the case where the  
interface is removed between samples, which can happen with USB network  
adapters or virtual interfaces that are torn down dynamically.  

## Computing packet loss from counters

Calculates the receive error rate from sysfs statistics.  

```scala
import scala.io.Source

def readCounter(path: String): Long =
    try Source.fromFile(path).mkString.trim.toLong
    catch case _: Exception => 0L

@main def main() =
    val sysnet = "/sys/class/net"
    val dirs   = new java.io.File(sysnet).listFiles().filter(_.isDirectory)
    dirs.sortBy(_.getName).foreach: dir =>
        val base     = s"$sysnet/${dir.getName}/statistics"
        val rxPkts   = readCounter(s"$base/rx_packets")
        val rxErrors = readCounter(s"$base/rx_errors")
        val rxDropped = readCounter(s"$base/rx_dropped")
        val total    = rxErrors + rxDropped
        val lossRate = if rxPkts > 0 then
            f"${total.toDouble / rxPkts * 100.0}%.4f%%"
        else "n/a"
        println(s"${dir.getName.padTo(12,' ')}  rxPkts=$rxPkts" +
                s"  errors=$rxErrors  dropped=$rxDropped  loss=$lossRate")
end main
```

Receive errors (`rx_errors`) count frames that failed CRC checks, had  
length errors, or were rejected by the driver. Dropped packets  
(`rx_dropped`) count frames that the driver discarded because kernel  
ring buffers were full. The packet loss rate is the ratio of lost frames  
to total received frames. A non-zero error rate on a physical interface  
typically indicates a bad cable, duplex mismatch, or NIC hardware  
problem. A non-zero drop rate often indicates that the CPU is not  
processing interrupts fast enough.  

## Detecting interface flaps

Polls operational state repeatedly and reports transitions.  

```scala
import java.nio.file.{Files, Paths}

def readState(iface: String): String =
    val path = Paths.get(s"/sys/class/net/$iface/operstate")
    if Files.exists(path) then Files.readString(path).trim else "missing"

@main def main() =
    val iface    = "eth0"
    val pollMs   = 500L
    val samples  = 10
    var prevState = readState(iface)
    var flaps    = 0

    println(s"Monitoring $iface for $samples samples (${pollMs}ms interval)")
    for _ <- 1 to samples do
        Thread.sleep(pollMs)
        val cur = readState(iface)
        if cur != prevState then
            flaps += 1
            println(s"FLAP: $iface transitioned $prevState -> $cur")
            prevState = cur

    if flaps == 0 then
        println(s"No flaps detected; final state: $prevState")
    else
        println(s"Total flaps detected: $flaps")
end main
```

A flap occurs when an interface transitions from `up` to `down` and  
back, or vice versa, typically caused by cable faults, SFP module  
failures, or negotiation problems at the physical layer. Repeated  
flapping within seconds or minutes indicates a hardware problem that  
requires physical investigation. This simple polling loop provides  
a lightweight alternative to kernel netlink event monitoring for  
short-duration diagnostics.  

## Detecting routing changes

Periodically snapshots the routing table and reports added or removed routes.  

```scala
import scala.io.Source

def readRoutes(): Set[String] =
    val source = Source.fromFile("/proc/net/route")
    try source.getLines().drop(1).map(_.trim).filter(_.nonEmpty).toSet
    finally source.close()

@main def main() =
    val pollMs  = 1000L
    val samples = 5
    var prev    = readRoutes()

    println(s"Monitoring routing table for $samples samples...")
    for _ <- 1 to samples do
        Thread.sleep(pollMs)
        val cur     = readRoutes()
        val added   = cur -- prev
        val removed = prev -- cur
        if added.nonEmpty then
            println(s"ADDED routes:")
            added.foreach(r => println(s"  $r"))
        if removed.nonEmpty then
            println(s"REMOVED routes:")
            removed.foreach(r => println(s"  $r"))
        if added.isEmpty && removed.isEmpty then
            println("Routing table unchanged")
        prev = cur
end main
```

Representing each routing table entry as a `String` and using set  
arithmetic (`--`) gives a clean one-pass implementation of change  
detection. The `added` set contains entries present in the current  
snapshot but absent from the previous one, and `removed` is the  
complement. This approach works equally well for detecting route  
installation by network daemons such as OSPF or BGP speakers as for  
detecting manual `ip route add` commands.  

## Detecting ARP changes

Monitors the ARP table for new, removed, or modified entries.  

```scala
import scala.io.Source

def readArp(): Map[String, String] =
    val source = Source.fromFile("/proc/net/arp")
    try
        source.getLines().drop(1).flatMap: line =>
            val f = line.trim.split("""\s+""")
            if f.length >= 4 then Some(f(0) -> f(3)) else None
        .toMap
    finally
        source.close()

@main def main() =
    val pollMs  = 1000L
    val samples = 5
    var prev    = readArp()

    println(s"Monitoring ARP table for $samples samples...")
    for _ <- 1 to samples do
        Thread.sleep(pollMs)
        val cur = readArp()
        val allKeys = prev.keySet ++ cur.keySet
        allKeys.foreach: ip =>
            (prev.get(ip), cur.get(ip)) match
                case (None,       Some(mac)) => println(s"NEW:      $ip  $mac")
                case (Some(_),    None)      => println(s"REMOVED:  $ip")
                case (Some(old),  Some(nw)) if old != nw =>
                    println(s"CHANGED:  $ip  $old -> $nw")
                case _ => ()
        prev = cur
end main
```

Representing the ARP table as a `Map[IP, MAC]` and iterating over the  
union of keys from both snapshots handles all three change types in a  
single traversal. A MAC change for an existing IP may indicate ARP  
spoofing, a device replacement, or a legitimate address migration such  
as a VRRP failover. Logging these changes with timestamps creates a  
lightweight audit trail without requiring any specialised tools.  

## Logging OS-level network state

Writes a timestamped snapshot of interface statistics, routes, and ARP.  

```scala
import scala.io.Source
import java.time.Instant

def snap(path: String): List[String] =
    val s = Source.fromFile(path)
    try s.getLines().toList finally s.close()

@main def main() =
    val ts = Instant.now()
    println(s"=== Network state snapshot at $ts ===")

    println("\n--- Interfaces ---")
    snap("/proc/net/dev").drop(2).foreach: line =>
        val f = line.trim.split("""\s+""")
        if f.length >= 10 then
            println(s"${f(0).stripSuffix(":")}  " +
                    s"rx=${f(1)}  tx=${f(9)}")

    println("\n--- ARP ---")
    snap("/proc/net/arp").drop(1).foreach: line =>
        val f = line.trim.split("""\s+""")
        if f.length >= 6 then
            println(s"${f(0)}  ${f(3)}  ${f(5)}")

    println("\n--- Routes ---")
    def hexToIp(h: String): String =
        val v = java.lang.Long.parseUnsignedLong(h, 16)
        s"${v&0xff}.${(v>>8)&0xff}.${(v>>16)&0xff}.${(v>>24)&0xff}"
    snap("/proc/net/route").drop(1).foreach: line =>
        val f = line.trim.split("""\s+""")
        if f.length >= 8 then
            println(s"${hexToIp(f(1))}  via ${hexToIp(f(2))}  dev ${f(0)}")
end main
```

Combining multiple data sources into a single timestamped snapshot  
creates a useful diagnostic artefact that can be stored, compared, or  
transmitted to a monitoring system. The `Instant.now()` timestamp  
ensures that entries from different files are known to be captured  
at approximately the same point in time, which is important when  
investigating time-sensitive events such as routing convergence.  

## Building a netstat clone

Combines `/proc/net/tcp` and `/proc/net/udp` into a netstat-style listing.  

```scala
import scala.io.Source

def hexToIp(h: String): String =
    val v = java.lang.Long.parseUnsignedLong(h, 16)
    s"${v&0xff}.${(v>>8)&0xff}.${(v>>16)&0xff}.${(v>>24)&0xff}"

val tcpStates = Map(
    "01"->"ESTABLISHED","02"->"SYN_SENT","03"->"SYN_RECV",
    "04"->"FIN_WAIT1",  "05"->"FIN_WAIT2","06"->"TIME_WAIT",
    "07"->"CLOSE",      "08"->"CLOSE_WAIT","09"->"LAST_ACK",
    "0A"->"LISTEN",     "0B"->"CLOSING"
)

def parseSockets(path: String, proto: String): List[String] =
    val source = Source.fromFile(path)
    try
        source.getLines().drop(1).flatMap: line =>
            val f = line.trim.split("""\s+""")
            if f.length >= 4 then
                val Array(lip, lp) = f(1).split(":")
                val Array(rip, rp) = f(2).split(":")
                val state = tcpStates.getOrElse(f(3).toUpperCase, f(3))
                val local  = s"${hexToIp(lip)}:${Integer.parseInt(lp, 16)}"
                val remote = s"${hexToIp(rip)}:${Integer.parseInt(rp, 16)}"
                Some(f"$proto%-5s $local%-25s $remote%-25s $state")
            else None
        .toList
    finally source.close()

@main def main() =
    println(f"${"Proto"}%-5s ${"Local Address"}%-25s" +
            f" ${"Foreign Address"}%-25s State")
    (parseSockets("/proc/net/tcp", "tcp") ++
     parseSockets("/proc/net/udp", "udp"))
        .sortBy(identity)
        .foreach(println)
end main
```

This clone replicates the core functionality of `netstat -tn` and  
`netstat -un` by reading both TCP and UDP socket tables and formatting  
them into a unified listing. The `f` string interpolator with width  
specifiers aligns columns consistently, producing output that is easy  
to scan. Sorting the combined list groups similar entries together.  
The implementation is entirely read-only and works without root  
privileges on typical Linux systems.  

## Building an ip addr clone

Lists all interfaces with their addresses, mimicking `ip addr show`.  

```scala
import java.net.NetworkInterface
import scala.jdk.CollectionConverters.*

@main def main() =
    NetworkInterface.getNetworkInterfaces.asScala
        .toList.sortBy(_.getIndex)
        .foreach: iface =>
            val hw = Option(iface.getHardwareAddress)
                .filter(_.nonEmpty)
                .map(b => b.map(x => f"${x & 0xff}%02x").mkString(":"))
                .getOrElse("none")
            val state = if iface.isUp then "UP" else "DOWN"
            println(s"${iface.getIndex}: ${iface.getName}: " +
                    s"<$state>  link/ether $hw")
            iface.getInterfaceAddresses.asScala.foreach: ia =>
                val proto = ia.getAddress match
                    case _: java.net.Inet4Address => "inet"
                    case _                        => "inet6"
                val addr   = ia.getAddress.getHostAddress
                val prefix = ia.getNetworkPrefixLength
                println(s"    $proto $addr/$prefix")
end main
```

This clone mirrors the output format of `ip addr show` by printing  
the interface index, name, state, and hardware address on one line  
followed by indented address lines for each assigned IP. Distinguishing  
`inet` from `inet6` with a pattern match on the concrete `InetAddress`  
subclass avoids string parsing. The `NetworkInterface` API is fully  
cross-platform, making this code portable to macOS and other UNIX  
systems without modification.  

## Building an ip route clone

Parses `/proc/net/route` and displays it in `ip route show` style.  

```scala
import scala.io.Source

def hexToIp(h: String): String =
    val v = java.lang.Long.parseUnsignedLong(h, 16)
    s"${v&0xff}.${(v>>8)&0xff}.${(v>>16)&0xff}.${(v>>24)&0xff}"

def maskToCidr(h: String): Int =
    java.lang.Integer.bitCount(
        java.lang.Long.parseUnsignedLong(h, 16).toInt)

@main def main() =
    val source = Source.fromFile("/proc/net/route")
    try
        source.getLines().drop(1).foreach: line =>
            val f = line.trim.split("""\s+""")
            if f.length >= 11 then
                val dest   = hexToIp(f(1))
                val cidr   = maskToCidr(f(7))
                val gw     = hexToIp(f(2))
                val iface  = f(0)
                val metric = f(6)
                val prefix = if dest == "0.0.0.0" && cidr == 0 then "default"
                             else s"$dest/$cidr"
                val via    = if gw == "0.0.0.0" then "" else s" via $gw"
                println(s"$prefix$via dev $iface metric $metric")
    finally
        source.close()
end main
```

Displaying `0.0.0.0/0` as `default` matches the output of `ip route show`  
and improves readability. Omitting the `via` clause for directly  
connected routes (gateway `0.0.0.0`) also follows the same convention.  
The `maskToCidr` function counts set bits in the mask integer to produce  
the CIDR prefix length. This implementation reads the same data source  
that the kernel's `ip route` tool uses, so the output is always accurate  
and reflects the live routing table at the moment of execution.  
