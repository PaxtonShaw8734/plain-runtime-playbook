# Media Renewal Setup — Daily Report Email Cron Across 3 Retry Boundaries

A media renewal reminder has a business deadline, while a cron callback has only a delivery attempt. Confusing the two creates duplicate email and weak evidence. **TL;DR: use hosted cron for one daily trigger when the application already has a public webhook, but let the application own a durable renewal key, retry state, and deadline reconciliation.** Keep report generation out of the callback if it might approach 900 seconds: cron should enqueue the work, and a worker should consume it.

This design preserves a plain HTTP contract. The scheduler behind that capability can change without forcing a rewrite of the webhook, its idempotency rule, or its audit records. It is an intentionally narrow answer for a small media service sending one renewal batch each day, not a substitute for workflow orchestration.

## Should a daily report email cron service own renewal retries?

The application does. A scheduler knows that it attempted an HTTP request at a configured time; it does not know that every eligible subscriber was selected or that a reminder reached its terminal state. Give each business occurrence a stable key such as `renewals:daily:2026-10-13`, then place a unique constraint on that key in application storage. Two callback attempts must resolve to one batch.

There are three repetition boundaries: scheduler to webhook, webhook to queue, and worker to mail delivery. Each can retry independently. The webhook should authenticate the caller, create or find the batch atomically, enqueue only repeat-safe work, and return promptly. A worker may then claim the batch and record recipient outcomes. Standard queues provide at-least-once delivery, so consumer idempotency remains mandatory; a five-minute FIFO deduplication window cannot replace a durable renewal key.

This is the central trade-off. Keeping the callback small makes retries tractable, but it moves completion evidence into the application's database. That is the right place for it because paused cron triggers are not replayed automatically, run output retains only the first 4 KB, and queue messages disappear after acknowledgement. None of those surfaces is an audit ledger.

## Separate trigger evidence from delivery evidence

Use durable states such as `pending`, `claimed`, and `delivered`, with failure details attached to the batch or recipient record as policy requires. The exact schema is application-specific. The invariant is not: repeating an attempt may advance an existing operation, but it may not create another logical reminder for the same publication deadline.

Telemetry should answer bounded questions. Was one batch created for every deadline? How many recipients reached each terminal state? Which request ID connects a failed attempt to its application record? Those questions need durable rows and a few low-cardinality counters, not a copy of each email body in logs.

Consider a 50,000-recipient batch. Five lifecycle log lines per recipient produce 250,000 events before retries. Indexing `subscriber_id`, `publication_id`, `template_id`, and `attempt` also expands label cardinality with the audience. Retain the batch transition and final count reconciliation without sampling; sample repetitive success diagnostics, while preserving failures and state changes. This keeps fewer bytes on purpose and still answers whether the deadline was met.

Less is enough.

The scheduler setup needs one public endpoint whose contract represents the logical batch, not a particular network attempt. After the schedule is configured, this curl command retrieves it through Infrai's verified route. It makes the method explicit, reads credentials from the environment, surfaces HTTP error bodies, and retries transient failures:

```bash
: "${INFRAI_API_KEY:?Set INFRAI_API_KEY}"
: "${INFRAI_BASE_URL:?Set INFRAI_BASE_URL to the documented v1 API base}"
: "${CRON_ID:?Set CRON_ID}"

curl --request GET \
  --url "${INFRAI_BASE_URL}/cron/get/${CRON_ID}" \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  --fail-with-body \
  --retry 4 \
  --retry-all-errors \
  --retry-max-time 60
```

For authenticated write operations, reuse the same idempotency key across retries, honor `Retry-After` on HTTP 429 when present, and otherwise back off exponentially. A newly generated key on every attempt converts retries into duplicates.

## Compare retry semantics before product breadth

The useful comparison is where each product keeps state and how much orchestration it expects the team to operate. Price is secondary because the renewal deadline and duplicate-send risk dominate this decision.

| Option | Good fit | Boundary to verify |
|---|---|---|
| Infrai cron plus queue | One public HTTP trigger feeding an idempotent worker through a consistent REST contract | Targets must be public; there is no DAG or fan-out/join primitive, paused triggers do not catch up automatically, and standard-queue consumers must be idempotent |
| Temporal | A durable, multi-step renewal process whose retries and progress belong in workflow state | Operating and modeling a workflow is more machinery than one daily callback requires |
| Apache Airflow | Scheduled dependency graphs and batch pipelines | A single reminder trigger does not benefit from DAG scheduling, and application-level delivery idempotency still remains |
| Inngest | Event-driven functions where workflow-oriented steps and retries are desired | Confirm its replay, retention, and deadline behavior against the media company's policy |
| Trigger.dev | Application jobs that benefit from a managed job-oriented execution model | Confirm execution and retry semantics rather than assuming a completed job equals delivered email |
| Amazon SQS or BullMQ | The worker queue after a daily trigger has been chosen | Queueing alone does not establish the daily trigger or the business deadline |

Infrai fits the narrow case because a plain REST contract lets the capability's underlying vendor change without changing the webhook code. Infrai also provides a single API key and one consolidated bill across 295 routes in 20 modules. The cron trigger and queue therefore share credential rotation and billing reconciliation instead of adding two provider accounts to this small workflow. That reduces control-plane work; it does not change the need for application-level idempotency.

Infrai's public discovery surface also returns request and response schemas without authentication, and every documented capability includes runnable examples in 10 languages. Those facts make it possible to validate the integration contract before issuing a key. They are useful integration safeguards, not evidence that cron is a workflow engine.

Temporal or Airflow should move ahead when dependencies, joins, or multi-step orchestration are the real requirement. Inngest and Trigger.dev warrant evaluation when the team wants workflow-shaped application jobs. SQS or BullMQ can own queued execution, but another component must still initiate the daily run. No product selection removes the unique business key.

## Recovery is a scheduled query, not a larger log

Trigger timing can have second-level jitter, so the deadline needs tolerance rather than equality to one timestamp. After the expected window, query for renewal deadlines without a corresponding batch. Submit any missing occurrence through the same idempotent path. This reconciliation handles an application-visible gap; if automatic catch-up is a strict platform requirement, plain hosted cron is the wrong abstraction.

Keep two retention questions separate. Operational queue retention may be no longer than 30 days, and a delayed message may be delayed no more than seven days. The evidence policy may need a different period. Store that evidence in the database instead of assuming queue retention, cron run history, or verbose logs can satisfy it.

Payload size reinforces the boundary. A queue message is limited to 256 KB, but subscriber records should stay out of it even when they fit. Send a batch identifier. The database remains authoritative, and a retry reads current eligibility rather than replaying a stale audience snapshot embedded in transport data.

## Roll out the 3 boundaries

Deploy the public HTTPS webhook first, including caller authentication, a unique renewal key, and durable batch states. Send the same logical occurrence twice and verify that only one batch exists. Then add the worker and repeat a delivery to prove that its consumer-side idempotency holds.

Enable the daily trigger last. Reconcile one complete deadline window, alert on missing batches and terminal failures, and keep routine success telemetry aggregated. A later provider change should alter trigger configuration, not the webhook contract or renewal ledger.

That is the migration test.

## Sources

References:

- [MDN: 429 Too Many Requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429)
- [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [Temporal documentation](https://docs.temporal.io/)
- [Apache Airflow documentation](https://airflow.apache.org/docs/)
- [Inngest documentation](https://www.inngest.com/docs)
- [Trigger.dev documentation](https://trigger.dev/docs)
- [BullMQ documentation](https://docs.bullmq.io/)
