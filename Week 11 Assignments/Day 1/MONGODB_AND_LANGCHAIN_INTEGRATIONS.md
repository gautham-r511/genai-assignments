# MongoDB and LangChain Integrations in the QA Bot

## Introduction

The `qa-bot-ts` application uses MongoDB Atlas as its database and LangChain as the framework for orchestrating AI operations. These two integrations are deeply connected — MongoDB is not just used as a regular database here but as a vector store, and LangChain provides the abstractions that make working with embeddings, language models, and conversation memory feel natural. This document explains how each integration works and why the design choices were made the way they were.

---

## MongoDB as a Vector Database

### What a Vector Index Is

Traditional databases store data and let you search it by matching exact values or ranges. Vector databases work differently — they store numerical representations of content (called embeddings) and let you search by meaning rather than exact terms. MongoDB Atlas has a feature called Atlas Vector Search that adds this capability on top of a regular MongoDB collection.

Before any vector search can happen, a special index must be created on the collection. This is not a regular MongoDB index. It is a vector search index that specifies which field contains the embeddings, how many dimensions those embeddings have, and what mathematical method to use when comparing them. In our application, the embedding field stores 1024-dimensional vectors produced by Mistral's `mistral-embed` model, and the similarity function used is cosine similarity. Cosine similarity measures the angle between two vectors, which makes it well-suited for comparing text embeddings because similar meanings tend to point in similar directions in the vector space.

One important constraint is that this index cannot be created programmatically through the MongoDB driver. It has to be created through the MongoDB Atlas web interface or command line tools. The application includes a script that prints the exact index definition to use, but it cannot create the index itself — the developer has to do that step manually in Atlas before the search features will work.

### How Documents Are Stored

When a resume is ingested into the system, LangChain stores it as a MongoDB document with a specific structure. The full text content of the resume goes into a field called `text`. The embedding vector — the 1024-dimensional array of numbers representing the meaning of that text — goes into a field called `embedding`. Any additional information like the candidate's email address, phone number, and the original file name is stored in a nested `metadata` field.

This structure is not arbitrary. It is what LangChain's MongoDB integration expects by default. The field names can be configured, but the concept is the same: the content that gets embedded lives in one field, the vector itself lives in another, and metadata lives in a third. When we later search the collection, LangChain knows exactly where to look for each piece.

### Ingesting Resumes

The ingestion process is the part that converts raw PDF or Word document files into searchable entries in the database. LangChain plays a role at every step of this process.

First, LangChain provides document loaders that can read PDF and DOCX files and extract their text content. The PDF loader can be configured to keep all pages as a single document (which is what we want for ingestion, so the full resume is embedded as a coherent unit) or to split by page (which is useful when you want the LLM to reference specific pages in its answers). This flexibility is built into the loader.

Once the text is extracted, it is packaged into a LangChain `Document` object — a simple container that holds the text content and any associated metadata. These Document objects are then passed to the vector store's `addDocuments` method. At this point, LangChain automatically calls the embedding model to convert the text into a vector, and then stores both the text and the vector in MongoDB. The developer does not need to explicitly call the embedding service or write the database insert — LangChain orchestrates all of that.

Because the Mistral embedding API has rate limits, the ingestion process runs in batches rather than trying to embed all resumes at once. Batches are processed one at a time, sequentially, so the application does not accidentally overwhelm the API with too many concurrent requests.

### Keyword Search

Vector search is powerful for finding candidates based on meaning, but sometimes you want to find candidates based on exact terms — a specific skill name, a company, or an email address. For this, the application uses MongoDB's standard query capabilities directly, without going through LangChain.

The keyword search works by splitting the user's query into individual words and using those words as a regular expression pattern. This pattern is then matched against multiple fields simultaneously — the full text content, the candidate's name, email, skills, role, and company. When results come back, they are scored based on how many times the keywords appear in each field. Skills and job titles are given a higher weight than general text content, because a keyword match in those fields is more likely to indicate a genuinely relevant candidate. The scores are normalized and the top results are returned.

### Vector Search

For vector search, the application delegates entirely to LangChain's `MongoDBAtlasVectorSearch` class. When a search query comes in, LangChain first converts the query text into a vector using the same embedding model that was used during ingestion. It then runs this vector against MongoDB's `$vectorSearch` pipeline, which compares the query vector against all stored embedding vectors and returns the most similar documents along with a similarity score.

