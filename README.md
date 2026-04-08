# RAG Mongo Demo - Enterprise RAG System for Test Cases & User Stories

A production-grade **Retrieval-Augmented Generation (RAG)** system built with Node.js, React, MongoDB Atlas Vector Search, Mistral AI, and Groq AI. Enables intelligent search, summarization, and analysis of test cases and user stories with semantic understanding.

---

## 📋 Project Overview

**RAG Mongo Demo** is an enterprise-ready system that combines:
- **Vector Search** (semantic similarity) and **BM25 Search** (keyword matching) for comprehensive document retrieval
- **AI-Powered Reranking** using Groq LLM for intelligent result ordering
- **Batch Embeddings Generation** using Mistral AI for scalable processing
- **Query Preprocessing** with abbreviation expansion and synonym mapping
- **Results Deduplication** and **AI Summarization** for better insights
- **Job Tracking** for long-running operations

**Use Cases:**
- QA teams searching test cases by functionality or risk level
- Product managers querying user stories by acceptance criteria
- Healthcare compliance teams analyzing regulatory requirements
- Finding related test cases or user stories for impact analysis

---

## 🎯 Core Functionalities

### 1. **Data Management**
| Functionality | Endpoint | Method | Purpose |
|---|---|---|---|
| Upload & Convert Excel | `/api/upload-excel` | POST | Convert Excel sheets to JSON format (test cases or user stories) |
| List Available Files | `/api/files` | GET | Retrieve data files ready for embedding |
| Fetch Metadata | `/api/metadata/distinct` | GET | Get distinct values for filter options (modules, priorities, risks) |
| Environment Config | `/api/env` | GET/POST | Read/update environment variables |

