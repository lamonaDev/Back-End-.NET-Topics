# HTTP/1.1 vs HTTP/2 vs HTTP/3

## Overview

HTTP (Hypertext Transfer Protocol) has evolved significantly since its inception. Understanding the differences between HTTP/1.1, HTTP/2, and HTTP/3 is crucial for making architectural decisions and optimizing web application performance.

## Protocol Evolution Timeline

```
HTTP/0.9 ─── 1991 ──── The original protocol (one-line requests)
    │
HTTP/1.0 ─── 1996 ──── Headers, status codes, content types
    │
HTTP/1.1 ─── 1997 ──── Persistent connections, pipelining
    │
HTTP/2 ───── 2015 ──── Multiplexing, header compression, server push
    │
HTTP/3 ───── 2022 ──── QUIC transport, improved performance
```

## Protocol Overview

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| Transport | TCP | TCP | QUIC (UDP-based) |
| Released | 1997 | 2015 | 2022 |
| Multiplexing | One request per connection (or pipelining, rarely used) | Multiple streams per connection | Independent streams (no TCP HoL blocking) |
| Head-of-Line Blocking | Per-connection blocking | Solved at HTTP layer; TCP-level blocking remains | Fully solved — packet loss only affects one stream |
| Header Compression | None (repeated verbatim) | HPACK compression | QPACK compression |
| Server Push | - | (deprecated in practice) | Removed |
| Connection Setup | TCP + TLS: 2-3 RTT | TCP + TLS: 2-3 RTT | QUIC: 0-1 RTT (0-RTT resumption) |
| TLS Required | No (HTTPS optional) | De facto TLS only | Always TLS 1.3 |
| Mobile Networks | Poor (TCP retransmits) | Moderate | Excellent (QUIC handles IP changes) |
| Browser Support | Universal | ~97% global | ~95% global (2024) |

## Key Concepts Explained

### Multiplexing

**HTTP/1.1**: Browser opens 6 parallel TCP connections to bypass the single-request limitation:

```
Connection 1: GET /style.css ──────────────────────────→
Connection 2: GET /script.js ──────────────────────────→
Connection 3: GET /image1.jpg ──────────────────────────→
Connection 4: GET /image2.jpg ──────────────────────────→
Connection 5: GET /image3.jpg ──────────────────────────→
Connection 6: GET /api/data ──────────────────────────→
```

**HTTP/2**: Single TCP connection with multiple streams:

```
TCP Connection 1:
  Stream 1: GET /style.css ──────────────────────────→
  Stream 2: GET /script.js ──────────────────────────→
  Stream 3: GET /image1.jpg ──────────────────────────→
  (All concurrent, no connection limit!)
```

**HTTP/3**: Same as HTTP/2, but with QUIC transport:

```
QUIC Connection:
  Stream 1: GET /style.css ──────────────────────────→
  Stream 2: GET /script.js ──────────────────────────→
  Stream 3: GET /image1.jpg ──────────────────────────→
  (Independent streams, no blocking!)
```

### Head-of-Line Blocking Explained

**HTTP/1.1**: A browser opens 6 parallel TCP connections to the same server (browser workaround). Within each, requests queue — a slow response blocks all following ones on that connection.

**HTTP/2**: One TCP connection with multiplexed streams. But if a TCP packet is lost, *all streams* wait for TCP retransmission — this is TCP-level HoL blocking.

**HTTP/3**: Uses QUIC (over UDP). Each stream is truly independent. A lost packet only delays its own stream, not others. Critical for high-latency mobile networks.

### Visual Comparison of HoL Blocking

```
HTTP/1.1:                    HTTP/2:                      HTTP/3:
┌─────────────────┐         ┌─────────────────┐        ┌─────────────────┐
│ Request A ██████ │         │ Stream 1 ██████  │        │ Stream 1 ██████  │
│ Request B ░░░░░░ │Wait     │ Stream 2 ░░░░░░░  │Wait    │ Stream 2 ░░░░░░░  │
│ Request C ░░░░░░ │         │ Stream 3 ████░░░░ │        │ Stream 3 ████░░░  │
│                 │         │                 │        │                 │
│ TCP Level       │         │ TCP Level       │        │ QUIC Level      │
│ Request A lost! │         │ Packet lost!    │        │ Packet lost!    │
│                 │         │                 │        │                 │
│ Blocked ░░░░░░░ │         │ Blocked ░░░░░░░░  │        │ Stream 3 alone  │
│ Blocked ░░░░░░░ │         │ Blocked ░░░░░░░░  │        │ continues!  ████│
└─────────────────┘         └─────────────────┘        └─────────────────┘
```

## Header Compression

### HTTP/1.1 — No Compression
Every request repeats all headers:

