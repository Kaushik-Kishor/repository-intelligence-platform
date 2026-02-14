# Design Document: Repository Intelligence & Contribution Strategy Platform

## Overview

The Repository Intelligence & Contribution Strategy Platform is a web-based system that combines static code analysis, graph algorithms, and LLM-powered reasoning to accelerate developer onboarding for large GitHub repositories. The platform provides complexity heatmaps, dependency analysis, personalized skill profiling, and intelligent contribution recommendations.

### Design Philosophy

This design prioritizes:
- **Modularity**: Independent components with clear interfaces for parallel development and testing
- **Incremental Value**: Core features work independently; advanced features enhance the experience
- **Performance**: Efficient algorithms and caching for responsive user experience
- **Security**: OAuth best practices and encrypted credential storage
- **Extensibility**: Architecture supports future GitHub App integration and additional data sources

### Phase 1 MVP Scope

This design focuses on a hackathon-ready prototype demonstrating:
- Repository analysis with complexity and centrality metrics
- GitHub OAuth-based skill profiling with manual fallback
- Personalized complexity adjustments and contribution recommendations
- RAG-based query system for repository understanding
- Web UI for visualization and interaction

Future enhancements (GitHub App integration, real-time monitoring, team features) are noted but not detailed.


## Architecture

### System Architecture Overview

The platform follows a modular architecture with five core engines and supporting infrastructure:

```
┌─────────────────────────────────────────────────────────────────┐
│                         Web Frontend (React)                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Dashboard   │  │  Heatmap     │  │  Query       │          │
│  │  View        │  │  Visualizer  │  │  Interface   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ HTTPS/REST API
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API Gateway / Router                        │
│              (Authentication, Rate Limiting, Routing)            │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  Repository      │  │  User Skill      │  │  Query Engine    │
│  Intelligence    │  │  Profiling       │  │  (RAG)           │
│  Engine          │  │  Engine          │  │                  │
└──────────────────┘  └──────────────────┘  └──────────────────┘
        │                     │                     │
        │                     │                     │
        ▼                     ▼                     ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  Personalization │  │  Contribution    │  │  Vector Store    │
│  Engine          │  │  Optimization    │  │  (Embeddings)    │
│                  │  │  Engine          │  │                  │
└──────────────────┘  └──────────────────┘  └──────────────────┘
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Data Storage Layer                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  PostgreSQL  │  │  Redis       │  │  S3/Object   │          │
│  │  (Metadata)  │  │  (Cache)     │  │  Storage     │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      External Services                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  GitHub API  │  │  OpenAI API  │  │  Pinecone/   │          │
│  │              │  │  (LLM)       │  │  Weaviate    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

**Web Frontend**
- Single-page React application
- Visualizes heatmaps, dependency graphs, and recommendations
- Handles OAuth flow initiation
- Provides structured query interface

**API Gateway**
- Routes requests to appropriate engines
- Validates JWT tokens from OAuth flow
- Implements rate limiting (per-user and global)
- Handles CORS and security headers

**Repository Intelligence Engine**
- Fetches repository data from GitHub API
- Parses source files to extract imports/dependencies
- Constructs dependency graph (adjacency list representation)
- Computes complexity scores (LOC, cyclomatic complexity, nesting depth)
- Calculates module centrality using PageRank
- Generates general complexity heatmap

**User Skill Profiling Engine**
- Manages GitHub OAuth flow
- Retrieves user's GitHub contribution history
- Infers skill levels from language usage statistics
- Accepts manual skill input
- Combines inferred and manual skills (max confidence)
- Stores encrypted OAuth tokens

**Personalization Engine**
- Adjusts complexity scores based on user skills
- Computes suitability confidence scores
- Generates personalized heatmaps
- Estimates cognitive load per file

**Contribution Optimization Engine**
- Ranks files by suitability confidence
- Computes risk scores (centrality + complexity + skill match)
- Generates contribution paths (progressive complexity)
- Prioritizes files with recent activity and open issues

**Query Engine (RAG)**
- Embeds repository documentation and code
- Performs semantic search for relevant context
- Generates LLM responses with citations
- Handles structured query panels and custom queries

**Data Storage Layer**
- PostgreSQL: User profiles, analysis metadata, skill profiles
- Redis: Analysis result cache, session data
- S3/Object Storage: Large analysis artifacts (dependency graphs, embeddings)


## Components and Interfaces

### 1. Repository Intelligence Engine

**Purpose**: Extract and analyze repository structure, dependencies, and complexity.

**Key Interfaces**:

```typescript
interface RepositoryIntelligenceEngine {
  // Initiate repository analysis
  analyzeRepository(repoUrl: string, userId: string): Promise<AnalysisJob>
  
  // Get analysis status
  getAnalysisStatus(jobId: string): Promise<AnalysisStatus>
  
  // Retrieve completed analysis
  getAnalysis(repoUrl: string): Promise<RepositoryAnalysis>
  
  // Get dependency graph
  getDependencyGraph(repoUrl: string): Promise<DependencyGraph>
  
  // Get complexity heatmap
  getComplexityHeatmap(repoUrl: string): Promise<Heatmap>
}

interface RepositoryAnalysis {
  repoUrl: string
  analyzedAt: Date
  fileCount: number
  languageDistribution: Map<string, number>
  files: FileMetadata[]
  dependencyGraph: DependencyGraph
  centralityScores: Map<string, number>
  complexityScores: Map<string, number>
}

interface FileMetadata {
  path: string
  language: string
  linesOfCode: number
  complexity: number
  centrality: number
  lastModified: Date
  commitCount: number
}

interface DependencyGraph {
  nodes: string[]  // file paths
  edges: [string, string][]  // [from, to]
  adjacencyList: Map<string, string[]>
}
```

**Internal Components**:

1. **GitHub Fetcher**
   - Uses GitHub REST API and GraphQL API
   - Retrieves file tree, commit history, language stats
   - Implements pagination for large repositories
   - Handles rate limiting with exponential backoff

2. **Dependency Parser**
   - Language-specific parsers for import statements
   - Supports: Python (import/from), JavaScript/TypeScript (import/require), Java (import), Go (import), Rust (use)
   - Distinguishes internal vs external dependencies
   - Detects circular dependencies

3. **Complexity Analyzer**
   - Computes cyclomatic complexity using AST parsing
   - Measures nesting depth
   - Applies size penalty for files >500 LOC
   - Normalizes to 0-100 scale: `complexity = min(100, (0.4 * cyclomatic + 0.3 * nesting + 0.3 * sizePenalty))`

4. **Centrality Calculator**
   - Implements PageRank algorithm on dependency graph
   - Iterates until convergence (delta < 0.001) or max 100 iterations
   - Normalizes scores to 0-100 range
   - Formula: `PR(A) = (1-d) + d * Σ(PR(Ti) / C(Ti))` where d=0.85

5. **Heatmap Generator**
   - Aggregates file scores by directory
   - Generates color gradient (green: 0-33, yellow: 34-66, red: 67-100)
   - Supports drill-down navigation

**Data Flow**:
1. User submits repository URL
2. GitHub Fetcher retrieves file tree and metadata
3. Dependency Parser analyzes each source file
4. Dependency Graph constructed from parsed imports
5. Centrality Calculator runs PageRank on graph
6. Complexity Analyzer processes each file
7. Results stored in PostgreSQL and cached in Redis
8. Heatmap Generator creates visualization data


### 2. User Skill Profiling Engine

**Purpose**: Infer and manage user skill profiles through GitHub OAuth and manual input.

**Key Interfaces**:

```typescript
interface UserSkillProfilingEngine {
  // Initiate GitHub OAuth flow
  initiateOAuth(): Promise<OAuthUrl>
  
