# Reasoning Integration Master Plan

## Requirements Summary

This implementation plan outlines the process for integrating our reasoning pattern with a lead orchestrator using OpenAI's Responses API and AI SDK agents. The key requirements include:

1. **Maintain UI/UX:** Preserve the current UI/UX with only text changes
2. **Backend Overhaul:** Restructure the backend while preserving file structure
3. **OpenAI Model Migration:** Move to OpenAI model family with built-in tools
4. **Search Integration:** Use Responses API web search as primary tool, integrated with Jina, Serper, Perplexity, and Firecrawl
5. **Financial Tools:** Incorporate financial tools for agentic workflows
6. **UI Components:** Integrate generative UI finance components
7. **Reasoning Pattern:** Implement structured reasoning with a lead orchestrator

## Implementation Plan

### Phase 1: Model System and Tool Registry (Foundation)

#### 1.1 Update Model System

**File:** `lib/models.ts`
- Replace Anthropic models with OpenAI models
- Implement feature flagging for model capabilities
- Add support for reasoning controls with object-based format
- Create proper typing for model features

```typescript
// Example implementation
import { openai, OpenAILanguageModel } from "@ai-sdk/openai";
import { customProvider } from "ai";

interface ModelCapabilities {
  webSearch: boolean;
  reasoning: boolean;
  responsesApi: boolean;
  parallelTools: boolean;
  financialTools: boolean;
}

interface Model {
  id: string;
  name: string;
  description: string;
  capabilities: ModelCapabilities;
}

export const myProvider = customProvider({
  languageModels: {
    "gpt-4o": openai("gpt-4o"),
    "gpt-4o-mini": openai("gpt-4o-mini"),
    "gpt-4-turbo": openai("gpt-4-turbo"),
  },
});

export const models: Array<Model> = [
  {
    id: "gpt-4o",
    name: "GPT-4o",
    description: "OpenAI's most advanced model with multimodal capabilities.",
    capabilities: {
      webSearch: true,
      reasoning: true,
      responsesApi: true,
      parallelTools: true,
      financialTools: true,
    },
  },
  // Add other models...
];
```

#### 1.1.1 Model Selection Strategy for Development Stages

This section outlines a strategic approach to model selection across different development stages, optimizing for the right balance of speed, cost, and quality. By using different models for different tasks, we can maximize efficiency while ensuring high-quality outputs where it matters most.

**Model Selection by Development Stage:**

| Development Stage | Key Tasks | Recommended Models | Reasoning |
|-------------------|-----------|-------------------|-----------|
| **Exploration** | Understanding codebases, searching documentation | **Primary:** OpenAI o3‑mini<br>**Fallback:** GPT-4o | o3-mini provides fast responses (10x quicker than larger models) with sufficient reasoning for routine exploration. GPT-4o as fallback enhances complex searches and nuanced understanding. |
| **Prioritization** | Bug triage, feature analysis | **Primary:** OpenAI o3‑mini<br>**Selective:** GPT-4o | o3-mini offers balanced reasoning capability and speed for evaluating complexity. GPT-4o selectively improves accuracy for intricate dependency analysis. |
| **Planning** | Architecture decisions, system design | **Primary:** OpenAI o1<br>**Validation:** GPT-4o | o1 provides high reasoning capabilities for complex architectural decisions. GPT-4o supports iterative feedback loops and validation of design decisions. |
| **Execution** | Code generation, editing, reviews | **Routine:** OpenAI o3‑mini<br>**Complex:** GPT-4o<br>**Critical:** OpenAI o1 | Task-specific selection optimizes for speed in routine tasks while ensuring comprehensive reasoning for complex or critical components. |

**Implementation Details:**

1. **Exploration Stage:** 
   - Use o3-mini for most documentation searches and codebase exploration
   - Fallback to GPT-4o for complex documentation analysis or poorly documented codebases
   - Parallel approach can be used when documentation quality is unknown

2. **Prioritization Stage:**
   - o3-mini handles most bug triage and feature analysis with good speed
   - Use GPT-4o selectively for high-stakes prioritization with complex dependencies
   - Consider impact scope when choosing between models

3. **Planning Stage:**
   - o1 is optimal for system-level architectural planning due to superior reasoning
   - Use GPT-4o for iterative validation of design decisions
   - Multi-step approach combines o1's planning with GPT-4o's validation

4. **Execution Stage:**
   - Task complexity determines model selection:
     - Routine coding: o3-mini for speed
     - Complex reviews/debugging: GPT-4o for better reasoning
     - Critical system components: o1 for comprehensive analysis

**Code Implementation:**

```typescript
// In lib/models.ts
export type DevelopmentStage = 'exploration' | 'prioritization' | 'planning' | 'execution';
export type TaskComplexity = 'routine' | 'complex' | 'critical';

// Add the complete model registry with o1, o3-mini, and GPT-4o models
export const models: Array<Model> = [
  {
    id: "o1",
    name: "OpenAI o1",
    description: "High reasoning capabilities for complex planning and critical tasks.",
    capabilities: {
      webSearch: true,
      reasoning: true,
      responsesApi: true,
      parallelTools: true,
      financialTools: true,
    },
  },
  {
    id: "o3-mini",
    name: "OpenAI o3-mini",
    description: "Fast responses with balanced reasoning for exploration and routine tasks.",
    capabilities: {
      webSearch: true,
      reasoning: true,
      responsesApi: true,
      parallelTools: true,
      financialTools: true,
    },
  },
  {
    id: "gpt-4o",
    name: "GPT-4o",
    description: "Advanced reasoning with parallel tool calls for complex tasks and validation.",
    capabilities: {
      webSearch: true,
      reasoning: true,
      responsesApi: true,
      parallelTools: true,
      financialTools: true,
    },
  },
];

// Model selection function
export function selectModelForStage(
  stage: DevelopmentStage, 
  taskComplexity: TaskComplexity = 'routine'
): Model {
  // Primary model selections
  const primarySelections: Record<DevelopmentStage, string> = {
    exploration: 'o3-mini',
    prioritization: 'o3-mini',
    planning: 'o1',
    execution: taskComplexity === 'routine' ? 'o3-mini' : (taskComplexity === 'complex' ? 'gpt-4o' : 'o1')
  };

  // Fallback model selections
  const fallbackSelections: Record<DevelopmentStage, string> = {
    exploration: 'gpt-4o',
    prioritization: 'gpt-4o',
    planning: 'gpt-4o',
    execution: taskComplexity === 'routine' ? 'gpt-4o' : 'o1'
  };

  // Get the primary model ID for the stage and task complexity
  const primaryModelId = primarySelections[stage];
  
  // Find the model in our registry
  const primaryModel = models.find(model => model.id === primaryModelId);
  
  // If the primary model is available, use it
  if (primaryModel) return primaryModel;
  
  // Otherwise, use the fallback model
  const fallbackModelId = fallbackSelections[stage];
  const fallbackModel = models.find(model => model.id === fallbackModelId);
  
  // Return the fallback model or the first available model as a last resort
  return fallbackModel || models[0];
}
```

