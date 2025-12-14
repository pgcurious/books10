# Chapter 2: Networks from Scratch

> **First Principles Question**: When you type a URL in your browser, what actually happens to get data from a server thousands of miles away to your screen?

---

## Chapter Overview

Before you can understand cloud computing, microservices, or APIs, you need to understand networks. Most developers treat networking as a black box—data goes in, data comes out. This chapter opens that box and shows you what's really happening.

**What readers will understand after this chapter:**
- Why we need network protocols at all
- How data physically travels across the internet
- What TCP/IP actually means and why it matters
- How DNS translates names to numbers
- Why HTTP became the language of the web
- The client-server model that underpins everything

---

## Section 1: The Fundamental Problem of Communication

### 1.1 Two Computers, One Problem

**The Scenario:**
You have two computers. You want them to share data. What do you need to figure out?

**The Questions Nobody Thinks to Ask:**
1. How does data physically move between them? (Signals)
2. How do they find each other? (Addressing)
3. How do they know data arrived correctly? (Reliability)
4. How do they structure conversations? (Protocols)
5. How do they handle multiple simultaneous conversations? (Multiplexing)

Every networking technology exists to answer one or more of these questions.

### 1.2 The Physical Reality

**Data is Physical:**
Digital data seems abstract, but it always has a physical form:
- Electrical pulses on copper wire
- Light pulses through fiber optic cables
- Radio waves through the air (WiFi)

