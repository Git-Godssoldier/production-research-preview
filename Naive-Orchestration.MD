import { z } from 'zod';
import { openai } from '@ai-sdk/openai';
import { generateText, generateObject, tool, TokenUsage } from 'ai';
import { trace, custom_span } from '../tracing';
import crypto from 'crypto';
import { AgentExecutor } from '../agent-executor';
import { AgentMemory, MemoryType } from './Memory';
import { AgentRegistry, AgentDomain, AgentRole } from './AgentRegistry';
import { ParallelProcessor } from './ParallelProcessor';

// Import the AI SDK models
import { gpt4oResponses, gpt45Responses, gpt4oMini } from '../../../lib/models';

/**
 * Configuration for lead orchestrator
 */
export interface LeadOrchestratorConfig {
  agentRegistry: AgentRegistry;
  memory: AgentMemory;
  parallelProcessor: ParallelProcessor;
  defaultModel?: any; // AI SDK model
  deepReasoningModel?: any; // AI SDK model for complex reasoning
  searchContextSize?: 'low' | 'medium' | 'high';
  userLocation?: {
    type: 'approximate' | 'precise';
    city: string;
    region: string;
    country?: string;
  };
  tokenBudget?: number;
  maxSteps?: number;
  injectTraceId?: boolean;
}

/**
 * Query types for routing determination
 */
export enum QueryType {
  FINANCIAL_ANALYSIS = 'financial_analysis',
  FINANCIAL_QUESTION = 'financial_question',
  DEEP_RESEARCH = 'deep_research',
  GENERAL_KNOWLEDGE = 'general_knowledge',
  GENERAL_QUERY = 'general_query'
}

/**
 * Result of the orchestrator's query analysis
 */
export interface QueryAnalysisResult {
  queryType: QueryType;
  confidenceScore: number;
  reasoning: string;
  metadata: {
    complexity: 'low' | 'medium' | 'high';
    needsWebSearch: boolean;
    needsMarketData?: boolean;
    symbols?: string[];
    timeframe?: string;
    financialConcepts?: string[];
    topics: string[];
  };
  responseId?: string;
  usage?: TokenUsage;
}

/**
 * Result of query processing
 */
export interface QueryResult {
  query: string;
  response: string;
  sources?: Array<{
    title?: string;
    url: string;
    publishedDate?: string;
  }>;
  usage?: TokenUsage;
  metadata: {
    queryType: QueryType;
    confidence: number;
    processingTime: number;
    sessionId: string;
    traceId: string;
    responseId?: string;
    reasoningTokens?: number;
    [key: string]: any;
  };
}

/**
 * Agent telemetry event types
 */
export interface AgentEvent {
  sessionId: string;
  type: string;
  name: string;
  data?: Record<string, any>;
  error?: string;
  stack?: string;
  timestamp?: number;
}

/**
 * Session information
 */
export interface AgentSession {
  sessionId: string;
  startTime: number;
  endTime?: number;
  endState?: 'completed' | 'error';
  endStateReason?: string;
  tags: string[];
  metadata?: Record<string, any>;
  queryType?: QueryType;
  queryMetadata?: Record<string, any>;
}

/**
 * Simple agent operations tracer
 */
export class AgentOpsTracer {
  private sessions = new Map<string, AgentSession>();
  private events: AgentEvent[] = [];
  
  startSession(options: { sessionId: string; tags?: string[] }): AgentSession {
    const session: AgentSession = {
      sessionId: options.sessionId,
      startTime: Date.now(),
      tags: options.tags || []
    };
    
    this.sessions.set(options.sessionId, session);
    return session;
  }
  
  updateSession(session: AgentSession, updates: Partial<AgentSession>): void {
    const updatedSession = { ...session, ...updates };
    this.sessions.set(session.sessionId, updatedSession);
  }
  
  recordEvent(event: AgentEvent | { sessionId: string, type: string, name: string, data?: any, error?: string, stack?: string }): void {
    const completeEvent: AgentEvent = {
      ...event,
      timestamp: Date.now()
    };
    
    this.events.push(completeEvent);
    console.log(`[EVENT:${event.type}] ${event.name}`, event.data || {});
  }
}

/**
 * Lead orchestrator using OpenAI Responses API and AI SDK
 * Implements patterns from the Python examples in TypeScript
 */
export class LeadOrchestrator {
  private agentRegistry: AgentRegistry;
  private memory: AgentMemory;
  private parallelProcessor: ParallelProcessor;
  private agentOpsTracer: AgentOpsTracer;
  private defaultModel: any;
  private deepReasoningModel: any;
  private searchContextSize: 'low' | 'medium' | 'high';
  private userLocation: {
    type: 'approximate' | 'precise';
    city: string;
    region: string;
    country?: string;
  };
  private tokenBudget: number;
  private maxSteps: number;
  private injectTraceId: boolean;
  
  constructor(config: LeadOrchestratorConfig) {
    this.agentRegistry = config.agentRegistry;
    this.memory = config.memory;
    this.parallelProcessor = config.parallelProcessor;
    this.agentOpsTracer = new AgentOpsTracer();
    this.defaultModel = config.defaultModel || gpt4oResponses;
    this.deepReasoningModel = config.deepReasoningModel || gpt45Responses;
    this.searchContextSize = config.searchContextSize || 'medium';
    this.userLocation = config.userLocation || {
      type: 'approximate',
      city: 'New York',
      region: 'New York',
      country: 'United States'
    };
    this.tokenBudget = config.tokenBudget || 100000;
    this.maxSteps = config.maxSteps || 5;
    this.injectTraceId = config.injectTraceId !== undefined ? config.injectTraceId : true;
  }
  