**Usage in API Route:**

```typescript
// In app/api/chat/route.ts
import { selectModelForStage } from '@/lib/models';

export async function POST(request: NextRequest) {
  const { messages, stage, taskComplexity } = await request.json();
  
  // Select the appropriate model based on development stage and task complexity
  const selectedModel = selectModelForStage(stage, taskComplexity);
  
  const stream = streamText({
    model: myProvider.languageModel(selectedModel.id),
    // ...other options
    messages,
  });
  
  return stream.toDataStreamResponse();
}
```

This implementation ensures consistent model selection across the application while allowing for contextual adaptation based on the specific development stage and task complexity.

#### 1.2 Create Tool Registry System

**Files to create:**
- `lib/tools/registry.ts`: Central tool registry
- `lib/tools/types.ts`: Tool type definitions
- `lib/tools/validation.ts`: Tool validation using Zod

```typescript
// lib/tools/types.ts example
import { z } from "zod";

export interface Tool<TParams = any, TResult = any> {
  name: string;
  description: string;
  parameters: z.ZodType<TParams>;
  execute: (params: TParams) => Promise<TResult>;
  metadata?: {
    source?: string;
    category?: string;
  };
}

// lib/tools/registry.ts example
import { Tool } from "./types";

class ToolRegistry {
  private tools: Map<string, Tool> = new Map();

  register<TParams, TResult>(tool: Tool<TParams, TResult>): void {
    this.tools.set(tool.name, tool);
  }

  get(name: string): Tool | undefined {
    return this.tools.get(name);
  }

  getAll(): Tool[] {
    return Array.from(this.tools.values());
  }
}

export const toolRegistry = new ToolRegistry();
```

### Phase 2: Search and Financial Integration

#### 2.1 Implement Search Provider Integration

**Files to create:**
- `lib/search/providers/index.ts`: Provider selection logic
- `lib/search/providers/openai.ts`: OpenAI web search integration
- `lib/search/providers/jina.ts`: Jina search APIs
- `lib/search/providers/serper.ts`: Serper API
- `lib/search/providers/perplexity.ts`: Perplexity Sonar
- `lib/search/providers/firecrawl.ts`: Firecrawl API

```typescript
// lib/search/providers/index.ts example
import { searchWithOpenAI } from "./openai";
import { searchWithJina } from "./jina";
import { searchWithSerper } from "./serper";
import { searchWithPerplexity } from "./perplexity";
import { searchWithFirecrawl } from "./firecrawl";

export type SearchProvider = "openai" | "jina" | "serper" | "perplexity" | "firecrawl";

export interface SearchOptions {
  query: string;
  provider?: SearchProvider;
  fallback?: boolean;
  timeoutMs?: number;
}

export interface SearchResult {
  content: string;
  url: string;
  title?: string;
  source: SearchProvider;
}

export async function search(options: SearchOptions): Promise<SearchResult[]> {
  const { provider = "openai", fallback = true } = options;
  
  try {
    switch (provider) {
      case "openai":
        return await searchWithOpenAI(options);
      case "jina":
        return await searchWithJina(options);
      // Other providers...
      default:
        return await searchWithOpenAI(options);
    }
  } catch (error) {
    if (fallback && provider !== "openai") {
      // Try OpenAI as fallback
      return await searchWithOpenAI(options);
    }
    throw error;
  }
}

#### 2.1.1 Search Provider Implementation Patterns

Each search provider implementation should follow these robust patterns for reliability and consistency:

**Retry Logic & Error Handling:**
```typescript
// lib/search/providers/base.ts
export async function makeRequestWithRetry<T>(
  requestFn: () => Promise<T>,
  options: {
    maxRetries?: number;
    retryDelay?: number;
    exponentialBackoff?: boolean;
    retryableErrors?: Array<string | RegExp>;
    tracker?: TokenTracker;
  } = {}
): Promise<T> {
  const {
    maxRetries = 3,
    retryDelay = 1000,
    exponentialBackoff = true,
    retryableErrors = [/rate limit/, /timeout/, /socket hang up/, /ECONNRESET/],
    tracker,
  } = options;

  let lastError: any;
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await requestFn();
    } catch (error: any) {
      lastError = error;
      
      // Check if this error is retryable
      const errorMsg = error.message || String(error);
      const shouldRetry = retryableErrors.some(pattern => 
        typeof pattern === "string" 
          ? errorMsg.includes(pattern) 
          : pattern.test(errorMsg)
      );
      
      if (!shouldRetry || attempt === maxRetries) {
        throw error;
      }
      
      // Calculate backoff time
      const delay = exponentialBackoff
        ? retryDelay * Math.pow(2, attempt)
        : retryDelay;
      
      console.warn(`Search request failed (attempt ${attempt + 1}/${maxRetries + 1}), retrying in ${delay}ms`, error);
      
      // Wait before retrying
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
  
  throw lastError;
}
```

**Token Usage Tracking:**
```typescript
// lib/search/tracking.ts
export interface TokenTracker {
  addTokens(count: number, source: string): void;
  getTotal(source?: string): number;
  reset(): void;
}

export class SimpleTokenTracker implements TokenTracker {
  private tokenCounts: Record<string, number> = {};
  
  addTokens(count: number, source: string): void {
    this.tokenCounts[source] = (this.tokenCounts[source] || 0) + count;
  }
  
  getTotal(source?: string): number {
    if (source) {
      return this.tokenCounts[source] || 0;
    }
    
    return Object.values(this.tokenCounts).reduce((total, count) => total + count, 0);
  }
  
