# Code Structure, Integration Design, and Effort Comparison — LangChain vs Raw API

## Overview

Both applications solve the same problem — ingesting resumes, searching them semantically, and answering natural language questions. But the way each project is organised, and the way each integration is implemented, is completely different. This document looks at those differences across structure, integration design, and where each project actually spends its development effort.

---

## Architecture: How the Two Projects Are Structured

The most fundamental difference is not the AI-related code — it is the overall shape of the application.

### resumes-ai-rag — Layered MVC

```
resumes-ai-rag/src/
│
├── config/          ← Environment config, MongoDB connection, API clients, Multer setup
├── middleware/      ← Request ID injection, logging, centralised error handling
├── routes/          ← HTTP endpoints (thin layer — just parses request, calls controller)
├── controllers/     ← Orchestrates service calls, handles HTTP-level decisions
├── services/        ← All business logic: search, chat, ingestion, parsing, LLM, embeddings
├── repositories/    ← All database queries: inserts, BM25, vector search
├── types/           ← TypeScript type definitions
└── utils/           ← Text cleaning, regex helpers, profile heuristics, error utilities
```

This is a classic layered architecture where each layer has exactly one responsibility. Routes do not talk to repositories. Services do not know about HTTP. The separation is strict and deliberate. A developer coming into this project would immediately recognise the structure because it mirrors how most backend Express applications are organised.

### qa-bot-ts — Pipeline Architecture

```
qa-bot-ts/src/
│
├── config/          ← Environment config only
├── lib/             ← All LangChain primitives: chains, prompts, memory, models, embeddings, vector store
├── pipelines/
│   ├── ingestion/   ← Document loading → embedding → storage pipeline
│   └── retrieval/   ← Keyword search, vector search, hybrid merge, LLM reranker
├── scripts/         ← One-time setup utilities (vector index creation)
├── types/           ← TypeScript + Zod schemas for request/response validation
├── utils/           ← Document loading, field extraction helpers
└── server.ts        ← All HTTP route handlers (inline in one file)
```

This is a pipeline architecture. The application is thought of as a series of data transformation steps rather than layers of responsibility. LangChain encourages this thinking — you compose a chain out of steps, and each step does one thing. The tradeoff is visible in how the server entry point works: in the raw app, `server.ts` is a lean startup file that delegates almost everything to controllers, whereas in the LangChain app, all HTTP route handling lives inline in `server.ts` because LangChain's chain invocations are expressive enough to sit directly in the route handler without needing a separate controller layer.

---

## How the Integrations Are Implemented Differently

### The Embedding Layer

**Raw app:** Embedding is handled by a dedicated service class backed by a raw HTTP function. The service handles retry logic, timeout management via `AbortController`, exponential backoff between retries, and a deterministic fallback vector generator for when the Mistral API is unavailable. The HTTP response can come back in two different shapes depending on the API version, and both cases are handled manually. When something fails, a fake-but-deterministic vector is returned instead and a `fallbackUsed` flag is set, so the caller always knows whether the result is real or degraded.

**LangChain app:** The equivalent wraps LangChain's `MistralAIEmbeddings` class with a thin custom layer that validates the vector dimension, and a factory function that selects between Mistral and OpenAI based on configuration. There is no HTTP code, no retry logic, no fallback vector written by the developer. All of that is handled by the LangChain package internally. The developer-written layer is purely about configuration and validation — not about making the API call work.

The key difference is not the amount of code but what that code is doing. In the raw app, the developer owns the network layer. In the LangChain app, the developer does not.

### The LLM Call Layer

**Raw app:** Every call to an LLM is a manual `fetch` to the Groq REST API. The request body is manually constructed, the HTTP status is manually checked, the JSON response is manually parsed, and the content is extracted by navigating a nested object path. The application is locked to Groq — all LLM-dependent features assume Groq's specific response shape. Adding a second provider would mean writing another HTTP function and threading it through every feature that uses LLM calls.

**LangChain app:** The model factory reads a single environment variable and returns one of four possible LangChain model classes. Each class is just a constructor call — no HTTP code, no response parsing, no provider-specific logic. The entire rest of the application is typed against the abstract `BaseChatModel` interface, so every chain, reranker, and conversational system calls `.invoke()` identically regardless of whether the underlying model is Groq, Anthropic, OpenAI, or the custom Testleaf provider.

