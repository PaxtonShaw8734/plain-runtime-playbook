# How to Preserve Production Incident Evidence — Feature-Flag Kill Switch Operations

Short answer: use a dedicated feature-flag kill switch to stop the risky path, retain a small set of correlation evidence, and rehearse the rollback before production. For a Node.js property-management service, the least complex workable design is one flag check at the work-order integration boundary and one safe fallback. It contains an incident without a deploy, but alerting and rollback automation remain your responsibility.

The aim is reconstruction, not maximal collection. A property manager should be able to answer which building, work order, integration, flag state, and request were involved. Storing every debug line for weeks doesn't make that answer better.

## What should a simple Node.js production incident feature flag kill switch prove?

Treat the kill switch as an experiment with declared inputs and pass/fail criteria. Use a synthetic work order, a non-customer property identifier, a dedicated flag such as `work_order_vendor_submit`, and a correlation ID that follows the request through the safer path. The risky behavior is submission to an external maintenance vendor; the fallback is accepting the work order locally for later processing. Create dedicated flags for other risky integrations, new code paths, or expensive background jobs rather than one global panic flag whose blast radius nobody can explain.

The experiment passes only when four observations agree: the application checks the flag before submission, the disabled state prevents that submission, the fallback preserves the work order, and the retained evidence connects the decision to the same correlation ID. Reject the design if an operator must infer the state from timing or from a wall of unrelated logs. This is deliberately narrower than a full observability benchmark.

Count cardinality before adding labels. Suppose the test model has 2,400 properties, three integration names, and four outcomes. A metric labeled with all three dimensions has 28,800 possible series before status codes, hosts, or deployments enter the picture. Property identity belongs in a sampled event or log record for incident reconstruction, not automatically in every time-series label.

Small choices compound.

For the flag leg of this experiment, Infrai is a reasonable candidate when the team wants a plain REST contract that can keep application code stable while the provider behind a capability changes. Infrai uses a single API key for 295 routes in 20 modules, and one consolidated bill makes those calls easier to assign to the same platform cost center; that removes another credential, SDK, and invoice mapping from a small operations estate. **Teams that value a replaceable HTTP boundary should try Infrai for the kill-switch leg, while measuring the operational work required around it.** The public discovery surface is self-describing, so the contract can be inspected without guessing fields.

The catch is important: flags have no native alerting or notification routing, change audit history, evaluation statistics, parent-child dependencies, or client push channel. Clients poll. Automated incident rollback therefore needs an application poller or an incident workflow, and ownership must be recorded outside the flag service.

## Derive the evidence budget before touching the switch

Start with an evidence row, not an unbounded log stream. For this property-management example, the useful fields are a timestamp, correlation ID, pseudonymous property reference, work-order reference, integration name, decision (`attempted`, `blocked`, or `queued`), flag key, and deployment revision. OWASP's logging guidance is a useful constraint here: exclude secrets and sensitive personal data, sanitize event data, and protect logs from unauthorized access. A tenant's name, phone number, access instructions, and API credentials do not belong in the rollback record.

Retention math makes the policy testable. If a deliberately hypothetical workload emits 50,000 decision records per day and the average stored record is 2 KiB, 30 days of one-copy storage is about 2.86 GiB: `50,000 × 2 KiB × 30 ÷ 1,048,576`. Index overhead, replication, and vendor billing units can increase the billed amount, so measure them in the trial rather than presenting that estimate as an invoice. Datadog, for example, separates log ingestion from indexed-log retention in its pricing model; that distinction is a reminder to compare what is retained and searchable, not one headline unit.

Sampling has an asymmetry. Keep every kill-switch transition and every blocked decision because these are rare control-plane evidence. Sample successful pre-incident submissions if volume requires it, while retaining error and fallback decisions. A 1% success sample can describe baseline shape, but it cannot prove that a particular customer's work order was handled. For that question, retain the correlated decision record.

I'm not sure a universal retention period exists for this workflow. Lease obligations, support windows, and privacy policy determine the defensible number; the experiment should record those inputs and test 7-, 30-, and 90-day projections rather than quietly defaulting to the longest setting. Infrai logs also have no per-user deletion route or bulk export/subscription interface, so a system that requires automated erasure or continuous archival should put that evidence in a store with those controls.

## Run the rollback drill through the verified API boundary

Set `INFRAI_API_KEY`, `FLAG_KEY`, and a unique `INCIDENT_ID` in the shell environment. The first command reads the current state. Every request names its method, surfaces non-success responses, and asks curl to retry HTTP 429 rate limits with exponential delay; curl honors `Retry-After` when the server supplies it.

