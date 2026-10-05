# Migration Guide: AWS Bedrock to Oracle OCI GenAI

**Independent community guide. Published January 2026.** Not affiliated with, endorsed by, or sponsored by Oracle Corporation.

> Service, pricing and API details have not been revalidated since publication. Check the official OCI documentation before relying on them.

This guide helps enterprise architects and engineering teams migrate from AWS Bedrock to Oracle OCI GenAI.

---

## Migration Overview

### Key Differences

| Aspect | AWS Bedrock | OCI GenAI |
|--------|-------------|-----------|
| **API Type** | `InvokeModel` | **Chat API only** (GenerateText/SummarizeText deprecated) |
| **Pricing** | Per token | **Per character** |
| **SDK** | `boto3` | `oci` (Python) |
| **Region** | aws-region | oci-region |
| **Auth** | AWS credentials | OCI API keys or instance principal |

---

## Phase 1: Assessment

### 1. Inventory Bedrock Usage

```bash
# List all Bedrock model invocations
aws bedrock list-invocation-jobs --region us-east-1

# Get model usage by foundation model
aws bedrock list-foundation-models --region us-east-1
```

### 2. Identify Models in Use

| Bedrock Model | OCI Equivalent | Notes |
|---------------|----------------|-------|
| `anthropic.claude-3-5-sonnet-20241022` | Not available (use Cohere/Meta) | Requires model change |
| `anthropic.claude-3-haiku-20240307` | Not available | Requires model change |
| `cohere.command-r-plus-v1:0` | `cohere.command-r-plus-08-2024` | Direct equivalent |
| `cohere.command-r-v1:0` | `cohere.command-r-08-2024` | Direct equivalent |
| `meta.llama3-3-70b-instruct-v1:0` | `meta.llama-3.3-70b-instruct` | Version difference |
| `amazon.titan-text-premier-v1:0` | Not available | Requires model change |
| `stability.stable-diffusion-xl-base-v1` | Not available | Different modality |

### 3. Estimate Cost Impact

**Critical**: OCI pricing is per character, not per token.

| Model | Bedrock (per 1K tokens) | OCI (per 10K chars) | Impact |
|-------|-------------------------|---------------------|--------|
| Command R+ | ~$0.003 | ~$0.015 | Higher per-unit, but char/token ratio may offset |
| Command R | ~$0.0005 | ~$0.01 | Needs calculation |
| Llama 3 70B | ~$0.00265 | ~$0.025 | Needs calculation |

**Action**: Run sample workloads through both pricing models to estimate.

---

## Phase 2: Authentication Migration

### AWS (boto3)

```python
import boto3

bedrock = boto3.client(
    service_name='bedrock-runtime',
    region_name='us-east-1',
    aws_access_key_id=AWS_ACCESS_KEY,
    aws_secret_access_key=AWS_SECRET_KEY
)
```

### OCI (oci SDK)

```python
from oci.config import from_file
from oci.generative_ai_inference import GenerativeAiInferenceClient

# Option 1: API Key authentication
config = from_file("~/.oci/config", "DEFAULT")
client = GenerativeAiInferenceClient(config, region="us-phoenix-1")

# Option 2: Instance Principal (for OCI resources)
from oci.auth.signers import InstancePrincipalsSecurityTokenSigner
signer = InstancePrincipalsSecurityTokenSigner()
client = GenerativeAiInferenceClient(config={}, signer=signer)
```

### Authentication Setup