  /**
   * Main query processing orchestrator 
   * Routes all queries through the appropriate handler based on type
   * All handlers use OpenAI Responses API for consistent implementation
   */
  async processQuery(query: string, options: {
    traceId?: string;
    sessionId?: string;
    previousResponseId?: string;
    userId?: string;
    userContext?: Record<string, any>;
    useWebSearch?: boolean;
    useMarketData?: boolean;
    isReasoningEnabled?: boolean;
    tokenBudget?: number;
    tags?: string[];
    [key: string]: any;
  } = {}): Promise<QueryResult> {
    // Set up tracing and session ID
    const traceId = options.traceId || crypto.randomUUID();
    const sessionId = options.sessionId || crypto.randomUUID();
    
    // Start workflow trace
    const workflowTrace = trace('query-orchestrator', {
      traceId,
      sessionId,
      query,
      options,
      userContext: options.userContext || {}
    });
    workflowTrace.start();
    
    // Start AgentOps session for tracking
    const agentOpsSession = this.agentOpsTracer.startSession({
      sessionId,
      tags: ['query-orchestrator', options.tags || []].flat()
    });
    
    try {
      // First, analyze the query to determine the best routing approach
      const analyzeSpan = custom_span('analyze-query', { 
        query, 
        traceId,
        sessionId
      });
      analyzeSpan.start();
      
      // Record event in AgentOps
      this.agentOpsTracer.recordEvent({
        sessionId: agentOpsSession.sessionId,
        type: 'phase_start',
        name: 'query_analysis',
        data: { query }
      });
      
      // Analyze query to determine routing
      const analysisResult = await this.analyzeQueryForRouting(query, {
        traceId: `${traceId}-analysis`,
        sessionId,
        previousResponseId: options.previousResponseId
      });
      
      // Record completion of analysis
      this.agentOpsTracer.recordEvent({
        sessionId: agentOpsSession.sessionId,
        type: 'phase_complete',
        name: 'query_analysis',
        data: { 
          queryType: analysisResult.queryType,
          confidenceScore: analysisResult.confidenceScore
        }
      });
      
      analyzeSpan.finish();
      
      // Update session with query type information
      this.agentOpsTracer.updateSession(agentOpsSession, {
        queryType: analysisResult.queryType,
        queryMetadata: analysisResult.metadata
      });
      
      // Route to appropriate handler based on query type
      const routeSpan = custom_span('route-query', {
        queryType: analysisResult.queryType,
        traceId,
        sessionId
      });
      routeSpan.start();
      
      let result;
      
      // Execute handler based on query type
      switch (analysisResult.queryType) {
        case QueryType.FINANCIAL_ANALYSIS:
          // Record routing decision
          this.agentOpsTracer.recordEvent({
            sessionId: agentOpsSession.sessionId,
            type: 'routing_decision',
            name: 'financial_analysis',
            data: {
              symbols: analysisResult.metadata?.symbols || [],
              timeframe: analysisResult.metadata?.timeframe
            }
          });
          
          // Handle financial analysis query (stock analysis, market data, etc.)
          result = await this.handleFinancialAnalysis(query, {
            ...options,
            traceId: `${traceId}-financial`,
            sessionId,
            metadata: analysisResult.metadata,
            previousResponseId: analysisResult.responseId
          });
          break;
          
        case QueryType.FINANCIAL_QUESTION:
          // Record routing decision
          this.agentOpsTracer.recordEvent({
            sessionId: agentOpsSession.sessionId,
            type: 'routing_decision',
            name: 'financial_question',
            data: { 
              concepts: analysisResult.metadata?.financialConcepts || [] 
            }
          });
          
          // Handle general financial questions without deep analysis
          result = await this.handleFinancialQuestion(query, {
            ...options,
            traceId: `${traceId}-finance-q`,
            sessionId,
            metadata: analysisResult.metadata,
            previousResponseId: analysisResult.responseId
          });
          break;
          
        case QueryType.DEEP_RESEARCH:
          // Record routing decision
          this.agentOpsTracer.recordEvent({
            sessionId: agentOpsSession.sessionId,
            type: 'routing_decision',
            name: 'deep_research',
            data: { complexity: analysisResult.metadata?.complexity || 'medium' }
          });
          
          // For complex, multi-step research questions, use the full DeepSearch
          result = await this.runDeepSearch(query, {
            ...options,
            traceId: `${traceId}-deep`,
            sessionId,
            tokenBudget: options.tokenBudget || this.tokenBudget,
            previousResponseId: analysisResult.responseId
          });
          break;
          
        case QueryType.GENERAL_KNOWLEDGE:
          // Record routing decision
          this.agentOpsTracer.recordEvent({
            sessionId: agentOpsSession.sessionId,
            type: 'routing_decision',
            name: 'general_knowledge',
            data: { 
              needsWebSearch: analysisResult.metadata?.needsWebSearch || true
            }
          });
          
          // For factual questions that can be answered directly
          result = await this.handleGeneralKnowledge(query, {
            ...options,
            traceId: `${traceId}-knowledge`,
            sessionId,
            metadata: analysisResult.metadata,
            previousResponseId: analysisResult.responseId
          });
          break;
          
        default:
          // For all other queries, use the general handler
          this.agentOpsTracer.recordEvent({
            sessionId: agentOpsSession.sessionId,
            type: 'routing_decision',
            name: 'general_query',
            data: { fallback: true }
          });
          
          // Default to general query handler
          result = await this.handleGeneralQuery(query, {
            ...options,
            traceId: `${traceId}-general`,
            sessionId,
            previousResponseId: analysisResult.responseId
          });
      }
      
      routeSpan.finish();
      
      // Update session with completion status
      this.agentOpsTracer.updateSession(agentOpsSession, {
        endState: 'completed',
        endStateReason: 'Query processed successfully',
        endTime: Date.now()
      });
      
      // Finish workflow trace
      workflowTrace.finish();
      
      // Return result along with metadata for client-side use
      return {
        ...result,
        metadata: {
          ...result.metadata,
          queryType: analysisResult.queryType,
          confidence: analysisResult.confidenceScore,
          processingTime: workflowTrace.duration || 0,
          sessionId,
          traceId
        }
      };
    } catch (error) {
      // Ensure workflow trace is completed even on error
      workflowTrace.finish();
      
      // Record error in AgentOps
      this.agentOpsTracer.recordEvent({
        sessionId: agentOpsSession.sessionId,
        type: 'error',
        name: 'query_processing_error',
        error: error instanceof Error ? error.message : String(error),
        stack: error instanceof Error ? error.stack : undefined
      });
      
      // Update session with error status
      this.agentOpsTracer.updateSession(agentOpsSession, {
        endState: 'error',
        endStateReason: error instanceof Error ? error.message : String(error),
        endTime: Date.now()
      });
      
      throw error;
    }
  }
  
