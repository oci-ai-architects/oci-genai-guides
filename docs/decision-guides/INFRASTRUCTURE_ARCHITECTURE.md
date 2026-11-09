# OCI GenAI Infrastructure Architecture

**Independent community guide. Published January 2026.** Not affiliated with, endorsed by, or sponsored by Oracle Corporation.

> Service, pricing and API details have not been revalidated since publication. Check the official OCI documentation before relying on them.

This guide provides enterprise architects with infrastructure patterns for deploying OCI GenAI services. All implementations reference official Oracle IaC modules.

---

## Infrastructure Patterns

### Pattern 1: Serverless GenAI API (Simplest)

```
┌─────────────────────────────────────────────────────────────────┐
│                         OCI Infrastructure                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  API Gateway │──▶│  GenAI       │──▶│  OCI GenAI Service  │  │
│  │  (Optional)  │   │  Integration │   │  - Chat API only    │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  VCN with Private Endpoint                               │  │
│  │  - No public internet access to GenAI                    │  │
│  │  - Service gateway for OCI APIs                          │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Use Case**: Simple LLM integration, rapid prototyping, low-volume production

### Pattern 2: Production Agent Deployment

```
┌─────────────────────────────────────────────────────────────────┐
│                         OCI Infrastructure                       │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Application VCN                                           │ │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────────────────┐  │ │
│  │  │ Compute   │  │ Functions │  │ Kubernetes (OKE)      │  │ │
│  │  │ Instance  │  │           │  │ - Agent application   │  │ │
│  │  └─────┬─────┘  └─────┬─────┘  └───────────────────────┘  │ │
│  │        │              │                                    │ │
│  │        └──────────────┼────────────────────────────────────┘ │
│  │                       ▼                                      │
│  │  ┌─────────────────────────────────────────────────────────┐│
│  │  │  Private Endpoint (GenAI)                               ││
│  │  │  - Traffic never leaves OCI                             ││
│  │  │  - Identity-based access control                        ││
│  │  └─────────────────────────────────────────────────────────┘│
│  └────────────────────────────────────────────────────────────┘
│
│  ┌────────────────────────────────────────────────────────────┐
│  │  Data VPC (Optional isolation)                             │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────────────────┐  │
│  │  │ Object    │  │ Database  │  │ Database AI           │  │
│  │  │ Storage   │  │ 23ai      │  │ Vector Search         │  │
│  │  │ (RAG)     │  │           │  │                       │  │
│  │  └───────────┘  └───────────┘  └───────────────────────┘  │
│  └────────────────────────────────────────────────────────────┘
└─────────────────────────────────────────────────────────────────┘
```

**Use Case**: Production agents with custom tools, RAG pipelines, enterprise security

### Pattern 3: Multi-Agent with Agent Hub

```
┌─────────────────────────────────────────────────────────────────┐
│                    OCI AI Agent Platform                         │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Agent Hub (Beta Nov 2025)                                  ││
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────────┐   ││
│  │  │ Agent 1 │  │ Agent 2 │  │ Agent 3 │  │ Shared      │   ││
│  │  │ (RAG)   │  │ (SQL)   │  │ (API)   │  │ Memory      │   ││
│  │  └────┬────┘  └────┬────┘  └────┬────┘  └─────────────┘   ││
│  │       │            │            │                         ││
│  │       └────────────┼────────────┘                         ││
│  │                    ▼                                      ││
│  │       ┌─────────────────────────────────────┐             ││
│  │       │     GenAI Service (Chat API)        │             ││
│  │       │     - Cohere, Meta, OpenAI models   │             ││
│  │       └─────────────────────────────────────┘             ││
│  └─────────────────────────────────────────────────────────────┘
└─────────────────────────────────────────────────────────────────┘
```

**Use Case**: Complex workflows, specialized agents, enterprise orchestration

---

## Networking Architecture

### VCN Design for GenAI

```
┌─────────────────────────────────────────────────────────────────┐
│                         Customer VCN                             │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  10.0.0.0/16 (example)                                      ││
│  │                                                             ││
│  │  ┌─────────────────────────────────────────────────────┐   ││
│  │  │  Public Subnet (Optional)                           │   ││
│  │  │  - Bastion host (if needed)                         │   ││
│  │  │  - Load balancer public endpoint                    │   ││
│  │  └─────────────────────────────────────────────────────┘   ││
│  │                                                             ││
│  │  ┌─────────────────────────────────────────────────────┐   ││
│  │  │  Private Subnet - Application                       │   ││
│  │  │  - Compute instances                                │   ││
│  │  │  - OKE clusters                                     │   ││
│  │  │  - Functions                                        │   ││
│  │  └─────────────────────────────────────────────────────┘   ││
│  │                                                             ││
│  │  ┌─────────────────────────────────────────────────────┐   ││
│  │  │  Private Subnet - Data                              │   ││
│  │  │  - Object Storage (RAG corpus)                      │   ││
│  │  │  - Database 23ai                                    │   ││
│  │  │  - Vault for secrets                                │   ││
│  │  └─────────────────────────────────────────────────────┘   ││
│  └─────────────────────────────────────────────────────────────┘
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐
│  │  Security Lists / NSG Rules                                 │
│  │  - Allow 443 to service gateway (genai.oci)                 │
│  │  - Allow internal traffic between subnets                   │
│  │  - Deny all other outbound                                 │
│  └─────────────────────────────────────────────────────────────┘
└─────────────────────────────────────────────────────────────────┘
```

### Private Endpoint Configuration

OCI GenAI private endpoints ensure traffic never leaves Oracle's network:

```
oci genai private-endpoint create \
  --compartment-id $COMPARTMENT_ID \
  --display-name "genai-private-endpoint" \
  --subnet-id $SUBNET_OCID \
  --lifecycle-state ACTIVE
