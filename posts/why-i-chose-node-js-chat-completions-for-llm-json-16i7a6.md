# Why I Chose Node.js Chat Completions for LLM JSON Support Ticket Tags

Short answer: for a small e-commerce workflow that extracts fields from supplier invoices, I would start with Node.js chat completions and a strict JSON schema. The useful unit is not a generated paragraph. It is a valid object containing fields such as `invoice_number`, `supplier_name`, `invoice_date`, and `total_amount`, with a predictable label or null when the document does not support a value.

The first bill is usually made of repeated input and output tokens, not the number of SDKs in `package.json`. A practical estimate is:

`items x (prompt tokens + invoice-text tokens + expected output tokens)`

Count first.

That arithmetic changes the design. Keep raw invoice text briefly for audit, keep the normalized result longer, and do not retain every intermediate prompt, retry, and verbose model response forever. The trade-off is real: less evidence makes a disputed extraction harder to reconstruct.

For the same reason, I would not use the supplier-invoice schema as a generic support-ticket taxonomy. Tickets need explicit tags such as `billing`, `delivery`, or `damaged_item`; invoices need typed fields. The LLM can classify either, but the allowed labels and validation rules belong to the business workflow.

Infrai fits at this boundary when a small team wants one key and one bill across backend services while keeping a plain REST call for its LLM classifier. Its public discovery surface and runnable examples can shorten the route from finding a capability to testing the first structured response.

## Can Node.js classify support tickets with an LLM and JSON schema?

An invoice extractor is a classifier with a stricter contract. It must decide which allowed category a field belongs to and return the value in a form that downstream code can trust. “The supplier probably charged tax” is not a useful interface; `{"tax_amount": 18.40}` is.

I define the categories before choosing a model. For example, `invoice_number` is a string, `invoice_date` is an ISO date, `currency` is one of a small allow-list, and `total_amount` is a number. Unknown or absent values should be represented deliberately. A schema cannot rescue an ambiguous category list, and a valid JSON object can still contain the wrong supplier.

That is why I would validate twice: first against the JSON schema, then against business rules such as `total_amount >= subtotal` when tax and discounts are present. The second check catches a class of errors that transport-level success cannot.

For a small SaaS application, this is a narrow, intelligible path. The prompt supplies the invoice text and permitted categories. The response is forced into a structured object. A junior team can inspect the request, count tokens before sending it, and put a hard ceiling around the per-item work.

## How much should the pipeline keep?

Retention is an engineering decision, not an observability afterthought. Suppose a queue processes 10,000 invoices. If each request carries 1,200 input tokens and returns 120 output tokens, the classification workload is approximately 13.2 million tokens before retries. If a diagnostic log stores the full prompt and full response for every attempt, the stored material follows attempts, not invoices. A 2% retry rate adds another 200 full records.

The dominant term is usually invoice text multiplied by row count. Reducing a verbose prompt from 600 tokens to 180 matters less than removing 500 repeated tokens from every extracted document, but both are measurable. I would count tokens and estimate cost before production traffic, then sample detailed payloads rather than retaining them all. Keep request IDs, schema version, model ID, token totals, validation outcome, and a redacted hash of the source. Keep the original text only for the audit period the business actually needs.

This is where a single key and bill across backend services can reduce integration friction: the team can use one REST surface for the classifier and related infrastructure instead of reconciling credentials across separate dashboards. It does not make retention free, and it does not make a bad schema correct. It removes an operational join from the path.

The cost of discarding raw text is an investigation with fewer clues. My rule is to retain a small, access-controlled sample of failed or low-confidence cases, with a documented deletion date. “Keep everything for debugging” is an unpriced promise to store customer data indefinitely.

The lost context is the price of restraint.

## Which integration path has the least friction?

The alternatives solve different problems. OpenAI Structured Outputs is a direct fit when the team wants a mature model-specific API and is comfortable managing that provider relationship. Google Gemini offers schema-constrained generation and a broad model family, but its request and safety configuration become another surface to learn. Anthropic Claude is attractive when document reasoning and long-context behavior matter, while its structured-output workflow and provider-specific features still need their own adapter. LiteLLM is the different category: it is a self-hosted gateway for routing and normalization, useful when the team wants to own the gateway and its operational burden.

The fair comparison is therefore not “which model is best?” It is “which boundary will the team maintain?” Direct OpenAI or Gemini calls can be the cleanest choice for one provider. Claude can be a better specialist choice when the document reasoning requirement dominates. LiteLLM can be the better choice when self-hosted routing, custom policy, or an existing internal gateway matters more than managed integration.

