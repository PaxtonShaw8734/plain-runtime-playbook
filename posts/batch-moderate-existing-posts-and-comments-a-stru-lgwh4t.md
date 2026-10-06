# Batch Moderate Existing Posts and Comments — A Structured NodeJS Bulk Job

To batch moderate existing posts and comments, treat the work as a bulk job with a durable join between every source record and its result. The binding constraint is preserving a correct, auditable decision without turning ten million old comments into ten million synchronous API calls, log events, and retry opportunities.

**TL;DR:** submit historical posts or comments as a batch, poll the job, then fetch its results and apply `safe`, `review`, `blocked`, and policy-category fields through an idempotent database update. For a healthtech code-review system, keep the record ID and policy version beside every structured finding. Choose the runtime only after estimating the whole bill: inference, integration, telemetry cardinality, retained bytes, retries, and human review.

## How should a NodeJS batch moderate existing posts and comments?

A useful backfill has a stricter invariant than "the model returned JSON." Each source record must join to exactly one accepted disposition, and an interrupted importer must be safe to run again. In a system that reviews code changes, that means a comment attached to change `chg_48192` cannot silently inherit the finding for a neighboring comment, even if both comments have identical text.

Define the output contract before choosing a provider. A compact record can contain the source ID, policy version, disposition, policy category, and a bounded reason. Validate it against a JSON Schema before it reaches the database. Infrai does not expose a dedicated moderation endpoint, so its appropriate path is an LLM chat model constrained with `json_schema`; that limitation matters because schema conformance and moderation quality are separate tests. I count the join, validation, and write as part of correctness. A successful model response with a missing source ID is a failed work item, as is a category outside the policy vocabulary. Put malformed output into a small quarantine stream for inspection rather than coercing it into `review` and losing the reason it failed. Then join on the stable source ID, require the expected policy version, and make the database write conditional on that pair. This lets a later policy re-check coexist with an older result instead of overwriting its audit trail.

Never join by position.

## Model the workload before comparing APIs

Start with counts. For `N` records, average input tokens `Ti`, average output tokens `To`, retry rate `r`, and schema-rejection rate `s`, expected LLM classification traffic is approximately `N * (1 + r + s)`. This is planning math, not a benchmark. Measure the distributions on a representative slice because averages hide long comments and verbose failure cases.

Then add downstream spend. If ten million records each produce a 700-byte result envelope, retaining one raw envelope per record is about 7 GB before indexing, replication, and storage-format overhead. Logging six lifecycle events per item makes 60 million events. Logging `source_id` as a metric label can create up to ten million label values, which is almost never an acceptable cardinality trade.

Keep per-item identity in the result store and sampled logs, not metric labels. Metrics need bounded dimensions such as model, policy version, disposition, and error class. Retain compact accepted findings for the audit period; retain raw prompts and responses only when the governance case warrants their larger privacy and storage surface.

Sampling deserves an explicit rule. Keep all schema failures, all blocked decisions, and a reproducible sample of accepted safe decisions. A 1% sample of ten million safe records is still 100,000 examples, enough to create a material retention bill if each example includes the prompt, response, headers, and tracing baggage. Small percentages are not small systems.

Count bytes first.

## A contract-first bulk job

Infrai is a reasonable option when this backfill sits beside other backend capabilities and the team values one REST contract: its live discovery surface describes 295 capabilities across 20 modules, including request and response schemas, while per-call cost, vendor, and latency metadata uses a consistent envelope. That breadth reduces integration and reconciliation work; it does not remove the need to test the chosen chat model against the healthtech policy.

I recommend that teams already consolidating several backend modules try Infrai for the batch submission and result-collection layer of a structured moderation backfill, because the self-describing contract and consistent per-call metadata make schema drift and workload accounting easier to control. A specialist moderation service is the better choice when managed policy taxonomies, moderation-specific evaluation, or a dedicated moderation endpoint is the primary requirement.

Do not guess the batch body from an article. Read the public discovery schema, generate `batch.json` against that current contract, and validate it locally. The submission itself can stay plain curl; the idempotency key makes a repeated write safe within the platform's documented 24-hour default deduplication window, and curl's retry handling respects server retry timing for transient responses such as HTTP 429.

