# Live Game Production API Keys: Rotate Without Downtime Within a Refusal Budget

Rotate a production API key without downtime by using a bounded grace-period overlap, deploying the new secret-store reference before revoking the old key, and ending the overlap from observed usage rather than elapsed time alone. For a live game backend, the deciding constraint is the allowed volume of refused player traffic: a shorter overlap limits exposure and telemetry spend, while a longer one gives slow workers and delayed deploy targets more time to converge.

TL;DR: Treat rotation as a two-phase state transition. During the overlap, the verifier accepts old and new key identifiers, callers move to the new secret, and telemetry records only the bounded signals needed to prove convergence. Revoke the old key when every required target is healthy on the new key and old-key use has remained at zero for the chosen observation interval. If that evidence is missing, roll back the caller deployment or extend the window within a preapproved ceiling. Don't guess.

## How should teams rotate a production API key without downtime?

This decision record assumes a gaming service with continuously connected players, background match workers, regional API processes, and scheduled jobs. The objective isn't a mathematically perfect instant change. It is a controlled transition in which the service preserves availability without leaving two credentials valid indefinitely.

Four invariants govern the change. First, at least one valid credential must be available to every healthy caller throughout the rollout. Second, the new key must be stored and distributed before any caller is configured to use it. Third, the old key can't be revoked until the deployment population has converged and the observation window has passed. Fourth, the overlap must have a hard deadline and an owner. These invariants turn a vague grace period into an operational contract.

The failure boundaries matter more than the happy path. A secret-store write can succeed while one region still serves a cached value. A process can pass a shallow health check while its authenticated dependency calls are being refused. A long-lived worker can avoid restart and continue using the old key. A dashboard can report no old-key traffic because the identifying field was dropped, sampled away, or queried over the wrong interval. Silence isn't proof unless the measurement path was tested.

The deadline is a control.

The refusal budget should be explicit. Consider an example policy with a ceiling of 20 authentication refusals across all production callers and a 30-minute maximum overlap. Those are illustrative local decision inputs, not universal recommendations. If refusals reach 10, pause the rollout and inspect the failing target; at 20, roll callers back to the last known working secret mapping while both keys are still accepted. The halfway trigger preserves room for diagnosis before the budget is exhausted.

## Decision and option comparison

The chosen design is dual-key acceptance with new-key-first deployment, authenticated readiness checks, and bounded identifier-level telemetry. It makes the critical transition reversible until final revocation. The service never depends on every target changing at the same instant.

Order carries the design.

| Option | Refused-traffic risk | Exposure boundary | Operational evidence | Telemetry cost shape | Decision |
|---|---:|---|---|---|---|
| Revoke old, then deploy new | High during propagation | One key valid | Refusals reveal lag after impact | Low, but late | Reject for live traffic |
| Accept old and new for a bounded window | Low when ordering holds | Two keys valid until deadline | Per-key counts show convergence | Bounded by window and labels | Adopt |
| Accept both without an expiry | Low initially | Two keys remain valid | Convergence may never trigger action | Retention grows without a stop rule | Reject |
| Route a small caller cohort to the new key first | Low and measurable | Two keys valid during staged rollout | Cohort health exposes incompatibility | Extra cohort dimension | Use when targets can be partitioned |

The staged variant is compatible with the adopted design. A game backend with independent regions can move one low-traffic region first, provided the cohort label is bounded to a fixed regional set. Don't label by player, session, match, raw credential, or request ID. Those dimensions multiply time-series cardinality and can disclose sensitive material. A useful rotation counter needs few dimensions: environment, caller target, key generation, and outcome. Even those should be reviewed against the actual number of targets.

Retention math keeps the observability plan honest. If there are 12 deploy targets, 2 key generations, and 3 outcomes, the maximum series count for that counter is 12 x 2 x 3 = 72 before infrastructure-added labels. Sampling request logs at 10% would reduce stored events, but it weakens the claim that old-key use reached zero. For the revocation gate, use unsampled aggregate counters and retain them only long enough to cover the rollout, observation interval, and incident-review requirement. Rich request logs can be sampled because they support diagnosis rather than the zero-use decision.

Keep less, on purpose.

## Critical path: prepare, move, observe, revoke

The secret manager should hold distinct old and new values under versioned references. Callers should receive the selected reference through the established deployment mechanism; the key itself must not appear in source control, command history, logs, metrics, or deployment annotations. The OWASP Secrets Management Cheat Sheet describes secrets as lifecycle-managed assets and covers central storage, access control, auditing, rotation, and revocation.

