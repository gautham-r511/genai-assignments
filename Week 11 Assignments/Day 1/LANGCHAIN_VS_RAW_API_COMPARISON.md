# Comparing Applications Built With and Without LangChain

## Overview

As part of this week's work we built two applications that solve the same problem — searching resumes and answering questions about them using AI. The first one (`resumes-ai-rag`) was built entirely from scratch without any AI framework. The second (`qa-bot-ts`) was built using LangChain. Both applications use MongoDB to store resumes, use embeddings to enable semantic search, and use an LLM to rerank results and answer questions. The core logic is identical. What changes is how each layer is implemented, and that difference turns out to be quite significant.

---

## Architecture and Project Structure

The most immediately visible difference is how the projects are organized. The non-LangChain app (`resumes-ai-rag`) follows a traditional layered MVC structure — routes call controllers, controllers call services, services call repositories, and repositories talk to the database. This is the kind of structure you would see in any backend project. Each layer has a clear responsibility and the layers don't bleed into each other.

The LangChain app (`qa-bot-ts`) is organized around pipelines and the concept of a "chain." Instead of controllers and repositories, you have ingestion pipelines, retrieval pipelines, and chain modules. The structure reflects how LangChain thinks about AI applications — as a series of composable steps rather than a layered service architecture. This is neither better nor worse, but it does mean the project feels quite different to navigate if you are used to traditional backend patterns.

The LangChain app also has no test suite, while the non-LangChain app has a full Jest test suite covering search, reranking, ingestion, chat, and parsing. This was not a consequence of using LangChain, but it does reflect the difference in maturity between a prototype and a more production-oriented build.

---

## Making LLM Calls

This is where the two approaches diverge the most noticeably. In the non-LangChain app, every call to an LLM is a raw HTTP request. We wrote a function that manually constructs the request body, sets the authorization header, sends the request using the global `fetch` API, checks if the response was successful, parses the JSON, and then navigates to the correct field in the response to extract the generated text. If the response shape from the API ever changes, we have to update that parsing logic ourselves.

The LangChain app does none of this. Instead, you instantiate a class like `ChatGroq` or `ChatAnthropic`, pass it your API key and model name, and then call `.invoke()` on it. LangChain handles the HTTP request, authentication, response parsing, and error handling internally. From the application's perspective, calling any LLM feels exactly the same — you pass in a prompt, you get back text.

One important practical effect of this is that the non-LangChain app is tightly coupled to Groq. If we wanted to switch to Anthropic or OpenAI, we would need to write new HTTP client functions and update every part of the code that currently calls Groq. In the LangChain app, switching providers is just a matter of changing an environment variable. The rest of the application doesn't need to change at all because everything is typed against LangChain's abstract `BaseChatModel` interface — it doesn't matter what model is underneath.

---

## Embeddings

Embeddings are vectors — long arrays of numbers that represent the meaning of a piece of text. Both applications use Mistral's `mistral-embed` model to generate these vectors, but the way they do it is very different.

In the non-LangChain app, generating an embedding means writing another raw HTTP request function. We send the text to Mistral's embedding endpoint, receive a JSON response, and manually extract the embedding array from it. The Mistral API can return the embedding in two different shapes depending on the version of the endpoint, so we had to handle both cases. We also implemented retry logic with exponential backoff in case the API is temporarily unavailable. And because embedding APIs can go down, we wrote a fallback that generates a deterministic pseudo-random vector from the text — so the application keeps working even when Mistral is unreachable, at the cost of search quality.

In the LangChain app, we simply create a `MistralAIEmbeddings` object, pass it an API key, and LangChain handles everything else. There is no HTTP code, no response parsing, no retry logic written by us. We did wrap LangChain's class in a thin custom layer that validates the vector dimension (checking that it is exactly 1024 dimensions as expected), but that was a minor addition on top of what LangChain already provides. The non-LangChain app spent roughly 100 lines of code on embedding infrastructure. The LangChain app achieves the same functionality in about 15 lines.

---

## Vector Storage and Search

Both applications store resume embeddings in MongoDB Atlas and use its vector search feature to find semantically similar resumes. But how they interact with MongoDB is completely different.

In the non-LangChain app, vector search requires writing a MongoDB aggregation pipeline manually. This involves using MongoDB's `$vectorSearch` stage, specifying the index name, the query vector, the field path where embeddings are stored, the number of candidates to consider, and the limit. You also have to handle the projection yourself to extract the similarity score. It is not particularly difficult, but it is low-level work that requires understanding MongoDB's internals.

In the LangChain app, all of that is handled by the `MongoDBAtlasVectorSearch` class from the `@langchain/mongodb` package. You give it a collection, an index name, and an embeddings object, and it takes care of everything. Searching becomes a single method call — you ask for the top K similar documents and you get back results with scores, without ever having to write an aggregation pipeline.

For keyword search, both apps directly query MongoDB — this is one area where both apps work at the same level, because LangChain does not provide a built-in keyword search abstraction over MongoDB.

---

## Prompts

A prompt is the instruction you give to the LLM. Both applications have prompts, but they are managed very differently.

In the non-LangChain app, prompts are just strings. They are constructed by joining arrays of strings with newline characters, with variable values interpolated directly into the string. Every feature — reranking, summarization, chat responses — builds its prompt string from scratch in its own part of the code. There is no concept of a reusable prompt template. If you want to change how prompts are structured across the application, you have to hunt down every place where a prompt string is built and update them individually.

The LangChain app uses `ChatPromptTemplate`, which is a structured object that represents a prompt with named placeholders. Instead of building a string, you declare the structure of your prompt once and then fill in the variables when you actually need to make a call. Prompts are also properly separated into system messages (instructions and persona) and human messages (the actual query), which maps directly to how modern chat APIs work. The application ships four prompt variants — default, detailed, concise, and technical — all following a consistent structure called ICEPOT (Instructions, Context, Examples, Persona, Output format, Tone). Being able to select a prompt style per request, without changing any code, is something the non-LangChain app cannot do without significant refactoring.

