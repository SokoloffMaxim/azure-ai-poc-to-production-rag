# Architecture Notes: Azure AI POC to Production-Ready RAG

These notes describe the architecture thinking behind moving an Azure AI proof of concept into a more production-ready Retrieval Augmented Generation solution.

The goal is not to document every possible Azure service. The goal is to capture the main design decisions that matter when an AI demo starts becoming a real business application.

## 1. Starting point: simple Azure AI POC

A typical Azure AI POC starts with a simple flow:

```text
User → Chat UI → Orchestrator → Test data → Azure OpenAI model → Response
```

This is useful for proving the idea quickly.

It can answer questions such as:

* Can the model understand the user’s request?
* Can the application send a prompt and receive a response?
* Can we connect a basic UI to an Azure OpenAI model?
* Can we demonstrate value to the business?

However, this flow does not prove that the solution is ready for production.

## 2. Why the POC is not enough

A POC usually does not fully address:

* private business data
* trusted document sources
* network isolation
* identity and access control
* quota and throttling
* monitoring and logging
* cost visibility
* response quality testing
* support ownership
* deployment automation

This is the main architectural gap.

The question changes from:

> Can we build an AI demo?

to:

> Can we run this safely, reliably, and repeatedly as a business service?

## 3. Target production direction

A more realistic production direction looks like this:

```text
User
  → Application Gateway with WAF
  → App Service or front-end application
  → Orchestrator or agent layer
  → Azure AI Search
  → Azure OpenAI model in Microsoft Foundry
  → Response grounded in business data
```

This design separates the main responsibilities:

| Layer                     | Responsibility                                   |
| ------------------------- | ------------------------------------------------ |
| User interface            | Provides the chat or application experience      |
| Application Gateway / WAF | Protects and inspects inbound traffic            |
| App Service               | Hosts the front end, API, or orchestration code  |
| Orchestrator              | Controls the AI workflow and prompt construction |
| Azure AI Search           | Retrieves relevant business content              |
| Azure OpenAI model        | Generates the final answer from grounded context |
| Monitoring                | Tracks usage, failures, latency, and cost        |

## 4. RAG architecture

Retrieval Augmented Generation allows the model to answer using trusted business data instead of relying only on its training data.

The RAG pipeline looks like this:

```text
Documents
  → Cleaning
  → Chunking
  → Metadata enrichment
  → Embeddings
  → Azure AI Search index
  → Retrieval
  → Model response
  → Evaluation
```

The quality of the answer depends heavily on the quality of the retrieval process.

If the wrong chunks are retrieved, the model may still produce a fluent answer, but the answer may be weak, incomplete, or misleading.

## 5. Document ingestion decisions

Before indexing documents, the team should decide:

* which documents are approved sources of truth
* who owns the documents
* how often the documents change
* whether old versions should remain searchable
* whether some documents require restricted access
* whether answers must include source references

This is not just a technical step. It needs business ownership.

## 6. Chunking strategy

Chunking is one of the most important RAG design decisions.

Possible approaches include:

* fixed-size chunks
* sentence-based chunks
* paragraph-based chunks
* section-based chunks
* semantic chunking
* overlapping chunks

There is no universal best option.

For policies and legal-style documents, section-aware chunking is often more useful than blindly splitting text every fixed number of tokens. For simple FAQ-style content, smaller chunks may work well.

Chunking should be tested, not guessed.

## 7. Metadata

Metadata helps retrieval, filtering, traceability, and user trust.

Useful metadata may include:

* document name
* document type
* business area
* section heading
* version
* published date
* source URL or file path
* confidentiality level
* owner
* last indexed date

Good metadata makes the solution easier to operate later.

## 8. Azure AI Search

Azure AI Search is the retrieval layer.

It can support different search approaches, including:

* keyword search
* vector search
* hybrid search
* semantic ranking

For many enterprise RAG scenarios, hybrid search is a strong starting point because it combines keyword matching with vector similarity.

The search index should be designed around how users actually ask questions, not only around how documents are stored.

## 9. Orchestration layer

The orchestrator is responsible for controlling the AI workflow.

It may handle:

* user input validation
* intent detection
* query rewriting
* retrieval from Azure AI Search
* prompt construction
* model call
* response formatting
* citations
* fallback behaviour
* logging

In a simple POC, this may be a small amount of code.

In production, the orchestration layer becomes more important because it controls how the business rules, retrieval, and model behaviour come together.

