# What Happens When You Type https://www.google.com and Press Enter

It feels instant. You type eight characters, hit Enter, and a page appears.
But in that fraction of a second your request crosses a dozen systems, gets
encrypted, routed, filtered, balanced, executed and queried. This is the
classic full-stack interview question, and it is a good one, because
answering it well means you understand how the web actually works rather
than just how to use it.

Let's walk the whole path.

---

## 1. DNS: turning a name into an address

Computers do not route traffic by names. They route by IP addresses. So the
first thing your browser must do is translate `www.google.com` into something
like `142.250.185.4`.

It does not ask the internet immediately. It checks a series of caches first,
in order:

1. **The browser's own DNS cache** — did we just look this up?
2. **The operating system cache**, and the `hosts` file, which can hard-code
   a name to an address locally.
3. **The resolver** configured for your machine — usually your ISP's, or a
   public one like `8.8.8.8`.

If none of them know the answer, the resolver does a **recursive lookup**:

- It asks a **root name server**: "who handles `.com`?"
- The root replies with the **TLD name servers** for `.com`.
- It asks a `.com` server: "who handles `google.com`?"
- That server replies with Google's **authoritative name servers**.
- It asks one of those: "what is `www.google.com`?"
- The authoritative server returns the **A record** (IPv4) or **AAAA record**
  (IPv6) — the actual IP address.

The answer is cached at every level along the way for the duration of its
**TTL** (time to live), which is why the second visit is so much faster than
the first.

Your browser now has an IP address.

---

## 2. TCP/IP: opening a reliable pipe

The browser knows *where* to send the request. Now it needs a connection.

**IP** (Internet Protocol) handles addressing and routing: it wraps the data
in packets stamped with a source and a destination address, and routers
forward them hop by hop across the internet until they reach the destination
network. IP alone makes no promises — packets can arrive out of order, be
duplicated, or vanish entirely.

**TCP** (Transmission Control Protocol) sits on top and makes the connection
reliable. It numbers every byte, acknowledges what it receives, retransmits
what gets lost, reassembles everything in order, and adjusts its sending rate
to avoid congesting the network.

A TCP connection starts with the **three-way handshake**:

```
Client  --- SYN     --->  Server      "I want to talk, my sequence number is X"
Client  <-- SYN/ACK ---   Server      "Go ahead, mine is Y, and I got your X"
Client  --- ACK     --->  Server      "Got your Y. We're connected."
```

Because the URL is `https://`, the browser connects to **port 443** — the
standard port for HTTPS. Port 80 is plain HTTP; 443 is the encrypted one.
An IP address plus a port is called a **socket**, and that pair is what
uniquely identifies the endpoint of the conversation.

---

## 3. The firewall: deciding what is allowed through

Before the packets reach any application, they pass through firewalls — often
several. There may be one on your own machine, one on your home router, one
at the edge of Google's network, and one in front of each internal tier.

A firewall inspects traffic against a rule set and allows or blocks it based
on source address, destination address, port and protocol. Its purpose is to
shrink the **attack surface**: only the ports that genuinely need to be
public — 443 here — are reachable from the internet. Everything else, the
database port, the SSH port, the internal application ports, is blocked from
the outside even though the service is running.

If one machine behind the firewall is ever compromised, strict rules between
tiers also limit how far the attacker can move.

Our packets are destined for port 443, which is open, so they pass.

---

## 4. HTTPS/SSL: encrypting the conversation

The TCP connection is open, but it is a clear-text pipe. Anyone between you
and the server — someone on the same Wi-Fi, an ISP, a compromised router —
could read or modify everything. That is unacceptable for a login form, a
session cookie, or a search query.

So the browser and the server perform a **TLS handshake** (TLS is the modern
successor to SSL; people still say "SSL" out of habit):

1. The client sends a `ClientHello` listing the TLS versions and cipher
   suites it supports, plus a random value.
2. The server replies with a `ServerHello` choosing a version and cipher, and
   sends its **certificate**.
3. The browser **validates the certificate**: is it issued by a Certificate
   Authority the browser trusts? Is it still valid? Does the name on it
   actually match `www.google.com`? If any check fails, you get the big red
   warning page.
4. The two sides perform a key exchange (typically ECDHE) and derive a shared
   **session key**. Modern suites give **forward secrecy**: even if the
   server's private key leaks later, this session cannot be decrypted
   retroactively.
5. Both switch to symmetric encryption using that session key.