```

---

## Identity & Access Management

### Required IAM Policies

```oci
# Allow use of GenAI Service
Allow group GenAI-Users to use generative-ai-family in compartment $COMPARTMENT

# Allow management of private endpoints
Allow group GenAI-Admins to manage private-endpoints in compartment $COMPARTMENT

# Allow read from Object Storage (RAG corpus)
Allow group GenAI-Apps to read objects in compartment $COMPARTMENT

# Allow access to Database AI Vector Search
Allow group GenAI-Apps to use database-family in compartment $COMPARTMENT
```

### Dynamic Groups for Agents

```oci
# For agent compute instances
Allow dynamic-group agent-instances to use generative-ai-family in compartment $COMPARTMENT

# For agent functions
Allow dynamic-group agent-functions to use generative-ai-family in compartment $COMPARTMENT
```

---

## Data Architecture

### RAG Pipeline with OCI Services

```
┌─────────────────────────────────────────────────────────────────┐
│                    RAG Data Flow                                 │
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────────────┐ │
│  │ Documents│───▶│  Object  │───▶│  Database 23ai            │ │
│  │ (PDF,    │    │ Storage  │    │  with Vector Search      │ │
│  │  DOCX)   │    │          │    │                          │ │
│  └──────────┘    └──────────┘    └──────────────────────────┘ │
│                                               │                │
│                                               ▼                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  Query Flow                                              │ │
│  │  1. Embed query (GenAI Embeddings API)                   │ │
│  │  2. Vector similarity search                             │ │
│  │  3. Retrieve top-K chunks                                │ │
│  │  4. Construct prompt with context                        │ │
│  │  5. Generate response (GenAI Chat API)                   │ │
│  └──────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Database AI Vector Search Integration

Oracle Database 23ai provides native vector search:

```sql
-- Create vector index for RAG
CREATE SEARCH INDEX rag_index ON documents (embedding)
  FOR VECTOR;

-- Similarity search query
SELECT id, content, similarity
FROM documents
ORDER BY embedding cosine SIMILARITY TO query_embedding
FETCH FIRST 5 ROWS ONLY;
```

---

## Observability Architecture

### OCI Monitoring for GenAI

```
┌─────────────────────────────────────────────────────────────────┐
│                    OCI Observability Stack                       │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Metrics (OCI Monitoring)                                   ││
│  │  - genai.inference.count                                    ││
│  │  - genai.inference.latency                                  ││
│  │  - genai.characters.processed                               ││
│  │  - genai.cost                                               ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Logs (OCI Logging)                                         ││
│  │  - Service logs (automated)                                 ││
│  │  - Custom logs (application)                                ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Tracing (OCI Application Performance Monitoring)           ││
│  │  - End-to-end request tracing                               ││
│  │  - Agent step-by-step visibility                            ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Alerts (OCI Monitoring)                                    ││
│  │  - Latency > threshold                                      ││
│  │  - Cost spike                                               ││
│  │  - Error rate increase                                      ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

---

## Official IaC Resources

### Oracle Cloud Foundation (Terraform)

| Resource | Link | Purpose |
|----------|------|---------|
| OCI Terraform Provider | https://registry.terraform.io/providers/oracle/oci/latest | Core infrastructure |
| OCI GenAI Module | https://github.com/oracle/terraform-oci-genai | GenAI deployment |
| OCI AI Agent Module | https://github.com/oracle/terraform-oci-ai-agents | Agent platform |
| OCI Object Storage | https://registry.terraform.io/providers/oracle/oci/latest/docs/resources/object_storage_bucket | RAG storage |
| OCI Database 23ai | https://registry.terraform.io/providers/oracle/oci/latest/docs/resources/database_db_home | Vector database |

### OCI CLI Reference

| Resource | Link |
|----------|------|
| GenAI CLI Commands | https://docs.oracle.com/en-us/iaas/Content/generative-ai/cli.htm |
| OCI CLI Setup | https://docs.oracle.com/en-us/iaas/Content/Identity/tasks/managingcredentials.htm |

### OCI Resource Manager

- [Deploy GenAI Stack](https://cloud.oracle.com/resource-manager/stacks/create?zipUrl=https://github.com/oracle/terraform-oci-genai/archive/refs/heads/main.zip)

---

## Security Checklist

| Control | Implementation |
|---------|----------------|
| ✅ Private endpoints | Use private endpoint for GenAI |
| ✅ Identity-based access | OCI IAM policies |
| ✅ Network isolation | VCN with security lists/NSGs |
| ✅ Audit logging | OCI Logging for GenAI operations |
| ✅ Secrets management | OCI Vault |
| ✅ Data encryption | Encryption at rest (default) |
| ✅ Cost controls | Quotas and budgets in OCI |
| ✅ Compliance | FedRAMP, SOC, DoD available |

---

## Cost Estimation

### GenAI Cost Components

| Component | Pricing | Estimation |
|-----------|---------|------------|
| GenAI Chat API | $X per 10K characters | Depends on usage volume |
| GenAI Embeddings | $Y per 10K characters | Depends on RAG corpus size |
| Agent Platform | $Z per agent/month | Per agent |
| Agent Hub | $W per month | Plus usage |
| OCI Compute | Per OCPU/hour | For agent infrastructure |
| Database 23ai | Per OCPU/hour | Vector search |
| Object Storage | Per GB/month | RAG corpus |

### Cost Estimation Tools

- [OCI Pricing Calculator](https://www.oracle.com/cloud/price-list/)
- [OCI Cost Estimator](https://www.oracle.com/cloud/estimator/)

---

*Independent community guide | Published January 2026, not revalidated since*