The key insight is that this similarity score is not measuring whether the words match — it is measuring whether the meanings are similar. A query for "automation testing engineer" might surface a candidate whose resume never uses that exact phrase but talks extensively about Selenium, pytest, and CI/CD pipelines, because those concepts are semantically close in the embedding space.

### Hybrid Search

The most powerful search mode combines both approaches. Keyword search is good at finding exact matches but misses semantic similarity. Vector search is good at finding semantic matches but might miss someone who uses very precise terminology. By running both searches in parallel and combining their results, the application gets the benefits of both.

The merging step assigns a weight to each search type — by default, vector search contributes 70% to the final score and keyword search contributes 30%. A candidate who appears in both result sets gets contributions from both weighted paths added together, which means candidates that are both semantically relevant and use the right keywords end up ranked highest. The weights are configurable through environment variables, so they can be tuned without changing any code.

---

## LangChain Integrations

### Embeddings

LangChain has an abstract concept of an "Embeddings" object. Any class that implements the right interface — providing methods to embed a single query and to embed multiple documents — can be used anywhere in the system that needs embeddings. The application uses Mistral's embedding model as its default, with OpenAI available as an alternative.

A custom wrapper class was built around LangChain's built-in Mistral embedding class to add dimension validation. Every time an embedding is generated, the wrapper checks that the returned vector is exactly 1024 dimensions long. This might seem like unnecessary checking, but it catches configuration problems early — if someone accidentally points the system at a different model that produces different-sized vectors, the error is caught immediately rather than causing confusing failures later when the vectors don't match what the database index expects.

The choice of embedding provider can be changed through configuration without touching any application code. This is because everything that uses embeddings — the vector store, the search pipeline — receives the embedding object through dependency injection rather than creating it themselves.

### Document Loaders

LangChain's `@langchain/community` package includes loaders for many file types. The application uses the PDF loader and the DOCX loader. Both loaders follow the same interface — you point them at a file, call `load()`, and get back an array of `Document` objects containing the extracted text and some basic metadata about the file.

The loaders are loaded on demand the first time they are needed and then cached, so they don't need to be re-imported on every request. This is a minor optimization but it matters when the server is handling many concurrent requests.

### Chat Models and the Model Factory

LangChain defines an abstract `BaseChatModel` interface that all chat model implementations follow. The application supports four providers — Groq, Anthropic, OpenAI, and a custom internal provider called Testleaf — but all of them are used identically through this common interface. A factory function reads the `MODEL_PROVIDER` environment variable and returns the appropriate model class. Everything else in the application — the chains, the reranker, the conversational RAG system — receives a `BaseChatModel` and calls `.invoke()` on it, with no knowledge of which provider is underneath.

This design means switching from Groq to Anthropic, or from Anthropic to OpenAI, is a one-line change in an environment file. It also means adding a new provider only requires changes in the factory function, not throughout the codebase.

### The Custom Testleaf Model

One of the more interesting aspects of the LangChain integration is the custom model for Testleaf — an internal LLM provider with its own API. Rather than hardcoding calls to Testleaf throughout the application, we extended LangChain's `BaseChatModel` class. This means implementing two methods: one that declares what kind of model this is (used for logging and tracing), and one that accepts a list of LangChain message objects, converts them to the format Testleaf expects, makes the HTTP request, extracts the response content, and returns it in LangChain's standard output format.

Once that class exists, Testleaf becomes a fully equal participant in the LangChain ecosystem. All the chain code, the reranker, and the conversational memory system work with Testleaf exactly as they do with any officially supported provider. The application doesn't need to know or care that Testleaf has an unusual nested response structure — that detail is hidden inside the custom model class.

### Prompt Templates

Instead of building prompt strings through string concatenation, LangChain uses structured `ChatPromptTemplate` objects. A template declares the shape of the prompt — which parts are system instructions, which part is the user's message, and where in the conversation history should be injected — with named placeholders for variables. When the template is used at runtime, the placeholders are filled in with actual values.

This matters for a few reasons. First, it keeps prompts organized and reusable. The application defines four different prompt styles for the document QA feature (default, detailed, concise, and technical), all following the same ICEPOT structure, and the caller just picks which one they want by name. Second, it handles multi-turn conversations naturally — there is a special placeholder type for injecting an entire list of previous messages, which is how conversation history flows from memory into the prompt without any manual string building.

The LLM reranker uses a particularly detailed system prompt that instructs the model to distinguish between specific queries (which have criteria like a city or a required skill) and generic queries (which just ask for the best resumes without specific criteria). For specific queries, the model is told to be strict — a candidate's location must be explicitly written in their resume, not inferred from a phone number. For generic queries, the model is told to rank all candidates by overall quality. All of this logic lives in the prompt template, not in application code.