| Step | Action | Link |
|------|--------|------|
| 1 | Create OCI API keys | [OCI Credentials](https://docs.oracle.com/en-us/iaas/Content/Identity/tasks/managingcredentials.htm) |
| 2 | Configure OCI CLI | [OCI CLI Setup](https://docs.oracle.com/en-us/iaas/Content/Identity/tasks/managingcredentials.htm) |
| 3 | Set up IAM policies | [OCI IAM for GenAI](https://docs.oracle.com/en-us/iaas/Content/generative-ai/iam.htm) |

---

## Phase 3: API Migration

### AWS Bedrock Invocation

```python
# AWS Bedrock
import boto3
import json

bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')

response = bedrock.invoke_model(
    modelId='cohere.command-r-plus-v1:0',
    body=json.dumps({
        "message": "Hello, world",
        "max_tokens": 100,
        "temperature": 0.7
    })
)
result = json.loads(response['body'].read())
```

### OCI GenAI Chat API (Required)

```python
# OCI GenAI - Chat API ONLY (not InvokeModel)
from oci.generative_ai_inference import GenerativeAiInferenceClient
from oci.generative_ai_inference.models import CohereChatRequest, OnDemandServingMode

config = from_file("~/.oci/config", "DEFAULT")
client = GenerativeAiInferenceClient(config, region="us-phoenix-1")

request = CohereChatRequest(
    message="Hello, world",
    temperature=0.7
)

response = client.chat(
    chat_details=request,
    compartment_id=config["tenancy"],
    serving_mode=OnDemandServingMode(model_id="cohere.command-r-plus-08-2024")
)

result = response.data.text
```

### API Comparison Table

| Operation | AWS Bedrock | OCI GenAI |
|-----------|-------------|-----------|
| **Endpoint** | `bedrock-runtime` | `genai-inference` |
| **Method** | `InvokeModel` | `Chat` (Chat API) |
| **Model ID** | `provider.model-version` | `provider.model-version` (versioned) |
| **Request Body** | Provider-specific | Provider-specific |
| **Response** | `body` field | `data.text` field |

---

## Phase 4: Agent Migration

### AWS Bedrock Agents

```typescript
// AWS Bedrock Agent
const agent = new bedrock.Agent({ /* config */ });

// Add knowledge base
agent.associateKnowledgeBase({
  knowledgeBaseId: kbId,
  description: "Product docs"
});

// Create alias for deployment
agent.createAgentAlias({ agentId, aliasName: "prod" });
```

### OCI AI Agent Platform

```python
# OCI AI Agent Platform
# Reference: https://docs.oracle.com/en/solutions/deploy-agentic-ai-agent-platform/

from oci.generative_ai_agent import AgentsClient
from oci.generative_ai_agent.models import Agent, Tool

# Create agent with tools
agent = Agent(
    display_name="Customer Support Agent",
    description="Handles customer inquiries",
    tools=[
        # Define custom tools (SQL, API, RAG, etc.)
    ]
)

# Deploy via OCI Console or API
# Reference: https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/overview.htm
```

### Agent Platform Comparison

| Capability | AWS Bedrock Agents | OCI AI Agent Platform |
|------------|-------------------|----------------------|
| **Knowledge Base** | ✅ Native | RAG via Object Storage + Vector Search |
| **Custom Tools** | ✅ Function calling | ✅ Custom tool definitions |
| **Orchestration** | ✅ Built-in | Agent Hub (Beta Nov 2025) |
| **Guardrails** | ✅ Bedrock Guardrails | ✅ OCI AI Guardrails |
| **Monitoring** | CloudWatch | OCI Monitoring |

---

## Phase 5: Knowledge Base Migration

### AWS Bedrock Knowledge Bases

```
S3 Bucket (documents)
       │
       ▼
Bedrock Knowledge Base (indexed)
       │
       ▼
OpenSearch orpgvector (vector store)
```

### OCI RAG Architecture

```
OCI Object Storage (documents)
          │
          ▼
OCI Document Understanding (optional)
          │
          ▼
OCI Database 23ai Vector Search
(Oracle Database native vector indexes)
```

### Migration Steps

1. **Export from Bedrock Knowledge Base**
   ```bash
   # List knowledge base IDs
   aws bedrock list-knowledge-bases --region us-east-1
   
   # Export documents from associated S3
   aws s3 sync s3://bedrock-kb-bucket ./local-docs/
   ```

2. **Import to OCI**
   ```bash
   # Upload to OCI Object Storage
   oci os object bulk-upload \
     --namespace $NAMESPACE \
     --bucket-name genai-rag-corpus \
     --source-dir ./local-docs
   
   # Configure RAG pipeline with Database 23ai Vector Search
   # Reference: https://docs.oracle.com/en/solutions/oci-genai-rag/
   ```

### RAG Reference Architectures

| Cloud | Resource |
|-------|----------|
| **OCI** | [GenAI RAG Solution](https://docs.oracle.com/en/solutions/oci-genai-rag/) |
| **OCI** | [Multicloud RAG with OCI](https://docs.oracle.com/en/solutions/oci-multicloud-genai-rag/) |
| **AWS** | [Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html) |

---

## Phase 6: Guardrails Migration

### AWS Bedrock Guardrails

```python
guardrail = bedrock.create_guardrail(
    name="content-filter",
    blocked_inputMessaging="I cannot help with that",
    blockedOutputsMessaging="I cannot provide that response",
    # Configure topic, word, sensitive content filters
)
```

### OCI AI Guardrails

```python
# Reference: https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/guardrails.htm

from oci.generative_ai_agent.models import Guardrail

guardrail = Guardrail(
    name="content-safety",
    input_guardrail_config={ /* topics, word filters */ },
    output_guardrail_config={ /* PII, profanity filters */ }
)
```

---

## Migration Checklist

| Phase | Task | Status |
|-------|------|--------|
| **1. Assessment** | Inventory Bedrock usage | ⬜ |
| | Identify model equivalents | ⬜ |
| | Estimate cost impact | ⬜ |
| **2. Auth** | Set up OCI credentials | ⬜ |
| | Configure IAM policies | ⬜ |
| **3. API** | Update SDK calls to OCI | ⬜ |
| | Change to Chat API only | ⬜ |
| | Test with equivalent models | ⬜ |
| **4. Agents** | Recreate agents in OCI | ⬜ |
| | Migrate knowledge base | ⬜ |
| | Configure guardrails | ⬜ |
| **5. Deployment** | Update CI/CD pipelines | ⬜ |
| | Configure monitoring | ⬜ |
| | Test production workload | ⬜ |

---

## Timeline Estimation

| Phase | Effort | Duration |
|-------|--------|----------|
| Assessment | 1 FTE week | 1-2 weeks |
| Authentication | 1 FTE day | 1 day |
| API Migration | 1-2 FTE weeks | 1-2 weeks |
| Agent Migration | 2-4 FTE weeks | 3-4 weeks |
| Knowledge Base | 1-2 FTE weeks | 2-3 weeks |
| Testing & Deployment | 2 FTE weeks | 2-3 weeks |
| **Total** | | **10-15 weeks** |

---

## Rollback Plan

| Scenario | Action |
|----------|--------|
| OCI GenAI unavailable | Redirect to Bedrock via feature flag |
| Performance regression | Fallback to Bedrock with alerting |
| Cost overrun | Implement OCI budgets + alerts |

---

## Resources

### Official Documentation
- [OCI GenAI Documentation](https://docs.oracle.com/en-us/iaas/Content/generative-ai/home.htm)
- [OCI AI Agent Platform](https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/overview.htm)
- [AWS Bedrock Documentation](https://docs.aws.amazon.com/bedrock/)
- [OCI Chat API Reference](https://docs.oracle.com/en-us/iaas/Content/generative-ai/chat.htm)

### SDK Reference
- [OCI Python SDK](https://docs.oracle.com/en-us/iaas/tools/python/2.164.0/)
- [OCI GenAI SDK Examples](https://github.com/oracle/oci-python-sdk/tree/main/examples/generative_ai)

### Migration Tools
- [OCI CLI](https://docs.oracle.com/en-us/iaas/Content/CLI/overview.htm)
- [OCI Resource Manager](https://docs.oracle.com/en-us/iaas/Content/ResourceManager/home.htm)

---

*Independent community guide | Published January 2026, not revalidated since*