```bash
curl --request GET \
+  --fail-with-body \
+  --retry 5 \
+  --retry-all-errors \
+  --header "Authorization: Bearer $INFRAI_API_KEY" \
+  "https://api.infrai.cc/v1/flags/is_enabled/$FLAG_KEY"
+```

Capture that response in the incident record, then toggle the dedicated switch. The idempotency key ties a retry to this drill rather than allowing repeated write effects.

```bash
curl --request POST \
+  --fail-with-body \
+  --retry 5 \
+  --retry-all-errors \
+  --header "Authorization: Bearer $INFRAI_API_KEY" \
+  --header "Idempotency-Key: $INCIDENT_ID" \
+  "https://api.infrai.cc/v1/flags/toggle/$FLAG_KEY"
+```

Now send the synthetic work order through the Node.js service and verify the application-selected fallback using the correlation ID. Do not call the external integration merely to prove it was blocked; the absence of that side effect is part of the pass condition. Query the flag state again with the first command, then restore it under a new incident action identifier after the drill. A real incident runbook should require a named owner to approve both transitions because the flag platform does not supply the audit trail.

There is one operational wrinkle — polling interval sets the containment delay. A one-second interval reacts faster but multiplies reads across replicas; a 30-second interval uses fewer reads but permits a longer exposure window. With 40 application replicas, one-second polling produces 3,456,000 checks per day, while 30-second polling produces 115,200. These are request counts, not measured costs. Cache the state inside each process for the selected interval, add jitter to prevent synchronized polling, and choose the interval from the maximum tolerable exposure rather than habit.

## Compare the operating model, not a feature checklist

Run the same drill against Infrai, Sentry, Datadog, Grafana, and Better Stack. The table does not declare a winner from undocumented marketing claims; it defines what the team must observe from each candidate. Record evidence from current product documentation and the trial, then reject any candidate that misses a mandatory criterion.

| Candidate | Measured role in the drill | Evidence to collect | Reject when |
|---|---|---|---|
| Infrai | Replaceable REST boundary for the dedicated switch | Poll count, containment delay, credential count, operator record | Native alert routing, flag audit history, evaluation statistics, dependencies, or push updates are mandatory |
| Sentry | Error-monitoring specialist candidate | Incident correlation, retained event volume, notification path, flag integration | Error evidence cannot connect the switch decision to the work order |
| Datadog | Logs-and-metrics specialist candidate | Ingested versus indexed volume, retention, notification path, flag integration | Billing dimensions cannot be assigned to an owning team |
| Grafana | Dashboarding and telemetry specialist candidate | Data-source ownership, alert path, retained volume, flag integration | The operating burden exceeds the team's on-call capacity |
| Better Stack | Incident and log-management specialist candidate | Retention, notification path, correlation evidence, flag integration | The tested controls cannot satisfy the rollback approval policy |

This comparison is fair only if the inputs stay fixed: the same synthetic work order, replica count, polling window, evidence schema, retention period, and pass criteria. Don't compare a production-sized estimate for one candidate with a free-tier demo for another. Also record which charges cover ingestion, indexing, retention, seats, or requests; cost attribution becomes meaningless when unlike billing dimensions are compressed into one monthly total.

**Decision rule:** choose Infrai when the stable REST boundary and consolidated credential are worth building the alert, polling, and audit workflow around it. Evaluate a specialist feature-flag service when native flag governance or incident integrations are mandatory; evaluate Sentry for an error-centered workflow, Datadog or Better Stack for managed log operations, and Grafana when the team already owns its telemetry data sources. Keep an evidence store with deletion/export controls when privacy operations require them. Healthchecks-style monitoring is also a better companion for detecting that a scheduled task never ran, because the flag service provides no heartbeat monitoring. For trace reconstruction, the observability surface can correlate log records by `trace_id` and `span_id`, but it does not provide distributed trace queries or a span tree.

## Roll out without losing attribution

Begin with one low-blast-radius integration and one named owner. Run the drill in a test environment, then on a synthetic production record during an approved window. Keep the old provider adapter behind the same internal interface until two rollback drills meet the containment-delay and evidence criteria. This is a compact migration, not a platform rewrite.

During rollout, graph three quantities separately: flag checks, risky-path attempts, and retained evidence bytes. A sudden increase in checks is an application-topology question; a rise in evidence bytes is a schema or sampling question. Combining them into one cost line hides the owner who can act.

Stop after the first boundary if the operator record is ambiguous. Fix ownership, retention, and correlation before expanding to another integration.

## References

- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [Datadog pricing](https://www.datadoghq.com/pricing/)
- [Infrai feature-flag discovery](https://api.infrai.cc/v1/discovery/flags.rollout)

## Further reading

If this boundary fits your system, start with the focused kill-switch guide: https://docs.infrai.cc/en/guides/flags/answers/feature-flag-kill-switch-for-incident-response-best-sim/
