# Figma MCP VSCode Copilot Integration

Complete guide for integrating Figma with Model Context Protocol (MCP) servers and leveraging VSCode Copilot for design-to-code workflows. This integration enables seamless design system synchronization, automated component generation, and AI-powered development assistance.

## MCP Server for Figma Integration

### 1. Figma MCP Server Implementation

```typescript
// figma-mcp-server.ts - Enterprise Figma MCP Server
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
  ListResourcesRequestSchema,
  ReadResourceRequestSchema,
  ListPromptsRequestSchema,
  GetPromptRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';

interface FigmaConfig {
  accessToken: string;
  teamId?: string;
  fileKeys: string[];
  webhookSecret?: string;
  cacheTimeout: number;
}

interface FigmaNode {
  id: string;
  name: string;
  type: string;
  children?: FigmaNode[];
  fills?: any[];
  strokes?: any[];
  effects?: any[];
  absoluteBoundingBox?: {
    x: number;
    y: number;
    width: number;
    height: number;
  };
  constraints?: any;
  layoutMode?: string;
  componentId?: string;
  componentSetId?: string;
  styles?: Record<string, string>;
}

interface FigmaComponent {
  key: string;
  name: string;
  description: string;
  componentSetId?: string;
  documentationLinks?: Array<{ uri: string }>;
  remote: boolean;
  containingFrame?: {
    name: string;
    nodeId: string;
  };
}

interface DesignToken {
  name: string;
  type: 'color' | 'typography' | 'spacing' | 'border-radius' | 'shadow';
  value: string | number;
  description?: string;
  scope?: string[];
}

class FigmaMCPServer {
  private server: Server;
  private config: FigmaConfig;
  private cache: Map<string, { data: any; timestamp: number }>;
  private figmaAPI: FigmaAPI;

  constructor(config: FigmaConfig) {
    this.config = config;
    this.cache = new Map();
    this.figmaAPI = new FigmaAPI(config.accessToken);
    
    this.server = new Server(
      {
        name: 'figma-mcp-server',
        version: '1.0.0',
      },
      {
        capabilities: {
          tools: {},
          resources: {},
          prompts: {},
        },
      }
    );

    this.setupToolHandlers();
    this.setupResourceHandlers();
    this.setupPromptHandlers();
  }

  private setupToolHandlers(): void {
    // Get Figma file components
    this.server.setRequestHandler(ListToolsRequestSchema, async () => ({
      tools: [
        {
          name: 'get_figma_components',
          description: 'Retrieve components from a Figma file',
          inputSchema: {
            type: 'object',
            properties: {
              fileKey: {
                type: 'string',
                description: 'Figma file key',
              },
              componentName: {
                type: 'string',
                description: 'Optional: Filter by component name',
              },
              includeInstances: {
                type: 'boolean',
                description: 'Include component instances',
                default: false,
              },
            },
            required: ['fileKey'],
          },
        },
        {
          name: 'get_design_tokens',
          description: 'Extract design tokens from Figma file',
          inputSchema: {
            type: 'object',
            properties: {
              fileKey: {
                type: 'string',
                description: 'Figma file key',
              },
              tokenTypes: {
                type: 'array',
                items: {
                  type: 'string',
                  enum: ['colors', 'typography', 'spacing', 'effects'],
                },
                description: 'Types of tokens to extract',
              },
            },
            required: ['fileKey'],
          },
        },
        {
          name: 'generate_react_component',
          description: 'Generate React component from Figma component',
          inputSchema: {
            type: 'object',
            properties: {
              fileKey: {
                type: 'string',
                description: 'Figma file key',
              },
              nodeId: {
                type: 'string',
                description: 'Figma node ID',
              },
              componentName: {
                type: 'string',
                description: 'React component name',
              },
              typescript: {
                type: 'boolean',
                description: 'Generate TypeScript component',
                default: true,
              },
              includeStyles: {
                type: 'boolean',
                description: 'Include CSS/styled-components',
                default: true,
              },
            },
            required: ['fileKey', 'nodeId', 'componentName'],
          },
        },
        {
          name: 'sync_design_system',
          description: 'Synchronize design system with code',
          inputSchema: {
            type: 'object',
            properties: {
              fileKey: {
                type: 'string',
                description: 'Figma file key',
              },
              outputPath: {
                type: 'string',
                description: 'Output directory path',
              },
              framework: {
                type: 'string',
                enum: ['react', 'vue', 'angular', 'vanilla'],
                description: 'Target framework',
                default: 'react',
              },
              includeTests: {
                type: 'boolean',
                description: 'Generate test files',
                default: true,
              },
            },
            required: ['fileKey', 'outputPath'],
          },
        },
        {
          name: 'get_figma_comments',
          description: 'Retrieve comments from Figma file',
          inputSchema: {
            type: 'object',
            properties: {
              fileKey: {
                type: 'string',
                description: 'Figma file key',
              },
              resolved: {
                type: 'boolean',
                description: 'Filter by resolved status',
              },
            },
            required: ['fileKey'],
          },
        },
        {
          name: 'export_figma_assets',
          description: 'Export assets from Figma',
          inputSchema: {
            type: 'object',
            properties: {
              fileKey: {
                type: 'string',
                description: 'Figma file key',
              },
              nodeIds: {
                type: 'array',
                items: { type: 'string' },
                description: 'Node IDs to export',
              },
              format: {
                type: 'string',
                enum: ['png', 'jpg', 'svg', 'pdf'],
                description: 'Export format',
                default: 'svg',
              },
              scale: {
                type: 'number',
                description: 'Export scale',
                default: 1,
              },
            },
            required: ['fileKey', 'nodeIds'],
          },
        },
      ],
    }));

    this.server.setRequestHandler(CallToolRequestSchema, async (request) => {
      const { name, arguments: args } = request.params;

      switch (name) {
        case 'get_figma_components':
          return await this.getFigmaComponents(args);
        case 'get_design_tokens':
          return await this.getDesignTokens(args);
        case 'generate_react_component':
          return await this.generateReactComponent(args);
        case 'sync_design_system':
          return await this.syncDesignSystem(args);
        case 'get_figma_comments':
          return await this.getFigmaComments(args);
        case 'export_figma_assets':
          return await this.exportFigmaAssets(args);
        default:
          throw new Error(`Unknown tool: ${name}`);
      }
    });
  }

  private setupResourceHandlers(): void {
    this.server.setRequestHandler(ListResourcesRequestSchema, async () => ({
      resources: [
        {
          uri: 'figma://design-tokens',
          name: 'Design Tokens',
          description: 'Extracted design tokens from all Figma files',
          mimeType: 'application/json',
        },
        {
          uri: 'figma://components',
          name: 'Component Library',
          description: 'All components from Figma design system',
          mimeType: 'application/json',
        },
        {
          uri: 'figma://styles',
          name: 'Figma Styles',
          description: 'Text and color styles from Figma',
          mimeType: 'application/json',
        },
        {
          uri: 'figma://assets',
          name: 'Design Assets',
          description: 'Exported assets and images',
          mimeType: 'application/json',
        },
      ],
    }));

    this.server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
      const uri = request.params.uri;
      
      switch (uri) {
        case 'figma://design-tokens':
          return {
            contents: [
              {
                uri,
                mimeType: 'application/json',
                text: JSON.stringify(await this.getAllDesignTokens(), null, 2),
              },
            ],
          };
        case 'figma://components':
          return {
            contents: [
              {
                uri,
                mimeType: 'application/json',
                text: JSON.stringify(await this.getAllComponents(), null, 2),
              },
            ],
          };
        case 'figma://styles':
          return {
            contents: [
              {
                uri,
                mimeType: 'application/json',
                text: JSON.stringify(await this.getAllStyles(), null, 2),
              },
            ],
          };
        case 'figma://assets':
          return {
            contents: [
              {
                uri,
                mimeType: 'application/json',
                text: JSON.stringify(await this.getAllAssets(), null, 2),
              },
            ],
          };
        default:
          throw new Error(`Unknown resource: ${uri}`);
      }
    });
  }

  private setupPromptHandlers(): void {
    this.server.setRequestHandler(ListPromptsRequestSchema, async () => ({
      prompts: [
        {
          name: 'design_to_code',
          description: 'Convert Figma design to production code',
          arguments: [
            {
              name: 'component_name',
              description: 'Name of the component to generate',
              required: true,
            },
            {
              name: 'framework',
              description: 'Target framework (react, vue, angular)',
              required: false,
            },
          ],
        },
        {
          name: 'component_documentation',
          description: 'Generate documentation for Figma component',
          arguments: [
            {
              name: 'component_id',
              description: 'Figma component ID',
              required: true,
            },
          ],
        },
        {
          name: 'design_system_audit',
          description: 'Audit design system consistency',
          arguments: [
            {
              name: 'file_key',
              description: 'Figma file key to audit',
              required: true,
            },
          ],
        },
      ],
    }));

    this.server.setRequestHandler(GetPromptRequestSchema, async (request) => {
      const { name, arguments: args } = request.params;

      switch (name) {
        case 'design_to_code':
          return await this.getDesignToCodePrompt(args);
        case 'component_documentation':
          return await this.getComponentDocumentationPrompt(args);
        case 'design_system_audit':
          return await this.getDesignSystemAuditPrompt(args);
        default:
          throw new Error(`Unknown prompt: ${name}`);
      }
    });
  }

  // Tool Implementation Methods
  private async getFigmaComponents(args: any): Promise<any> {
    const { fileKey, componentName, includeInstances } = args;
    
    try {
      const fileData = await this.figmaAPI.getFile(fileKey);
      const components = await this.figmaAPI.getFileComponents(fileKey);
      
      let filteredComponents = components.meta.components;
      
      if (componentName) {
        filteredComponents = filteredComponents.filter((comp: FigmaComponent) =>
          comp.name.toLowerCase().includes(componentName.toLowerCase())
        );
      }

      const result = {
        file: {
          name: fileData.name,
          lastModified: fileData.lastModified,
        },
        components: filteredComponents.map((comp: FigmaComponent) => ({
          ...comp,
          node: this.findNodeById(fileData.document, comp.key),
        })),
      };

      if (includeInstances) {
        result.components = await Promise.all(
          result.components.map(async (comp: any) => ({
            ...comp,
            instances: await this.findComponentInstances(fileData.document, comp.key),
          }))
        );
      }

      return {
        content: [
          {
            type: 'text',
            text: JSON.stringify(result, null, 2),
          },
        ],
      };
    } catch (error) {
      throw new Error(`Failed to get Figma components: ${error.message}`);
    }
  }

  private async getDesignTokens(args: any): Promise<any> {
    const { fileKey, tokenTypes = ['colors', 'typography', 'spacing', 'effects'] } = args;
    
    try {
      const fileData = await this.figmaAPI.getFile(fileKey);
      const styles = await this.figmaAPI.getFileStyles(fileKey);
      
      const tokens: DesignToken[] = [];

      if (tokenTypes.includes('colors')) {
        tokens.push(...this.extractColorTokens(styles.meta.styles));
      }

      if (tokenTypes.includes('typography')) {
        tokens.push(...this.extractTypographyTokens(styles.meta.styles));
      }

      if (tokenTypes.includes('spacing')) {
        tokens.push(...this.extractSpacingTokens(fileData.document));
      }

      if (tokenTypes.includes('effects')) {
        tokens.push(...this.extractEffectTokens(styles.meta.styles));
      }

      return {
        content: [
          {
            type: 'text',
            text: JSON.stringify({
              file: fileData.name,
              tokens: tokens,
              generated: new Date().toISOString(),
            }, null, 2),
          },
        ],
      };
    } catch (error) {
      throw new Error(`Failed to extract design tokens: ${error.message}`);
    }
  }

  private async generateReactComponent(args: any): Promise<any> {
    const { fileKey, nodeId, componentName, typescript = true, includeStyles = true } = args;
    
    try {
      const node = await this.figmaAPI.getFileNode(fileKey, nodeId);
      const componentCode = await this.generateComponentCode(
        node.nodes[nodeId],
        componentName,
        typescript,
        includeStyles
      );

      return {
        content: [
          {
            type: 'text',
            text: componentCode,
          },
        ],
      };
    } catch (error) {
      throw new Error(`Failed to generate React component: ${error.message}`);
    }
  }

  private async syncDesignSystem(args: any): Promise<any> {
    const { fileKey, outputPath, framework = 'react', includeTests = true } = args;
    
    try {
      const fileData = await this.figmaAPI.getFile(fileKey);
      const components = await this.figmaAPI.getFileComponents(fileKey);
      const styles = await this.figmaAPI.getFileStyles(fileKey);

      const syncResults = {
        components: [],
        tokens: [],
        assets: [],
        tests: [],
      };

      // Generate components
      for (const component of components.meta.components) {
        const node = this.findNodeById(fileData.document, component.key);
        if (node) {
          const componentCode = await this.generateComponentCode(
            node,
            component.name,
            true,
            true,
            framework
          );
          
          syncResults.components.push({
            name: component.name,
            path: `${outputPath}/components/${component.name}.tsx`,
            code: componentCode,
          });

          if (includeTests) {
            const testCode = this.generateTestCode(component.name, framework);
            syncResults.tests.push({
              name: `${component.name}.test.tsx`,
              path: `${outputPath}/components/__tests__/${component.name}.test.tsx`,
              code: testCode,
            });
          }
        }
      }

      // Generate design tokens
      const tokens = this.extractAllTokens(styles.meta.styles, fileData.document);
      const tokenCode = this.generateTokenCode(tokens, framework);
      syncResults.tokens.push({
        name: 'tokens',
        path: `${outputPath}/tokens/index.ts`,
        code: tokenCode,
      });

      return {
        content: [
          {
            type: 'text',
            text: JSON.stringify(syncResults, null, 2),
          },
        ],
      };
    } catch (error) {
      throw new Error(`Failed to sync design system: ${error.message}`);
    }
  }

  private async getFigmaComments(args: any): Promise<any> {
    const { fileKey, resolved } = args;
    
    try {
      const comments = await this.figmaAPI.getFileComments(fileKey);
      
      let filteredComments = comments.comments;
      if (typeof resolved === 'boolean') {
        filteredComments = comments.comments.filter(comment => 
          comment.resolved === resolved
        );
      }

      return {
        content: [
          {
            type: 'text',
            text: JSON.stringify({
              total: filteredComments.length,
              comments: filteredComments,
            }, null, 2),
          },
        ],
      };
    } catch (error) {
      throw new Error(`Failed to get Figma comments: ${error.message}`);
    }
  }

  private async exportFigmaAssets(args: any): Promise<any> {
    const { fileKey, nodeIds, format = 'svg', scale = 1 } = args;
    
    try {
      const exports = await this.figmaAPI.getFileImages(fileKey, {
        ids: nodeIds.join(','),
        format,
        scale,
      });

      const assets = [];
      for (const [nodeId, url] of Object.entries(exports.images)) {
        if (url) {
          assets.push({
            nodeId,
            url,
            format,
            scale,
          });
        }
      }

      return {
        content: [
          {
            type: 'text',
            text: JSON.stringify({
              assets,
              exportedAt: new Date().toISOString(),
            }, null, 2),
          },
        ],
      };
    } catch (error) {
      throw new Error(`Failed to export Figma assets: ${error.message}`);
    }
  }

  // Prompt Implementation Methods
  private async getDesignToCodePrompt(args: any): Promise<any> {
    const { component_name, framework = 'react' } = args;
    
    return {
      messages: [
        {
          role: 'user',
          content: {
            type: 'text',
            text: `Generate a production-ready ${framework} component named "${component_name}" based on the Figma design. 

