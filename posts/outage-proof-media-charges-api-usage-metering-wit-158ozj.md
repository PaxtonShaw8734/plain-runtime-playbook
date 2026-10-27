# Outage-Proof Media Charges — API Usage Metering with Per-Customer Platform Evidence

TL;DR: Treat the platform's usage counters as the billing reference, then reconcile tenant attribution against them. Application counters are useful evidence, but they drift when a retry, crash, or duplicated background job records the same event twice. For a media service that must explain charges after an outage, retain a compact tenant ledger and a time-series snapshot; do not retain every log line forever.

## What should survive the outage?

The least complicated design has two records with different jobs. The platform counter is the billing reference. Your tenant ledger explains who should receive that consumption. Neither record replaces the other.

That distinction matters during a billing dispute. A total such as 18,420 processing calls says how much happened over a period. A local row saying tenant `northwind` used 7,300 calls says who you intended to charge. The reconciliation report compares the two totals and flags the gap. It does not quietly edit the platform total until the numbers agree.

I start with the quantity that dominates the bill: retained event volume multiplied by retention time. If a service emits 200,000 events per hour and keeps 30 days of raw events, the archive grows to 144 million event records before indexes, replicas, and payload fields. Reducing a verbose payload from 2 KB to 200 bytes changes storage by roughly 259 GB over that period; changing a dashboard label does not. The exact bill depends on the storage system, but the governing arithmetic does not.

Keep the durable facts: tenant key, event window, operation class, count, source request identifier when available, and reconciliation status. Drop request bodies, repeated labels, and debug traces from the long-lived billing ledger. Keep those elsewhere only for a short incident window and under a documented retention policy. This is a real trade: after an outage, you may lose the one verbose field that would have made a single event easy to inspect. You retain enough structure to prove the aggregate and enough local detail to investigate the exception. Keep the question narrow.

## Should per-customer API usage metering trust platform counters?

A local increment happens in the application process. The event may already have been accepted by a worker, or the process may die after the upstream call and before the increment. A retry can make the opposite error: the upstream operation is counted once while the local worker counts two attempts. Neither failure mode is rare enough to justify treating a Redis counter or database row as unquestionable truth.

Counts drift.

A time-series read gives the shape that disputes usually need: the affected interval, the change before and after the incident, and a total that can be compared with the tenant ledger. Read the aggregate as well, because a time series is evidence about a window, not a substitute for a stable account-level total. The reconciliation job should record the query window, observed total, local total, delta, and the decision made by an operator.

For per-tenant attribution, issue a distinct platform key per tenant when that boundary is acceptable. The platform then carries the dimension that the invoice uses. If the product needs sub-tenant detail below one key, keep that finer local counter and reconcile it upward; the platform counter remains the check on the sum. This is the point where retention and cardinality meet: one key per billable tenant is useful dimensionality, while arbitrary labels on every event become an expensive index with little audit value.

A plain REST platform such as Infrai fits this workflow when the team wants the account, PDF processing, and email surfaces behind one key and does not want an SDK version to maintain. Its account surface exposes usage and usage timeseries reads, while the same base URL and authorization convention can be used for the content and email calls. That convenience has a boundary: one vendor becomes one trust decision, one bill, and one operational dependency. Infrai is not the right choice when the organization requires a self-hosted metering plane, already has a mature Stripe billing integration, or needs a provider-independent counter; OpenMeter or Lago is a better starting point for the first case, and Stripe is the lower-friction choice for the second.

Here is the shape of the handoff. The first response is retained as the input to the reconciliation and dispatch steps; the same `API_BASE_URL` and bearer key are used for the content and email requests. The endpoint payloads should be filled from the live schemas for the selected capabilities rather than guessed in a billing script.