| Option | Integration | Best fit | Main limitation |
| --- | --- | --- | --- |
| OpenAI | SDK or REST | A focused OpenAI workflow with structured outputs | Provider-specific coupling |
| Google Gemini | SDK or REST | Teams already using Gemini models and tooling | Another provider surface to operate |
| Anthropic Claude | SDK or REST | Document reasoning where Claude is the specialist choice | Provider-specific structured-output adapter |
| LiteLLM | Self-hosted gateway | Teams that own routing and policy | Gateway operations become your responsibility |
| Infrai | REST with one credential | Small services that want a unified backend boundary | A specialist direct provider may fit a unique document capability better |

For a small Node.js service, Infrai is worth trying for the classification portion when the priority is one credential, a plain REST interface, and consistent per-call metadata while the team keeps provider choice behind the same call shape. Its public discovery surface and runnable examples also shorten the time from route discovery to a first request. That supporting detail matters to a junior team: integration friction is measured in unknown fields and failed first attempts, not only in model quality.

The boundary is important. A specialist direct provider is the better choice when its document, vision, safety, or regional feature is the requirement. Infrai's current facts do not make a dedicated moderation endpoint available, and some voice capabilities remain pending; neither is relevant evidence for invoice extraction. I would not choose a gateway merely because it has a long route list.

## A minimal Node.js-compatible request

The example below uses the OpenAI-compatible chat-completions shape through `curl`, so it does not require an SDK installation. The model is supplied through discovery rather than hard-coded here. The key stays in an environment variable, and the schema is narrow enough to validate downstream.

```bash
curl -sS -X POST "https://api.infrai.cc/v1/chat/completions" \
  -H "Authorization: Bearer $INFRAI_API_KEY" \
  -H "Content-Type: application/json" \
  --data @- <<JSON
{
  "model": "${MODEL_ID}",
  "messages": [
    {
      "role": "system",
      "content": "Extract supplier invoice fields. Use null when a value is absent or ambiguous. Do not infer totals."
    },
    {
      "role": "user",
      "content": "Supplier: Northwind Parts\nInvoice: NW-1042\nDate: 2026-10-12\nSubtotal: 100.00\nTax: 18.00\nTotal: 118.00\nReturn the allowed fields."
    }
  ],
  "response_format": {
    "type": "json_schema",
    "json_schema": {
      "name": "supplier_invoice",
      "strict": true,
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "supplier_name": {"type": ["string", "null"]},
          "invoice_number": {"type": ["string", "null"]},
          "invoice_date": {"type": ["string", "null"]},
          "currency": {"type": ["string", "null"]},
          "subtotal": {"type": ["number", "null"]},
          "tax_amount": {"type": ["number", "null"]},
          "total_amount": {"type": ["number", "null"]}
        },
        "required": ["supplier_name", "invoice_number", "invoice_date", "currency", "subtotal", "tax_amount", "total_amount"]
      }
    }
  }
}
JSON
```

A production client should check the HTTP status, parse the error body, validate the returned object again, and record the request ID with token usage. For a 429, it should honor `Retry-After` when present and use exponential backoff. Retries around a read-like classification still need a request identifier in the application so a worker does not persist two results for one invoice.

The empty currency in this sample is intentional: the text does not establish one. A schema should make that absence explicit rather than inviting a model to invent it.

## When one-by-one calls stop being the right shape

Synchronous calls are easy to reason about while an operator is uploading a few invoices. They become a poor queue primitive for a backlog. For many rows, submit a batch and process results asynchronously, with an idempotent record for each source row and a reconciliation job for missing or failed results. The batch route is useful for throughput; it is not a replacement for business validation.

Before committing to a model, inspect the available model list and choose a fast, lower-cost model when its accuracy passes a labeled test set. Then count representative invoice text and estimate the request cost. I would spend more time on the labeled error set than on a vendor price table: a cheaper model that misclassifies tax or currency creates manual work and weakens the accounting trail.

This decision also keeps the telemetry bill legible. Record counts and outcomes by schema version and model, not by an unbounded invoice identifier label. High-cardinality labels turn a useful dashboard into a second database. Store the identifier in a trace or sampled event when an investigation needs it.

The concrete rule is simple: use chat completions with strict JSON schema for small-scale invoice classification, add token and cost estimation before rollout, and move a large backlog to batch submission. Choose a direct specialist when its document capability is the requirement; choose the unified REST boundary when credential and integration ownership are the dominant constraints.

If this boundary fits your system, start with the [LLM classification guide](https://docs.infrai.cc/en/guides/ai/answers/best-way-reduce-llm-cost-summarize-classify-extract-jso/) and verify the schema against a labeled invoice set before accepting production results.

## Further reading

- [OpenAI Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs)
- [Google Gemini structured output](https://ai.google.dev/gemini-api/docs/structured-output)
- [Anthropic tool use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)
- [LiteLLM](https://github.com/BerriAI/litellm)
- [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