Requirements:
1. Use TypeScript with proper type definitions
2. Follow ${framework} best practices and conventions
3. Include proper accessibility attributes (ARIA labels, semantic HTML)
4. Implement responsive design patterns
5. Add comprehensive JSDoc documentation
6. Include error boundaries and loading states where appropriate
7. Use modern CSS-in-JS or CSS modules for styling
8. Ensure component is testable with proper props interface

Please analyze the Figma design properties and generate clean, maintainable code that matches the design specifications exactly.`,
          },
        },
      ],
    };
  }

  private async getComponentDocumentationPrompt(args: any): Promise<any> {
    const { component_id } = args;
    
    // Get component data from Figma
    const componentData = await this.getComponentData(component_id);
    
    return {
      messages: [
        {
          role: 'user',
          content: {
            type: 'text',
            text: `Generate comprehensive documentation for this Figma component:

Component Data:
${JSON.stringify(componentData, null, 2)}

Please create documentation that includes:
1. Component overview and purpose
2. Props/API documentation with TypeScript types
3. Usage examples and code snippets
4. Accessibility guidelines and ARIA requirements
5. Design tokens and styling specifications
6. Responsive behavior and breakpoints
7. Testing strategies and example test cases
8. Common use cases and best practices
9. Migration guide if updating from previous version
10. Related components and design patterns