  reset(): void {
    this.tokenCounts = {};
  }
}
```

**Response Normalization:**
```typescript
// lib/search/providers/jina.ts example
export async function searchWithJina(options: SearchOptions): Promise<SearchResult[]> {
  const { query, timeoutMs = 10000 } = options;
  const tracker = new SimpleTokenTracker();
  
  return await makeRequestWithRetry(
    async () => {
      const response = await fetch(`https://s.jina.ai/${encodeURIComponent(query)}?count=10`, {
        headers: {
          'Accept': 'application/json',
          'Authorization': `Bearer ${process.env.JINA_API_KEY}`,
          'X-Respond-With': 'no-content',
          'X-Retain-Images': 'none'
        },
        signal: timeoutMs ? AbortSignal.timeout(timeoutMs) : undefined
      });
      
      if (!response.ok) {
        const text = await response.text();
        throw new Error(`Jina API error (${response.status}): ${text}`);
      }
      
      const data = await response.json();
      
      // Add token usage if available
      if (data.usage?.tokens) {
        tracker.addTokens(data.usage.tokens, 'jina_search');
      }
      
      // Normalize to common format
      return data.data.map((item: any) => ({
        content: item.content || item.description || '',
        url: item.url,
        title: item.title || '',
        source: 'jina' as SearchProvider
      }));
    },
    { 
      tracker,
      maxRetries: 2,
      retryDelay: 1500
    }
  );
}
```

#### 2.1.2 Search Results Processing Pipeline

Implement a multi-stage processing pipeline for search results to maximize quality and relevance:

**1. Query Rewriting:**
```typescript
// lib/search/query-rewriter.ts
export async function rewriteQuery(query: string): Promise<string> {
  const { text } = await generateText({
    model: models.find(m => m.capabilities.webSearch)?.id || 'gpt-4o-mini',
    system: "You are an expert at optimizing search queries. Rewrite the given query to be more effective for web search engines. Make it clear, specific, and use common search operators when beneficial. Preserve the original meaning and intent.",
    messages: [
      {
        role: "user",
        content: `Rewrite this search query to be more effective:\n\n"${query}"`
      }
    ]
  });
  
  return text.replace(/^["']|["']$/g, ''); // Remove quotes if present
}
```

**2. Deduplication:**
```typescript
// lib/search/deduplication.ts
export function deduplicateResults(results: SearchResult[]): SearchResult[] {
  const seenUrls = new Set<string>();
  const seenContents = new Map<string, number>(); // Map content fingerprint to similarity score
  
  const contentFingerprint = (content: string): string => {
    // Simplified fingerprint - in production use a more robust method
    return content.trim().toLowerCase().slice(0, 100);
  };
  
  const similarityScore = (content1: string, content2: string): number => {
    // Simplified similarity - in production use a more robust method like Jaccard similarity
    const fp1 = contentFingerprint(content1);
    const fp2 = contentFingerprint(content2);
    
    // Count matching characters
    let matches = 0;
    for (let i = 0; i < Math.min(fp1.length, fp2.length); i++) {
      if (fp1[i] === fp2[i]) matches++;
    }
    
    return matches / Math.max(fp1.length, fp2.length);
  };
  
  return results.filter(result => {
    // Filter by URL first
    if (seenUrls.has(result.url)) {
      return false;
    }
    seenUrls.add(result.url);
    
    // Check content similarity
    const fp = contentFingerprint(result.content);
    for (const [existingFp, _] of seenContents.entries()) {
      if (similarityScore(fp, existingFp) > 0.8) {
        return false;
      }
    }
    
    seenContents.set(fp, 1);
    return true;
  });
}
```

**3. Re-ranking:**
```typescript
// lib/search/reranking.ts
export async function rerankResults(
  query: string, 
  results: SearchResult[]
): Promise<SearchResult[]> {
  // If no results or only one result, no need to rerank
  if (results.length <= 1) return results;
  
  try {
    // Use Jina reranker API
    const response = await fetch('https://api.jina.ai/v1/rerank', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.JINA_API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        model: 'jina-reranker-v2-base-multilingual',
        query: query,
        documents: results.map(r => r.content)
      })
    });
    
    if (!response.ok) {
      const error = await response.text();
      throw new Error(`Reranking failed: ${error}`);
    }
    
    const data = await response.json();
    
    // Map scores to original results
    const scoredResults = results.map((result, i) => ({
      ...result,
      score: data.results[i]?.score || 0
    }));
    
    // Sort by score descending
    return scoredResults.sort((a, b) => (b.score || 0) - (a.score || 0));
  } catch (error) {
    console.warn('Reranking failed, returning original results:', error);
    return results;
  }
}
```

**4. Content Extraction:**
```typescript
// lib/search/content-extraction.ts
export async function extractContentFromUrl(url: string): Promise<string> {
  try {
    // Fetch the content with timeout
    const response = await fetch(url, {
      signal: AbortSignal.timeout(15000),
      headers: {
        'User-Agent': 'Mozilla/5.0 (compatible; SearchBot/1.0)'
      }
    });
    
    if (!response.ok) {
      throw new Error(`Failed to fetch URL: ${response.status}`);
    }
    
    const html = await response.text();
    
    // Use a combination of techniques to extract meaningful content
    // 1. First try removing script, style, header, footer, etc.
    let content = html
      .replace(/<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi, ' ')
      .replace(/<style\b[^<]*(?:(?!<\/style>)<[^<]*)*<\/style>/gi, ' ')
      .replace(/<header\b[^<]*(?:(?!<\/header>)<[^<]*)*<\/header>/gi, ' ')
      .replace(/<footer\b[^<]*(?:(?!<\/footer>)<[^<]*)*<\/footer>/gi, ' ')
      .replace(/<nav\b[^<]*(?:(?!<\/nav>)<[^<]*)*<\/nav>/gi, ' ')
      .replace(/<aside\b[^<]*(?:(?!<\/aside>)<[^<]*)*<\/aside>/gi, ' ');
    
    // 2. Extract text from remaining content
    content = content.replace(/<[^>]*>/g, ' '); // Remove HTML tags
    content = content.replace(/\s+/g, ' ').trim(); // Normalize whitespace
    
    // 3. If content is too long, keep only meaningful parts
    if (content.length > 10000) {
      content = content.substring(0, 10000);
    }
    
    return content;
  } catch (error) {
    console.error(`Content extraction failed for ${url}:`, error);
    return '';
  }
}
```

**5. Grounding:**
```typescript
// lib/search/grounding.ts
export async function groundInformation(query: string, results: SearchResult[]): Promise<string> {
  const concatenatedResults = results
    .map(r => `Source: ${r.url}\nTitle: ${r.title || 'Untitled'}\nContent: ${r.content}`)
    .join('\n\n')
    .slice(0, 32000); // Limit token usage
  
  const { text } = await generateText({
    model: models.find(m => m.capabilities.webSearch)?.id || 'gpt-4o',
    system: `Current date is ${new Date().toISOString()}. You're a fact-checking system that verifies information in search results.`,
    messages: [
      {
        role: "user",
        content: `The user's query is: "${query}"\n\nSearch results:\n${concatenatedResults}\n\nProvide a well-grounded answer based only on these search results. Include key supporting facts from the sources. If sources disagree, acknowledge it. Include the date when relevant.`
      }
    ]
  });
  
  return text;
}
```

#### 2.1.3 Advanced Search Features

Implement these additional features to enhance search robustness and quality:

**Cross-Provider Strategy:**
```typescript
// lib/search/cross-provider.ts
export async function executeCrossProviderSearch(
  query: string,
  maxProviders: number = 2
): Promise<SearchResult[]> {
  // Define provider priority based on query type
  const providers = determineProviderPriority(query);
  
  // Restrict to maximum providers to avoid excessive API calls
  const selectedProviders = providers.slice(0, maxProviders);
  
  // Execute searches in parallel with Promise.all
  const searchPromises = selectedProviders.map(provider =>
    search({ query, provider }).catch(error => {
      console.error(`Error with ${provider} search:`, error);
      return [] as SearchResult[];
    })
  );
  
  const allResults = await Promise.all(searchPromises);
  
  // Flatten results and remove duplicates
  const flattenedResults = allResults.flat();
  const dedupedResults = deduplicateResults(flattenedResults);
  
  // Rerank combined results
  const rerankedResults = await rerankResults(query, dedupedResults);
  
  return rerankedResults;
}

