# 🚀 Corrective RAG (CRAG)

> **An advanced Corrective Retrieval-Augmented Generation system that evaluates retrieval quality, corrects poor retrieval results, performs web-search fallback, and generates grounded answers using validated context.**

---

## 📌 Overview

**Corrective RAG (CRAG)** is an advanced RAG architecture designed to overcome one of the major limitations of traditional Retrieval-Augmented Generation systems: **poor-quality retrieval**.

In a traditional RAG pipeline, retrieved documents are usually passed directly to the LLM:

```text
User Query
    ↓
Retriever
    ↓
Top-K Documents
    ↓
LLM
    ↓
Answer
```

If the retrieved documents are irrelevant or insufficient, the LLM may generate an inaccurate or hallucinated response.

This project introduces a **Corrective Retrieval layer** that evaluates every retrieved document and determines whether the retrieved knowledge is:

* ✅ **CORRECT**
* ⚠️ **AMBIGUOUS**
* ❌ **INCORRECT**

Based on the evaluation, the system dynamically chooses the appropriate retrieval strategy.

---

# 🎯 Key Idea

The core CRAG workflow is:

```text
                         User Query
                             │
                             ▼
                      Document Retrieval
                             │
                             ▼
                    Document Relevance
                         Evaluation
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
           CORRECT       AMBIGUOUS      INCORRECT
              │              │              │
              │              │              ▼
              │              │        Query Rewriting
              │              │              │
              │              │              ▼
              │              │         Tavily Search
              │              │              │
              │              └───────┬──────┘
              │                      │
              └──────────────────────┤
                                     ▼
                              Context Refinement
                                     │
                                     ▼
                             Sentence Filtering
                                     │
                                     ▼
                            Refined Context
                                     │
                                     ▼
                                   LLM
                                     │
                                     ▼
                            Grounded Answer
```

---

# ✨ Features

## 🔎 1. Vector-Based Document Retrieval

The system loads PDF documents, splits them into chunks, generates embeddings, and stores them in a FAISS vector database.

```text
PDF Documents
     ↓
PyPDFLoader
     ↓
RecursiveCharacterTextSplitter
     ↓
HuggingFace Embeddings
     ↓
FAISS
     ↓
Similarity Retrieval
```

---

## 🧠 2. LLM-Based Document Relevance Scoring

Every retrieved document is individually evaluated by the LLM.

Each document receives a score between:

```text
0.0 ─────────────── 1.0
```

Where:

* `1.0` → Highly relevant / sufficient context
* `0.0` → Completely irrelevant

The evaluator also generates a short explanation for the score.

### Example

```text
Question:
Introduction to Language Modeling

Retrieved Chunk:
Language modeling is the task of predicting the probability
of a sequence of words...

Score:
0.91

Reason:
The document directly explains language modeling.
```

---

# 🎯 3. Three-Way Retrieval Classification

The system uses two thresholds:

```python
UPPER_TH = 0.7
LOWER_TH = 0.3
```

### ✅ CORRECT

If at least one document has:

```text
score > 0.7
```

the retrieval is considered **CORRECT**.

The system proceeds using the internal knowledge base.

---

### ⚠️ AMBIGUOUS

If:

```text
No document > 0.7
```

but at least one document is:

```text
> 0.3
```

the retrieval is considered **AMBIGUOUS**.

The system combines:

```text
Internal Documents
        +
Web Search Results
```

---

### ❌ INCORRECT

If all retrieved documents have:

```text
score < 0.3
```

the retrieval is considered **INCORRECT**.

The system triggers:

```text
Query Rewriting
      ↓
Tavily Web Search
      ↓
Web Documents
```

---

# 🔄 4. Intelligent Routing

The routing logic is implemented using **LangGraph conditional edges**.

```text
                 Document Evaluation
                         │
             ┌───────────┴───────────┐
             │                       │
          CORRECT               NOT CORRECT
             │                       │
             ▼                       ▼
          Refine              Rewrite Query
                                     │
                                     ▼
                                Web Search
                                     │
                                     ▼
                                   Refine
```

This allows the workflow to dynamically change its behavior based on retrieval quality.

---

# ✍️ 5. Query Rewriting

When the internal retrieval is insufficient, the original question is transformed into a concise web-search query.

The query rewriting model follows rules such as:

* Keep the query short
* Use relevant keywords
* Preserve the original intent
* Add recency constraints when required
* Never answer the question during rewriting

Example:

```text
Original:
What are the latest developments in large language models?

Rewritten:
latest large language model developments
```

---

# 🌐 6. Tavily Web Search Fallback

The project uses **Tavily Search** for external retrieval.

```text
Internal Retrieval
       ↓
Poor Retrieval
       ↓
Query Rewriting
       ↓
Tavily Web Search
       ↓
Web Documents
       ↓
Context Refinement
```

