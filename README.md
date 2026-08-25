# Multimodal-Hybrid-RAG


## The Problem
A property SME was managing a growing volume of leases, property reports, planning and compliance documents, contractor records, images and external information across disconnected files and formats. Staff had to search and cross-check these sources manually when responding to operational and client queries. This made research slow and made it difficult to maintain a clear evidence trail behind decisions. They needed a practical knowledge system that could search property information quickly, that was sourced, and remain maintainable as new documents were added.

## The Solution
I developed a multimodal hybrid Retrieval-Augmented Generation (RAG) system that allows users to ask questions about an organisation’s document collection. The system retrieves relevant source material, combines semantic and keyword search, reranks the results, and generates an answer. If the system decides it cannot provide a sufficient answer, it will utilise three parallel web-agents to gather the relevant information externally. 

The system supports multiple model providers, allowing an SME to balance cost, performance, privacy and vendor dependency.

![Image](https://github.com/Emerald-Bit/Multimodal-Hybrid-RAG/blob/main/RAG_Chart.png)

## Core Tech Stack
- AI and orchestration: LangChain, LangGraph, OpenAI, Google Gemini, Anthropic Claude, and Ollama.
- Backend: Python, FastAPI, Pydantic, asynchronous request handling.
- Retrieval: Chroma vector search, Elasticsearch BM25 keyword search, hybrid retrieval, FlashRank reranking.
- Embeddings: OpenAI text-embedding-3-small and local BAAI/bge-base-en-v1.5 embeddings, with disk-backed LangChain embedding caching.
- Memory and persistence: SQLite checkpointing and Neon PostgreSQL-backed LangGraph storage.
- Data pipeline: Multimodal Python ingestion for text, Markdown, PDFs, images and audio; Florence-2 image/PDF captioning, Whisper transcription, SHA-256 file-change detection, JSONL normalisation and Chroma/Elasticsearch indexing.
- Security and resilience: Regex prompt filtering, input/output validation, retry controls and fallback models.
- Evaluation: Evidently AI and MLflow LLM-as-Judge, synthetic golden dataset generation from corpus, retrieval quality testing (Precision@K, Recall@K and NDCG@K), chunking experiments, latency measurements and automated defensive red-team testing.

 
# Key Features and Engineering Justifications
## Table of Contents
* [Key Features and Engineering Justifications](#key-features-and-engineering-justifications)
  * [Hybrid Knowledge Retrieval](#hybrid-knowledge-retrieval)
  * [Reranking for Context Quality](#reranking-for-context-quality)
  * [Corrective RAG with Parallel Web Research:](#corrective-rag-with-parallel-web-research)
  * [Data Ingestion](#data-ingestion)
  * [Embedding Caching](#embedding-caching)
  * [Semantic Answer Caching](#semantic-answer-caching)
* [Other Noticeable Features and Justifications](#other-noticeable-features-and-justifications)
  * [Configurable LLM Providers](#configurable-llm-providers)
  * [Controlled Generation](#controlled-generation)
  * [Document and Chunk Metadata](#document-and-chunk-metadata)
  * [Semantic Answer Caching](#semantic-answer-caching-1)
  * [Multi-Stage Request Validation](#multi-stage-request-validation)
  * [Separation of Retrieval, Generation and Validation](#separation-of-retrieval-generation-and-validation)
  * [Metadata-Aware Document Storage](#metadata-aware-document-storage)
  * [Hosted and Local Model Options](#hosted-and-local-model-options)
  * [Fallback Model Handling](#fallback-model-handling)
  * [Short-Term and Long-Term Memory Separation](#short-term-and-long-term-memory-separation)
  * [Thread-Based Workflow Continuity](#thread-based-workflow-continuity)
  * [Operational Guardrails](#operational-guardrails)
  * [Graceful Failure Boundaries](#graceful-failure-boundaries)
* [Evaluations](#evaluations)
  * [Evaluation and Testing](#evaluation-and-testing)
  * [Synthetic Evaluation Dataset](#synthetic-evaluation-dataset)
  * [LLM-Based RAG Evaluation](#llm-based-rag-evaluation)
  * [Edge-Case Regression Testing](#edge-case-regression-testing)
  * [Automated Red-Team Agent](#automated-red-team-agent)
  * [Structured Red-Team Findings](#structured-red-team-findings)
  * [Fixed Testing and Exploratory Red Teaming](#fixed-testing-and-exploratory-red-teaming)
  * [Evaluation-Driven Development](#evaluation-driven-development)
  * [Prompt Engineering](#prompt-engineering)


## Hybrid Knowledge Retrieval
The system combines Elasticsearch BM25 search with Chroma vector retrieval. BM25 handles exact terms, names and identifiers, while embeddings handle semantic similarity. This combination is more suitable for SME documents than relying on keyword or vector search alone.

## Reranking for Context Quality
Initial results are reranked with FlashRank before being passed to the language model. This reduces irrelevant context, improves the quality of the evidence supplied to the model and helps control prompt size and token usage. FlashRank in particular was chosen for its speed.

## Corrective RAG with Parallel Web Research
The system implements a Corrective RAG workflow that evaluates the quality of retrieved internal evidence before generating an answer. After hybrid retrieval and FlashRank reranking, the system checks both the highest document relevance score and the average relevance of the top three results. If the internal evidence falls below configured confidence thresholds, the workflow automatically escalates to external web research rather than generating a confident answer from weak context.
A planning agent converts the original question into exactly three targeted research queries. These are then executed concurrently through three parallel web-search agent calls using asynchronous execution. Running the searches in parallel reduces the additional latency introduced by corrective retrieval while allowing the system to investigate the question from multiple search angles.
The resulting web sources are converted into the same document format used by the internal RAG pipeline, duplicate URLs are removed, and the external evidence is combined with the internal retrieved documents before final answer generation.
For a property SME, this provides a controlled way to answer questions involving recent planning changes, regulations, market developments or other information that may not yet exist in the company’s internal knowledge base.

## Data Ingestion
The ingestion pipeline accepts heterogeneous SME data including text and Markdown files, PDFs, images and audio. Text-heavy PDF pages are extracted directly, while visually complex pages and images can be processed with Florence-2 and audio can be transcribed with Whisper before the content is normalised into a common JSONL format.
A SHA-256 content-hash system identifies files that have already been processed. File hashes are compared against a persistent hash cache so unchanged files can be skipped rather than repeatedly running document parsing, image captioning, transcription and embedding operations.
This supports efficient incremental knowledge-base updates without rebuilding the entire corpus whenever new or changed property documents are added.

## Embedding Caching
Embedding generation is wrapped with LangChain CacheBackedEmbeddings and a local file store. Previously calculated embeddings can therefore be reused instead of repeatedly calling a hosted embedding API or recomputing vectors with the local model.
Cache namespaces include the embedding model and version so vectors produced by different configurations remain separate. SHA-256 cache keys provide stable lookup identifiers, while query-embedding caching can be enabled or disabled through configuration.
This reduces repeated compute, API expenditure and indexing time when the SME refreshes a largely unchanged document collection.

## Semantic Answer Caching
The application maintains a separate Chroma-based semantic answer cache for previously answered questions. New queries are compared with cached questions using cosine similarity and, when the configured threshold is met, an existing answer can be returned without repeating retrieval and language-model generation.


# Other Noticeable Features and Justifications

## Configurable LLM Providers
The application supports OpenAI, Google, Anthropic, and Ollama models through a common model-selection layer. This gives an SME flexibility to:
- Change providers without redesigning the API.
- Use hosted models for quality and scale.
- Use local Ollama models where privacy or operating cost is more important.
- Maintain a fallback model if the primary provider is unavailable.


## Controlled Generation

The generation layer uses structured prompts and retrieved documents as context. The API also includes input and output validation stages through LangGraph, helping prevent unsupported requests and reducing the likelihood of ungrounded responses.

## Document and Chunk Metadata

Documents retain metadata such as source, document type, page number, and chunk identifiers. This supports traceability, debugging, filtering, and future source-link generation.


## Multi-Stage Request Validation

The API does not send every request directly to the RAG pipeline. An input-validation node checks whether the request is suitable for processing before retrieval begins. Invalid or unsupported requests are routed to a blocked-response node, reducing unnecessary model calls and helping protect the system from misuse.

## Separation of Retrieval, Generation and Validation

The application uses layered controls rather than sending every request directly to the retrieval pipeline. A lightweight regex pre-filter rejects obvious suspicious inputs before graph execution, after which a LangGraph input-validation node determines whether the request should proceed or be routed to a blocked response.
Retrieval and answer generation are separated from the final output-validation stage. This makes failures easier to isolate and allows retrieval, generation, input safety and output behaviour to be tested or modified independently.

## Metadata-Aware Document Storage

Documents and chunks retain metadata such as source, document type, page number, chunk identifiers and retrieval scores. This supports filtering, traceability, debugging and the later presentation of source references to users.


## Hosted and Local Model Options

Ollama provides a local model route for situations where an SME wants to reduce external API dependency or keep selected workloads on-premises. Hosted providers remain available when higher capability or easier scaling is required.

## Fallback Model Handling

The validation and agent layers support fallback-model middleware. If the primary model fails, the application can attempt to continue using a configured backup model, improving resilience against transient provider failures.

Cache entries include the model and prompt version to reduce the risk of serving responses generated under incompatible configurations. Entries can also expire through a configurable TTL, while SHA-256 identifiers provide deterministic cache keys.
This reduces response latency and language-model expenditure for repeated or closely related SME queries.


## Short-Term and Long-Term Memory Separation

SQLite checkpointing stores thread-level workflow state, while Neon PostgreSQL storage provides a durable cross-session store. Keeping these responsibilities separate prevents temporary execution state from being confused with information that should persist across conversations.

## Thread-Based Workflow Continuity

Requests use a thread_id so conversational state and checkpointed workflow data can be associated with the correct session. This allows the API to preserve chat history and maintain continuity across multiple requests.

## Operational Guardrails

The application includes request-length validation, prompt filtering, rate limiting, retry controls and LangGraph recursion limits. These measures reduce the risk of malformed requests, prompt abuse, runaway workflows and uncontrolled API spending.

## Graceful Failure Boundaries

The service distinguishes between a cache hit, successful retrieval, web-assisted retrieval and failed or unavailable infrastructure. Internal control errors can be logged for operators without exposing raw middleware or provider errors as the final SME-facing response.


# Evaluations

![Image](https://github.com/Emerald-Bit/Multimodal-Hybrid-RAG/blob/main/evals.png)

## Evaluation and Testing

The project uses a multi-layer evaluation approach covering retrieval quality, answer quality, latency, behavioural edge cases and adversarial safety testing. This provides a more complete view of RAG performance than relying on a single accuracy metric.

## Synthetic Evaluation Dataset
A synthetic evaluation dataset was generated using Evidently's RAG dataset-generation tooling. The benchmark contains realistic questions generated from the source documents, with each test case containing a question, target answer, expected context and expected source identifiers.
Persisting the expected source IDs directly in the evaluation dataset provides stable ground truth for retrieval testing. This avoids having to recreate expected documents from article titles during later evaluation runs and makes results easier to reproduce and compare.
The dataset acted as a repeatable regression benchmark for testing changes to retrieval, chunking, prompts and model configuration. 


## LLM-Based RAG Evaluation
Evidently is used to evaluate generated responses in addition to the deterministic retrieval metrics.

The evaluation workflow measures:
- Context Relevance: Whether the retrieved evidence is relevant to the user's question.
- Faithfulness: Whether the generated response is supported by the retrieved context.
- Context Quality: Whether the retrieved evidence provides sufficient useful information for answering the question.
- Correctness: Whether the generated answer is consistent with the expected answer in the evaluation dataset.

This separates retrieval failures from generation failures. A poor answer can therefore be investigated to determine whether the system retrieved the wrong evidence or whether the language model failed to use correct evidence appropriately.
Evaluation runs are stored in a local Evidently workspace, allowing results to be inspected and compared without requiring Evidently's hosted cloud service.

## Edge-Case Regression Testing
A separate fixed regression suite tests behaviours that are not adequately represented by retrieval metrics alone.
The current test set covers scenarios including:
- Ambiguous questions
- Incomplete input
- Multi-part questions
- Foreign-language requests
- Brand-safety scenarios

Each test defines the expected application behaviour, sends the input through the RAG application and uses an LLM judge to classify the result as PASS or FAIL. Response latency is recorded alongside the judgement.
The initial tests successfully identified behavioural weaknesses rather than producing artificially perfect results. Ambiguous and incomplete queries passed their expected-behaviour checks, while the initial brand-safety, foreign-language and multi-part tests exposed areas requiring further refinement.
These tests are designed primarily as regression cases. Once behaviour is corrected, the same inputs can be rerun after changes to models, prompts or application logic to identify whether previously resolved problems have returned.

## Automated Red-Team Agent
A dedicated defensive red-team agent provides exploratory adversarial testing in addition to the fixed regression suite.

The red-team agent generates synthetic attacks across eight major categories:
- Prompt injection
- Jailbreaks
- Harmful content
- Data security
- Illegal activities
- Forbidden or regulated domains
- Manipulation attempts
- Sensitive scenarios

These categories contain more specific attack types such as system-prompt extraction, instruction override, roleplay bypasses, context leakage, cross-tenant leakage, fraud attempts, unauthorised business commitments and vulnerable-user scenarios.


The red-team wrapper also records retrieved document evidence alongside the draft RAG answer and final validated answer. This allows failures to be investigated as either retrieval-level or generation-level problems rather than judging only the final response.

## Structured Red-Team Findings
Red-team results are returned using a structured Pydantic schema containing:
- Category and subcategory
- Severity
- Generated attack query
- Expected safe behaviour
- Actual behaviour
- PASS/FAIL result
- Supporting evidence
- Retrieved-document evidence
- Recommended remediation
- Findings are persisted to JSONL after each test. This creates an auditable record of adversarial testing that can be reviewed over time and used to determine whether security changes have resolved previously discovered weaknesses.

## Fixed Testing and Exploratory Red Teaming
The two safety testing approaches serve different purposes.
Fixed edge case tests provide repeatable regression testing against known scenarios, while the red-team agent generates new adversarial variations to discover failures that may not already exist in the test suite.
Together, these provide both repeatability and broader exploratory coverage.

## Evaluation-Driven Development
Evaluation outputs are retained at both summary and individual-query level. This allows changes to chunk size, overlap, retrieval configuration, ranking, prompts and models to be compared quantitatively rather than selected solely through manual inspection.
The evaluation process therefore acts as a pre-release quality check for changes to the RAG system, helping identify whether an improvement in one area introduces regressions elsewhere.
The existing numerical chunking and latency results were produced against the earlier retrieval baseline. As the application has since expanded to include hybrid retrieval, reranking, caching and conditional web research, the same evaluation workflow can be rerun against the newer architecture before attributing the earlier benchmark figures directly to the current production oriented design.

## Prompt Engineering
Prompt engineering is treated as a measurable and version-controlled part of the RAG system rather than a one-off set of instructions. Prompts are stored and loaded through MLflow with explicit versioning, allowing alternative prompt designs to be tested without changing the production application code. During evaluation, a selected prompt can be injected into a dedicated RAG evaluation endpoint while the semantic answer cache is bypassed, ensuring that each test reflects the behaviour of the prompt currently under review. Prompt variants are then scored against expected answers and facts for correctness, alongside behavioural criteria such as conciseness and professional tone. This provides a repeatable way to compare prompt changes, track improvements between versions and reduce the risk of deploying prompt modifications based only on subjective judgement.