function determineProviderPriority(query: string): SearchProvider[] {
  // Financial queries prioritize specialized providers
  if (/stock|market|finance|price|company|earnings|dividend/i.test(query)) {
    return ['jina', 'perplexity', 'openai', 'serper', 'firecrawl'];
  }
  
  // News queries prioritize fresh content
  if (/news|latest|recent|today|yesterday|update|announcement/i.test(query)) {
    return ['openai', 'serper', 'perplexity', 'jina', 'firecrawl'];
  }
  
  // Technical queries prioritize different providers
  if (/code|programming|technical|software|library|framework|api/i.test(query)) {
    return ['perplexity', 'jina', 'openai', 'serper', 'firecrawl'];
  }
  
  // Default priority for general queries
  return ['openai', 'jina', 'perplexity', 'serper', 'firecrawl'];
}
```

**Metadata Gathering:**
```typescript
// lib/search/metadata.ts
export interface SearchMetadata {
  totalResults: number;
  executionTimeMs: number;
  providers: { [key: string]: boolean };
  tokensUsed: { [key: string]: number };
  queryRewritten: boolean;
  rewritten?: string;
  deduplication?: { 
    before: number;
    after: number; 
  };
  rerankingApplied: boolean;
  groundingApplied: boolean;
  error?: string;
}

export class SearchMetadataCollector {
  private metadata: SearchMetadata = {
    totalResults: 0,
    executionTimeMs: 0,
    providers: {},
    tokensUsed: {},
    queryRewritten: false,
    rerankingApplied: false,
    groundingApplied: false
  };
  
  private startTime: number = Date.now();
  
  constructor() {
    this.reset();
  }
  
  reset(): void {
    this.startTime = Date.now();
    this.metadata = {
      totalResults: 0,
      executionTimeMs: 0,
      providers: {},
      tokensUsed: {},
      queryRewritten: false,
      rerankingApplied: false,
      groundingApplied: false
    };
  }
  
  setTotalResults(count: number): void {
    this.metadata.totalResults = count;
  }
  
  addProvider(provider: SearchProvider): void {
    this.metadata.providers[provider] = true;
  }
  
  addTokenUsage(provider: string, count: number): void {
    this.metadata.tokensUsed[provider] = (this.metadata.tokensUsed[provider] || 0) + count;
  }
  
  setQueryRewritten(rewritten: boolean, newQuery?: string): void {
    this.metadata.queryRewritten = rewritten;
    if (newQuery) {
      this.metadata.rewritten = newQuery;
    }
  }
  
  setDeduplication(before: number, after: number): void {
    this.metadata.deduplication = { before, after };
  }
  
  setRerankingApplied(applied: boolean): void {
    this.metadata.rerankingApplied = applied;
  }
  
  setGroundingApplied(applied: boolean): void {
    this.metadata.groundingApplied = applied;
  }
  
  setError(error: string): void {
    this.metadata.error = error;
  }
  
  finalize(): SearchMetadata {
    this.metadata.executionTimeMs = Date.now() - this.startTime;
    return { ...this.metadata };
  }
}
```

**Fault Tolerance Mechanism:**
```typescript
// lib/search/fault-tolerance.ts
export async function reliableSearch(query: string): Promise<{
  results: SearchResult[];
  metadata: SearchMetadata;
}> {
  const metadataCollector = new SearchMetadataCollector();
  
  try {
    // Step 1: Rewrite query if needed
    let effectiveQuery = query;
    try {
      const rewritten = await rewriteQuery(query);
      if (rewritten && rewritten !== query) {
        effectiveQuery = rewritten;
        metadataCollector.setQueryRewritten(true, rewritten);
      }
    } catch (error) {
      console.warn('Query rewriting failed, using original query:', error);
      metadataCollector.setError(`Query rewrite failed: ${error}`);
    }
    
    // Step 2: Execute search with cross-provider strategy
    // (fallback automatically handles provider failures)
    const searchResults = await executeCrossProviderSearch(effectiveQuery);
    metadataCollector.setTotalResults(searchResults.length);
    
    // Step 3: Process results with pipeline, skipping steps if they fail
    let processedResults = searchResults;
    
    // Deduplication
    try {
      const beforeCount = processedResults.length;
      processedResults = deduplicateResults(processedResults);
      metadataCollector.setDeduplication(beforeCount, processedResults.length);
    } catch (error) {
      console.warn('Deduplication failed:', error);
      metadataCollector.setError(`Deduplication failed: ${error}`);
    }
    
    // Reranking (already handled in cross-provider, but can be reapplied)
    metadataCollector.setRerankingApplied(true);
    
    // Return results and metadata
    return {
      results: processedResults,
      metadata: metadataCollector.finalize()
    };
  } catch (error) {
    console.error('Search failed completely:', error);
    metadataCollector.setError(`Complete search failure: ${error}`);
    return {
      results: [],
      metadata: metadataCollector.finalize()
    };
  }
}
```

#### 2.2 Implement Financial Tools

**Files to create:**
- `lib/financial/tools/index.ts`: Financial tool integration
- `lib/financial/api/client.ts`: API client for financialdatasets.ai
- `lib/financial/types.ts`: Financial data type definitions

```typescript
// lib/financial/tools/index.ts example
import { z } from "zod";
import { toolRegistry } from "../../tools/registry";
import { getStockPrices, getIncomeStatements, getBalanceSheets, getCashFlowStatements, getFinancialMetrics, searchStocksByFilters } from "../api/client";

// Register getStockPrices tool
toolRegistry.register({
  name: "getStockPrices",
  description: "Use this tool to get stock prices and market cap for a company.",
  parameters: z.object({
    ticker: z.string().describe("The ticker of the company"),
    start_date: z.string().describe("The start date (YYYY-MM-DD)"),
    end_date: z.string().describe("The end date (YYYY-MM-DD)"),
    interval: z.enum(["second", "minute", "day", "week", "month", "year"]).default("day"),
    interval_multiplier: z.number().default(1),
  }),
  execute: async (params) => {
    return await getStockPrices(params);
  },
  metadata: {
    source: "financialdatasets.ai",
    category: "financial",
  },
});