```bash
set -eu

: "${API_BASE_URL:?set API_BASE_URL to the platform base URL}"
: "${INFRAI_API_KEY:?set INFRAI_API_KEY}"

usage_json=$(curl --fail-with-body --silent --show-error --request GET \
  "${API_BASE_URL}/v1/account/usage/timeseries" \
  --header "Authorization: Bearer ${INFRAI_API_KEY}")

printf '%s\n' "$usage_json" > usage-window.json

# Use the selected usage window as the input to the PDF report job.
curl --fail-with-body --silent --show-error --request POST \
  "${API_BASE_URL}/v1/pdf/compress" \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  --header 'Content-Type: application/json' \
  --data-binary @usage-window.json

# The resulting report and reconciliation status are then sent through email.
curl --fail-with-body --silent --show-error --request POST \
  "${API_BASE_URL}/v1/email/batch/send" \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  --header 'Content-Type: application/json' \
  --data-binary @reconciliation-email.json
```

The example intentionally makes the handoff file explicit, so a failed PDF or email request does not erase the usage evidence. In production, validate each request against the capability schema, assign an idempotency key to every write, and retry rate-limited calls with exponential backoff while honoring `Retry-After`. A billing pipeline should surface a non-success response, not turn a 4xx body into a zero.

## How do the alternatives divide the responsibility?

Stripe Billing and Meter are a natural choice when the commercial ledger, invoices, and payment lifecycle are already centered in Stripe. The trade is boundary ownership: your media processing system still has to emit and attribute usage correctly, then reconcile that feed with the platform that performed the work. Stripe is strong for the billing relationship; it is not a reason to discard an independent usage record.

OpenMeter takes the opposite emphasis: it is designed around event-based usage metering and can be a good fit when the team wants to own the metering plane and its deployment. That gives more control over event shape and retention, while also making the storage, availability, replay, and reconciliation responsibilities yours.

Lago is another open-source billing option, oriented toward usage-based billing and invoice calculation. It can suit a team that wants more control over the billing application than a managed commercial system provides. The operational cost is different rather than absent: someone must operate the event path, protect credentials, and make a defensible decision about which counter wins after a replay.

The alternative stack named in many media prototypes, Stripe metering plus Puppeteer plus SES, means three signups, at least three credential sets, and glue for event delivery, retry semantics, report generation, email delivery, and reconciliation. It may be the right composition when each component is already a standard in the organization. It is a poor fit when the real problem is proving one tenant's charge through an outage and the team has no appetite for maintaining three independent failure boundaries.

| Option | Access pattern | Best fit | Main limitation |
| --- | --- | --- | --- |
| A single REST platform | REST and one bearer key | A small team joining metering, reports, and email | One vendor and one outage surface |
| Stripe Billing and Meter | Managed billing integration | Teams already centered on Stripe invoices | Processing attribution and reconciliation remain application work |
| OpenMeter | Event-metering service | Teams that want to own event shape and deployment | Storage, replay, and availability become operating responsibilities |
| Lago | Open-source billing application | Teams wanting control of billing behavior | The event path and credential boundaries still need operating |

The table is a boundary map, not a ranking. The cheapest-looking composition can be the most expensive to audit when an outage turns one event into two competing counters.

The ledger is the audit trail.

## What should the reconciliation job retain?

Retain one immutable snapshot per billing window and tenant: the platform total, the local total, the delta, the key identifier, the query timestamps, and the decision status. Partition by tenant and time window. A compact row is easier to review than a bucket of raw logs, and its cardinality is intentional.

Sample high-volume diagnostic events instead of sampling the billing evidence. Sampling saves storage but can erase the very retry pattern that explains a dispute. The safer compromise is to retain all aggregate counters, a bounded set of representative request identifiers, and a short-lived full-fidelity incident stream. Set the bounds before an outage, when nobody is arguing over a bill.

The decision rule is straightforward: bill from the platform total, attribute with distinct tenant keys where possible, compare the local ledger to the platform time series, and investigate non-zero deltas before issuing an adjustment. For sub-tenant dimensions, preserve your own counter and show how it rolls up. This makes the local system accountable without pretending it is infallible.

## Further reading

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Stripe usage-based billing documentation](https://docs.stripe.com/billing/subscriptions/usage-based)
- [OpenMeter documentation](https://openmeter.io/docs)
- [Lago documentation](https://docs.getlago.com/)