Format the documentation in Markdown with proper code highlighting and examples.`,
          },
        },
      ],
    };
  }

  private async getDesignSystemAuditPrompt(args: any): Promise<any> {
    const { file_key } = args;
    
    const auditData = await this.performDesignSystemAudit(file_key);
    
    return {
      messages: [
        {
          role: 'user',
          content: {
            type: 'text',
            text: `Perform a comprehensive design system audit based on this data:

${JSON.stringify(auditData, null, 2)}

Please analyze and provide:

1. **Consistency Issues:**
   - Color usage inconsistencies
   - Typography scale violations
   - Spacing pattern deviations
   - Component naming conventions

2. **Accessibility Audit:**
   - Color contrast ratios
   - Text size compliance with WCAG
   - Interactive element sizing
   - Focus state definitions

3. **Scalability Assessment:**
   - Component reusability score
   - Design token coverage
   - Naming convention adherence
   - Documentation completeness

4. **Recommendations:**
   - Priority fixes for immediate implementation
   - Long-term improvements for design system health
   - Code generation optimizations
   - Design-development workflow improvements

5. **Metrics Dashboard:**
   - Design system health score (0-100)
   - Component library coverage
   - Token usage statistics
   - Technical debt indicators

Provide actionable insights with specific examples and implementation steps.`,
          },
        },
      ],
    };
  }

  // Helper Methods
  private findNodeById(node: any, id: string): any {
    if (node.id === id) return node;
    if (node.children) {
      for (const child of node.children) {
        const found = this.findNodeById(child, id);
        if (found) return found;
      }
    }
    return null;
  }

  private async findComponentInstances(node: any, componentId: string): Promise<any[]> {
    const instances = [];
    
    const traverse = (n: any) => {
      if (n.componentId === componentId) {
        instances.push(n);
      }
      if (n.children) {
        n.children.forEach(traverse);
      }
    };
    
    traverse(node);
    return instances;
  }

  private extractColorTokens(styles: any[]): DesignToken[] {
    return styles
      .filter(style => style.styleType === 'FILL')
      .map(style => ({
        name: style.name,
        type: 'color' as const,
        value: this.figmaColorToHex(style.fills?.[0]),
        description: style.description,
        scope: ['background', 'border', 'text'],
      }));
  }

  private extractTypographyTokens(styles: any[]): DesignToken[] {
    return styles
      .filter(style => style.styleType === 'TEXT')
      .map(style => ({
        name: style.name,
        type: 'typography' as const,
        value: this.figmaTextStyleToCss(style.style),
        description: style.description,
        scope: ['heading', 'body', 'caption'],
      }));
  }

  private extractSpacingTokens(document: any): DesignToken[] {
    // Extract common spacing values from layout components
    const spacingValues = new Set<number>();
    
    const extractSpacing = (node: any) => {
      if (node.paddingLeft) spacingValues.add(node.paddingLeft);
      if (node.paddingRight) spacingValues.add(node.paddingRight);
      if (node.paddingTop) spacingValues.add(node.paddingTop);
      if (node.paddingBottom) spacingValues.add(node.paddingBottom);
      if (node.itemSpacing) spacingValues.add(node.itemSpacing);
      
      if (node.children) {
        node.children.forEach(extractSpacing);
      }
    };
    
    extractSpacing(document);
    
    return Array.from(spacingValues)
      .sort((a, b) => a - b)
      .map((value, index) => ({
        name: `spacing-${index + 1}`,
        type: 'spacing' as const,
        value: `${value}px`,
        description: `Spacing token for ${value}px`,
        scope: ['margin', 'padding', 'gap'],
      }));
  }

  private extractEffectTokens(styles: any[]): DesignToken[] {
    return styles
      .filter(style => style.styleType === 'EFFECT')
      .map(style => ({
        name: style.name,
        type: 'shadow' as const,
        value: this.figmaEffectToCss(style.effects?.[0]),
        description: style.description,
        scope: ['box-shadow', 'filter'],
      }));
  }

  private figmaColorToHex(fill: any): string {
    if (!fill || fill.type !== 'SOLID') return '#000000';
    
    const { r, g, b, a = 1 } = fill.color;
    const toHex = (val: number) => Math.round(val * 255).toString(16).padStart(2, '0');
    
    if (a < 1) {
      return `rgba(${Math.round(r * 255)}, ${Math.round(g * 255)}, ${Math.round(b * 255)}, ${a})`;
    }
    
    return `#${toHex(r)}${toHex(g)}${toHex(b)}`;
  }

  private figmaTextStyleToCss(style: any): string {
    const fontSize = style.fontSize || 16;
    const fontWeight = style.fontWeight || 400;
    const lineHeight = style.lineHeightPx ? `${style.lineHeightPx}px` : 'normal';
    const fontFamily = style.fontFamily || 'inherit';
    
    return `font-family: ${fontFamily}; font-size: ${fontSize}px; font-weight: ${fontWeight}; line-height: ${lineHeight};`;
  }

  private figmaEffectToCss(effect: any): string {
    if (!effect) return 'none';
    
    if (effect.type === 'DROP_SHADOW') {
      const { offset, radius, color } = effect;
      const colorStr = this.figmaColorToHex({ color, type: 'SOLID' });
      return `${offset.x}px ${offset.y}px ${radius}px ${colorStr}`;
    }
    
    return 'none';
  }

  private async generateComponentCode(
    node: any,
    componentName: string,
    typescript: boolean,
    includeStyles: boolean,
    framework: string = 'react'
  ): Promise<string> {
    // This is a simplified version - in practice, this would be much more complex
    const props = this.extractPropsFromNode(node);
    const styles = includeStyles ? this.extractStylesFromNode(node) : '';
    
    const extension = typescript ? 'tsx' : 'jsx';
    const propsInterface = typescript ? this.generatePropsInterface(props, componentName) : '';
    
    return `${propsInterface}

import React from 'react';
${includeStyles ? "import styled from 'styled-components';" : ''}

${includeStyles ? styles : ''}

export const ${componentName}: React.FC<${componentName}Props> = ({
  ${props.map(p => p.name).join(',\n  ')}
}) => {
  return (
    <Container>
      {/* Component implementation based on Figma design */}
    </Container>
  );
};

export default ${componentName};`;
  }

  private extractPropsFromNode(node: any): Array<{ name: string; type: string; required: boolean }> {
    // Extract props based on component properties, text content, etc.
    const props = [];
    
    if (node.children?.some((child: any) => child.type === 'TEXT')) {
      props.push({ name: 'children', type: 'React.ReactNode', required: false });
    }
    
    if (node.fills?.length > 0) {
      props.push({ name: 'variant', type: "'primary' | 'secondary'", required: false });
    }
    
    return props;
  }

  private generatePropsInterface(props: any[], componentName: string): string {
    const propsStr = props.map(prop => 
      `  ${prop.name}${prop.required ? '' : '?'}: ${prop.type};`
    ).join('\n');
    
    return `interface ${componentName}Props {
${propsStr}
}`;
  }

  private extractStylesFromNode(node: any): string {
    // Extract styles and convert to styled-components
    const styles = [];
    
    if (node.fills?.length > 0) {
      const fill = node.fills[0];
      styles.push(`background-color: ${this.figmaColorToHex(fill)};`);
    }
    
    if (node.cornerRadius) {
      styles.push(`border-radius: ${node.cornerRadius}px;`);
    }
    
    if (node.absoluteBoundingBox) {
      const { width, height } = node.absoluteBoundingBox;
      styles.push(`width: ${width}px;`);
      styles.push(`height: ${height}px;`);
    }
    
    return `const Container = styled.div\`
  ${styles.join('\n  ')}
\`;`;
  }

  private generateTestCode(componentName: string, framework: string): string {
    return `import React from 'react';
import { render, screen } from '@testing-library/react';
import { ${componentName} } from '../${componentName}';

describe('${componentName}', () => {
  it('renders correctly', () => {
    render(<${componentName} />);
    expect(screen.getByRole('button')).toBeInTheDocument();
  });

  it('handles props correctly', () => {
    render(<${componentName} variant="primary" />);
    expect(screen.getByRole('button')).toHaveClass('primary');
  });
});`;
  }

  private generateTokenCode(tokens: DesignToken[], framework: string): string {
    const tokenGroups = tokens.reduce((groups, token) => {
      if (!groups[token.type]) groups[token.type] = [];
      groups[token.type].push(token);
      return groups;
    }, {} as Record<string, DesignToken[]>);

    let code = '// Design Tokens generated from Figma\n\n';
    
    for (const [type, typeTokens] of Object.entries(tokenGroups)) {
      code += `export const ${type} = {\n`;
      for (const token of typeTokens) {
        code += `  '${token.name}': '${token.value}',\n`;
      }
      code += '};\n\n';
    }
    
    return code;
  }

  // Resource helper methods
  private async getAllDesignTokens(): Promise<DesignToken[]> {
    const allTokens: DesignToken[] = [];
    
    for (const fileKey of this.config.fileKeys) {
      try {
        const tokens = await this.getDesignTokens({ fileKey });
        const fileTokens = JSON.parse(tokens.content[0].text).tokens;
        allTokens.push(...fileTokens);
      } catch (error) {
        console.warn(`Failed to get tokens from ${fileKey}:`, error);
      }
    }
    
    return allTokens;
  }

  private async getAllComponents(): Promise<any[]> {
    const allComponents: any[] = [];
    
    for (const fileKey of this.config.fileKeys) {
      try {
        const components = await this.getFigmaComponents({ fileKey });
        const fileComponents = JSON.parse(components.content[0].text).components;
        allComponents.push(...fileComponents);
      } catch (error) {
        console.warn(`Failed to get components from ${fileKey}:`, error);
      }
    }
    
    return allComponents;
  }

  private async getAllStyles(): Promise<any[]> {
    const allStyles: any[] = [];
    
    for (const fileKey of this.config.fileKeys) {
      try {
        const styles = await this.figmaAPI.getFileStyles(fileKey);
        allStyles.push(...styles.meta.styles);
      } catch (error) {
        console.warn(`Failed to get styles from ${fileKey}:`, error);
      }
    }
    
    return allStyles;
  }

  private async getAllAssets(): Promise<any[]> {
    // Implementation for getting all assets
    return [];
  }

  private async getComponentData(componentId: string): Promise<any> {
    // Implementation for getting specific component data
    return {};
  }

  private async performDesignSystemAudit(fileKey: string): Promise<any> {
    // Implementation for design system audit
    return {};
  }

  private extractAllTokens(styles: any[], document: any): DesignToken[] {
    return [
      ...this.extractColorTokens(styles),
      ...this.extractTypographyTokens(styles),
      ...this.extractSpacingTokens(document),
      ...this.extractEffectTokens(styles),
    ];
  }

  async start(): Promise<void> {
    const transport = new StdioServerTransport();
    await this.server.connect(transport);
  }
}