// Register other financial tools...
```

### Phase 3: API and Agent Integration

#### 3.1 Update API Route

**File:** `app/api/chat/route.ts`
- Replace current implementation with Responses API
- Add lead orchestrator agent
- Implement tool registry and calling
- Set up reasoning configuration

```typescript
import { NextRequest } from "next/server";
import { Message, smoothStream, streamText } from "ai";
import { myProvider } from "@/lib/models";
import { toolRegistry } from "@/lib/tools/registry";
import { orchestrateQuery } from "@/lib/agents/orchestrator";
import { prepareTools } from "@/lib/tools/preparation";

export async function POST(request: NextRequest) {
  const {
    messages,
    selectedModelId,
    isReasoningEnabled,
  }: {
    messages: Array<Message>;
    selectedModelId: string;
    isReasoningEnabled: boolean;
  } = await request.json();

  // Get available tools based on model capabilities
  const selectedModel = models.find(model => model.id === selectedModelId);
  const availableTools = prepareTools(selectedModel?.capabilities);

  // Set up reasoning configuration
  const reasoningConfig = isReasoningEnabled ? {
    effort: "comprehensive",
    generate_summary: true,
    budget_tokens: 12000,
  } : undefined;

  // Use Responses API with streamText
  const stream = streamText({
    system: prompt,
    providerOptions: {
      openai: {
        responses: {
          enabled: true,
          reasoning: reasoningConfig,
          parallel_tool_calls: true,
        },
      },
    },
    model: myProvider.languageModel(selectedModelId),
    experimental_tools: tools,
    experimental_transform: [
      smoothStream({
        chunking: "word",
      }),
    ],
    messages,
  });

  return stream.toDataStreamResponse({
    sendReasoning: isReasoningEnabled,
    getErrorMessage: () => {
      return `An error occurred, please try again!`;
    },
  });
}
```

#### 3.2 Implement Agent Registry and Orchestrator

**Files to create:**
- `lib/agents/registry.ts`: Agent registry with specializations
- `lib/agents/orchestrator.ts`: Lead orchestrator implementation
- `lib/agents/types.ts`: Agent type definitions
- `lib/agents/collaboration.ts`: Collaboration protocols

```typescript
// lib/agents/types.ts example
export type AgentDomain = "General" | "Financial" | "Scientific" | "Legal" | "Medical" | "Technical" | "Creative";
export type AgentRole = "Orchestrator" | "Researcher" | "Analyst" | "Writer" | "Critic" | "Specialist";

export interface Agent {
  name: string;
  domain: AgentDomain;
  role: AgentRole;
  description: string;
  systemPrompt: string;
  defaultModel: string;
  tools: string[]; // Tool names from the registry
}

// lib/agents/registry.ts example
import { Agent } from "./types";

class AgentRegistry {
  private agents: Map<string, Agent> = new Map();

  register(agent: Agent): void {
    this.agents.set(`${agent.domain}-${agent.role}`, agent);
  }

  getByDomainAndRole(domain: AgentDomain, role: AgentRole): Agent | undefined {
    return this.agents.get(`${domain}-${agent.role}`);
  }

  getBestMatch(query: string): Agent {
    // Find the best agent based on semantic similarity
    // For now, return the orchestrator
    return this.getByDomainAndRole("General", "Orchestrator")!;
  }
}

export const agentRegistry = new AgentRegistry();

// Register default agents
agentRegistry.register({
  name: "Lead Orchestrator",
  domain: "General",
  role: "Orchestrator",
  description: "Coordinates the overall workflow and delegates to specialized agents.",
  systemPrompt: "You are a Lead Orchestrator agent responsible for breaking down complex queries...",
  defaultModel: "gpt-4o",
  tools: ["search"],
});

// Register other agents...
```

### Phase 4: Frontend Integration

#### 4.1 Update Chat Component

**File:** `components/chat.tsx`
- Add support for OpenAI models
- Enable reasoning visualization
- Maintain existing UI layout

```tsx
// Snippet of changes to components/chat.tsx
import { useState } from "react";
import { useChat } from "@ai-sdk/react";
import { models } from "@/lib/models";
// Other imports...

export function Chat() {
  const [input, setInput] = useState<string>("");
  const [selectedModelId] = useState<string>("gpt-4o"); // Default to GPT-4o
  const [isReasoningEnabled, setIsReasoningEnabled] = useState<boolean>(true);

  const selectedModel = models.find((model) => model.id === selectedModelId);

  const { messages, append, status, stop } = useChat({
    id: "primary",
    body: {
      selectedModelId,
      isReasoningEnabled,
    },
    onError: () => {
      toast.error("An error occurred, please try again!");
    },
  });

  // Rest of the component remains similar
  // ...
}
```

#### 4.2 Update Messages Component

**File:** `components/messages.tsx`
- Enhance for reasoning steps display
- Add citation rendering
- Maintain current UI aesthetic

```tsx
// Snippet of changes to components/messages.tsx
// Add support for citations and sources
interface SourceCitation {
  url: string;
  title?: string;
  provider: string;
}

interface MessageWithSources extends UIMessage {
  sources?: SourceCitation[];
}

// Add citation component
function Citation({ source }: { source: SourceCitation }) {
  return (
    <div className="text-xs text-zinc-500 mt-1 flex items-center">
      <a 
        href={source.url} 
        target="_blank" 
        rel="noopener noreferrer"
        className="underline hover:text-zinc-800 dark:hover:text-zinc-300 flex items-center"
      >
        {source.title || source.url}
        <span className="ml-1 text-[10px] bg-zinc-200 dark:bg-zinc-800 px-1 rounded">
          {source.provider}
        </span>
      </a>
    </div>
  );
}

// Update to support sources in the message content
export function MessageContent({ message }: { message: MessageWithSources }) {
  // Existing code...
  
  // Add sources if available
  return (
    <div>
      {/* Existing markdown rendering */}
      <Markdown components={markdownComponents}>
        {message.content}
      </Markdown>
      
      {/* Render sources if available */}
      {message.sources && message.sources.length > 0 && (
        <div className="mt-2">
          <div className="text-xs text-zinc-500 mb-1">Sources:</div>
          {message.sources.map((source, index) => (
            <Citation key={index} source={source} />
          ))}
        </div>
      )}
    </div>
  );
}
```

### Phase 5: Memory and Collaboration System

#### 5.1 Implement Memory System

**Files to create:**
- `lib/memory/index.ts`: Memory system implementation
- `lib/memory/types.ts`: Memory type definitions
- `lib/memory/storage.ts`: Storage implementation

```typescript
// lib/memory/types.ts example
export type MemoryType = "SESSION" | "USER_PREFERENCE" | "CONVERSATION" | "DOMAIN_KNOWLEDGE" | "TOOL_USAGE";

