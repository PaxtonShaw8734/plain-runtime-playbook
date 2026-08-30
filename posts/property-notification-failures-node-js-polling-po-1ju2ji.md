# Property Notification Failures: Node.js Polling, Postgres Counts, and Alert Thresholds

Short answer: count terminal delivery failures in Postgres, poll a narrow time window from a Node.js cron worker, and alert only when both an absolute threshold and a minimum attempt volume are met. Keep low-cardinality counters longer than detailed logs, and retain a bounded sample of failure evidence for diagnosis.

For a property-management notification service, the bill is usually easier to reason about after splitting telemetry into two classes: compact counts used for detection and detailed records used for investigation. Start with bytes retained, not with a list of tools. A threshold can be perfect and still be a poor system if every tenant, building, recipient, template, provider response, and request ID becomes a permanent label.

The least complex useful design is a scheduled query over delivery outcomes already written by the application. It is deliberately boring. That is an advantage when the job is to notice that rent reminders or maintenance updates are failing, not to build a second delivery database inside an observability stack.

## Retained evidence sets the operating limit

Storage is the dominant term to quantify before choosing a polling interval. Use a capacity model rather than a guessed monthly total:

`retained bytes = events per day x bytes per event x retention days x replication factor`

This is arithmetic, not a benchmark. Consider an illustrative service that emits 2,000,000 delivery events per day. If a detailed event averages 900 bytes and is kept for 30 days with a replication factor of two, the raw retained volume in the model is 108 GB. Indexes, compression, metadata, and backups will change the actual bill, so measure those terms in the system you operate. I'm not sure which of them will dominate in a particular deployment; a seven-day storage sample and the provider's invoice dimensions would resolve that uncertainty.

Now compare the detection record. Suppose the service aggregates outcomes into 5-minute buckets across 20 stable categories, and each aggregate row is modeled as 200 bytes. That is 5,760 rows and about 1.15 MB per day before replication. The exact byte counts are assumptions, but the order-of-magnitude distinction exposes the useful move: retain aggregates for trend and alert evaluation, while expiring most high-detail event data much sooner.

Cardinality matters just as much as event size. Good dimensions describe bounded operational cohorts: channel, outcome class, deployment region, and perhaps a coarse customer tier. Recipient address, message ID, free-form provider text, and property ID are poor metric labels because their possible value sets grow with traffic or customers. Those fields may belong in sampled evidence with access controls, but they should not define long-lived time series.

Keep less, on purpose.

The cost is real. When an unusual failure appears after the detailed retention window, an aggregate can prove that a cohort deteriorated but cannot reconstruct every affected message. The decision is therefore not “logs or metrics.” It is how much diagnostic evidence the team is willing to lose in exchange for a bounded storage curve.

## How can a Node.js cron worker query Postgres failure counts without noisy alerts?

Give each scheduled run a half-open evaluation window, such as `[window_start, window_end)`, and persist the last completed boundary. Half-open windows prevent the same timestamp from belonging to two adjacent runs. The worker should query terminal attempts, group only by approved low-cardinality dimensions, and calculate at least `attempt_count` and `failure_count`. A retry still in progress is not terminal and should not be counted as a final delivery failure.

Use two gates. The absolute gate catches a burst that matters operationally; the volume gate prevents a tiny denominator from looking catastrophic. A ratio may be useful too, but `1 failure / 1 attempt` should not page a team merely because it equals 100%. For example, a policy could require at least 25 terminal attempts and at least 10 failures in the evaluation window. Those numbers are policy examples, not universal defaults. Set them from the volume of each notification class and the response the team can actually perform.

The polling API contract should return data, not decide incident state. A generic request can remain small:

```bash
curl --fail-with-body --get 'https://telemetry.example.test/query' \
  --data-urlencode 'metric=notification_delivery_failures' \
  --data-urlencode 'window=5m' \
  --data-urlencode 'group_by=channel,outcome_class,region'
```

The scheduler owns the evaluation timestamp and the policy version. Postgres owns the durable delivery outcomes. The query surface owns aggregation. Keeping those responsibilities separate makes replay possible: given the same closed window and policy version, a run should reach the same result.

Don't page directly from a single failed poll. Record a missed evaluation, retry with bounded delay, and evaluate the original closed window when connectivity returns. A worker also needs overlap protection so that a slow run cannot race its successor. A database advisory lock or a lease row can provide that coordination; the important property is one active evaluator per policy, not the particular primitive.