**Key File**: [server/index.js](server/index.js#L230-L280) - Upload handling with data conversion

---

### 2. **Embedding Generation**
| Functionality | Endpoint | Method | Purpose |
|---|---|---|---|
| Create Embeddings | `/api/create-embeddings` | POST | Generate embeddings from selected files (individual processing) |
| Batch Embeddings | `/api/create-embeddings-batch` | POST | Fast batch embedding using Mistral AI (optimized) |
| Get Job Status | `/api/jobs/:jobId` | GET | Track embedding creation progress |
| Active Jobs | `/api/jobs/active` | GET | List currently running embedding jobs |

**Key Implementation**:
- **Mistral AI** for embedding generation at $0.10/M tokens
- **Individual**: [server/index.js](server/index.js#L950-L1050) - Single file processing
- **Batch**: [server/index.js](server/index.js#L825-L920) - Optimized batch processing
- **Utility**: [src/scripts/utilities/mistralEmbedding.js](src/scripts/utilities/mistralEmbedding.js#L25-L80) - Core embedding functions

**Embedding Metadata Stored:**
```javascript
{
  embedding: [...vector array of 1024 dimensions],
  embeddingMetadata: {
    model: 'mistral-embed',
    provider: 'mistral-ai',
    cost: 0.000001,
    tokens: 250,
    apiSource: 'testleaf'
  }
}
```

---

### 3. **Search APIs** (Multiple Strategies)

#### **A) Vector Search** (Semantic Similarity)
```
POST /api/search
Body: { query: string, limit: number, filters: object }
```
- Uses MongoDB Atlas `$vectorSearch` stage
- Mistral AI embeddings for query representation
- Returns top K semantically similar documents
- **Key Code**: [server/index.js](server/index.js#L1790-L1880) - Vector search pipeline

#### **B) BM25 Search** (Keyword/Lexical)
```
POST /api/search/bm25
Body: { query: string, limit: number, filters: object, fields: string[] }
```
- Full-text search across specified fields
- Fuzzy matching with `maxEdits: 1`, `prefixLength: 2`
- Fast for exact phrase matching
- **Key Code**: [server/index.js](server/index.js#L1920-L2000+)

#### **C) Hybrid Search** (Vector + BM25 Fusion)
- Combines results from both search methods
- Score fusion algorithm for intelligent ranking
- **Frontend Component**: [client/src/components/search/HybridSearch.js](client/src/components/search/HybridSearch.js)

#### **D) Groq-Only Reranking** (LLM-Powered)
```
POST /api/search/rerank
Body: { query: string, limit: number, filters: object }
```
- Fetches 50 candidates from BM25 + Vector search
- Sends to Groq LLM for semantic reranking
- Returns top K semantically relevant results
- **Key Code**: [server/index.js](server/index.js#L1620-L1720)
- **Reranking Prompt**: [src/scripts/utilities/groqClient.js](src/scripts/utilities/groqClient.js#L40-L100)

---

### 4. **Query Processing**
| Endpoint | Method | Purpose |
|---|---|---|
| `/api/search/preprocess` | POST | Normalize query, expand abbreviations/synonyms |
| `/api/search/analyze` | POST | Analyze query without applying changes |

**Preprocessing Steps**:
1. **Normalization**: Remove extra spaces, lowercase
2. **Abbreviation Expansion**: "QA" → "Quality Assurance"
3. **Synonym Expansion**: "bug" → "bug", "defect", "issue"
4. **Smart Expansion**: Preserve test case IDs like "TC-001"

**Key Implementation**: [src/scripts/query-preprocessing/queryPreprocessor.js](src/scripts/query-preprocessing/queryPreprocessor.js)

---

### 5. **Post-Search Intelligence**
| Endpoint | Method | Purpose |
|---|---|---|
| `/api/search/deduplicate` | POST | Remove similar results based on semantic similarity |
| `/api/search/summarize` | POST | Generate AI summary of search results |
| `/api/test-prompt` | POST | Test custom prompts with Groq AI |

**Summarization Models** (Using Groq):
- **Concise**: 2-3 sentence summary
- **Detailed**: Comprehensive analysis with metrics and insights
- **Custom**: User-defined summarization style

**Key Code**: [src/scripts/utilities/groqClient.js](src/scripts/utilities/groqClient.js#L120-L220)

---

## 🏗️ Architecture

### System Design

```
┌─────────────────────────────────────────────────────────┐
│                  FRONTEND (React + MUI)                 │
│  - QuerySearch, BM25Search, HybridSearch, Reranking     │
│  - ConvertToJson, EmbeddingsStore                       │
│  - QueryPreprocessing, SummarizationDedup               │
└──────────────────────┬──────────────────────────────────┘
                       │ HTTP (Axios)
                       ▼
┌─────────────────────────────────────────────────────────┐
│               BACKEND (Node.js + Express)               │
│  - REST API endpoints (port 3001)                       │
│  - Job tracking and monitoring                          │
│  - Embedding generation orchestration                   │
│  - Search logic and reranking                           │
└──────────────────────┬──────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   ┌─────────┐   ┌──────────┐   ┌──────────┐
   │MongoDB  │   │Mistral   │   │Groq AI   │
   │Vector   │   │Embeddings│   │Reranking │
   │Search   │   │          │   │Summarize │
   └─────────┘   └──────────┘   └──────────┘
```

### Data Flow

```yaml
Excel Upload
  ↓
JSON Conversion (Excel → JSON)
  ↓
Batch Embedding Generation (Mistral AI)
  ↓
MongoDB Storage with Vector Index
  ↓
Search Query Preprocessing
  ↓
Vector/BM25/Hybrid Search
  ↓
Groq LLM Reranking & Summarization
  ↓
Results Display (React Frontend)
```

---

## 📊 Database Schema

### Test Cases Collection
```javascript
{
  _id: ObjectId,
  id: "TC-001",
  module: "User Authentication",
  title: "Login with valid credentials",
  description: "Verify user can login with correct email and password",
  steps: ["Step 1...", "Step 2..."],
  expectedResults: "User logged in successfully",
  automationManual: "Automated",
  priority: "High",
  risk: "Medium",
  embedding: [1024-dimensional vector],  // Mistral embedding
  createdAt: ISODate,
  sourceFile: "converted-1774790892723.json",
  embeddingMetadata: {...}
}
```

### User Stories Collection (Alternative)
```javascript
{
  _id: ObjectId,
  key: "US-001",
  summary: "As admin, I want to manage user roles",
  description: "System should allow admin to create/update roles",
  status: { name: "In Progress", category: "In Progress" },
  priority: { name: "High", id: "1" },
  acceptanceCriteria: "...",
  embedding: [1024-dimensional vector],
  storyPoints: 5,
  components: ["Auth", "Admin Panel"],
  labels: ["backend", "security"]
}
```

### Search Indexes

**Vector Index** (Atlas Search):
```json
{
  "type": "vectorSearch",
  "path": "embedding",
  "numCandidates": 100,
  "similarity": "cosine"
}
```

**BM25 Index** (Atlas Search):
```json
{
  "type": "text",
  "paths": [
    "id", "title", "description", "steps", 
    "expectedResults", "module"
  ],
  "analyzer": "lucene.standard"
}
```

---

## 🔑 Critical Code Implementation

### Backend API Overview

**File**: [server/index.js](server/index.js)

| Lines | Functionality | Purpose |
|-------|---------------|---------|
| 1-50 | Imports & Setup | Express app, middleware, DNS config |
| 51-100 | Job Tracking | In-memory job management |
| 159-200 | Validation Helpers | MongoDB connection validation |
| 207-210 | Health Check | `GET /api/health` |
| 230-280 | Metadata API | `GET /api/metadata/distinct` |
| 300-450 | Excel Upload | `POST /api/upload-excel` |
| 450-750 | Embeddings (Individual) | `POST /api/create-embeddings` |
| 750-920 | Embeddings (Batch) | `POST /api/create-embeddings-batch` |
| 920-1050 | Background Processing | `processEmbeddings()` function |
| 1100-1200 | Query Preprocessing | `POST /api/search/preprocess` |
| 1200-1280 | Deduplication | `POST /api/search/deduplicate` |
| 1280-1450 | Summarization | `POST /api/search/summarize` |
| 1450-1600 | Prompt Testing | `POST /api/test-prompt` |
| 1620-1720 | Groq Reranking | `handleGroqOnlyReranking()` |
| 1790-1880 | Vector Search | `POST /api/search` |
| 1920-2000+ | BM25 Search | `POST /api/search/bm25` |

### Frontend Components

**Location**: [client/src/components](client/src/components/)

| Component | Purpose | Key Features |
|-----------|---------|--------------|
| **QuerySearch.js** | Main vector search | Query input, result pagination, filter options |
| **BM25Search.js** | Keyword search | Fuzzy matching, field selection |
| **HybridSearch.js** | Combined search | Score fusion, dual indexing |
| **RerankingSearch.js** | Groq reranking | LLM-powered reranking with candidate selection |
| **ConvertToJson.js** | Excel conversion | File upload, data type selection, preview |
| **EmbeddingsStore.js** | Batch embedding | File selection, job tracking, progress monitoring |
| **QueryPreprocessing.js** | Query analysis | Abbreviation/synonym expansion, normalization |
| **SummarizationDedup.js** | Post-processing | Deduplication, summarization, analytics |
| **PromptSchemaManager.js** | Prompt testing | Custom prompt execution with models |
| **Settings.js** | Configuration | API keys, model selection, environment variables |

### Utility Scripts

**Embeddings**: [src/scripts/utilities/mistralEmbedding.js](src/scripts/utilities/mistralEmbedding.js)
- `generateEmbedding(text)` - Single embedding (lines 25-80)
- `generateBatchEmbeddings(texts)` - Batch processing (lines 85-150)
- `generateEmbeddingsChunked(texts, chunkSize)` - Large dataset processing (lines 155-220)

**Reranking & Summarization**: [src/scripts/utilities/groqClient.js](src/scripts/utilities/groqClient.js)
- `rerankDocuments(query, documents, topK)` - LLM reranking (lines 40-110)
- `summarizeResults(query, documents, options)` - AI summarization (lines 120-220)
- `generateAnswer(query, documents, options)` - QA generation (lines 230-310)

---

## 🚀 Setup & Installation

### Prerequisites
- Node.js 16+ and npm
- MongoDB Atlas account (free tier OK)
- Mistral AI API key (free credits available)
- Groq API key (free tier available)

### Environment Configuration

Create `.env` file in project root:

```env
# MongoDB Configuration
MONGODB_URI=mongodb+srv://username:password@cluster.mongodb.net/?appName=YOUR_APP
DB_NAME=db_stories_tests
COLLECTION_NAME=test_cases
VECTOR_INDEX_NAME=vector_index
BM25_INDEX_NAME=bm25_index

# Mistral AI Configuration (Embeddings)
MISTRAL_API_KEY=your_mistral_key_here
MISTRAL_EMBEDDING_MODEL=mistral-embed

# Groq AI Configuration (Reranking & Summarization)
GROQ_API_KEY=your_groq_key_here
GROQ_RERANK_MODEL=llama-3.2-3b-preview  # Fast, cost-effective
GROQ_SUMMARIZATION_MODEL=llama-3.3-70b-versatile  # High quality

# Server Configuration
PORT=3001
```

### Installation Steps

```bash
# 1. Clone repository
git clone <repo-url>
cd rag-mongo-demo-v8

# 2. Install root and client dependencies
npm install

# 3. Setup MongoDB indexes
# Navigate to MongoDB Atlas → Your Database → Collections
# Create Vector Index on `embedding` field (cosine similarity)
# Create BM25 Index on text fields (title, description, etc.)

# 4. Start development server
npm run dev
# Backend runs on http://localhost:3001
# Frontend runs on http://localhost:3000
```

---

## 💻 Usage Guide

### 1. Upload & Convert Data

1. Go to **Data > Upload Excel** tab
2. Select Excel file with test cases or user stories
3. Choose correct sheet name
4. Select data type (Test Cases or User Stories)
5. Click "Convert to JSON"
6. File appears in data directory

### 2. Generate Embeddings

**Option A: Individual Processing**
1. Go to **Data > Create Embeddings**
2. Select files from list
3. Click "Create Embeddings"
4. Monitor progress in job tracker

**Option B: Batch Processing (Faster)**
1. Go to **Data > Embeddings Store**
2. Select multiple files
3. Click "Create Batch Embeddings"
4. Processes all files in parallel
5. Real-time progress tracking

### 3. Search Test Cases

#### Vector Search (Semantic)
1. Go to **Search > Query Search**
2. Enter query: "Login functionality testing"
3. Optional: Set filters (Module, Priority, Risk)
4. Results sorted by semantic relevance

#### BM25 Search (Keyword)
1. Go to **Search > BM25 Search**
2. Enter keywords: "login OR authentication"
3. Toggle fuzzy matching
4. Results sorted by keyword match score

#### Hybrid Search
1. Go to **Search > Hybrid Search**
2. Enter query
3. System combines Vector + BM25 results
4. Intelligent score fusion

#### Groq Reranking
1. Go to **Search > Reranking**
2. Enter complex query requiring semantic understanding
3. Results reranked by Groq LLM
4. Top-K most relevant results

### 4. Post-Search Analytics

1. **Deduplication**: Remove similar results (threshold: 0.85)
2. **Summarization**: Generate concise/detailed summaries
3. **Query Preprocessing**: Expand abbreviations and synonyms
4. **Prompt Testing**: Test custom prompts with Groq AI

---

## 📈 Performance Metrics

### Costs Involved

| Provider | Operation | Cost | Notes |
|----------|-----------|------|-------|
| **Mistral** | Embedding Generation | $0.10/M tokens | Batch processing recommended |
| **Groq** | Reranking | Free tier | 30 req/min limit |
| **Groq** | Summarization | Free tier | High quality, no charge |
| **MongoDB** | Vector Search | (Plan-based) | Included with Atlas |
| **MongoDB** | Storage | (Plan-based) | ~1.2KB per embedded document |

### Sample Latencies

```
Vector Search:      200-500ms (Mistral embedding = 100ms + search = 100-400ms)
BM25 Search:        50-150ms   (No network calls)
Hybrid Search:      300-700ms  (Both in parallel)
Groq Reranking:     2-5s       (LLM inference)
Summarization:      3-8s       (70B model, comprehensive analysis)
```

---

## 🔐 Security Notes

⚠️ **Never commit .env file** - Contains API keys

Protection measures implemented:
- Authentication tokens handled server-side
- CORS enabled only for localhost (development)
- MongoDB connection uses SSL/TLS
- API keys never exposed to frontend
- Input validation on all endpoints

**For Production**:
- Add authentication middleware (JWT/OAuth)
- Use environment-based config
- Implement rate limiting
- Add request validation schemas
- Use Redis for job tracking (not in-memory)

---

## 🛠️ Troubleshooting

### Common Issues

**1. "Search index not found"**
- Ensure MongoDB indexes are created in Atlas
- Allow 5-10 minutes for index creation
- **Check**: `GET /api/metadata/distinct`

**2. "MISTRAL_API_KEY is required"**
- Add key to .env file
- Restart server: `npm run dev`
- Get free key: https://console.mistral.ai

**3. "GROQ_API_KEY is required"**
- Required only for reranking/summarization
- Get free key: https://console.groq.com
- Free tier has rate limits

**4. Slow Embedding Generation**
- Use batch processing instead of individual
- Reduce file size or split large files
- Check network connection to Mistral

**5. No Results from Search**
- Ensure embeddings are generated first: `GET /api/jobs/active`
- Verify documents exist: Check MongoDB
- Try different query terms

---

## 📚 API Reference

### Core Endpoints

```javascript
// Health & Status
GET    /api/health                    // Server status
GET    /api/jobs/active               // List running jobs
GET    /api/jobs/:jobId               // Get specific job

// Data Management
GET    /api/files                     // List data files
GET    /api/metadata/distinct         // Get filter options
POST   /api/upload-excel              // Convert Excel to JSON
GET    /api/env                       // Read env vars
POST   /api/env                       // Update env vars

// Embeddings
POST   /api/create-embeddings         // Individual embedding
POST   /api/create-embeddings-batch   // Batch embedding

// Search APIs
POST   /api/search                    // Vector search
POST   /api/search/bm25               // BM25 search
POST   /api/search/rerank             // Groq reranking

// Query Processing
POST   /api/search/preprocess         // Normalize & expand query
POST   /api/search/analyze            // Analyze query

// Post-Search
POST   /api/search/deduplicate        // Remove duplicates
POST   /api/search/summarize          // Summarize results
POST   /api/test-prompt               // Test custom prompts
```

---

## 📖 Technology Stack

- **Frontend**: React 18, Material-UI, Axios
- **Backend**: Node.js, Express, Multer
- **Database**: MongoDB Atlas, Vector Search
- **AI Models**: 
  - Embeddings: Mistral AI (`mistral-embed`, 1024 dims)
  - Reranking: Groq (`llama-3.2-3b-preview`)
  - Summarization: Groq (`llama-3.3-70b-versatile`)
- **Utilities**: XLSX for Excel parsing, Concurrently for dev server

---

## 📝 License

MIT License - Free for personal and commercial use

---

## 🤝 Support & Contribution

For issues or features:
1. Check **Troubleshooting** section above
2. Review API responses for error details
3. Check MongoDB Atlas status
4. Verify API keys and quotas

**Debug Mode**:
- Check browser console for frontend errors
- Check terminal for backend logs
- Enable verbose logging in settings

---

## 📞 Contact & Resources

- **Mistral AI**: https://console.mistral.ai
- **Groq Console**: https://console.groq.com
- **MongoDB Atlas**: https://www.mongodb.com/cloud/atlas
- **GitHub**: [Repository Link]

---

**Last Updated**: April 2026  
**Version**: 1.0.0  
**Status**: Production Ready ✅