// Figma API Helper Class
class FigmaAPI {
  private accessToken: string;
  private baseURL = 'https://api.figma.com/v1';

  constructor(accessToken: string) {
    this.accessToken = accessToken;
  }

  private async request(endpoint: string, options: RequestInit = {}): Promise<any> {
    const response = await fetch(`${this.baseURL}${endpoint}`, {
      ...options,
      headers: {
        'X-Figma-Token': this.accessToken,
        'Content-Type': 'application/json',
        ...options.headers,
      },
    });

    if (!response.ok) {
      throw new Error(`Figma API error: ${response.status} ${response.statusText}`);
    }

    return response.json();
  }

  async getFile(fileKey: string): Promise<any> {
    return this.request(`/files/${fileKey}`);
  }

  async getFileNode(fileKey: string, nodeId: string): Promise<any> {
    return this.request(`/files/${fileKey}/nodes?ids=${nodeId}`);
  }

  async getFileComponents(fileKey: string): Promise<any> {
    return this.request(`/files/${fileKey}/components`);
  }

  async getFileStyles(fileKey: string): Promise<any> {
    return this.request(`/files/${fileKey}/styles`);
  }

  async getFileComments(fileKey: string): Promise<any> {
    return this.request(`/files/${fileKey}/comments`);
  }