### Chains and LCEL

The core idea behind LangChain's chain system is that each step in an AI pipeline takes some input and produces some output, and steps can be connected together so that one step's output automatically becomes the next step's input. LangChain calls this the LangChain Expression Language, or LCEL.

The simplest chain in the application has three steps. First, the prompt template formats the inputs — a document and a question — into a properly structured list of messages. Second, the language model receives those messages and generates a response. Third, an output parser extracts the plain text string from the model's response object. These three steps are chained together, and the whole thing can be invoked with a single call, passing in the document and question.

For the conversational RAG endpoint, the chain is slightly more complex because it also needs to inject search results and conversation history into the prompt. But the pattern is the same — a prompt template, a model, and an output parser, connected as a pipeline.

One practical optimization the application makes is caching compiled chains. Building a chain is not expensive, but doing it on every request adds unnecessary overhead. The application stores built chains in a map keyed by the model type and prompt style, so if the same combination is requested again, it reuses the existing chain object.

### Conversation Memory

LangChain provides memory primitives that store the history of a conversation and make it available to be injected into future prompts. The application uses `BufferMemory` backed by an in-memory `ChatMessageHistory`. Each time a user sends a message and receives a response, both the question and the answer are saved to memory. When the next message arrives in the same conversation, the history is loaded and inserted into the prompt, so the model has full context about what was said before.

The application wraps LangChain's memory in its own manager class that adds two additional features. The first is message trimming — to prevent conversations from growing indefinitely and using up too much context window space, the manager keeps only the most recent N messages and discards older ones. The second is search result caching — after each search, the retrieved candidates are stored alongside the conversation memory. When the next message looks like a filtering request, the application can ask the LLM to filter those cached candidates rather than going back to MongoDB for a fresh search. This makes follow-up interactions much faster and feels more natural to the user.

All active conversations are tracked in a singleton store that maps conversation IDs to memory managers. This is why the application can support endpoints for retrieving conversation history and deleting a specific conversation — the server keeps these objects alive in memory for the duration of the server process.

### LLM Reranking

After the initial hybrid search retrieves candidates, an LLM reranker evaluates each candidate against the query and assigns a semantic relevance score. This is the step that turns raw search results into intelligently filtered, properly ranked recommendations.

The reranker works by packaging all the candidate resumes into a structured prompt context, along with the user's original query, and asking the LLM to evaluate each candidate and return a JSON response. The JSON response is then validated against a schema before being used — this is important because LLMs can produce slightly malformed JSON, and catching validation errors early (and falling back to the original results if necessary) prevents the application from crashing on bad output.

The schema validation also handles a common inconsistency in LLM output: sometimes the model returns a list of skills as a comma-separated string instead of a proper JSON array. The schema includes a transformation that automatically splits the string into an array when this happens, so the rest of the application always sees a consistent data structure regardless of minor variations in how the model formats its response.

### Conversational Filter

When a user sends a follow-up message that looks like a refinement request rather than a new search — phrases like "only show me those from service-based companies" or "filter to candidates with 10 or more years of experience" — the application uses a second LLM step called the conversational filter.

The key difference between the filter and the full reranker is what the LLM is given to work with. The reranker sees raw resume content. The filter sees the structured information that the reranker already extracted — the candidate's current company, location, skills, experience, and key highlights — in a compact format. This means the filter doesn't need to re-read entire resumes or make any database calls. It just applies the new criteria to the already-structured data, which is much faster and cheaper.

The filter is triggered by detecting certain keywords in the user's message. If those keywords are present and there are cached results from a previous search, the filter is used instead of the full pipeline. This two-tier approach — full RAG search for new queries, lightweight LLM filter for follow-up refinements — is what makes the conversational experience feel smooth and responsive.

---

## Putting It Together

What makes this application interesting is how seamlessly MongoDB and LangChain work together. MongoDB provides durable, scalable storage that supports both traditional queries and vector similarity search. LangChain provides abstractions for every AI operation — reading files, generating embeddings, calling models, managing conversation state, and composing steps into pipelines. Neither tool is trying to do the other's job.

The result is an application where adding a new LLM provider is a few lines of configuration, where a conversation can span multiple messages with the server maintaining context, where search results can be progressively refined through natural language without re-querying the database, and where the entire search pipeline from query to ranked results is expressed as a clear sequence of composable steps. All of that would be significantly harder to build without these two integrations.
