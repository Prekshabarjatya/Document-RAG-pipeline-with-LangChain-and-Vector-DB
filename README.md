# Chat-with-PDF

A minimal Retrieval-Augmented Generation (RAG) reference that lets you ask questions about a single PDF and returns answers strictly grounded in the document with page-level citations.

This repository is a compact, readable implementation intended for developers who want a deterministic, inspectable RAG flow without hidden abstractions.

Table of contents
- [What this is](#what-this-is)
- [What this is not](#what-this-is-not)
- [Quick overview](#quick-overview)
- [Architecture](#architecture)
- [Requirements](#requirements)
- [Installation](#installation)
- [Usage](#usage)
  - [Ingest a PDF](#ingest-a-pdf)
  - [Query the document](#query-the-document)
- [Project structure](#project-structure)
- [Implementation details](#implementation-details)
- [Design decisions](#design-decisions)
- [Limitations](#limitations)
- [Extending the project](#extending-the-project)
- [Assumptions](#assumptions)
- [License](#license)

## What this is

A compact CLI-first example of a PDF-based question answering system using:
- semantic embeddings (OpenAI embeddings)
- a local vector store (Qdrant)
- a small system prompt to enforce document-grounded answers with citations

It demonstrates the RAG pattern: index a PDF, store chunk embeddings, then answer queries by retrieving relevant chunks and generating answers constrained to those chunks.

## What this is not

- Not a hosted SaaS or production service
- Not a web UI (this is CLI-first)
- Not designed for multi-document corpora or large-scale indexing
- Not production-hardened (no auth, persistence guarantees, or rate limiting)

## Quick overview

High-level flow:

```
PDF → chunking → embeddings → vector store
Query → similarity search → context assembly → LLM → answer + page citations
```

If you need background on the concepts used here:
- RAG overview: https://www.promptingguide.ai/techniques/rag
- Vector databases: https://qdrant.tech/documentation/concepts/
- Text embeddings: https://platform.openai.com/docs/guides/embeddings

## Architecture

Two clear phases:

1. Ingestion (offline / run once per document)
   - Extract text
   - Chunk with overlap
   - Encode with embeddings
   - Store vectors + metadata (page numbers) in Qdrant

2. Querying (online / per question)
   - Embed the query
   - Retrieve top-K nearest chunks
   - Build a strict context prompt that only allows answers from retrieved chunks
   - Generate the answer and display page citations

## Requirements

- Python 3.10+
- Docker (to run Qdrant locally)
- OpenAI API key (or other embedding/LLM provider if you adapt the code)
- pip (to install Python dependencies)

## Installation

1. Clone the repository

```bash
git clone https://github.com/Prekshabarjatya/Chat-with-PDF.git
cd Chat-with-PDF
```

2. Create and activate a virtual environment

```bash
python -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate
```

3. Install dependencies

```bash
pip install -r requirements.txt
```

4. Start Qdrant locally

```bash
docker run -p 6333:6333 qdrant/qdrant
```

5. Configure environment variables

Create a `.env` file in the project root:

```env
OPENAI_API_KEY=your_openai_api_key
QDRANT_URL=http://localhost:6333   # optional; default assumes local Qdrant
```

## Usage

There are two main scripts:

- `ingest.py` — index a PDF into Qdrant
- `chat.py` — ask questions against the indexed PDF

### Ingest a PDF

Place your PDF in the project directory (default example: `nodejs.pdf`) or pass a path accepted by the script.

Run:

```bash
python ingest.py
```

What this does:
- Loads the PDF and extracts text and page metadata
- Splits the text into overlapping chunks
- Computes embeddings with `text-embedding-3-large`
- Writes vectors and metadata (including page numbers) to a Qdrant collection (default: `my_collection`)

Run this step once per document or whenever the PDF changes.

### Query the document

Start the interactive CLI:

```bash
python chat.py
```

You will be prompted:

```
Ask Something:
```

Example:

```
Ask Something: What is the Node.js event loop?
```

Example output:

```
HappyBot: The Node.js event loop is responsible for handling asynchronous operations by...
(Page 12)
```

Behavior notes:
- Performs top-K similarity search with k = 4
- Assembles a strict context prompt and forces the LLM to answer only using retrieved content
- Prints page numbers for each supporting citation

## Project structure

```
.
├── index.py        # PDF ingestion and vector indexing
├── chat.py          # CLI-based question answering
├── nodejs.pdf       # Example PDF used in the demo
├── requirements.txt # Python requirements
├── .env.example     # Example environment variables
└── README.md
```

## Implementation details

- Chunk size: ~1000 characters
- Chunk overlap: ~40 characters
- Embedding model: `text-embedding-3-large` (used for both ingestion and query embedding to ensure compatibility)
- Vector DB: Qdrant (local, default collection `my_collection`)
- Retrieval: top-K nearest neighbors (no additional reranking beyond vector similarity)
- Prompting: strict system prompt that instructs the LLM to answer only from the provided context and to include page-level citations

Side effects:
- `index.py` will create or overwrite the Qdrant collection `my_collection` by default.

## Design decisions

- RAG (retrieval + generation) instead of fine-tuning: faster iteration, lower cost, and easier updates for private documents.
- Strict system prompt and source citation: minimizes hallucinations and forces traceability.
- Single embedding model: avoids embedding space mismatch between ingestion and querying.
- CLI-first: keeps implementation compact and easy to inspect for learning purposes.

## Limitations

- Single-PDF support only (no catalog or routing)
- No advanced metadata filtering or document-level routing
- No reranking beyond nearest neighbor search
- No streaming of responses
- No authentication, rate limiting, or long-term persistence guarantees
- Not optimized for production/performance at scale

This is intentionally a reference implementation and not a production-ready service.

## Extending the project

Common extensions:
- Add a FastAPI or web UI layer
- Support multiple PDFs and a document index/catalog
- Add hybrid search (BM25 + vectors)
- Add reranking and relevance tuning
- Support streaming LLM responses
- Add evaluation, logging, and observability

## Assumptions

- OpenAI APIs (or the configured embedding/LLM provider) are available
- Qdrant runs locally without authentication (or you adapt the code to a hosted Qdrant)
- PDF text extraction is reasonably accurate for semantic chunking

If these assumptions change you will need to adapt the code (e.g., change vector DB credentials, switch embedding model, or improve OCR).


## Contributing

Contributions are welcome. Please open an issue or a pull request if you'd like to:
- Report bugs or inaccuracies
- Add features (multiple documents, web UI, streaming)
- Improve tests, documentation, or code quality

If you add functionality that changes behavior (for example multi-document support), update the README usage sections and provide migration notes.