  /**
   * Analyzes the query to determine the best routing approach
   * Uses OpenAI Responses API with structured output for consistent classification
   */
  private async analyzeQueryForRouting(query: string, options: any = {}): Promise<QueryAnalysisResult> {
    const analyzeSpan = custom_span('analyze-query-for-routing', { query: query.substring(0, 100) });
    analyzeSpan.start();
    
    try {
      // Define schema for query analysis
      const queryAnalysisSchema = z.object({
        queryType: z.enum([
          QueryType.FINANCIAL_ANALYSIS,
          QueryType.FINANCIAL_QUESTION,
          QueryType.DEEP_RESEARCH,
          QueryType.GENERAL_KNOWLEDGE,
          QueryType.GENERAL_QUERY
        ]).describe("The type of query being asked"),
        confidenceScore: z.number().min(0).max(1).describe("Confidence in this classification (0-1)"),
        reasoning: z.string().describe("Reasoning behind this classification"),
        metadata: z.object({
          complexity: z.enum(['low', 'medium', 'high']).describe("Complexity of the query"),
          needsWebSearch: z.boolean().describe("Whether the query likely requires web search"),
          needsMarketData: z.boolean().optional().describe("Whether the query requires market data"),
          symbols: z.array(z.string()).optional().describe("Stock symbols mentioned in the query"),
          timeframe: z.string().optional().describe("Timeframe mentioned (e.g., '1y', 'YTD', 'last quarter')"),
          financialConcepts: z.array(z.string()).optional().describe("Financial concepts mentioned"),
          topics: z.array(z.string()).describe("Key topics/concepts in the query")
        }).describe("Metadata extracted from the query to help with processing")
      });
      
      // Define query types and their characteristics
      const queryTypes = `
        Query Types:
        1. financial_analysis - Deep stock/market analysis, requiring financial data and charting
           Examples: "Analyze AAPL stock performance", "Compare MSFT and GOOGL financials"
        
        2. financial_question - General financial knowledge question without deep analysis
           Examples: "What is a P/E ratio?", "How do dividends work?"
        
        3. deep_research - Complex questions requiring multi-step research, citation, and synthesis
           Examples: "What are the long-term implications of AI on the economy?", 
                    "How has climate change affected agricultural productivity?"
        
        4. general_knowledge - Factual questions that can be answered directly
           Examples: "What is quantum computing?", "When was the Declaration of Independence signed?"
        
        5. general_query - Default category for queries that don't fit other types
           Examples: Ambiguous questions, conversational queries, etc.
      `;
      
      // Use AI SDK to classify the query with structured output
      const result = await generateObject({
        model: this.defaultModel,
        schema: queryAnalysisSchema,
        messages: [
          { 
            role: 'system', 
            content: `You are an expert at analyzing user queries to determine the best way to handle them.
             Your job is to analyze the query and classify it into one of several categories,
             extracting key metadata that will help specialized agents handle the query effectively.
             
             ${queryTypes}` 
          },
          { 
            role: 'user', 
            content: `Analyze this query: "${query}"` 
          }
        ],
        providerOptions: {
          openai: {
            store: true, // Enable persistence with Responses API
            reasoning: {
              effort: 'medium',
              generate_summary: null
            },
            previousResponseId: options.previousResponseId, // For conversation continuity
            metadata: {
              workflowName: 'query-analysis',
              traceId: options.traceId,
              sessionId: options.sessionId
            }
          }
        }
      });
      
      analyzeSpan.finish();
      
      // Return classification result with responseId for chain-of-thought persistence
      return {
        ...result.object,
        responseId: result.providerMetadata?.openai?.responseId, // For conversation continuity
        usage: result.usage
      };
    } catch (error) {
      analyzeSpan.finish();
      console.error('Error in query analysis:', error);
      
      // Default to general query if analysis fails
      return {
        queryType: QueryType.GENERAL_QUERY,
        confidenceScore: 0.5,
        reasoning: 'Defaulting to general query due to analysis error',
        metadata: {
          complexity: 'medium',
          needsWebSearch: true,
          topics: []
        }
      };
    }
  }
  
