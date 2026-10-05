# Connecting OpenCode Agents to Oracle GenAI

**Independent community guide. Published January 2026.** Not affiliated with, endorsed by, or sponsored by Oracle Corporation.

> Service, pricing and API details have not been revalidated since publication. Check the official OCI documentation before relying on them.

This guide provides solution architects and developers with a streamlined approach to integrating OpenCode, Roo Code, and similar IDE coding agents with Oracle Cloud Infrastructure GenAI Service.

---

## Architecture Decision

The integration pattern is straightforward: your coding agent sends requests to the OCI GenAI **Chat API**, which routes to the appropriate model (Cohere, Meta, or OpenAI).

```
┌─────────────┐     Chat API      ┌─────────────────────┐
│ OpenCode    │ ───────────────► │ OCI GenAI Service   │
│ Roo Code    │                   │ - Cohere Command R+ │
│ Kilo Code   │                   │ - Meta Llama 3.3    │
└─────────────┘                   └─────────────────────┘
```

**Note:** The OCI GenAI Service uses the **Chat API exclusively**. The GenerateText and SummarizeText APIs are deprecated as of June 2026.

---

## Prerequisites

1. OCI account with GenAI Service access
2. OCI API keys configured (`~/.oci/config`)
3. Compartment OCID with GenAI permissions

---

## Python Integration Pattern

Use the `oci.generative_ai_inference` SDK:

```python
from oci.generative_ai_inference import GenerativeAiInferenceClient
from oci.generative_ai_inference.models import CohereChatRequest, OnDemandServingMode

config = oci.config.from_file("~/.oci/config", "DEFAULT")
client = GenerativeAiInferenceClient(config, region="us-phoenix-1")

request = CohereChatRequest(
    message="Explain this Python function: def fib(n): return n if n < 2 else fib(n-1) + fib(n-2)",
    temperature=0.2
)

response = client.chat(
    chat_details=request,
    compartment_id=config["tenancy"],
    serving_mode=OnDemandServingMode(model_id="cohere.command-r-plus-08-2024")
)

print(response.data.text)
```

**Pricing:** Per character, not per token. 10,000 characters = 10,000 transactions.

---

## Model Selection

| Use Case | Model ID | Context |
|----------|----------|---------|
| Code explanation | `cohere.command-r-plus-08-2024` | 128K |
| Code completion | `meta.llama-3.3-70b-instruct` | 128K |
| Fast/cheap tasks | `cohere.command-r-08-2024` | 128K |

---

## Security Best Practices

1. **Never commit API keys** - Use OCI Vault for secrets
2. **Least privilege** - Create dedicated IAM group for GenAI access
3. **Network isolation** - Use private endpoints for production

---

*Independent community guide | Version 1.1, published January 2026, not revalidated since*
