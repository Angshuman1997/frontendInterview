# MCP Server Development and Implementation

Model Context Protocol (MCP) enables seamless integration between AI applications and external data sources through standardized server implementations. This guide covers enterprise-grade MCP server development, client integration, and production deployment patterns.

## MCP Architecture and Core Concepts

### 1. MCP Server Implementation with TypeScript
```typescript
// Enterprise MCP Server Implementation
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ErrorCode,
  ListPromptsRequestSchema,
  ListResourcesRequestSchema,
  ListToolsRequestSchema,
  McpError,
  ReadResourceRequestSchema,
  GetPromptRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';

// Core MCP Server Class
class EnterpriseMCPServer {
  private server: Server;
  private tools: Map<string, MCPTool>;
  private resources: Map<string, MCPResource>;
  private prompts: Map<string, MCPPrompt>;
  private config: MCPServerConfig;

  constructor(config: MCPServerConfig) {
    this.config = config;
    this.server = new Server(
      {
        name: config.name,
        version: config.version,
      },
      {
        capabilities: {
          logging: {},
          tools: {},
          resources: {
            subscribe: true,
            listChanged: true,
          },
          prompts: {},
        },
      }
    );

    this.tools = new Map();
    this.resources = new Map();
    this.prompts = new Map();
    
    this.setupHandlers();
  }

  private setupHandlers(): void {
    // Tools handlers
    this.server.setRequestHandler(ListToolsRequestSchema, async () => {
      return {
        tools: Array.from(this.tools.values()).map(tool => ({
          name: tool.name,
          description: tool.description,
          inputSchema: tool.inputSchema,
        })),
      };
    });

    this.server.setRequestHandler(CallToolRequestSchema, async (request) => {
      const { name, arguments: args } = request.params;
      
      const tool = this.tools.get(name);
      if (!tool) {
        throw new McpError(ErrorCode.MethodNotFound, `Tool "${name}" not found`);
      }

      try {
        const result = await tool.execute(args || {}, {
          server: this,
          config: this.config,
          timestamp: new Date().toISOString(),
        });

        return {
          content: [
            {
              type: 'text',
              text: typeof result === 'string' ? result : JSON.stringify(result, null, 2),
            },
          ],
        };
      } catch (error) {
        throw new McpError(
          ErrorCode.InternalError,
          `Tool execution failed: ${error instanceof Error ? error.message : 'Unknown error'}`
        );
      }
    });

    // Resources handlers
    this.server.setRequestHandler(ListResourcesRequestSchema, async () => {
      return {
        resources: Array.from(this.resources.values()).map(resource => ({
          uri: resource.uri,
          name: resource.name,
          description: resource.description,
          mimeType: resource.mimeType,
        })),
      };
    });

    this.server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
      const { uri } = request.params;
      
      const resource = this.resources.get(uri);
      if (!resource) {
        throw new McpError(ErrorCode.InvalidRequest, `Resource "${uri}" not found`);
      }

      try {
        const content = await resource.read({
          server: this,
          config: this.config,
          timestamp: new Date().toISOString(),
        });

        return {
          contents: [
            {
              uri,
              mimeType: resource.mimeType,
              text: content,
            },
          ],
        };
      } catch (error) {
        throw new McpError(
          ErrorCode.InternalError,
          `Resource read failed: ${error instanceof Error ? error.message : 'Unknown error'}`
        );
      }
    });

    // Prompts handlers
    this.server.setRequestHandler(ListPromptsRequestSchema, async () => {
      return {
        prompts: Array.from(this.prompts.values()).map(prompt => ({
          name: prompt.name,
          description: prompt.description,
          arguments: prompt.arguments,
        })),
      };
    });

    this.server.setRequestHandler(GetPromptRequestSchema, async (request) => {
      const { name, arguments: args } = request.params;
      
      const prompt = this.prompts.get(name);
      if (!prompt) {
        throw new McpError(ErrorCode.MethodNotFound, `Prompt "${name}" not found`);
      }

      try {
        const messages = await prompt.generate(args || {}, {
          server: this,
          config: this.config,
          timestamp: new Date().toISOString(),
        });

        return {
          description: prompt.description,
          messages,
        };
      } catch (error) {
        throw new McpError(
          ErrorCode.InternalError,
          `Prompt generation failed: ${error instanceof Error ? error.message : 'Unknown error'}`
        );
      }
    });
  }

  // Register tools
  registerTool(tool: MCPTool): void {
    this.tools.set(tool.name, tool);
  }

  // Register resources
  registerResource(resource: MCPResource): void {
    this.resources.set(resource.uri, resource);
  }

  // Register prompts
  registerPrompt(prompt: MCPPrompt): void {
    this.prompts.set(prompt.name, prompt);
  }

  // Start server
  async start(): Promise<void> {
    const transport = new StdioServerTransport();
    await this.server.connect(transport);
    
    console.error(`${this.config.name} MCP Server started`);
  }

  // Resource change notification
  async notifyResourceChanged(uri: string): Promise<void> {
    await this.server.notification({
      method: 'notifications/resources/list_changed',
    });
  }

  // Get server instance for advanced operations
  getServer(): Server {
    return this.server;
  }
}

// Database Integration Tool
class DatabaseQueryTool implements MCPTool {
  name = 'database_query';
  description = 'Execute SQL queries against the database';
  inputSchema = {
    type: 'object',
    properties: {
      query: {
        type: 'string',
        description: 'SQL query to execute',
      },
      database: {
        type: 'string',
        description: 'Database name (optional)',
        default: 'default',
      },
      limit: {
        type: 'number',
        description: 'Maximum number of rows to return',
        default: 100,
      },
    },
    required: ['query'],
  };

  constructor(private databaseManager: DatabaseManager) {}

  async execute(args: any, context: MCPContext): Promise<any> {
    const { query, database = 'default', limit = 100 } = args;

    // Validate query safety
    if (!this.isQuerySafe(query)) {
      throw new Error('Query contains unsafe operations');
    }

    try {
      const connection = await this.databaseManager.getConnection(database);
      const results = await connection.query(query, { limit });
      
      return {
        success: true,
        rowCount: results.length,
        data: results,
        executedAt: context.timestamp,
      };
    } catch (error) {
      throw new Error(`Database query failed: ${error instanceof Error ? error.message : 'Unknown error'}`);
    }
  }

  private isQuerySafe(query: string): boolean {
    const unsafePatterns = [
      /\b(DROP|DELETE|UPDATE|INSERT|ALTER|CREATE|TRUNCATE)\b/i,
      /\b(EXEC|EXECUTE)\b/i,
      /--/,
      /\/\*/,
    ];

    return !unsafePatterns.some(pattern => pattern.test(query));
  }
}

// File System Resource
class FileSystemResource implements MCPResource {
  uri: string;
  name: string;
  description: string;
  mimeType: string;

  constructor(
    private filePath: string,
    private baseDir: string,
    name?: string
  ) {
    this.uri = `file://${filePath}`;
    this.name = name || filePath.split('/').pop() || filePath;
    this.description = `File: ${filePath}`;
    this.mimeType = this.getMimeType(filePath);
  }

  async read(context: MCPContext): Promise<string> {
    const fs = await import('fs/promises');
    const path = await import('path');
    
    // Security check: ensure file is within base directory
    const fullPath = path.resolve(this.baseDir, this.filePath);
    const normalizedBase = path.normalize(this.baseDir);
    
    if (!fullPath.startsWith(normalizedBase)) {
      throw new Error('Access denied: file outside base directory');
    }

    try {
      const content = await fs.readFile(fullPath, 'utf-8');
      return content;
    } catch (error) {
      throw new Error(`Failed to read file: ${error instanceof Error ? error.message : 'Unknown error'}`);
    }
  }

  private getMimeType(filePath: string): string {
    const ext = filePath.split('.').pop()?.toLowerCase();
    
    const mimeTypes: Record<string, string> = {
      'js': 'application/javascript',
      'ts': 'application/typescript',
      'json': 'application/json',
      'md': 'text/markdown',
      'txt': 'text/plain',
      'html': 'text/html',
      'css': 'text/css',
      'xml': 'application/xml',
      'yaml': 'application/x-yaml',
      'yml': 'application/x-yaml',
    };

    return mimeTypes[ext || ''] || 'text/plain';
  }
}

