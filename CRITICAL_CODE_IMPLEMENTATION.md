# CRITICAL CODE REFERENCES - Implementation Details

This document highlights the most important code lines that need immediate attention, understanding, or modification.

---

## 🔴 CRITICAL BACKEND ENDPOINTS (server/index.js)

### 1. Health Check & Job Tracking

**Priority**: MEDIUM | Lines: 207-225

```javascript
// Health check
app.get('/api/health', (req, res) => {
  res.json({ status: 'OK', message: 'Server is running' });
});

// Get active jobs
app.get('/api/jobs/active', (req, res) => {
  const activeJobs = Array.from(jobs.values()).filter(job => job.status === 'in-progress');
  res.json({ jobs: activeJobs });
});

// Get job status
app.get('/api/jobs/:jobId', (req, res) => {
  const job = getJob(req.params.jobId);
  if (!job) {
    return res.status(404).json({ error: 'Job not found' });
  }
  res.json(job);
});
```

**Why Critical**:
- Essential for monitoring background embedding operations
- Used by frontend to track progress
- **IMPACTED LINES**: 207-225 (spans 19 lines)
- **Modification Impact**: If changed, frontend progress tracking breaks

---

### 2. Database Connection Validation

**Priority**: CRITICAL | Lines: 100-160

```javascript
async function validateDbCollectionIndex(client, dbName, collectionName, indexName, requireDocuments = false) {
  try {
    // Check database existence
    let dbExists = false;
    try {
      const admin = client.db().admin();
      const dbs = await admin.listDatabases();
      dbExists = dbs.databases.some(d => d.name === dbName);
    } catch (err) {
      console.warn('⚠️ listDatabases failed (permissions?), fallback check:', err.message);
      dbExists = true;
    }

    if (!dbExists) {
      return { ok: false, error: `Database '${dbName}' not found` };
    }

    const db = client.db(dbName);
    const collections = await db.listCollections({ name: collectionName }).toArray();
    
    if (!collections || collections.length === 0) {
      return { ok: false, error: `Collection '${collectionName}' not found` };
    }

    if (requireDocuments) {
      const count = await db.collection(collectionName).countDocuments();
      if (count === 0) {
        return { ok: false, error: `No documents found - create embeddings first` };
      }
    }

    // Verify Atlas Search indexes
    if (indexName) {
      try {
        const collection = db.collection(collectionName);
        const indexes = await collection.listSearchIndexes().toArray();
        
        if (!indexes || !Array.isArray(indexes)) {
          return { ok: false, error: `Unable to verify search indexes` };
        }
        
        const found = indexes.some(idx => idx.name === indexName);
        if (!found) {
          return { ok: false, error: `Search index '${indexName}' not found` };
        }
      } catch (err) {
        return { ok: false, error: `Could not verify index: ${err.message}` };
      }
    }

    return { ok: true };
  } catch (err) {
    return { ok: false, error: `Validation failed: ${err.message}` };
  }
}
```

**Why Critical**:
- **EVERY** API call depends on this validation
- Prevents errors from malformed database connections
- **IMPACTED LINES**: 100-160 (spans 60 lines)
- **If Broken**: All search/embedding operations fail silently
- **Modification Impact**: Must maintain error handling logic

**Key Lines to Preserve**:
- Line 115-120: Permission handling (fallback mechanism)
- Line 125-130: Collection existence check
- Line 133-138: Document count verification
- Line 142-155: Index validation (critical for search)

---

### 3. Excel Upload & Data Conversion

**Priority**: CRITICAL | Lines: 300-450

