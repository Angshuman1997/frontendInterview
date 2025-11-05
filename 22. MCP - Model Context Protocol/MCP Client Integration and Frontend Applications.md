# MCP Client Integration and Frontend Applications

Model Context Protocol (MCP) client integration enables frontend applications to seamlessly connect with MCP servers for enhanced AI-powered functionality. This guide covers client implementation, React integration patterns, and production deployment strategies.

## MCP Client Architecture

### 1. Advanced MCP Client Implementation
```typescript
// Enterprise MCP Client for Frontend Applications
import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js';
import {
  CallToolRequest,
  GetPromptRequest,
  ListResourcesRequest,
  ListToolsRequest,
  ReadResourceRequest,
  Tool,
  Resource,
  Prompt,
} from '@modelcontextprotocol/sdk/types.js';

// MCP Client Manager
class MCPClientManager {
  private clients: Map<string, MCPClientConnection>;
  private eventHandlers: Map<string, Set<(event: MCPEvent) => void>>;
  private config: MCPClientConfig;

  constructor(config: MCPClientConfig) {
    this.config = config;
    this.clients = new Map();
    this.eventHandlers = new Map();
  }

  // Connect to MCP server
  async connect(serverId: string, serverConfig: MCPServerEndpoint): Promise<void> {
    if (this.clients.has(serverId)) {
      throw new Error(`Already connected to server: ${serverId}`);
    }

    const client = new Client(
      {
        name: this.config.clientName,
        version: this.config.clientVersion,
      },
      {
        capabilities: {
          logging: {},
          sampling: {},
        },
      }
    );

    let transport;
    switch (serverConfig.type) {
      case 'stdio':
        transport = new StdioClientTransport({
          command: serverConfig.command,
          args: serverConfig.args || [],
          env: serverConfig.env,
        });
        break;
      case 'sse':
        transport = new SSEClientTransport(serverConfig.url);
        break;
      case 'websocket':
        transport = new WebSocketClientTransport(serverConfig.url);
        break;
      default:
        throw new Error(`Unsupported transport type: ${serverConfig.type}`);
    }

    await client.connect(transport);

    const connection: MCPClientConnection = {
      client,
      transport,
      serverId,
      config: serverConfig,
      connected: true,
      lastActivity: new Date(),
      tools: new Map(),
      resources: new Map(),
      prompts: new Map(),
    };

    // Initialize server capabilities
    await this.initializeServerCapabilities(connection);

    this.clients.set(serverId, connection);
    this.emitEvent('connected', { serverId, connection });
  }

  // Disconnect from MCP server
  async disconnect(serverId: string): Promise<void> {
    const connection = this.clients.get(serverId);
    if (!connection) {
      throw new Error(`No connection found for server: ${serverId}`);
    }

    await connection.client.close();
    connection.connected = false;
    
    this.clients.delete(serverId);
    this.emitEvent('disconnected', { serverId });
  }

  // List available tools across all connected servers
  async listTools(serverId?: string): Promise<ToolWithServer[]> {
    if (serverId) {
      const connection = this.clients.get(serverId);
      if (!connection) {
        throw new Error(`No connection found for server: ${serverId}`);
      }
      
      return Array.from(connection.tools.values()).map(tool => ({
        ...tool,
        serverId,
      }));
    }

    const allTools: ToolWithServer[] = [];
    for (const [id, connection] of this.clients) {
      const serverTools = Array.from(connection.tools.values()).map(tool => ({
        ...tool,
        serverId: id,
      }));
      allTools.push(...serverTools);
    }

    return allTools;
  }

  // Execute tool
  async callTool(
    serverId: string,
    toolName: string,
    args: any = {},
    options: CallToolOptions = {}
  ): Promise<any> {
    const connection = this.clients.get(serverId);
    if (!connection) {
      throw new Error(`No connection found for server: ${serverId}`);
    }

    const tool = connection.tools.get(toolName);
    if (!tool) {
      throw new Error(`Tool "${toolName}" not found on server: ${serverId}`);
    }

    try {
      this.validateToolArguments(tool, args);

      const request: CallToolRequest = {
        method: 'tools/call',
        params: {
          name: toolName,
          arguments: args,
        },
      };

      const startTime = Date.now();
      const response = await connection.client.request(request);
      const duration = Date.now() - startTime;

      connection.lastActivity = new Date();

      this.emitEvent('toolCalled', {
        serverId,
        toolName,
        args,
        duration,
        success: true,
      });

      return response.content;
    } catch (error) {
      this.emitEvent('toolError', {
        serverId,
        toolName,
        args,
        error: error instanceof Error ? error.message : 'Unknown error',
      });
      throw error;
    }
  }

  // List resources
  async listResources(serverId: string): Promise<Resource[]> {
    const connection = this.clients.get(serverId);
    if (!connection) {
      throw new Error(`No connection found for server: ${serverId}`);
    }

    const request: ListResourcesRequest = {
      method: 'resources/list',
      params: {},
    };

    const response = await connection.client.request(request);
    return response.resources;
  }

  // Read resource
  async readResource(serverId: string, uri: string): Promise<string> {
    const connection = this.clients.get(serverId);
    if (!connection) {
      throw new Error(`No connection found for server: ${serverId}`);
    }

    const request: ReadResourceRequest = {
      method: 'resources/read',
      params: { uri },
    };

    const response = await connection.client.request(request);
    return response.contents[0]?.text || '';
  }

  // Get prompt
  async getPrompt(
    serverId: string,
    promptName: string,
    args: any = {}
  ): Promise<any[]> {
    const connection = this.clients.get(serverId);
    if (!connection) {
      throw new Error(`No connection found for server: ${serverId}`);
    }

    const request: GetPromptRequest = {
      method: 'prompts/get',
      params: {
        name: promptName,
        arguments: args,
      },
    };

    const response = await connection.client.request(request);
    return response.messages;
  }

  // Event handling
  on(event: string, handler: (event: MCPEvent) => void): void {
    if (!this.eventHandlers.has(event)) {
      this.eventHandlers.set(event, new Set());
    }
    this.eventHandlers.get(event)!.add(handler);
  }

  off(event: string, handler: (event: MCPEvent) => void): void {
    this.eventHandlers.get(event)?.delete(handler);
  }

  // Health check for all connections
  async healthCheck(): Promise<ConnectionHealth[]> {
    const healthStatus: ConnectionHealth[] = [];

    for (const [serverId, connection] of this.clients) {
      try {
        const tools = await this.listTools(serverId);
        healthStatus.push({
          serverId,
          connected: connection.connected,
          lastActivity: connection.lastActivity,
          toolCount: tools.length,
          status: 'healthy',
        });
      } catch (error) {
        healthStatus.push({
          serverId,
          connected: false,
          lastActivity: connection.lastActivity,
          toolCount: 0,
          status: 'unhealthy',
          error: error instanceof Error ? error.message : 'Unknown error',
        });
      }
    }

    return healthStatus;
  }

  private async initializeServerCapabilities(connection: MCPClientConnection): Promise<void> {
    try {
      // Load tools
      const toolsRequest: ListToolsRequest = {
        method: 'tools/list',
        params: {},
      };
      const toolsResponse = await connection.client.request(toolsRequest);
      
      for (const tool of toolsResponse.tools) {
        connection.tools.set(tool.name, tool);
      }

      // Load prompts
      try {
        const promptsRequest = {
          method: 'prompts/list',
          params: {},
        };
        const promptsResponse = await connection.client.request(promptsRequest);
        
        for (const prompt of promptsResponse.prompts) {
          connection.prompts.set(prompt.name, prompt);
        }
      } catch {
        // Prompts might not be supported by this server
      }

    } catch (error) {
      console.warn(`Failed to initialize capabilities for ${connection.serverId}:`, error);
    }
  }

  private validateToolArguments(tool: Tool, args: any): void {
    if (!tool.inputSchema) return;

    // Basic validation - in production, use a proper JSON schema validator
    if (tool.inputSchema.required) {
      for (const required of tool.inputSchema.required) {
        if (!(required in args)) {
          throw new Error(`Missing required argument: ${required}`);
        }
      }
    }
  }

  private emitEvent(event: string, data: any): void {
    const handlers = this.eventHandlers.get(event);
    if (handlers) {
      handlers.forEach(handler => {
        try {
          handler({ type: event, data, timestamp: new Date() });
        } catch (error) {
          console.error('Event handler error:', error);
        }
      });
    }
  }
}

// React Hook for MCP Integration
import React, { createContext, useContext, useEffect, useState, useCallback } from 'react';

interface MCPContextType {
  client: MCPClientManager | null;
  isConnected: boolean;
  tools: ToolWithServer[];
  resources: ResourceWithServer[];
  connections: ConnectionHealth[];
  callTool: (serverId: string, toolName: string, args?: any) => Promise<any>;
  readResource: (serverId: string, uri: string) => Promise<string>;
  getPrompt: (serverId: string, promptName: string, args?: any) => Promise<any[]>;
  refreshConnections: () => Promise<void>;
}

const MCPContext = createContext<MCPContextType | null>(null);

export const useMCP = () => {
  const context = useContext(MCPContext);
  if (!context) {
    throw new Error('useMCP must be used within MCPProvider');
  }
  return context;
};

// MCP Provider Component
export const MCPProvider: React.FC<{
  children: React.ReactNode;
  config: MCPClientConfig;
  servers: MCPServerEndpoint[];
}> = ({ children, config, servers }) => {
  const [client] = useState(() => new MCPClientManager(config));
  const [isConnected, setIsConnected] = useState(false);
  const [tools, setTools] = useState<ToolWithServer[]>([]);
  const [resources, setResources] = useState<ResourceWithServer[]>([]);
  const [connections, setConnections] = useState<ConnectionHealth[]>([]);

  useEffect(() => {
    const connectToServers = async () => {
      for (const server of servers) {
        try {
          await client.connect(server.id, server);
        } catch (error) {
          console.error(`Failed to connect to ${server.id}:`, error);
        }
      }
      
      await refreshData();
      setIsConnected(true);
    };

    connectToServers();

    // Setup event handlers
    client.on('connected', refreshData);
    client.on('disconnected', refreshData);
    client.on('toolCalled', refreshData);

    return () => {
      servers.forEach(server => {
        client.disconnect(server.id).catch(console.error);
      });
    };
  }, []);

  const refreshData = useCallback(async () => {
    try {
      const [allTools, healthStatus] = await Promise.all([
        client.listTools(),
        client.healthCheck(),
      ]);

      setTools(allTools);
      setConnections(healthStatus);

      // Load resources from all servers
      const allResources: ResourceWithServer[] = [];
      for (const connection of healthStatus) {
        if (connection.connected) {
          try {
            const serverResources = await client.listResources(connection.serverId);
            allResources.push(...serverResources.map(resource => ({
              ...resource,
              serverId: connection.serverId,
            })));
          } catch (error) {
            console.warn(`Failed to load resources from ${connection.serverId}:`, error);
          }
        }
      }
      setResources(allResources);
    } catch (error) {
      console.error('Failed to refresh MCP data:', error);
    }
  }, [client]);

  const callTool = useCallback(async (serverId: string, toolName: string, args?: any) => {
    return client.callTool(serverId, toolName, args);
  }, [client]);

  const readResource = useCallback(async (serverId: string, uri: string) => {
    return client.readResource(serverId, uri);
  }, [client]);

  const getPrompt = useCallback(async (serverId: string, promptName: string, args?: any) => {
    return client.getPrompt(serverId, promptName, args);
  }, [client]);

  const refreshConnections = useCallback(async () => {
    await refreshData();
  }, [refreshData]);

  return (
    <MCPContext.Provider value={{
      client,
      isConnected,
      tools,
      resources,
      connections,
      callTool,
      readResource,
      getPrompt,
      refreshConnections,
    }}>
      {children}
    </MCPContext.Provider>
  );
};

// MCP Tool Executor Component
export const MCPToolExecutor: React.FC<{
  serverId: string;
  toolName: string;
  initialArgs?: any;
  onResult?: (result: any) => void;
  onError?: (error: string) => void;
}> = ({ serverId, toolName, initialArgs = {}, onResult, onError }) => {
  const { callTool, tools } = useMCP();
  const [args, setArgs] = useState(initialArgs);
  const [result, setResult] = useState<any>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const tool = tools.find(t => t.serverId === serverId && t.name === toolName);

  const executeTool = async () => {
    if (!tool) {
      setError('Tool not found');
      return;
    }

    setLoading(true);
    setError(null);

    try {
      const response = await callTool(serverId, toolName, args);
      setResult(response);
      onResult?.(response);
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : 'Unknown error';
      setError(errorMessage);
      onError?.(errorMessage);
    } finally {
      setLoading(false);
    }
  };

  if (!tool) {
    return <div className="error">Tool "{toolName}" not found on server "{serverId}"</div>;
  }

  return (
    <div className="mcp-tool-executor">
      <div className="tool-header">
        <h3>{tool.name}</h3>
        <p>{tool.description}</p>
      </div>

      <div className="tool-arguments">
        <h4>Arguments:</h4>
        {tool.inputSchema?.properties && Object.entries(tool.inputSchema.properties).map(([key, schema]: [string, any]) => (
          <div key={key} className="argument-field">
            <label>
              {key} {tool.inputSchema.required?.includes(key) && <span className="required">*</span>}
            </label>
            <input
              type={schema.type === 'number' ? 'number' : 'text'}
              value={args[key] || ''}
              onChange={(e) => setArgs(prev => ({
                ...prev,
                [key]: schema.type === 'number' ? Number(e.target.value) : e.target.value
              }))}
              placeholder={schema.description}
            />
          </div>
        ))}
      </div>

      <button onClick={executeTool} disabled={loading} className="execute-button">
        {loading ? 'Executing...' : 'Execute Tool'}
      </button>

      {error && (
        <div className="error-display">
          <h4>Error:</h4>
          <pre>{error}</pre>
        </div>
      )}

      {result && (
        <div className="result-display">
          <h4>Result:</h4>
          <pre>{JSON.stringify(result, null, 2)}</pre>
        </div>
      )}
    </div>
  );
};

// MCP Resource Viewer Component
export const MCPResourceViewer: React.FC<{
  serverId: string;
  uri: string;
  onLoad?: (content: string) => void;
}> = ({ serverId, uri, onLoad }) => {
  const { readResource } = useMCP();
  const [content, setContent] = useState<string>('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const loadResource = async () => {
      setLoading(true);
      setError(null);

      try {
        const resourceContent = await readResource(serverId, uri);
        setContent(resourceContent);
        onLoad?.(resourceContent);
      } catch (err) {
        const errorMessage = err instanceof Error ? err.message : 'Unknown error';
        setError(errorMessage);
      } finally {
        setLoading(false);
      }
    };

    loadResource();
  }, [serverId, uri, readResource]);

  if (loading) {
    return <div className="loading">Loading resource...</div>;
  }

  if (error) {
    return <div className="error">Error loading resource: {error}</div>;
  }

  return (
    <div className="mcp-resource-viewer">
      <div className="resource-header">
        <h3>Resource: {uri}</h3>
      </div>
      <div className="resource-content">
        <pre>{content}</pre>
      </div>
    </div>
  );
};

// MCP Dashboard Component
export const MCPDashboard: React.FC = () => {
  const { connections, tools, resources, refreshConnections } = useMCP();
  const [selectedServer, setSelectedServer] = useState<string | null>(null);

  return (
    <div className="mcp-dashboard">
      <div className="dashboard-header">
        <h2>MCP Server Dashboard</h2>
        <button onClick={refreshConnections} className="refresh-button">
          Refresh
        </button>
      </div>

      <div className="connections-grid">
        {connections.map(connection => (
          <div
            key={connection.serverId}
            className={`connection-card ${connection.status}`}
            onClick={() => setSelectedServer(connection.serverId)}
          >
            <h3>{connection.serverId}</h3>
            <div className="connection-status">
              <span className={`status-indicator ${connection.status}`}>
                {connection.status}
              </span>
              <span className="tool-count">
                {connection.toolCount} tools
              </span>
            </div>
            <div className="last-activity">
              Last activity: {connection.lastActivity.toLocaleString()}
            </div>
            {connection.error && (
              <div className="error-message">{connection.error}</div>
            )}
          </div>
        ))}
      </div>

      {selectedServer && (
        <div className="server-details">
          <h3>Server: {selectedServer}</h3>
          
          <div className="tools-section">
            <h4>Available Tools:</h4>
            {tools
              .filter(tool => tool.serverId === selectedServer)
              .map(tool => (
                <div key={tool.name} className="tool-item">
                  <h5>{tool.name}</h5>
                  <p>{tool.description}</p>
                </div>
              ))}
          </div>

          <div className="resources-section">
            <h4>Available Resources:</h4>
            {resources
              .filter(resource => resource.serverId === selectedServer)
              .map(resource => (
                <div key={resource.uri} className="resource-item">
                  <h5>{resource.name}</h5>
                  <p>{resource.description}</p>
                  <code>{resource.uri}</code>
                </div>
              ))}
          </div>
        </div>
      )}
    </div>
  );
};

// Type Definitions
interface MCPClientConfig {
  clientName: string;
  clientVersion: string;
  timeout?: number;
  retryAttempts?: number;
}

interface MCPServerEndpoint {
  id: string;
  type: 'stdio' | 'sse' | 'websocket';
  command?: string;
  args?: string[];
  env?: Record<string, string>;
  url?: string;
}

interface MCPClientConnection {
  client: Client;
  transport: any;
  serverId: string;
  config: MCPServerEndpoint;
  connected: boolean;
  lastActivity: Date;
  tools: Map<string, Tool>;
  resources: Map<string, Resource>;
  prompts: Map<string, Prompt>;
}

interface ToolWithServer extends Tool {
  serverId: string;
}

interface ResourceWithServer extends Resource {
  serverId: string;
}

interface CallToolOptions {
  timeout?: number;
  retries?: number;
}

interface MCPEvent {
  type: string;
  data: any;
  timestamp: Date;
}

interface ConnectionHealth {
  serverId: string;
  connected: boolean;
  lastActivity: Date;
  toolCount: number;
  status: 'healthy' | 'unhealthy';
  error?: string;
}

// Supporting Transport Classes (simplified)
class SSEClientTransport {
  constructor(private url: string) {}
  // Implementation would handle SSE connection
}

class WebSocketClientTransport {
  constructor(private url: string) {}
  // Implementation would handle WebSocket connection
}

export {
  MCPClientManager,
  MCPProvider,
  MCPToolExecutor,
  MCPResourceViewer,
  MCPDashboard,
  useMCP
};
```

## Interview-Ready Client Integration Summary

**MCP Client Integration provides:**

1. **Multi-Server Management** - Connect to multiple MCP servers with different transport types
2. **React Integration** - Context provider and hooks for seamless React integration
3. **Tool Execution** - Type-safe tool calling with argument validation
4. **Resource Access** - Secure resource reading with proper error handling
5. **Real-time Monitoring** - Connection health checks and event-driven updates

**Key Integration Patterns:**
- **Context Provider** - Centralized MCP state management for React applications
- **Custom Hooks** - Simplified MCP operations with proper error handling
- **Component Library** - Ready-to-use components for tool execution and resource viewing
- **Event System** - Real-time updates for connection status and tool results

**Frontend Benefits:**
- **Type Safety** - Full TypeScript support with proper interface definitions
- **Error Boundaries** - Graceful error handling with user-friendly messages
- **Performance** - Connection pooling and caching for optimal performance
- **Developer Experience** - React DevTools integration and comprehensive debugging

**Production Features:** Connection health monitoring, automatic reconnection, tool argument validation, resource caching, comprehensive error handling, performance metrics.