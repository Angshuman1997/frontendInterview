# LLM Integration and AI-Powered Frontend Features

Large Language Models (LLMs) are revolutionizing frontend development by enabling intelligent, context-aware applications. This guide covers practical LLM integration patterns, AI-powered UI components, and enterprise-grade implementation strategies for modern web applications.

## Frontend LLM Integration Architecture

### 1. OpenAI GPT Integration with React
```typescript
// Advanced OpenAI Integration with React and TypeScript
import React, { useState, useCallback, useRef, useEffect } from 'react';
import OpenAI from 'openai';

// OpenAI Client Configuration
class OpenAIClient {
  private client: OpenAI;
  private rateLimiter: RateLimiter;
  private cache: LRUCache<string, any>;

  constructor(apiKey: string) {
    this.client = new OpenAI({
      apiKey,
      dangerouslyAllowBrowser: false, // Use backend proxy in production
    });
    
    this.rateLimiter = new RateLimiter({
      tokensPerMinute: 60000,
      requestsPerMinute: 1000
    });
    
    this.cache = new LRUCache({ max: 1000, ttl: 300000 }); // 5-minute cache
  }

  async generateCompletion(
    prompt: string,
    options: CompletionOptions = {}
  ): Promise<CompletionResponse> {
    const cacheKey = this.getCacheKey(prompt, options);
    const cached = this.cache.get(cacheKey);
    
    if (cached && options.useCache !== false) {
      return cached;
    }

    await this.rateLimiter.waitForToken();

    try {
      const response = await this.client.chat.completions.create({
        model: options.model || 'gpt-4',
        messages: [
          {
            role: 'system',
            content: options.systemPrompt || 'You are a helpful assistant.'
          },
          {
            role: 'user',
            content: prompt
          }
        ],
        temperature: options.temperature || 0.7,
        max_tokens: options.maxTokens || 2048,
        top_p: options.topP || 1,
        frequency_penalty: options.frequencyPenalty || 0,
        presence_penalty: options.presencePenalty || 0,
        stream: options.stream || false
      });

      const result: CompletionResponse = {
        content: response.choices[0]?.message?.content || '',
        usage: response.usage,
        model: response.model,
        id: response.id,
        created: response.created
      };

      if (options.useCache !== false) {
        this.cache.set(cacheKey, result);
      }

      return result;
    } catch (error) {
      console.error('OpenAI API Error:', error);
      throw new APIError('Failed to generate completion', error);
    }
  }

  async generateStreamCompletion(
    prompt: string,
    options: CompletionOptions,
    onChunk: (chunk: string) => void
  ): Promise<void> {
    await this.rateLimiter.waitForToken();

    try {
      const stream = await this.client.chat.completions.create({
        model: options.model || 'gpt-4',
        messages: [
          {
            role: 'system',
            content: options.systemPrompt || 'You are a helpful assistant.'
          },
          {
            role: 'user',
            content: prompt
          }
        ],
        temperature: options.temperature || 0.7,
        max_tokens: options.maxTokens || 2048,
        stream: true
      });

      for await (const chunk of stream) {
        const content = chunk.choices[0]?.delta?.content || '';
        if (content) {
          onChunk(content);
        }
      }
    } catch (error) {
      console.error('OpenAI Streaming Error:', error);
      throw new APIError('Failed to generate streaming completion', error);
    }
  }

  async generateEmbedding(text: string): Promise<number[]> {
    const cacheKey = `embedding:${this.hashString(text)}`;
    const cached = this.cache.get(cacheKey);
    
    if (cached) {
      return cached;
    }

    await this.rateLimiter.waitForToken();

    try {
      const response = await this.client.embeddings.create({
        model: 'text-embedding-ada-002',
        input: text
      });

      const embedding = response.data[0].embedding;
      this.cache.set(cacheKey, embedding);
      
      return embedding;
    } catch (error) {
      console.error('OpenAI Embedding Error:', error);
      throw new APIError('Failed to generate embedding', error);
    }
  }

  private getCacheKey(prompt: string, options: CompletionOptions): string {
    return `completion:${this.hashString(prompt + JSON.stringify(options))}`;
  }

  private hashString(str: string): string {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash; // Convert to 32-bit integer
    }
    return hash.toString();
  }
}

// AI-Powered Chat Component
interface ChatMessage {
  id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  timestamp: Date;
  metadata?: {
    model?: string;
    tokens?: number;
    processingTime?: number;
  };
}

interface AIChatProps {
  systemPrompt?: string;
  model?: string;
  onMessageSent?: (message: ChatMessage) => void;
  onResponseReceived?: (message: ChatMessage) => void;
  placeholder?: string;
  maxMessages?: number;
  enableFunctionCalling?: boolean;
  className?: string;
}

const AIChat: React.FC<AIChatProps> = ({
  systemPrompt = "You are a helpful assistant.",
  model = "gpt-4",
  onMessageSent,
  onResponseReceived,
  placeholder = "Type your message...",
  maxMessages = 100,
  enableFunctionCalling = false,
  className
}) => {
  const [messages, setMessages] = useState<ChatMessage[]>([]);
  const [input, setInput] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const [streamingResponse, setStreamingResponse] = useState('');
  const messagesEndRef = useRef<HTMLDivElement>(null);
  const openAIClient = useRef(new OpenAIClient(process.env.REACT_APP_OPENAI_API_KEY!));

  const scrollToBottom = () => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  };

  useEffect(() => {
    scrollToBottom();
  }, [messages, streamingResponse]);

  const handleSendMessage = useCallback(async () => {
    if (!input.trim() || isLoading) return;

    const userMessage: ChatMessage = {
      id: generateId(),
      role: 'user',
      content: input.trim(),
      timestamp: new Date()
    };

    setMessages(prev => [...prev, userMessage]);
    setInput('');
    setIsLoading(true);
    setStreamingResponse('');

    onMessageSent?.(userMessage);

    try {
      const startTime = Date.now();
      let fullResponse = '';

      // Use streaming for better UX
      await openAIClient.current.generateStreamCompletion(
        input.trim(),
        {
          model,
          systemPrompt,
          stream: true,
          maxTokens: 2048
        },
        (chunk) => {
          fullResponse += chunk;
          setStreamingResponse(fullResponse);
        }
      );

      const assistantMessage: ChatMessage = {
        id: generateId(),
        role: 'assistant',
        content: fullResponse,
        timestamp: new Date(),
        metadata: {
          model,
          processingTime: Date.now() - startTime
        }
      };

      setMessages(prev => [...prev, assistantMessage]);
      setStreamingResponse('');
      onResponseReceived?.(assistantMessage);

    } catch (error) {
      console.error('Chat error:', error);
      const errorMessage: ChatMessage = {
        id: generateId(),
        role: 'assistant',
        content: 'Sorry, I encountered an error while processing your request.',
        timestamp: new Date()
      };
      setMessages(prev => [...prev, errorMessage]);
    } finally {
      setIsLoading(false);
    }
  }, [input, isLoading, model, systemPrompt, onMessageSent, onResponseReceived]);

  const handleKeyPress = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      handleSendMessage();
    }
  };

  const clearConversation = () => {
    setMessages([]);
    setStreamingResponse('');
  };

  return (
    <div className={`ai-chat ${className || ''}`}>
      <div className="chat-header">
        <h3>AI Assistant</h3>
        <button onClick={clearConversation} className="clear-btn">
          Clear
        </button>
      </div>
      
      <div className="messages-container">
        {messages.map((message) => (
          <div key={message.id} className={`message ${message.role}`}>
            <div className="message-content">
              <ReactMarkdown>{message.content}</ReactMarkdown>
            </div>
            <div className="message-metadata">
              {message.timestamp.toLocaleTimeString()}
              {message.metadata?.processingTime && (
                <span> • {message.metadata.processingTime}ms</span>
              )}
            </div>
          </div>
        ))}
        
        {streamingResponse && (
          <div className="message assistant streaming">
            <div className="message-content">
              <ReactMarkdown>{streamingResponse}</ReactMarkdown>
            </div>
            <div className="typing-indicator">AI is typing...</div>
          </div>
        )}
        
        <div ref={messagesEndRef} />
      </div>
      
      <div className="input-container">
        <textarea
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyPress={handleKeyPress}
          placeholder={placeholder}
          disabled={isLoading}
          rows={3}
          className="message-input"
        />
        <button
          onClick={handleSendMessage}
          disabled={!input.trim() || isLoading}
          className="send-button"
        >
          {isLoading ? 'Sending...' : 'Send'}
        </button>
      </div>
    </div>
  );
};

// AI-Powered Text Editor with Auto-completion
interface AITextEditorProps {
  initialValue?: string;
  onChange?: (value: string) => void;
  onAIAssist?: (suggestion: string) => void;
  placeholder?: string;
  enableAutoComplete?: boolean;
  enableGrammarCheck?: boolean;
  enableToneAnalysis?: boolean;
}

const AITextEditor: React.FC<AITextEditorProps> = ({
  initialValue = '',
  onChange,
  onAIAssist,
  placeholder = 'Start writing...',
  enableAutoComplete = true,
  enableGrammarCheck = true,
  enableToneAnalysis = false
}) => {
  const [content, setContent] = useState(initialValue);
  const [suggestions, setSuggestions] = useState<AISuggestion[]>([]);
  const [isAnalyzing, setIsAnalyzing] = useState(false);
  const [cursorPosition, setCursorPosition] = useState(0);
  const editorRef = useRef<HTMLTextAreaElement>(null);
  const debounceTimeoutRef = useRef<NodeJS.Timeout>();
  const openAIClient = useRef(new OpenAIClient(process.env.REACT_APP_OPENAI_API_KEY!));

  const handleContentChange = (newContent: string) => {
    setContent(newContent);
    onChange?.(newContent);

    // Debounce AI analysis
    if (debounceTimeoutRef.current) {
      clearTimeout(debounceTimeoutRef.current);
    }

    debounceTimeoutRef.current = setTimeout(() => {
      if (enableAutoComplete || enableGrammarCheck) {
        analyzeContent(newContent);
      }
    }, 1000);
  };

  const analyzeContent = async (text: string) => {
    if (!text.trim() || text.length < 10) return;

    setIsAnalyzing(true);
    
    try {
      const analysisPrompts = [];

      if (enableAutoComplete) {
        analysisPrompts.push(
          openAIClient.current.generateCompletion(
            `Complete this text naturally and concisely (max 50 words): "${text}"`,
            {
              systemPrompt: 'You are a writing assistant. Provide natural completions for text.',
              maxTokens: 100,
              temperature: 0.7
            }
          )
        );
      }

      if (enableGrammarCheck) {
        analysisPrompts.push(
          openAIClient.current.generateCompletion(
            `Check for grammar and style issues in this text: "${text}"`,
            {
              systemPrompt: 'You are a grammar checker. Identify issues and suggest improvements.',
              maxTokens: 200,
              temperature: 0.3
            }
          )
        );
      }

      if (enableToneAnalysis) {
        analysisPrompts.push(
          openAIClient.current.generateCompletion(
            `Analyze the tone of this text: "${text}"`,
            {
              systemPrompt: 'You are a tone analyzer. Identify the tone and suggest improvements.',
              maxTokens: 100,
              temperature: 0.3
            }
          )
        );
      }

      const results = await Promise.allSettled(analysisPrompts);
      const newSuggestions: AISuggestion[] = [];

      results.forEach((result, index) => {
        if (result.status === 'fulfilled') {
          const suggestion: AISuggestion = {
            id: generateId(),
            type: index === 0 ? 'completion' : index === 1 ? 'grammar' : 'tone',
            content: result.value.content,
            position: text.length
          };
          newSuggestions.push(suggestion);
        }
      });

      setSuggestions(newSuggestions);
    } catch (error) {
      console.error('AI analysis error:', error);
    } finally {
      setIsAnalyzing(false);
    }
  };

  const applySuggestion = (suggestion: AISuggestion) => {
    if (suggestion.type === 'completion') {
      const newContent = content + ' ' + suggestion.content;
      setContent(newContent);
      onChange?.(newContent);
    } else if (suggestion.type === 'grammar') {
      // For grammar suggestions, we'd need more sophisticated replacement logic
      onAIAssist?.(suggestion.content);
    }
    
    setSuggestions(prev => prev.filter(s => s.id !== suggestion.id));
  };

  const dismissSuggestion = (suggestionId: string) => {
    setSuggestions(prev => prev.filter(s => s.id !== suggestionId));
  };

  return (
    <div className="ai-text-editor">
      <div className="editor-container">
        <textarea
          ref={editorRef}
          value={content}
          onChange={(e) => handleContentChange(e.target.value)}
          placeholder={placeholder}
          className="editor-textarea"
          onSelect={(e) => {
            const target = e.target as HTMLTextAreaElement;
            setCursorPosition(target.selectionStart);
          }}
        />
        
        {isAnalyzing && (
          <div className="analysis-indicator">
            AI analyzing...
          </div>
        )}
      </div>
      
      {suggestions.length > 0 && (
        <div className="suggestions-panel">
          <h4>AI Suggestions</h4>
          {suggestions.map((suggestion) => (
            <div key={suggestion.id} className={`suggestion ${suggestion.type}`}>
              <div className="suggestion-type">
                {suggestion.type.charAt(0).toUpperCase() + suggestion.type.slice(1)}
              </div>
              <div className="suggestion-content">
                {suggestion.content}
              </div>
              <div className="suggestion-actions">
                <button
                  onClick={() => applySuggestion(suggestion)}
                  className="apply-btn"
                >
                  Apply
                </button>
                <button
                  onClick={() => dismissSuggestion(suggestion.id)}
                  className="dismiss-btn"
                >
                  Dismiss
                </button>
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};

// AI-Powered Search Component
interface AISearchProps {
  onSearch: (query: string, filters?: SearchFilters) => Promise<SearchResult[]>;
  onSuggestion?: (suggestion: string) => void;
  placeholder?: string;
  enableSemanticSearch?: boolean;
  enableQueryExpansion?: boolean;
  enableAutoComplete?: boolean;
}

const AISearch: React.FC<AISearchProps> = ({
  onSearch,
  onSuggestion,
  placeholder = 'Search with AI...',
  enableSemanticSearch = true,
  enableQueryExpansion = true,
  enableAutoComplete = true
}) => {
  const [query, setQuery] = useState('');
  const [suggestions, setSuggestions] = useState<string[]>([]);
  const [isSearching, setIsSearching] = useState(false);
  const [expandedQuery, setExpandedQuery] = useState('');
  const debounceTimeoutRef = useRef<NodeJS.Timeout>();
  const openAIClient = useRef(new OpenAIClient(process.env.REACT_APP_OPENAI_API_KEY!));

  const handleQueryChange = (newQuery: string) => {
    setQuery(newQuery);

    // Debounce suggestions
    if (debounceTimeoutRef.current) {
      clearTimeout(debounceTimeoutRef.current);
    }

    debounceTimeoutRef.current = setTimeout(() => {
      if (newQuery.length > 2 && enableAutoComplete) {
        generateSuggestions(newQuery);
      } else {
        setSuggestions([]);
      }
    }, 300);
  };

  const generateSuggestions = async (searchQuery: string) => {
    try {
      const response = await openAIClient.current.generateCompletion(
        `Generate 5 search suggestions related to: "${searchQuery}"`,
        {
          systemPrompt: 'You are a search assistant. Generate relevant, concise search suggestions.',
          maxTokens: 150,
          temperature: 0.7
        }
      );

      const suggestionLines = response.content
        .split('\n')
        .filter(line => line.trim())
        .slice(0, 5)
        .map(line => line.replace(/^\d+\.\s*/, '').trim());

      setSuggestions(suggestionLines);
    } catch (error) {
      console.error('Error generating suggestions:', error);
    }
  };

  const expandQuery = async (originalQuery: string): Promise<string> => {
    if (!enableQueryExpansion) return originalQuery;

    try {
      const response = await openAIClient.current.generateCompletion(
        `Expand this search query with relevant keywords and synonyms: "${originalQuery}"`,
        {
          systemPrompt: 'You are a search query optimizer. Expand queries with relevant terms while keeping them concise.',
          maxTokens: 100,
          temperature: 0.5
        }
      );

      return response.content.trim();
    } catch (error) {
      console.error('Error expanding query:', error);
      return originalQuery;
    }
  };

  const handleSearch = async (searchQuery: string = query) => {
    if (!searchQuery.trim()) return;

    setIsSearching(true);
    setSuggestions([]);

    try {
      let finalQuery = searchQuery;

      if (enableQueryExpansion) {
        finalQuery = await expandQuery(searchQuery);
        setExpandedQuery(finalQuery);
      }

      const results = await onSearch(finalQuery, {
        semantic: enableSemanticSearch,
        originalQuery: searchQuery,
        expandedQuery: finalQuery
      });

      // Optional: Generate search insights
      if (results.length === 0) {
        const suggestion = await generateNoResultsSuggestion(searchQuery);
        onSuggestion?.(suggestion);
      }

    } catch (error) {
      console.error('Search error:', error);
    } finally {
      setIsSearching(false);
    }
  };

  const generateNoResultsSuggestion = async (searchQuery: string): Promise<string> => {
    try {
      const response = await openAIClient.current.generateCompletion(
        `No results found for "${searchQuery}". Suggest alternative search terms or approaches.`,
        {
          systemPrompt: 'You are a search assistant. Provide helpful suggestions when searches return no results.',
          maxTokens: 100,
          temperature: 0.7
        }
      );

      return response.content.trim();
    } catch (error) {
      return 'Try using different keywords or broader search terms.';
    }
  };

  const handleKeyPress = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter') {
      handleSearch();
    }
  };

  return (
    <div className="ai-search">
      <div className="search-input-container">
        <input
          type="text"
          value={query}
          onChange={(e) => handleQueryChange(e.target.value)}
          onKeyPress={handleKeyPress}
          placeholder={placeholder}
          className="search-input"
          disabled={isSearching}
        />
        <button
          onClick={() => handleSearch()}
          disabled={!query.trim() || isSearching}
          className="search-button"
        >
          {isSearching ? 'Searching...' : 'Search'}
        </button>
      </div>

      {expandedQuery && expandedQuery !== query && (
        <div className="expanded-query">
          Searching for: <em>{expandedQuery}</em>
        </div>
      )}

      {suggestions.length > 0 && (
        <div className="suggestions-dropdown">
          {suggestions.map((suggestion, index) => (
            <div
              key={index}
              className="suggestion-item"
              onClick={() => {
                setQuery(suggestion);
                handleSearch(suggestion);
              }}
            >
              {suggestion}
            </div>
          ))}
        </div>
      )}
    </div>
  );
};

// Utility Classes and Interfaces
class RateLimiter {
  private tokens: number;
  private lastRefill: number;
  private maxTokens: number;
  private refillRate: number; // tokens per millisecond

  constructor(config: { tokensPerMinute: number; requestsPerMinute: number }) {
    this.maxTokens = config.tokensPerMinute;
    this.tokens = this.maxTokens;
    this.lastRefill = Date.now();
    this.refillRate = config.tokensPerMinute / (60 * 1000); // per millisecond
  }

  async waitForToken(): Promise<void> {
    this.refillTokens();
    
    if (this.tokens >= 1) {
      this.tokens -= 1;
      return;
    }

    // Wait until we have tokens
    const waitTime = (1 / this.refillRate);
    await new Promise(resolve => setTimeout(resolve, waitTime));
    return this.waitForToken();
  }

  private refillTokens(): void {
    const now = Date.now();
    const timePassed = now - this.lastRefill;
    const tokensToAdd = timePassed * this.refillRate;
    
    this.tokens = Math.min(this.maxTokens, this.tokens + tokensToAdd);
    this.lastRefill = now;
  }
}

class LRUCache<K, V> {
  private cache = new Map<K, V>();
  private maxSize: number;
  private ttl: number;
  private timestamps = new Map<K, number>();

  constructor(options: { max: number; ttl: number }) {
    this.maxSize = options.max;
    this.ttl = options.ttl;
  }

  get(key: K): V | undefined {
    const timestamp = this.timestamps.get(key);
    if (timestamp && Date.now() - timestamp > this.ttl) {
      this.delete(key);
      return undefined;
    }

    const value = this.cache.get(key);
    if (value !== undefined) {
      // Move to end (most recently used)
      this.cache.delete(key);
      this.cache.set(key, value);
    }
    return value;
  }

  set(key: K, value: V): void {
    if (this.cache.has(key)) {
      this.cache.delete(key);
    } else if (this.cache.size >= this.maxSize) {
      // Remove least recently used
      const firstKey = this.cache.keys().next().value;
      this.delete(firstKey);
    }

    this.cache.set(key, value);
    this.timestamps.set(key, Date.now());
  }

  delete(key: K): void {
    this.cache.delete(key);
    this.timestamps.delete(key);
  }
}

// Type Definitions
interface CompletionOptions {
  model?: string;
  systemPrompt?: string;
  temperature?: number;
  maxTokens?: number;
  topP?: number;
  frequencyPenalty?: number;
  presencePenalty?: number;
  stream?: boolean;
  useCache?: boolean;
}

interface CompletionResponse {
  content: string;
  usage?: {
    prompt_tokens: number;
    completion_tokens: number;
    total_tokens: number;
  };
  model: string;
  id: string;
  created: number;
}

interface AISuggestion {
  id: string;
  type: 'completion' | 'grammar' | 'tone';
  content: string;
  position: number;
}

interface SearchFilters {
  semantic?: boolean;
  originalQuery?: string;
  expandedQuery?: string;
}

interface SearchResult {
  id: string;
  title: string;
  content: string;
  score: number;
  metadata?: Record<string, any>;
}

class APIError extends Error {
  constructor(message: string, public originalError?: any) {
    super(message);
    this.name = 'APIError';
  }
}

function generateId(): string {
  return Math.random().toString(36).substring(2, 15) + Date.now().toString(36);
}

export {
  OpenAIClient,
  AIChat,
  AITextEditor,
  AISearch,
  RateLimiter,
  LRUCache
};
```

