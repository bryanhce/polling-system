# Context
The application requires live polling. Here we explore different solutions to meet this need.

# Strategies considered

## Short polling

### Definition

Short polling is a client-driven communication pattern where the browser sends repeated HTTP requests to the server at fixed time intervals (eg five seconds) to inspect for new data. The server processes each request as a standalone transaction and responds immediately with either new information or an empty payload.

### Pros

* Simple to implement with standard HTTP request libraries and basic caching infrastructure.
* Stateless design allows requests to distribute across standard load balancers without sticky sessions.

### Cons

* Creates high request overhead and consumes server CPU when data has not changed.
* Introduces a latency delay equal to the polling interval between data updates.
* Scales poorly under high concurrent user counts due to the constant flood of HTTP requests.
* Consumes mobile client battery and network bandwidth unnecessarily.

---

## Long polling

### Definition

Long polling is an HTTP technique where the client requests data from the server, and the server keeps the request open until new data arrives or a timeout occurs. Once the server sends data or the connection reaches its timeout limit, the client immediately initiates a new request.

### Pros

* Reduces the latency between data creation on the server and receipt on the client compared to short polling.
* Operates over standard HTTP/HTTPS infrastructure without custom protocols or firewall adjustments.

### Cons

* Re-establishes a full HTTP connection for every individual message.
* Increases memory utilization on web servers that maintain open connection threads.
* Complicates connection state handling when network interruptions drop pending requests.
* Produces message ordering issues if multiple requests race during network reconnection.

### Common use cases

* Simple web chat widgets on legacy server architectures.
* Notification systems in environments where proxy configurations block persistent streaming protocols.

---

## Server-sent events

### Definition

Server-sent events (SSE) is a web standard built on HTTP that allows a server to push real-time text updates to clients over a single, persistent connection. The browser uses the native `EventSource` API to receive incoming events automatically.

### Common data delivery patterns

#### 1. Full payload

The server sends the complete resource inside the event payload. When a user submits an answer, the event contains the entire answer object with its identifier, author details, body text, and timestamp. The client directly appends this object to its local state without extra network calls. This pattern is effective when individual payloads remain small.

#### 2. Cache invalidation signal

The server pushes a lightweight notification containing only an entity ID or an update flag. The client then initiates a standard REST query to retrieve the fresh dataset. This approach decouples real-time transport from data serialization, and it lets the client use standard HTTP caching for the retrieved data.

#### 3. Delta updates

The server transmits only the specific fields that changed since the last known state. For a Q&A interface, this includes answer edits or upvote count increments. Delta updates reduce network bandwidth, although the client must execute state patches to maintain consistency.

### Pros

* Operates over standard HTTP/1.1 and HTTP/2, which simplifies deployment through existing reverse proxies and CDNs.
* Includes native browser support for automatic reconnection and event IDs.
* Consumes fewer server resources than bidirectional protocols because communication is strictly unidirectional.
* Supports custom event types directly through standard protocol framing.

### Cons

* Restricts communication to a single direction from server to client.
* Limits maximum open connections to six per domain when operating on HTTP/1.1 without HTTP/2 multiplexing.

### Common use cases

* Live feeds, news tickers, and public status monitors.
* Real-time text generation interfaces, such as AI model output streaming.

---

## Websockets

### Definition

WebSocket is a transport protocol that provides full-duplex, bidirectional communication channels over a single TCP connection. After an initial HTTP handshake, the connection upgrades to the WebSocket protocol (`ws://` or `wss://`). Both client and server can then exchange data frames independently at any time.

### Pros

* Delivers low-latency data exchange in both directions with minimal frame overhead.
* Supports both text strings and binary data formats.

### Cons

* Bypasses standard HTTP request-response flow. This requires custom connection lifecycle management and heartbeat mechanisms.
* Creates configuration hurdles with corporate firewalls and reverse proxies that do not maintain long-lived TCP connections cleanly.
* Adds operational overhead for horizontal scaling. System operators must use pub/sub backplanes like Redis to sync state across nodes.
* Lacks native automatic reconnection in browser clients. Developers must write custom reconnect logic manually.

### Common use cases

* Multiplayer online browser games requiring sub-50ms round-trip latency.
* Collaborative editing platforms where multiple participants type simultaneously into a shared document.

---

## Final decision

Server-sent events is the recommended choice for this Q&A application.

In this architecture, users submit a single answer through a standard HTTP POST endpoint, while other participants view incoming answers in real time. Because data creation is a one-time transactional action per user, bidirectional streaming is unnecessary. SSE satisfies the unidirectional read requirement efficiently.

Evaluating the alternative strategies clarifies why SSE fits best:

* **Short polling** creates unnecessary server load through constant polling cycles. It also delays answer delivery until the next polling tick.
* **Long polling** requires the server to repeatedly open and close HTTP connections after every answer delivery. This cycle adds redundant TCP and TLS handshakes.
* **WebSockets** introduces excess architectural complexity for a unidirectional feed. Adopting WebSockets requires custom reconnection handlers and specialized load balancer configurations. It also requires separate authentication flows for the socket connection.

SSE provides native browser reconnection, standard HTTP integration, and low bandwidth usage. Combining SSE with full payload delivery distributes new answers to users promptly with minimal operational friction.