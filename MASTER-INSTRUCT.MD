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

## STARTING DIRECTORY
# Libraries Map

This document provides a map of the `app/`, `components/`, and `lib/` directories, including a brief description of each file.

## app/

*   **api/**: Contains API route(s).
    *   **chat/**: Contains chat API route(s).
        *   `route.ts`: Handles the chat API logic.
*   `favicon.ico`: The favicon for the website.
*   `globals.css`: Global CSS file for styling the application.
*   `layout.tsx`: Defines the root layout for the application.
*   `opengraph-image.png`: The Open Graph image for the website.
*   `page.tsx`: The main page component for the application.
*   `twitter-image.png`: The Twitter image for the website.

## components/

*   `chat.tsx`: The main chat component.
*   `deploy-button.tsx`: A component that renders a button to deploy the application to Vercel.
*   `footnote.tsx`: A component that renders a footnote with links to Next.js and the AI SDK.
*   `icons.tsx`: Defines various icons used in the application.
*   `input.tsx`: The input component for the chat interface.
*   `markdown-components.tsx`: Defines custom components for rendering Markdown.
*   `messages.tsx`: Renders the messages in the chat interface.
*   `star-button.tsx`: A component that renders a button to star the project on GitHub.

## lib/

*   `models.ts`: Defines the available language models and their configurations.




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

#### 2.1.4 Jina Search Integration

**Files to create:**
- `lib/search/providers/jina.ts`: Jina search implementation
- `lib/search/providers/jina-reader.ts`: Jina Reader API integration
- `lib/search/grounding-jina.ts`: Jina grounding implementation
- `lib/search/reranking-jina.ts`: Jina reranker integration
- `lib/search/segmenter-jina.ts`: Jina content segmentation

**Jina Search Provider Implementation:**
```typescript
// lib/search/providers/jina.ts
import { SearchOptions, SearchResult } from "./providers";
import { makeRequestWithRetry } from "./base";
import { SimpleTokenTracker } from "../tracking";

export async function searchWithJina(options: SearchOptions): Promise<SearchResult[]> {
  const { query, timeoutMs = 15000 } = options;
  const tracker = new SimpleTokenTracker();
  
  return await makeRequestWithRetry(
    async () => {
      // Use Jina Search API (s.reader) to get search results
      const response = await fetch('https://s.jina.ai/', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${process.env.JINA_API_KEY}`, // Get your Jina AI API key for free: https://jina.ai/?sui=apikey
          'Content-Type': 'application/json',
          'Accept': 'application/json',
          'X-With-Links-Summary': 'true',
          'X-No-Cache': 'true'
        },
        body: JSON.stringify({ 
          q: query,
          options: 'Markdown'
        }),
        signal: timeoutMs ? AbortSignal.timeout(timeoutMs) : undefined
      });
      
      if (!response.ok) {
        const text = await response.text();
        throw new Error(`Jina Search API error (${response.status}): ${text}`);
      }
      
      const data = await response.json();
      
      // Add token usage if available
      if (data.usage?.tokens) {
        tracker.addTokens(data.usage.tokens, 'jina_search');
      }
      
      // Normalize to common format
      return data.data.map((item: any) => ({
        content: item.content || '',
        url: item.url,
        title: item.title || '',
        source: 'jina' as const
      }));
    },
    { 
      tracker,
      maxRetries: 2,
      retryDelay: 1500,
      retryableErrors: [/rate limit/, /timeout/, /socket hang up/, /ECONNRESET/, /502/, /503/, /504/]
    }
  );
}
```

**Jina Reader Integration:**
```typescript
// lib/search/providers/jina-reader.ts
import { makeRequestWithRetry } from "./base";
import { SimpleTokenTracker } from "../tracking";

export interface ReaderOptions {
  url: string;
  format?: 'markdown' | 'html' | 'text';
  withLinks?: boolean;
  withImages?: boolean;
  noCache?: boolean;
  removeSelectors?: string[];
  targetSelectors?: string[];
  waitForSelectors?: string[];
  timeout?: number;
}

export interface ReaderResult {
  content: string;
  title?: string;
  description?: string;
  links?: Record<string, string>;
  images?: Record<string, string>;
  url: string;
  tokens?: number;
}

export async function readContentWithJina(options: ReaderOptions): Promise<ReaderResult> {
  const { 
    url, 
    format = 'markdown',
    withLinks = true,
    withImages = false,
    noCache = false,
    removeSelectors = [],
    targetSelectors = [],
    waitForSelectors = [],
    timeout = 15
  } = options;
  
  const tracker = new SimpleTokenTracker();
  
  return await makeRequestWithRetry(
    async () => {
      // Prepare headers
      const headers: Record<string, string> = {
        'Authorization': `Bearer ${process.env.JINA_API_KEY}`, // Get your Jina AI API key for free: https://jina.ai/?sui=apikey
        'Content-Type': 'application/json',
        'Accept': 'application/json'
      };
      
      // Add optional headers
      if (withLinks) headers['X-With-Links-Summary'] = 'true';
      if (withImages) headers['X-With-Images-Summary'] = 'true';
      if (noCache) headers['X-No-Cache'] = 'true';
      if (removeSelectors.length) headers['X-Remove-Selector'] = removeSelectors.join(',');
      if (targetSelectors.length) headers['X-Target-Selector'] = targetSelectors.join(',');
      if (waitForSelectors.length) headers['X-Wait-For-Selector'] = waitForSelectors.join(',');
      if (timeout) headers['X-Timeout'] = timeout.toString();
      
      // Set return format
      if (format) {
        headers['X-Return-Format'] = format;
      }
      
      const response = await fetch('https://r.jina.ai/', {
        method: 'POST',
        headers,
        body: JSON.stringify({ url })
      });
      
      if (!response.ok) {
        const text = await response.text();
        throw new Error(`Jina Reader API error (${response.status}): ${text}`);
      }
      
      const data = await response.json();
      
      if (data.usage?.tokens) {
        tracker.addTokens(data.usage.tokens, 'jina_reader');
      }
      
      return {
        content: data.data.content || '',
        title: data.data.title || '',
        description: data.data.description || '',
        links: data.data.links || {},
        images: data.data.images || {},
        url: data.data.url || url,
        tokens: data.usage?.tokens
      };
    },
    { 
      tracker,
      maxRetries: 2,
      retryDelay: 2000,
      retryableErrors: [/rate limit/, /timeout/, /socket hang up/, /ECONNRESET/, /502/, /503/, /504/]
    }
  );
}
```

**Jina Reranker Integration:**
```typescript
// lib/search/reranking-jina.ts
import { SearchResult } from "./providers";
import { makeRequestWithRetry } from "./providers/base";
import { SimpleTokenTracker } from "./tracking";

export interface RerankOptions {
  query: string;
  documents: string[] | SearchResult[];
  model?: string;
  topN?: number;
  returnDocuments?: boolean;
}

export interface RerankResult {
  index: number;
  score: number;
  document?: string | SearchResult;
}

export async function rerankWithJina(options: RerankOptions): Promise<RerankResult[]> {
  const { 
    query, 
    documents, 
    model = 'jina-reranker-v2-base-multilingual',
    topN = documents.length,
    returnDocuments = true 
  } = options;
  
  const tracker = new SimpleTokenTracker();
  
  // Prepare documents for the API
  const processedDocuments = documents.map(doc => {
    if (typeof doc === 'string') {
      return doc;
    } else {
      return doc.content;
    }
  });
  
  return await makeRequestWithRetry(
    async () => {
      const response = await fetch('https://api.jina.ai/v1/rerank', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${process.env.JINA_API_KEY}`, // Get your Jina AI API key for free: https://jina.ai/?sui=apikey
          'Content-Type': 'application/json',
          'Accept': 'application/json'
        },
        body: JSON.stringify({
          model,
          query,
          documents: processedDocuments,
          top_n: topN,
          return_documents: returnDocuments
        })
      });
      
      if (!response.ok) {
        const text = await response.text();
        throw new Error(`Jina Reranker API error (${response.status}): ${text}`);
      }
      
      const data = await response.json();
      
      if (data.usage?.total_tokens) {
        tracker.addTokens(data.usage.total_tokens, 'jina_reranker');
      }
      
      return data.results.map((result: any, index: number) => ({
        index: result.index,
        score: result.relevance_score,
        document: returnDocuments ? (
          typeof documents[result.index] === 'string' 
            ? result.document.text 
            : documents[result.index]
        ) : undefined
      }));
    },
    { 
      tracker,
      maxRetries: 2,
      retryDelay: 1000
    }
  );
}
```

**Jina Grounding Integration:**
```typescript
// lib/search/grounding-jina.ts
import { makeRequestWithRetry } from "./providers/base";
import { SimpleTokenTracker } from "./tracking";

export interface GroundingOptions {
  statement: string;
  sites?: string[];
  noCache?: boolean;
}

export interface GroundingReference {
  url: string;
  keyQuote: string;
  isSupportive: boolean;
}

export interface GroundingResult {
  factuality: number;
  result: boolean;
  reason: string;
  references: GroundingReference[];
  tokens?: number;
}

export async function groundWithJina(options: GroundingOptions): Promise<GroundingResult> {
  const { statement, sites = [], noCache = false } = options;
  const tracker = new SimpleTokenTracker();
  
  return await makeRequestWithRetry(
    async () => {
      // Prepare headers
      const headers: Record<string, string> = {
        'Authorization': `Bearer ${process.env.JINA_API_KEY}`, // Get your Jina AI API key for free: https://jina.ai/?sui=apikey
        'Content-Type': 'application/json',
        'Accept': 'application/json'
      };
      
      // Add optional headers
      if (sites.length > 0) {
        headers['X-Site'] = sites.join(',');
      }
      
      if (noCache) {
        headers['X-No-Cache'] = 'true';
      }
      
      const response = await fetch('https://g.jina.ai/', {
        method: 'POST',
        headers,
        body: JSON.stringify({ statement })
      });
      
      if (!response.ok) {
        const text = await response.text();
        throw new Error(`Jina Grounding API error (${response.status}): ${text}`);
      }
      
      const data = await response.json();
      
      if (data.usage?.tokens) {
        tracker.addTokens(data.usage.tokens, 'jina_grounding');
      }
      
      return {
        factuality: data.data.factuality,
        result: data.data.result,
        reason: data.data.reason,
        references: data.data.references,
        tokens: data.usage?.tokens
      };
    },
    { 
      tracker,
      maxRetries: 1, // Ground truth verification is expensive, limit retries
      retryDelay: 2000,
      retryableErrors: [/timeout/, /socket hang up/, /ECONNRESET/, /502/, /503/, /504/]
    }
  );
}
```

**Jina Segmenter Integration:**
```typescript
// lib/search/segmenter-jina.ts
import { makeRequestWithRetry } from "./providers/base";
import { SimpleTokenTracker } from "./tracking";

export interface SegmenterOptions {
  content: string;
  tokenizer?: 'cl100k_base' | 'o200k_base' | 'p50k_base' | 'r50k_base' | 'p50k_edit' | 'gpt2';
  returnTokens?: boolean;
  returnChunks?: boolean;
  maxChunkLength?: number;
  head?: number;
  tail?: number;
}

export interface SegmenterResult {
  numTokens: number;
  tokenizer: string;
  chunks?: string[];
  chunkPositions?: [number, number][];
  tokens?: any[][];
}

export async function segmentWithJina(options: SegmenterOptions): Promise<SegmenterResult> {
  const { 
    content, 
    tokenizer = 'cl100k_base',
    returnTokens = false,
    returnChunks = true,
    maxChunkLength = 1000,
    head,
    tail
  } = options;
  
  const tracker = new SimpleTokenTracker();
  
  return await makeRequestWithRetry(
    async () => {
      const response = await fetch('https://segment.jina.ai/', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${process.env.JINA_API_KEY}`, // Get your Jina AI API key for free: https://jina.ai/?sui=apikey
          'Content-Type': 'application/json',
          'Accept': 'application/json'
        },
        body: JSON.stringify({
          content,
          tokenizer,
          return_tokens: returnTokens,
          return_chunks: returnChunks,
          max_chunk_length: maxChunkLength,
          ...(head !== undefined && { head }),
          ...(tail !== undefined && { tail })
        })
      });
      
      if (!response.ok) {
        const text = await response.text();
        throw new Error(`Jina Segmenter API error (${response.status}): ${text}`);
      }
      
      const data = await response.json();
      
      if (data.usage?.tokens) {
        tracker.addTokens(data.usage.tokens, 'jina_segmenter');
      }
      
      return {
        numTokens: data.num_tokens,
        tokenizer: data.tokenizer,
        chunks: data.chunks,
        chunkPositions: data.chunk_positions,
        tokens: data.tokens
      };
    },
    { 
      tracker,
      maxRetries: 2,
      retryDelay: 1000
    }
  );
}
```

**Enhanced Provider Selection with Jina:**
```typescript
// lib/search/providers/index.ts
export function determineProviderPriority(query: string): SearchProvider[] {
  // Multilingual queries prioritize Jina
  if (/[^\x00-\x7F]/.test(query)) {
    return ['jina', 'openai', 'perplexity', 'serper', 'firecrawl'];
  }
  
  // Financial queries prioritize specialized providers
  if (/stock|market|finance|price|company|earnings|dividend/i.test(query)) {
    return ['jina', 'perplexity', 'openai', 'serper', 'firecrawl'];
  }
  
  // News queries prioritize fresh content
  if (/news|latest|recent|today|yesterday|update|announcement/i.test(query)) {
    return ['openai', 'jina', 'serper', 'perplexity', 'firecrawl'];
  }
  
  // Technical queries prioritize different providers
  if (/code|programming|technical|software|library|framework|api/i.test(query)) {
    return ['perplexity', 'jina', 'openai', 'serper', 'firecrawl'];
  }
  
  // Default priority for general queries
  return ['openai', 'jina', 'perplexity', 'serper', 'firecrawl'];
}
```

#### 2.1.5 Integration Example for Advanced Jina Features

**Comprehensive Search with Jina-Enhanced Pipeline:**
```typescript
// lib/search/advanced-jina-search.ts
import { search, SearchOptions, SearchResult } from "./providers";
import { rerankWithJina } from "./reranking-jina";
import { readContentWithJina } from "./providers/jina-reader";
import { groundWithJina } from "./grounding-jina";
import { segmentWithJina } from "./segmenter-jina";
import { deduplicateResults } from "./deduplication";
import { SearchMetadataCollector } from "./metadata";

export async function performComprehensiveSearch(query: string): Promise<{
  results: SearchResult[];
  groundedAnswer?: string;
  isFactual?: boolean;
  reason?: string;
  metadata: any;
}> {
  const metadataCollector = new SearchMetadataCollector();
  metadataCollector.setQueryRewritten(true, query);
  
  try {
    // Step 1: Perform initial search
    const searchResults = await search({ query, provider: 'jina' });
    metadataCollector.addProvider('jina');
    metadataCollector.setTotalResults(searchResults.length);
    
    // Early return if no results
    if (searchResults.length === 0) {
      return { results: [], metadata: metadataCollector.finalize() };
    }
    
    // Step 2: Rerank results for better relevance
    const rerankedResults = await rerankWithJina({
      query,
      documents: searchResults,
      topN: Math.min(5, searchResults.length)
    });
    metadataCollector.setRerankingApplied(true);
    
    // Create sorted results from reranking
    const sortedResults = rerankedResults
      .map(result => searchResults[result.index])
      .filter(Boolean);
    
    // Step 3: For top result, get full content via Reader API
    if (sortedResults.length > 0) {
      const topResult = sortedResults[0];
      try {
        const fullContent = await readContentWithJina({
          url: topResult.url,
          format: 'markdown',
          withLinks: true
        });
        
        // Enhance top result with full content
        topResult.content = fullContent.content;
        if (fullContent.title) topResult.title = fullContent.title;
        
        // Perform content segmentation if content is large
        if (fullContent.content.length > 2000) {
          const segments = await segmentWithJina({
            content: fullContent.content,
            returnChunks: true,
            maxChunkLength: 1000
          });
          
          // Use the first few segments to update content (to manage token usage)
          if (segments.chunks && segments.chunks.length > 0) {
            topResult.content = segments.chunks.slice(0, 3).join('\n\n');
          }
        }
      } catch (error) {
        console.warn('Failed to enhance top result with Reader API:', error);
      }
    }
    
    // Step 4: Ground the answer based on the search results
    let groundedInfo = undefined;
    let factuality = undefined;
    let reason = undefined;
    
    try {
      // Construct a statement from the search results
      const statementFromResults = `Based on the search results, ${query}`;
      
      // Ground this statement
      const grounding = await groundWithJina({
        statement: statementFromResults,
        sites: sortedResults.slice(0, 3).map(result => result.url)
      });
      
      groundedInfo = grounding.reason;
      factuality = grounding.result;
      reason = grounding.reason;
      metadataCollector.setGroundingApplied(true);
    } catch (error) {
      console.warn('Grounding failed:', error);
      metadataCollector.setError(`Grounding failed: ${error}`);
    }
    
    return {
      results: sortedResults,
      groundedAnswer: groundedInfo,
      isFactual: factuality,
      reason,
      metadata: metadataCollector.finalize()
    };
  } catch (error) {
    console.error('Comprehensive search failed:', error);
    metadataCollector.setError(`Comprehensive search failed: ${error}`);
    return {
      results: [],
      metadata: metadataCollector.finalize()
    };
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
    - [ ] Jina search provider integration with multilingual support
    - [ ] Jina reader API integration for comprehensive content extraction
    - [ ] Jina re-ranker implementation for improved result relevance
    - [ ] Jina grounding API integration for fact-checking
    - [ ] Jina segmenter API integration for content chunking
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
import { agentAsTool } from '@/lib/agents/AgentExtensions';

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