### 2. Advanced AI Hooks and Context
```typescript
// AI Context Provider and Custom Hooks
import React, { createContext, useContext, useReducer, useCallback } from 'react';

// AI Context State Management
interface AIState {
  isInitialized: boolean;
  apiKey: string | null;
  models: AIModel[];
  activeModel: string;
  usage: UsageStats;
  cache: Map<string, CachedResponse>;
  rateLimits: RateLimit;
  error: string | null;
}

interface AIModel {
  id: string;
  name: string;
  provider: 'openai' | 'anthropic' | 'google' | 'local';
  type: 'chat' | 'completion' | 'embedding' | 'image';
  maxTokens: number;
  costPer1kTokens: number;
  available: boolean;
}

interface UsageStats {
  totalTokens: number;
  totalRequests: number;
  totalCost: number;
  requestsToday: number;
  tokensToday: number;
  lastReset: Date;
}

interface RateLimit {
  requestsPerMinute: number;
  tokensPerMinute: number;
  currentRequests: number;
  currentTokens: number;
  windowStart: Date;
}

interface CachedResponse {
  response: any;
  timestamp: Date;
  ttl: number;
}

// AI Actions
type AIAction =
  | { type: 'INITIALIZE'; payload: { apiKey: string; models: AIModel[] } }
  | { type: 'SET_MODEL'; payload: string }
  | { type: 'UPDATE_USAGE'; payload: { tokens: number; cost: number } }
  | { type: 'UPDATE_RATE_LIMIT'; payload: { requests: number; tokens: number } }
  | { type: 'CACHE_RESPONSE'; payload: { key: string; response: any; ttl: number } }
  | { type: 'SET_ERROR'; payload: string }
  | { type: 'CLEAR_ERROR' };

// AI Reducer
const aiReducer = (state: AIState, action: AIAction): AIState => {
  switch (action.type) {
    case 'INITIALIZE':
      return {
        ...state,
        isInitialized: true,
        apiKey: action.payload.apiKey,
        models: action.payload.models,
        error: null
      };

    case 'SET_MODEL':
      return {
        ...state,
        activeModel: action.payload
      };

    case 'UPDATE_USAGE':
      const now = new Date();
      const isNewDay = now.toDateString() !== state.usage.lastReset.toDateString();
      
      return {
        ...state,
        usage: {
          ...state.usage,
          totalTokens: state.usage.totalTokens + action.payload.tokens,
          totalRequests: state.usage.totalRequests + 1,
          totalCost: state.usage.totalCost + action.payload.cost,
          requestsToday: isNewDay ? 1 : state.usage.requestsToday + 1,
          tokensToday: isNewDay ? action.payload.tokens : state.usage.tokensToday + action.payload.tokens,
          lastReset: isNewDay ? now : state.usage.lastReset
        }
      };

    case 'UPDATE_RATE_LIMIT':
      const windowStart = new Date();
      windowStart.setMinutes(windowStart.getMinutes() - 1);
      
      return {
        ...state,
        rateLimits: {
          ...state.rateLimits,
          currentRequests: action.payload.requests,
          currentTokens: action.payload.tokens,
          windowStart
        }
      };

    case 'CACHE_RESPONSE':
      const newCache = new Map(state.cache);
      newCache.set(action.payload.key, {
        response: action.payload.response,
        timestamp: new Date(),
        ttl: action.payload.ttl
      });
      
      return {
        ...state,
        cache: newCache
      };

    case 'SET_ERROR':
      return {
        ...state,
        error: action.payload
      };

    case 'CLEAR_ERROR':
      return {
        ...state,
        error: null
      };

    default:
      return state;
  }
};

// AI Context
const AIContext = createContext<{
  state: AIState;
  dispatch: React.Dispatch<AIAction>;
  generateCompletion: (prompt: string, options?: CompletionOptions) => Promise<string>;
  generateEmbedding: (text: string) => Promise<number[]>;
  analyzeText: (text: string, analysisType: string) => Promise<any>;
  checkRateLimit: () => boolean;
} | null>(null);

// AI Provider Component
export const AIProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const initialState: AIState = {
    isInitialized: false,
    apiKey: null,
    models: [],
    activeModel: 'gpt-4',
    usage: {
      totalTokens: 0,
      totalRequests: 0,
      totalCost: 0,
      requestsToday: 0,
      tokensToday: 0,
      lastReset: new Date()
    },
    cache: new Map(),
    rateLimits: {
      requestsPerMinute: 60,
      tokensPerMinute: 60000,
      currentRequests: 0,
      currentTokens: 0,
      windowStart: new Date()
    },
    error: null
  };

  const [state, dispatch] = useReducer(aiReducer, initialState);

  const checkRateLimit = useCallback((): boolean => {
    const now = new Date();
    const windowDuration = 60 * 1000; // 1 minute
    
    if (now.getTime() - state.rateLimits.windowStart.getTime() > windowDuration) {
      // Reset window
      dispatch({
        type: 'UPDATE_RATE_LIMIT',
        payload: { requests: 0, tokens: 0 }
      });
      return true;
    }

    return (
      state.rateLimits.currentRequests < state.rateLimits.requestsPerMinute &&
      state.rateLimits.currentTokens < state.rateLimits.tokensPerMinute
    );
  }, [state.rateLimits]);

  const getCachedResponse = useCallback((key: string): any | null => {
    const cached = state.cache.get(key);
    if (!cached) return null;

    const now = new Date();
    if (now.getTime() - cached.timestamp.getTime() > cached.ttl) {
      // Cache expired
      const newCache = new Map(state.cache);
      newCache.delete(key);
      dispatch({
        type: 'CACHE_RESPONSE',
        payload: { key, response: null, ttl: 0 }
      });
      return null;
    }

    return cached.response;
  }, [state.cache]);

  const generateCompletion = useCallback(async (
    prompt: string,
    options: CompletionOptions = {}
  ): Promise<string> => {
    if (!checkRateLimit()) {
      throw new Error('Rate limit exceeded');
    }

    const cacheKey = `completion:${prompt}:${JSON.stringify(options)}`;
    const cached = getCachedResponse(cacheKey);
    
    if (cached && options.useCache !== false) {
      return cached;
    }

    try {
      // Simulate API call (replace with actual implementation)
      const response = await fetch('/api/ai/completion', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          prompt,
          model: state.activeModel,
          ...options
        })
      });

      if (!response.ok) {
        throw new Error(`API Error: ${response.statusText}`);
      }

      const data = await response.json();
      
      // Update usage stats
      dispatch({
        type: 'UPDATE_USAGE',
        payload: {
          tokens: data.usage?.total_tokens || 0,
          cost: calculateCost(data.usage?.total_tokens || 0, state.activeModel)
        }
      });

      // Cache response
      if (options.useCache !== false) {
        dispatch({
          type: 'CACHE_RESPONSE',
          payload: {
            key: cacheKey,
            response: data.content,
            ttl: 300000 // 5 minutes
          }
        });
      }

      return data.content;
    } catch (error) {
      dispatch({ type: 'SET_ERROR', payload: error.message });
      throw error;
    }
  }, [state, checkRateLimit, getCachedResponse]);

  const generateEmbedding = useCallback(async (text: string): Promise<number[]> => {
    const cacheKey = `embedding:${text}`;
    const cached = getCachedResponse(cacheKey);
    
    if (cached) {
      return cached;
    }

    try {
      const response = await fetch('/api/ai/embedding', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ text })
      });

      const data = await response.json();
      
      // Cache embedding
      dispatch({
        type: 'CACHE_RESPONSE',
        payload: {
          key: cacheKey,
          response: data.embedding,
          ttl: 3600000 // 1 hour
        }
      });

      return data.embedding;
    } catch (error) {
      dispatch({ type: 'SET_ERROR', payload: error.message });
      throw error;
    }
  }, [getCachedResponse]);

  const analyzeText = useCallback(async (text: string, analysisType: string): Promise<any> => {
    const prompt = generateAnalysisPrompt(text, analysisType);
    return generateCompletion(prompt, {
      systemPrompt: getAnalysisSystemPrompt(analysisType),
      temperature: 0.3,
      maxTokens: 500
    });
  }, [generateCompletion]);

  const value = {
    state,
    dispatch,
    generateCompletion,
    generateEmbedding,
    analyzeText,
    checkRateLimit
  };

  return <AIContext.Provider value={value}>{children}</AIContext.Provider>;
};

// Custom Hooks
export const useAI = () => {
  const context = useContext(AIContext);
  if (!context) {
    throw new Error('useAI must be used within an AIProvider');
  }
  return context;
};

export const useAICompletion = (
  prompt: string,
  options: CompletionOptions & { enabled?: boolean } = {}
) => {
  const { generateCompletion, state } = useAI();
  const [result, setResult] = useState<string>('');
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const execute = useCallback(async () => {
    if (!prompt.trim() || options.enabled === false) return;

    setIsLoading(true);
    setError(null);

    try {
      const completion = await generateCompletion(prompt, options);
      setResult(completion);
    } catch (err) {
      setError(err.message);
    } finally {
      setIsLoading(false);
    }
  }, [prompt, options, generateCompletion]);

  useEffect(() => {
    if (options.enabled !== false) {
      execute();
    }
  }, [execute, options.enabled]);

  return { result, isLoading, error, refetch: execute };
};

export const useAIAnalysis = (text: string, analysisTypes: string[]) => {
  const { analyzeText } = useAI();
  const [results, setResults] = useState<Record<string, any>>({});
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    if (!text.trim() || analysisTypes.length === 0) return;

    const analyze = async () => {
      setIsLoading(true);
      setError(null);

      try {
        const analysisPromises = analysisTypes.map(async (type) => {
          const result = await analyzeText(text, type);
          return { [type]: result };
        });

        const analysisResults = await Promise.all(analysisPromises);
        const combinedResults = analysisResults.reduce((acc, curr) => ({ ...acc, ...curr }), {});
        
        setResults(combinedResults);
      } catch (err) {
        setError(err.message);
      } finally {
        setIsLoading(false);
      }
    };

    analyze();
  }, [text, analysisTypes, analyzeText]);

  return { results, isLoading, error };
};

export const useAIEmbedding = (text: string, options: { enabled?: boolean } = {}) => {
  const { generateEmbedding } = useAI();
  const [embedding, setEmbedding] = useState<number[]>([]);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    if (!text.trim() || options.enabled === false) return;

    const generate = async () => {
      setIsLoading(true);
      setError(null);

      try {
        const result = await generateEmbedding(text);
        setEmbedding(result);
      } catch (err) {
        setError(err.message);
      } finally {
        setIsLoading(false);
      }
    };

    generate();
  }, [text, generateEmbedding, options.enabled]);

  return { embedding, isLoading, error };
};

// Utility Functions
function calculateCost(tokens: number, model: string): number {
  const pricing: Record<string, number> = {
    'gpt-4': 0.03,
    'gpt-3.5-turbo': 0.002,
    'text-davinci-003': 0.02,
    'text-embedding-ada-002': 0.0001
  };

  return (tokens / 1000) * (pricing[model] || 0.01);
}

function generateAnalysisPrompt(text: string, analysisType: string): string {
  const prompts: Record<string, string> = {
    sentiment: `Analyze the sentiment of this text: "${text}"`,
    tone: `Analyze the tone and style of this text: "${text}"`,
    grammar: `Check for grammar and style issues in this text: "${text}"`,
    readability: `Analyze the readability and suggest improvements for this text: "${text}"`,
    keywords: `Extract key themes and important keywords from this text: "${text}"`,
    summary: `Provide a concise summary of this text: "${text}"`
  };

  return prompts[analysisType] || `Analyze this text: "${text}"`;
}

function getAnalysisSystemPrompt(analysisType: string): string {
  const systemPrompts: Record<string, string> = {
    sentiment: 'You are a sentiment analysis expert. Provide detailed sentiment analysis.',
    tone: 'You are a writing expert. Analyze tone, style, and voice.',
    grammar: 'You are a grammar and style checker. Identify issues and suggest improvements.',
    readability: 'You are a readability expert. Analyze text complexity and suggest improvements.',
    keywords: 'You are a text analysis expert. Extract key themes and important keywords.',
    summary: 'You are a summarization expert. Provide concise, accurate summaries.'
  };

  return systemPrompts[analysisType] || 'You are a helpful text analysis assistant.';
}

export type { AIState, AIModel, UsageStats, CompletionOptions };
```