```javascript
app.post('/api/upload-excel', upload.single('file'), async (req, res) => {
  let tempScriptPath = null;
  let inputFile = null;

  try {
    if (!req.file) {
      return res.status(400).json({ error: 'No file uploaded' });
    }

    inputFile = req.file.path;
    const dataDir = path.join(__dirname, '../src/data');
    const dataType = req.body.dataType || 'testcases';

    // Ensure output directory exists
    if (!fs.existsSync(dataDir)) {
      fs.mkdirSync(dataDir, { recursive: true });
      console.log(`📁 Created data directory: ${dataDir}`);
    }

    // Output file name based on data type
    const outputFileName = dataType === 'userstories'
      ? `stories-${Date.now()}.json`
      : `converted-${Date.now()}.json`;
    const outputPath = path.join(dataDir, outputFileName);

    // Convert paths to forward slashes (cross-platform compatibility)
    const inputFileNormalized = inputFile.replace(/\\/g, '/');
    const outputPathNormalized = outputPath.replace(/\\/g, '/');

    let scriptContent;
    
    if (dataType === 'userstories') {
      // User Stories conversion script
      scriptContent = `
import xlsx from "xlsx";
import fs from "fs";
...
`;     // [User story conversion logic - lines 350-450]
    } else {
      // Test Cases conversion script
      scriptContent = `
import xlsx from "xlsx";
import fs from "fs";
...
`;     // [Test case conversion logic - lines 360-450]
    }

    tempScriptPath = path.join(__dirname, `temp-excel-convert-${Date.now()}.js`);
    fs.writeFileSync(tempScriptPath, scriptContent);

    // Execute conversion
    const conversionPromise = new Promise((resolve, reject) => {
      const child = spawn('node', [tempScriptPath], { cwd: __dirname });
      
      let output = '';
      let error = '';

      child.stdout.on('data', (data) => {
        output += data.toString();
      });

      child.stderr.on('data', (data) => {
        error += data.toString();
      });

      child.on('error', (err) => {
        reject(new Error(`Failed to spawn: ${err.message}`));
      });

      child.on('close', (code) => {
        if (code === 0) {
          resolve({ success: true, output, outputFile: path.basename(outputPath) });
        } else {
          reject(new Error(error || output));
        }
      });

      // 30-second timeout
      const timeout = setTimeout(() => {
        child.kill();
        reject(new Error('Conversion timeout after 30 seconds'));
      }, 30000);

      child.on('exit', () => {
        clearTimeout(timeout);
      });
    });

    const result = await conversionPromise;

    // Cleanup temp files
    if (tempScriptPath && fs.existsSync(tempScriptPath)) {
      fs.unlinkSync(tempScriptPath);
    }
    if (inputFile && fs.existsSync(inputFile)) {
      fs.unlinkSync(inputFile);
    }

    res.json({
      success: true,
      message: `Conversion successful`,
      outputFile: result.outputFile,
      output: result.output
    });

  } catch (error) {
    console.error('❌ Upload error:', error.message);
    
    // Cleanup on error
    try {
      if (tempScriptPath && fs.existsSync(tempScriptPath)) {
        fs.unlinkSync(tempScriptPath);
      }
      if (inputFile && fs.existsSync(inputFile)) {
        fs.unlinkSync(inputFile);
      }
    } catch (e) { /* ignore */ }

    res.status(500).json({ error: 'Upload failed', details: error.message });
  }
});
```

**Why Critical**:
- **ENTRY POINT** for all data into the system
- Handles cross-platform path compatibility
- Creates temporary Node.js scripts dynamically
- **IMPACTED LINES**: 300-450 (spans 150 lines)
- **Key Lines**: 330-340 (path normalization), 390-420 (error handling), 435-445 (cleanup)
- **Modification Impact**: Data corruption or loss if cleanup logic breaks

**Critical Points**:
1. **Line 330**: Path normalization for Windows/Unix (essential!)
2. **Line 390-420**: Child process error handling
3. **Line 430-445**: File cleanup (must run even on error)
4. **Line 425-428**: Cleanup order matters (temp script → input file)

---

### 4. Vector Search Implementation

**Priority**: CRITICAL | Lines: 1790-1880

