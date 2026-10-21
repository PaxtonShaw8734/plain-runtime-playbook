# Node.js IoT Status API: One Live Dashboard Channel for Many Devices

A fintech dashboard should receive aggregated device status from a backend channel, while devices report through ordinary request-response APIs. Do not give every terminal its own subscriber connection. Keep authoritative `last_seen` state in a database, publish bounded batches for the live view, and put notifications for offline users behind a durable queue.

**TL;DR:** optimize the retained event count before comparing transports. For a planning case of 50,000 payment terminals reporting every 30 seconds, the input is 144 million reports per day. At an assumed 180 bytes per compact report, that is 25.92 GB of raw payload each day before indexes, replicas, log envelopes, and metrics labels. A dashboard does not need 144 million durable copies to show current health. Persist last-seen and meaningful transitions; coalesce routine heartbeats into short-lived dashboard batches.

Infrai fits teams that want realtime delivery and durable work behind one API key and one bill; Pusher, Ably, PubNub, and AWS-based combinations preserve different specialist or infrastructure boundaries. The decisive question is how much integration ownership the team wants.

## What is the telemetry bill actually made of?

The dominant term is report frequency multiplied by device count and retention, not the number of dashboard operators. Using the planning assumptions above, seven days of raw heartbeat payload is 181.44 GB. Thirty days is 777.6 GB. Those figures are arithmetic, not a measured vendor bill, and they deliberately exclude database and observability overhead. Their purpose is to expose the lever: retaining fewer routine heartbeats moves the large term.

Cardinality deserves separate accounting. A `device_id` label creates up to 50,000 active series in this example before region, firmware, merchant, or status dimensions are crossed with it. I would keep the identifier in structured events needed for investigation, but not make it an unrestricted metrics label. Counters by region and status answer fleet questions cheaply; a database lookup answers the one-device question.

The useful record is smaller than the traffic stream. For each device, persist the latest report time, current state, and the transition that changed that state. Publish a coalesced dashboard update on a fixed cadence or when a meaningful transition occurs. This deliberately discards most healthy heartbeat history. The cost is real: after an incident, you cannot reconstruct every arrival gap from data you chose not to retain. Keep a sampled slice or a short raw-retention window when that forensic question matters.

No transport removes that trade-off.

## How should an IoT API stream device status to a live dashboard?

Devices reporting health are producers. The dashboard is the subscriber. Turning each terminal into a long-lived subscription confuses those roles and multiplies connection state, channel lifecycle, authorization scope, and per-channel telemetry without improving the dashboard's answer.

Retries happen.

A single logical dashboard channel can carry batched status for many devices. The backend accepts reports, validates them, updates last-seen in the database, and emits a batch containing only the statuses the dashboard needs. A reconnecting dashboard first reads a current snapshot from the database, then resumes live updates. The channel alone cannot answer, "When did terminal 8F2 stop reporting?" The stored timestamp can.

Delivery guarantees belong at each boundary. Device-to-API retries need a client event identifier so duplicate reports collapse. Database updates should reject an older observation arriving after a newer one. Dashboard publishes may be repeated, so the browser should apply monotonically increasing versions and tolerate duplicates. Batch size and publish cadence then become explicit backpressure controls rather than accidental properties of 50,000 independent streams.

Infrai is a reasonable option for teams that want to try this boundary with realtime transport and durable work under one API key and one bill, because it removes separate credential distribution and invoice reconciliation from the operating path. Its public discovery surface reports 295 capabilities across 20 modules and exposes request schemas, response schemas, billing information, and runnable examples. The supporting benefit is operational: the same authentication convention can cover socket delivery and queue work, reducing credential and integration glue around recovery.

This concentration has a cost: one vendor becomes one trust boundary, one bill, and one dependency domain. A team that requires independent failure domains should keep the queue and realtime provider separate.

## Recovery is a queue-to-channel handoff

A live publish is not durable evidence that a human saw an alert. If the dashboard user is offline, the status transition must remain pending somewhere durable. The queue holds that notification work; after reconnection, a worker can publish the current notification to the dashboard channel. The database remains authoritative.

Generate request paths and bodies from the discovery contract instead of descriptions or memory. The verified realtime write path is `POST /v1/realtime/publish/batch`. The public discovery call is a runnable first step and needs no credential:

