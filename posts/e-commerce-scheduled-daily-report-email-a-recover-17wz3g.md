# E-commerce Scheduled Daily Report Email: A Recovery Ledger for Cleanup

The least complex dependable design is a host scheduler that creates one durable cleanup run, followed by a worker that performs the cleanup and requests the daily report email. Keep compact state for the run and each delivery outcome; do not keep every successful attempt as a full log document.

**TL;DR:** choose the boundary by delivery guarantees, not by the familiarity of cron syntax. A scheduler alone is sufficient only when a missed invocation can be detected and replayed from durable application state. Once the e-commerce cleanup must survive a process exit, retry an email independently, or distribute work, the scheduler should enqueue durable jobs. Retain identifiers, state transitions, attempt counts, and final outcomes. Sample routine success telemetry and preserve all failures for a bounded period.

The observability bill is driven by event volume multiplied by bytes per event, retention time, and the cost of indexed dimensions. Suppose a daily cleanup covers 100,000 expired carts. If the worker emits five 800-byte records per cart, the raw payload is about 400 MB per run before indexing, replication, or metadata. A 30-day window holds roughly 12 GB of raw events. One 2 KB run summary plus one 300-byte terminal record per failed item changes the dominant term from all processed carts to failures. Those are planning assumptions, not benchmark results; substitute measured encoded event sizes and failure counts before setting policy.

## Should a scheduled daily report email backend use cron or a queue?

The useful question is not whether cron or a queue is more reliable in the abstract. It is whether the system can prove that a particular cleanup window was claimed, completed, and reported, and whether it can safely resume after interruption.

A run record should have a stable key derived from the cleanup date or business window, plus states such as `created`, `cleanup_started`, `cleanup_completed`, `email_requested`, and `email_sent`. Enforce uniqueness on that run key. If the scheduler fires twice, the second invocation finds the existing run instead of deleting or emailing twice. The cleanup itself also needs an idempotent predicate: select records that are expired and not already finalized, then commit progress in bounded batches.

Consider the awkward sequence rather than the happy path. The worker marks the last expired-cart batch complete, creates the email intent, sends the request, and exits before saving the provider outcome. On restart, neither a cron timestamp nor a success counter can tell the worker whether another request is safe. The durable email key can: the worker retries with the same identity, records the observed result, and leaves an ambiguous outcome for reconciliation instead of silently declaring success. This does not manufacture exactly-once behavior at the external boundary. It limits the uncertainty to one named operation that an operator can inspect, while the cleanup remains complete and does not rerun.

Retries happen.

Cron defines when a command should be invoked. It does not, by itself, establish that the business operation completed. Traditional crontab scheduling also has operational details that matter: jobs run in the cron daemon's environment, day-of-month and day-of-week matching has defined semantics, and clock changes can cause some local-time jobs to run zero or two times. Use an explicit time zone policy, and make the durable run key authoritative rather than treating the wall clock as proof of execution.

A health check can create or reassert today's run through an authenticated internal endpoint. The endpoint must return success only after the run record is durably accepted, not after cleanup and email delivery finish.

```bash
curl --fail-with-body \
  --request POST \
  --header "Authorization: Bearer ${SCHEDULER_TOKEN}" \
  --header "Idempotency-Key: cart-cleanup-${RUN_DATE}" \
  --header "Content-Type: application/json" \
  --data "{\"run_date\":\"${RUN_DATE}\",\"job\":\"expired-cart-cleanup\"}" \
  https://jobs.example.test/internal/runs
```

This request is deliberately small. The web exchange accepts work and closes; a worker owns the long operation. Keep the token outside the crontab entry where possible, constrain endpoint access, and ensure the client timeout cannot be confused with cancellation of an already accepted run.

## Model the guarantee before choosing the mechanism

There are three different promises hiding behind “send the daily report.” At-most-once processing avoids duplicates but can lose work after a crash. At-least-once processing retries work but requires idempotency because a worker can complete an effect and fail before recording acknowledgement. Exactly-once business effects are not supplied by a scheduler label; they require an application-level invariant around the database mutation and email request.

For a single host, a scheduler plus a durable runs table may be enough. A reconciliation task queries for the expected date key and recreates missing or stalled work. This keeps the control plane small, but the team owns leasing, retry timing, and recovery queries.

A durable queue becomes justified when the cleanup and email need separate retry histories, when multiple workers claim partitions, or when backpressure matters. Consumer acknowledgements let a broker distinguish work that may be removed from work that must be redelivered. Acknowledgement still cannot prove that an external email provider accepted a message exactly once. Record an email intent with a stable message key in the same durable workflow, and make repeated dispatch observable.