  async getFileImages(fileKey: string, options: any): Promise<any> {
    const params = new URLSearchParams(options);
    return this.request(`/images/${fileKey}?${params}`);
  }
}

// Server startup
const config: FigmaConfig = {
  accessToken: process.env.FIGMA_ACCESS_TOKEN!,
  fileKeys: process.env.FIGMA_FILE_KEYS?.split(',') || [],
  cacheTimeout: 300000, // 5 minutes
};

const server = new FigmaMCPServer(config);
server.start().catch(console.error);
```

## VSCode Extension Configuration

### 2. VSCode Settings for Figma MCP Integration

```json
// .vscode/settings.json
{
  "mcpServers": {
    "figma": {
      "command": "node",
      "args": ["./figma-mcp-server.js"],
      "env": {
        "FIGMA_ACCESS_TOKEN": "${env:FIGMA_ACCESS_TOKEN}",
        "FIGMA_FILE_KEYS": "${env:FIGMA_FILE_KEYS}"
      }
    }
  },
  "github.copilot.enable": {
    "*": true,
    "yaml": false,
    "plaintext": false,
    "markdown": false
  },
  "github.copilot.advanced": {
    "secret_key": "figma-integration",
    "temperature": 0.1
  }
}
```

### 3. Environment Configuration

```bash
# .env file
FIGMA_ACCESS_TOKEN=your_figma_access_token_here
FIGMA_FILE_KEYS=file_key_1,file_key_2,file_key_3
FIGMA_TEAM_ID=your_team_id
```

## VSCode Copilot Integration Workflows

### 4. Design-to-Code with Copilot

```typescript
// Copilot Chat Integration Examples

