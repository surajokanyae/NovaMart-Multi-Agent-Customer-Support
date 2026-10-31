<div align="center">

# 🤖 NovaMart Multi-Agent Customer Support

**A production-style, multi-agent AI system that resolves customer order, refund and policy requests end to end.**

Built with **Strands Agents SDK** · **Amazon Bedrock** · **AgentCore** · **DynamoDB** · **S3 Vectors**

![Python](https://img.shields.io/badge/Python-3.12-blue)
![AWS](https://img.shields.io/badge/AWS-Bedrock%20AgentCore-orange)
![Agents](https://img.shields.io/badge/Agents-5-green)
![RAG](https://img.shields.io/badge/RAG-3%20Knowledge%20Bases-purple)

</div>

---

## 💡 The Problem

Customer support teams spend most of their time on repetitive questions: *"Where is my order?"*, *"Can I return this?"*, *"What does the warranty cover?"* Answering correctly means combining **live order data**, **customer tier rules** and **written company policy**, then explaining the result kindly.

A single prompt-based chatbot handles this poorly. It guesses, ignores tier rules, and can't be audited.

## ✅ The Solution

A team of **five specialized agents** that each do one job well, share verified state, and are protected by safety guardrails:

| Agent | Responsibility |
|---|---|
| 🎯 **Orchestrator** | Starts the session, routes work to the right specialists, manages shared state |
| 📦 **Inventory Agent** | Fetches real order and customer-tier data from DynamoDB (facts only, no decisions) |
| 📚 **Policy Agent** | Multi-agent RAG: three retriever sub-agents search Returns, Shipping and Warranty knowledge bases **in parallel** |
| 💳 **Refund Agent** | Decides eligibility using the right window per tier (Standard 30 days, Premium 60 days) and initiates returns |
| 💬 **Communication Agent** | Reads the full workflow state and writes a clear, empathetic customer reply |

---

## 🏗️ Architecture

```
                    Customer Request
                           │
                 ┌─────────▼─────────┐
                 │ OrchestratorAgent │◄──── Bedrock Guardrails (all agents)
                 └─────────┬─────────┘
      ┌──────────┬─────────┴────┬──────────────┐
      ▼          ▼              ▼              ▼
 Inventory     Policy         Refund      Communication
  Agent        Agent          Agent          Agent
 (DynamoDB)  (Multi-agent     (DynamoDB)    (final reply)
              RAG)
                │
     ┌──────────┼──────────┐        ← run in parallel
     ▼          ▼          ▼
  Returns    Shipping   Warranty
    KB          KB         KB
 (S3 Vectors)

        Shared state: DynamoDB WorkflowState table
        (optimistic locking, 24h TTL)
```

---

## ⭐ Engineering Highlights

**🔒 Safe concurrent state with optimistic locking**
Every agent writes to a shared DynamoDB `WorkflowState` record. Updates are conditional on a `version` counter, with automatic retry and backoff on conflict, so no agent silently overwrites another's work.

**⚡ Parallel multi-agent RAG**
The Policy Agent fans out to three retriever agents using a thread pool, each querying its own Knowledge Base, then synthesizes one grounded answer. Lower latency and cleaner separation of knowledge domains.

**🛡️ Enterprise guardrails, tuned on real behavior**
Content filters (sexual, violence, hate, insults, misconduct), PII handling (block card and SSN numbers, anonymize emails and phone numbers), profanity filtering and denied topics (competitor products, price haggling, legal threats). The pricing-negotiation topic was carefully defined so legitimate arithmetic like *"5 items at $29.99 with 10% off"* is allowed while haggling is blocked.

**🚀 Reproducible deployment**
One command stages the code, packages arm64 dependencies and deploys to **AgentCore Runtime** through the AgentCore CLI. Re-running updates in place.

**🧠 Memory and 🔭 Observability**
AgentCore Memory keeps session summaries for 7 days. Every request is traced in **AWS X-Ray** (each worker agent appears as a node on the Service Map) with logs in **CloudWatch**.

**🧩 Clear separation of concerns**
Orchestrator uses a fast model for routing, workers focus on a single task, and tools are instrumented as X-Ray subsegments for easy debugging.

---

## 🛠️ Tech Stack

| Area | Technology |
|---|---|
| Agent framework | Strands Agents SDK |
| LLMs | Amazon Bedrock (Claude models) |
| Runtime and deployment | Amazon Bedrock AgentCore Runtime, AgentCore CLI |
| Data | Amazon DynamoDB |
| Retrieval | Bedrock Knowledge Bases on S3 Vectors |
| Safety | Amazon Bedrock Guardrails |
| Memory | AgentCore Memory (session summaries) |
| Observability | AWS X-Ray, CloudWatch Logs |
| Language | Python 3.12, boto3 |

---

## 🎬 Demo

Check Images folder
---

## 🚀 Getting Started

### Prerequisites
- AWS account with Amazon Bedrock model access
- Python 3.12, Node.js 20+, and `uv` (for AgentCore CLI 0.30.0)
- Three Knowledge Bases created and synced (Returns, Shipping, Warranty)

### Configure
Create a `.env` file (never commit it):

```
AWS_REGION=us-east-1
PROJECT_NAME=your-project-name
RETURNS_KB_ID=
SHIPPING_KB_ID=
WARRANTY_KB_ID=
GUARDRAIL_ID=
GUARDRAIL_VERSION=
AGENTCORE_RUNTIME_ARN=
```

### Run

```bash
python src/agent_orchestrator.py test            # 3 sample scenarios, traced to X-Ray
python src/agent_orchestrator.py chat            # interactive chat with live agent trace
python src/agent_orchestrator.py deploy          # guardrail, runtime, memory, observability
python src/agent_orchestrator.py invoke "<msg>"  # call the deployed runtime
```

### Sample scenarios

| Customer | Request | What happens |
|---|---|---|
| CUST-001 (Premium) | Return wireless headphones, ORD-27176 | Order lookup, tier check, refund decision, reply |
| CUST-002 (Standard) | Return policy for premium customers? | Parallel RAG across policy knowledge bases |
| CUST-003 (Premium) | 5 items at $29.99 with 10% off? | Math only, skips data agents, still replies via Communication Agent |

---

## 📁 Project Structure

```
.
├── README.md
├── config.py                       # Environment-based settings
├── infrastructure/
│   └── seed_data.py                # Sample customers and orders
└── src/
    ├── agent_orchestrator.py       # Agents, guardrail, deployment, CLI
    ├── agent_utils.py              # Terminal trace UI
    ├── agent_observability.py      # X-Ray and logging
    ├── agentcore_cli.py            # AgentCore CLI wrapper
    └── bedrock_kb_retrieval.py     # Knowledge Base retrieval
```

---

## 📈 What I Learned

- Designing **multi-agent systems** with clear roles instead of one overloaded prompt
- Managing **shared state safely** across agents with optimistic concurrency
- Building **RAG** with multiple knowledge bases and parallel retrieval
- Applying **responsible AI** controls with guardrails and testing them for false positives
- **Deploying and observing** agents on AWS with tracing and logging

## 🔮 Possible Next Steps

- Connect agents to the AgentCore Gateway (MCP) for independently versioned tools
- Add automated evaluation of response quality
- Add a web chat front end

---

## 👤 Author

**Surajo Garba** · [LinkedIn](www.linkedin.com/in/surajo-garba-77324a229) · [Email](mailto:surajogarba80@gmail.com)

*Built as part of the Udacity program and extended as a personal portfolio project.*

## 📄 License

See [LICENSE](LICENSE).