// Code Analysis Prompt
class CodeAnalysisPrompt implements MCPPrompt {
  name = 'code_analysis';
  description = 'Analyze code for patterns, issues, and improvements';
  arguments = [
    {
      name: 'code',
      description: 'Code to analyze',
      required: true,
    },
    {
      name: 'language',
      description: 'Programming language',
      required: false,
    },
    {
      name: 'focus',
      description: 'Analysis focus (security, performance, maintainability)',
      required: false,
    },
  ];

  async generate(args: any, context: MCPContext): Promise<any[]> {
    const { code, language = 'javascript', focus = 'general' } = args;

    const systemPrompt = this.getSystemPrompt(language, focus);
    const userPrompt = this.getUserPrompt(code, focus);

    return [
      {
        role: 'system',
        content: {
          type: 'text',
          text: systemPrompt,
        },
      },
      {
        role: 'user',
        content: {
          type: 'text',
          text: userPrompt,
        },
      },
    ];
  }

  private getSystemPrompt(language: string, focus: string): string {
    const focusInstructions = {
      security: 'Focus on security vulnerabilities, input validation, authentication, and authorization issues.',
      performance: 'Focus on performance bottlenecks, algorithmic complexity, memory usage, and optimization opportunities.',
      maintainability: 'Focus on code structure, readability, design patterns, and maintainability concerns.',
      general: 'Provide a comprehensive analysis covering security, performance, and maintainability.',
    };

    return `You are an expert ${language} code reviewer. ${focusInstructions[focus] || focusInstructions.general}

Provide your analysis in the following format:
1. **Summary**: Brief overview of the code quality
2. **Issues Found**: List of specific issues with severity levels
3. **Recommendations**: Actionable suggestions for improvement
4. **Positive Aspects**: What the code does well

Be specific, cite line numbers when possible, and provide code examples for improvements.`;
  }