## Interview-Ready Summary

**LLM Integration Architecture:**

1. **Client Architecture** - Secure API client with rate limiting, caching, and error handling
2. **Context Management** - React Context for state management, usage tracking, and configuration
3. **Custom Hooks** - Reusable hooks for completions, analysis, embeddings, and streaming
4. **Component Patterns** - AI-powered chat, text editor, search with real-time features
5. **Performance Optimization** - Caching, debouncing, request batching, and intelligent prefetching

**Advanced Features:**
- **Rate Limiting** - Token bucket algorithm for API quota management
- **Caching Strategy** - LRU cache with TTL for response optimization
- **Streaming Support** - Real-time response rendering for better UX
- **Error Handling** - Graceful degradation and retry logic
- **Usage Analytics** - Token tracking, cost monitoring, performance metrics

**Security Considerations:**
- **API Key Management** - Backend proxy for secure key handling
- **Input Sanitization** - Prompt injection prevention and validation
- **Content Filtering** - Output moderation and safety checks
- **Privacy Protection** - Data encryption and user consent management

**Production Best Practices:**
- **Monitoring** - Request logging, error tracking, performance metrics
- **Scalability** - Request queuing, load balancing, circuit breakers
- **Cost Management** - Usage limits, budget alerts, model optimization
- **Compliance** - GDPR compliance, data retention policies, audit trails

**Key Interview Topics:** LLM integration patterns, prompt engineering, token optimization, streaming implementations, caching strategies, security considerations, cost management, production monitoring, AI UX patterns.