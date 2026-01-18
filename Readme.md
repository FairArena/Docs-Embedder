<p align="center">
  <img src="https://fairarena.blob.core.windows.net/fairarena/fairArenaLogo.png" alt="FairArena Logo" width="140" height="140">
</p>

# Documentation Embeddings Generator

A Google Colab notebook that fetches, parses, and indexes all documentation, configuration files, and developer resources from the FairArena repositories into Pinecone vector databases for semantic search capabilities.

## 📋 Overview

This notebook creates **two separate Pinecone indices**:

1. **`fairarena-docs-768`** - Client-facing documentation
   - 27 MDX/MD files from `FairArena/FairArena-Docs` repo
   - API documentation, features, and user guides
   
2. **`fairarena-main-768`** - Developer documentation & configuration
   - Markdown files from `FairArena/FairArena` repo
   - Postman collections, Prisma schemas, Docker configs
   - Shell scripts, GitHub workflows, VS Code settings
   - YAML configurations, package.json files

All embeddings use **768-dimensional vectors** (all-mpnet-base-v2 model) for superior semantic understanding.

## 🚀 Quick Start

### Prerequisites

1. **Google Colab** account
2. **GitHub Personal Access Token (PAT)**
   - Go to [GitHub Settings → Developer settings → Personal access tokens](https://github.com/settings/tokens)
   - Create token with `repo` scope (for private repos)
   - Copy the token
   
3. **Pinecone Account**
   - Create account at [pinecone.io](https://pinecone.io)
   - Get your API key
   
4. **ngrok Account** (for embedding API tunnel)
   - Create account at [ngrok.com](https://ngrok.com)
   - Get your auth token

### Setup

1. Open the notebook in Google Colab
2. Add secrets to Colab:
   - Click 🔑 Secrets icon (left sidebar)
   - Add these secrets:
     - `GITHUB_PAT` → Your GitHub Personal Access Token
     - `PINECODE_DB_API_KEY` → Your Pinecone API key
     - `NGROK_AUTH_TOKEN` → Your ngrok auth token

3. Run cells in order (don't skip any!)

## 📚 Notebook Structure

### Cell 1: Install Dependencies
```python
%pip install -q sentence-transformers fastapi uvicorn nest-asyncio pyngrok torch pinecone PyGithub python-frontmatter requests
```
Installs required Python packages for embeddings, API, and GitHub interactions.

### Cell 2: Load Secrets
Imports `userdata` to access secrets from Google Colab vault.

### Cell 3: Configure ngrok
Sets up ngrok tunnel for local embedding API exposure.

### Cell 4: Create Embedding API
- Loads `all-mpnet-base-v2` model (768 dimensions)
- Creates FastAPI server on localhost:8000
- Exposes via ngrok tunnel
- Endpoints:
  - `GET /` → Returns model info
  - `POST /embed` → Generates embeddings for text batch

### Cell 5: Configuration
Sets up credentials and index names:
```python
GITHUB_TOKEN = userdata.get('GITHUB_PAT')
GITHUB_REPO = "FairArena/FairArena-Docs"
PINECONE_API_KEY = userdata.get('PINECODE_DB_API_KEY')
PINECONE_INDEX_NAME = "fairarena-docs-768"
```

### Cell 6: Pipeline for Client Docs
- Fetches all `.mdx` and `.md` files from `FairArena/FairArena-Docs`
- Parses frontmatter metadata
- Generates embeddings via API
- Uploads to `fairarena-docs-768` Pinecone index

### Cell 7: Fetch Specialized Files
Fetches from `FairArena/FairArena` (main repo):
- **Entire folders**: `.github/`, `.husky/`, `.vscode/`, `Backend/postman/`, `Backend/prisma/`
- **File patterns**: 
  - `package.json`, `package-lock.json`
  - `Dockerfile`, `docker-compose.*`, `.dockerignore`
  - `*.sh` scripts
  - `*.yaml`, `*.yml` files
  - (Skips `pnpm-lock.yaml` - too large)

**Rate limiting**: 0.5s delay between API calls to respect GitHub rate limits.

### Cell 8: Embed & Upload Specialized Files
- Parses specialized files
- Generates embeddings
- Uploads to `fairarena-main-768` Pinecone index with metadata tags

## 🔑 Key Features

### ✅ Rate Limiting
- 0.5s delay between GitHub API calls
- Prevents hitting 5000 req/hour limit
- Auto-retry on rate limit with 60s wait

### ✅ Private Repository Access
- Uses GitHub PAT for authenticated requests
- Supports private repos like FairArena

### ✅ Metadata Preservation
All vectors include rich metadata:
```json
{
  "id": "unique_id",
  "title": "Document title",
  "description": "Brief description",
  "file_path": "path/in/repo",
  "url": "github_url",
  "category": "config_files|docker|postman|etc",
  "file_type": "json|yaml|sh|etc",
  "source": "FairArena-Docs|FairArena-Main|FairArena-Specialized"
}
```

### ✅ Error Handling
- Graceful handling of encoding issues
- Retries failed batch requests
- Informative error messages

## 📊 Output Indices

### fairarena-docs-768
```
Vector Count: 27
Dimension: 768
Metric: Cosine Similarity
Content: Client documentation (MDX/MD)
```

### fairarena-main-768
```
Vector Count: ~100+ (depending on repo structure)
Dimension: 768
Metric: Cosine Similarity
Content: Dev docs + configs + scripts + workflows
```

## 🔍 Querying the Indices

### Using Python
```python
from pinecone import Pinecone

pc = Pinecone(api_key="your_api_key")
index = pc.Index("fairarena-docs-768")

# Query
results = index.query(
    vector=[0.1, 0.2, ...],  # Your 768-dim embedding
    top_k=5,
    include_metadata=True
)

for match in results['matches']:
    print(f"Title: {match['metadata']['title']}")
    print(f"URL: {match['metadata']['url']}")
    print(f"Score: {match['score']}\n")
```

### Using REST API
```bash
curl -X POST "https://your-index.pinecone.io/query" \
  -H "Api-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "vector": [0.1, 0.2, ...],
    "topK": 5,
    "includeMetadata": true
  }'
```

## ⚠️ Troubleshooting

### "Rate limit exceeded" error
- Notebook auto-handles with 60s wait
- Reduce `rate_limit_delay` if running multiple times
- Create new GitHub PAT if limit already hit

### "Embedding API not responding"
- Cell 4 (embedding API) must run successfully first
- Check ngrok tunnel is active: visit the logged URL
- Verify EMBEDDING_API_URL is set correctly

### "Pinecone index already exists"
- Code checks and reuses existing index
- To reset: delete index manually from pinecone.io console

### "GitHub authentication failed"
- Verify GITHUB_PAT secret is added to Colab
- Check token has `repo` scope
- Token must not be expired

### "Files not found in repo"
- Verify folder paths exist (e.g., `Backend/postman/`)
- Some repos may have different structure
- Check with: `repo.get_contents(path)` manually

## 📈 Performance Notes

- **First run**: 5-10 minutes (depends on file count)
- **API calls**: ~150-200 GitHub API calls
- **Embedding generation**: ~0.5s per batch (10 texts)
- **Pinecone upload**: Batched in groups of 50

## 🔐 Security

- All secrets stored in Colab vault (not in code)
- GitHub PAT only used for authenticated requests
- Pinecone API key never logged
- ngrok tunnel auto-killed after session

## 🎯 Use Cases

1. **AI-Powered Documentation Chat** - Query indices to find relevant docs
2. **Developer Assistant** - Search for configs/setup instructions
3. **Code Generation** - Reference documentation for context
4. **Documentation Discovery** - Find related docs via similarity
5. **API Documentation Search** - Quick lookup of endpoints

## 🚀 Next Steps

After running this notebook:

1. **Test the indices**:
   ```python
   # Generate embedding for query
   query_embedding = model.encode("How to setup authentication?")
   # Query both indices
   results = index.query(query_embedding, top_k=5)
   ```

2. **Build a chatbot** using the indices
3. **Integrate with your app** via Pinecone API
4. **Schedule daily updates** using GitHub Act

---

## 📄 License

This project is licensed under the **Proprietary License** — see the [LICENSE](LICENSE) file for details.

---

<p align="center">
  <a href="https://fair.sakshamg.me">🌐 Website</a> •
  <a href="https://github.com/FairArena/FairArena">💻 GitHub</a> •
  <a href="mailto:fairarena.contact@gmail.com">📧 Support</a>
</p>

<p align="center">
  <sub>Built with ❤️ by the FairArena Team</sub>
</p>