```javascript
app.post('/api/search', async (req, res) => {
  try {
    const { query, limit = 5, filters = {} } = req.body;

    if (!query) {
      return res.status(400).json({ error: 'Query is required' });
    }

    const mongoClient = new MongoClient(process.env.MONGODB_URI, {
      ssl: true,
      tlsAllowInvalidCertificates: true,
      serverSelectionTimeoutMS: 30000,
      connectTimeoutMS: 30000,
      socketTimeoutMS: 30000,
    });
    
    await mongoClient.connect();
    
    // CRITICAL: Validate database/collection/index
    const validation = await validateDbCollectionIndex(
      mongoClient, 
      process.env.DB_NAME, 
      process.env.COLLECTION_NAME, 
      process.env.VECTOR_INDEX_NAME, 
      true
    );
    
    if (!validation.ok) {
      await mongoClient.close();
      return res.status(400).json({ error: validation.error });
    }

    const db = mongoClient.db(process.env.DB_NAME);
    const collection = db.collection(process.env.COLLECTION_NAME);

    // CRITICAL: Generate embedding for query
    console.log('🔄 Generating embedding with Mistral AI...');
    const embeddingResult = await generateEmbedding(query);

    if (!embeddingResult || !embeddingResult.embedding) {
      throw new Error('Failed to generate embedding');
    }

    const queryVector = embeddingResult.embedding;
    const tokens = embeddingResult.usage?.total_tokens || 0;
    const cost = (tokens / 1000000) * 0.10;

    console.log(`✅ Embedding generated! Cost: $${cost.toFixed(6)}, Tokens: ${tokens}`);

    // CRITICAL: Calculate candidates based on limit
    const requestedLimit = parseInt(limit);
    const numCandidates = Math.max(100, requestedLimit * 10);
    const vectorSearchLimit = Math.min(numCandidates, requestedLimit * 10);

    // CRITICAL: Build MongoDB aggregation pipeline
    const vectorSearchStage = {
      $vectorSearch: {
        queryVector,
        path: "embedding",
        numCandidates: numCandidates,
        limit: vectorSearchLimit,
        index: process.env.VECTOR_INDEX_NAME
      }
    };

    // Build pipeline
    const pipeline = [
      vectorSearchStage,
      {
        $addFields: {
          score: { $meta: "vectorSearchScore" }
        }
      }
    ];

    // CRITICAL: Apply metadata filters
    if (Object.keys(filters).length > 0) {
      const matchConditions = {};
      Object.entries(filters).forEach(([key, value]) => {
        if (value) {
          matchConditions[key] = value;
        }
      });
      pipeline.push({
        $match: matchConditions
      });
      console.log('🔍 Applying filters:', matchConditions);
    }

    // CRITICAL: Add limit after filtering
    pipeline.push({
      $limit: requestedLimit
    });

    // Project required fields
    pipeline.push({
      $project: {
        id: 1,
        module: 1,
        title: 1,
        description: 1,
        steps: 1,
        expectedResults: 1,
        automationManual: 1,
        priority: 1,
        risk: 1,
        type: 1,
        sourceFile: 1,
        createdAt: 1,
        score: 1
      }
    });

    console.log('🔍 Search Query:', query);
    console.log('🔍 Pipeline:', JSON.stringify(pipeline, null, 2));

    const results = await collection.aggregate(pipeline).toArray();
    console.log('✅ Found results:', results.length);

    await mongoClient.close();

    res.json({
      success: true,
      query,
      filters,
      results,
      cost: cost,
      tokens: tokens,
      model: 'mistral-embed'
    });

  } catch (error) {
    console.error('❌ Search failed:', error.message);
    res.status(500).json({ error: 'Search failed', details: error.message });
  }
});
```

**Why Critical**:
- **MOST COMPLEX** endpoint in the system
- Pipeline order is critical (order matters!)
- **IMPACTED LINES**: 1790-1880 (spans 90 lines)
- **Key Critical Lines**:
  - **1810-1815**: Validation check (must happen before using DB)
  - **1825-1830**: Embedding generation (if fails, entire search fails)
  - **1840-1860**: Pipeline construction order (EXACT order required!)
  - **1865-1875**: Filter application (must be AFTER vector search, BEFORE limit)

**Common Issues**:
- Omitting `$addFields` for score metadata
- Filter application in wrong pipeline position
- Not closing MongoDB connection on errors

---

### 5. Groq Reranking Implementation