This allows the system to recover when the local knowledge base does not contain sufficient information.

---

# 🧹 7. Sentence-Level Context Refinement

Retrieved context is not blindly passed to the LLM.

First, the system:

1. Combines retrieved documents
2. Splits the context into sentences
3. Evaluates each sentence using an LLM
4. Keeps only sentences that directly help answer the question

```text
Retrieved Documents
        ↓
Sentence Decomposition
        ↓
Sentence-Level Relevance Filter
        ↓
Relevant Sentences
        ↓
Refined Context
```

This reduces unnecessary context and helps improve grounding.

---

# 🛡️ 8. Grounded Generation

The final LLM is instructed to answer **only from the refined context**.

If sufficient context is unavailable:

```text
"I don't know."
```

This provides an additional layer of protection against unsupported answers.

---

# 🏗️ Architecture

```text
                         ┌─────────────────┐
                         │   User Query    │
                         └────────┬────────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                         │    Retriever    │
                         └────────┬────────┘
                                  │
                                  ▼
                      ┌───────────────────────┐
                      │ Document Evaluation   │
                      │     LLM Grader        │
                      └───────────┬───────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
              ▼                   ▼                   ▼
         CORRECT              AMBIGUOUS           INCORRECT
              │                   │                   │
              │                   │                   ▼
              │                   │             Query Rewriter
              │                   │                   │
              │                   │                   ▼
              │                   │             Tavily Search
              │                   │                   │
              │                   └─────────┬─────────┘
              │                             │
              └─────────────────────────────┤
                                            ▼
                                  ┌─────────────────┐
                                  │ Context Refiner│
                                  └────────┬────────┘
                                           │
                                           ▼
                                  ┌─────────────────┐
                                  │ Sentence Filter │
                                  └────────┬────────┘
                                           │
                                           ▼
                                  ┌─────────────────┐
                                  │ Refined Context │
                                  └────────┬────────┘
                                           │
                                           ▼
                                  ┌─────────────────┐
                                  │   Groq LLM      │
                                  └────────┬────────┘
                                           │
                                           ▼
                                  ┌─────────────────┐
                                  │ Final Answer    │
                                  └─────────────────┘
```

---

# 🧩 LangGraph Workflow

The complete workflow is implemented using **LangGraph**.

```text
START
  │
  ▼
retrieve
  │
  ▼
eval_each_doc
  │
  ├────────────── CORRECT ───────────────┐
  │                                      │
  │                                      ▼
  │                                    refine
  │                                      │
  ├────── AMBIGUOUS ───► rewrite_query   │
  │                            │          │
  │                            ▼          │
  │                       web_search      │
  │                            │          │
  │                            ▼          │
  └────── INCORRECT ───►     refine ◄────┘
                               │
                               ▼
                            generate
                               │
                               ▼
                              END
```

---

# 📂 Project Structure

```text
CRAG/
│
├── documents/
│   ├── Aligarah_Movement.pdf
│   ├── course_note_CMU_NLP_Lecture1_5.pdf
│   └── Full_Stack.pdf
│
├── src/
│   ├── ingestion/
│   ├── retrieval/
│   ├── evaluation/
│   ├── refinement/
│   ├── web_search/
│   └── generation/
│
├── notebooks/
│
├── .env
├── .env.example
├── .gitignore
├── requirements.txt
├── Dockerfile
└── README.md
```

> The exact folder structure can be adjusted according to the final production implementation.

---

# 🛠️ Tech Stack

### Programming Language

* Python

### LLM

* Groq
* Llama 3.3 70B Versatile

### RAG

* Corrective RAG
* Vector Similarity Search
* Context Refinement

### Frameworks

* LangChain
* LangGraph

### Embeddings

* Hugging Face
* `sentence-transformers/all-MiniLM-L6-v2`

### Vector Database

* FAISS

### Document Processing

* PyPDFLoader
* RecursiveCharacterTextSplitter

### Web Search

* Tavily

### Configuration

* python-dotenv

---

# 📦 Installation

## 1. Clone the Repository

```bash
git clone https://github.com/YOUR_USERNAME/CRAG.git

cd CRAG
```

---

## 2. Create Virtual Environment

### Windows

```bash
python -m venv venv

venv\Scripts\activate
```

### Linux / macOS

```bash
python3 -m venv venv

source venv/bin/activate
```

---

## 3. Install Dependencies

```bash
pip install -r requirements.txt
```

---

# 🔐 Environment Variables

Create a `.env` file:

```env
GROQ_API_KEY=your_groq_api_key
TAVILY_API_KEY=your_tavily_api_key
```

Never commit your `.env` file to GitHub.

Add it to `.gitignore`:

```gitignore
.env
venv/
__pycache__/
*.pyc
```

---

# ▶️ Run the Project

After configuring the environment variables:

```bash
python main.py
```

The system will:

```text
1. Load PDF documents
2. Split documents into chunks
3. Generate embeddings
4. Build FAISS vector store
5. Retrieve relevant documents
6. Evaluate document relevance
7. Determine retrieval verdict
8. Rewrite query if required
9. Search Tavily when necessary
10. Refine the context
11. Generate the final grounded answer
```

---

# 🧪 Example

### Input

```text
Introduction to Language Modeling
```

### Internal Retrieval

```text
Top-K Documents
      ↓
LLM Relevance Evaluation
```

Possible result:

```text
VERDICT: CORRECT

REASON:
At least one retrieved chunk scored > 0.7.
```

Then:

```text
Relevant Documents
       ↓
Sentence Filtering
       ↓
Refined Context
       ↓
Groq LLM
       ↓
Final Answer
```

---

# 📊 Retrieval Decision Logic

| Condition                     | Verdict      | Action             |
| ----------------------------- | ------------ | ------------------ |
| Any score > 0.7               | ✅ CORRECT    | Internal documents |
| No score > 0.7 but some > 0.3 | ⚠️ AMBIGUOUS | Internal + Web     |
| All scores < 0.3              | ❌ INCORRECT  | Web retrieval      |

---

# 💡 Why This Is Better Than Basic RAG

A basic RAG system assumes:

> **Retrieved context = useful context**

CRAG does not make this assumption.

Instead:

```text
Retrieve
   ↓
Evaluate
   ↓
Correct if necessary
   ↓
Refine
   ↓
Generate
```

This creates a more adaptive retrieval pipeline.

---

# 🔬 Key Components

### Document Evaluator

Uses structured output:

```python
class DocEvalScore(BaseModel):
    score: float
    reason: str
```

### Sentence Filter

Uses structured output:

```python
class KeepOrDrop(BaseModel):
    keep: bool
```

### Web Query Generator

Uses structured output:

```python
class WebQuery(BaseModel):
    query: str
```

Structured outputs make the LLM-based decision layers easier to control programmatically.

---

# 🚀 Future Production Improvements

The current implementation provides the core CRAG workflow. For a fully production-grade deployment, the following can be added:

### Retrieval

* [ ] Hybrid Search
* [ ] BM25 + Vector Search
* [ ] Cross-Encoder Reranking
* [ ] Metadata Filtering
* [ ] Parent Document Retrieval
* [ ] Multi-Query Retrieval

### Evaluation

* [ ] RAGAS evaluation
* [ ] Context Precision
* [ ] Context Recall
* [ ] Faithfulness
* [ ] Answer Relevance
* [ ] Retrieval Hit Rate
* [ ] MRR

### Security

* [ ] Prompt Injection Detection
* [ ] PII Detection & Redaction
* [ ] Input Validation
* [ ] Output Validation
* [ ] Rate Limiting
* [ ] Authentication
* [ ] Secrets Management

### Production

* [ ] FastAPI
* [ ] Docker
* [ ] Docker Compose
* [ ] CI/CD
* [ ] Health Checks
* [ ] Structured Logging
* [ ] Error Monitoring
* [ ] LangSmith Tracing
* [ ] Metrics & Observability

### Advanced RAG

* [ ] Self-RAG
* [ ] Adaptive RAG
* [ ] Agentic RAG
* [ ] Knowledge Graph RAG
* [ ] Multimodal RAG

---

# 📈 Production Architecture Roadmap

The project can be extended from the current prototype into a production architecture:

```text
                    Client
                      │
                      ▼
                 API Gateway
                      │
                      ▼
                Authentication
                      │
                      ▼
                  FastAPI
                      │
                      ▼
              CRAG Orchestrator
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
     Vector Search            Web Search
          │                       │
          └───────────┬───────────┘
                      ▼
                Reranker
                      │
                      ▼
               Context Filter
                      │
                      ▼
                    LLM
                      │
                      ▼
              Output Guardrails
                      │
                      ▼
                   Client
```

---

# 🎯 Use Cases

CRAG can be applied to:

* 📚 Enterprise Knowledge Assistants
* 📄 Document Question Answering
* 🎓 Educational AI Assistants
* 🏢 Internal Company Knowledge Systems
* 🔬 Research Assistants
* 🛠️ Technical Documentation Assistants
* 📖 Knowledge Retrieval Systems

---

# 👨‍💻 Author

**Shayan Ahmed**

AI/ML Engineer | Generative AI | RAG | Agentic AI | LLMOps

---

# ⭐ Support

If you find this project useful, consider giving the repository a ⭐.

---

## 📜 License

This project is licensed under the MIT License.