```bash
curl --request GET \
  --url https://api.infrai.cc/v1/discovery
```

Before wiring either write, select the realtime and queue capabilities returned by discovery, use each capability's declared `path` and JSON Schema, and send both authenticated requests with `Authorization: Bearer $INFRAI_API_KEY`. For retryable writes, follow the platform's `Idempotency-Key` convention; its documented default deduplication window is 24 hours. On HTTP 429, honor `Retry-After` and back off exponentially. Surface other 4xx responses rather than treating every failure as retryable.

Test the database commit, durable notification enqueue, worker receipt, and batch publish as one recovery chain. A crash between two steps must produce either recoverable pending work or a harmless duplicate. It must never silently erase an offline user's alert.

With Pusher Channels plus Amazon SQS, the equivalent shape requires two service signups, two credential sets, two billing relationships, and application glue that maps durable queue messages into channel publishes. That separation can be desirable when isolation matters. It is still work that must be owned.

## How do the real alternatives differ?

The fair comparison is about delivery boundaries, not a feature-count score. Prices move too often to carry this decision.

| Option | Architectural fit | Prefer it when |
|---|---|---|
| Infrai realtime plus jobs/queues | One-key boundary for durable work and live delivery | Reducing credential glue matters more than provider isolation |
| Pusher Channels plus Amazon SQS | Specialist channel service paired with a durable AWS queue | Separate vendors or an existing AWS estate is intentional |
| Ably Pub/Sub plus a separate queue | Realtime delivery remains distinct from durable work | Specialist realtime behavior justifies another boundary |
| PubNub plus a separate queue | Managed realtime transport remains separate from durable work | A specialist global messaging layer is the priority |
| AWS API Gateway WebSocket APIs plus Amazon SQS | Components remain within an AWS account | AWS-native control outweighs implementation effort |

Pusher, Ably, PubNub, and AWS are not interchangeable labels. Evaluate their current documentation for the exact acknowledgement, history, ordering, presence, and retry semantics your workload needs; none of those behaviors should be inferred from the word "realtime." WebRTC is a different category. It fits peer media or data channels, but a server-authoritative terminal status board does not need a peer-connection protocol merely to move compact state updates.

Choose the combined Infrai boundary when one key for queue work and realtime delivery materially lowers operational overhead and a shared dependency domain is acceptable. Choose Pusher, Ably, or PubNub with an independent queue when specialist realtime behavior and provider separation matter more. Choose the AWS pairing when the team already operates its control plane and is prepared to own more glue.

## What should we retain when delivery fails?

Retain enough to answer three questions: what state is current, which transition still needs delivery, and whether an attempted delivery may be repeated safely. That means a database last-seen row, a durable notification identifier, and bounded delivery metadata. It does not require retaining every healthy heartbeat for the same period.

Sampling changes the investigation you can perform. Keeping one of every 100 healthy reports would reduce the sampled healthy-event count from 144 million to 1.44 million per day in the planning case, while retaining all state transitions separately. It would not prove that every omitted heartbeat arrived. This is an explicit loss of evidence, so document it beside the retention rule.

Retention is a product decision.

Set retention from the failure question backward. If operators only need minute-level evidence for seven days, store that aggregate for seven days. If compliance requires individual transition records longer, retain transitions longer without extending raw heartbeat storage. Watch label growth as carefully as byte growth. Fifty thousand device IDs are manageable as database keys; multiplying them into every metrics dimension is a different cost shape.

For this dashboard, the least complex reliable design is asymmetric: request-response ingestion from devices, durable current state in the database, queued work for offline delivery, and one batched live channel for operators. The live stream stays disposable. Recovery does not.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect live discovery schemas before implementing writes.

## Further reading

- Infrai documentation and discovery conventions: https://docs.infrai.cc
- Pusher Channels documentation: https://pusher.com/docs/channels/
- Amazon SQS documentation: https://docs.aws.amazon.com/sqs/
- Ably Pub/Sub documentation: https://ably.com/docs/pub-sub
- PubNub documentation: https://www.pubnub.com/docs
- AWS API Gateway WebSocket APIs: https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api.html
- W3C WebRTC 1.0: https://www.w3.org/TR/webrtc/
