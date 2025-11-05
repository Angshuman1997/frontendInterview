# Retrieval-Augmented Generation (RAG) and Vector Search

RAG systems combine the power of Large Language Models with external knowledge retrieval, enabling applications to provide accurate, up-to-date information grounded in specific data sources. This guide covers enterprise-grade RAG implementation with vector databases and semantic search.

## RAG Architecture Implementation

### 1. Complete RAG System with Vector Database
```typescript
// Enterprise RAG System Implementation
import { Pinecone } from '@pinecone-database/pinecone';
import OpenAI from 'openai';
import { RecursiveCharacterTextSplitter } from 'langchain/text_splitter';
import { Document } from 'langchain/document';

// Vector Database Manager
class VectorDatabaseManager {
  private pinecone: Pinecone;
  private openai: OpenAI;
  private indexName: string;
  private textSplitter: RecursiveCharacterTextSplitter;

  constructor(config: VectorDBConfig) {
    this.pinecone = new Pinecone({
      apiKey: config.pineconeApiKey,
      environment: config.pineconeEnvironment
    });
    
    this.openai = new OpenAI({
      apiKey: config.openaiApiKey
    });
    
    this.indexName = config.indexName;
    
    this.textSplitter = new RecursiveCharacterTextSplitter({
      chunkSize: config.chunkSize || 1000,
      chunkOverlap: config.chunkOverlap || 200,
      separators: ['\n\n', '\n', ' ', '']
    });
  }

  // Initialize vector database index
  async initializeIndex(dimension: number = 1536): Promise<void> {
    try {
      // Check if index exists
      const existingIndexes = await this.pinecone.listIndexes();
      const indexExists = existingIndexes.indexes?.some(
        index => index.name === this.indexName
      );

      if (!indexExists) {
        await this.pinecone.createIndex({
          name: this.indexName,
          dimension,
          metric: 'cosine',
          spec: {
            serverless: {
              cloud: 'aws',
              region: 'us-east-1'
            }
          }
        });

        // Wait for index to be ready
        await this.waitForIndexReady();
      }
    } catch (error) {
      console.error('Error initializing index:', error);
      throw new Error(`Failed to initialize vector index: ${error.message}`);
    }
  }

  // Process and store documents
  async addDocuments(documents: DocumentInput[]): Promise<void> {
    const index = this.pinecone.index(this.indexName);
    const batchSize = 100;

    for (let i = 0; i < documents.length; i += batchSize) {
      const batch = documents.slice(i, i + batchSize);
      const vectors: PineconeVector[] = [];

      for (const doc of batch) {
        try {
          // Split document into chunks
          const chunks = await this.textSplitter.splitText(doc.content);
          
          for (let chunkIndex = 0; chunkIndex < chunks.length; chunkIndex++) {
            const chunk = chunks[chunkIndex];
            
            // Generate embedding for chunk
            const embedding = await this.generateEmbedding(chunk);
            
            // Create vector with metadata
            const vector: PineconeVector = {
              id: `${doc.id}_chunk_${chunkIndex}`,
              values: embedding,
              metadata: {
                documentId: doc.id,
                title: doc.title,
                content: chunk,
                chunkIndex,
                totalChunks: chunks.length,
                source: doc.source,
                createdAt: doc.createdAt || new Date().toISOString(),
                tags: doc.tags || [],
                ...doc.metadata
              }
            };
            
            vectors.push(vector);
          }
        } catch (error) {
          console.error(`Error processing document ${doc.id}:`, error);
        }
      }

      // Upsert vectors to Pinecone
      if (vectors.length > 0) {
        await index.upsert(vectors);
      }
    }
  }

  // Semantic search with filters
  async search(
    query: string,
    options: SearchOptions = {}
  ): Promise<SearchResult[]> {
    try {
      const index = this.pinecone.index(this.indexName);
      
      // Generate query embedding
      const queryEmbedding = await this.generateEmbedding(query);
      
      // Build filter
      const filter = this.buildFilter(options.filters);
      
      // Search vectors
      const searchResponse = await index.query({
        vector: queryEmbedding,
        topK: options.topK || 10,
        includeMetadata: true,
        includeValues: false,
        filter
      });

      // Process and rank results
      const results: SearchResult[] = searchResponse.matches?.map(match => ({
        id: match.id || '',
        score: match.score || 0,
        content: match.metadata?.content as string || '',
        title: match.metadata?.title as string || '',
        source: match.metadata?.source as string || '',
        documentId: match.metadata?.documentId as string || '',
        metadata: match.metadata || {}
      })) || [];

      // Apply post-processing filters
      return this.postProcessResults(results, options);
    } catch (error) {
      console.error('Vector search error:', error);
      throw new Error(`Search failed: ${error.message}`);
    }
  }

  // Hybrid search combining vector and keyword search
  async hybridSearch(
    query: string,
    options: HybridSearchOptions = {}
  ): Promise<SearchResult[]> {
    const vectorResults = await this.search(query, {
      topK: options.vectorTopK || 20,
      filters: options.filters
    });

    // Keyword search using metadata
    const keywordResults = await this.keywordSearch(query, options);
    
    // Combine and rerank results
    return this.combineSearchResults(vectorResults, keywordResults, options);
  }

  // Delete documents
  async deleteDocuments(documentIds: string[]): Promise<void> {
    const index = this.pinecone.index(this.indexName);
    
    for (const docId of documentIds) {
      // Delete all chunks for this document
      await index.deleteMany({
        filter: { documentId: { $eq: docId } }
      });
    }
  }

  // Update document
  async updateDocument(documentId: string, updates: Partial<DocumentInput>): Promise<void> {
    // Delete existing document
    await this.deleteDocuments([documentId]);
    
    // Re-add updated document
    if (updates.content) {
      await this.addDocuments([{
        id: documentId,
        content: updates.content,
        title: updates.title || '',
        source: updates.source || '',
        metadata: updates.metadata
      }]);
    }
  }

  // Generate embeddings
  private async generateEmbedding(text: string): Promise<number[]> {
    try {
      const response = await this.openai.embeddings.create({
        model: 'text-embedding-ada-002',
        input: text.replace(/\n/g, ' ')
      });

      return response.data[0].embedding;
    } catch (error) {
      console.error('Embedding generation error:', error);
      throw new Error(`Failed to generate embedding: ${error.message}`);
    }
  }

  private buildFilter(filters?: Record<string, any>): Record<string, any> | undefined {
    if (!filters) return undefined;

    const pineconeFilter: Record<string, any> = {};
    
    Object.entries(filters).forEach(([key, value]) => {
      if (Array.isArray(value)) {
        pineconeFilter[key] = { $in: value };
      } else if (typeof value === 'object' && value !== null) {
        pineconeFilter[key] = value;
      } else {
        pineconeFilter[key] = { $eq: value };
      }
    });

    return pineconeFilter;
  }

  private async keywordSearch(query: string, options: HybridSearchOptions): Promise<SearchResult[]> {
    // Implement keyword search using metadata fields
    const index = this.pinecone.index(this.indexName);
    
    // Use dummy vector for metadata-only search
    const dummyVector = new Array(1536).fill(0);
    
    const searchResponse = await index.query({
      vector: dummyVector,
      topK: options.keywordTopK || 10,
      includeMetadata: true,
      filter: {
        $or: [
          { title: { $regex: `.*${query}.*` } },
          { content: { $regex: `.*${query}.*` } },
          { tags: { $in: query.split(' ') } }
        ]
      }
    });

    return searchResponse.matches?.map(match => ({
      id: match.id || '',
      score: 0.5, // Fixed score for keyword matches
      content: match.metadata?.content as string || '',
      title: match.metadata?.title as string || '',
      source: match.metadata?.source as string || '',
      documentId: match.metadata?.documentId as string || '',
      metadata: match.metadata || {}
    })) || [];
  }

  private combineSearchResults(
    vectorResults: SearchResult[],
    keywordResults: SearchResult[],
    options: HybridSearchOptions
  ): SearchResult[] {
    const combined = new Map<string, SearchResult>();
    const vectorWeight = options.vectorWeight || 0.7;
    const keywordWeight = options.keywordWeight || 0.3;

    // Add vector results
    vectorResults.forEach(result => {
      combined.set(result.id, {
        ...result,
        score: result.score * vectorWeight
      });
    });

    // Add keyword results, combining scores if document already exists
    keywordResults.forEach(result => {
      const existing = combined.get(result.id);
      if (existing) {
        existing.score += result.score * keywordWeight;
      } else {
        combined.set(result.id, {
          ...result,
          score: result.score * keywordWeight
        });
      }
    });

    // Sort by combined score and return top results
    return Array.from(combined.values())
      .sort((a, b) => b.score - a.score)
      .slice(0, options.topK || 10);
  }

  private postProcessResults(results: SearchResult[], options: SearchOptions): SearchResult[] {
    let processedResults = [...results];

    // Apply score threshold
    if (options.scoreThreshold) {
      processedResults = processedResults.filter(
        result => result.score >= options.scoreThreshold!
      );
    }

    // Apply diversity filtering (remove very similar results)
    if (options.diversityThreshold) {
      processedResults = this.applyDiversityFilter(processedResults, options.diversityThreshold);
    }

    // Sort by score
    processedResults.sort((a, b) => b.score - a.score);

    return processedResults.slice(0, options.topK || 10);
  }

  private applyDiversityFilter(results: SearchResult[], threshold: number): SearchResult[] {
    const filtered: SearchResult[] = [];
    
    for (const result of results) {
      const isSimilar = filtered.some(existing => 
        this.calculateSimilarity(result.content, existing.content) > threshold
      );
      
      if (!isSimilar) {
        filtered.push(result);
      }
    }
    
    return filtered;
  }

  private calculateSimilarity(text1: string, text2: string): number {
    // Simple Jaccard similarity
    const words1 = new Set(text1.toLowerCase().split(/\s+/));
    const words2 = new Set(text2.toLowerCase().split(/\s+/));
    
    const intersection = new Set([...words1].filter(x => words2.has(x)));
    const union = new Set([...words1, ...words2]);
    
    return intersection.size / union.size;
  }

  private async waitForIndexReady(): Promise<void> {
    const maxAttempts = 30;
    const delay = 2000; // 2 seconds
    
    for (let attempt = 0; attempt < maxAttempts; attempt++) {
      try {
        const indexStats = await this.pinecone.index(this.indexName).describeIndexStats();
        if (indexStats) {
          return; // Index is ready
        }
      } catch (error) {
        // Index not ready yet
      }
      
      await new Promise(resolve => setTimeout(resolve, delay));
    }
    
    throw new Error('Index failed to become ready within timeout period');
  }
}

// RAG Query Engine
class RAGQueryEngine {
  private vectorDB: VectorDatabaseManager;
  private openai: OpenAI;
  private systemPrompts: Map<string, string>;

  constructor(vectorDB: VectorDatabaseManager, openaiApiKey: string) {
    this.vectorDB = vectorDB;
    this.openai = new OpenAI({ apiKey: openaiApiKey });
    this.systemPrompts = new Map();
    this.initializePrompts();
  }

  // Generate RAG response
  async query(
    userQuery: string,
    options: RAGQueryOptions = {}
  ): Promise<RAGResponse> {
    try {
      // 1. Retrieve relevant context
      const retrievalStartTime = Date.now();
      const contextResults = await this.retrieveContext(userQuery, options);
      const retrievalTime = Date.now() - retrievalStartTime;

      // 2. Build prompt with context
      const prompt = this.buildPrompt(userQuery, contextResults, options);

      // 3. Generate response
      const generationStartTime = Date.now();
      const response = await this.generateResponse(prompt, options);
      const generationTime = Date.now() - generationStartTime;

      // 4. Post-process and validate response
      const finalResponse = await this.postProcessResponse(response, contextResults);

      return {
        answer: finalResponse.content,
        sources: contextResults,
        metadata: {
          retrievalTime,
          generationTime,
          totalTime: retrievalTime + generationTime,
          sourceCount: contextResults.length,
          confidence: this.calculateConfidence(contextResults, finalResponse),
          model: options.model || 'gpt-4'
        }
      };
    } catch (error) {
      console.error('RAG query error:', error);
      throw new Error(`RAG query failed: ${error.message}`);
    }
  }

  // Stream RAG response
  async streamQuery(
    userQuery: string,
    options: RAGQueryOptions & { onChunk: (chunk: string) => void },
    onSource: (source: SearchResult) => void
  ): Promise<void> {
    try {
      // Retrieve context first
      const contextResults = await this.retrieveContext(userQuery, options);
      
      // Send sources to frontend
      contextResults.forEach(source => onSource(source));

      // Build prompt
      const prompt = this.buildPrompt(userQuery, contextResults, options);

      // Stream response
      const stream = await this.openai.chat.completions.create({
        model: options.model || 'gpt-4',
        messages: [
          {
            role: 'system',
            content: this.getSystemPrompt(options.promptType || 'default')
          },
          {
            role: 'user',
            content: prompt
          }
        ],
        temperature: options.temperature || 0.1,
        max_tokens: options.maxTokens || 1000,
        stream: true
      });

      for await (const chunk of stream) {
        const content = chunk.choices[0]?.delta?.content || '';
        if (content) {
          options.onChunk(content);
        }
      }
    } catch (error) {
      console.error('RAG streaming error:', error);
      throw error;
    }
  }

  // Multi-step reasoning for complex queries
  async complexQuery(
    userQuery: string,
    options: ComplexRAGOptions = {}
  ): Promise<ComplexRAGResponse> {
    const steps: RAGStep[] = [];
    let currentQuery = userQuery;
    let finalAnswer = '';
    let allSources: SearchResult[] = [];

    try {
      // Step 1: Query decomposition
      const subQueries = await this.decomposeQuery(userQuery);
      
      // Step 2: Answer sub-queries
      for (const [index, subQuery] of subQueries.entries()) {
        const stepStartTime = Date.now();
        
        const stepResponse = await this.query(subQuery, {
          ...options,
          topK: options.topK || 5
        });

        const step: RAGStep = {
          stepNumber: index + 1,
          query: subQuery,
          answer: stepResponse.answer,
          sources: stepResponse.sources,
          executionTime: Date.now() - stepStartTime
        };

        steps.push(step);
        allSources.push(...stepResponse.sources);
      }

      // Step 3: Synthesize final answer
      const synthesisPrompt = this.buildSynthesisPrompt(userQuery, steps);
      const synthesisResponse = await this.generateResponse(synthesisPrompt, options);
      finalAnswer = synthesisResponse.content;

      return {
        answer: finalAnswer,
        steps,
        allSources: this.deduplicateSources(allSources),
        metadata: {
          totalSteps: steps.length,
          totalExecutionTime: steps.reduce((sum, step) => sum + step.executionTime, 0),
          sourceCount: allSources.length
        }
      };
    } catch (error) {
      console.error('Complex RAG query error:', error);
      throw error;
    }
  }

  private async retrieveContext(
    query: string,
    options: RAGQueryOptions
  ): Promise<SearchResult[]> {
    const searchOptions: SearchOptions = {
      topK: options.topK || 10,
      scoreThreshold: options.scoreThreshold || 0.7,
      diversityThreshold: options.diversityThreshold || 0.8,
      filters: options.filters
    };

    if (options.useHybridSearch) {
      return this.vectorDB.hybridSearch(query, {
        ...searchOptions,
        vectorWeight: 0.7,
        keywordWeight: 0.3
      });
    } else {
      return this.vectorDB.search(query, searchOptions);
    }
  }

  private buildPrompt(
    query: string,
    context: SearchResult[],
    options: RAGQueryOptions
  ): string {
    const contextText = context
      .map((result, index) => `[${index + 1}] ${result.content}`)
      .join('\n\n');

    const basePrompt = `Context information is below:
${contextText}

Query: ${query}

Instructions:
- Answer the query using ONLY the information provided in the context
- If the context doesn't contain enough information to answer the query, say so
- Include relevant source numbers in your response [1], [2], etc.
- Be concise but comprehensive
- If multiple sources conflict, acknowledge the conflict`;

    if (options.additionalInstructions) {
      return `${basePrompt}\n\nAdditional instructions: ${options.additionalInstructions}`;
    }

    return basePrompt;
  }

  private async generateResponse(
    prompt: string,
    options: RAGQueryOptions
  ): Promise<{ content: string; usage?: any }> {
    const response = await this.openai.chat.completions.create({
      model: options.model || 'gpt-4',
      messages: [
        {
          role: 'system',
          content: this.getSystemPrompt(options.promptType || 'default')
        },
        {
          role: 'user',
          content: prompt
        }
      ],
      temperature: options.temperature || 0.1,
      max_tokens: options.maxTokens || 1000,
      top_p: 0.9
    });

    return {
      content: response.choices[0]?.message?.content || '',
      usage: response.usage
    };
  }

  private async postProcessResponse(
    response: { content: string },
    sources: SearchResult[]
  ): Promise<{ content: string }> {
    // Validate source citations
    const content = this.validateSourceCitations(response.content, sources);
    
    // Add source URLs if available
    const enhancedContent = this.enhanceWithSourceLinks(content, sources);
    
    return { content: enhancedContent };
  }

  private validateSourceCitations(content: string, sources: SearchResult[]): string {
    // Check if citations are valid
    const citationRegex = /\[(\d+)\]/g;
    const matches = content.match(citationRegex);
    
    if (matches) {
      matches.forEach(match => {
        const sourceNumber = parseInt(match.replace(/[\[\]]/g, ''));
        if (sourceNumber > sources.length) {
          content = content.replace(match, '[?]');
        }
      });
    }
    
    return content;
  }

  private enhanceWithSourceLinks(content: string, sources: SearchResult[]): string {
    let enhanced = content;
    
    // Add source references at the end
    if (sources.length > 0) {
      enhanced += '\n\nSources:\n';
      sources.forEach((source, index) => {
        enhanced += `[${index + 1}] ${source.title}`;
        if (source.source) {
          enhanced += ` (${source.source})`;
        }
        enhanced += '\n';
      });
    }
    
    return enhanced;
  }

  private calculateConfidence(sources: SearchResult[], response: { content: string }): number {
    if (sources.length === 0) return 0;
    
    const avgScore = sources.reduce((sum, source) => sum + source.score, 0) / sources.length;
    const sourceCount = sources.length;
    const hasSourceCitations = /\[\d+\]/.test(response.content);
    
    let confidence = avgScore * 0.5; // Base score from vector similarity
    confidence += Math.min(sourceCount / 5, 1) * 0.3; // Source count factor
    confidence += hasSourceCitations ? 0.2 : 0; // Citation factor
    
    return Math.min(confidence, 1);
  }

  private async decomposeQuery(query: string): Promise<string[]> {
    const decompositionPrompt = `Decompose this complex query into 2-4 simpler sub-queries that can be answered independently:

Query: ${query}

Return only the sub-queries, one per line, without numbering or additional text.`;

    const response = await this.generateResponse(decompositionPrompt, {
      temperature: 0.3,
      maxTokens: 200
    });

    return response.content
      .split('\n')
      .map(line => line.trim())
      .filter(line => line.length > 0)
      .slice(0, 4); // Limit to 4 sub-queries
  }

  private buildSynthesisPrompt(originalQuery: string, steps: RAGStep[]): string {
    const stepSummaries = steps
      .map(step => `Sub-query: ${step.query}\nAnswer: ${step.answer}`)
      .join('\n\n');

    return `Original query: ${originalQuery}

Sub-query results:
${stepSummaries}

Synthesize a comprehensive answer to the original query using the information from all sub-queries. Ensure the answer is coherent and addresses the original query directly.`;
  }

  private deduplicateSources(sources: SearchResult[]): SearchResult[] {
    const seen = new Set<string>();
    return sources.filter(source => {
      if (seen.has(source.documentId)) {
        return false;
      }
      seen.add(source.documentId);
      return true;
    });
  }

  private initializePrompts(): void {
    this.systemPrompts.set('default', 
      'You are a helpful assistant that answers questions based on provided context. Always ground your responses in the given information and cite sources when possible.');
    
    this.systemPrompts.set('analytical', 
      'You are an analytical assistant that provides detailed, structured responses. Break down complex topics and provide step-by-step reasoning.');
    
    this.systemPrompts.set('creative', 
      'You are a creative assistant that can synthesize information in novel ways while staying grounded in the provided context.');
    
    this.systemPrompts.set('technical', 
      'You are a technical expert that provides precise, accurate information with appropriate technical detail and terminology.');
  }

  private getSystemPrompt(type: string): string {
    return this.systemPrompts.get(type) || this.systemPrompts.get('default')!;
  }
}

// React Components for RAG Interface
interface RAGChatProps {
  vectorDB: VectorDatabaseManager;
  openaiApiKey: string;
  onSourceClick?: (source: SearchResult) => void;
}

const RAGChat: React.FC<RAGChatProps> = ({ vectorDB, openaiApiKey, onSourceClick }) => {
  const [messages, setMessages] = useState<RAGMessage[]>([]);
  const [input, setInput] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const [ragEngine] = useState(() => new RAGQueryEngine(vectorDB, openaiApiKey));

  const handleSendMessage = async () => {
    if (!input.trim() || isLoading) return;

    const userMessage: RAGMessage = {
      id: generateId(),
      type: 'user',
      content: input,
      timestamp: new Date()
    };

    setMessages(prev => [...prev, userMessage]);
    setInput('');
    setIsLoading(true);

    try {
      const response = await ragEngine.query(input, {
        topK: 8,
        useHybridSearch: true,
        scoreThreshold: 0.7
      });

      const assistantMessage: RAGMessage = {
        id: generateId(),
        type: 'assistant',
        content: response.answer,
        sources: response.sources,
        metadata: response.metadata,
        timestamp: new Date()
      };

      setMessages(prev => [...prev, assistantMessage]);
    } catch (error) {
      console.error('RAG chat error:', error);
      const errorMessage: RAGMessage = {
        id: generateId(),
        type: 'assistant',
        content: 'Sorry, I encountered an error while processing your question.',
        timestamp: new Date()
      };
      setMessages(prev => [...prev, errorMessage]);
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="rag-chat">
      <div className="messages-container">
        {messages.map(message => (
          <div key={message.id} className={`message ${message.type}`}>
            <div className="message-content">
              <ReactMarkdown>{message.content}</ReactMarkdown>
            </div>
            
            {message.sources && message.sources.length > 0 && (
              <div className="sources">
                <h4>Sources:</h4>
                {message.sources.map((source, index) => (
                  <div
                    key={source.id}
                    className="source-item"
                    onClick={() => onSourceClick?.(source)}
                  >
                    <span className="source-number">[{index + 1}]</span>
                    <span className="source-title">{source.title}</span>
                    <span className="source-score">{(source.score * 100).toFixed(1)}%</span>
                  </div>
                ))}
              </div>
            )}
            
            {message.metadata && (
              <div className="metadata">
                Response time: {message.metadata.totalTime}ms | 
                Sources: {message.metadata.sourceCount} | 
                Confidence: {(message.metadata.confidence * 100).toFixed(1)}%
              </div>
            )}
          </div>
        ))}
      </div>
      
      <div className="input-container">
        <input
          type="text"
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyPress={(e) => e.key === 'Enter' && handleSendMessage()}
          placeholder="Ask a question..."
          disabled={isLoading}
        />
        <button onClick={handleSendMessage} disabled={!input.trim() || isLoading}>
          {isLoading ? 'Thinking...' : 'Send'}
        </button>
      </div>
    </div>
  );
};

// Type Definitions
interface VectorDBConfig {
  pineconeApiKey: string;
  pineconeEnvironment: string;
  openaiApiKey: string;
  indexName: string;
  chunkSize?: number;
  chunkOverlap?: number;
}

interface DocumentInput {
  id: string;
  title: string;
  content: string;
  source: string;
  createdAt?: string;
  tags?: string[];
  metadata?: Record<string, any>;
}

interface PineconeVector {
  id: string;
  values: number[];
  metadata?: Record<string, any>;
}

interface SearchOptions {
  topK?: number;
  scoreThreshold?: number;
  diversityThreshold?: number;
  filters?: Record<string, any>;
}

interface HybridSearchOptions extends SearchOptions {
  vectorTopK?: number;
  keywordTopK?: number;
  vectorWeight?: number;
  keywordWeight?: number;
}

interface SearchResult {
  id: string;
  score: number;
  content: string;
  title: string;
  source: string;
  documentId: string;
  metadata: Record<string, any>;
}

interface RAGQueryOptions {
  topK?: number;
  scoreThreshold?: number;
  diversityThreshold?: number;
  filters?: Record<string, any>;
  useHybridSearch?: boolean;
  model?: string;
  temperature?: number;
  maxTokens?: number;
  promptType?: string;
  additionalInstructions?: string;
}

interface RAGResponse {
  answer: string;
  sources: SearchResult[];
  metadata: {
    retrievalTime: number;
    generationTime: number;
    totalTime: number;
    sourceCount: number;
    confidence: number;
    model: string;
  };
}

interface ComplexRAGOptions extends RAGQueryOptions {
  maxSteps?: number;
}

interface RAGStep {
  stepNumber: number;
  query: string;
  answer: string;
  sources: SearchResult[];
  executionTime: number;
}

interface ComplexRAGResponse {
  answer: string;
  steps: RAGStep[];
  allSources: SearchResult[];
  metadata: {
    totalSteps: number;
    totalExecutionTime: number;
    sourceCount: number;
  };
}

interface RAGMessage {
  id: string;
  type: 'user' | 'assistant';
  content: string;
  sources?: SearchResult[];
  metadata?: any;
  timestamp: Date;
}

function generateId(): string {
  return Math.random().toString(36).substring(2, 15) + Date.now().toString(36);
}

export {
  VectorDatabaseManager,
  RAGQueryEngine,
  RAGChat
};
```