  /**
   * Handler for financial analysis queries
   * Uses market data tools and specialized financial models
   */
  private async handleFinancialAnalysis(query: string, options: any = {}): Promise<QueryResult> {
    const metadata = options.metadata || {};
    const symbols = metadata.symbols || [];
    const timeframe = metadata.timeframe || '1y';
    
    const analysisSpan = custom_span('financial-analysis', {
      query,
      symbols: symbols.join(','),
      timeframe,
      traceId: options.traceId
    });
    analysisSpan.start();
    
    try {
      // Create financial analysis agent using Responses API
      const financialAgent = this.agentRegistry.getAgent('financial') ||
        await this.agentRegistry.findBestAgent(query, AgentDomain.FINANCIAL, AgentRole.ANALYST);
      
      if (!financialAgent.agent) {
        throw new Error('Financial analysis agent not found');
      }
      
      // Get the agent's system prompt
      const instructions = `You are an expert financial analyst specializing in market analysis and investment research.
       Your task is to provide detailed, insightful analysis based on the latest financial data.
       Always consider both technical and fundamental factors in your analysis.
       When appropriate, reference specific data points to support your conclusions.`;
      
      // Set up system context with available market data
      let systemContext = instructions;
      
      // Use web search for latest market news
      // Use AI SDK with Responses API to generate analysis
      const result = await generateText({
        model: this.defaultModel,
        messages: [
          { role: 'system', content: systemContext },
          { role: 'user', content: query }
        ],
        tools: {
          web_search_preview: openai.tools.webSearchPreview({
            searchContextSize: this.searchContextSize,
            userLocation: this.userLocation
          }),
          // Additional tools would be added here from tools registry
        },
        maxSteps: this.maxSteps,
        providerOptions: {
          openai: {
            store: true, // Enable persistence with Responses API
            parallelToolCalls: true, // Allow parallel tool calls for efficiency
            reasoning: {
              effort: options.isReasoningEnabled ? 'high' : 'medium',
              generate_summary: options.isReasoningEnabled ? 'detailed' : null
            },
            previousResponseId: options.previousResponseId, // For conversation continuity
            metadata: {
              workflowName: 'financial-analysis',
              symbols: symbols.join(','),
              timeframe,
              traceId: options.traceId,
              sessionId: options.sessionId
            }
          }
        }
      });
      
      analysisSpan.finish();
      
      // Return comprehensive result with market data and analysis
      return {
        query,
        response: result.text,
        sources: result.sources || [],
        usage: result.usage,
        metadata: {
          responseId: result.providerMetadata?.openai?.responseId, // For conversation continuity
          reasoningTokens: result.providerMetadata?.openai?.reasoningTokens,
          symbols,
          timeframe,
          queryType: QueryType.FINANCIAL_ANALYSIS,
          confidence: 0.9,
          processingTime: analysisSpan.duration || 0,
          sessionId: options.sessionId,
          traceId: options.traceId
        }
      };
    } catch (error) {
      analysisSpan.finish();
      console.error('Error in financial analysis:', error);
      throw error;
    }
  }
  
  /**
   * Handler for general financial questions
   * Uses web search and financial knowledge base
   */
  private async handleFinancialQuestion(query: string, options: any = {}): Promise<QueryResult> {
    const questionSpan = custom_span('financial-question', {
      query,
      traceId: options.traceId
    });
    questionSpan.start();
    
    try {
      // Create financial knowledge agent using Responses API
      const instructions = `You are an expert in financial concepts, markets, and investing principles.
       Your task is to provide clear, accurate explanations of financial topics
       based on well-established financial knowledge and principles.
       Use plain language that is accessible to someone without a finance background,
       but don't oversimplify to the point of inaccuracy.`;
      
      // Use AI SDK with Responses API to answer financial question
      const result = await generateText({
        model: this.defaultModel,
        messages: [
          { role: 'system', content: instructions },
          { role: 'user', content: query }
        ],
        tools: {
          web_search_preview: openai.tools.webSearchPreview({
            searchContextSize: this.searchContextSize,
            userLocation: this.userLocation
          })
        },
        maxSteps: this.maxSteps,
        providerOptions: {
          openai: {
            store: true, // Enable persistence with Responses API
            reasoning: {
              effort: options.isReasoningEnabled ? 'high' : 'medium',
              generate_summary: options.isReasoningEnabled ? 'concise' : null
            },
            previousResponseId: options.previousResponseId, // For conversation continuity
            metadata: {
              workflowName: 'financial-question',
              concepts: options.metadata?.financialConcepts?.join(','),
              traceId: options.traceId,
              sessionId: options.sessionId
            }
          }
        }
      });
      
      questionSpan.finish();
      
      // Return result with sources
      return {
        query,
        response: result.text,
        sources: result.sources || [],
        usage: result.usage,
        metadata: {
          responseId: result.providerMetadata?.openai?.responseId, // For conversation continuity
          reasoningTokens: result.providerMetadata?.openai?.reasoningTokens,
          queryType: QueryType.FINANCIAL_QUESTION,
          confidence: 0.85,
          processingTime: questionSpan.duration || 0,
          sessionId: options.sessionId,
          traceId: options.traceId
        }
      };
    } catch (error) {
      questionSpan.finish();
      console.error('Error in financial question handling:', error);
      throw error;
    }
  }
  
