# 🧠 Semantic Vector Search API (FastAPI + Pinecone + EmbeddingGemma)

This repository contains a production-ready, serverless Retrieval-Augmented Generation (RAG) backend that demonstrates the paradigm shift from traditional keyword matching (lexical search) to conceptual matching (semantic search). 

Built using FastAPI, Google’s embeddinggemma-300m model, and a Pinecone vector database, this system transforms raw, unstructured data into a mathematically indexed knowledge base capable of answering deep architectural and operational questions.

---

## 🛠️ System Architecture & Workflow

The codebase is engineered around two distinct lifecycle pipelines:

### 1. The Ingestion Pipeline (`ingest.py`)
This pipeline acts as the ETL (Extract, Transform, Load) engine for your text data:
* Extraction: Discovers and parses raw source text documents from local storage.
* Transformation (Recursive Chunking): Divides documents systematically using a cascading hierarchy of linguistic separators (Paragraphs -> Sentences -> Words) to stay beneath model context token boundaries while preserving integrity.
* Vectorization: Passes raw text chunks through the embedding model to convert semantic meaning into continuous numerical representations (768-dimensional vectors).
* Loading: Streams vector coordinates bundled with their original text payloads into the cloud vector index.

### 2. The Real-Time Search API (`main.py`)
A highly responsive FastAPI gateway serving production query traffic:
* Initialization: Pre-loads the heavy machine learning model into global memory exactly once at server boot time to prevent endpoint latency overhead.
* Routing: Exposes a single GET interface accepting human query strings.
* Real-Time Embedding: Computes the query vector coordinates on the fly.
* Spatial Matching: Queries Pinecone to run coordinate comparisons and returns the top matching records based on geometric proximity.

---

## 📊 Core Concepts & Operational Engineering

Developing this pipeline highlights several foundational concepts governing vector search and retrieval infrastructure:

### Lexical vs. Semantic Search
Traditional indexing engines execute lexical queries, searching exclusively for matching character sequences (Ctrl+F). If a document uses synonyms, a lexical scanner fails. 

Semantic search analyzes the holistic conceptual framework of the document. The embedding model reads text and maps it onto a high-dimensional mathematical hypersphere. Sentences expressing similar core ideas orient toward identical directions in vector space regardless of the exact vocabulary utilized.

### Understanding Cosine Similarity Scores
In high-dimensional vector spaces, similarity is calculated using the geometric angle between the user query vector and document vectors. The closer the vectors align, the closer the score approaches 1.0.

| Experience Scenario | Expected Score Range | Architectural Explanation |
| :--- | :--- | :--- |
| Exact Concept / Identical String | 0.95 - 1.00 | Query text and document text are perfectly parallel in vector space. |
| Strong Semantic Match (Problem to Solution) | 0.55 - 0.75 | The database record contains the query's core idea but adds context (e.g., matching a traffic spike query against a chunk describing auto-scaling mechanics). The added content shifts the vector's spatial center of mass. |
| Irrelevant / Out-of-Domain Data | Below 0.30 | Vectors point in orthogonal or completely unrelated directions. |

### Structural Text Processing: Recursive Chunking
Machine learning models possess finite context windows. Large text manuals must be segmented without slicing words or splitting single sentences in half. 

Recursive chunking processes files via a top-down waterfall strategy:
1. Try dividing exclusively by structural paragraph blocks (\n\n).
2. If an individual block exceeds the maximum character threshold, split it dynamically by sentences (\n).
3. If an isolated sentence remains too large, partition it safely across whitespace boundaries.

An intentionally configured chunk overlap (e.g., 20 characters) functions as structural glue. It duplicates marginal text between back-to-back chunks, retaining linguistic context (such as pronoun references) across boundaries.

---

## 💡 Key Technical Lessons (Production Gotchas)

* Global Model Warmup: Instantiating deep learning models inside an active HTTP route handler forces the server to reload weights on every request, causing severe response lag. Storing the initialized model object globally and warming it up via an explicit application startup event keeps search endpoints instantaneous.
* Gated Weights Authentication: Leading open-weights models are gated on Hugging Face. Running default repository initialization blocks results in immediate 401 Unauthorized exceptions. Code must securely map local environment access tokens to authenticate programmatically with the remote model registry.
* Automated Environment Anchoring: Virtualized running containers often execute scripts from shifting root folders, causing default configuration lookups to fail silently. Forcing the environment loader to actively hunt down the absolute filesystem path ensures configuration strings load reliably across all environments.
* SDK Vector Specification: Modern vector index client libraries enforce strict, keyword-driven typing. Standard positional arguments are discarded; developers must explicitly declare target parameters (specifying your vector array and result thresholds directly) to prevent runtime rejection.
* Container Disk Preservation: Large machine learning frameworks store heavy binaries and build components by default during system installations. Utilizing specialized flags during package installations to clear cache arrays prevents storage saturation inside cloud development workspaces.

---

## ⚙️ Local Deployment & Configuration

### 1. Environment Keys
Create a `.env` file in your root workspace. Ensure there are no spaces flanking the assignment operators:
```env
PINECONE_API_KEY="your_secret_pinecone_key"
PINECONE_INDEX_NAME="gemma-lecture-index"
EMBEDDING_MODEL_NAME="google/embeddinggemma-300m"
HF_TOKEN="your_huggingface_read_token"