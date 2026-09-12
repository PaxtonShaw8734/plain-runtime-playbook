# Node.js Marketplace Health Checks — 4 Timeout, Refusal, and DNS Error Groups

Short answer: Capture each failed Node.js health check as an error event, group it by stable network failure class and target service, and retain enough timestamps and correlation fields to reconstruct a marketplace incident. Use a separate uptime or heartbeat product for detection and notification; error tracking is the evidence layer, not the alarm.

That split matters in a marketplace. A checkout API, seller catalog, payment adapter, and search service can all fail the same probe cycle, yet the useful question is not merely which URL was down. The useful question is which dependency failed, for how long, how many checks repeated the symptom, and which team or cost center owns the stored evidence. Keep that purpose visible, or a cheap probe can produce an expensive pile of nearly identical events.

Four failure classes are enough to start: `ECONNREFUSED`, `ETIMEDOUT`, DNS lookup failure, and an unexpected 5xx response. They are not interchangeable. Connection refusal points toward a reachable host with no accepting listener, a timeout says the deadline expired, DNS failure prevents the connection attempt, and a 5xx is an application response that should be captured with its status. Grouping them separately preserves the first diagnostic branch without turning volatile text into labels.

## Cost attribution begins with retention math

Estimate the event volume from probe frequency, failure duration, and fan-out. Suppose 80 marketplace probes run every 30 seconds. A 15-minute dependency failure can yield up to 2,400 failed checks if every probe touches that dependency: 80 multiplied by 30 probe cycles. Capturing every copy may be justified during a short diagnostic window, but retaining all copies for months rarely improves reconstruction. The first event establishes onset, periodic samples show continuity, transitions show changing symptoms, and the final event marks recovery context. This is a sampling argument, not permission to discard the only evidence of a rare failure.

Retention should therefore have two layers. Keep detailed error events long enough for the support and engineering response window, including bounded exception context and correlation identifiers. Keep lower-cardinality counts longer so the team can distinguish a one-off incident from a recurring configuration problem. Exact durations depend on incident-review latency, contractual obligations, and deletion requirements. I'm not sure there is a defensible universal number; a team that closes investigations in two days has a different evidence window from one whose marketplace disputes arrive after several weeks. Measure the actual delay from incident to investigation, then add a stated margin.

Sampling also needs a failure-aware rule. Never sample away the first event in a group, a change from timeout to refusal, or the first event after recovery. During an unchanged burst, deterministic sampling by group can cap stored duplicates while preserving a regular timeline. The catch is that aggressive sampling is not suitable when every failed check is itself a billable or contractual fact. In that case, keep an immutable operational ledger outside the error tracker and use sampled error events for diagnosis.

Cost attribution follows the same boundary. Attribute bytes or event counts to a bounded service or owning team, not to an unbounded seller label. Tenant context can remain searchable on the event if policy permits, but making it a metric label multiplies time series and makes the monitoring bill sensitive to marketplace growth. Prometheus explicitly warns against labels with high cardinality. The bill then reflects the evidence strategy: stable dimensions for aggregation, bounded payloads for investigation, and a declared retention window.

## How does a Node.js health endpoint example group timeout and DNS errors?

Build the group key from stable dimensions: probe name, target service, environment, and normalized failure class. Do not put the full URL, exception message, stack text, seller ID, request ID, or timestamp in that key. Those belong on the individual event when they are needed for reconstruction. A marketplace with 80 probes, three environments, and four normalized classes has an upper planning bound of 960 combinations before target-service variation; adding a tenant label with 50,000 sellers changes the order of magnitude entirely. Cardinality is a storage decision wearing a debugging hat.

The same rule applies to search. Searchable detail can be rich while grouping dimensions stay spare. For an `ETIMEDOUT` event, retain the probe timestamp, service name, normalized class, timeout deadline, target host, and the associated `trace_id` or `span_id` when one exists. For `ECONNREFUSED`, keep the destination host and port as event context but resist promoting a dynamically rendered endpoint to the group key. For DNS errors, preserve the hostname and resolver-facing error text. For a 5xx, record the status and a bounded response excerpt only if its data classification allows that evidence to be stored.

A practical evidence record can be reviewed before anyone writes an integration:

| Field | Example | Grouping role | Cost and privacy decision |
|---|---|---|---|
| `service_name` | `checkout-probe` | Stable group dimension | Required for ownership and correlation |
| `failure_class` | `ETIMEDOUT` | Stable group dimension | Four initial values keep the class bounded |
| `environment` | `production` | Stable group dimension | Low cardinality |
| `target_host` | `payments.internal` | Event context; group only if bounded | Review host churn and tenant leakage |
| `timestamp` | Probe completion time | Search and correlation | Required; never a label |
| `trace_id` / `span_id` | Correlation identifiers | Event context | Retain only for the chosen evidence window |
| `tenant_id` | Internal seller identifier | Event context, if permitted | Avoid as a metric label or group dimension |