  /**
   * Handler for general knowledge queries
   * Leverages web search for up-to-date information
   */
  private async handleGeneralKnowledge(query: string, options: any = {}): Promise<QueryResult> {
    const knowledgeSpan = custom_span('general-knowledge', {
      query,
      traceId: options.traceId
    });
    knowledgeSpan.start();
    
    try {
      // Import tool registry for search tools and the AI SDK converter
      const { toolRegistry } = await import('../registry/tools');
      const { convertToZodSchema } = await import('./AgentsSDK');
      
      // Get the tools registry for specialized search tools
      const jinaSearchTool = toolRegistry.jina_search;
      const firecrawlSearchTool = toolRegistry.firecrawl_search;
      const perplexitySearchTool = toolRegistry.web_search;
      
      // First, define our schema for structured output
      const responseSchema = z.object({
        answer: z.string().describe("The complete, factual answer to the user's question"),
        key_facts: z.array(z.string()).describe("Bullet points of the most important facts from the answer"),
        uncertainty: z.string().describe("Areas where information might be incomplete or where sources disagree"),
        sources: z.array(z.object({
          url: z.string().describe("Source URL"),
          title: z.string().optional().describe("Title of the source"),
          relevance: z.enum(["high", "medium", "low"]).describe("Relevance of this source to the answer")
        })).describe("Sources used to compile the answer")
      });
      
      // Create knowledge agent using Responses API with structured output
      const instructions = `You are an expert at answering factual questions with accurate, up-to-date information.
       Your primary goal is to provide clear, precise answers to user questions.
       You have access to multiple search providers (OpenAI web search, Jina, Firecrawl, and Perplexity).
       Use the most appropriate search tools based on the query type:
       
       - For current events and recent information: use OpenAI's built-in web_search_preview tool
       - For specialized or technical topics: use the jina_search tool
       - For in-depth analysis of specific websites or documents: use firecrawl_search or firecrawl_analyze_url
       - For general research questions: consider combining multiple sources for cross-verification
       
       Always provide a comprehensive, factual answer based on the search results.
       Include clear source citations and identify any areas of uncertainty.`;
      
      // Create tools using AI SDK tool() function for Responses API compatibility
      const advancedSearchTools = {
        web_search_preview: openai.tools.webSearchPreview({
          searchContextSize: this.searchContextSize,
          userLocation: this.userLocation
        }),
        jina_search: tool({
          name: jinaSearchTool.name,
          description: jinaSearchTool.description,
          parameters: convertToZodSchema(jinaSearchTool.parameters),
          execute: async (args: any) => {
            try {
              return await jinaSearchTool.execute(args);
            } catch (error) {
              console.error(`Error executing jina_search:`, error);
              return { error: `Failed to execute jina_search` };
            }
          }
        }),
        firecrawl_search: tool({
          name: firecrawlSearchTool.name,
          description: firecrawlSearchTool.description,
          parameters: convertToZodSchema(firecrawlSearchTool.parameters),
          execute: async (args: any) => {
            try {
              return await firecrawlSearchTool.execute(args);
            } catch (error) {
              console.error(`Error executing firecrawl_search:`, error);
              return { error: `Failed to execute firecrawl_search` };
            }
          }
        }),
        perplexity_search: tool({
          name: perplexitySearchTool.name,
          description: perplexitySearchTool.description,
          parameters: convertToZodSchema(perplexitySearchTool.parameters),
          execute: async (args: any) => {
            try {
              return await perplexitySearchTool.execute(args);
            } catch (error) {
              console.error(`Error executing perplexity_search:`, error);
              return { error: `Failed to execute perplexity_search` };
            }
          }
        })
      };
      
      // First execution phase: Use OpenAI Responses API with structured output
      const result = await generateObject({
        model: this.defaultModel,
        schema: responseSchema,
        messages: [
          { role: 'system', content: instructions },
          { role: 'user', content: query }
        ],
        tools: advancedSearchTools,
        maxSteps: this.maxSteps,
        providerOptions: {
          openai: {
            store: true, // Enable persistence with Responses API
            reasoning: options.isReasoningEnabled ? 'verbose' : 'auto',
            parallelToolCalls: true, // Enable parallel tool calls for efficiency
            previousResponseId: options.previousResponseId, // For conversation continuity
            metadata: {
              workflowName: 'enhanced-general-knowledge',
              topics: options.metadata?.topics?.join(','),
              traceId: options.traceId,
              sessionId: options.sessionId,
              clientId: 'opulentia-reasoning-platform'
            }
          }
        }
      });
      
      // Format structured output into a rich response
      let formattedResponse = result.object.answer;
      
      // Add key facts in a bulleted list
      if (result.object.key_facts.length > 0) {
        formattedResponse += "\n\n**Key Facts:**\n";
        for (const fact of result.object.key_facts) {
          formattedResponse += `- ${fact}\n`;
        }
      }
      
      // Add uncertainty notes if present
      if (result.object.uncertainty && result.object.uncertainty.trim() !== '') {
        formattedResponse += `\n\n**Note on limitations:** ${result.object.uncertainty}`;
      }
      
      // Get sources for the response
      const sources = result.object.sources.map(source => ({
        title: source.title || source.url,
        url: source.url,
        relevance: source.relevance
      }));
      
      knowledgeSpan.finish();
      
      // Return structured answer with sources from all search providers
      return {
        query,
        response: formattedResponse,
        sources: sources,
        usage: result.usage,
        metadata: {
          responseId: result.providerMetadata?.openai?.responseId, // For conversation continuity
          reasoningTokens: result.providerMetadata?.openai?.reasoningTokens,
          queryType: QueryType.GENERAL_KNOWLEDGE,
          confidence: 0.95, // Higher confidence with multi-provider verification
          processingTime: knowledgeSpan.duration || 0,
          sessionId: options.sessionId,
          traceId: options.traceId,
          structuredData: {
            key_facts: result.object.key_facts,
            uncertainty: result.object.uncertainty
          }
        }
      };
    } catch (error) {
      knowledgeSpan.finish();
      console.error('Error in enhanced general knowledge handling:', error);
      
      // Fall back to basic web search if structured output fails
      try {
        // Use AI SDK with Responses API to answer knowledge question
        const fallbackResult = await generateText({
          model: this.defaultModel,
          messages: [
            { role: 'system', content: "You are an expert at answering factual questions with accurate, up-to-date information." },
            { role: 'user', content: query }
          ],
          tools: {
            web_search_preview: openai.tools.webSearchPreview({
              searchContextSize: this.searchContextSize,
              userLocation: this.userLocation
            })
          },
          maxSteps: this.maxSteps,
          providerOptions: {
            openai: {
              store: true,
              reasoning: options.isReasoningEnabled ? 'verbose' : 'auto',
              previousResponseId: options.previousResponseId,
              metadata: {
                workflowName: 'fallback-knowledge',
                traceId: options.traceId,
                sessionId: options.sessionId
              }
            }
          }
        });
        
        return {
          query,
          response: fallbackResult.text,
          sources: fallbackResult.sources || [],
          usage: fallbackResult.usage,
          metadata: {
            responseId: fallbackResult.providerMetadata?.openai?.responseId,
            reasoningTokens: fallbackResult.providerMetadata?.openai?.reasoningTokens,
            queryType: QueryType.GENERAL_KNOWLEDGE,
            confidence: 0.7, // Lower confidence for fallback
            processingTime: knowledgeSpan.duration || 0,
            sessionId: options.sessionId,
            traceId: options.traceId,
            fallback: true
          }
        };
      } catch (fallbackError) {
        console.error('Fallback also failed:', fallbackError);
        throw error; // Throw the original error
      }
    }
  }
  