  // Handle OAuth callback
  handleOAuthCallback(code: string): Promise<UserSession>
  
  // Infer skills from GitHub activity
  inferSkills(userId: string): Promise<SkillProfile>
  
  // Update manual skills
  updateManualSkills(userId: string, skills: ManualSkill[]): Promise<SkillProfile>
  
  // Get combined skill profile
  getSkillProfile(userId: string): Promise<SkillProfile>
  
  // Revoke OAuth token
  revokeToken(userId: string): Promise<void>
}

interface SkillProfile {
  userId: string
  skills: Map<string, SkillLevel>
  inferredSkills: Map<string, SkillLevel>
  manualSkills: Map<string, SkillLevel>
  lastUpdated: Date
}

interface SkillLevel {
  language: string
  proficiency: 'Beginner' | 'Intermediate' | 'Advanced' | 'Expert'
  confidenceScore: number  // 0.25, 0.5, 0.75, 1.0
  source: 'inferred' | 'manual' | 'combined'
  linesContributed?: number  // for inferred skills
}

interface ManualSkill {
  language: string
  proficiency: 'Beginner' | 'Intermediate' | 'Advanced' | 'Expert'
}
```

**Internal Components**:

1. **OAuth Manager**
   - Implements GitHub OAuth 2.0 flow
   - Requested scopes: `read:user`, `repo` (public only)
   - Generates secure state parameter for CSRF protection
   - Exchanges authorization code for access token
   - Stores encrypted tokens in PostgreSQL

2. **GitHub Activity Analyzer**
   - Fetches user's public repositories
   - Retrieves language statistics from GitHub API
   - Analyzes commit history (past 24 months)
   - Computes lines contributed per language
   - Maps contribution levels to proficiency:
     - Expert: >10,000 lines
     - Advanced: 5,000-10,000 lines
     - Intermediate: 1,000-5,000 lines
     - Beginner: <1,000 lines

3. **Skill Combiner**
   - Merges inferred and manual skills
   - Uses maximum confidence score when both exist
   - Prioritizes manual input for user control

4. **Token Encryptor**
   - Uses AES-256-GCM encryption
   - Generates unique encryption key per user
   - Stores encrypted tokens with IV and auth tag
   - Implements automatic token expiration (30 days)

**Data Flow**:
1. User clicks "Connect GitHub"
2. OAuth Manager redirects to GitHub authorization
3. User approves, GitHub redirects back with code
4. OAuth Manager exchanges code for access token
5. Token Encryptor encrypts and stores token
6. GitHub Activity Analyzer fetches contribution data
7. Skill inference algorithm computes proficiency levels
8. Skill Combiner merges with any manual skills
9. Final profile stored in PostgreSQL


### 3. Personalization Engine

**Purpose**: Adjust complexity metrics based on user skills and compute suitability scores.

**Key Interfaces**:

```typescript
interface PersonalizationEngine {
  // Generate personalized heatmap
  getPersonalizedHeatmap(repoUrl: string, userId: string): Promise<Heatmap>
  
  // Compute suitability scores for all files
  computeSuitabilityScores(repoUrl: string, userId: string): Promise<Map<string, SuitabilityScore>>
  
  // Estimate cognitive load
  estimateCognitiveLoad(filePath: string, userId: string): Promise<CognitiveLoad>
}

interface SuitabilityScore {
  filePath: string
  score: number  // 0-1
  confidence: 'High' | 'Medium' | 'Low'
  adjustedComplexity: number
  skillMatch: boolean
  matchedLanguage?: string
}

interface CognitiveLoad {
  filePath: string
  loadScore: number  // 1-10
  estimatedHours: number
  factors: {
    complexity: number
    dependencies: number
    size: number
    skillMatch: number
  }
}
```

**Algorithms**:

1. **Complexity Adjustment Algorithm**
   ```
   adjustedComplexity = baseComplexity * adjustmentFactor
   
   adjustmentFactor = {
     0.6  if user has Expert skill in file's language
     0.75 if user has Advanced skill
     0.9  if user has Intermediate skill
     1.0  if user has Beginner skill
     1.2  if user has no skill in file's language
   }
   
   adjustedComplexity = clamp(adjustedComplexity, 0, 100)
   ```

2. **Suitability Confidence Algorithm**
   ```
   suitabilityScore = (skillConfidence * 0.6) + ((100 - adjustedComplexity) / 100 * 0.4)
   
   confidence = {
     'High'   if suitabilityScore > 0.7
     'Medium' if 0.4 <= suitabilityScore <= 0.7
     'Low'    if suitabilityScore < 0.4
   }
   ```

3. **Cognitive Load Algorithm**
   ```
   cognitiveLoad = (
     complexity * 0.35 +
     dependencyCount * 0.25 +
     (linesOfCode / 1000) * 0.2 +
     (1 - skillConfidence) * 0.2
   )
   
   loadScore = clamp(round(cognitiveLoad), 1, 10)
   
   estimatedHours = {
     1-3:  1-2 hours
     4-6:  3-6 hours
     7-10: >6 hours
   }
   ```

**Data Flow**:
1. Retrieve base analysis from Repository Intelligence Engine
2. Fetch user skill profile
3. For each file, compute adjusted complexity
4. Calculate suitability scores
5. Generate personalized heatmap with adjusted scores
6. Cache results in Redis (TTL: 1 hour)


### 4. Contribution Optimization Engine

**Purpose**: Rank contribution opportunities and generate learning paths.

**Key Interfaces**:

```typescript
interface ContributionOptimizationEngine {
  // Get ranked contribution opportunities
  getRecommendations(repoUrl: string, userId: string, limit: number): Promise<Recommendation[]>
  
  // Compute risk assessment
  assessRisk(filePath: string, userId: string): Promise<RiskAssessment>
  
  // Generate contribution path
  generateContributionPath(repoUrl: string, userId: string, goal?: ContributionGoal): Promise<ContributionPath>
}

interface Recommendation {
  filePath: string
  suitabilityScore: number
  adjustedComplexity: number
  centrality: number
  riskLevel: 'Low' | 'Medium' | 'High'
  lastModified: Date
  hasOpenIssues: boolean
  estimatedHours: number
  rationale: string
}

interface RiskAssessment {
  filePath: string
  riskLevel: 'Low' | 'Medium' | 'High'
  factors: {
    centrality: number
    complexity: number
    skillMismatch: number
  }
  recommendation: string
}

interface ContributionPath {
  steps: PathStep[]
  totalEstimatedHours: number
  milestones: Milestone[]
}

interface PathStep {
  order: number
  filePath: string
  rationale: string
  estimatedHours: number
  dependencies: string[]
  concepts: string[]
}

type ContributionGoal = 'BugFixes' | 'NewFeatures' | 'Documentation' | 'Refactoring'
```

**Algorithms**:

1. **Recommendation Ranking Algorithm**
   ```
   rankingScore = (
     suitabilityScore * 0.5 +
     recencyBonus * 0.2 +
     issueBonus * 0.15 +
     (1 - riskPenalty) * 0.15
   )
   
   recencyBonus = {
     1.0 if lastModified within 30 days
     0.5 if lastModified within 90 days
     0.0 otherwise
   }
   
   issueBonus = 1.0 if file has open issues, else 0.0
   
   riskPenalty = {
     0.3 if High risk
     0.1 if Medium risk
     0.0 if Low risk
   }
   
   Sort files by rankingScore descending, return top N
   ```

2. **Risk Assessment Algorithm**
   ```
   riskScore = (
     (centrality / 100) * 0.4 +
     (complexity / 100) * 0.3 +
     (1 - skillConfidence) * 0.3
   )
   
   riskLevel = {
     'High'   if riskScore > 0.7
     'Medium' if 0.4 <= riskScore <= 0.7
     'Low'    if riskScore < 0.4
   }
   ```

3. **Contribution Path Generation Algorithm**
   ```
   1. Filter files by suitabilityScore > 0.6
   2. Sort by adjustedComplexity ascending
   3. Build dependency-aware sequence:
      - Start with files having no dependencies
      - Add files whose dependencies are already in path
      - Ensure progressive complexity increase
   4. Select 10-15 files spanning complexity range
   5. Add milestones every 3-4 files
   6. Compute total estimated time
   ```

**Data Flow**:
1. Retrieve personalized analysis from Personalization Engine
2. Fetch recent commit activity and open issues from GitHub
3. Compute ranking scores for all files
4. Sort and filter top recommendations
5. For contribution path, apply dependency-aware sorting
6. Return ranked results with rationale


### 5. Query Engine (RAG)

**Purpose**: Answer user questions using retrieval-augmented generation.

**Key Interfaces**:

```typescript
interface QueryEngine {
  // Answer structured query
  answerStructuredQuery(repoUrl: string, panel: QueryPanel, userId: string): Promise<QueryResponse>
  