```
GET /api/users HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer abc123
Cookie: session=xyz

[Response...]

GET /api/products HTTP/1.1
Host: api.example.com                    <-- Repeated!
Accept: application/json                 <-- Repeated!
Authorization: Bearer abc123             <-- Repeated!
Cookie: session=xyz                      <-- Repeated!
```

### HTTP/2 — HPACK

Uses index tables and delta encoding:
- Static table: Common headers indexed (method, path, etc.)
- Dynamic table: Session-specific headers indexed
- Huffman encoding for strings

```
First request sends: all headers
Subsequent requests: send only new/changed headers
Headers already indexed by number → tiny payload
```

### HTTP/3 — QPACK

Similar to HPACK but optimized for QUIC streams:
- Allows streams to be independent (no blocking on header updates)
- More efficient than HPACK in lossy conditions

## Connection Setup Latency

```
HTTP/1.1 over HTTPS:
TCP SYN → 1 RTT → TCP SYN-ACK → 1 RTT → TLS Handshake → 1-2 RTT → First byte
         ≈ 30ms                                          ≈ 30-60ms
Total: ~60-90ms for first byte

HTTP/2 over HTTPS:
Same TCP/TLS as HTTP/1.1
Total: ~60-90ms for first byte

HTTP/3 over QUIC:
QUIC Initial → 0 RTT (resumed) or 1 RTT (new) → First byte
         ≈ 0-15ms
Total: ~15-30ms for first byte (up to 3x faster!)
```

## QUIC Advantages

### Connection Migration
QUIC supports changing IP addresses without dropping the connection:

```
User on mobile:
  - WiFi → Cellular (IP changes)
  - HTTP/1.1/2: Connection drops, must reconnect
  - HTTP/3: Connection persists, seamless handover
```

### 0-RTT Resumption
Returning visitors can send data immediately:

```
First visit:
  Client → QUIC Initial (1 RTT) → Server
  (TLS handshake completes)

Second visit:
  Client → QUIC Initial + Encrypted Data (0 RTT)
  (Data encrypted with cached keys from first visit)
```

## Decision Guide

| Scenario | Best Choice | Why |
|----------|-------------|-----|
| High-traffic API, many small requests | HTTP/2 | Multiplexing reduces connection overhead |
| Mobile-first app, lossy networks | HTTP/3 | QUIC resilient to packet loss and IP changes |
| Legacy internal microservice | HTTP/1.1 or HTTP/2 | Simpler, well-supported everywhere |
| Video/media streaming | HTTP/3 | Lower latency, no HoL blocking per chunk |
| Public-facing web app (2024+) | HTTP/2 minimum, HTTP/3 preferred | CDNs and browsers support both |
| Internal server-to-server gRPC | HTTP/2 | gRPC is built on HTTP/2 framing |
| Real-time applications | HTTP/3 | Faster connection, better on lossy networks |

## ASP.NET Core HTTP/2 and HTTP/3 Support

### Enable HTTP/2 and HTTP/3

```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(5001, listenOptions =>
    {
        listenOptions.UseHttps();
        listenOptions.Protocols = HttpProtocols.Http1AndHttp2AndHttp3;
    });
});
```

### Kestrel Protocol Configuration

```csharp
// Listen on specific protocols
options.Listen(IPAddress.Any, 5000, listenOptions =>
{
    listenOptions.Protocols = HttpProtocols.Http1;  // HTTP/1.1 only
});

options.Listen(IPAddress.Any, 5001, listenOptions =>
{
    listenOptions.Protocols = HttpProtocols.Http1AndHttp2;  // HTTP/1.1 + HTTP/2
});

options.Listen(IPAddress.Any, 5002, listenOptions =>
{
    listenOptions.Protocols = HttpProtocols.Http1AndHttp2AndHttp3;  // All
});
```

## Browser Support (2024)

```
HTTP/1.1:  100% (Universal)
HTTP/2:    97% (Excellent)
HTTP/3:    95% (Very Good)
           └─ Chrome, Firefox, Safari, Edge all support HTTP/3
```

## Performance Impact Summary

| Metric | HTTP/1.1 | HTTP/2 | HTTP/3 |
|--------|----------|--------|--------|
| Page Load Time | Baseline | 30-50% faster | 40-70% faster |
| Latency | Baseline | 30% lower | 50% lower |
| Mobile Performance | Poor | Better | Best |
| Connection Reuse | Limited (6 connections) | Unlimited | Unlimited |

## References

- [Microsoft Docs — HTTP/2 in Kestrel](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel/http2)
- [Microsoft Docs — HTTP/3 in Kestrel](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel/http3)
- [HTTP/3 Explained — Daniel Stenberg](https://http3-explained.haxx.se/)
- [Cloudflare — HTTP/3: Past, Present, Future](https://blog.cloudflare.com/http3-the-past-present-and-future/)