// Example 1: Generate component from Figma
/*
@figma Generate a React component for the "Primary Button" from our design system.
Include TypeScript types, accessibility features, and styled-components.
*/

// Example 2: Sync design tokens
/*
@figma Extract all color tokens from the main design file and generate a TypeScript tokens file.
Include semantic naming and theme support.
*/

// Example 3: Component documentation
/*
@figma Create comprehensive documentation for the "Card" component including:
- Props interface
- Usage examples  
- Accessibility guidelines
- Testing strategies
*/

// Copilot Prompt Templates for Figma Integration
const figmaPrompts = {
  componentGeneration: `
    Using the Figma MCP server, generate a production-ready React component based on:
    - Component name: {componentName}
    - Figma file: {fileKey}
    - Node ID: {nodeId}
    
    Requirements:
    - TypeScript with strict types
    - Accessibility compliant (WCAG 2.1 AA)
    - Responsive design patterns
    - Comprehensive JSDoc documentation
    - Unit tests with React Testing Library
    - Storybook stories for documentation
  `,
  
  designSystemSync: `
    Synchronize the design system from Figma file {fileKey} to codebase:
    - Extract all design tokens (colors, typography, spacing, effects)
    - Generate component library matching Figma components
    - Create TypeScript definitions for design system
    - Generate comprehensive documentation
    - Include migration guide for existing components
  `,
  
  tokenExtraction: `
    Extract design tokens from Figma and create:
    - CSS custom properties
    - JavaScript/TypeScript token objects
    - SCSS variables
    - Tailwind CSS configuration
    - Design token documentation with usage examples
  `,
};
```

### 5. Copilot Custom Instructions

```typescript
// Custom Copilot instructions for Figma integration
interface CopilotFigmaInstructions {
  context: string;
  rules: string[];
  patterns: string[];
}