  // Answer custom scoped query
  answerScopedQuery(repoUrl: string, scope: string[], question: string, userId: string): Promise<QueryResponse>
  
  // Embed repository content
  embedRepository(repoUrl: string): Promise<EmbeddingJob>
}

type QueryPanel = 'UnderstandRepository' | 'LearnTechStack' | 'ContributionPath'

interface QueryResponse {
  answer: string
  citations: Citation[]
  confidence: number
  inferredContent: string[]
  relatedFiles: string[]
}

interface Citation {
  filePath: string
  lineRange?: [number, number]
  snippet: string
  relevanceScore: number
}
```

**RAG Architecture**:

```
┌─────────────────────────────────────────────────────────────┐
│                    Query Processing                          │
│  1. Query Understanding                                      │
│  2. Query Expansion (add context from user profile)         │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Context Retrieval                           │
│  ┌────────────────────────────────────────────────────┐     │
│  │  Vector Store (Pinecone/Weaviate)                  │     │
│  │  - Code embeddings (function/class level)          │     │
│  │  - Documentation embeddings (section level)        │     │
│  │  - README and markdown embeddings                  │     │
│  └────────────────────────────────────────────────────┘     │
│                                                              │
│  Semantic Search: Retrieve top 10 most relevant chunks      │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Context Ranking                             │
│  1. Re-rank by relevance (cross-encoder)                    │
│  2. Filter by minimum relevance threshold (0.6)             │
│  3. Deduplicate similar chunks                              │
│  4. Select top 5 chunks                                     │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Prompt Construction                         │
│  System: "You are a code analysis assistant..."             │
│  Context: [Retrieved chunks with file paths]                │
│  User Query: [Original question]                            │
│  Instructions: "Cite sources, mark inferred content"        │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  LLM Generation                              │
│  Model: GPT-4 or Claude 2                                   │
│  Max tokens: 1000                                            │
│  Temperature: 0.3 (factual responses)                       │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Response Post-Processing                    │
│  1. Extract citations from response                         │
│  2. Validate citations against retrieved context            │
│  3. Mark inferred vs verified content                       │
│  4. Format with syntax highlighting                         │
│  5. Limit to 500 words                                      │
└─────────────────────────────────────────────────────────────┘
```

**Embedding Strategy**:

1. **Code Embeddings**
   - Embed at function/class level (not entire files)
   - Include function signature, docstring, and body
   - Model: OpenAI text-embedding-ada-002 or Cohere embed-v3
   - Chunk size: ~500 tokens with 50 token overlap

2. **Documentation Embeddings**
   - Embed README, CONTRIBUTING, docs/ directory
   - Split by markdown sections
   - Include section headers in chunks for context

3. **Metadata Enrichment**
   - Store file path, language, complexity, centrality with embeddings
   - Enable filtering by metadata during retrieval

**Structured Query Panels**:

1. **Understand Repository**
   - Pre-defined questions: "What does this repository do?", "What is the architecture?", "What are the main modules?"
   - Retrieval focus: README, architecture docs, high-centrality modules
   - Response includes: Purpose, architecture diagram (text), key modules with centrality scores

2. **Learn Tech Stack**
   - Analyzes language distribution and framework usage
   - Compares against user skill profile
   - Identifies gaps and provides learning resources
   - Response includes: Core technologies (>30% usage), secondary (10-30%), minor (<10%), recommended learning order

3. **Personalized Contribution Path**
   - Combines RAG with Contribution Optimization Engine
   - Generates narrative explanation of recommended path
   - Includes specific file recommendations with rationale
   - Response includes: Path overview, file sequence, estimated timeline, prerequisite learning

**Scoped Custom Query**:
- User selects specific files/modules
- Retrieval limited to selected scope + immediate dependencies
- Supports questions about: functionality, dependencies, complexity, recent changes, contribution opportunities


## Data Models

### Database Schema (PostgreSQL)

```sql
-- Users table
CREATE TABLE users (
  user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  github_username VARCHAR(255) UNIQUE,
  email VARCHAR(255),
  created_at TIMESTAMP DEFAULT NOW(),
  last_login TIMESTAMP,
  oauth_token_encrypted BYTEA,
  oauth_token_iv BYTEA,
  oauth_token_expires_at TIMESTAMP
);

-- Skill profiles table
CREATE TABLE skill_profiles (
  profile_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
  language VARCHAR(100),
  proficiency VARCHAR(20) CHECK (proficiency IN ('Beginner', 'Intermediate', 'Advanced', 'Expert')),
  confidence_score DECIMAL(3,2) CHECK (confidence_score BETWEEN 0 AND 1),
  source VARCHAR(20) CHECK (source IN ('inferred', 'manual', 'combined')),
  lines_contributed INTEGER,
  last_updated TIMESTAMP DEFAULT NOW(),
  UNIQUE(user_id, language)
);

-- Repository analyses table
CREATE TABLE repository_analyses (
  analysis_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  repo_url VARCHAR(500) UNIQUE NOT NULL,
  repo_owner VARCHAR(255),
  repo_name VARCHAR(255),
  analyzed_at TIMESTAMP DEFAULT NOW(),
  file_count INTEGER,
  total_loc INTEGER,
  status VARCHAR(20) CHECK (status IN ('pending', 'processing', 'completed', 'failed')),
  error_message TEXT,
  cache_expires_at TIMESTAMP
);

-- File metadata table
CREATE TABLE file_metadata (
  file_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  analysis_id UUID REFERENCES repository_analyses(analysis_id) ON DELETE CASCADE,
  file_path VARCHAR(1000) NOT NULL,
  language VARCHAR(100),
  lines_of_code INTEGER,
  complexity_score DECIMAL(5,2),
  centrality_score DECIMAL(5,2),
  last_modified TIMESTAMP,
  commit_count INTEGER,
  has_open_issues BOOLEAN DEFAULT FALSE,
  UNIQUE(analysis_id, file_path)
);

-- Dependencies table (for dependency graph)
CREATE TABLE dependencies (
  dependency_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  analysis_id UUID REFERENCES repository_analyses(analysis_id) ON DELETE CASCADE,
  from_file VARCHAR(1000),
  to_file VARCHAR(1000),
  is_external BOOLEAN DEFAULT FALSE,
  UNIQUE(analysis_id, from_file, to_file)
);

-- Personalized analyses table
CREATE TABLE personalized_analyses (
  personalized_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
  analysis_id UUID REFERENCES repository_analyses(analysis_id) ON DELETE CASCADE,
  file_path VARCHAR(1000),
  adjusted_complexity DECIMAL(5,2),
  suitability_score DECIMAL(3,2),
  cognitive_load INTEGER CHECK (cognitive_load BETWEEN 1 AND 10),
  estimated_hours DECIMAL(4,1),
  computed_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(user_id, analysis_id, file_path)
);