  /**
   * Default handler for general queries
   * Uses basic web search tools
   */
  private async handleGeneralQuery(query: string, options: any = {}): Promise<QueryResult> {
    const generalSpan = custom_span('general-query', {
      query,
      traceId: options.traceId
    });
    generalSpan.start();
    
    try {
      // Create general agent using Responses API
      const instructions = `You are a helpful AI assistant that can answer a wide range of questions.
       Use web search when needed to provide accurate, up-to-date information.
       Always be helpful, accurate, and concise in your responses.`;
      
      // Use AI SDK with Responses API to handle general query
      const result = await generateText({
        model: this.defaultModel,
        messages: [
          { role: 'system', content: instructions },
          { role: 'user', content: query }
        ],
        tools: {
          web_search_preview: openai.tools.webSearchPreview({
            searchContextSize: this.searchContextSize,
            userLocation: this.userLocation
          })
        },
        maxSteps: this.maxSteps,
        providerOptions: {
          openai: {
            store: true, // Enable persistence with Responses API
            reasoning: {
              effort: options.isReasoningEnabled ? 'high' : 'medium',
              generate_summary: options.isReasoningEnabled ? 'concise' : null
            },
            previousResponseId: options.previousResponseId, // For conversation continuity
            metadata: {
              workflowName: 'general-query',
              traceId: options.traceId,
              sessionId: options.sessionId
            }
          }
        }
      });
      
      generalSpan.finish();
      
      // Return general response
      return {
        query,
        response: result.text,
        sources: result.sources || [],
        usage: result.usage,
        metadata: {
          responseId: result.providerMetadata?.openai?.responseId, // For conversation continuity
          reasoningTokens: result.providerMetadata?.openai?.reasoningTokens,
          queryType: QueryType.GENERAL_QUERY,
          confidence: 0.7,
          processingTime: generalSpan.duration || 0,
          sessionId: options.sessionId,
          traceId: options.traceId
        }
      };
    } catch (error) {
      generalSpan.finish();
      console.error('Error in general query handling:', error);
      throw error;
    }
  }
  
