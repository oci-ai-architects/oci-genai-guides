# OCI GenAI Guides

> **Disclaimer:** Unofficial community project. Not affiliated with, endorsed by, or sponsored by Oracle Corporation. Oracle and OCI are trademarks or registered trademarks of Oracle Corporation. Other names are marks of their respective owners.

An independent community project. A curated collection of architectural guidance, implementation patterns, and implementation examples for building enterprise GenAI systems on Oracle Cloud Infrastructure.

> Published January 2026. Service, pricing and API details have not been revalidated since; check the official OCI documentation before relying on them.

---

## Oracle AI Services Landscape

Understanding the Oracle AI ecosystem is essential for solution architects designing production systems.

| Service | Purpose | GA Status |
|---------|---------|-----------|
| **OCI GenAI Service** | LLM inference via Chat API | GA |
| **OCI AI Agent Platform** | Enterprise agent orchestration | GA (March 2025) |
| **Agent Hub** | Multi-agent routing & coordination | Beta (Nov 2025) |
| **AI Agent Studio** | Low-code agent builder for Fusion Apps | GA (March 2025) |

### Key Architectural Decision

**OCI GenAI Service ≠ AI Agent Platform**

The GenAI Service provides raw LLM inference through the **Chat API** (GenerateText/SummarizeText APIs are deprecated as of June 2026). The AI Agent Platform builds on this foundation, adding tool calling, memory management, and enterprise orchestration capabilities.

---

## Recommended Reading

**For Solution Architects:**

- [Production Architecture Guide](github-ai-coe/best-practices/PRODUCTION_LLM_SYSTEMS_ARCHITECTURE.md): Enterprise six-plane model for GenAI on OCI
- [AI Services Decision Guide](docs/decision-guides/ORACLE_AI_SERVICES_DECISION_GUIDE.md): Service selection framework

**For Developer Integration:**

- [OpenCode Integration Guide](content/blog/connecting-opencode-agents-to-oracle-genai.md): Connect IDE coding agents to OCI GenAI
- [QUICKSTART.py](github-ai-coe/QUICKSTART.py): Example CLI with cost tracking

**Reference Architectures:**

- [Enterprise RAG](github-ai-coe/reference-architectures/enterprise-rag/README.md): Retrieval-augmented generation at scale

---

## Quick Start: Chat API Integration

The GenAI Service uses the **Chat API** exclusively. Ensure your code reflects this:

```python
from oci.generative_ai_inference import GenerativeAiInferenceClient
from oci.generative_ai_inference.models import (
    ChatDetails, CohereChatRequest, OnDemandServingMode
)

client = GenerativeAiInferenceClient(config, region="us-phoenix-1")
request = CohereChatRequest(message="Hello", temperature=0.7)
response = client.chat(
    chat_details=request,
    compartment_id=compartment_id,
    serving_mode=OnDemandServingMode(model_id="cohere.command-r-plus-08-2024")
)
```

**Do not use:** `GenerateText`, `SummarizeText` (deprecated June 2026)

---

## Pricing Model

OCI GenAI pricing is **per character**, not per token.

| Operation | Billing Unit |
|-----------|--------------|
| Chat API | prompt_chars + response_chars |
| Embeddings | input_chars |

10,000 characters = 10,000 transactions.

---

## Oracle Reference Repositories

| Repository | Purpose |
|------------|---------|
| [oracle/langchain-oracle](https://github.com/oracle/langchain-oracle) | Official LangChain integration |
| [oracle/agent-spec](https://github.com/oracle/agent-spec) | Open Agent Specification |
| [oracle-devrel/ai-solutions](https://github.com/oracle-devrel/ai-solutions) | Reference implementations |
| [oracle-quickstart/oci-ai-blueprints](https://github.com/oracle-quickstart/oci-ai-blueprints) | OKE deployments |

---

## Document Versioning

| Version | Date | Changes |
|---------|------|---------|
| 1.1 | Jan 2026 (publication date, not revalidated since) | Chat API migration, Agent Platform updates |
| 1.0 | Jan 2025 | Initial release |

---

## License

MIT. See [LICENSE](LICENSE).

---

*Independent community project maintained by [frankxai](https://github.com/frankxai). Verify services, pricing and API status against the official OCI documentation before relying on them.*