-- Analysis jobs table (for async processing)
CREATE TABLE analysis_jobs (
  job_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(user_id),
  repo_url VARCHAR(500),
  status VARCHAR(20) CHECK (status IN ('queued', 'processing', 'completed', 'failed')),
  progress_percentage INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  started_at TIMESTAMP,
  completed_at TIMESTAMP,
  error_message TEXT
);

-- Indexes for performance
CREATE INDEX idx_skill_profiles_user ON skill_profiles(user_id);
CREATE INDEX idx_file_metadata_analysis ON file_metadata(analysis_id);
CREATE INDEX idx_file_metadata_complexity ON file_metadata(complexity_score);
CREATE INDEX idx_dependencies_analysis ON dependencies(analysis_id);
CREATE INDEX idx_personalized_user_analysis ON personalized_analyses(user_id, analysis_id);
CREATE INDEX idx_repo_analyses_url ON repository_analyses(repo_url);
CREATE INDEX idx_analysis_jobs_user ON analysis_jobs(user_id);
```

### Redis Cache Schema

```
# Analysis result cache
analysis:{repo_url} -> JSON (RepositoryAnalysis)
TTL: 6 hours

# Dependency graph cache
graph:{repo_url} -> JSON (DependencyGraph)
TTL: 6 hours

# Personalized analysis cache
personalized:{user_id}:{repo_url} -> JSON (PersonalizedAnalysis)
TTL: 1 hour

# User session cache
session:{session_id} -> JSON (UserSession)
TTL: 24 hours

# Rate limiting
ratelimit:{user_id}:{endpoint} -> counter
TTL: 1 hour

# GitHub API rate limit tracking
github_ratelimit:{user_id} -> JSON {remaining, reset_at}
TTL: 1 hour
```

### Object Storage Structure (S3)

```
embeddings/
  {repo_url_hash}/
    code_embeddings.json
    doc_embeddings.json
    metadata.json

dependency_graphs/
  {repo_url_hash}/
    graph.json
    visualization.json

heatmaps/
  {repo_url_hash}/
    general_heatmap.json
    {user_id}_personalized.json
```


## GitHub OAuth Authentication Flow

### OAuth 2.0 Flow Diagram

```
┌─────────┐                                    ┌─────────────┐
│ Browser │                                    │   GitHub    │
└────┬────┘                                    └──────┬──────┘
     │                                                │
     │  1. Click "Connect GitHub"                    │
     │────────────────────────────────────────►      │
     │                                                │
┌────▼────────────────────────────────────────────────────────┐
│  Frontend: Generate state token, redirect to GitHub OAuth   │
└────┬────────────────────────────────────────────────────────┘
     │
     │  2. Redirect to GitHub authorization
     │     https://github.com/login/oauth/authorize
     │     ?client_id={CLIENT_ID}
     │     &redirect_uri={REDIRECT_URI}
     │     &scope=read:user repo
     │     &state={CSRF_TOKEN}
     │────────────────────────────────────────►
     │                                                │
     │  3. User approves authorization                │
     │                                                │
     │  4. GitHub redirects back with code            │
     │◄────────────────────────────────────────────────
     │     {REDIRECT_URI}?code={AUTH_CODE}&state={CSRF_TOKEN}
     │                                                │
┌────▼────────────────────────────────────────────────────────┐
│  Frontend: Validate state, send code to backend             │
└────┬────────────────────────────────────────────────────────┘
     │
     │  5. POST /api/auth/github/callback
     │     {code, state}
     │────────────────────────────────────────►
     │                                         │
┌────────────────────────────────────────────▼────────────────┐
│  Backend: OAuth Manager                                      │
│  - Validate state token (CSRF protection)                   │
│  - Exchange code for access token                           │
└────┬────────────────────────────────────────────────────────┘
     │
     │  6. POST https://github.com/login/oauth/access_token
     │     {client_id, client_secret, code}
     │────────────────────────────────────────►
     │                                                │
     │  7. Receive access token                       │
     │◄────────────────────────────────────────────────
     │     {access_token, token_type, scope}          │
     │                                                │
┌────▼────────────────────────────────────────────────────────┐
│  Backend: Token Encryptor                                    │
│  - Generate encryption key from user secret                 │
│  - Encrypt token with AES-256-GCM                           │
│  - Store encrypted token, IV, auth tag in database          │
└────┬────────────────────────────────────────────────────────┘
     │
┌────▼────────────────────────────────────────────────────────┐
│  Backend: GitHub Activity Analyzer                           │
│  - Fetch user profile                                       │
│  - Fetch user repositories                                  │
│  - Analyze language statistics                              │
│  - Infer skill levels                                       │
└────┬────────────────────────────────────────────────────────┘
     │
     │  8. Return JWT session token + skill profile
     │◄────────────────────────────────────────
     │     {sessionToken, skillProfile}
     │
┌────▼────────────────────────────────────────────────────────┐
│  Frontend: Store session token, display skill profile       │
└─────────────────────────────────────────────────────────────┘
```

### Security Measures

1. **CSRF Protection**
   - Generate cryptographically random state parameter
   - Store state in session before redirect
   - Validate state matches on callback
   - State expires after 10 minutes

2. **Token Encryption**
   - Algorithm: AES-256-GCM (authenticated encryption)
   - Key derivation: PBKDF2 with user-specific salt
   - Store: encrypted_token, IV, auth_tag, salt
   - Never log or expose tokens in plain text

3. **Token Storage**
   - Encrypted tokens in PostgreSQL
   - Session tokens (JWT) in Redis with 24h TTL
   - Automatic cleanup of expired tokens

4. **Scope Minimization**
   - Request only necessary scopes: `read:user`, `repo` (public only)
   - No write access to repositories
   - No access to private repositories (MVP phase)

5. **Token Revocation**
   - User can revoke access from settings
   - Immediate deletion from database
   - Revoke token with GitHub API
   - Clear all cached data

### Token Refresh Strategy

For MVP (Phase 1):
- GitHub OAuth tokens don't expire automatically
- Implement manual re-authentication when API returns 401
- Display clear message: "GitHub access expired, please reconnect"

For Future (Phase 2):
- Implement refresh token flow
- Automatic token refresh before expiration
- Background job to refresh tokens nearing expiration


## Repository Analysis Pipeline

### Analysis Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Step 1: Job Initialization                                 │
│  - Create analysis job record                               │
│  - Check cache for existing analysis                        │
│  - If cached and fresh, return immediately                  │
│  - Otherwise, queue for processing                          │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 2: Repository Metadata Extraction                     │
│  - Fetch repository info (owner, name, default branch)      │
│  - Get file tree (recursive)                                │
│  - Filter source files (exclude .git, node_modules, etc.)   │
│  - Get language statistics                                  │
│  - Estimate total work (file count)                         │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 3: Batch Processing Setup                             │
│  - Divide files into batches (1000 files per batch)         │
│  - Prioritize: high-centrality files, recently modified     │
│  - Create processing queue                                  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 4: File Content Retrieval (Parallel)                  │
│  - Fetch file contents via GitHub API                       │
│  - Batch requests (up to 50 concurrent)                     │
│  - Handle rate limiting (exponential backoff)               │
│  - Store raw content temporarily                            │
│  - Update progress: 0-30%                                   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 5: Dependency Parsing (Parallel)                      │
│  - Parse each file for import statements                    │
│  - Language-specific parsers:                               │
│    * Python: ast.parse()                                    │
│    * JavaScript/TypeScript: @babel/parser                   │
│    * Java: regex + validation                               │
│    * Go: go/parser                                          │
│    * Rust: syn crate                                        │
│  - Resolve relative imports to absolute paths               │
│  - Distinguish internal vs external dependencies            │
│  - Build edge list for dependency graph                     │
│  - Update progress: 30-50%                                  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 6: Dependency Graph Construction                      │
│  - Build adjacency list from edge list                      │
│  - Detect cycles (Tarjan's algorithm)                       │
│  - Compute in-degree and out-degree for each node           │
│  - Identify isolated nodes (no dependencies)                │
│  - Store graph structure                                    │
│  - Update progress: 50-60%                                  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 7: Centrality Computation                             │
│  - Initialize PageRank scores (1/N for each node)           │
│  - Iterate until convergence:                               │
│    * PR(A) = (1-d) + d * Σ(PR(Ti) / C(Ti))                 │
│    * d = 0.85 (damping factor)                              │
│    * Max 100 iterations or delta < 0.001                    │
│  - Normalize scores to 0-100 range                          │
│  - Update progress: 60-70%                                  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 8: Complexity Analysis (Parallel)                     │
│  - For each file:                                           │
│    * Parse AST                                              │
│    * Compute cyclomatic complexity                          │
│    * Measure nesting depth                                  │
│    * Count lines of code                                    │
│    * Apply size penalty if LOC > 500                        │
│    * Normalize to 0-100 scale                               │
│  - Store complexity scores                                  │
│  - Update progress: 70-85%                                  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 9: Commit History Analysis                            │
│  - Fetch commit history (past 12 months)                    │
│  - Count commits per file                                   │
│  - Identify files with recent activity (30 days)            │
│  - Check for open issues per file (GitHub API)              │
│  - Update progress: 85-95%                                  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 10: Heatmap Generation                                │
│  - Aggregate complexity by directory                        │
│  - Generate color gradient mapping                          │
│  - Create hierarchical structure for drill-down             │
│  - Store heatmap data                                       │
│  - Update progress: 95-100%                                 │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 11: Result Storage and Caching                        │
│  - Store analysis in PostgreSQL                             │
│  - Cache results in Redis (6 hour TTL)                      │
│  - Store dependency graph in S3                             │
│  - Mark job as completed                                    │
│  - Notify user (if async)                                   │
└─────────────────────────────────────────────────────────────┘
```