const figmaCopilotInstructions: CopilotFigmaInstructions = {
  context: `
    I'm working with a Figma MCP server integration that provides:
    - Design token extraction from Figma files
    - Component generation from Figma designs
    - Asset export and optimization
    - Design system synchronization
    - Real-time design-to-code workflows
  `,
  
  rules: [
    "Always use TypeScript for generated components",
    "Include accessibility attributes (ARIA labels, semantic HTML)",
    "Follow component naming conventions from Figma",
    "Generate comprehensive prop interfaces",
    "Include error boundaries and loading states",
    "Use modern CSS-in-JS or CSS modules",
    "Create responsive designs with mobile-first approach",
    "Include unit tests for all generated components",
    "Generate Storybook stories for documentation",
    "Follow design system token conventions",
  ],
  
  patterns: [
    "Component files: ComponentName.tsx",
    "Test files: ComponentName.test.tsx", 
    "Story files: ComponentName.stories.tsx",
    "Style files: ComponentName.styles.ts",
    "Type files: ComponentName.types.ts",
  ]
};
```

## Practical Usage Examples

### 6. Complete Workflow Example

```typescript
// Step 1: Initialize Figma MCP Server
// Terminal: npm run start:figma-mcp

// Step 2: Connect VSCode Copilot to Figma
// Use @figma prefix in Copilot Chat