---

## Chains

A chain in LangChain is a sequence of steps where the output of one step becomes the input of the next. In the context of a RAG application, a typical chain is: format the prompt → call the model → extract the text from the response.

In the non-LangChain app, there is no explicit concept of a chain. The equivalent logic is just sequential function calls in a service method — call one function, take the result, pass it to the next function. This works, but the pipeline is implicit. There is no object you can inspect or modify to understand how data flows through the system.

In the LangChain app, chains are first-class objects using something called LCEL (LangChain Expression Language). You compose steps with a `.pipe()` method, and the result is an actual runnable object that represents the full pipeline. This makes the intent of the code very clear. It also means you can cache the compiled chain for reuse, which the application does — if the same model and prompt combination is requested again, it returns the already-built chain rather than constructing a new one.

---

## Conversation Memory

This is probably the biggest practical difference between the two applications.

The non-LangChain app has no server-side memory. Each request to the chat endpoint is treated as if it were the first message. The application accepts a history array in the request body, but it doesn't actually use it to answer the question — it just runs a fresh search every time. This means follow-up questions like "now filter those results by experience" are not possible. The concept of a conversation does not exist on the server side.

The LangChain app has proper conversation memory using `BufferMemory` and `ChatMessageHistory`. Each conversation gets a unique ID, and the server maintains a store of active conversations in memory. When a user sends a message, the application retrieves the history for that conversation ID, injects it into the prompt alongside the search results, gets a response from the LLM, and then saves both the question and the answer back into memory. The next time the same conversation ID is used, the LLM can see everything that was said before.

Beyond basic memory, the LangChain app also caches the search results from each query. When the server detects that a follow-up message is a filtering request (using keywords like "only", "filter", "from those"), it skips the full search-embed-retrieve cycle entirely and instead asks the LLM to filter the already-retrieved candidates. This is significantly more efficient and also makes the conversation feel natural — the user can progressively refine results without the system having to start from scratch each time.

---

## Error Handling

The two applications take very different approaches to what happens when something goes wrong.

The non-LangChain app is extremely defensive. At every layer — embedding generation, LLM reranking, summarization — there is an explicit fallback. If the embedding API fails, the app generates a deterministic fake vector instead. If the LLM reranker returns invalid JSON, the app falls back to scoring candidates by their raw search scores. If summarization fails, a pre-built text snippet is used. Each service method returns a `fallbackUsed` flag so the API consumer can tell whether any part of the response is degraded. The application is designed to keep running and return something useful even when external services are unavailable.

The LangChain app is much simpler in this regard. If an external call fails, an exception is thrown and propagates to the HTTP handler, which returns a 400 error. The only explicit fallback is in the LLM reranker — if the response can't be parsed, the original unfiltered results are returned. There is no fallback for embeddings or chain execution. The application assumes external services will be available, which is a reasonable assumption for a prototype but would need to be hardened for production use.

---

## Search Pipeline Design

Both applications implement the same search strategy: retrieve candidates using both keyword matching and vector similarity, merge the results, and then use an LLM to rerank and filter them. The difference is in how this pipeline is organized.

The non-LangChain app puts all of this logic in a single large `SearchService` file. This file handles BM25 search, vector search, deduplication of candidates across the two result sets, profile normalization, reranking, summarization, and the final merging and ordering of results. It is thorough and handles many edge cases — for example, it recognizes that the same candidate might appear in both the keyword and vector results, merges their scores intelligently, and uses multiple signals (email, phone number, filename, profile fingerprint) to deduplicate them. All of this had to be built from scratch.

The LangChain app separates these concerns into individual classes — one for keyword search, one for vector search, one for hybrid merging, one for LLM reranking — each with a single responsibility. The pipeline orchestrator combines them. Because LangChain's model abstraction is used for the reranker, it doesn't matter which LLM provider is configured — the reranker works the same regardless.

---

## Custom Model Integration

One thing worth highlighting is that the LangChain app integrates a custom in-house LLM called Testleaf by extending LangChain's `BaseChatModel` class. This demonstrates an important advantage of using a framework — you write the integration once, following LangChain's interface contract, and from that point the custom model becomes a first-class provider. All the chain code, reranker code, and conversational RAG code works with Testleaf exactly as it does with Groq or Anthropic, without any changes.

Doing the equivalent in the non-LangChain app would require writing a new HTTP client function and then threading it through every part of the codebase that makes LLM calls.

---

## Overall Observations

Building both applications side by side made a few things clear.

The raw API approach gives you complete visibility and control. You know exactly what HTTP request is being made, what the response looks like, and what happens if something goes wrong. This makes the application more predictable and easier to test — you can mock a `fetch` call and write unit tests for individual services. The non-LangChain app has a proper test suite precisely because of this. The downside is that you write a lot of plumbing code that isn't specific to your problem — HTTP handling, retry logic, response parsing, fallback strategies.

The LangChain approach removes most of that plumbing. You spend far less time on infrastructure and far more time on the actual behavior of your application. The model switching, memory management, and chain composition that would take days to build from scratch are available out of the box. The downside is that some of what the framework does is not immediately visible, and if something goes wrong inside a LangChain abstraction it can be harder to debug than a raw HTTP call.

For rapid prototyping and learning, LangChain is clearly the better starting point. For a production system where you need fine-grained control, explicit fallbacks, and comprehensive testing, it makes sense to either build raw or to selectively wrap LangChain abstractions with your own resilience logic. The two repos we built this week represent exactly that spectrum.