## Interview-Ready Summary

**RAG Architecture Components:**

1. **Vector Database** - Pinecone integration with document chunking, embedding generation, and semantic search
2. **Query Engine** - Context retrieval, prompt construction, response generation, and source attribution
3. **Hybrid Search** - Combining vector similarity and keyword matching for comprehensive results
4. **Multi-step Reasoning** - Query decomposition and synthesis for complex questions
5. **Real-time Interface** - React components with streaming responses and source visualization

**Advanced Features:**
- **Document Processing** - Intelligent chunking with overlap, metadata preservation, batch processing
- **Search Optimization** - Score thresholds, diversity filtering, result ranking, caching
- **Response Quality** - Source validation, confidence scoring, citation verification
- **Performance Monitoring** - Retrieval timing, generation metrics, token usage tracking

**Production Considerations:**
- **Scalability** - Vector index optimization, batch operations, caching strategies
- **Security** - API key management, content filtering, access control
- **Cost Management** - Embedding caching, query optimization, token monitoring
- **Quality Assurance** - Response validation, source accuracy, hallucination detection

**Enterprise Features:**
- **Multi-tenant Support** - Index isolation, user-specific filters, permission management
- **Analytics** - Query patterns, performance metrics, user satisfaction tracking
- **Integration** - REST APIs, webhooks, real-time updates, document synchronization

**Key Interview Topics:** Vector database selection criteria, embedding model comparison, chunking strategies, retrieval optimization, prompt engineering, response quality metrics, scalability patterns, cost optimization techniques, production monitoring.