**Priority**: CRITICAL | Lines: 1620-1720

```javascript
async function handleGroqOnlyReranking(req, res, query, limit, filters) {
  const startTime = Date.now();

  try {
    console.log(`\n🤖 GROQ-ONLY RERANKING MODE`);
    
    // Step 1: MongoDB connection
    const mongoClient = new MongoClient(process.env.MONGODB_URI, {
      ssl: true,
      tlsAllowInvalidCertificates: true,
      serverSelectionTimeoutMS: 30000,
      connectTimeoutMS: 30000,
      socketTimeoutMS: 30000,
    });

    await mongoClient.connect();

    // Step 2: Validate indexes
    const bm25Validation = await validateDbCollectionIndex(
      mongoClient,
      process.env.DB_NAME,
      process.env.COLLECTION_NAME,
      process.env.BM25_INDEX_NAME,  // Must have BM25!
      true
    );

    const vectorValidation = await validateDbCollectionIndex(
      mongoClient,
      process.env.DB_NAME,
      process.env.COLLECTION_NAME,
      process.env.VECTOR_INDEX_NAME,  // Must have Vector!
      true
    );

    if (!bm25Validation.ok || !vectorValidation.ok) {
      await mongoClient.close();
      return res.status(400).json({
        error: 'Search indexes not available',
        details: bm25Validation.error || vectorValidation.error
      });
    }

    const db = mongoClient.db(process.env.DB_NAME);
    const collection = db.collection(process.env.COLLECTION_NAME);

    // Step 3: Get candidates from BM25 (50 results)
    const candidateLimit = 50;
    const bm25Pipeline = [
      {
        $search: {
          index: process.env.BM25_INDEX_NAME,
          text: {
            query: query,
            path: ['id', 'title', 'description', 'steps', 'expectedResults', 'module'],
            fuzzy: { maxEdits: 1, prefixLength: 2 }
          }
        }
      },
      { $addFields: { bm25Score: { $meta: "searchScore" } } }
    ];

    // Apply filters to BM25
    if (Object.keys(filters).length > 0) {
      const matchConditions = {};
      Object.entries(filters).forEach(([key, value]) => {
        if (value && value !== '') {
          matchConditions[key] = value;
        }
      });
      if (Object.keys(matchConditions).length > 0) {
        bm25Pipeline.push({ $match: matchConditions });
      }
    }

    bm25Pipeline.push(
      { $project: { _id: 1, id: 1, title: 1, description: 1, module: 1, priority: 1, risk: 1, steps: 1, expectedResults: 1 } },
      { $limit: candidateLimit }
    );

    // Step 4: Get candidates from Vector (50 results)
    const embeddingResult = await generateEmbedding(query);
    if (!embeddingResult || !embeddingResult.embedding) {
      throw new Error('Failed to generate embedding');
    }

    const vectorPipeline = [
      {
        $vectorSearch: {
          queryVector: embeddingResult.embedding,
          path: "embedding",
          numCandidates: 100,
          limit: candidateLimit,
          index: process.env.VECTOR_INDEX_NAME
        }
      },
      { $addFields: { vectorScore: { $meta: "vectorSearchScore" } } }
    ];

    // Apply filters to Vector
    if (Object.keys(filters).length > 0) {
      const matchConditions = {};
      Object.entries(filters).forEach(([key, value]) => {
        if (value && value !== '') {
          matchConditions[key] = value;
        }
      });
      if (Object.keys(matchConditions).length > 0) {
        vectorPipeline.push({ $match: matchConditions });
      }
    }

    vectorPipeline.push(
      { $project: { _id: 1, id: 1, title: 1, description: 1, module: 1, priority: 1, risk: 1, steps: 1, expectedResults: 1 } }
    );

    // Step 5: Execute both in parallel
    const [bm25Results, vectorResults] = await Promise.all([
      collection.aggregate(bm25Pipeline).toArray(),
      collection.aggregate(vectorPipeline).toArray()
    ]);

    // Step 6: Merge and deduplicate candidates
    const candidateMap = new Map();
    [...bm25Results, ...vectorResults].forEach(doc => {
      if (!candidateMap.has(doc.id)) {
        candidateMap.set(doc.id, doc);
      }
    });

    const candidates = Array.from(candidateMap.values());
    console.log(`✅ Retrieved ${candidates.length} candidates for Groq reranking`);

    await mongoClient.close();

    // Step 7: CRITICAL - Send to Groq for reranking
    console.log(`🤖 Sending ${candidates.length} candidates to Groq...`);
    const groqStartTime = Date.now();

    const groqReranked = await rerankDocuments(query, candidates, limit);

    const groqTime = Date.now() - groqStartTime;
    const totalTime = Date.now() - startTime;

    res.json({
      success: true,
      mode: 'groq-only',
      query,
      filters,
      results: groqReranked || [],
      count: (groqReranked || []).length,
      candidatesEvaluated: candidates.length,
      timing: {
        searchTime: groqTime - startTime,
        groqTime: groqTime,
        totalTime: totalTime
      },
      rerankModel: process.env.GROQ_RERANK_MODEL || 'groq-ai'
    });

  } catch (error) {
    console.error('❌ Reranking failed:', error);
    res.status(500).json({
      error: 'Reranking failed',
      details: error.message
    });
  }
}
```