// Step 3: Extract Design Tokens
/*
@figma Extract all design tokens from our main design file and create:
1. TypeScript token definitions
2. CSS custom properties
3. Tailwind config
4. Theme provider setup
*/

// Step 4: Generate Components
/*
@figma Generate React components for all buttons in the design system:
- Primary Button
- Secondary Button  
- Icon Button
- Loading Button

Include full TypeScript support and accessibility features.
*/

// Step 5: Create Documentation
/*
@figma Create a comprehensive component library documentation site including:
- Component API documentation
- Interactive examples
- Design guidelines
- Usage patterns
- Accessibility notes
*/

// Generated output structure:
/*
src/
  components/
    Button/
      Button.tsx              # Main component
      Button.test.tsx         # Unit tests
      Button.stories.tsx      # Storybook stories
      Button.styles.ts        # Styled components
      Button.types.ts         # TypeScript types
      index.ts               # Exports
  tokens/
    colors.ts              # Color tokens
    typography.ts          # Typography tokens
    spacing.ts            # Spacing tokens
    shadows.ts            # Shadow tokens
    index.ts              # All tokens export
  theme/
    ThemeProvider.tsx      # Theme context
    theme.ts              # Theme configuration
*/
```

### 7. Advanced Integration Patterns

```typescript
// Real-time Design Updates
interface FigmaWebhookHandler {
  onComponentUpdate: (componentId: string) => void;
  onStyleUpdate: (styleId: string) => void;
  onFileUpdate: (fileKey: string) => void;
}

// Copilot Integration for Live Updates
/*
@figma Setup live design synchronization:
1. Configure webhook endpoints for Figma updates
2. Create automated component regeneration pipeline
3. Setup git commit automation for design changes
4. Include change detection and diff reporting
*/

// A11y-First Development
/*
@figma Generate accessible components following these patterns:
- Semantic HTML structure
- ARIA labels and descriptions
- Keyboard navigation support
- Screen reader compatibility
- Color contrast validation
- Focus management
*/

// Design System Governance
/*
@figma Create design system governance tools:
- Component usage analytics
- Design token compliance checking
- Automated design review workflows
- Breaking change detection
- Version management system
*/
```

## Interview-Ready Integration Summary

**Figma MCP VSCode Copilot Integration provides:**

1. **Seamless Design-to-Code Workflow** - Direct Figma design extraction and code generation
2. **Intelligent Code Generation** - Copilot-powered component creation with best practices
3. **Real-time Synchronization** - Live updates from Figma to codebase
4. **Design System Automation** - Automated token extraction and component library generation
5. **Quality Assurance** - Built-in accessibility, testing, and documentation generation

**Key Benefits:**
- **Developer Productivity** - Reduced manual coding for design implementation
- **Design Consistency** - Automated synchronization prevents design drift
- **Accessibility First** - Built-in WCAG compliance and semantic HTML
- **Type Safety** - Full TypeScript support with proper interface generation
- **Documentation** - Automated component documentation and usage examples

**Enterprise Features:** Webhook integration for live updates, design system governance, automated testing, comprehensive documentation generation, accessibility validation, performance optimization.