  /**
   * Handler for deep research queries using multi-step research
   * Leverages multiple specialized sub-agents for comprehensive research
   */
  private async runDeepSearch(query: string, options: any = {}): Promise<QueryResult> {
    const deepSearchSpan = custom_span('deep-search', {
      query,
      tokenBudget: options.tokenBudget,
      traceId: options.traceId
    });
    deepSearchSpan.start();
    
    try {
      // Record deep search start in AgentOps
      this.agentOpsTracer.recordEvent({
        sessionId: options.sessionId,
        type: 'phase_start',
        name: 'deep_search',
        data: { query, tokenBudget: options.tokenBudget }
      });
      
      // Create research plan using planning agent
      const planningSpan = custom_span('research-planning', {
        query,
        traceId: options.traceId
      });
      planningSpan.start();
      
      // Define schema for research plan
      const researchPlanSchema = z.object({
        researchSteps: z.array(z.object({
          stepNumber: z.number().int().positive(),
          description: z.string().describe("Description of this research step"),
          expectedOutput: z.string().describe("What information this step should produce"),
          importance: z.enum(['critical', 'high', 'medium', 'low']).describe("Importance of this step")
        })).min(3).max(10).describe("Sequential research steps to follow"),
        keyQuestions: z.array(z.string()).min(3).max(10).describe("Key questions that need to be answered"),
        searchTerms: z.array(z.string()).min(3).max(10).describe("Effective search terms for web research"),
        synthesisApproach: z.string().describe("How to synthesize findings into a coherent response"),
        potentialChallenges: z.array(z.string()).describe("Potential challenges and how to address them")
      });
      
      // Create planning agent using Responses API
      const instructions = `You are an expert research planner specializing in creating structured, thorough research plans.
       Your task is to break down complex research questions into logical steps that will lead to a 
       comprehensive answer. Consider what information needs to be gathered, what sources would be most 
       valuable, and how to synthesize findings into a coherent response.`;
      
      // Create a research plan with structured output
      const researchPlan = await generateObject({
        model: this.deepReasoningModel,
        schema: researchPlanSchema,
        messages: [
          { role: 'system', content: instructions },
          { role: 'user', content: `Create a comprehensive research plan for this query: "${query}"` }
        ],
        providerOptions: {
          openai: {
            store: true, // Enable persistence with Responses API
            reasoning: {
              effort: 'high',
              generate_summary: 'detailed'
            },
            parallelToolCalls: false, // Sequential planning
            previousResponseId: options.previousResponseId, // For conversation continuity
            metadata: {
              workflowName: 'research-planning',
              traceId: options.traceId,
              sessionId: options.sessionId
            }
          }
        }
      });
      
      planningSpan.finish();
      
      // Record research plan in AgentOps
      this.agentOpsTracer.recordEvent({
        sessionId: options.sessionId,
        type: 'research_plan_created',
        name: 'research_planning_complete',
        data: { 
          steps: researchPlan.object.researchSteps.length,
          keyQuestions: researchPlan.object.keyQuestions
        }
      });
      
      // Execute research steps in sequence
      const researchResults: {
        steps: Array<{
          stepNumber: number;
          description: string;
          findings: string;
          sources: Array<{
            title?: string;
            url: string;
            publishedDate?: string;
          }>;
        }>;
        sources: Array<{
          title?: string;
          url: string;
          publishedDate?: string;
        }>;
      } = {
        steps: [],
        sources: []
      };
      
      // Allocate tokens for each step based on importance
      const tokenAllocation = this.allocateTokenBudget(
        researchPlan.object.researchSteps,
        options.tokenBudget || this.tokenBudget
      );
      
      // Research execution span
      const researchExecutionSpan = custom_span('research-execution', {
        steps: researchPlan.object.researchSteps.length,
        traceId: options.traceId
      });
      researchExecutionSpan.start();
      
      let previousResponseId = researchPlan.providerMetadata?.openai?.responseId;
      
      // Execute each research step
      for (const step of researchPlan.object.researchSteps) {
        const stepSpan = custom_span('research-step', {
          stepNumber: step.stepNumber,
          description: step.description,
          tokenBudget: tokenAllocation[step.stepNumber - 1],
          traceId: options.traceId
        });
        stepSpan.start();
        
        // Record step start in AgentOps
        this.agentOpsTracer.recordEvent({
          sessionId: options.sessionId,
          type: 'step_start',
          name: `research_step_${step.stepNumber}`,
          data: { 
            stepNumber: step.stepNumber, 
            description: step.description,
            tokenBudget: tokenAllocation[step.stepNumber - 1]
          }
        });
        
        // Create research agent for this specific step
        const instructions = `You are a specialized research agent working on step ${step.stepNumber} of a multi-step research plan.
         You are focused specifically on: ${step.description}
         Your goal is to produce: ${step.expectedOutput}
         Use web search to find accurate, up-to-date information.
         Always cite your sources.`;
        
        // Include relevant search terms from the plan
        const searchContext = `
          This is step ${step.stepNumber} of ${researchPlan.object.researchSteps.length} in researching: "${query}"
          
          Key questions relevant to this step:
          ${researchPlan.object.keyQuestions.join('\n')}
          
          Suggested search terms:
          ${researchPlan.object.searchTerms.join('\n')}
        `;
        
        // Execute research for this step
        const stepResult = await generateText({
          model: this.defaultModel,
          messages: [
            { role: 'system', content: `${instructions}\n\n${searchContext}` },
            { role: 'user', content: `Execute research step ${step.stepNumber}: ${step.description}` }
          ],
          tools: {
            web_search_preview: openai.tools.webSearchPreview({
              searchContextSize: 'high',
              userLocation: this.userLocation
            })
          },
          maxTokens: tokenAllocation[step.stepNumber - 1],
          maxSteps: this.maxSteps,
          providerOptions: {
            openai: {
              store: true, // Enable persistence with Responses API
              parallelToolCalls: true, // Allow parallel tool calls for efficient research
              previousResponseId, // Chain research steps
              metadata: {
                workflowName: 'deep-research',
                stepNumber: step.stepNumber,
                traceId: options.traceId,
                sessionId: options.sessionId
              }
            }
          }
        });
        
        // Update response ID for next step
        previousResponseId = stepResult.providerMetadata?.openai?.responseId;
        
        // Store research results
        researchResults.steps.push({
          stepNumber: step.stepNumber,
          description: step.description,
          findings: stepResult.text,
          sources: stepResult.sources || []
        });
        
        // Accumulate sources from all steps
        if (stepResult.sources?.length > 0) {
          for (const source of stepResult.sources) {
            // Only add if not already present
            if (!researchResults.sources.some(s => s.url === source.url)) {
              researchResults.sources.push(source);
            }
          }
        }
        
        stepSpan.finish();
        
        // Record step completion in AgentOps
        this.agentOpsTracer.recordEvent({
          sessionId: options.sessionId,
          type: 'step_complete',
          name: `research_step_${step.stepNumber}_complete`,
          data: { 
            stepNumber: step.stepNumber,
            sourceCount: stepResult.sources?.length || 0,
            tokenUsage: stepResult.usage?.totalTokens || 0
          }
        });
      }
      
      researchExecutionSpan.finish();
      
      // Now synthesize findings into a comprehensive response
      const synthesisSpan = custom_span('research-synthesis', {
        steps: researchResults.steps.length,
        traceId: options.traceId
      });
      synthesisSpan.start();
      
      // Create synthesis agent
      const synthesisInstructions = `You are an expert at synthesizing complex research findings into clear, coherent responses.
       Your task is to combine multiple pieces of research into a comprehensive answer that addresses
       the original query directly. Organize information logically, highlight key insights, and ensure
       all claims are properly supported by the research. Include citations.`;
      
      // Prepare research findings for synthesis
      let researchFindings = 'Research Findings:\n\n';
      
      for (const step of researchResults.steps) {
        researchFindings += `--- Step ${step.stepNumber}: ${step.description} ---\n\n`;
        researchFindings += `${step.findings}\n\n`;
      }
      
      // Use the synthesis approach from the research plan
      const synthesisPrompt = `
        Original Query: "${query}"
        
        Synthesis Approach: ${researchPlan.object.synthesisApproach}
        
        ${researchFindings}
        
        Now, synthesize these findings into a comprehensive, well-structured response that 
        directly answers the original query. Ensure proper citations to sources.
      `;
      
      // Generate the synthesized response
      const synthesisResult = await generateText({
        model: this.deepReasoningModel,
        messages: [
          { role: 'system', content: synthesisInstructions },
          { role: 'user', content: synthesisPrompt }
        ],
        providerOptions: {
          openai: {
            store: true, // Enable persistence with Responses API
            reasoning: 'high', // Detailed reasoning for synthesis
            previousResponseId, // Chain with previous research
            metadata: {
              workflowName: 'research-synthesis',
              traceId: options.traceId,
              sessionId: options.sessionId
            }
          }
        }
      });
      
      synthesisSpan.finish();
      deepSearchSpan.finish();
      
      // Record synthesis completion in AgentOps
      this.agentOpsTracer.recordEvent({
        sessionId: options.sessionId,
        type: 'phase_complete',
        name: 'deep_search_complete',
        data: { 
          steps: researchResults.steps.length,
          sourceCount: researchResults.sources.length,
          tokenUsage: synthesisResult.usage?.totalTokens || 0
        }
      });
      
      // Return comprehensive deep search results
      return {
        query,
        response: synthesisResult.text,
        sources: researchResults.sources,
        usage: synthesisResult.usage,
        metadata: {
          responseId: synthesisResult.providerMetadata?.openai?.responseId, // For conversation continuity
          reasoningTokens: synthesisResult.providerMetadata?.openai?.reasoningTokens,
          queryType: QueryType.DEEP_RESEARCH,
          confidence: 0.95,
          processingTime: deepSearchSpan.duration || 0,
          sessionId: options.sessionId,
          traceId: options.traceId,
          research: {
            plan: {
              steps: researchPlan.object.researchSteps.map(step => ({
                stepNumber: step.stepNumber,
                description: step.description
              }))
            }
          }
        }
      };
    } catch (error) {
      deepSearchSpan.finish();
      console.error('Error in deep search:', error);
      throw error;
    }
  }
  
