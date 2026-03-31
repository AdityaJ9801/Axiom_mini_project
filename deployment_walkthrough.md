# AXIOM AI — Azure Deployment Walkthrough

> Complete guide to deploy all 7 microservices to Azure using GitHub Actions CI/CD.
> **Your choices**: Multi-repo | OpenAI | No Redis | Central India | Auto-generated URLs

---

## What I Changed (Code Fixes)

### ✅ Fixed 3 Corrupted `requirements.txt` Files

These files had null byte corruption that would have broken `pip install` in CI:

```diff:requirements.txt
# ── Core framework ────────────────────────────────────────────────────────────
fastapi>=0.111.0
uvicorn[standard]>=0.29.0
pydantic>=2.7.0
pydantic-settings>=2.2.0

# ── HTTP client ───────────────────────────────────────────────────────────────
httpx>=0.27.0
boto3>=1.34.0

# ── Redis async client ────────────────────────────────────────────────────────
redis[asyncio]>=5.0.0

# ── LLM SDKs ─────────────────────────────────────────────────────────────────
langchain-core>=0.2.0
langchain-groq>=0.1.0
langchain-openai>=0.1.0
langchain-anthropic>=0.1.0
langchain-ollama>=0.1.0

# ── Testing ───────────────────────────────────────────────────────────────────
pytest>=8.0.0
pytest-asyncio>=0.23.0
a