TLS gives three guarantees at once: **confidentiality** (nobody can read it),
**integrity** (nobody can tamper with it undetected), and **authentication**
(the server is who it claims to be).

Only now does the browser send the actual request:

```http
GET / HTTP/1.1
Host: www.google.com
User-Agent: Mozilla/5.0 ...
Accept: text/html,application/xhtml+xml
Accept-Encoding: gzip, deflate, br
```

---

## 5. The load balancer: spreading the work

A site at Google's scale is never a single machine. The IP your DNS lookup
returned belongs to a **load balancer**, not to a web server.

The load balancer is the single public entry point. It receives the
connection and forwards it to one of many backend servers. How it chooses
depends on the configured algorithm:

- **Round Robin** — each new request goes to the next server in the list.
  Simple, and fair when the servers are identical.
- **Least Connections** — send it to whichever server is currently handling
  the fewest requests. Better when request durations vary a lot.
- **IP Hash / sticky sessions** — hash the client's IP so the same client
  always lands on the same server. Useful when session state lives on the
  server.

It also runs continuous **health checks**. If a backend stops responding, the
load balancer takes it out of the pool and the users never notice.

There are two common topologies. In **Active-Active**, every node serves
traffic simultaneously — all the hardware is working and capacity degrades
gracefully when one fails. In **Active-Passive**, one node serves while a
standby waits to take over. Active-Passive is simpler; Active-Active uses
your capacity properly.

And because the load balancer itself would otherwise be a single point of
failure, it is usually deployed as a pair sharing a **virtual IP**, so the
address survives the loss of either node.

---

## 6. The web server: the HTTP specialist

The request arrives at a **web server** — Nginx or Apache.

Its job is the HTTP protocol itself: parsing requests, managing thousands of
concurrent connections efficiently, handling keep-alive, compression, caching
headers, and serving **static content** — HTML files, CSS, JavaScript bundles,
images — straight off disk, as fast as possible.

If the request is for something static, the web server answers it alone and
we are nearly done. If the response has to be *computed*, the web server acts
as a **reverse proxy** and hands the request to the application server.

---

## 7. The application server: running the code

The **application server** is where your actual program lives — the Python,
Node.js, Ruby or Java process that implements the business logic.

It takes the parsed request, figures out what is being asked, runs the code
that answers it: checks whether you are authenticated, applies permissions,
performs the search, formats the results, renders the template.

The distinction is worth being precise about in an interview: **the web
server speaks HTTP; the application server runs your program.** One deals
with the protocol and the connection, the other with the logic. Separating
them lets each be scaled and tuned for its very different workload — the web
tier is network- and I/O-bound, the application tier is CPU- and memory-bound.

To do its job, the application almost always needs data it does not have.

---

## 8. The database: where the data actually lives

So the application queries the **database** — MySQL, PostgreSQL, MongoDB,
whatever fits.

The database is the system of record. It stores data durably, enforces
constraints, handles transactions so concurrent writes do not corrupt each
other, and answers queries efficiently using indexes.

At scale it is rarely a single node either. A common arrangement is a
**Primary-Replica** cluster: the Primary is the only node that accepts
writes, and it streams every change through its binary log to one or more
Replicas, which replay those changes on their own copies. The application
sends writes to the Primary and can spread reads across the Replicas, which
takes load off the Primary — at the cost of a small **replication lag**, so
data written a millisecond ago might not be readable from a Replica quite
yet.

Frequently requested data is usually cached in something like Redis or
Memcached first, so the database is only consulted when the cache misses.

---

## 9. The journey back

The database returns rows. The application builds the HTML. The web server
wraps it in an HTTP response, compresses it, and sends it back through the
load balancer, encrypted with the TLS session key, across the internet, to
your browser.

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Content-Encoding: gzip
```

Your browser then parses the HTML into the **DOM**, parses the CSS into the
**CSSOM**, combines them into a render tree, computes the layout of every
element, and paints pixels to the screen — firing off more requests for every
stylesheet, script, font and image it encounters along the way, each one
repeating a (now much faster, thanks to caching and connection reuse) version
of this entire journey.

---

## Why this question is asked

Because it has no bottom. You can answer it in thirty seconds or spend an
hour on the TLS handshake alone. It lets an interviewer find the edge of what
you know, in whichever layer they care about — DNS and load balancing for an
SRE role, the render path for a front-end role, query planning for a data
role.

A good instinct before answering: **ask which layer they want you to go deep
on.** It shows you understand the question is a map, not a single road.

---

*Written as part of the Holberton School software engineering program.*
