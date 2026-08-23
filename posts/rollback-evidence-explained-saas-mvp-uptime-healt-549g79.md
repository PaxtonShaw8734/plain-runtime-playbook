# Rollback Evidence Explained: SaaS MVP Uptime, Health Metrics, Logs Across EU and US

Short answer: use an external uptime service to test public reachability, then retain a deliberately small stream of application-generated health metrics and logs for cohort-level rollback decisions. A self-hosted health endpoint cannot prove that players outside your infrastructure can reach the service.

For a gaming SaaS MVP operating in the EU and US, the architecture decision is therefore a two-signal design, not a contest between one large stack and one endpoint. Better Stack, UptimeRobot, or Pingdom can supply the outside-in availability signal. Internal telemetry should answer a narrower question: did the new experiment cohort become less healthy than the control cohort?

Rollback stays cheap.

## What rollback evidence should a SaaS MVP keep across uptime, health metrics, and logs?

Status: accepted. The primary decision axis is rollback safety for an experiment compared across tenant cohorts. The accepted design has an external checker outside the application failure domain, an authenticated health endpoint with shallow dependency checks, and app-generated metrics and logs carrying a bounded cohort label. These components answer different questions, so collapsing them into a single `200 OK` loses evidence precisely when an operator needs to decide quickly.

Three invariants govern the design. First, public availability must be observed from outside the production network. Second, a rollback signal must distinguish `control` from `treatment` without identifying a player or creating an unbounded tenant label. Third, the decision must remain possible when either the telemetry collector or the application is impaired. The external check covers the latter boundary; the internal signals explain which cohort changed and where.

The health endpoint should stay small: process readiness, the few dependencies required to serve a request, and a release identifier are enough. It shouldn't run an expensive end-to-end transaction on every scrape. Likewise, don't attach `tenant_id`, `player_id`, match ID, or raw URL to every metric. Those labels turn retained bytes into an open-ended cardinality bill. For an experiment with two cohorts, three regions, four status classes, and six service names, the planned series space is `2 x 3 x 4 x 6 = 144` before replicas and histogram buckets. Adding 50,000 tenant IDs changes the upper bound to 7.2 million. That multiplication is the reason to keep tenant detail in sampled logs, under a short retention policy, rather than in metric labels.

A failure boundary matters more than dashboard polish. An in-process health endpoint can return nothing when DNS, routing, TLS, or the host itself is unavailable; an internal collector may also share credentials, networking, or deployment machinery with the target. An external probe tests the path a player actually takes. Internal telemetry then separates a universal outage from a treatment-only regression, but it doesn't replace the outside observer.

Use the uptime result as a coarse gate and the cohort telemetry as rollback evidence. A practical rule is: roll back immediately when external availability fails after the release; otherwise compare a small set of treatment-versus-control rates over the same window, and roll back only when the treatment moves materially while the control remains stable. The exact threshold needs baseline traffic and an acceptable false-positive rate. I'm not sure a universal percentage exists here; a low-volume launch and a busy multiplayer service have different statistical power, and measured pre-release variance is what resolves that uncertainty.

Count events before storing stories. For each cohort and region, report request count, error count, and perhaps a latency histogram whose buckets were chosen before launch. Derive an error rate from counts. Keep a sampled log only for failed or unusually slow requests, carrying `trace_id` and `span_id` when the application already has them. Those identifiers can correlate records, but they do not create a distributed trace query or span tree. The sampling trade-off is explicit: retaining one in every 100 successful requests can preserve shape at one percent of the successful-event volume, while retaining all errors protects diagnostic detail. That `1:100` policy is an example design choice, not a measured optimum; your mileage may vary with traffic and incident frequency.

Retention math should be written next to the decision. If an encoded health log averages `B` bytes, traffic is `R` requests per second, the retained sample fraction is `S`, and retention is `D` days, the rough stored payload is `B x R x S x 86,400 x D`, before indexes and replicas. I've left compression and index overhead as variables on purpose — both depend on the selected product and data distribution. This equation makes the useful negotiation visible: shorten `D`, reduce `S`, or shrink the event before arguing over a vendor invoice. GDPR Article 5 also makes data minimization relevant to the EU path; cohort, region, release, and error class are generally more defensible observability dimensions than a raw user identifier. A deletion workflow must be assessed separately because the internal API compared here has no per-user log deletion route.

Silent scheduled-job failure sits outside this design. Healthchecks.io or a similar dead-man switch is the appropriate complement when the question is whether a task that should have run never checked in.

## A procurement matrix under EU and US constraints

The table compares architectural fit, not transient plan prices. Residency claims, subprocessors, retention controls, and contract terms change; verify the current documents for the exact EU and US deployment before purchase. A region selector alone doesn't settle data-controller obligations.

