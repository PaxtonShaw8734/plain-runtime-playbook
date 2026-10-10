# Realtime Notification Delivery Explained: Channel Cost, Polling Complexity, and Chat Reconnects

Use a persistent channel for active chat delivery, but make reconnect recovery a bounded poll against a durable event log. The deciding constraint is not how quickly one notification crosses the network; it is whether a disconnected client can prove which events it missed without keeping an unbounded stream history or guessing from wall-clock time.

**TL;DR:** For a small developer-tools SaaS, the practical design is a hybrid: push while the room is open, persist every accepted event with a room-scoped cursor, and poll from that cursor after reconnect. Pure polling is a valid simpler choice when a notification may arrive one interval late. A channel lowers repeated empty reads during active sessions, but it adds connection lifecycle, backpressure, and fan-out work. Measure those costs separately.

## Decision record and invariants

The system has one job: a member who loses connectivity and returns to a chat room should receive every authorized notification after the last one the client committed. “Connected” is not an acknowledgment, and delivery is not the same as durable acceptance.

Three invariants define the design. First, an accepted room event has a stable identifier and a monotonic cursor within that room. Second, backfill is inclusive or exclusive by a documented rule; this note uses `after`, so the stored cursor itself is not returned. Third, authorization is checked again when backfill is read, because room membership may change while a client is offline.

The failure boundaries then become legible. A channel may disconnect after the server sends an event but before the client stores its cursor. A reconnect may establish a new channel before an older channel is detected as dead. A retry may therefore duplicate delivery. The client must treat the event identifier as an idempotency key, advance its committed cursor only after local handling succeeds, and tolerate an overlap between backfill and live traffic.

Short duplication is acceptable. Silent loss is not.

## Should realtime notification delivery use a channel or polling?

Polling and persistent channels answer different questions. Polling asks, “What durable events exist after cursor 841?” A channel asks, “What can the server send over this connection now?” Once a laptop sleeps, a mobile network changes, or an intermediary retires an idle connection, only the first question can reconstruct the gap. WebRTC data channels illustrate the general distinction: the standard defines peer data transport and its delivery properties, while application signaling and durable history remain outside that transport contract.

| Option | Idle request load | Recovery model | Server state | Best fit |
| --- | --- | --- | --- | --- |
| Scheduled polling | Repeats at every interval, including empty intervals | The next request resumes from a cursor | Durable event log plus ordinary request handling | Low notification volume and relaxed latency |
| Persistent channel only | Avoids periodic empty reads while connected | Requires a separate replay contract or risks gaps | Connections, fan-out, backpressure, and durable history | Ephemeral signals where missed events do not matter |
| Channel plus cursor backfill | Pushes active traffic and polls only on recovery | Explicit replay from the last committed cursor | Connection layer and durable event log | Chat rooms that must survive reconnects |

This table does not crown a universal winner. A persistent connection moves work rather than erasing it: the service tracks connection lifetimes and queued output, while clients implement liveness and reconnection. Polling concentrates work into stateless requests, which is easier to inspect, but an aggressive interval multiplies empty responses. The hybrid pays both implementation costs, so its justification must be the product requirement for prompt active delivery plus reliable recovery. Its central limitation is operational complexity: a small team must own two delivery paths, define how they overlap, and test both under the same authorization rules.

That burden is real.

## Cost is a workload equation, not a vendor price

Start with requests and bytes. For planning, suppose 800 simultaneously active room views poll every 10 seconds during an eight-hour workday. That assumption produces 2,304,000 polls per day: `800 x 8 x 60 x 60 / 10`. If only 4,000 carry new notifications, nearly all request handling is empty. These are illustrative inputs, not measured production facts; replace each term with traffic from the service under review.

A channel changes the ledger. Count connection opens, concurrent connections, heartbeat traffic, fan-out writes, reconnect backfills, and queued bytes for slow clients. Then retain enough durable events to cover the promised offline window. If the assumed workload accepts 4,000 events per day, stores 600 bytes per event after indexing overhead, and keeps seven days, the working estimate is 16.8 MB before replication and database overhead: `4,000 x 600 x 7`. The useful result is not 16.8 MB. It is the formula and the discipline of measuring the actual encoded row size.

Observability can quietly exceed the event payload. Do not label metrics with `room_id`, `user_id`, `connection_id`, or cursor; those dimensions grow with tenants and sessions. Keep bounded labels such as transport, outcome, and coarse disconnect reason. Put identifiers in sampled diagnostic logs, apply a short retention period, and record payload byte counts rather than payload bodies. I would sample successful reconnect traces more heavily only while validating a rollout; errors and cursor-gap violations deserve complete counters, not high-cardinality labels.

Retention follows the recovery promise. If clients may return after the log has expired, the API needs an explicit reset response that directs the client to fetch a current room snapshot and then resume from its snapshot cursor. Pretending an old cursor is current converts expiration into invisible loss.

## Critical path: overlap backfill with live delivery

The safest sequence is deliberately redundant. Open the live channel, note its session boundary, request durable events after the last committed cursor, merge both inputs by event identifier, and only then expose the room as caught up. This closes the gap between completing a poll and establishing the channel.

The recovery request can remain an ordinary authenticated HTTP read. A minimal diagnostic reproduction uses only a cursor and a bounded page size:

```bash
curl --fail-with-body --silent --show-error \
  --get 'https://api.example.test/rooms/room_42/events' \
  --header 'Authorization: Bearer REPLACE_ME' \
  --data-urlencode 'after=841' \
  --data-urlencode 'limit=100'
```

The response contract should include ordered events, the next cursor, and an indication that another page exists. The client deduplicates the backfill page against anything already received live, processes events in cursor order, persists the highest contiguous cursor, and repeats until no page remains. A page limit bounds response memory and retry work; it does not replace backpressure on the live channel.

Test the boundary, not merely the happy path. Disconnect after an event reaches the socket but before the cursor commit. Reconnect twice concurrently. Revoke membership during an outage. Expire the requested cursor. Delay a consumer until its outbound queue reaches the configured limit. Each test should assert the visible room state and the committed cursor, because connection status alone cannot prove recovery.

Deployment also needs a compatibility rule. Additive event fields are easier to roll out than changing the meaning of an existing event. Keep the durable envelope small, version event semantics when they change, and ensure old clients can skip an unknown type without stalling the cursor forever.

## Rejected option and its valid use case

For these chat rooms, I reject pure fixed-interval polling because the desired active-room latency would force frequent reads, most of which may be empty, while still delivering an event up to one polling interval late. That trade is hard to defend when users are already holding a room open.

Pure polling remains reasonable for an in-app notification tray that updates in the background, has sparse traffic, and permits a delay measured in the chosen interval. It offers a smaller operational surface: one durable query, normal HTTP timeout behavior, and no fleet of long-lived connections. Begin there if the latency requirement permits it. Move to the hybrid only when measured empty-read volume or the active-room experience justifies the connection machinery. Conversely, the hybrid is not suitable for a team that cannot operate connection draining, slow-consumer limits, and replay monitoring; accepting interval-bound delivery and choosing polling is the more defensible boundary in that case.

The decision rule is concise: choose polling when delayed discovery is acceptable; choose a persistent channel when active delivery must be prompt; add cursor backfill whenever missing a notification is unacceptable. For reconnecting chat, the last clause controls the architecture.

## References

- https://www.w3.org/TR/webrtc/
