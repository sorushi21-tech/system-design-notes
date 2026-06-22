# Transport and Connections

Transport protocols move data between processes. This is where ports, connections, reliability, retransmission, and connection reuse matter.

## TCP

TCP is reliable, ordered, and connection-oriented.

It provides:

- Three-way handshake.
- Retransmission of lost packets.
- Ordered delivery.
- Flow control.
- Congestion control.

Used by HTTP/1.1, HTTP/2, databases, SSH, and email protocols. TCP is the default choice when correctness and ordering matter.

## UDP

UDP is connectionless and lightweight. It does not provide built-in reliability, ordering, retransmission, or connection state.

Used by:

- DNS.
- Streaming.
- Gaming.
- VoIP.
- WebRTC.
- QUIC/HTTP/3.

UDP is useful when low overhead matters or when the application/protocol handles reliability itself.

## QUIC

QUIC runs over UDP but adds reliability, encryption, multiplexing, and faster connection setup. HTTP/3 uses QUIC.

Benefits:

- Faster reconnects.
- Better behavior on mobile networks.
- Built-in encryption.
- Avoids some TCP head-of-line blocking problems.

## Connection Lifecycle

For a new HTTPS request over TCP, the client may pay for:

```text
DNS lookup
  -> TCP handshake
  -> TLS handshake
  -> HTTP request
  -> HTTP response
```

Connection setup adds latency. Reusing connections avoids paying the handshake cost for every request.

## Keep-Alive

Keep-alive lets a client reuse an existing TCP connection for multiple HTTP requests.

Benefits:

- Lower latency.
- Less CPU spent on handshakes.
- Fewer short-lived connections.
- Better throughput under load.

Risks:

- Idle connections can be closed by proxies or load balancers.
- Clients may reuse stale connections and see occasional failures.
- Too many open connections can exhaust resources.

## Connection Pooling

Backend services commonly use connection pools for:

- HTTP clients.
- Database clients.
- Redis clients.
- Message brokers.

Pool settings matter:

| Setting Problem | Result |
|-----------------|--------|
| Pool too small | Requests wait for a free connection |
| Pool too large | Downstream service can be overloaded |
| No timeout | Calls can hang and exhaust worker threads |
| No idle cleanup | Stale connections can fail later |

## Port Exhaustion

Clients use temporary ephemeral ports for outbound connections.

```text
Client 10.0.1.10:51544 -> Server 203.0.113.10:443
```

If a client opens too many short-lived connections, it can run out of usable ephemeral ports or accumulate many connections in `TIME_WAIT`.

Prevention:

- Reuse connections.
- Use connection pools.
- Set sensible timeouts.
- Avoid retry storms.
- Scale clients horizontally when needed.