  /**
   * Helper method to allocate token budget to research steps
   * Distributes tokens based on step importance
   */
  private allocateTokenBudget(steps: any[], totalBudget: number): number[] {
    // Define weights for each importance level
    const importanceWeights: Record<string, number> = {
      'critical': 4,
      'high': 3,
      'medium': 2,
      'low': 1
    };
    
    // Calculate total weight
    const totalWeight = steps.reduce((sum, step) => 
      sum + (importanceWeights[step.importance as keyof typeof importanceWeights] || 1), 0
    );
    
    // Reserve 20% of budget for synthesis
    const researchBudget = Math.floor(totalBudget * 0.8);
    
    // Allocate budget proportionally based on importance
    return steps.map(step => {
      const weight = importanceWeights[step.importance as keyof typeof importanceWeights] || 1;
      return Math.floor((weight / totalWeight) * researchBudget);
    });
  }
}

/**
 * Create a lead orchestrator with default configuration
 */
export async function createLeadOrchestrator(
  agentRegistry: AgentRegistry,
  memory: AgentMemory,
  parallelProcessor: ParallelProcessor,
  config: Partial<LeadOrchestratorConfig> = {}
): Promise<LeadOrchestrator> {
  return new LeadOrchestrator({
    agentRegistry,
    memory,
    parallelProcessor,
    ...config
  });
}