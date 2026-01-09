# Analysis: RAG Retrieval & Vector Search

## Executive Summary
This document outlines a critical finding in the local RAG implementation (`faiss_client.py`). 

**The Finding**: The local retrieval system is **not performing semantic vector search**. It is simulating retrieval using a "Lucky Dip" strategy. 

While the system appears to retrieve documents, it relies entirely on **Metadata Filtering** (checking Product IDs) applied to a broad, semi-random sample of documents. It does not utilize the semantic meaning of the user's query (e.g., "specifications," "benefits") to rank results.

---

## 1. How Vector Search *Should* Work
In a functioning RAG system, the "Semantic Match" relies on a shared mathematical space:

1.  **Ingestion**: A specific AI Model (e.g., OpenAI `text-embedding-3`) converts Document A into a vector (e.g., `[0.1, 0.9, ...]`).
2.  **Query**: The **SAME** AI Model converts the User's Query into a vector (e.g., `[0.1, 0.8, ...]`).
3.  **Math**: The system calculates the angle (Cosine Similarity) between the two vectors.
4.  **Result**: If the angles are similar, the document is retrieved because it "means" the same thing as the query.

## 2. How `faiss_client.py` Actually Works
The current local implementation breaks the "Shared Space" rule.

### The Configuration
```python
# faiss_client.py
vector_store = FAISS.from_embeddings(
    text_embeddings=zip(texts, real_vectors), # Real vectors from disk
    embedding=FakeEmbeddings(size=dim),       # <--- THE PROBLEM
    metadatas=metadatas,
)
```

1.  **The Database**: Loaded from `.npy`. These contain **Real Vectors** (computed previously by a real model).
2.  **The Query**: At runtime, `FakeEmbeddings` is used. This generates a **Fake/Random Vector** for the user's question.
3.  **The Comparison**: Comparing a *Real Vector* to a *Fake Vector* results in mathematical noise. The similarity scores are meaningless.

### The "Lucky Dip" Mechanism
Since the vector search returns garbage results, the code employs a fallback strategy to find relevant data:

1.  **Oversampling (The Dip)**: 
    It requests `k * 15` documents (e.g., `16 * 15 = 240` documents). Because the query vector is fake, FSISS returns 240 random documents from the index.
    
2.  **Metadata Filtering (The Check)**:
    It executes a Python loop over these 240 documents:
    ```python
    for doc in raw_results:
        if product_id in doc.metadata['doc_summary']:
            keep(doc)
    ```

3.  **The Result**:
    *   **Standard Search**: "Find me documents about *cooling efficiency* for C9300." -> Returns documents about cooling.
    *   **Current Search**: "Give me 240 random documents. I will keep the ones labeled 'C9300'."

## 3. Implications & Risks

### A. Scalability (The "Needle in a Haystack" Problem)
*   **Small Dataset**: If your total database has 300 chunks, fetching 240 random ones covers 80% of the data. The "Lucky Dip" will likely work.
*   **Large Dataset**: If your database grows to 10,000 chunks, fetching 240 random ones covers 2.4% of the data. You have a **97.6% chance of missing the correct document**, even if it exists.

### B. Loss of Semantic Nuance
The system ignores the *intent* of the query.
*   **User Query**: "What are the *cabling requirements* for C9200?"
*   **System Action**: Returns *any* chunk tagged C9200.
*   **Result**: It might return chunks about "Power Supply" or "Dimensions" first, simply because they appeared in the random sample, while missing the "Cabling" chunk.

### C. False Evaluation
Developers might believe the RAG pipeline is working. However, they are testing the **Metadata Filter**, not the **Retrieval Quality**. Moving this code to production (with a real database) without swapping the Embedding Class will result in immediate failure.

## 4. Recommendations -> to be discussed with the team

### Immediate Fix (Local)
To make the local search act like the real search:
1.  Identify which model created the `embeddings.npy` file (e.g., `text-embedding-3-small`, `all-MiniLM-L6-v2`).
2.  Instantiate that exact model in `faiss_client.py` instead of `FakeEmbeddings`.
    ```python
    # Example if using OpenAI
    from langchain_openai import OpenAIEmbeddings
    embedding = OpenAIEmbeddings(model="text-embedding-3-small")
    ```

### Production Check
Verify `weaviate_client.py`. If Weaviate is configured to perform vector searches on the server-side, it is likely working correctly (assuming the Weaviate server has the embedder configured). The issue is isolated to the local FAISS fallback.