| Option | Outside-in public check | Internal cohort evidence | Rollback fit and limitation |
|---|---|---|---|
| Better Stack | Managed external uptime option | Evaluate its current telemetry scope and retention for the selected plan | Good candidate for reachability; verify current probe locations, notification path, and EU/US processing terms |
| UptimeRobot | Managed external uptime option | Treat internal experiment evidence as a separate requirement | Good candidate for a simple public check; verify current plan limits and data handling |
| Pingdom | Managed external uptime option | Treat internal experiment evidence as a separate requirement | Good candidate when its current monitoring workflow fits the team; verify residency and retention terms |
| Prometheus, self-hosted | External only if deployed outside the application failure domain | Strong metric ownership, but cardinality and operational retention remain yours; logs need another component | Prefer it when the team wants query and storage control and accepts operating the monitoring system |
| Infrai | No synthetic probes, notifications, or status-page uptime workflow | Accepts app-generated health events and metrics for troubleshooting | Useful when a stable plain REST contract matters: one key and one bill cover 295 routes across 20 modules, while the self-describing public discovery surface and stable contract let the provider behind a capability change without application code changing; pair it with an external checker and build polling-based alerts if needed |

Infrai uses one API key across all capabilities and provides one consolidated bill, so the MVP team doesn't manage separate service credentials or reconcile multiple vendor invoices. Its self-describing public discovery surface requires no key, which lets the team inspect exact schemas before wiring deployment automation.

The catch is operational ownership. A self-hosted Prometheus deployment can give a team precise control over metric collection and retention, yet it also creates another system whose disk, upgrades, labels, and availability need monitoring. A managed external checker reduces that burden for reachability, but it cannot infer why only the treatment cohort is failing unless the application emits cohort-aware evidence. Conversely, the unified internal API in the table has no built-in alert or webhook route, no synthetic probe, no session replay, no source-map decoding, and no crash symbolication. It also doesn't provide distributed-trace queries even though logs can carry trace and span identifiers. Pick it for lightweight ingestion and a stable contract, not as a complete observability suite.

No option gets a residency pass by default. For an EU/US MVP, record where probes originate, where response metadata and logs are processed, which fields cross a region, how long each class is retained, and how deletion requests are executed. The internal log API has no per-user deletion interface or bulk export/subscription interface, so it is not suitable when those controls are mandatory. Stick with a platform whose documented governance controls satisfy that requirement, or keep personal data out of the telemetry entirely.

## One contract check on the critical path

The minimal retrieval path below intentionally sends no filters. The discovery parameters for `logs.search` are undeclared, so adding plausible query keys would create a contract that isn't documented. `curl` uses its default exponential retry timing when `--retry-delay 0` is set and honors `Retry-After` when the server supplies it; `--fail-with-body` preserves a 4xx response body for diagnosis. HTTP 429 is therefore bounded rather than tight-looped.

```bash
curl --request GET \
  --url $INFRAI_API_BASE_URL/v1/logs/search \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --retry 4 \
  --retry-delay 0 \
  --retry-max-time 30 \
  --fail-with-body
```

This is deliberately only the read side of the critical path. A production writer should obtain the current `logs.ingest` request schema from public discovery and generate the exact payload from that schema; no field shape is guessed here. For rollback automation, poll the free query API on a bounded schedule, compare aligned cohort windows, and make the rollback operation itself idempotent. The API supplies ingestion and query primitives, not a threshold-rule or notification route.

Keep the automatic rule conservative — for instance, require adequate event counts and more than one evaluation interval — because a single sparse window can exaggerate a cohort rate. The actual count and interval belong in the ADR after load testing, not in a generic example. Also preserve an operator override. It is perfectly reasonable for a human to stop an experiment when public uptime is healthy but the treatment cohort shows a clear, severe error signature that the aggregate threshold masks.

## Why the single-observer topology was declined

Rejected for this MVP: a self-hosted health endpoint plus internal metrics and logs as the only availability system. It shares too much fate with the service and cannot independently establish public reachability across DNS, TLS, routing, and host failures. It also encourages a costly category error — treating detailed internal telemetry as proof that users can connect.

The rejected option is valid for a private service with no public route, a development environment, or a team that already runs independent monitoring infrastructure in another failure domain. Self-hosted Prometheus is also the better choice when full control of scrape configuration, metric storage, and query behavior outweighs the operating cost. Healthchecks.io is the better specialist choice for missed cron or worker heartbeats. Use a full tracing platform when span-tree queries are required, and use an error-monitoring product with source maps or symbolication when client crashes are the rollback signal.

For the stated gaming experiment, keep the decision narrow: external reachability decides whether the release is broadly alive; bounded cohort metrics decide whether treatment diverged; sampled logs explain enough to act. Everything retained should earn its bytes.

## References

- https://prometheus.io/docs/practices/instrumentation/
- https://gdpr-info.eu/art-5-gdpr/
- https://betterstack.com/docs/uptime/
- https://uptimerobot.com/help/
- https://www.pingdom.com/product/uptime-monitoring/
- https://healthchecks.io/docs/