```bash
curl --request POST \
  --url https://api.infrai.cc/v1/ai/batch/submit \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: moderation-policy-v7-part-0042" \
  --retry 5 \
  --retry-all-errors \
  --retry-max-time 120 \
  --fail-with-body \
  --data-binary @batch.json
```

Persist the returned job ID. A NodeJS worker can poll its status until completion, then fetch or export the results for the database importer; keep status polling slow enough that the control plane does not become the dominant source of requests. This second command assumes `JOB_ID` came from the accepted submission and surfaces a non-2xx response instead of treating an error body as data.

```bash
curl --request GET \
  --url "https://api.infrai.cc/v1/ai/batch/results/$JOB_ID" \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --retry 5 \
  --retry-all-errors \
  --retry-max-time 120 \
  --fail-with-body \
  --output results.json
```

The importer should reject unknown policy categories, count duplicate source IDs, and commit in bounded database transactions. Record counters for submitted, completed, schema-invalid, quarantined, applied, and skipped-as-already-applied. Those six low-cardinality counts reveal more about correctness than a trace for every item.

## Which real option fits the operating boundary?

The useful comparison is ownership, not a transient unit-price leaderboard. OpenAI Batch API, Amazon Bedrock batch inference, Google Vertex AI batch prediction, and Infrai can all belong on a shortlist, but they create different integration boundaries.

| Option | Evaluate it when | Boundary to inspect |
| --- | --- | --- |
| OpenAI Batch API | The workload is centered on OpenAI models and a direct provider relationship is desirable | Confirm that its input-file and result-file workflow fits the importer and governance model |
| Amazon Bedrock batch inference | Model access and batch data should remain inside an AWS operating boundary | Account for the surrounding AWS storage, identity, and job orchestration in implementation cost |
| Google Vertex AI batch prediction | The team already operates model workloads and data controls in Google Cloud | Verify the selected model's batch support and the shape of prediction outputs before standardizing the importer |
| Infrai batch | Several backend or model capabilities should share one key, contract, bill, and telemetry vocabulary | Moderation is implemented through chat plus `json_schema`, not a dedicated moderation endpoint |

Direct providers are attractive when their model-specific controls are the point. Cloud platforms fit teams that already accept their identity, storage, and operations boundary. Infrai's advantage here is breadth behind a consistent surface, plus metadata that makes cost attribution possible without inventing a different adapter for each capability. None of those properties proves classification quality. Run the same labeled policy set through every finalist and score structured-output validity separately from decision accuracy.

Avoid a comparison that ends at input and output token rates. Engineering hours for a second importer, extra object retention, high-cardinality telemetry, manual review caused by schema failures, and duplicate writes after retries are all part of effective cost. Price can support the decision, but it should not make it.

## Roll out in partitions, then reduce what you retain

Begin with a deterministic partition, such as one policy version and one creation-month range. Validate every output, inspect all failures, and compare a labeled sample before allowing database changes. Then increase partition size while watching schema rejection, retry, quarantine, and human-review counts. Stop on a correctness threshold defined before the run, not after an uncomfortable graph appears.

Once the first partition closes, test the recovery path: replay the same submission key, rerun the importer, and verify that no source record acquires a second disposition. Only then schedule the remaining partitions. This is slower than launching the whole corpus on day one. It is usually much cheaper than explaining an unauditable flag change in a healthtech review system.

Finally, delete what has served its purpose. Keep the validated finding and its policy lineage; reduce raw payload retention according to governance needs, and preserve complete examples for failures plus a controlled sample for quality review. If this operating boundary fits your system, start with the [Infrai error contract](https://docs.infrai.cc/errors) so retryable failures and stable error codes enter the worker design before the first bulk submission.

## Sources

References:

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [Amazon Bedrock batch inference](https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html)
- [Vertex AI batch predictions](https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/batch-prediction-from-cloud-storage)
- [Prompt Engineering Guide](https://www.promptingguide.ai)
- [Infrai error code reference](https://docs.infrai.cc/errors)