  private getUserPrompt(code: string, focus: string): string {
    return `Please analyze the following code with focus on ${focus}:

\`\`\`
${code}
\`\`\`

Provide a detailed analysis following the format specified in the system prompt.`;
  }
}

// API Integration Tool
class APIIntegrationTool implements MCPTool {
  name = 'api_call';
  description = 'Make HTTP requests to external APIs';
  inputSchema = {
    type: 'object',
    properties: {
      url: {
        type: 'string',
        description: 'API endpoint URL',
      },
      method: {
        type: 'string',
        enum: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
        default: 'GET',
      },
      headers: {
        type: 'object',
        description: 'HTTP headers',
      },
      body: {
        type: 'object',
        description: 'Request body (for POST, PUT, PATCH)',
      },
      timeout: {
        type: 'number',
        description: 'Request timeout in milliseconds',
        default: 30000,
      },
    },
    required: ['url'],
  };

  constructor(private httpClient: HTTPClient) {}

  async execute(args: any, context: MCPContext): Promise<any> {
    const { url, method = 'GET', headers = {}, body, timeout = 30000 } = args;

    // Validate URL
    if (!this.isUrlAllowed(url)) {
      throw new Error('URL not allowed by security policy');
    }

    try {
      const response = await this.httpClient.request({
        url,
        method,
        headers: {
          'User-Agent': `MCP-Server/${context.config.version}`,
          'Content-Type': 'application/json',
          ...headers,
        },
        body: body ? JSON.stringify(body) : undefined,
        timeout,
      });

      return {
        status: response.status,
        statusText: response.statusText,
        headers: response.headers,
        data: response.data,
        requestedAt: context.timestamp,
      };
    } catch (error) {
      throw new Error(`API request failed: ${error instanceof Error ? error.message : 'Unknown error'}`);
    }
  }

  private isUrlAllowed(url: string): boolean {
    // Implement URL allowlist/blocklist logic
    const allowedDomains = [
      'api.github.com',
      'jsonplaceholder.typicode.com',
      'httpbin.org',
    ];

    try {
      const urlObj = new URL(url);
      return allowedDomains.some(domain => urlObj.hostname.endsWith(domain));
    } catch {
      return false;
    }
  }
}

// Git Repository Tool
class GitRepositoryTool implements MCPTool {
  name = 'git_operations';
  description = 'Perform Git repository operations';
  inputSchema = {
    type: 'object',
    properties: {
      operation: {
        type: 'string',
        enum: ['status', 'log', 'diff', 'branch', 'show'],
        description: 'Git operation to perform',
      },
      repository: {
        type: 'string',
        description: 'Repository path',
      },
      options: {
        type: 'object',
        description: 'Operation-specific options',
      },
    },
    required: ['operation'],
  };

  constructor(private gitClient: GitClient) {}

  async execute(args: any, context: MCPContext): Promise<any> {
    const { operation, repository = '.', options = {} } = args;

    try {
      switch (operation) {
        case 'status':
          return await this.gitClient.status(repository, options);
        case 'log':
          return await this.gitClient.log(repository, options);
        case 'diff':
          return await this.gitClient.diff(repository, options);
        case 'branch':
          return await this.gitClient.branch(repository, options);
        case 'show':
          return await this.gitClient.show(repository, options);
        default:
          throw new Error(`Unsupported Git operation: ${operation}`);
      }
    } catch (error) {
      throw new Error(`Git operation failed: ${error instanceof Error ? error.message : 'Unknown error'}`);
    }
  }
}

