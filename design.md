# ALMOND - Design Document

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        CLI Interface                         │
│                    (Python-based Tool)                       │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                      Query Processor                         │
│              (Parse, Normalize, Route Query)                 │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                   Three-Tier Search Engine                   │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Tier 1:    │  │   Tier 2:    │  │   Tier 3:    │      │
│  │ Team History │→ │ Smart Cache  │→ │ RLM Generator│      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                    Execution & Verification                  │
│                  (Sandboxed Python REPL)                     │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                      Response Formatter                      │
│         (PR Links, Code Snippets, Confidence Scores)         │
└─────────────────────────────────────────────────────────────┘
```

## Component Design

### 1. CLI Interface

**Purpose:** User-facing command-line tool for querying ALMOND

**Components:**
- Command parser (argparse/click)
- Interactive prompt mode
- Output formatter (rich/colorama for formatting)

**Commands:**
```bash
almond query "Error: Connection timeout on database"
almond search --pr 402
almond history --since "2024-01-01"
almond stats
```

**Input Format:**
- Natural language error descriptions
- Stack traces
- Error codes
- PR numbers for specific lookups

### 2. Query Processor

**Purpose:** Normalize and prepare queries for search

**Functions:**
- Extract error messages from stack traces
- Normalize error formats
- Identify key terms and patterns
- Route to appropriate tier

**Processing Pipeline:**
```python
query → extract_error() → normalize() → extract_context() → route_to_tier()
```

### 3. Three-Tier Search Engine

#### Tier 1: Team History Search

**Data Sources:**
- Git repository (commits, branches)
- GitHub/GitLab API (PRs, issues, comments)
- Deployment logs
- CI/CD pipeline logs

**Search Strategy:**
1. Semantic search using vector embeddings
2. Keyword matching on error messages
3. Temporal analysis (when did error start?)
4. Causal analysis (what changed before error?)

**Data Structure:**
```python
{
  "pr_number": 402,
  "author": "Sarah Chen",
  "timestamp": "2024-08-15T10:30:00Z",
  "error_pattern": "Connection timeout",
  "fix_description": "Increased connection pool size",
  "files_changed": ["db/config.py"],
  "related_prs": [401, 405],
  "success_rate": 1.0
}
```

#### Tier 2: Smart Cache

**Purpose:** Store and learn from AI-generated solutions

**Cache Entry:**
```python
{
  "query_hash": "abc123...",
  "error_pattern": "Connection timeout",
  "solution": "...",
  "confidence": 0.85,
  "times_suggested": 12,
  "times_accepted": 10,
  "success_rate": 0.83,
  "last_used": "2024-09-01T14:20:00Z",
  "feedback": ["worked", "worked", "didn't help", ...]
}
```

**Learning Mechanism:**
- Track acceptance rate
- Decay old solutions
- Promote frequently successful solutions
- Remove consistently rejected solutions

#### Tier 3: RLM Generator

**Purpose:** Generate fresh solutions using Recursive Language Model

**RLM Process:**
1. Analyze problem context
2. Generate initial solution hypothesis
3. Execute verification code in sandbox
4. Evaluate results
5. Refine solution (recursive step)
6. Return verified solution

**Implementation:**
```python
def rlm_solve(problem, max_depth=5):
    if max_depth == 0:
        return best_solution
    
    # Generate hypothesis
    hypothesis = llm.generate(problem)
    
    # Verify in sandbox
    result = sandbox.execute(hypothesis)
    
    if result.success:
        return hypothesis
    else:
        # Recursive refinement
        refined_problem = problem + result.feedback
        return rlm_solve(refined_problem, max_depth - 1)
```

### 4. LLM Integration

**Provider:** Anthropic Claude 3.5 Sonnet

**API Features Used:**
- Context Caching (reduce costs for repeated context)
- Streaming responses
- Function calling for structured outputs

**Prompt Structure:**
```
System: You are ALMOND, a bug-fixing assistant...

Context (cached):
- Repository structure
- Recent PRs
- Common error patterns

User Query:
Error: {error_message}
Stack trace: {stack_trace}
Recent changes: {git_log}

Task: Find similar historical fixes or generate solution
```

**Cost Optimization:**
- Cache repository context (refreshed daily)
- Batch similar queries
- Use smaller models for simple lookups
- Only use Claude for complex reasoning

### 5. Sandboxed Execution Environment

**Purpose:** Safely execute verification code

**Technology:** Docker container with restricted permissions

**Capabilities:**
- Run Python code
- Access mock databases
- Network isolation
- Resource limits (CPU, memory, time)

**Security:**
- No file system write access
- No network access to production
- Timeout after 30 seconds
- Memory limit: 512MB

### 6. Data Storage

#### Vector Database (Pinecone/Weaviate/ChromaDB)
- Store embeddings of error messages and fixes
- Fast semantic search
- Metadata filtering

#### Relational Database (PostgreSQL)
- PR metadata
- User feedback
- Cache entries
- Analytics data

**Schema:**
```sql
CREATE TABLE fixes (
    id SERIAL PRIMARY KEY,
    pr_number INTEGER,
    author VARCHAR(255),
    timestamp TIMESTAMP,
    error_pattern TEXT,
    fix_description TEXT,
    code_changes JSONB,
    embedding VECTOR(1536),
    success_count INTEGER DEFAULT 0
);