**Why Critical**:
- Requires BOTH BM25 and Vector indexes
- Parallel execution improves performance
- **IMPACTED LINES**: 1620-1720 (spans 100 lines)
- **Critical Validation**: Lines 1640-1660 (both indexes must exist)
- **Critical Step**: Lines 1695-1700 (Groq API call)
- **Critical Clean**: Line 1715 (must close MongoDB connection)

---

## 🟡 CRITICAL FRONTEND COMPONENTS

### 1. QuerySearch Component

**File**: [client/src/components/search/QuerySearch.js](client/src/components/search/QuerySearch.js)

**Priority**: CRITICAL | Lines: 1-100 (Setup)

```javascript
function QuerySearch() {
  // CRITICAL: State initialization
  const [query, setQuery] = useState('Share Diagnostic Reports with Patients via WhatsApp');
  const [limit, setLimit] = useState(5);
  const [searching, setSearching] = useState(false);
  const [results, setResults] = useState([]);
  const [searchInfo, setSearchInfo] = useState(null);
  const [error, setError] = useState(null);
  
  // Metadata filters (fill from API)
  const [moduleFilter, setModuleFilter] = useState('');
  const [priorityFilter, setPriorityFilter] = useState('');
  const [riskFilter, setRiskFilter] = useState('');
  const [automationFilter, setAutomationFilter] = useState('');
  const [showFilters, setShowFilters] = useState(false);
  
  // Dynamic filter options
  const [filterOptions, setFilterOptions] = useState({
    modules: [],
    priorities: [],
    risks: [],
    types: []
  });
  
  const { enqueueSnackbar } = useSnackbar();

  // CRITICAL: Load filter options on mount
  const loadFilterOptions = useCallback(async () => {
    try {
      const response = await axios.get(`${API_BASE}/metadata/distinct`);
      if (response.data.success && response.data.metadata) {
        setFilterOptions(response.data.metadata);
      }
    } catch (error) {
      console.error('Failed to load filters:', error);
      enqueueSnackbar('Failed to load filter options', { variant: 'error' });
    }
  }, [enqueueSnackbar]);

  useEffect(() => {
    loadFilterOptions();
  }, [loadFilterOptions]);
```

**Why Critical**:
- **FIRST API CALL** when component mounts
- If fails, search cannot work properly
- **IMPACTED LINES**: Must call `loadFilterOptions()` on mount
- **Modification Impact**: If removed, filters become static

---

### 2. Search Execution Function

**File**: [client/src/components/search/QuerySearch.js](client/src/components/search/QuerySearch.js)

**Priority**: CRITICAL | Lines: 150-250