### Error Handling Strategy

1. **GitHub API Rate Limiting**
   - Monitor rate limit headers: X-RateLimit-Remaining, X-RateLimit-Reset
   - When limit reached: pause processing, display reset time to user
   - Option to queue job for automatic retry after reset
   - Use conditional requests (If-None-Match) to save quota

2. **Parsing Failures**
   - Log parsing errors with file path and error message
   - Continue processing remaining files
   - Mark file as "analysis_failed" in metadata
   - Include partial results in final analysis
   - Display warning to user about incomplete analysis

3. **Network Failures**
   - Retry failed requests up to 3 times
   - Exponential backoff: 1s, 2s, 4s
   - If all retries fail, mark batch as failed
   - Allow user to retry failed batches

4. **Timeout Protection**
   - Set timeout for entire analysis: 5 minutes for <5000 files
   - Set timeout per batch: 30 seconds
   - If timeout occurs, save partial results
   - Allow resumption from last completed batch

### Performance Optimizations

1. **Parallel Processing**
   - Process files in parallel (up to 50 concurrent)
   - Use worker pool for CPU-intensive tasks (AST parsing, complexity)
   - Batch GitHub API requests

2. **Caching Strategy**
   - Cache analysis results for 6 hours
   - Cache dependency graphs separately (rarely change)
   - Invalidate cache on repository push (webhook in future)
   - Use ETags for conditional requests

3. **Incremental Analysis**
   - For large repos (>10,000 files), process in batches
   - Display partial results as batches complete
   - Prioritize high-value files (high centrality, recent activity)

4. **Smart Filtering**
   - Exclude test files from complexity analysis
   - Exclude generated files (detected by patterns)
   - Exclude configuration files (package.json, etc.)
   - Focus on source code files


## Deployment Architecture for MVP

### Infrastructure Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Internet / Users                        │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ HTTPS
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   CDN / CloudFront                           │
│              (Static assets, caching)                        │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  Load Balancer (ALB)                         │
│            (SSL termination, health checks)                  │
└────────────────────────┬────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Frontend   │  │   Frontend   │  │   Frontend   │
│  Container   │  │  Container   │  │  Container   │
│  (React SPA) │  │  (React SPA) │  │  (React SPA) │
└──────────────┘  └──────────────┘  └──────────────┘
                         │
                         │ API calls
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  API Gateway / Router                        │
│         (Authentication, rate limiting, routing)             │
└────────────────────────┬────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Backend    │  │   Backend    │  │   Backend    │
│  Container   │  │  Container   │  │  Container   │
│  (Node.js/   │  │  (Node.js/   │  │  (Node.js/   │
│   Python)    │  │   Python)    │  │   Python)    │
└──────────────┘  └──────────────┘  └──────────────┘
        │                │                │
        └────────────────┼────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  PostgreSQL  │  │    Redis     │  │  S3 Bucket   │
│   (RDS)      │  │ (ElastiCache)│  │  (Storage)   │
└──────────────┘  └──────────────┘  └──────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   External Services                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  GitHub API  │  │  OpenAI API  │  │  Pinecone    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### Technology Stack

**Frontend**
- Framework: React 18 with TypeScript
- State Management: Zustand or Redux Toolkit
- UI Components: Tailwind CSS + shadcn/ui
- Visualization: D3.js for heatmaps and dependency graphs
- Build Tool: Vite
- Deployment: Static hosting on S3 + CloudFront

**Backend**
- Runtime: Node.js 20 (TypeScript) or Python 3.11
- Framework: Express.js (Node) or FastAPI (Python)
- API Style: REST with JSON
- Authentication: JWT tokens
- Background Jobs: Bull (Node) or Celery (Python)

**Data Storage**
- Primary Database: PostgreSQL 15 (AWS RDS)
- Cache: Redis 7 (AWS ElastiCache)
- Object Storage: AWS S3
- Vector Database: Pinecone or Weaviate (managed)

**Infrastructure**
- Container Orchestration: AWS ECS Fargate or Kubernetes (EKS)
- Load Balancer: AWS Application Load Balancer
- CDN: AWS CloudFront
- Monitoring: CloudWatch + Datadog
- Logging: CloudWatch Logs + structured JSON logging

### Deployment Options

**Option 1: AWS ECS Fargate (Recommended for MVP)**
- Pros: Serverless containers, easy scaling, managed infrastructure
- Cons: Slightly higher cost than EC2
- Best for: Quick deployment, minimal ops overhead

**Option 2: AWS EKS (Kubernetes)**
- Pros: Maximum flexibility, portable, advanced orchestration
- Cons: Higher complexity, more ops overhead
- Best for: Long-term scalability, multi-cloud strategy

**Option 3: Vercel + Serverless Functions**
- Pros: Extremely fast deployment, automatic scaling
- Cons: Cold starts, function timeout limits (10s)
- Best for: Hackathon demo, rapid prototyping

### MVP Deployment Strategy (Recommended)

**Phase 1A: Hackathon Demo**
- Frontend: Vercel or Netlify (free tier)
- Backend: Railway or Render (free/hobby tier)
- Database: Supabase (free tier) or Railway PostgreSQL
- Cache: Upstash Redis (free tier)
- Vector DB: Pinecone (free tier)
- Storage: Cloudflare R2 (free tier) or Supabase Storage

**Phase 1B: Post-Hackathon Production**
- Frontend: AWS S3 + CloudFront
- Backend: AWS ECS Fargate (2 containers)
- Database: AWS RDS PostgreSQL (db.t3.micro)
- Cache: AWS ElastiCache Redis (cache.t3.micro)
- Vector DB: Pinecone (starter plan)
- Storage: AWS S3