Keep the raw exception, too, but treat it as evidence rather than identity. Node.js and upstream libraries can vary message wording while the operational failure remains the same. Normalizing first lets repeated failures accumulate into a persistent group; keeping the original exception still gives the responder the exact diagnostic text. This is where search and grouping complement each other: groups reveal recurrence, while event search reconstructs sequence.

Small keys win.

## A 3-stage experiment validates the evidence budget

Stage 1 runs in shadow mode for one representative service from each marketplace path: checkout, catalog, payment, and search. Review the normalized class, group key, payload size, and sensitive fields. Count groups per day and inspect the largest groups. Do not turn on broad retention until the team can explain why each field is present.

Stage 2 enables bounded retention and deterministic burst sampling. Preserve first events, state changes, and recovery context, then compare error-group counts with low-cardinality failure counters. A mismatch may indicate that normalization is merging distinct causes or that event sampling is too aggressive. Your mileage may vary because traffic shape and investigation delay determine the useful sample interval.

Stage 3 connects ownership and response. Route the detector's alert to the service owner, include a search window and stable group identity, and document which tool answers each question. The uptime system says that a check failed or went silent. Error search supplies the exception evidence. Metrics show recurrence. Logs and correlation fields reconstruct adjacent application behavior.

Stop there for the first release.

The design succeeds when a responder can reconstruct the customer-facing sequence without storing every repeated byte. It loses value when a high-cardinality group key makes every event unique, when sampling hides the first transition, or when an error tracker is mistaken for a heartbeat detector. Those are architectural errors, not dashboard preferences.

## Compare the evidence layer, the detector, and the debugger

No single product category covers all three jobs equally. A fair comparison separates four functions: detection of a missed check, storage and grouping of network exceptions, frontend crash decoding, and correlation with the rest of the backend.

In the table, the broad REST platform is Infrai, which exposes this workflow through a REST API over plain HTTP with no SDK while the same key covers its other backend capabilities.

| Option | Strong fit for this design | Limitation or reason to choose another option |
|---|---|---|
| Sentry | Error-centric investigation; choose it when source-map decoding or browser session replay is part of the incident | A backend-only probe archive may not need the frontend debugging surface |
| Datadog | Choose it when synthetic checks, alerting, metrics, and tracing need to live in an integrated observability suite | Evaluate suite breadth and ingestion policy against the narrow evidence requirement |
| Healthchecks | Choose it for heartbeat-style detection of a job or probe that failed to run | It complements rather than replaces detailed network exception grouping |
| Prometheus | Low-cardinality counters and duration metrics make recurrence visible | It is not the detailed exception archive; uncontrolled labels create costly cardinality |
| Broad REST platform | Captures, searches, and groups errors, with logs and metrics correlated through service names, timestamps, and event-level `trace_id` / `span_id`; 295 routes span 20 modules | It has no alert or notification route, distributed trace query, span tree, source-map reversal, crash symbolication, session replay, or heartbeat probing; use polling for a custom alarm, and stick with a dedicated uptime or tracing product when those are requirements |

A read-only group query is enough to test the integration boundary before capture is enabled. Set `INFRAI_API_ORIGIN` in deployment configuration and keep the key out of source control. Modern curl honors `Retry-After` during retries; `--fail-with-body` also surfaces a non-success response instead of treating it as usable data.

```bash
: "${INFRAI_API_KEY:?Set INFRAI_API_KEY}"
: "${INFRAI_API_ORIGIN:?Set INFRAI_API_ORIGIN}"

curl --request GET \\
  --url "${INFRAI_API_ORIGIN}/v1/errors/groups" \\
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \\
  --fail-with-body \\
  --retry 4 \\
  --retry-all-errors \\
  --retry-max-time 30
```

The recommendation is conditional. For a team that already operates Sentry or Datadog and needs their richer debugging or integrated monitoring surface, consolidating solely for error capture has little architectural value. For a small backend platform that needs programmatic error grouping alongside other backend modules and accepts building the alert loop, the broad REST option can reduce integration variety. Healthchecks remains the clearer companion for silent failure — the task that should have run but did not generate an exception at all.

Tracing deserves special care. Shared service names and timestamps can correlate an error with logs and metrics, and `trace_id` plus `span_id` can carry correlation fields. That does not create a distributed trace query or a span tree. If responders need to navigate parent-child spans across checkout, inventory, and payment services, use a tracing system that provides that model rather than presenting field equality as full tracing. Likewise, source-map reversal, Electron minidump symbolication, and session replay require a product built for those artifacts.

The detector must stay independent enough to report that the evidence system is unreachable or that a scheduled probe never ran. AWS's guidance on timeouts, retries, and backoff with jitter applies to the probe loop: set an explicit deadline, avoid synchronized retry storms, and cap retries. Capture the final classified outcome after the retry policy has done its work; otherwise one customer incident becomes several storage events per retry layer, with no extra diagnostic value.

## References

- https://prometheus.io/docs/practices/instrumentation/
- https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/
- https://docs.sentry.io/platforms/javascript/guides/node/sourcemaps/
- https://docs.datadoghq.com/synthetics/api_tests/http_tests/
- https://healthchecks.io/docs/