## 10. API gateway pattern

In a POC, the application may call the model deployment directly.

In production, an API gateway should be considered between the application and model deployments.

```text
Application → API Management → Primary model deployment
                           → Secondary model deployment or fallback path
```

The gateway can help with:

* centralised access control
* quota management
* token usage control
* throttling behaviour
* routing between model deployments
* supporting multiple applications
* reducing application code changes when backend models change

This becomes important when the model endpoint returns HTTP 429 responses due to throttling or quota limits.

## 11. Throttling and HTTP 429

HTTP 429 means the service is receiving too many requests or has reached a rate limit.

A basic application may simply fail or retry.

A better production pattern is to plan for throttling:

* use retry with backoff where appropriate
* monitor token usage and request volume
* understand quota limits
* consider provisioned throughput for predictable workloads
* consider fallback routing through an API gateway
* avoid exposing raw service failures directly to users

The user experience should be controlled even when the backend is under pressure.

## 12. Private networking

For sensitive data, private networking should be reviewed early.

A production design may include:

* private endpoints for Microsoft Foundry resources
* private endpoints for Azure AI Search
* private endpoints for storage accounts
* VNet integration for App Service
* private DNS zones
* disabled public network access where supported
* managed identity for service-to-service authentication

Private networking needs proper design.

Private endpoints without correct DNS and routing can make the solution harder to troubleshoot.

## 13. Identity and access control

The preferred direction is to use managed identity and RBAC instead of secrets in code.

Important principles:

* use least privilege
* separate application identity from developer identity
* avoid shared keys where possible
* keep secrets out of source control
* keep secrets out of pipeline logs
* review access to indexes, storage, and model deployments

Identity design should be part of the architecture, not an afterthought.

## 14. Monitoring and operations

A production AI workload needs monitoring from the beginning.

Useful signals include:

* request count
* latency
* failed requests
* HTTP 429 responses
* token usage
* model cost
* search query volume
* retrieval quality
* application exceptions
* user feedback
* content safety events

Monitoring should help answer simple operational questions:

* Is the app working?
* Is it slow?
* Is it being throttled?
* Is cost increasing?
* Are users getting useful answers?
* Which part of the flow is failing?

## 15. Evaluation

RAG quality must be measured.

Useful evaluation areas include:

| Metric           | Meaning                                      |
| ---------------- | -------------------------------------------- |
| Precision        | Are retrieved chunks relevant?               |
| Recall           | Are important chunks being missed?           |
| Groundedness     | Is the answer supported by source content?   |
| Relevance        | Does the answer address the user’s question? |
| Correctness      | Is the answer factually correct?             |
| Citation quality | Are references useful and traceable?         |

Evaluation should be repeated when documents, prompts, models, chunking, or indexing logic changes.

This prevents accidental regression.

## 16. Business ownership

A production AI solution needs business ownership, not only technical ownership.

The business should define:

* approved source documents
* acceptable use cases
* unacceptable use cases
* required disclaimers
* review process for wrong answers
* content update ownership
* quality expectations
* go-live criteria

Without this, the solution can become technically impressive but operationally unclear.

## 17. Recommended delivery approach

A practical delivery path:

1. Pick one clear business use case.
2. Select a small trusted document set.
3. Build a simple RAG flow.
4. Add source references.
5. Test with real business questions.
6. Improve chunking and metadata.
7. Add logging and cost tracking.
8. Add private networking where required.
9. Add API Management if routing, quota, or multiple clients are needed.
10. Automate deployment with infrastructure as code.
11. Add formal evaluation and regression testing.
12. Expand only after the first use case is working well.

## 18. Design principles

The architecture should follow these principles:

* keep the first use case narrow
* use trusted data sources
* make answers traceable
* design for failure and throttling
* secure private data
* monitor from day one
* avoid hardcoded secrets
* use managed identity where possible
* measure quality before scaling
* do not over-engineer the POC
* do not deploy a demo as production without hardening

## 19. Summary

An Azure AI POC proves the idea.

A production-ready Azure AI solution needs:

* secure application entry point
* controlled access to model deployments
* Azure AI Search for grounding
* private networking for sensitive workloads
* managed identity and RBAC
* monitoring and cost visibility
* RAG evaluation
* clear business ownership
* a delivery path that can grow without being rebuilt

The best approach is to build the POC with the production path in mind.

Do not overbuild the first version, but do not ignore the decisions that will matter later.