export interface MemoryEntry {
  id: string;
  type: MemoryType;
  content: string;
  metadata?: Record<string, any>;
  embedding?: number[];
  createdAt: Date;
  expiresAt?: Date;
}

// lib/memory/index.ts example
import { MemoryEntry, MemoryType } from "./types";
import { createMemoryStorage } from "./storage";

class MemorySystem {
  private storage = createMemoryStorage();

  async store(type: MemoryType, content: string, metadata?: Record<string, any>, ttlMs?: number): Promise<string> {
    const id = crypto.randomUUID();
    const entry: MemoryEntry = {
      id,
      type,
      content,
      metadata,
      createdAt: new Date(),
      expiresAt: ttlMs ? new Date(Date.now() + ttlMs) : undefined,
    };
    
    await this.storage.set(id, entry);
    return id;
  }

  async retrieve(id: string): Promise<MemoryEntry | undefined> {
    return this.storage.get(id);
  }

  async searchSimilar(content: string, type?: MemoryType, limit = 5): Promise<MemoryEntry[]> {
    // Implementation with embeddings would go here
    // For now, return recent entries of the specified type
    const entries = await this.storage.getAll();
    return entries
      .filter(entry => !type || entry.type === type)
      .sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime())
      .slice(0, limit);
  }
}

export const memorySystem = new MemorySystem();
```

#### 5.2 Implement Collaboration Protocols

**Files to create:**
- `lib/agents/collaboration/sequential.ts`: Sequential execution
- `lib/agents/collaboration/parallel.ts`: Parallel execution
- `lib/agents/collaboration/debate.ts`: Debate protocol
- `lib/agents/collaboration/consensus.ts`: Consensus building

```typescript
// lib/agents/collaboration/sequential.ts example
import { Agent } from "../types";
import { Message } from "ai";
import { myProvider } from "@/lib/models";
import { generateText } from "ai";

export async function sequentialExecution(
  agents: Agent[],
  initialMessages: Message[],
  context?: Record<string, any>
): Promise<{ result: string; intermediateSteps: Array<{ agent: Agent; output: string }> }> {
  let currentMessages = [...initialMessages];
  const intermediateSteps: Array<{ agent: Agent; output: string }> = [];

  for (const agent of agents) {
    // Generate response from this agent
    const response = await generateText({
      model: myProvider.languageModel(agent.defaultModel),
      system: agent.systemPrompt,
      messages: currentMessages,
    });

    // Store the intermediate step
    intermediateSteps.push({
      agent,
      output: response.content,
    });

    // Add this response to the messages for the next agent
    currentMessages.push({
      role: "assistant",
      content: response.content,
    });
  }

  // The final result is the last agent's output
  return {
    result: intermediateSteps[intermediateSteps.length - 1].output,
    intermediateSteps,
  };
}
```

### Phase 6: Financial UI Components Integration

#### 6.1 Copy and Update UI Components

**Files to copy from gen-ui-finance/ to components/finance/:**
- `financials-table.tsx`
- `stock-chart.tsx`
- `stock-screener-table.tsx`
- `table.tsx`
- `ticker-suggestions.tsx`

**File to create:**
- `lib/tools/ui/index.ts`: UI tool integration

```typescript
// lib/tools/ui/index.ts example
import { z } from "zod";
import { toolRegistry } from "../registry";

// Register createDocument tool
toolRegistry.register({
  name: "createDocument",
  description: "Create a document for a writing activity.",
  parameters: z.object({
    title: z.string().describe("The title of the document"),
    kind: z.enum(["text", "code"]).describe("The kind of document to create"),
  }),
  execute: async (params) => {
    // Implementation would go here
    const id = crypto.randomUUID();
    return {
      id,
      title: params.title,
      kind: params.kind,
      content: "",
    };
  },
});

// Register other UI tools...
```

#### 6.2 Integrate Financial Components

**File to create:**
- `components/finance/index.tsx`: Component selection and rendering

```tsx
// components/finance/index.tsx example
import dynamic from "next/dynamic";
import { useState, useEffect } from "react";

// Dynamically import components to reduce initial bundle size
const FinancialsTable = dynamic(() => import("./financials-table"));
const StockChart = dynamic(() => import("./stock-chart"));
const StockScreenerTable = dynamic(() => import("./stock-screener-table"));
const TickerSuggestions = dynamic(() => import("./ticker-suggestions"));

export type FinanceComponentType = 
  | "financials-table"
  | "stock-chart"
  | "stock-screener-table"
  | "ticker-suggestions";

interface FinanceComponentProps {
  type: FinanceComponentType;
  data: any;
}