**Estimated Costs (Phase 1B)**
- ECS Fargate: ~$30/month (2 tasks, 0.5 vCPU, 1GB RAM each)
- RDS PostgreSQL: ~$15/month (db.t3.micro)
- ElastiCache Redis: ~$15/month (cache.t3.micro)
- S3 + CloudFront: ~$5/month (low traffic)
- Pinecone: $70/month (starter plan)
- Total: ~$135/month

### Scaling Considerations

**Horizontal Scaling**
- Frontend: Automatic via CDN
- Backend: Scale ECS tasks based on CPU/memory
- Database: Read replicas for query scaling
- Cache: Redis cluster for high availability

**Vertical Scaling**
- Database: Upgrade RDS instance size
- Backend: Increase container CPU/memory
- Cache: Upgrade ElastiCache node type

**Performance Targets**
- API response time: <200ms (p95)
- Analysis completion: <90s for 5000 files
- Query response: <10s (p95)
- Concurrent users: 100+ (MVP)

### CI/CD Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│  Developer pushes to GitHub                                  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  GitHub Actions Workflow                                     │
│  1. Run linters (ESLint, Prettier)                          │
│  2. Run type checks (TypeScript)                            │
│  3. Run unit tests                                          │
│  4. Run integration tests                                   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Build Docker Images                                         │
│  - Frontend: nginx + static files                           │
│  - Backend: Node.js/Python app                              │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Push to Container Registry (ECR)                            │
│  - Tag with commit SHA and 'latest'                         │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Deploy to Staging                                           │
│  - Update ECS task definition                               │
│  - Run smoke tests                                          │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Manual Approval (for production)                            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Deploy to Production                                        │
│  - Blue/green deployment                                    │
│  - Health checks                                            │
│  - Automatic rollback on failure                            │
└─────────────────────────────────────────────────────────────┘
```

### Environment Configuration

**Development**
- Local Docker Compose setup
- PostgreSQL + Redis containers
- Mock GitHub API responses
- Local LLM or OpenAI API

**Staging**
- Mirrors production architecture
- Separate database and cache
- Real GitHub API (test account)
- OpenAI API (separate key)

**Production**
- Full infrastructure as described
- Encrypted secrets (AWS Secrets Manager)
- Monitoring and alerting
- Automated backups


## Future GitHub App Integration Path

### Current OAuth vs Future GitHub App

**Current OAuth Approach (Phase 1)**
- User-initiated authentication
- Per-user token management
- Limited to user's accessible repositories
- Manual re-authentication required
- Rate limits per user (5000 req/hour)

**Future GitHub App Approach (Phase 2+)**
- Organization-wide installation
- Single app token per installation
- Access to all org repositories
- Automatic token refresh
- Higher rate limits (15,000 req/hour)
- Webhook support for real-time updates

### Migration Path

**Step 1: Create GitHub App**
- Register app in GitHub Developer Settings
- Define permissions:
  - Repository: Read-only access to code, metadata, commits
  - Organization: Read-only access to members (optional)
- Set webhook URL for repository events
- Generate app credentials (App ID, private key)

**Step 2: Implement App Installation Flow**
```
┌─────────────────────────────────────────────────────────────┐
│  Organization Admin installs GitHub App                     │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  GitHub redirects to installation callback                   │
│  - Includes installation_id                                 │
│  - Includes list of accessible repositories                 │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Backend stores installation                                 │
│  - installation_id                                          │
│  - organization_id                                          │
│  - accessible_repositories[]                                │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Generate installation access token                          │
│  - Use app credentials + installation_id                    │
│  - Token valid for 1 hour                                   │
│  - Automatically refresh before expiration                  │
└─────────────────────────────────────────────────────────────┘
```

**Step 3: Implement Webhook Handlers**
```typescript
interface WebhookHandlers {
  // Repository events
  onPush(payload: PushEvent): void
  onPullRequest(payload: PullRequestEvent): void
  onIssues(payload: IssuesEvent): void
  
  // Installation events
  onInstallation(payload: InstallationEvent): void
  onInstallationRepositories(payload: InstallationRepositoriesEvent): void
}

// Example: Invalidate cache on push
async function onPush(payload: PushEvent) {
  const repoUrl = payload.repository.html_url
  
  // Invalidate cached analysis
  await redis.del(`analysis:${repoUrl}`)
  await redis.del(`graph:${repoUrl}`)
  
  // Optionally: trigger incremental re-analysis
  await queueIncrementalAnalysis(repoUrl, payload.commits)
}
```

**Step 4: Incremental Analysis**
- Analyze only changed files (from webhook payload)
- Update dependency graph incrementally
- Recompute centrality only if dependencies changed
- Update complexity scores for modified files
- Merge with existing analysis

**Step 5: Real-time Features**
- Live updates when repository changes
- Automatic re-analysis on push
- Track issue creation/closure
- Monitor pull request activity
- Update recommendations in real-time

### Benefits of GitHub App

1. **Better User Experience**
   - One-time installation per organization
   - No per-user authentication required
   - Automatic access to new repositories

2. **Improved Performance**
   - Higher rate limits (15,000 vs 5,000 req/hour)
   - Webhook-driven updates (no polling)
   - Incremental analysis (faster updates)

3. **Enhanced Features**
   - Real-time repository monitoring
   - Automatic cache invalidation
   - Team-wide analytics
   - Cross-repository insights

4. **Operational Benefits**
   - Centralized token management
   - Automatic token refresh
   - Better error handling
   - Audit logging

### Migration Strategy

**Backward Compatibility**
- Support both OAuth and GitHub App simultaneously
- Detect installation and prefer App token
- Fall back to OAuth if App not installed
- Gradual migration of existing users

**Database Schema Changes**
```sql
-- Add GitHub App installations table
CREATE TABLE github_installations (
  installation_id BIGINT PRIMARY KEY,
  organization_id BIGINT,
  organization_name VARCHAR(255),
  installed_at TIMESTAMP DEFAULT NOW(),
  suspended_at TIMESTAMP,
  access_token_encrypted BYTEA,
  access_token_expires_at TIMESTAMP
);

-- Add installation-repository mapping
CREATE TABLE installation_repositories (
  installation_id BIGINT REFERENCES github_installations(installation_id),
  repository_id BIGINT,
  repository_name VARCHAR(255),
  repository_url VARCHAR(500),
  added_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (installation_id, repository_id)
);

-- Add webhook events log
CREATE TABLE webhook_events (
  event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  installation_id BIGINT REFERENCES github_installations(installation_id),
  event_type VARCHAR(50),
  repository_url VARCHAR(500),
  payload JSONB,
  processed_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);
```

**Rollout Plan**
1. Phase 2A: Implement GitHub App infrastructure
2. Phase 2B: Beta test with select organizations
3. Phase 2C: Implement webhook handlers
4. Phase 2D: Enable incremental analysis
5. Phase 2E: General availability
6. Phase 2F: Deprecate OAuth (optional, maintain for individual users)


## System Guarantees & Validation Principles

The platform's correctness relies on a set of core guarantees that ensure reliable analysis, accurate personalization, and secure operation. These guarantees are validated through property-based testing with minimum 100 iterations per test.

### 1. Score Normalization Correctness

**Guarantee**: All computed metrics remain within valid ranges across all inputs.

*For any* dependency graph, complexity analysis, or personalization computation:
- Centrality scores: [0, 100]
- Complexity scores: [0, 100]
- Suitability scores: [0, 1]
- Cognitive load: [1, 10]

This ensures consistent interpretation of metrics and prevents overflow/underflow errors in downstream calculations.

**Validates: Requirements 3.2, 4.2, 9.6, 10.5, 19.2**

### 2. Skill-Based Personalization Correctness

**Guarantee**: Complexity adjustments accurately reflect user skill levels.

*For any* file and user skill profile, adjusted complexity follows the formula:
```
adjustedComplexity = baseComplexity × adjustmentFactor