CREATE TABLE cache (
    id SERIAL PRIMARY KEY,
    query_hash VARCHAR(64) UNIQUE,
    solution TEXT,
    confidence FLOAT,
    times_suggested INTEGER DEFAULT 0,
    times_accepted INTEGER DEFAULT 0,
    created_at TIMESTAMP,
    last_used TIMESTAMP
);
```

## Data Flow

### Query Flow
1. User submits query via CLI
2. Query processor extracts error pattern
3. Tier 1 searches team history
   - If match found (confidence > 0.9) → Return result
4. Tier 2 checks smart cache
   - If cached solution exists (confidence > 0.7) → Return result
5. Tier 3 generates new solution via RLM
   - Execute in sandbox
   - Verify solution
   - Save to cache
   - Return result

### Indexing Flow
1. Webhook receives PR merge event
2. Extract error patterns from PR description
3. Parse code changes
4. Generate embeddings
5. Store in vector + relational DB
6. Update related cache entries

## API Design

### Internal APIs

```python
class AlmondCore:
    def query(self, error_message: str, context: dict) -> Solution:
        """Main query interface"""
        
    def index_pr(self, pr_number: int) -> bool:
        """Index a new PR"""
        
    def feedback(self, solution_id: str, accepted: bool) -> None:
        """Record user feedback"""
        
    def get_stats(self) -> dict:
        """Get usage statistics"""

class TierOne:
    def search_history(self, query: str) -> List[Fix]:
        """Search team history"""
        
class TierTwo:
    def search_cache(self, query: str) -> Optional[Solution]:
        """Search smart cache"""
        
    def update_cache(self, solution: Solution, feedback: bool) -> None:
        """Update cache with feedback"""
        
class TierThree:
    def generate_solution(self, problem: str) -> Solution:
        """Generate new solution via RLM"""
```

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         Developer's Machine                  │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              ALMOND CLI Tool                          │   │
│  └────────────────────────┬─────────────────────────────┘   │
└───────────────────────────┼─────────────────────────────────┘
                            │ HTTPS
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      Cloud Infrastructure                    │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   API Gateway│  │  Core Engine │  │   Sandbox    │      │
│  │   (FastAPI)  │→ │   (Python)   │→ │   (Docker)   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                            │                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Vector DB   │  │  PostgreSQL  │  │   Redis      │      │
│  │  (ChromaDB)  │  │              │  │   (Cache)    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │         Anthropic Claude API (External)               │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## Technology Stack

### Core
- **Language:** Python 3.11+
- **CLI Framework:** Click or Typer
- **Async:** asyncio, aiohttp

### LLM
- **Provider:** Anthropic Claude 3.5 Sonnet
- **SDK:** anthropic-sdk-python
- **Caching:** Context Caching API

### Storage
- **Vector DB:** ChromaDB (self-hosted) or Pinecone (managed)
- **Relational DB:** PostgreSQL 15+
- **Cache:** Redis 7+

### Execution
- **Sandbox:** Docker containers
- **Runtime:** Python REPL with restricted builtins

### APIs & Integration
- **Git:** GitPython, PyGithub
- **Web Framework:** FastAPI (if building API)
- **Embeddings:** OpenAI embeddings or sentence-transformers

## Security Considerations

1. **API Key Management:** Store in environment variables, never in code
2. **Sandbox Isolation:** No network access, limited resources
3. **Input Validation:** Sanitize all user inputs
4. **Rate Limiting:** Prevent abuse of LLM API
5. **Data Privacy:** No PII in logs or storage
6. **Access Control:** Team-based permissions for sensitive repos

## Performance Optimization

1. **Caching Strategy:**
   - Cache repository context for 24 hours
   - Cache embeddings indefinitely
   - Cache LLM responses for identical queries

2. **Indexing:**
   - Incremental indexing (only new PRs)
   - Background indexing jobs
   - Batch processing for bulk imports

3. **Query Optimization:**
   - Parallel tier searches
   - Early termination on high-confidence matches
   - Pre-computed embeddings

## Monitoring & Analytics

**Metrics to Track:**
- Query response time
- Tier hit rates (Tier 1 vs 2 vs 3)
- Solution acceptance rate
- Cost per query
- User satisfaction scores

**Logging:**
- All queries and responses
- Tier routing decisions
- LLM API calls and costs
- User feedback

## Future Enhancements

1. **Multi-language Support:** Extend beyond Python to JavaScript, Java, Go
2. **IDE Integration:** VS Code extension, IntelliJ plugin
3. **Proactive Monitoring:** Alert before errors occur based on patterns
4. **Cross-team Learning:** Share anonymized solutions across organizations
5. **Visual Interface:** Web dashboard for analytics and management
