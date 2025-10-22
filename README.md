# robaicrawler

**Web Content Extraction Service for RobAI Tools**

robaicrawler is the foundational web scraping service in the RobAI Tools ecosystem, powered by [Crawl4AI 0.7.3](https://github.com/unclecode/crawl4ai). It extracts, cleans, and transforms web content into structured data for downstream AI processing.

## Overview

robaicrawler serves as the **primary data ingestion layer** for the entire RAG (Retrieval-Augmented Generation) and Knowledge Graph pipeline. It converts raw HTML from any URL into clean, AI-ready markdown content that can be:

- Embedded for semantic search
- Processed into knowledge graphs
- Queried through the RAG API
- Displayed in the chat interface

### Position in the Ecosystem

```
┌─────────────────────────────────────────────────────────────┐
│                    RobAI Tools Pipeline                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [robaicrawler] ──→ [robaitragmcp] ──→ [robairagapi]        │
│       ↓                    ↓                                │
│   Extract                Store                              │
│   Content             Embeddings                            │
│                            ↓                                │
│                    [robaikg/kg-service]                     │
│                            ↓                                │
│                        [Neo4j KG]                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Service Level**: Level 0 (Core Infrastructure)
- **No dependencies** - Starts independently
- **Critical dependency for**: neo4j, kg-service, robaitragmcp, robairagapi
- **Failure impact**: Blocks content ingestion pipeline

## What It Provides

### Core Capabilities

1. **AI-Powered Web Scraping**
   - Extracts clean content from any URL
   - Handles JavaScript-rendered pages
   - Converts HTML to markdown format
   - Removes ads, navigation, and boilerplate

2. **High-Performance Crawling**
   - Concurrent request handling
   - Efficient resource usage
   - Browser engine integration (headless Chrome)
   - Shared memory optimization (1GB)

3. **Stateless Microservice**
   - No persistent storage
   - RESTful API interface
   - Health check endpoints
   - CORS-enabled for web access

4. **Content Transformation**
   - HTML → Markdown conversion
   - Metadata extraction (title, description, images)
   - Link extraction and normalization
   - Clean text output for LLM processing

### Integration Points

robaicrawler integrates with the RobAI ecosystem through:

1. **Direct API Calls**
   - `robaitragmcp` calls `/crawl` endpoint
   - Returns structured content for storage
   - Enables on-demand URL processing

2. **Health Monitoring**
   - Dependency health checks ensure availability
   - Services wait for `service_healthy` condition
   - 30-second health check intervals

3. **Shared Environment**
   - `CRAWL4AI_URL=http://localhost:11235`
   - All services reference the same endpoint
   - Host networking for low-latency communication

## Service Configuration

### Docker Container

```yaml
Service Name:    crawl4ai
Container Name:  robaicrawler
Image:           unclecode/crawl4ai:0.7.3
Port:            11235 (HTTP)
Memory:          1GB shared memory
Restart Policy:  unless-stopped
```

### Network & Ports

| Port  | Protocol | Purpose              | Exposed To        |
|-------|----------|----------------------|-------------------|
| 11235 | HTTP     | Crawl4AI API         | Host + Containers |

**Network Mode**: Host networking (`network_mode: "host"`)
- Services communicate via `localhost:11235`
- No Docker network isolation
- Optimized for low-latency communication

### Environment Variables

```bash
# Crawl4AI Service URL (referenced by other services)
CRAWL4AI_URL=http://localhost:11235

# CORS Configuration
CORS_ORIGINS=http://192.168.10.50:80,http://192.168.10.50,http://localhost:80,http://localhost,*

# Logging
LOG_LEVEL=INFO
```

### Health Checks

```yaml
Health Check:
  Command:      curl -f http://localhost:11235/
  Interval:     30s
  Timeout:      10s
  Retries:      3
  Start Period: 60s
```

**Health Status**: Required for dependent services
- `neo4j` waits for `crawl4ai: service_healthy`
- `robaitragmcp` waits for `crawl4ai: service_healthy`

## API Endpoints

### Base URL
```
http://localhost:11235
```

### Endpoints

#### `GET /`
**Health Check**

Returns service status.

```bash
curl http://localhost:11235/
```

**Response**: `200 OK`

---

#### `POST /crawl`
**Crawl URL**

Extracts content from a given URL.

**Request Body**:
```json
{
  "url": "https://example.com",
  "options": {
    "timeout": 30,
    "wait_for": "domcontentloaded"
  }
}
```

**Response**:
```json
{
  "url": "https://example.com",
  "title": "Example Domain",
  "markdown": "# Example Domain\n\nThis domain is for use...",
  "html": "<html>...</html>",
  "metadata": {
    "description": "Example site",
    "images": [...],
    "links": [...]
  }
}
```

**Example**:
```bash
curl -X POST http://localhost:11235/crawl \
  -H "Content-Type: application/json" \
  -d '{"url": "https://docs.python.org/3/library/asyncio.html"}'
```

## Startup & Dependencies

### Service Start Order

```
Level 0: crawl4ai, vllm-qwen3 (parallel start)
         ↓
Level 1: neo4j (waits for crawl4ai healthy)
         ↓
Level 2: kg-service (waits for neo4j + vllm healthy)
         ↓
Level 3: robaitragmcp (waits for crawl4ai + kg-service healthy)
         ↓
Level 4: robairagapi (waits for robaitragmcp healthy)
         ↓
Level 5: open-webui (waits for robairagapi started)
```

### Starting the Service

**Standalone**:
```bash
docker compose up -d crawl4ai
```

**With dependencies** (starts entire stack):
```bash
docker compose up -d
```

**Check status**:
```bash
docker compose ps crawl4ai
docker compose logs -f crawl4ai
```

## Data Flow

### Content Ingestion Pipeline

1. **User/API requests URL crawl**
   ```
   POST /api/v1/crawl → robairagapi
   ```

2. **API forwards to MCP server**
   ```
   robairagapi → robaitragmcp
   ```

3. **MCP server calls robaicrawler**
   ```
   POST http://localhost:11235/crawl
   ```

4. **robaicrawler extracts content**
   - Fetches URL with headless browser
   - Executes JavaScript
   - Cleans and converts to markdown
   - Returns structured data

5. **MCP server processes response**
   - Stores raw content in SQLite (`crawled_content` table)
   - Generates 384-dim embeddings (sentence-transformers)
   - Queues for KG extraction
   - Returns success to API

6. **KG service processes content** (async)
   - Extracts entities (GLiNER + LLM)
   - Identifies relationships
   - Stores in Neo4j graph database

### Storage Locations

| Data Type        | Storage Location                  | Service Responsible |
|------------------|-----------------------------------|---------------------|
| Raw HTML         | Not stored by robaicrawler        | -                   |
| Markdown Content | `/data/crawl4ai_rag.db` (SQLite)  | robaitragmcp        |
| Embeddings       | `/data/crawl4ai_rag.db` (vec0)    | robaitragmcp        |
| Knowledge Graph  | Neo4j (`/data`)                   | robaikg             |

**Note**: robaicrawler is **stateless** - it does not persist any data itself.

## Resource Requirements

### Minimum
- **Memory**: 1GB (shared memory for browser engine)
- **CPU**: 1 core
- **Disk**: Minimal (no persistent storage)

### Recommended
- **Memory**: 2GB (for concurrent crawling)
- **CPU**: 2 cores
- **Network**: Low latency to target websites

### Browser Engine
- Uses headless Chrome via Crawl4AI
- Requires `shm_size: '1gb'` for shared memory
- Handles JavaScript rendering

## Troubleshooting

### Service Won't Start

**Check logs**:
```bash
docker compose logs crawl4ai
```

**Common issues**:
- Insufficient shared memory → Increase `shm_size` in docker-compose.yml
- Port 11235 in use → Check for conflicting services
- Image pull failure → Check Docker Hub connectivity

### Health Check Failing

**Verify endpoint**:
```bash
curl -v http://localhost:11235/
```

**Expected response**: `200 OK`

**If failing**:
1. Check if container is running: `docker compose ps crawl4ai`
2. Check internal health: `docker exec robaicrawler curl localhost:11235/`
3. Restart service: `docker compose restart crawl4ai`

### Crawl Requests Timing Out

**Increase timeout in request**:
```json
{
  "url": "https://slow-site.com",
  "options": {
    "timeout": 60
  }
}
```

**Check crawler logs**:
```bash
docker compose logs -f crawl4ai | grep ERROR
```

### Dependent Services Not Starting

**Verify health status**:
```bash
docker compose ps
```

**If crawl4ai shows "unhealthy"**:
1. Check health check endpoint: `curl http://localhost:11235/`
2. Restart service: `docker compose restart crawl4ai`
3. Wait for health check to pass (up to 60s start period)

## Performance Optimization

### Concurrent Crawling
Crawl4AI supports concurrent requests. Downstream services can parallelize crawl operations:

```python
# In robaitragmcp or custom client
import asyncio
import httpx

async def crawl_urls(urls):
    async with httpx.AsyncClient() as client:
        tasks = [
            client.post("http://localhost:11235/crawl", json={"url": url})
            for url in urls
        ]
        return await asyncio.gather(*tasks)
```

### Caching Strategy
robaicrawler doesn't cache results. Implement caching in `robaitragmcp`:
- Check SQLite for existing content (by URL hash)
- Skip crawl if content is fresh (< 24 hours)
- Force re-crawl with `force_refresh` parameter

### Resource Limits
Adjust in `docker-compose.yml`:
```yaml
crawl4ai:
  deploy:
    resources:
      limits:
        memory: 2G
        cpus: '2'
```

## Security Considerations

### Network Exposure
- Service is exposed on `localhost:11235`
- **Not recommended** for public internet exposure
- Use reverse proxy with authentication if needed

### CORS Configuration
Current CORS setting allows all origins (`*`):
```yaml
CORS_ORIGINS=*
```

**For production**, restrict to known origins:
```yaml
CORS_ORIGINS=https://your-domain.com,https://app.your-domain.com
```

### Rate Limiting
robaicrawler doesn't implement rate limiting. Add in reverse proxy or `robairagapi` layer.

## Version Information

- **Crawl4AI Version**: 0.7.3
- **Base Image**: `unclecode/crawl4ai:0.7.3`
- **Service Version**: 1.0.0 (RobAI Tools integration)

## Related Services

- **[robaitragmcp](../robaitragmcp/README.md)** - Primary consumer of crawl data
- **[robaikg](../robaikg/README.md)** - Processes content into knowledge graphs
- **[robairagapi](../robairagapi/README.md)** - REST API for crawl operations
- **[robaidata](../robaidata/README.md)** - Shared data storage

## External Resources

- [Crawl4AI Documentation](https://github.com/unclecode/crawl4ai)
- [Docker Hub: unclecode/crawl4ai](https://hub.docker.com/r/unclecode/crawl4ai)
- [RobAI Tools Main Documentation](../../docs/)

## Contributing

This service is part of the RobAI Tools monorepo. For contributions:

1. Test changes in isolation: `docker compose up -d crawl4ai`
2. Verify health checks pass: `docker compose ps crawl4ai`
3. Test integration with dependent services
4. Submit PR to main repository

## License

Part of RobAI Tools - See main repository for license information.