where adjustmentFactor = {
  0.6  if Expert skill match
  0.75 if Advanced skill match
  0.9  if Intermediate skill match
  1.0  if Beginner skill match
  1.2  if no skill match
}
```

Suitability scores combine skill confidence with inverse complexity:
```
suitability = (skillConfidence × 0.6) + ((100 - adjustedComplexity) / 100 × 0.4)
```

This ensures personalized recommendations accurately reflect individual developer capabilities.

**Validates: Requirements 9.2-9.5, 10.2**

### 3. Security Guarantees

**Guarantee**: OAuth tokens are encrypted at rest and never exposed in plain text.

*For any* stored OAuth token:
- Encryption: AES-256-GCM with unique IV per token
- Storage: Encrypted bytes + IV + authentication tag
- Transmission: HTTPS only
- Logging: Never logged in plain text
- Expiration: Automatic cleanup after 30 days

This protects user credentials and ensures compliance with security best practices.

**Validates: Requirements 7.2, 7.3, 21.1-21.6**

### 4. RAG Citation Enforcement

**Guarantee**: Generated responses include verifiable citations and mark inferred content.

*For any* query response:
- All repository-specific claims include file path citations
- Content not present in retrieved context is marked "Inferred"
- Responses limited to 500 words for readability
- Code examples include syntax highlighting

This ensures transparency and allows users to verify AI-generated insights against source code.

**Validates: Requirements 18.3, 18.4, 18.5, 18.6**

### 5. Dependency Graph Integrity

**Guarantee**: Dependency graphs accurately represent code structure.

*For any* repository analysis:
- All import statements appear as edges in the graph
- Circular dependencies are detected and flagged
- Internal vs external dependencies are correctly classified
- In-degree and out-degree calculations are accurate

This ensures centrality scores and contribution recommendations are based on correct dependency relationships.

**Validates: Requirements 2.1, 2.3, 2.6, 2.4**

### 6. Cognitive Load Responsiveness

**Guarantee**: Cognitive load estimates adapt to user skill levels.

*For any* file analyzed by two users with different skill levels:
- User with higher skill confidence receives lower cognitive load score
- Formula: `(complexity × 0.35) + (dependencies × 0.25) + (LOC/1000 × 0.2) + ((1 - skillConfidence) × 0.2)`
- Time estimates: 1-2h (load 1-3), 3-6h (load 4-6), >6h (load 7-10)

This ensures learning path recommendations are realistic for individual developers.

**Validates: Requirements 19.1, 19.5, 19.6**

### 7. PageRank Consistency

**Guarantee**: Module centrality scores are deterministic and consistent.

*For any* dependency graph:
- PageRank converges within 100 iterations or delta < 0.001
- Modules with identical dependency patterns receive equal scores
- Scores are normalized to [0, 100] range
- Isolated nodes (no dependencies) receive score of 0

This ensures contribution recommendations prioritize genuinely central modules.

**Validates: Requirements 3.1, 3.2, 3.3, 3.6**

### 8. Skill Inference Accuracy

**Guarantee**: Automated skill profiling follows consistent thresholds.

*For any* language in user's GitHub contribution history:
- Expert (1.0): >10,000 lines contributed
- Advanced (0.75): 5,000-10,000 lines
- Intermediate (0.5): 1,000-5,000 lines
- Beginner (0.25): <1,000 lines

When both manual and inferred skills exist, the system uses the maximum confidence score. Only commits from the past 24 months are considered.

**Validates: Requirements 8.4-8.9**

### Validation Strategy

Each guarantee is implemented as a property-based test using fast-check (TypeScript) or Hypothesis (Python). Tests run with minimum 100 iterations and are tagged with the corresponding guarantee number for traceability.


## Error Handling

### Error Categories and Strategies

**1. GitHub API Errors**

*Rate Limiting (403 with rate limit headers)*
- Detection: Check X-RateLimit-Remaining header
- Response: Return error with X-RateLimit-Reset timestamp
- User message: "GitHub API rate limit reached. Resets at {timestamp}. You can queue this analysis to run automatically after reset."
- Recovery: Offer to queue job for automatic retry

*Authentication Errors (401)*
- Detection: 401 status code
- Response: Clear stored OAuth token
- User message: "GitHub authentication expired. Please reconnect your GitHub account."
- Recovery: Redirect to OAuth flow

*Repository Not Found (404)*
- Detection: 404 status code
- Response: Return descriptive error
- User message: "Repository not found. Please check the URL and ensure the repository is public."
- Recovery: Prompt user to verify URL

*Private Repository Access (403)*
- Detection: 403 status code without rate limit headers
- Response: Return access error
- User message: "This repository is private. Please ensure you have access or use a public repository."
- Recovery: Suggest OAuth authentication for private repo access (future feature)

**2. Parsing Errors**

*Syntax Errors in Source Files*
- Detection: Parser throws exception
- Response: Log error, mark file as "parse_failed", continue processing
- User message: Include in analysis report: "X files failed to parse"
- Recovery: Partial analysis with warnings

*Unsupported Language*
- Detection: No parser available for file extension
- Response: Skip file, log warning
- User message: Include in analysis report: "X files skipped (unsupported language)"
- Recovery: Continue with supported files

*Malformed Import Statements*
- Detection: Parser cannot extract import
- Response: Log warning, skip import, continue
- User message: No user-facing message (logged for debugging)
- Recovery: Partial dependency graph

**3. Computation Errors**

*PageRank Non-Convergence*
- Detection: Max iterations (100) reached without convergence
- Response: Use best-effort scores, log warning
- User message: "Centrality scores are approximate (graph did not fully converge)"
- Recovery: Return approximate results

*Division by Zero in Normalization*
- Detection: Check for zero denominators before division
- Response: Use default value (0 or 1 depending on context)
- User message: No user-facing message
- Recovery: Continue with safe default

*Overflow in Complexity Calculation*
- Detection: Check for values exceeding max
- Response: Clamp to maximum value (100)
- User message: No user-facing message
- Recovery: Use clamped value

**4. Database Errors**

*Connection Failures*
- Detection: Database connection timeout or error
- Response: Retry up to 3 times with exponential backoff
- User message: "Service temporarily unavailable. Please try again."
- Recovery: Return 503 status, suggest retry

*Constraint Violations*
- Detection: Unique constraint or foreign key violation
- Response: Log error, return appropriate HTTP status
- User message: "Unable to save data. Please try again."
- Recovery: Rollback transaction, return error

*Query Timeouts*
- Detection: Query exceeds timeout threshold
- Response: Cancel query, log slow query
- User message: "Request timed out. Please try again or analyze a smaller repository."
- Recovery: Suggest incremental analysis

**5. LLM API Errors**

*API Rate Limiting*
- Detection: 429 status code
- Response: Exponential backoff, retry
- User message: "Query processing delayed due to high demand. Please wait..."
- Recovery: Automatic retry with backoff

*API Timeout*
- Detection: Request timeout (>30s)
- Response: Cancel request, fall back to template response
- User message: Return structured data without LLM-generated narrative
- Recovery: Template-based response using available data

*Invalid Response Format*
- Detection: Response doesn't match expected schema
- Response: Log error, attempt to parse partial response
- User message: "Response may be incomplete"
- Recovery: Return partial response or template

**6. Cache Errors**

*Redis Connection Failure*
- Detection: Redis connection error
- Response: Log warning, bypass cache
- User message: No user-facing message (transparent fallback)
- Recovery: Query database directly

*Cache Deserialization Error*
- Detection: JSON parse error on cached data
- Response: Invalidate cache entry, recompute
- User message: No user-facing message
- Recovery: Recompute and cache fresh data

### Error Response Format

All API errors follow a consistent JSON format:

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "GitHub API rate limit reached",
    "details": {
      "resetAt": "2024-01-15T14:30:00Z",
      "remaining": 0,
      "limit": 5000
    },
    "userMessage": "GitHub API rate limit reached. Resets at 2:30 PM. You can queue this analysis to run automatically after reset.",
    "retryable": true,
    "retryAfter": 1800
  }
}
```

