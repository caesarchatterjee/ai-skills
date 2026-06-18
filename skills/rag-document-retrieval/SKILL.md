# RAG Document Retrieval

## When to use
Use this skill when building pipelines that retrieve relevant document chunks from a vector store to augment LLM prompts with factual context.

## Quick Start

```python
import chromadb
from openai import OpenAI

client = OpenAI()
chroma = chromadb.PersistentClient(path="./chroma_db")
collection = chroma.get_or_create_collection("documents")

def embed(text: str) -> list[float]:
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=text,
    )
    return response.data[0].embedding

def retrieve(query: str, top_k: int = 5, threshold: float = 0.7) -> list[dict]:
    results = collection.query(
        query_embeddings=[embed(query)],
        n_results=top_k,
    )
    return [
        {"text": doc, "metadata": meta, "distance": dist}
        for doc, meta, dist in zip(
            results["documents"][0],
            results["metadatas"][0],
            results["distances"][0],
        )
        if dist <= threshold
    ]
```

## Document Ingestion

### Chunking strategy

Split documents into chunks of 500-1000 tokens with 100-token overlap. For insurance documents:

- Split on section headers (policy sections, endorsements, exclusions)
- Keep tables intact — do not split mid-table
- Preserve the document title and section path as metadata

```python
def chunk_document(text: str, chunk_size: int = 800, overlap: int = 100) -> list[str]:
    words = text.split()
    chunks = []
    start = 0
    while start < len(words):
        end = start + chunk_size
        chunks.append(" ".join(words[start:end]))
        start = end - overlap
    return chunks
```

### Metadata to store

Always store these fields alongside each chunk:
- `source`: filename or document ID
- `section`: section header path (e.g., "Policy > Exclusions > Water Damage")
- `page`: page number if available
- `doc_type`: policy, claim, procedure, FAQ

## Retrieval Prompt Pattern

```python
def build_rag_prompt(query: str, contexts: list[dict]) -> str:
    context_block = "\n\n---\n\n".join(
        f"Source: {c['metadata']['source']} (Section: {c['metadata'].get('section', 'N/A')})\n{c['text']}"
        for c in contexts
    )
    return f"""Answer the question based on the provided context. If the context
doesn't contain enough information, say so explicitly.

Context:
{context_block}

Question: {query}"""
```

## Gotchas

- ChromaDB uses L2 distance by default. Lower distance = more similar. Set `threshold` accordingly.
- Always deduplicate overlapping chunks in results before sending to the LLM.
- For multilingual documents (common at Zurich), use a multilingual embedding model.
- Reindex after any document update — stale embeddings cause silent retrieval failures.