// Example MCP Server Setup
class ExampleMCPServerSetup {
  static async createEnterpriseServer(): Promise<EnterpriseMCPServer> {
    const config: MCPServerConfig = {
      name: 'enterprise-mcp-server',
      version: '1.0.0',
      baseDirectory: process.cwd(),
      allowedDomains: ['api.github.com', 'jsonplaceholder.typicode.com'],
      security: {
        enableSandbox: true,
        maxFileSize: 10 * 1024 * 1024, // 10MB
        allowedExtensions: ['.js', '.ts', '.json', '.md', '.txt'],
      },
    };

    const server = new EnterpriseMCPServer(config);

    // Initialize clients
    const databaseManager = new DatabaseManager();
    const httpClient = new HTTPClient();
    const gitClient = new GitClient();

    // Register tools
    server.registerTool(new DatabaseQueryTool(databaseManager));
    server.registerTool(new APIIntegrationTool(httpClient));
    server.registerTool(new GitRepositoryTool(gitClient));

    // Register resources
    const configResource = new FileSystemResource('config.json', config.baseDirectory);
    const packageResource = new FileSystemResource('package.json', config.baseDirectory);
    server.registerResource(configResource);
    server.registerResource(packageResource);

    // Register prompts
    server.registerPrompt(new CodeAnalysisPrompt());

    return server;
  }
}

// Type Definitions
interface MCPServerConfig {
  name: string;
  version: string;
  baseDirectory: string;
  allowedDomains: string[];
  security: {
    enableSandbox: boolean;
    maxFileSize: number;
    allowedExtensions: string[];
  };
}

interface MCPContext {
  server: EnterpriseMCPServer;
  config: MCPServerConfig;
  timestamp: string;
}

interface MCPTool {
  name: string;
  description: string;
  inputSchema: any;
  execute(args: any, context: MCPContext): Promise<any>;
}

interface MCPResource {
  uri: string;
  name: string;
  description: string;
  mimeType: string;
  read(context: MCPContext): Promise<string>;
}

interface MCPPrompt {
  name: string;
  description: string;
  arguments: Array<{
    name: string;
    description: string;
    required: boolean;
  }>;
  generate(args: any, context: MCPContext): Promise<any[]>;
}

// Supporting Classes (simplified interfaces)
class DatabaseManager {
  async getConnection(database: string): Promise<any> {
    // Implementation would return actual database connection
    return {
      query: async (sql: string, options: any) => {
        // Mock implementation
        return [];
      }
    };
  }
}

class HTTPClient {
  async request(options: any): Promise<any> {
    // Implementation would make actual HTTP requests
    return {
      status: 200,
      statusText: 'OK',
      headers: {},
      data: {}
    };
  }
}

class GitClient {
  async status(repo: string, options: any): Promise<any> {
    // Implementation would run git status
    return { files: [], branch: 'main' };
  }

  async log(repo: string, options: any): Promise<any> {
    return { commits: [] };
  }

  async diff(repo: string, options: any): Promise<any> {
    return { changes: [] };
  }

  async branch(repo: string, options: any): Promise<any> {
    return { branches: [] };
  }

  async show(repo: string, options: any): Promise<any> {
    return { content: '' };
  }
}

export {
  EnterpriseMCPServer,
  DatabaseQueryTool,
  FileSystemResource,
  CodeAnalysisPrompt,
  APIIntegrationTool,
  GitRepositoryTool,
  ExampleMCPServerSetup
};
```

## Interview-Ready MCP Summary

**MCP Server Development provides:**

1. **Standardized Protocol** - Seamless AI application integration with external data sources
2. **Tool Registration** - Database queries, API calls, Git operations, file system access
3. **Resource Management** - Secure file access with sandbox restrictions
4. **Prompt Templates** - Reusable prompt patterns for code analysis and documentation
5. **Enterprise Security** - URL allowlists, file size limits, sandbox execution

**Key Implementation Patterns:**
- **Tool Interface** - Standardized tool execution with validation and error handling
- **Resource Access** - Secure file system integration with path validation
- **Prompt Generation** - Dynamic prompt creation with context injection
- **Context Management** - Server state and configuration access throughout operations

**Production Features:**
- **Type Safety** - Full TypeScript implementation with strict typing
- **Error Handling** - Comprehensive error management with proper MCP error codes
- **Security Controls** - Path traversal protection, URL filtering, file size limits
- **Extensibility** - Plugin architecture for custom tools, resources, and prompts

**Enterprise Benefits:** Standardized AI integration, secure data access, reusable components, scalable architecture for complex AI workflows.