### Graceful Degradation

**Partial Analysis Mode**
- If some files fail to parse, continue with successful files
- Display warning: "Analysis completed with X warnings"
- Provide detailed error log for debugging

**Template Fallback for Queries**
- If LLM API fails, use template-based responses
- Example: "This repository contains {fileCount} files across {languageCount} languages. Top languages: {topLanguages}."
- Mark response as "template-generated"

**Cached Results on Failure**
- If re-analysis fails, serve stale cached results
- Display warning: "Showing cached results from {timestamp}"
- Offer manual retry option

### Monitoring and Alerting

**Error Metrics to Track**
- GitHub API rate limit hits per hour
- Parsing failure rate by language
- LLM API timeout rate
- Database query timeout rate
- Cache hit/miss ratio

**Alert Thresholds**
- GitHub rate limit >80% consumed: Warning
- Parsing failure rate >10%: Warning
- LLM timeout rate >20%: Critical
- Database errors >5 per minute: Critical


## Testing Strategy

### Dual Testing Approach

The platform requires both unit testing and property-based testing for comprehensive coverage:

- **Unit tests**: Verify specific examples, edge cases, and error conditions
- **Property tests**: Verify universal properties across all inputs
- Both are complementary and necessary for comprehensive coverage

Unit tests are helpful for specific examples and edge cases, but we should avoid writing too many unit tests since property-based tests handle covering lots of inputs. Unit tests should focus on:
- Specific examples that demonstrate correct behavior
- Integration points between components
- Edge cases and error conditions

Property tests should focus on:
- Universal properties that hold for all inputs
- Comprehensive input coverage through randomization

### Property-Based Testing Configuration

**Testing Library Selection**
- **JavaScript/TypeScript**: fast-check
- **Python**: Hypothesis
- **Go**: gopter
- **Rust**: proptest

**Test Configuration**
- Minimum 100 iterations per property test (due to randomization)
- Each property test must reference its design document property
- Tag format: `Feature: repository-intelligence-platform, Property {number}: {property_text}`

**Example Property Test (TypeScript with fast-check)**

```typescript
import fc from 'fast-check';

describe('Property 11: PageRank score normalization', () => {
  it('should ensure all centrality scores are within [0, 100]', () => {
    // Feature: repository-intelligence-platform, Property 11: PageRank score normalization
    fc.assert(
      fc.property(
        fc.array(fc.tuple(fc.string(), fc.string()), { minLength: 1, maxLength: 100 }),
        (edges) => {
          const graph = buildDependencyGraph(edges);
          const centralityScores = computeCentrality(graph);
          
          return centralityScores.every(score => score >= 0 && score <= 100);
        }
      ),
      { numRuns: 100 }
    );
  });
});
```

**Example Property Test (Python with Hypothesis)**

```python
from hypothesis import given, strategies as st
import pytest

@given(st.lists(st.tuples(st.text(), st.text()), min_size=1, max_size=100))
def test_property_11_pagerank_normalization(edges):
    """
    Feature: repository-intelligence-platform, Property 11: PageRank score normalization
    For any dependency graph, all computed centrality scores should be within [0, 100].
    """
    graph = build_dependency_graph(edges)
    centrality_scores = compute_centrality(graph)
    
    assert all(0 <= score <= 100 for score in centrality_scores.values())
```

### Unit Testing Strategy

**Repository Intelligence Engine**
- Test file tree extraction with known repositories
- Test language detection with sample files
- Test batch processing with large file lists
- Test error handling for rate limits and network failures
- Test circular dependency detection with crafted graphs

**User Skill Profiling Engine**
- Test OAuth flow with mocked GitHub API
- Test skill inference with sample contribution data
- Test proficiency level thresholds (boundary values)
- Test skill combination logic (manual + inferred)
- Test token encryption/decryption

**Personalization Engine**
- Test complexity adjustment with various skill levels
- Test suitability score calculation with edge cases
- Test cognitive load computation with extreme values
- Test score clamping at boundaries

**Contribution Optimization Engine**
- Test recommendation ranking with sample data
- Test risk assessment with various file characteristics
- Test contribution path generation with dependency chains

**Query Engine**
- Test retrieval with sample embeddings
- Test response generation with mocked LLM
- Test citation extraction
- Test response length limiting
- Test fallback to templates on LLM failure

### Integration Testing

**End-to-End Flows**
1. Complete repository analysis flow
   - Submit repository URL
   - Wait for analysis completion
   - Verify all components populated

2. OAuth and skill profiling flow
   - Initiate OAuth
   - Handle callback
   - Verify skill inference
   - Verify personalized metrics

3. Query flow
   - Submit query
   - Verify retrieval
   - Verify LLM generation
   - Verify response format

**Component Integration**
- Repository Intelligence → Personalization
- Personalization → Contribution Optimization
- All engines → Query Engine

### Performance Testing

**Load Testing**
- Concurrent repository analyses (10, 50, 100 users)
- Query throughput (requests per second)
- Database connection pool under load

**Stress Testing**
- Large repositories (10,000+ files)
- Complex dependency graphs (1,000+ nodes)
- High query volume (100+ concurrent queries)

**Performance Benchmarks**
- Repository analysis: <90s for 5,000 files
- Query response: <10s (p95)
- API response time: <200ms (p95)

### Security Testing

**Authentication Testing**
- OAuth flow security (CSRF protection)
- Token encryption verification
- Session management
- Token expiration handling

**Input Validation**
- Repository URL validation
- SQL injection prevention
- XSS prevention in responses
- Rate limiting effectiveness

**Data Privacy**
- Verify no source code stored permanently
- Verify token encryption
- Verify data deletion on account removal

### Test Coverage Goals

**Code Coverage**
- Unit tests: >80% line coverage
- Critical paths: 100% coverage
- Error handling: 100% coverage

**Property Coverage**
- All 43 correctness properties implemented as property tests
- Each property test runs minimum 100 iterations
- Property tests cover all core algorithms

### Continuous Testing

**Pre-commit Hooks**
- Run linters (ESLint, Prettier, Black)
- Run type checks (TypeScript, mypy)
- Run fast unit tests (<5s)

**CI Pipeline**
- Run all unit tests
- Run property tests (100 iterations each)
- Run integration tests
- Generate coverage report
- Fail build if coverage drops below threshold

**Nightly Tests**
- Extended property tests (1,000 iterations each)
- Performance benchmarks
- Security scans
- Dependency vulnerability checks

### Test Data Management

**Fixtures**
- Sample repositories (small, medium, large)
- Sample dependency graphs (acyclic, cyclic, disconnected)
- Sample skill profiles (beginner, expert, mixed)
- Sample embeddings and queries

**Mocking Strategy**
- Mock GitHub API for unit tests
- Mock LLM API for unit tests
- Use real APIs for integration tests (with test accounts)
- Use test databases (separate from production)

### Regression Testing

**Test Suite Maintenance**
- Add test for every bug fix
- Update tests when requirements change
- Remove obsolete tests
- Refactor tests for maintainability

**Regression Test Categories**
- Critical path tests (must always pass)
- Bug reproduction tests (prevent regressions)
- Performance regression tests (track performance over time)