**Analogy: Morse Code Over a Flashlight**
Imagine communicating with a friend across a valley using only a flashlight. You'd need to agree on:
- What patterns of flashes mean what (encoding)
- How to indicate "message start" and "message end"
- How to signal "please repeat, I missed that"
- How to take turns (so you're not both flashing at once)

Networking protocols solve these same problems, just with electrons instead of photons.

### 1.3 Why Standards Matter

**The Tower of Babel Problem:**
In the 1970s, IBM had their networking system, DEC had theirs, and they couldn't talk to each other. It was like having phones that could only call other phones from the same manufacturer.

**The Internet's Success:**
The Internet won because it defined open standards anyone could implement. Agree on the protocol, and your device can join the network—regardless of who made it.

---

## Section 2: The Physical and Data Link Layers

### 2.1 Getting Bits From Here to There

**The Physical Layer:**
This is about raw bit transmission. How do you represent 1s and 0s?

```
Electrical:  +5V = 1, 0V = 0
Optical:     Light on = 1, Light off = 0
Radio:       Frequency A = 1, Frequency B = 0
```

**Key Concepts:**
- **Bandwidth**: How many bits per second (like water through a pipe)
- **Latency**: How long for a bit to travel (limited by speed of light!)
- **Noise**: Interference that corrupts signals

**Physical Reality Check:**
Light travels at about 300,000 km/s. Even at this speed:
- New York to London: ~28ms minimum (just physics!)
- New York to Tokyo: ~70ms minimum

No amount of optimization can beat the speed of light. This is why latency matters in distributed systems.

### 2.2 Local Networks and MAC Addresses

**The Data Link Layer:**
Once you can send bits, how do you send them to the RIGHT computer on a local network?

**Ethernet's Solution:**
Every network card has a unique MAC (Media Access Control) address burned in at the factory:
```
00:1A:2B:3C:4D:5E
```

On a local network (your home WiFi, office LAN), computers use MAC addresses to find each other.

**Analogy: Shouting in a Room**
Ethernet originally worked like people in a room. To send a message:
1. Listen if anyone else is talking
2. Shout your message with the recipient's "name" (MAC address)
3. Everyone hears it; only the named recipient responds
4. If two people shout simultaneously (collision), wait random time and retry

Modern switches are smarter—they learn which MAC addresses are on which ports and send traffic directly.

### 2.3 The Limitation of MAC Addresses

**Why MAC Isn't Enough:**
MAC addresses are:
- Flat (no hierarchy—can't route efficiently)
- Hardware-bound (move your network card, your address moves)
- Scope-limited (only work on local network segments)

**The Problem:**
How do you send data to a computer on the other side of the world? You don't know its MAC address, and even if you did, there's no path to it.

This is where IP addresses come in.

---

## Section 3: The Network Layer — IP Addresses and Routing

### 3.1 IP Addresses: Hierarchical Addressing

**The Insight:**
What if addresses had structure—like postal addresses?

```
Postal:  Country → State → City → Street → House Number
IP:      Network → Subnetwork → → → Host
```

**IPv4 Address:**
```
192.168.1.100

192.168    = Likely a local network prefix
1          = Subnet within that network
100        = Specific host on that subnet
```

**Key Property:**
Routers can make forwarding decisions based on prefixes. They don't need to know about every computer—just "packets for 192.168.x.x go left, packets for 10.x.x.x go right."

### 3.2 How Routing Actually Works

**The Internet as a Graph:**
The Internet is millions of networks connected by routers. Each router knows:
- Directly connected networks
- "If destination is X, send to neighbor Y"

**Routing Tables:**
```
Destination       Next Hop         Interface
192.168.1.0/24    Local            eth0
10.0.0.0/8        192.168.1.1      eth0
0.0.0.0/0         192.168.1.1      eth0  (default route)
```

**Analogy: Highway Signs**
You're driving from Boston to Los Angeles. You don't need a map of LA streets in Boston. Signs just say "West →" and you follow them. Each city's signs get more specific as you get closer.

Routers work the same way. Each hop gets you closer, with more specific routing.

### 3.3 Public vs. Private IP Addresses

**The IP Address Shortage:**
IPv4 has ~4 billion addresses. With billions of devices, that's not enough.

**The Solution: Private Address Ranges**
Some IP ranges are reserved for private use:
```
10.0.0.0/8        (10.x.x.x)
172.16.0.0/12     (172.16.x.x - 172.31.x.x)
192.168.0.0/16    (192.168.x.x)
```

**Network Address Translation (NAT):**
Your home router has ONE public IP but gives each device a private IP. When devices access the internet, the router translates:

```
Your Laptop: 192.168.1.100 → Router → 73.45.123.89 (public) → Internet
```

Responses come back to the public IP, and the router remembers which internal device asked.

**Why This Matters for Developers:**
- Servers need public IPs (or port forwarding) to be reachable
- "localhost" (127.0.0.1) always means "this machine"
- Cloud servers have both private IPs (internal) and public IPs (internet-facing)

### 3.4 IPv6: The Long-Term Solution

**The Scale:**
IPv6 addresses are 128 bits:
```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
```

This provides 340 undecillion addresses (340,282,366,920,938,463,463,374,607,431,768,211,456).

That's enough for every grain of sand on Earth to have its own IP address.

**Adoption:**
IPv6 adoption is slow because IPv4 + NAT works "well enough" for now. But cloud providers and large networks increasingly use IPv6 internally.

---

## Section 4: The Transport Layer — TCP and UDP

### 4.1 The Problem IP Doesn't Solve

**IP's Limitations:**
IP gets packets from A to B, but:
- Packets can arrive out of order
- Packets can be lost
- Packets can be duplicated
- No concept of "connections"

**Analogy: Postcards vs. Phone Calls**
IP is like sending postcards. Each one might take a different route, arrive at different times, or get lost. Sometimes that's fine (sending vacation photos). Sometimes you need reliability (sending legal documents).

### 4.2 TCP: The Reliable Option

**TCP (Transmission Control Protocol):**
A connection-oriented protocol that guarantees:
- Data arrives in order
- Lost data is retransmitted
- Duplicates are eliminated
- Flow control (don't overwhelm the receiver)

**The Three-Way Handshake:**
```
Client → SYN →           Server
Client ← SYN-ACK ←       Server
Client → ACK →           Server
Connection Established!
```

**Analogy: Starting a Phone Call**
1. "Hello, can you hear me?" (SYN)
2. "Yes, I can hear you. Can you hear me?" (SYN-ACK)
3. "Yes, I hear you. Let's talk." (ACK)

**TCP's Reliability Mechanism:**
```
Sender: "Here's packet 1"
Receiver: "Got packet 1"
Sender: "Here's packet 2"
Receiver: ...silence...
Sender: (timeout) "Here's packet 2 again"
Receiver: "Got packet 2"
```

**The Cost of Reliability:**
- Extra round trips (latency)
- Retransmissions (bandwidth)
- Connection state (memory)

### 4.3 UDP: The Fast Option

**UDP (User Datagram Protocol):**
"Fire and forget"—send packets with no guarantees.

**When UDP Makes Sense:**
- Video streaming (old frame arriving late is useless—skip it)
- Online gaming (current position matters, not position 2 seconds ago)
- DNS queries (small, can easily retry)
- VoIP calls (slight glitches beat delayed audio)

**UDP's Simplicity:**
```
Source Port | Dest Port | Length | Checksum | Data
    2 bytes |  2 bytes  | 2bytes | 2 bytes  | ...
```

No sequence numbers, no acknowledgments, no connection state.

### 4.4 Ports: Multiplexing Connections

**The Problem:**
A server might handle web traffic, email, database connections, and SSH simultaneously. How does it know which packets go to which application?

**The Solution: Ports**
Ports are 16-bit numbers (0-65535) that identify applications:

```
IP Address = Which computer
Port       = Which application on that computer

192.168.1.100:80   = Web server on this machine
192.168.1.100:22   = SSH server on this machine
192.168.1.100:5432 = PostgreSQL on this machine
```

**Well-Known Ports:**
| Port | Service |
|------|---------|
| 20, 21 | FTP |
| 22 | SSH |
| 25 | SMTP (email) |
| 53 | DNS |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |

**Analogy: Apartment Numbers**
The IP address is the building. The port is the apartment number. Mail (packets) needs both to reach the right recipient.

---

## Section 5: DNS — The Internet's Phone Book

### 5.1 The Human-Readable Web

**The Problem:**
IP addresses are for computers. Humans remember names.

```
Which would you rather type?
142.250.80.46
google.com
```

**DNS (Domain Name System):**
A distributed database that maps names to IP addresses.

### 5.2 How DNS Resolution Works

**The Journey of a DNS Query:**

```
You type: www.example.com

1. Browser checks its cache
   → Found? Done! Use cached IP.

2. OS checks its cache
   → Found? Done!

3. Query your configured DNS server (often your ISP's)
   → It might have it cached

4. If not cached, recursive resolution begins:

   Your DNS Server → Root Server
   "Who handles .com?"

   Root → "Ask this .com TLD server"

   Your DNS Server → .com TLD Server
   "Who handles example.com?"

   TLD → "Ask this authoritative server"

   Your DNS Server → Authoritative Server
   "What's the IP for www.example.com?"

   Authoritative → "93.184.216.34"

5. Response cached at each level
6. Browser gets the IP, makes HTTP request
```

### 5.3 DNS Record Types

**Common Records:**

| Type | Purpose | Example |
|------|---------|---------|
| A | Maps name to IPv4 | example.com → 93.184.216.34 |
| AAAA | Maps name to IPv6 | example.com → 2606:2800:220:1:... |
| CNAME | Alias to another name | www.example.com → example.com |
| MX | Mail server for domain | example.com mail → mail.example.com |
| TXT | Arbitrary text (verification) | example.com → "google-site-verification=..." |
| NS | Nameservers for domain | example.com → ns1.example.com |

### 5.4 DNS Caching and TTL

**TTL (Time To Live):**
Each DNS record has a TTL—how long resolvers should cache it.

```
example.com   A   93.184.216.34   TTL: 3600 (1 hour)
```

**Trade-offs:**
- High TTL = Less DNS traffic, but slow to change
- Low TTL = Fast changes, but more DNS queries

**Why This Matters:**
When you deploy a new server and update DNS:
- Old cached records might persist for hours
- "DNS propagation" is really just cache expiration

---

## Section 6: HTTP — The Language of the Web

### 6.1 Request-Response Model

**The Fundamental Pattern:**
```
Client sends REQUEST  →  Server processes  →  Server sends RESPONSE
```

Every web interaction follows this pattern.

### 6.2 Anatomy of an HTTP Request

```http
GET /api/users/123 HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
User-Agent: Mozilla/5.0 ...

```

**Parts:**
- **Method**: GET, POST, PUT, DELETE, etc.
- **Path**: /api/users/123
- **Version**: HTTP/1.1
- **Headers**: Metadata (key-value pairs)
- **Body**: Optional data (empty for GET)

### 6.3 HTTP Methods and Their Meaning

| Method | Purpose | Idempotent? | Safe? |
|--------|---------|-------------|-------|
| GET | Retrieve resource | Yes | Yes |
| POST | Create resource | No | No |
| PUT | Replace resource | Yes | No |
| PATCH | Partially update | No | No |
| DELETE | Remove resource | Yes | No |
| HEAD | GET without body | Yes | Yes |
| OPTIONS | Get allowed methods | Yes | Yes |

**Idempotent**: Same request twice = same result
**Safe**: Doesn't modify server state

### 6.4 Anatomy of an HTTP Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 89
Cache-Control: max-age=3600

{
    "id": 123,
    "name": "Alice",
    "email": "alice@example.com"
}
```

**Parts:**
- **Status Code**: 200
- **Status Text**: OK
- **Headers**: Metadata
- **Body**: The actual content

### 6.5 HTTP Status Codes

**1xx: Informational**
- 100 Continue

**2xx: Success**
- 200 OK (success)
- 201 Created (POST success)
- 204 No Content (success, empty body)

**3xx: Redirection**
- 301 Moved Permanently
- 302 Found (temporary redirect)
- 304 Not Modified (use cached version)

**4xx: Client Error**
- 400 Bad Request (malformed)
- 401 Unauthorized (not authenticated)
- 403 Forbidden (authenticated but not allowed)
- 404 Not Found
- 429 Too Many Requests (rate limited)

**5xx: Server Error**
- 500 Internal Server Error
- 502 Bad Gateway
- 503 Service Unavailable
- 504 Gateway Timeout

### 6.6 HTTPS: HTTP + Security

**The Problem with HTTP:**
HTTP sends everything in plain text. Anyone between you and the server can read it.

```
You  →  Coffee Shop WiFi  →  ISP  →  ...  →  Server
         ↑ Can read everything!
```

**HTTPS (HTTP Secure):**
Adds TLS (Transport Layer Security) encryption:
1. Client and server negotiate encryption
2. Exchange certificates to verify identity
3. All data encrypted during transit

**The Handshake Simplified:**
```
Client: "Hi, I want a secure connection"
Server: "Here's my certificate proving I'm really example.com"
Client: (Verifies certificate with trusted authorities)
Client: "OK, let's agree on encryption keys"
Both: (Mathematical key exchange)
Both: Now all traffic is encrypted
```

---

## Section 7: The Client-Server Model

### 7.1 Roles and Responsibilities

**Client:**
- Initiates connections
- Sends requests
- Processes responses
- Multiple clients can exist

**Server:**
- Listens for connections
- Processes requests
- Sends responses
- Usually one server (or cluster) per service

### 7.2 What "Server" Actually Means

**The Overloaded Term:**
"Server" can mean:
1. A physical machine in a datacenter
2. A virtual machine running on physical hardware
3. A container running a process
4. The software process listening for connections
5. The role a machine plays in a client-server relationship

**Key Insight:**
Your laptop can be a server. A server can be a client to another server. The role depends on who initiated the connection.

### 7.3 Sockets: The Programming Interface

**What a Socket Is:**
A socket is an endpoint for network communication—the programming abstraction that lets you send/receive data.

**Server Socket Workflow:**
```java
// Create socket and bind to port
ServerSocket server = new ServerSocket(8080);

while (true) {
    // Wait for client connection
    Socket client = server.accept();

    // Handle client in a thread
    new Thread(() -> handleClient(client)).start();
}
```

**Client Socket Workflow:**
```java
// Connect to server
Socket socket = new Socket("example.com", 8080);

// Send request
OutputStream out = socket.getOutputStream();
out.write("GET / HTTP/1.1\r\n...".getBytes());

// Read response
InputStream in = socket.getInputStream();
// Read and process response
```

---

## Section 8: Putting It All Together

### 8.1 What Happens When You Visit google.com

Let's trace the complete journey:

**1. You type google.com and press Enter**

**2. DNS Resolution**
```
Browser cache miss → OS cache miss → Router → ISP DNS
→ Root servers → .com TLD → Google's nameservers
→ IP: 142.250.80.46
```

**3. TCP Connection**
```
Your Machine                      Google's Server
    |                                   |
    |──── SYN (port 443) ─────────────→|
    |←─── SYN-ACK ────────────────────|
    |──── ACK ────────────────────────→|
    |          Connection established   |
```

**4. TLS Handshake**
```
    |──── ClientHello ────────────────→|
    |←─── ServerHello, Certificate ───|
    |──── Key Exchange ───────────────→|
    |←─── Finished ───────────────────|
    |          Encrypted channel ready  |
```

**5. HTTP Request**
```http
GET / HTTP/1.1
Host: www.google.com
Accept: text/html
```

**6. Server Processing**
- Google's server receives request
- Generates personalized search page
- Prepares response

**7. HTTP Response**
```http
HTTP/1.1 200 OK
Content-Type: text/html
...

<!DOCTYPE html>...
```

**8. Browser Rendering**
- Parses HTML
- Requests additional resources (CSS, JS, images)
- Renders page

**Total time: ~100-500ms** (depending on location and network)

### 8.2 Mental Model: The Network Stack

```
┌─────────────────────────────────────────────────────────────┐
│  Application Layer (HTTP, HTTPS, FTP, SMTP, DNS)            │
│  "What do you want to say?"                                 │
├─────────────────────────────────────────────────────────────┤
│  Transport Layer (TCP, UDP)                                 │
│  "How reliably should we deliver it?"                       │
├─────────────────────────────────────────────────────────────┤
│  Network Layer (IP)                                         │
│  "Where should it go?"                                      │
├─────────────────────────────────────────────────────────────┤
│  Data Link Layer (Ethernet, WiFi)                           │
│  "How do we reach the next hop?"                            │
├─────────────────────────────────────────────────────────────┤
│  Physical Layer (Cables, Radio, Fiber)                      │
│  "How do we send raw bits?"                                 │
└─────────────────────────────────────────────────────────────┘
```

Each layer uses the services of the layer below and provides services to the layer above.

---

## Section 9: Now You Understand Why...

Armed with this foundation, you can now understand:

- **Why websites sometimes fail to load**: DNS failure, TCP connection timeout, HTTP 500 error—each points to a different layer failing

- **Why localhost is special**: 127.0.0.1 never leaves your machine—no network required

- **Why firewalls check ports**: Blocking port 22 prevents SSH. Blocking 80/443 prevents web traffic.

- **Why VPNs hide your traffic**: They encrypt everything and route it through a different IP, hiding both content and origin

- **Why CDNs improve performance**: They put servers geographically closer, reducing latency (speed of light matters!)

- **Why switching DNS providers can speed up browsing**: Faster DNS servers mean less time waiting before the TCP connection even starts

- **Why "serverless" still has servers**: The servers are just managed by someone else—HTTP requests still go somewhere

---

## Practical Exercises

### Exercise 1: DNS Lookup
```bash
# See DNS resolution in action
nslookup google.com
dig google.com

# See all DNS records
dig example.com ANY

# Trace the resolution path
dig +trace google.com
```
**Question:** What's the TTL on Google's A record? Why might they choose that value?

### Exercise 2: Examine TCP Connections
```bash
# See all active connections
netstat -an | grep ESTABLISHED

# Watch a TCP handshake (requires tcpdump)
sudo tcpdump -i any port 80 -c 10
```
**Question:** How many connections does your browser maintain to load a single webpage?

### Exercise 3: Raw HTTP Request
```bash
# Manual HTTP request using netcat
echo -e "GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n" | nc example.com 80
```
**Question:** What headers does example.com return?

### Exercise 4: Write a Simple Server
```java
import java.io.*;
import java.net.*;

public class SimpleServer {
    public static void main(String[] args) throws IOException {
        ServerSocket server = new ServerSocket(8080);
        System.out.println("Listening on port 8080...");

        while (true) {
            Socket client = server.accept();
            System.out.println("Client connected: " + client.getInetAddress());

            PrintWriter out = new PrintWriter(client.getOutputStream(), true);
            BufferedReader in = new BufferedReader(
                new InputStreamReader(client.getInputStream())
            );

            // Read request
            String line;
            while ((line = in.readLine()) != null && !line.isEmpty()) {
                System.out.println(line);
            }

            // Send response
            out.println("HTTP/1.1 200 OK");
            out.println("Content-Type: text/plain");
            out.println();
            out.println("Hello from Java!");

            client.close();
        }
    }
}
```
**Task:** Run this server and visit http://localhost:8080 in your browser. Observe the HTTP request headers.

### Exercise 5: Trace Route to Server
```bash
# See the path packets take
traceroute google.com
# Or on Windows
tracert google.com
```
**Question:** How many hops to reach Google? Where do the biggest latency jumps occur?

---

## Key Takeaways

1. **Networks are layered**. Physical transmission, addressing, reliability, and application logic are separate concerns solved at separate layers.

2. **IP addresses enable global routing**. Hierarchical addressing lets routers make forwarding decisions without knowing about every machine.

3. **TCP provides reliability at a cost**. When you need guaranteed delivery, use TCP. When you need speed and can tolerate loss, consider UDP.

4. **DNS is critical infrastructure**. Every internet request starts with translating a name to an IP. DNS failures break everything.

5. **HTTP is just text over TCP**. At its core, the web is simple request-response text messages.

6. **Every abstraction adds latency**. DNS lookup + TCP handshake + TLS handshake + HTTP request = multiple round trips before you get data.

---

## Looking Ahead

Now that you understand how machines communicate, Chapter 3 asks: **How do we get our code running on machines other than our own?** The evolution of deployment—from physical servers to VMs to containers to serverless.

---

## Chapter 2 Summary Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           THE NETWORK JOURNEY                               │
│                                                                             │
│   Your Computer                                                 Server      │
│  ┌────────────────┐                                    ┌────────────────┐  │
│  │  Application   │                                    │  Application   │  │
│  │  HTTP Request  │                                    │  HTTP Handler  │  │
│  ├────────────────┤                                    ├────────────────┤  │
│  │   Transport    │                                    │   Transport    │  │
│  │   TCP/UDP      │                                    │   TCP/UDP      │  │
│  ├────────────────┤                                    ├────────────────┤  │
│  │    Network     │                                    │    Network     │  │
│  │       IP       │                                    │       IP       │  │
│  ├────────────────┤                                    ├────────────────┤  │
│  │   Data Link    │                                    │   Data Link    │  │
│  │   Ethernet     │                                    │   Ethernet     │  │
│  └───────┬────────┘                                    └───────▲────────┘  │
│          │                                                     │           │
│          └──────────────→ [ Router ] → ... → [ Router ] ───────┘           │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                              DNS RESOLUTION                                 │
│                                                                             │
│   "example.com" → Browser Cache → OS Cache → DNS Server → Root → TLD →     │
│                   Authoritative → "93.184.216.34"                           │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                              TCP HANDSHAKE                                  │
│                                                                             │
│        Client                              Server                           │
│           │─── SYN ──────────────────────→│                                 │
│           │←── SYN-ACK ──────────────────│                                 │
│           │─── ACK ──────────────────────→│                                 │
│           │        Connection Ready        │                                 │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                              HTTP EXCHANGE                                  │
│                                                                             │
│   GET /api/users HTTP/1.1        →           HTTP/1.1 200 OK               │
│   Host: api.example.com                       Content-Type: application/json│
│                                               {"users": [...]}              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*Next Chapter: The Evolution of Deployment — From physical servers to the cloud*