Both choices have limitations. The scheduler-only shape is not suitable when one worker cannot absorb the backlog within the recovery objective or when independent email retries must not wait behind cleanup batches. A queue handoff is a poor fit when the workload is one small daily run and the team cannot operate or test the extra acknowledgement, redelivery, and dead-letter behavior. In that case, the additional mechanism creates more states without improving the required guarantee.

The mechanism changes. The invariant does not.

| Decision pressure | Scheduler plus durable run state | Scheduler handing off durable jobs |
|---|---|---|
| Missed start | Reconciler creates the absent run | Reconciler republishes the absent job |
| Worker crash | Lease expiry exposes unfinished batches | Unacknowledged work becomes eligible for redelivery |
| Duplicate start | Unique run key rejects it | Unique run or job key absorbs it |
| Email failure | Application retries from email state | Separate delivery job can retry independently |
| Operational burden | Fewer components, more application recovery logic | More components, explicit buffering and acknowledgement |

Do not claim exactly-once delivery merely because the table has a uniqueness constraint. The constraint can prevent duplicate run creation. The cleanup mutation, job acknowledgement, and external email acceptance cross transactional boundaries, so retries must be expected and their effects controlled.

## Retention should follow the state machine

Counting log lines is a poor capacity model. Count emitted records by transition, measure encoded bytes, and count the dimensions attached to them. A label such as `run_state` has a bounded set. A label containing `cart_id`, recipient address, or raw error text can create cardinality proportional to customers or events, while also increasing privacy exposure. Put high-cardinality identifiers in sampled logs or a queryable application record, not in metric labels.

Start with four telemetry classes: scheduler acceptance, cleanup progress, email outcome, and reconciliation. For each class, define the question it answers. The scheduler signal answers whether a date key was accepted. Progress counters answer whether processing is moving. The terminal email state answers whether the report was requested and sent. Reconciliation records answer what the automated recovery changed.

Then calculate retention from the investigation window. A useful planning equation is `daily stored bytes = events per run x average encoded bytes x runs per day`, evaluated separately for success, retry, and failure events. Multiply by retention days only after accounting for the storage system's indexing and replication behavior. Those multipliers are implementation-specific, so measure them rather than inserting a universal factor.

Consider a hypothetical policy: retain run summaries and terminal failures for 30 days, keep unsampled retry transitions for seven days, and sample one percent of routine per-item successes for two days. This is a policy example, not a universal recommendation. It is defensible only if reconciliation can recover from durable run state after the short-lived telemetry expires and if the audit requirement does not demand longer evidence.

Cardinality deserves its own budget. A metric with five job names, five states, and three worker pools has at most 75 combinations before other labels. Adding an unbounded customer identifier changes the shape of the system; the series count can now grow with the customer population. Reject that label in instrumentation review. Store the customer reference on the durable run or failure record, where retention and access controls can be explicit.

Keep less.

Sampling also has a hard edge. Sample successes after aggregating counts, never before recording the durable business transition. Keep all terminal failures and a bounded number of retry details. Otherwise, the cheapest telemetry policy can erase the evidence needed to distinguish “the cleanup never started” from “the cleanup finished but the report email failed.”

## Operate the recovery path, then trim the bill

Test duplicate scheduler invocations, a worker exit between cleanup commit and acknowledgement, an email timeout after acceptance, and a date boundary in the chosen time zone. In each test, assert both the business result and the retained evidence. A dashboard that turns green while two reports are sent is not a passing result.

Deployment needs the same discipline. Roll out state-machine changes so old and new workers understand overlapping states, pause destructive cleanup if reconciliation cannot establish ownership, and alert on age of the oldest incomplete run rather than raw error count alone. Error volume can fall to zero when the scheduler stops running. Age continues to expose the absence.

The final decision rule is compact: use the scheduler-only shape when one durable application record, one worker, and a reconciliation query satisfy the required recovery time. Add durable queued jobs when independent retries, buffering, or concurrent consumers are necessary. In both cases, the database state is the recovery ledger and telemetry is supporting evidence, not the sole record of truth.

What should be deliberately discarded? Routine per-cart success logs, recipient addresses in telemetry, unbounded identifiers in metric labels, and verbose retry payloads after their investigation window. Losing them means a later incident may support aggregate reconstruction rather than event-by-event replay. Keep longer-lived run summaries and terminal failure records to make that trade visible. **Retention is a recovery decision expressed as a storage policy.**

## Further reading

- https://man7.org/linux/man-pages/man5/crontab.5.html
- https://www.rabbitmq.com/docs/confirms