The custom Testleaf model is particularly worth noting. It was integrated into LangChain by extending `BaseChatModel` and implementing two methods — one that names the model type, and one that converts LangChain's message format into Testleaf's request format and converts the response back. Once that class exists, Testleaf is treated identically to any official provider everywhere in the codebase. In the raw app, doing the same would require writing a new HTTP client function and updating every service that makes LLM calls.

### The Vector Store Layer

**Raw app:** Vector search is implemented as a raw MongoDB aggregation pipeline. The developer writes the `$vectorSearch` stage, specifies the index name, the query vector field path, the number of candidates for approximate nearest-neighbour search, and the output limit. Score extraction uses MongoDB's `$meta: "vectorSearchScore"` operator. The database connection itself lives in a separate config file that handles connection pooling, retry with backoff, event listeners for connection state changes, and graceful shutdown hooks. This is entirely infrastructure code — no business logic, just keeping the database connection alive and healthy.

**LangChain app:** `MongoDBAtlasVectorSearch` from `@langchain/mongodb` wraps the entire aggregation pipeline. The developer passes it a collection, an index name, and an embeddings object, and LangChain takes care of converting a text query into a vector, constructing the aggregation, and returning scored results. There is no aggregation pipeline code written by the developer. There is also no separate database connection management file — the connection is established inside the vector store class, and LangChain's own connection lifecycle handles the rest.

### The Search Pipeline

Both applications implement the same search strategy: keyword search, vector search, merge the results, rerank using an LLM. How this is organised is very different.

**Raw app** puts the entire pipeline in a single, large service file. This file contains BM25 search, vector search, deduplication of candidates across both result sets, profile normalization (cleaning names, roles, and companies using heuristic validators), profile fingerprinting for deduplication using email, phone, filename, and content as identity signals, backfill logic when the reranker returns fewer results than requested, summarization, and end-to-end orchestration. All of this had to be hand-written because there was no framework providing any of these primitives. The deduplication logic alone is substantial — handling edge cases like the same candidate appearing under slightly different names or email formats across the keyword and vector result sets.

**LangChain app** spreads the same pipeline across five focused files:

```
pipeline.ts       ← orchestrates the stages, decides retrieval count before LLM filtering
hybridSearch.ts   ← weighted merge of keyword + vector results
keywordSearch.ts  ← regex-based search with field-weighted scoring
vectorSearch.ts   ← delegates to LangChain MongoDBAtlasVectorSearch, maps results
llmReranker.ts    ← prompt engineering + LLM invocation + Zod validation + fallback
```

Each file has one job. If you want to change how keyword scoring works, you change `keywordSearch.ts` and nothing else is affected. If you want to replace the reranker prompt, you change `llmReranker.ts`. This separation is not possible in the raw app's single-file design without significant refactoring.

The LLM reranker is worth looking at separately. It uses a highly detailed system prompt that instructs the model to distinguish between specific queries (which have explicit criteria like a city or required skill) and generic queries (which ask for the best resumes without criteria). For specific queries the model is told to be strict — a candidate's location must appear as explicit text in the resume, not be inferred from a phone number. For generic queries, all resumes match and ranking is by quality. This logic lives entirely in the prompt, not in application code, which is a meaningful difference — changing how reranking behaves means editing a prompt, not refactoring a function.

### Conversation Memory

This is where the two applications diverge most significantly — and it is not really a code structure difference, it is a capability difference.

**Raw app:** There is no server-side memory. Each request to the chat endpoint is treated as if it were the first message. The application accepts a history array in the request body but does not use it to answer the question — it runs a fresh search every time. Follow-up questions like "now filter those to candidates from service-based companies" are not supported.

**LangChain app:** Memory is a first-class concern. Each conversation gets a unique ID, and the server maintains a store of active conversations. When a user sends a message, the server retrieves that conversation's history, loads it into the prompt via a `{chat_history}` placeholder, gets a response, and saves both the question and answer back into memory. The next message in the same conversation sees everything that was said before.

Beyond basic memory, the application caches the search results from each query. When a follow-up message looks like a filtering request — phrases like "only", "filter those", "from those results" — the server skips the full search-embed-retrieve cycle and instead sends the already-retrieved candidate data to the LLM with the new filter criteria. This two-tier approach makes follow-up interactions fast and natural.

None of this exists in the raw app because it would require building all the memory primitives from scratch. LangChain's `BufferMemory` and `ChatMessageHistory` make it straightforward to add, which is why the LangChain app has it and the raw app does not.

### Prompt Management