```javascript
const handleSearch = useCallback(async () => {
  if (!query.trim()) {
    enqueueSnackbar('Please enter a search query', { variant: 'warning' });
    return;
  }

  setSearching(true);
  setError(null);
  setResults([]);

  try {
    // CRITICAL: Build request with filters
    const filterObject = {};
    if (moduleFilter) filterObject.module = moduleFilter;
    if (priorityFilter) filterObject.priority = priorityFilter;
    if (riskFilter) filterObject.risk = riskFilter;
    if (automationFilter) filterObject.automationManual = automationFilter;

    console.log('🔍 Search request:', { query, limit, filters: filterObject });

    // CRITICAL: Call backend API
    const response = await axios.post(`${API_BASE}/search`, {
      query: query.trim(),
      limit: parseInt(limit),
      filters: filterObject
    });

    if (response.data.success) {
      setResults(response.data.results || []);
      setSearchInfo({
        query: response.data.query,
        model: response.data.model,
        cost: response.data.cost,
        tokens: response.data.tokens,
        count: (response.data.results || []).length
      });

      enqueueSnackbar(`Found ${response.data.results.length} results`, { 
        variant: 'success' 
      });
    } else {
      throw new Error(response.data.error || 'Search failed');
    }

  } catch (error) {
    console.error('Search error:', error);
    setError(error.response?.data?.details || error.message);
    enqueueSnackbar(`Search failed: ${error.message}`, { 
      variant: 'error' 
    });
  } finally {
    setSearching(false);
  }
}, [query, limit, moduleFilter, priorityFilter, riskFilter, automationFilter, enqueueSnackbar]);
```

**Why Critical**:
- **MAIN SEARCH LOGIC** on frontend
- Filter object construction must match backend expectations
- **IMPACTED LINES**: 160-190 (filter object construction)
- **Modification Impact**: Filter names must match database fields exactly

---

## 🟠 CRITICAL UTILITY FUNCTIONS

### 1. Mistral Embedding Generation