export function FinanceComponent({ type, data }: FinanceComponentProps) {
  switch (type) {
    case "financials-table":
      return <FinancialsTable data={data} />;
    case "stock-chart":
      return <StockChart data={data} />;
    case "stock-screener-table":
      return <StockScreenerTable data={data} />;
    case "ticker-suggestions":
      return <TickerSuggestions data={data} />;
    default:
      return <div>Unknown component type</div>;
  }
}
```

### Implementation Verification Checklist

- [ ] **Model Integration**
  - [ ] OpenAI models configured in lib/models.ts
  - [ ] Feature flagging implemented for model capabilities
  - [ ] Reasoning controls with object-based format
  - [ ] Provider-specific options for OpenAI
  - [ ] Model selection strategy implemented
    - [ ] Development stage-specific model selection
    - [ ] Task complexity consideration
    - [ ] Fallback mechanisms for model selection
    - [ ] Integration with API route handlers

- [ ] **Tool Registry**
  - [ ] Central tool registry created
  - [ ] Tool type definitions and validation
  - [ ] Search tools integrated
    - [ ] Multi-provider search integration with retry logic
    - [ ] Search results processing pipeline implementation
    - [ ] Query rewriting and optimization
    - [ ] Result deduplication mechanism
    - [ ] Re-ranking for improved relevance
    - [ ] Content extraction implementation
    - [ ] Grounding mechanism for validation
    - [ ] Cross-provider fallback strategies
    - [ ] Token usage tracking and metadata collection
  - [ ] Financial tools integrated
  - [ ] UI tools integrated

- [ ] **API Routing**
  - [ ] app/api/chat/route.ts updated for Responses API
  - [ ] Lead orchestrator integration
  - [ ] Multi-step processing pipeline
  - [ ] Reasoning configuration

- [ ] **Agent System**
  - [ ] Agent registry with domain and role specializations
  - [ ] Lead orchestrator implementation
  - [ ] Domain-specific agents created
  - [ ] Collaboration protocols implemented

- [ ] **Frontend Components**
  - [ ] components/chat.tsx updated for OpenAI models
  - [ ] components/messages.tsx enhanced for reasoning
  - [ ] Citation and source rendering
  - [ ] Dynamic UI component generation

- [ ] **Memory System**
  - [ ] Memory type definitions
  - [ ] Storage implementation
  - [ ] Retrieval strategies
  - [ ] Memory integration with chat session

- [ ] **Financial UI Components**
  - [ ] UI components copied and updated
  - [ ] Component selection mechanism
  - [ ] Financial data visualization
  - [ ] Tool connection with UI components

- [ ] **Testing and Performance**
  - [ ] Unit tests for key components
  - [ ] Integration tests for full flow
  - [ ] Performance metrics collected
  - [ ] Optimizations implemented

### Phase 12: Advanced Agentic Patterns from OpenAI

After reviewing OpenAI's Python agent patterns, we can enhance our implementation with several powerful agentic patterns that address specific use cases and improve overall system robustness.

#### 12.1 Deterministic Agentic Workflows

**Implement explicit validation steps between agent executions:**

```typescript
// lib/agents/workflows/deterministicFlow.ts
import { z } from 'zod';
import { Agent from '@/lib/agents/Agent';
import { AgentRunner from '@/lib/agents/AgentRunner';
import { AgentError from '@/lib/errors/AgentError';

// Define structured output for validation
const OutlineValidationSchema = z.object({
  good_quality: z.boolean(),
  is_financial_related: z.boolean(),
  reasoning: z.string()
});

type OutlineValidation = z.infer<typeof OutlineValidationSchema>;

// Create specialized agents for each step
export const outlineGeneratorAgent = new Agent({
  name: 'outline_generator',
  instructions: 'Generate a very short outline for financial analysis based on the user query.',
  model: 'gpt-4o-mini'
});

export const outlineValidatorAgent = new Agent({
  name: 'outline_validator',
  instructions: 'Evaluate the financial analysis outline for quality and relevance.',
  model: 'gpt-4o-mini'
});

export const analysisWriterAgent = new Agent({
  name: 'analysis_writer',
  instructions: 'Write a comprehensive financial analysis based on the provided outline.',
  model: 'gpt-4o'
});

// Deterministic workflow with validation gates
export async function deterministicFinancialAnalysis(query: string): Promise<string> {
  // Step 1: Generate outline
  const outline = await AgentRunner.run(outlineGeneratorAgent, [query]);
  console.log("Outline generated");
  
  // Step 2: Validate outline
  const validationResult = await AgentRunner.run(outlineValidatorAgent, [outline.agentOutput]);
  const validation = validationResult.agentOutput as OutlineValidation;
  
  // Step 3: Gate - only proceed if outline passes validation
  if (!validation.good_quality) {
    throw new AgentError(`Outline quality insufficient: ${validation.reasoning}`);
  }
  
  if (!validation.is_financial_related) {
    throw new AgentError(`Outline not related to financial analysis: ${validation.reasoning}`);
  }
  
  console.log("Outline validated, proceeding to full analysis");
  
  // Step 4: Write full analysis
  const analysis = await AgentRunner.run(analysisWriterAgent, [outline.agentOutput]);
  
  return analysis.agentOutput as string;
}
```

#### 12.2 Agents as Tools Pattern

**Encapsulate agents as reusable tools:**

```typescript
// lib/agents/AgentExtensions.ts
import { Agent } from '@/lib/agents/Agent';
import { tool } from 'ai';
import { z } from 'zod';

// Extend the Agent class with a method to convert it to a tool
export function agentAsTool(agent: Agent, toolName: string, toolDescription: string) {
  return tool({
    name: toolName,
    description: toolDescription,
    parameters: z.object({
      input: z.string().describe('The input to pass to the agent')
    }),
    execute: async ({ input }) => {
      const result = await agent.execute(input);
      return result.agentOutput;
    }
  });
}

// Usage example
// lib/tools/agentTools.ts
import { financialAnalystAgent, researchAgent from '@/lib/agents/registry';
import { agentAsTool from '@/lib/agents/AgentExtensions';

export const financialAnalysisTool = agentAsTool(
  financialAnalystAgent,
  'analyze_financials',
  'Analyze financial data and provide insights.'
);

export const researchTool = agentAsTool(
  researchAgent,
  'research_topic',
  'Research a topic and provide relevant information.'
);

// In the lead orchestrator agent definition
export const leadOrchestratorAgent = new Agent({
  name: 'lead_orchestrator',
  instructions: 'You are a lead orchestrator agent. Coordinate tasks between specialized agents.',
  tools: [financialAnalysisTool, researchTool],
  model: 'gpt-4o'
});
```

#### 12.3 LLM-as-a-Judge Pattern

**Implement iterative feedback loops for quality improvement:**

```typescript
// lib/agents/workflows/llmJudge.ts
import { z } from 'zod';
import { Agent } from '@/lib/agents/Agent';
import { AgentRunner from '@/lib/agents/AgentRunner';

// Define the evaluation schema
const EvaluationSchema = z.object({
  score: z.enum(['pass', 'needs_improvement', 'fail']),
  feedback: z.string()
});

type Evaluation = z.infer<typeof EvaluationSchema>;

// Create the judge agent
export const evaluatorAgent = new Agent({
  name: 'evaluator',
  instructions: 'You evaluate content and provide detailed feedback for improvement. Be thorough and critical.',
  model: 'gpt-4o'
});

// Iterative improvement workflow
export async function iterativeImprovement(
  contentAgent: Agent,
  prompt: string,
  maxIterations: number = 3
): Promise<{ content: string; iterations: number; finalEvaluation: Evaluation }> {
  let iterations = 0;
  let currentPrompt = prompt;
  let latestContent = '';
  let evaluation: Evaluation = { score: 'needs_improvement', feedback: 'Initial evaluation' };
  
  while (evaluation.score !== 'pass' && iterations < maxIterations) {
    // Generate content
    const contentResult = await AgentRunner.run(contentAgent, [currentPrompt]);
    latestContent = contentResult.agentOutput as string;
    
    // Evaluate content
    const evaluationResult = await AgentRunner.run(
      evaluatorAgent, 
      [`Please evaluate this content and provide feedback:\n\n${latestContent}`]
    );
    evaluation = evaluationResult.agentOutput as Evaluation;
    
    // If not passed, update prompt with feedback
    if (evaluation.score !== 'pass') {
      currentPrompt = `${prompt}\n\nPrevious attempt: ${latestContent}\n\nFeedback: ${evaluation.feedback}\n\nPlease improve based on this feedback.`;
      iterations++;
      console.log(`Iteration ${iterations}: Score ${evaluation.score}`);
    }
  }
  
  return {
    content: latestContent,
    iterations,
    finalEvaluation: evaluation
  };
}
```

#### 12.4 Parallelization for Latency and Quality

**Implement parallel agent executions:**

```typescript
// lib/agents/workflows/parallelExecution.ts
import { Agent } from '@/lib/agents/Agent';
import { AgentRunner from '@/lib/agents/AgentRunner';

// Create a selection agent
export const selectionAgent = new Agent({
  name: 'result_selector',
  instructions: 'You analyze multiple results and select the best one, explaining your choice.',
  model: 'gpt-4o-mini'
});

// Parallel execution with best result selection
export async function parallelAgentExecution(
  agent: Agent, 
  input: string, 
  numParallel: number = 3
): Promise<string> {
  // Run agent multiple times in parallel
  const promiseArray = Array(numParallel).fill(0).map(() => 
    AgentRunner.run(agent, [input])
  );
  
  const results = await Promise.all(promiseArray);
  const outputs = results.map((result, index) => 
    `Option ${index + 1}:\n${result.agentOutput}`
  );
  
  // If only one execution, return that
  if (numParallel === 1) {
    return results[0].agentOutput as string;
  }
  
  // Select the best result
  const selectionPrompt = `
    I have ${numParallel} different results for the prompt: "${input}"
    
    ${outputs.join('\n\n')}
    
    Please analyze these options and select the best one. Explain your choice.
  `;
  
  const selection = await AgentRunner.run(selectionAgent, [selectionPrompt]);
  
  return selection.agentOutput as string;
}
```

#### 12.5 Enhanced Guardrails with Tripwires

**Implement specialized guardrail agents with immediate interruption capabilities:**

```typescript
// lib/guardrails/tripwireGuardrails.ts
import { z } from 'zod';
import { Agent } from '@/lib/agents/Agent';
import { AgentRunner from '@/lib/agents/AgentRunner';
import { GuardrailError from '@/lib/errors/AgentError';

// Define the guardrail result schema
const GuardrailResultSchema = z.object({
  passed: z.boolean(),
  reason: z.string(),
  confidence: z.number().min(0).max(1)
});

type GuardrailResult = z.infer<typeof GuardrailResultSchema>;

// Create a guardrail agent for financial homework detection
export const financialHomeworkGuardrailAgent = new Agent({
  name: 'financial_homework_guardrail',
  instructions: 'Detect if the user is asking for help with financial homework or academic assignments.',
  model: 'gpt-4o-mini'
});

// Guardrail function
export async function checkFinancialHomework(input: string): Promise<GuardrailResult> {
  const result = await AgentRunner.run(financialHomeworkGuardrailAgent, [input]);
  return result.agentOutput as GuardrailResult;
}

// Middleware to apply guardrails
export async function applyGuardrails(
  input: string, 
  guardrails: Array<(input: string) => Promise<GuardrailResult>>
) {
  for (const guardrail of guardrails) {
    const result = await guardrail(input);
    if (!result.passed) {
      throw new GuardrailError(`Guardrail triggered: ${result.reason}`, result);
    }
  }
  
  // If all guardrails pass, return the input
  return input;
}

// Usage in API route
// app/api/chat/route.ts
try {
  // Apply input guardrails first
  const validatedInput = await applyGuardrails(userInput, [
    checkFinancialHomework,
    // Add other guardrails here
  ]);
  
  // Continue with agent execution
  const result = await AgentRunner.run(leadOrchestratorAgent, [validatedInput]);
  
  // Return the result
  return new Response(JSON.stringify({ content: result.agentOutput }));
} catch (error) {
  if (error instanceof GuardrailError) {
    // Handle guardrail violations with appropriate response
    return new Response(JSON.stringify({ 
      error: "Policy violation", 
      message: "I'm sorry, I can't help with financial homework assignments." 
    }));
  }
  // Handle other errors
}
```

#### 12.6 Dynamic Handoffs and Routing

**Implement a frontline orchestrator with dynamic routing:**

```typescript
// lib/agents/workflows/dynamicRouting.ts
import { z } from 'zod';
import { Agent } from '@/lib/agents/Agent';
import { AgentRunner from '@/lib/agents/AgentRunner';
import { AgentError from '@/lib/errors/AgentError';
import { 
  financialAnalystAgent, 
  researchAgent,
  marketDataAgent,
  stockScreenerAgent 
} from '@/lib/agents/registry';

// Define the routing decision schema
const RoutingDecisionSchema = z.object({
  agent_type: z.enum([
    'financial_analyst', 
    'researcher', 
    'market_data',
    'stock_screener'
  ]),
  confidence: z.number().min(0).max(1),
  reasoning: z.string()
});

type RoutingDecision = z.infer<typeof RoutingDecisionSchema>;

// Create a routing agent
export const routingAgent = new Agent({
  name: 'routing_agent',
  instructions: `
    You determine the most appropriate specialized agent to handle the user query.
    Options:
    - financial_analyst: For financial analysis, investment advice, and company evaluations
    - researcher: For general information gathering or factual questions
    - market_data: For specific stock price or market index data queries
    - stock_screener: For finding stocks matching specific criteria
  `,
  model: 'gpt-4o-mini'
});

// Dynamic routing workflow
export async function dynamicRouting(query: string): Promise<string> {
  // Determine the appropriate agent
  const routingResult = await AgentRunner.run(routingAgent, [query]);
  const decision = routingResult.agentOutput as RoutingDecision;
  
  console.log(`Routing to: ${decision.agent_type} (confidence: ${decision.confidence})`);
  
  // Confidence threshold - if below this, use the most comprehensive agent
  if (decision.confidence < 0.7) {
    console.log(`Low confidence routing (${decision.confidence}), using financial analyst`);
    return await AgentRunner.run(financialAnalystAgent, [query]).then(res => res.agentOutput as string);
  }
  
  // Route to the appropriate agent
  switch (decision.agent_type) {
    case 'financial_analyst':
      return await AgentRunner.run(financialAnalystAgent, [query]).then(res => res.agentOutput as string);
    case 'researcher':
      return await AgentRunner.run(researchAgent, [query]).then(res => res.agentOutput as string);
    case 'market_data':
      return await AgentRunner.run(marketDataAgent, [query]).then(res => res.agentOutput as string);
    case 'stock_screener':
      return await AgentRunner.run(stockScreenerAgent, [query]).then(res => res.agentOutput as string);
    default:
      throw new Error(`Unknown agent type: ${decision.agent_type}`);
  }
}
```