**Raw app:** Prompts are plain strings built using array joins and template literals, written inline wherever the LLM is called. Each feature builds its own prompt string from scratch. There is no concept of a reusable template — if you want to change how prompts are structured across multiple features, you have to update them individually in each place they appear.

**LangChain app:** Prompts are `ChatPromptTemplate` objects with named placeholders for variables. System instructions and user messages are declared as separate roles, which maps directly to how modern chat APIs work. The application has four distinct prompt variants for the document QA feature — default, detailed, concise, and technical — all following a consistent ICEPOT structure (Instructions, Context, Examples, Persona, Output format, Tone). The caller picks a variant by name at request time. This kind of per-request configurability would require significant restructuring in the raw app.

### Chains

**Raw app:** There is no concept of a chain. The equivalent is sequential function calls in a service method — call one function, pass its output to the next, pass that output to the next. The pipeline is implicit, living only in the order of `await` calls. You cannot inspect it, replace a step, or cache a compiled version of it.

**LangChain app:** Chains are explicit, composable objects. A chain is assembled once — prompt template, language model, output parser — and the assembled object is reused across requests. The chain can be inspected as a data structure, individual steps can be swapped out without touching the others, and compiled chains are cached so the same configuration is only built once per server lifetime.

---

## Where Each Project Spends Its Development Effort

Looking at the two projects side by side at the level of concern:

```
CONCERN              resumes-ai-rag               qa-bot-ts
──────────────────────────────────────────────────────────────────
HTTP clients         config/clients.ts            Not written — LangChain
                     Manual fetch(), timeout,      handles it internally
                     response parsing per API

DB Connection        config/db.ts                 Not written — MongoDBAtlas
                     Pooling, retry, backoff,      VectorSearch handles
                     shutdown hooks, events        its own connection

Embeddings           services/EmbeddingService.ts lib/embeddings/mistralEmbeddings.ts
                     Retry, fallback, two          Dimension validation,
                     response shapes               factory for two providers

LLM Calls            services/LLMService.ts       lib/models/factory.ts
                     One provider (Groq),          Four providers, abstract
                     all features call Groq        interface throughout

Custom LLM           Not possible without          lib/models/testleafChat.ts
Provider             writing new HTTP client       Extends BaseChatModel,
                     + updating all callers        plugs in automatically

Vector Store         repositories/ResumeRepository lib/vectorstore/resumeVectorStore.ts
                     Raw $vectorSearch             Wraps MongoDBAtlasVectorSearch,
                     aggregation pipeline          no aggregation written

Search Pipeline      services/SearchService.ts    pipelines/retrieval/ (5 focused files)
                     All in one file: BM25,        Each file has one responsibility,
                     vector, dedup, normalize,     easily modified independently
                     backfill, rerank, summarize

Conversation         Does not exist               lib/memory/chatMemory.ts
Memory                                            lib/conversationalRAGChain.ts
                                                  lib/conversationalFilter.ts

Prompts              Inline string building        lib/prompts.ts
                     per feature, not shared       Four structured variants,
                                                  ChatPromptTemplate objects

Chains               Implicit (function calls)     lib/chain.ts
                                                  Explicit RunnableSequence

HTTP Layer           app.ts + server.ts            server.ts only
                     + routes/* + controllers/*    All inline, no controller layer

Middleware           middleware/*                  Minimal — not written
                     requestId, logging,           (LangChain error propagation
                     error handler                 used instead)
```

---

## What This Tells Us About the Two Approaches

The raw app spent effort in places that have nothing to do with the actual problem being solved — HTTP client infrastructure, database connection management, response parsing, provider-specific error handling. These are necessary but they are not the application. They are the plumbing underneath it.

The LangChain app did not write any of that plumbing. Instead, effort went into prompt engineering (the detailed reranker system prompt), conversational features that the raw app does not have at all, and the design of the pipeline architecture itself. These things are closer to what defines the behaviour of the application.

The raw app compensated by being more resilient — explicit fallbacks at every layer, test coverage, clear separation of concerns. The LangChain app compensated by being more capable — multiple providers, multi-turn conversations, follow-up filtering — in roughly the same total effort.

Neither approach is clearly better for all situations. The raw app would be better as a foundation for a production system that needs reliability and test coverage. The LangChain app would be better for rapidly exploring what an AI application can do, or for a team that wants to switch models and providers without restructuring the codebase. Understanding both approaches — and recognising what each one asks you to build vs what it gives you for free — is what makes it possible to make that choice deliberately.