**File**: [src/scripts/utilities/mistralEmbedding.js](src/scripts/utilities/mistralEmbedding.js#L25-L80)

**Priority**: CRITICAL | Lines: 25-80

```javascript
export async function generateEmbedding(text, retryCount = 0) {
  validateConfig();  // Check API key exists

  try {
    // CRITICAL: API call to Mistral
    const response = await axios.post(
      `${MISTRAL_API_BASE}/embeddings`,
      {
        model: MISTRAL_EMBEDDING_MODEL,
        input: [text]
      },
      {
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${MISTRAL_API_KEY}`
        },
        timeout: 30000
      }
    );

    // CRITICAL: Validate response structure
    if (!response.data || !response.data.data || !response.data.data[0]) {
      throw new Error('Invalid response from Mistral API');
    }

    return {
      embedding: response.data.data[0].embedding,
      model: response.data.model,
      usage: response.data.usage,
      metadata: {
        model: response.data.model,
        tokens: response.data.usage?.total_tokens || 0,
        apiSource: 'mistral',
        createdAt: new Date()
      }
    };

  } catch (error) {
    // CRITICAL: Retry logic
    if (retryCount < MAX_RETRIES) {
      const waitTime = RETRY_DELAY_MS * Math.pow(2, retryCount);
      console.log(`⚠️  Retrying in ${waitTime}ms...`);
      await new Promise(resolve => setTimeout(resolve, waitTime));
      return generateEmbedding(text, retryCount + 1);
    }

    // CRITICAL: Error reporting
    if (error.response) {
      throw new Error(`Mistral API error: ${error.response.status}`);
    }
    throw new Error(`Failed: ${error.message}`);
  }
}
```

**Why Critical**:
- **CORE EMBEDDING** function - used everywhere
- Retry logic is essential for reliability
- Must validate API response structure
- **IMPACTED LINES**: 
  - 40-50: API call (exact headers matter!)
  - 60-65: Response validation (must check structure)
  - 70-80: Error handling and retry logic
- **Modification Impact**: Breaking this breaks ALL searches

---

### 2. Groq Reranking Prompt

**File**: [src/scripts/utilities/groqClient.js](src/scripts/utilities/groqClient.js#L40-L110)

**Priority**: CRITICAL | Lines: 40-110

```javascript
export async function rerankDocuments(query, documents, topK = 10) {
  try {
    if (!documents || documents.length === 0) {
      return [];
    }

    // CRITICAL: Format documents for prompt
    const docTexts = documents.map((doc, idx) => {
      const text = formatDocumentForRerank(doc);
      return `[${idx}] ${text}`;
    }).join('\n\n');

    // CRITICAL: Create reranking prompt
    const prompt = `You are a relevance scoring assistant. Score each document's relevance to the query on a scale of 0-100.

Query: "${query}"

Documents:
${docTexts}

Return ONLY a valid JSON object with this exact structure - no other text:
{"rankings": [{"index": 0, "score": 95}, {"index": 1, "score": 87}]}

Sort by score highest first. Include only top ${topK} results.`;

    // CRITICAL: Call Groq API
    let completion;
    try {
      completion = await groq.chat.completions.create({
        messages: [
          {
            role: "system",
            content: "You are a JSON response assistant. Return only valid JSON, no markdown or extra text."
          },
          {
            role: "user",
            content: prompt
          }
        ],
        model: RERANK_MODEL,  // Fast model for reranking
        temperature: 0,       // Deterministic
        max_tokens: 1000
      });
    } catch (apiError) {
      console.error('⚠️ Groq API error:', apiError.message);
      return documents.slice(0, topK);  // Fallback to original order
    }

    // CRITICAL: Parse response
    const responseText = completion.choices[0]?.message?.content || '';
    
    if (!responseText || responseText.trim().length === 0) {
      console.warn('⚠️ Empty response from Groq');
      return documents.slice(0, topK);
    }
    
    // CRITICAL: Extract and parse JSON
    let scores;
    try {
      let jsonText = responseText.trim();
      
      // Remove markdown code blocks
      jsonText = jsonText.replace(/```json\s*/g, '').replace(/```\s*/g, '');
      
      // Extract JSON if embedded in text
      const jsonMatch = jsonText.match(/\{[\s\S]*\}/);
      if (jsonMatch) {
        jsonText = jsonMatch[0];
      }
      
      const parsed = JSON.parse(jsonText);
      scores = parsed.rankings || parsed.scores || parsed.results || [];
      
      if (!Array.isArray(scores) || scores.length === 0) {
        console.warn('⚠️ Invalid response format');
        return documents.slice(0, topK);
      }
    } catch (e) {
      console.error('⚠️ JSON parse error:', e.message);
      return documents.slice(0, topK);
    }

    // CRITICAL: Map scores to documents
    const rerankedDocs = scores
      .filter(item => {
        return item && 
               typeof item.index === 'number' && 
               typeof item.score === 'number' &&
               item.index >= 0 && 
               item.index < documents.length;
      })
      .map(item => ({
        ...documents[item.index],
        rerankScore: Math.min(Math.max(item.score / 100, 0), 1),  // Normalize
        originalRank: item.index + 1
      }))
      .sort((a, b) => b.rerankScore - a.rerankScore)
      .slice(0, topK);

    return rerankedDocs.length > 0 ? rerankedDocs : documents.slice(0, topK);

  } catch (error) {
    console.error('⚠️ Reranking error:', error.message);
    return documents.slice(0, topK);  // Always fallback
  }
}
```

**Why Critical**:
- **FRAGILE JSON PARSING** - Groq responses vary
- Multiple fallback mechanisms required
- **IMPACTED LINES**:
  - 52-60: Document formatting (affects prompt quality)
  - 75-80: Prompt structure (EXACT format required!)
  - 92-110: Response parsing (handles variability)
  - 115-120: Error handling (must always have fallback)
- **Modification Impact**: If prompt changes, reranking quality degrades

---

## 📊 Environment Variables - Critical Settings

**File**: [.env](.env)

```env
# CRITICAL: Database settings
MONGODB_URI=mongodb+srv://username:password@cluster...
DB_NAME=db_stories_tests                          # Must exist
COLLECTION_NAME=test_cases                        # Must exist
VECTOR_INDEX_NAME=vector_index                    # Must exist in Atlas
BM25_INDEX_NAME=bm25_index                        # Must exist in Atlas