Page once.

## Silence, recovery, and evidence belong to the incident state

`failure_count >= threshold` is only the transition into a candidate state. A usable alert also needs deduplication, recovery, and late-data behavior. Model at least healthy, firing, and resolved states, keyed by the small cohort used for the threshold. Persist the last evaluated window and an incident key so repeated worker runs update one incident instead of producing a new notification every five minutes. Late writes are awkward: pick a lateness allowance, close the window only after that delay, and document whether later corrections can amend historical dashboards without reopening an incident. There is no universally correct choice. A five-minute allowance produces slower detection than a zero-minute allowance, but it avoids evaluating records that are still arriving. Your mileage may vary because delivery providers and internal queues have different delay distributions; measure the observed arrival lag before choosing the allowance.

Severity deserves restraint. RFC 5424 defines severity levels for syslog messages, but it does not choose business impact for a property notification workflow. A failed marketing update and a failed urgent maintenance notice can share a transport error while requiring different responses. Keep technical outcome and business criticality as separate bounded fields, then let policy combine them. Do not convert every error log into a page.

Feature flags are useful at the policy boundary. Martin Fowler's treatment of feature toggles distinguishes categories with different lifetimes and decision dynamics. For this system, a release control can enable a new threshold policy for one stable cohort, while an operational control can suppress paging during a planned provider migration. Store the evaluated toggle state or policy version with the result so an investigator can explain why two otherwise similar windows behaved differently.

Counts answer “is this happening?” They rarely answer “why?” Keep a bounded evidence sample for each failure cohort: a redacted provider code, the notification class, attempt timing, and a correlation identifier that can be resolved through an access-controlled operational store while that store still retains the underlying record. Avoid copying recipient content into telemetry.

Sampling must protect rare failures. Uniformly taking one event in a hundred can erase a low-volume but high-impact outcome. A better rule is stratified: keep the first few examples for every bounded error class in each window, then sample additional repetitions. Put a hard byte or row ceiling on every stratum. The ceiling makes cost predictable; the “first few” rule retains evidence when a class first appears.

This loses detail by design. It is not suitable when regulation or contractual audit requirements demand a complete, immutable delivery history. In that case, keep the authoritative audit record in a purpose-built, access-controlled store and derive observability aggregates from it. Conversely, stick with short-lived sampled evidence when the goal is operational diagnosis and a complete payload archive would duplicate sensitive data without improving the alert.

Evidence has a ceiling.

## Rehearse the policy before it can page

Before deployment, replay closed windows that cover four conditions: ordinary volume, a high-volume failure burst, low-volume noise, and no data. “No data” must be explicit. It can mean that no notifications were due, that ingestion stopped, or that the query failed. Those cases demand different actions, so the query result should distinguish an empty valid window from an unavailable evaluation.

Test boundaries too: one count below the threshold, exactly at it, and one above it; one attempt below the volume floor and exactly at it; a record at the start boundary and one at the end boundary. These cases catch more policy mistakes than another dashboard. Run the same fixtures after query or schema changes, then deploy threshold changes behind a controlled toggle and compare candidate decisions with the active policy before allowing the candidate to page.

Review cost and signal quality together. Useful weekly measures include bytes ingested by telemetry class, retained bytes by age, active series count, sampled evidence rows, alerts per policy, deduplicated repeats, and alerts closed without action. A policy that pages constantly but never changes an operator's decision is noise. A policy that stays quiet while terminal failures accumulate is worse.

The practical stopping rule is clear: retain stable aggregates long enough to see seasonality relevant to the property workflow, retain detailed evidence only for the investigation period the team can justify, and delete the rest. When an incident falls outside that period, accept that the team will have cohort counts rather than a complete reconstruction. That limitation should be an explicit operational decision, reviewed when incident evidence shows it is too restrictive.

## References

- Martin Fowler, “Feature Toggles”: https://martinfowler.com/articles/feature-toggles.html
- RFC 5424, “The Syslog Protocol”: https://datatracker.ietf.org/doc/html/rfc5424

## Further reading

- https://martinfowler.com/articles/feature-toggles.html
- https://datatracker.ietf.org/doc/html/rfc5424