Before production, exercise the same ordering in a non-production environment. Verify that both key generations authenticate during overlap, that an invalid key is refused, and that the readiness probe makes a real authenticated call. Also test the measurement failure case: send a known old-key request and confirm that the old-generation counter advances. This distinguishes zero traffic from broken telemetry. The following curl sequence is an illustrative control-plane contract. Its endpoint variables and payloads must be mapped to documented interfaces in the actual system, and each control-plane response must be checked before proceeding to the next phase. Secrets remain in environment variables rather than being embedded in the commands. The example shows four distinct acts because accepting a generation, deploying its reference, probing it, and revoking its predecessor have different failure boundaries. Collapsing them into one opaque operation would remove the pause point that protects the refusal budget.

```bash
curl --fail-with-body --silent --show-error \
  --request POST \
  --header "Authorization: Bearer ${ROTATION_ADMIN_TOKEN}" \
  --header "Content-Type: application/json" \
  --data '{"generation":"next","state":"accepted"}' \
  "${CONTROL_BASE_URL}/credential-generations"

curl --fail-with-body --silent --show-error \
  --request POST \
  --header "Authorization: Bearer ${DEPLOY_TOKEN}" \
  --header "Content-Type: application/json" \
  --data '{"credentialReference":"production/game-api/next"}' \
  "${DEPLOY_BASE_URL}/releases"

curl --fail-with-body --silent --show-error \
  --header "Authorization: Bearer ${NEW_API_KEY}" \
  "${API_BASE_URL}/ready/authenticated"

curl --fail-with-body --silent --show-error \
  --request DELETE \
  --header "Authorization: Bearer ${ROTATION_ADMIN_TOKEN}" \
  "${CONTROL_BASE_URL}/credential-generations/old"
```

The final command is deliberately separated from preparation and deployment. An operator executes it only after the gate is satisfied: all 12 example targets report the expected secret version, authenticated readiness succeeds, aggregate refusals remain below the budget, and old-generation use is zero for the full observation interval. Automation may evaluate the same assertions, but it should emit the evidence and stop on ambiguity.

Revoke last.

Rollout order is also a cost control. Start with a target that can represent the authentication path without producing a large burst of diagnostic logs. Expand while watching three signals: authentication outcome counts, active caller versions, and old-generation use. Page on refused traffic according to the budget, but avoid generating a log line for every successful request. A counter answers the convergence question with far fewer stored bytes.

## Failure handling and the revocation gate

A rotation is incomplete until the old credential is revoked and the result is verified. If a target continues using the old generation, determine whether it is an undeployed process, a stale cache, a long-running job, or a misbound secret reference. Keep both keys accepted only while the bounded investigation continues. Restarting everything indiscriminately may hide which target violated the deployment contract and can create unrelated player disruption.

Unknown means stop.

If the new key is refused, halt expansion. Because the old key is still valid, callers can return to the prior reference without waiting for a new credential. That reversibility is the primary value of the overlap. If the old key may have been exposed, availability and containment become competing objectives; the incident owner can shorten the window, accept controlled refusals, or isolate affected callers according to the predeclared refusal budget. A routine rotation policy shouldn't overrule an active security response.

Use a small evidence record for audit: rotation identifier, old and new key identifiers or fingerprints, approval, target set, phase timestamps, observed refusal count, old-key zero-use interval, and revocation result. Don't store either secret. Limit retention to the organization's audit and incident requirements rather than keeping high-volume request data as a substitute for a concise decision record.

The gate is binary even though its inputs are measured: every required target is on the intended generation; authenticated health checks pass; old-key use is zero for the selected interval; refused traffic is below the ceiling; the maximum overlap hasn't expired; and a named operator or automation identity owns revocation. When any condition is unknown, the gate isn't met.

## Rejected option and its valid use case

The rejected design is a single cutover that revokes the old key before deploying the new one. It has a smaller dual-validity interval, but propagation delay becomes refused traffic. For a live gaming API, that exchanges a controllable exposure window for an availability event whose size depends on rollout speed and hidden stale consumers. The spend ceiling doesn't rescue this design: fewer telemetry bytes are irrelevant if players can't authenticate or game workers can't reach their dependency.

There is a valid use case. A single-process internal job that can be stopped, updated, tested, and restarted with no concurrent requests may prefer a single cutover. The same can apply when an exposed key demands immediate containment and the refusal budget explicitly permits interruption. In both cases, the operator can enumerate the callers and prove that no traffic exists during the transition.

For continuously serving systems, the bounded overlap remains the defensible choice. Its limitations are real: two credentials remain valid temporarily, the verifier and secret store must support distinct generations, and the revocation decision depends on trustworthy unsampled counters. Its cost is finite only when the time window, target labels, counter dimensions, and retention period are fixed before deployment. The design supplies a reversible step before the irreversible one.

## References

- OWASP, "Secrets Management Cheat Sheet": https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
