# Structured LLM Summary JSON Schema in Node.js — Telemetry for Code Review APIs

A structured LLM summary JSON schema gives a Node.js code-review API a stable object to render, but the observability bill is usually determined by everything retained around that object: the patch, prompt, raw completion, retry bodies, and high-cardinality labels.

**Short answer:** request `overview`, `bullets`, `risks`, and `action_items` as JSON; validate them on the server; and retain the compact result while sampling bulky inputs and outputs. Count tokens before sending a long change. If validation fails, retry one shorter, coherent chunk. This makes quality versus latency an explicit operating decision instead of an accidental consequence of log volume.

## What is the telemetry bill actually paying for?

Start with bytes, not vendors. A useful planning equation for one event class is:

`stored bytes = reviews x events per review x average event bytes x retained days x replication factor`

Consider a hypothetical service handling 40,000 reviews per day. Assume its validated finding is 2 KB, while the patch, prompt, and raw response total 180 KB. Retaining one finding for 30 days yields about 2.4 GB before indexing and replication. Retaining three 180 KB payload events per review for the same period yields about 648 GB on the same raw-byte basis. The 270-fold difference identifies the dominant term. Trimming a few attributes will not compensate for retaining every body.

These are capacity-planning inputs, not benchmark results. Replace them with a seven-day byte histogram from the real pipeline. The equation remains useful because it tells an operator which measurement matters first.

Cardinality is the second multiplier. `repository`, `model`, `result`, and a bounded rule identifier can be useful dimensions. `commit_sha`, `pull_request_url`, `request_id`, and full filenames usually make poor metric labels because their distinct-value count rises with traffic. Keep those identifiers in a sampled trace or indexed event when an investigation needs them. Do not create a time series for every commit.

I would keep counters for schema-valid responses, retries, missing fields, and reviewed diff-size buckets, plus latency histograms split by a small controlled model label. That set can reveal whether stricter output or smaller chunks change quality and tail latency. It does not require permanent copies of source patches.

## How should Node.js validate a structured LLM summary JSON schema?

For a developer tool, `overview`, `bullets`, `risks`, and `action_items` form a practical boundary. A frontend can render a review card, an email digest can reuse it, and a webhook can route actions without scraping prose. The contract also creates countable failure modes: missing field, wrong type, excessive item count, malformed JSON.

A strict schema-shaped prompt is the portable baseline. This minimal request uses an OpenAI-compatible chat surface and leaves routing to `auto`:

```bash
curl --request POST \
  --url "$INFRAI_BASE_URL/chat/completions" \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --header "Content-Type: application/json" \
  --retry 4 \
  --retry-all-errors \
  --retry-delay 1 \
  --fail-with-body \
  --data-binary @- <<'JSON'
{
  "model": "auto",
  "messages": [
    {
      "role": "system",
      "content": "Review the code change. Return only valid JSON with exactly these fields: overview (string), bullets (array of strings), risks (array of strings), action_items (array of strings). Do not add markdown or other fields."
    },
    {
      "role": "user",
      "content": "Change: the cache key now includes repository_id. Review for correctness and operational risk."
    }
  ]
}
JSON
```

The server still has work after a successful transport response. Parse the assistant content, validate every required field and type, cap array lengths for the renderer, and reject additional fields when the contract requires an exact shape. A 2xx response is not proof of schema validity, and a valid schema is not proof that a finding is correct.

Before submitting a long diff, call `/v1/ai/tokens/count` with the schema instructions and source text. If required fields are absent, retry once with a shorter coherent chunk and record the validation reason. Repeating the identical oversized request consumes latency without changing the input condition. Smaller chunks trade context for a bounded fallback; a held-out evaluation set must determine whether that trade loses important cross-file findings.

No JSON contract can replace that evaluation set. It should contain accepted findings, false positives, missed risks, and changes for which an empty finding list is correct.

## Which API surface deserves the workload?

OpenAI, Anthropic, Google Gemini, and Infrai are credible candidates, but they optimize different boundaries. Test them with the same diffs, contract, and acceptance criteria because documentation alone cannot settle review quality or latency.

| Option | Reason to shortlist it | Boundary to test |
|---|---|---|
| OpenAI | Its platform documents Structured Outputs on a direct model API. | Verify the selected model and schema mode against nested fields and refusal cases. |
| Anthropic | Its documented tool-use interface provides a structured integration path. | Measure missing, extra, and malformed tool fields in the chosen model. |
| Google Gemini | Its API documents structured output with JSON Schema. | Confirm support for the exact model and REST version selected for production. |
| Infrai | One key covers its plain REST API under one bill, so any runtime can call this capability over HTTP without installing another SDK. | Confirm readiness and model behavior for the required region and evaluation set. |

This option is not suitable when a team needs the newest provider-specific schema feature immediately; choose that provider directly instead. It is also a poor fit when region or data-control requirements outweigh interface consistency. These limitations matter more than catalog breadth. Provider catalogs and schema features change, so the shortlist should be rechecked at implementation time. The correct choice is the surface that passes the code-review evaluation within the latency budget, not the one with the longest feature list.

## How much evidence should survive?

Use retention tiers because a single duration confuses three jobs. Aggregate counters and histograms support capacity and regression analysis. Validated findings support product rendering and whatever audit period the application requires. Raw patches and completions help with rare debugging work, but they are large and can contain sensitive source code.

A defensible starting policy retains all compact validation metadata, retains final structured findings according to the product's audit requirement, and samples raw bodies. For illustration, a 1% sample of the hypothetical 180 KB bodies changes the earlier 30-day raw estimate from roughly 648 GB to 6.48 GB before storage overhead. Raise sampling around schema failures or a controlled evaluation, where the body answers a defined question. Blanket retention is not a substitute for an investigation plan.

Sampling has a real cost. A rare bad review may fall outside the sample, leaving only its validation code, latency bucket, bounded model label, and request identifier. Semantic reconstruction then becomes harder. Deliberately keeping less source material buys a smaller exposure surface and a predictable observability bill, while targeted, time-bounded capture can support an authorized investigation.

Keep it bounded.

The decision rule is compact: ship the smallest contract the interface can render; count before sending; validate after receiving; and retain raw bodies only when they change a debugging or quality decision. If chunking reduces accepted-finding quality, relax the latency target or preserve a larger coherent unit. If latency rises without an evaluation gain, send less context. The dashboard should expose that trade rather than bury it beneath stored bytes.

## Further reading

- [OpenAI Structured Outputs guide](https://platform.openai.com/docs/guides/structured-outputs)
- [Anthropic tool use documentation](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview)
- [Google Gemini structured output documentation](https://ai.google.dev/gemini-api/docs/structured-output)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