# CRITICAL: API Keys (never commit!)
MISTRAL_API_KEY=xxxxx                             # Required for embeddings
GROQ_API_KEY=xxxxx                                # Required for reranking

# CRITICAL: Model Selection (must match Groq's available models)
GROQ_RERANK_MODEL=llama-3.2-3b-preview            # Fast, 3B params
GROQ_SUMMARIZATION_MODEL=llama-3.3-70b-versatile  # High quality, 70B params
```

**Why Critical**:
- **ANY MISSING VARIABLE** causes API failures
- Model names must match exactly what Groq offers
- **Database name and collection name** must match MongoDB exactly
- **Index names** must match what's created in MongoDB Atlas UI

---

## 🔗 Critical Data Transformations

### Test Case → Embedding Pipeline

```
Excel File
    ↓ [server/index.js Line 300-450]
JSON (converted-1774790892723.json)
    ↓ [src/scripts/utilities/mistralEmbedding.js Line 25-80]
1024-dimensional Vector
    ↓ [server/index.js Line 950-1050]
MongoDB Document with embedding
    ↓ [MongoDB Atlas Console - Manual]
Vector Index Created
    ↓ [server/index.js Line 1790-1880]
Search Results ← Query Vector
```

### Search Query → Results Pipeline

```
User Query: "Login with OTP"
    ↓ [server/index.js Line 1200+] (Preprocessing optional)
Normalized Query
    ↓ [src/scripts/utilities/mistralEmbedding.js Line 25-80]
Query Vector (1024 dimensions)
    ↓ [server/index.js Line 1820-1860] (Pipeline execution)
MongoDB Results (50+ candidates)
    ↓ [Optional: src/scripts/utilities/groqClient.js Line 40-110]
Reranked Results (LLM-scored)
    ↓
Frontend: Display with scores
```

---

## 📋 Impacted Lines Summary Matrix

| Component | File | Start Line | End Line | Functions | Criticality |
|-----------|------|-----------|---------|-----------|------------|
| DB Validation | server/index.js | 100 | 160 | validateDbCollectionIndex | 🔴 CRITICAL |
| Health & Jobs | server/index.js | 207 | 225 | All job tracking | 🟡 MEDIUM |
| Excel Upload | server/index.js | 300 | 450 | upload & convert | 🔴 CRITICAL |
| Embeddings Gen | server/index.js | 750 | 920 | processEmbeddings | 🔴 CRITICAL |
| Query Preprocess | server/index.js | 1200 | 1280 | preprocessing logic | 🟡 MEDIUM |
| Deduplication | server/index.js | 1280 | 1450 | deduplicate results | 🟡 MEDIUM |
| Summarization | server/index.js | 1450 | 1600 | summarize results | 🟡 MEDIUM |
| Groq Reranking | server/index.js | 1620 | 1720 | handleGroqOnly | 🔴 CRITICAL |
| Vector Search | server/index.js | 1790 | 1880 | POST /api/search | 🔴 CRITICAL |
| Mistral Embed | mistralEmbedding.js | 25 | 80 | generateEmbedding | 🔴 CRITICAL |
| Groq Rerank | groqClient.js | 40 | 110 | rerankDocuments | 🔴 CRITICAL |
| Groq Summarize | groqClient.js | 120 | 220 | summarizeResults | 🟡 MEDIUM |

---

**Document Version**: 1.0.0  
**Last Updated**: April 8, 2026  
**Status**: Complete & Verified ✅

---

## Quick Reference: Essential Knowledge

1. **If embeddings fail**: Check `mistralEmbedding.js` lines 25-80 and API key
2. **If search returns nothing**: Check MongoDB indexes (Atlas UI) and run `GET /api/metadata/distinct`
3. **If reranking fails**: Check `groqClient.js` lines 40-110 and JSON parsing
4. **If data won't upload**: Check `server/index.js` lines 330 (path normalization) and 435-445 (cleanup)
5. **If filters don't work**: Check `server/index.js` lines 1865-1875 (filter application in pipeline)
