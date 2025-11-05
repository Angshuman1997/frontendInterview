# GraphQL Fundamentals and Query Optimization

GraphQL is a query language and runtime for APIs that provides a more efficient, powerful, and flexible alternative to REST. This guide covers GraphQL fundamentals, query optimization techniques, and best practices for frontend applications.

## GraphQL Core Concepts

### 1. GraphQL Schema and Type System
```typescript
// GraphQL Schema Definition Language (SDL)
const typeDefs = `
  # Scalar types
  scalar Date
  scalar Upload
  scalar JSON

  # Enums
  enum UserRole {
    ADMIN
    MODERATOR
    USER
    GUEST
  }

  enum PostStatus {
    DRAFT
    PUBLISHED
    ARCHIVED
  }

  enum SortOrder {
    ASC
    DESC
  }

  # Input types for mutations
  input CreateUserInput {
    name: String!
    email: String!
    password: String!
    role: UserRole = USER
    avatar: Upload
  }

  input UpdateUserInput {
    id: ID!
    name: String
    email: String
    role: UserRole
    avatar: Upload
  }

  input PostFilters {
    status: PostStatus
    authorId: ID
    tags: [String!]
    publishedAfter: Date
    publishedBefore: Date
  }

  input PaginationInput {
    limit: Int = 10
    offset: Int = 0
    cursor: String
  }

  input SortInput {
    field: String!
    order: SortOrder = ASC
  }

  # Object types
  type User {
    id: ID!
    name: String!
    email: String!
    role: UserRole!
    avatar: String
    posts(
      filters: PostFilters
      pagination: PaginationInput
      sort: SortInput
    ): PostConnection!
    followers: [User!]!
    following: [User!]!
    createdAt: Date!
    updatedAt: Date!
  }

  type Post {
    id: ID!
    title: String!
    content: String!
    excerpt: String
    status: PostStatus!
    author: User!
    tags: [Tag!]!
    comments: [Comment!]!
    likes: Int!
    views: Int!
    publishedAt: Date
    createdAt: Date!
    updatedAt: Date!
  }

  type Comment {
    id: ID!
    content: String!
    author: User!
    post: Post!
    parent: Comment
    replies: [Comment!]!
    likes: Int!
    createdAt: Date!
    updatedAt: Date!
  }

  type Tag {
    id: ID!
    name: String!
    slug: String!
    posts: [Post!]!
    color: String
  }

  # Connection types for pagination
  type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
  }

  type PostEdge {
    node: Post!
    cursor: String!
  }

  type PostConnection {
    edges: [PostEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }

  # Union types
  union SearchResult = User | Post | Tag

  # Interface types
  interface Node {
    id: ID!
  }

  interface Timestamped {
    createdAt: Date!
    updatedAt: Date!
  }

  # Root types
  type Query {
    # User queries
    user(id: ID!): User
    users(
      filters: UserFilters
      pagination: PaginationInput
      sort: SortInput
    ): [User!]!
    me: User

    # Post queries
    post(id: ID!): Post
    posts(
      filters: PostFilters
      pagination: PaginationInput
      sort: SortInput
    ): PostConnection!
    
    # Search
    search(
      query: String!
      type: SearchType
      pagination: PaginationInput
    ): [SearchResult!]!

    # Analytics
    postAnalytics(postId: ID!): PostAnalytics
  }

  type Mutation {
    # User mutations
    createUser(input: CreateUserInput!): User!
    updateUser(input: UpdateUserInput!): User!
    deleteUser(id: ID!): Boolean!

    # Post mutations
    createPost(input: CreatePostInput!): Post!
    updatePost(input: UpdatePostInput!): Post!
    deletePost(id: ID!): Boolean!
    publishPost(id: ID!): Post!

    # Comment mutations
    addComment(input: AddCommentInput!): Comment!
    updateComment(input: UpdateCommentInput!): Comment!
    deleteComment(id: ID!): Boolean!

    # Interaction mutations
    likePost(postId: ID!): Post!
    unlikePost(postId: ID!): Post!
    followUser(userId: ID!): User!
    unfollowUser(userId: ID!): User!
  }

  type Subscription {
    # Real-time updates
    postAdded: Post!
    postUpdated: Post!
    commentAdded(postId: ID!): Comment!
    userOnline: User!
    userOffline: User!
    
    # Live analytics
    postViewsUpdated(postId: ID!): PostAnalytics!
  }
`;

// TypeScript type generation from GraphQL schema
interface User {
  id: string;
  name: string;
  email: string;
  role: UserRole;
  avatar?: string;
  posts: PostConnection;
  followers: User[];
  following: User[];
  createdAt: Date;
  updatedAt: Date;
}

interface Post {
  id: string;
  title: string;
  content: string;
  excerpt?: string;
  status: PostStatus;
  author: User;
  tags: Tag[];
  comments: Comment[];
  likes: number;
  views: number;
  publishedAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}

enum UserRole {
  ADMIN = 'ADMIN',
  MODERATOR = 'MODERATOR',
  USER = 'USER',
  GUEST = 'GUEST'
}

enum PostStatus {
  DRAFT = 'DRAFT',
  PUBLISHED = 'PUBLISHED',
  ARCHIVED = 'ARCHIVED'
}

// GraphQL resolvers with TypeScript
import { GraphQLResolveInfo } from 'graphql';
import { Context } from './types';

interface ResolverArgs {
  [key: string]: any;
}

type Resolver<T = any> = (
  parent: any,
  args: ResolverArgs,
  context: Context,
  info: GraphQLResolveInfo
) => T | Promise<T>;

const resolvers = {
  Query: {
    user: async (parent, { id }, context, info): Promise<User | null> => {
      // Use DataLoader for efficient batching
      return context.dataloaders.userLoader.load(id);
    },

    users: async (parent, { filters, pagination, sort }, context): Promise<User[]> => {
      const { limit = 10, offset = 0 } = pagination || {};
      
      // Build database query with filters
      const query = context.db.user.findMany({
        where: buildUserFilters(filters),
        take: limit,
        skip: offset,
        orderBy: buildSortOrder(sort),
        include: {
          posts: true,
          followers: true,
          following: true,
        },
      });

      return query;
    },

    me: async (parent, args, context): Promise<User | null> => {
      if (!context.user) {
        throw new Error('Not authenticated');
      }
      return context.dataloaders.userLoader.load(context.user.id);
    },

    posts: async (parent, { filters, pagination, sort }, context): Promise<PostConnection> => {
      const { limit = 10, offset = 0, cursor } = pagination || {};
      
      // Cursor-based pagination for better performance
      const where = {
        ...buildPostFilters(filters),
        ...(cursor ? { id: { gt: cursor } } : {}),
      };

      const posts = await context.db.post.findMany({
        where,
        take: limit + 1, // Fetch one extra to determine hasNextPage
        orderBy: buildSortOrder(sort) || { createdAt: 'desc' },
        include: {
          author: true,
          tags: true,
          comments: {
            include: { author: true },
            orderBy: { createdAt: 'desc' },
            take: 5, // Limit initial comments
          },
        },
      });

      const hasNextPage = posts.length > limit;
      const nodes = hasNextPage ? posts.slice(0, -1) : posts;
      
      const edges = nodes.map(post => ({
        node: post,
        cursor: post.id,
      }));

      const totalCount = await context.db.post.count({ where: buildPostFilters(filters) });

      return {
        edges,
        pageInfo: {
          hasNextPage,
          hasPreviousPage: offset > 0,
          startCursor: edges[0]?.cursor,
          endCursor: edges[edges.length - 1]?.cursor,
        },
        totalCount,
      };
    },

    search: async (parent, { query, type, pagination }, context): Promise<SearchResult[]> => {
      const { limit = 20, offset = 0 } = pagination || {};
      
      // Full-text search implementation
      const results = await context.searchService.search({
        query,
        types: type ? [type] : ['USER', 'POST', 'TAG'],
        limit,
        offset,
      });

      return results;
    },
  },

  Mutation: {
    createPost: async (parent, { input }, context): Promise<Post> => {
      if (!context.user) {
        throw new Error('Not authenticated');
      }

      // Validate input
      const validatedInput = await validateCreatePostInput(input);
      
      // Create post with transaction
      const post = await context.db.$transaction(async (tx) => {
        const newPost = await tx.post.create({
          data: {
            ...validatedInput,
            authorId: context.user.id,
            status: 'DRAFT',
          },
          include: {
            author: true,
            tags: true,
          },
        });

        // Update search index
        await context.searchService.indexPost(newPost);
        
        return newPost;
      });

      // Publish to subscriptions
      context.pubsub.publish('POST_ADDED', { postAdded: post });

      return post;
    },

    likePost: async (parent, { postId }, context): Promise<Post> => {
      if (!context.user) {
        throw new Error('Not authenticated');
      }

      const post = await context.db.$transaction(async (tx) => {
        // Check if already liked
        const existingLike = await tx.like.findUnique({
          where: {
            userId_postId: {
              userId: context.user.id,
              postId,
            },
          },
        });

        if (existingLike) {
          throw new Error('Post already liked');
        }

        // Create like and update post likes count
        await tx.like.create({
          data: {
            userId: context.user.id,
            postId,
          },
        });

        const updatedPost = await tx.post.update({
          where: { id: postId },
          data: { likes: { increment: 1 } },
          include: {
            author: true,
            tags: true,
            comments: { include: { author: true } },
          },
        });

        return updatedPost;
      });

      // Real-time update
      context.pubsub.publish('POST_UPDATED', { postUpdated: post });

      return post;
    },
  },

  Subscription: {
    postAdded: {
      subscribe: (parent, args, context) => {
        return context.pubsub.asyncIterator(['POST_ADDED']);
      },
    },

    commentAdded: {
      subscribe: (parent, { postId }, context) => {
        return context.pubsub.asyncIterator([`COMMENT_ADDED_${postId}`]);
      },
    },

    postViewsUpdated: {
      subscribe: (parent, { postId }, context) => {
        return context.pubsub.asyncIterator([`POST_VIEWS_${postId}`]);
      },
    },
  },

  // Field resolvers
  User: {
    posts: async (user, { filters, pagination, sort }, context): Promise<PostConnection> => {
      // Use field-level resolver for posts
      const userPosts = await context.dataloaders.userPostsLoader.load({
        userId: user.id,
        filters,
        pagination,
        sort,
      });
      
      return userPosts;
    },

    followers: async (user, args, context): Promise<User[]> => {
      return context.dataloaders.userFollowersLoader.load(user.id);
    },
  },

  Post: {
    author: async (post, args, context): Promise<User> => {
      return context.dataloaders.userLoader.load(post.authorId);
    },

    comments: async (post, args, context): Promise<Comment[]> => {
      return context.dataloaders.postCommentsLoader.load(post.id);
    },

    tags: async (post, args, context): Promise<Tag[]> => {
      return context.dataloaders.postTagsLoader.load(post.id);
    },
  },

  // Union type resolver
  SearchResult: {
    __resolveType: (obj): string => {
      if (obj.email) return 'User';
      if (obj.content) return 'Post';
      if (obj.slug) return 'Tag';
      return null;
    },
  },
};

// Helper functions
function buildUserFilters(filters?: any): any {
  if (!filters) return {};
  
  const where: any = {};
  
  if (filters.role) where.role = filters.role;
  if (filters.createdAfter) where.createdAt = { gte: filters.createdAfter };
  if (filters.createdBefore) where.createdAt = { ...where.createdAt, lte: filters.createdBefore };
  
  return where;
}

function buildPostFilters(filters?: any): any {
  if (!filters) return {};
  
  const where: any = {};
  
  if (filters.status) where.status = filters.status;
  if (filters.authorId) where.authorId = filters.authorId;
  if (filters.tags?.length) {
    where.tags = {
      some: {
        name: { in: filters.tags }
      }
    };
  }
  if (filters.publishedAfter) where.publishedAt = { gte: filters.publishedAfter };
  if (filters.publishedBefore) where.publishedAt = { ...where.publishedAt, lte: filters.publishedBefore };
  
  return where;
}

function buildSortOrder(sort?: any): any {
  if (!sort) return undefined;
  
  return {
    [sort.field]: sort.order.toLowerCase(),
  };
}
```

### 2. DataLoader for N+1 Query Problem
```typescript
// DataLoader implementation for efficient data fetching
import DataLoader from 'dataloader';
import { User, Post, Comment, Tag } from './types';

interface DataLoaders {
  userLoader: DataLoader<string, User>;
  postLoader: DataLoader<string, Post>;
  userPostsLoader: DataLoader<UserPostsKey, PostConnection>;
  postCommentsLoader: DataLoader<string, Comment[]>;
  postTagsLoader: DataLoader<string, Tag[]>;
  userFollowersLoader: DataLoader<string, User[]>;
}

interface UserPostsKey {
  userId: string;
  filters?: any;
  pagination?: any;
  sort?: any;
}

// Create DataLoader instances
export function createDataLoaders(db: any): DataLoaders {
  return {
    // User loader with caching
    userLoader: new DataLoader<string, User>(
      async (userIds: readonly string[]) => {
        const users = await db.user.findMany({
          where: { id: { in: userIds as string[] } },
          include: {
            posts: {
              take: 5, // Limit to prevent large payloads
              orderBy: { createdAt: 'desc' },
            },
          },
        });

        // Maintain order of requested IDs
        const userMap = new Map(users.map(user => [user.id, user]));
        return userIds.map(id => userMap.get(id) || null);
      },
      {
        // Cache configuration
        cacheKeyFn: (key) => key,
        batchScheduleFn: (callback) => setTimeout(callback, 10), // Batch for 10ms
        maxBatchSize: 100,
      }
    ),

    // Post loader
    postLoader: new DataLoader<string, Post>(
      async (postIds: readonly string[]) => {
        const posts = await db.post.findMany({
          where: { id: { in: postIds as string[] } },
          include: {
            author: true,
            tags: true,
            comments: {
              take: 10,
              include: { author: true },
              orderBy: { createdAt: 'desc' },
            },
          },
        });

        const postMap = new Map(posts.map(post => [post.id, post]));
        return postIds.map(id => postMap.get(id) || null);
      }
    ),

    // User posts loader with complex keys
    userPostsLoader: new DataLoader<UserPostsKey, PostConnection>(
      async (keys: readonly UserPostsKey[]) => {
        // Group keys by similar filters to optimize queries
        const results = await Promise.all(
          keys.map(async ({ userId, filters, pagination, sort }) => {
            const { limit = 10, offset = 0 } = pagination || {};
            
            const posts = await db.post.findMany({
              where: {
                authorId: userId,
                ...buildPostFilters(filters),
              },
              take: limit + 1,
              skip: offset,
              orderBy: buildSortOrder(sort) || { createdAt: 'desc' },
              include: {
                author: true,
                tags: true,
                comments: { take: 5, include: { author: true } },
              },
            });

            const hasNextPage = posts.length > limit;
            const nodes = hasNextPage ? posts.slice(0, -1) : posts;
            
            const edges = nodes.map(post => ({
              node: post,
              cursor: post.id,
            }));

            const totalCount = await db.post.count({
              where: { authorId: userId, ...buildPostFilters(filters) },
            });

            return {
              edges,
              pageInfo: {
                hasNextPage,
                hasPreviousPage: offset > 0,
                startCursor: edges[0]?.cursor,
                endCursor: edges[edges.length - 1]?.cursor,
              },
              totalCount,
            };
          })
        );

        return results;
      },
      {
        cacheKeyFn: (key) => JSON.stringify(key),
        maxBatchSize: 50,
      }
    ),

    // Post comments loader
    postCommentsLoader: new DataLoader<string, Comment[]>(
      async (postIds: readonly string[]) => {
        const comments = await db.comment.findMany({
          where: { postId: { in: postIds as string[] } },
          include: {
            author: true,
            replies: {
              include: { author: true },
              orderBy: { createdAt: 'asc' },
            },
          },
          orderBy: { createdAt: 'desc' },
        });

        // Group comments by post ID
        const commentsByPost = new Map<string, Comment[]>();
        postIds.forEach(id => commentsByPost.set(id, []));
        
        comments.forEach(comment => {
          const postComments = commentsByPost.get(comment.postId) || [];
          postComments.push(comment);
          commentsByPost.set(comment.postId, postComments);
        });

        return postIds.map(id => commentsByPost.get(id) || []);
      }
    ),

    // Post tags loader
    postTagsLoader: new DataLoader<string, Tag[]>(
      async (postIds: readonly string[]) => {
        const postTags = await db.postTag.findMany({
          where: { postId: { in: postIds as string[] } },
          include: { tag: true },
        });

        const tagsByPost = new Map<string, Tag[]>();
        postIds.forEach(id => tagsByPost.set(id, []));
        
        postTags.forEach(({ postId, tag }) => {
          const tags = tagsByPost.get(postId) || [];
          tags.push(tag);
          tagsByPost.set(postId, tags);
        });

        return postIds.map(id => tagsByPost.get(id) || []);
      }
    ),

    // User followers loader
    userFollowersLoader: new DataLoader<string, User[]>(
      async (userIds: readonly string[]) => {
        const follows = await db.follow.findMany({
          where: { followingId: { in: userIds as string[] } },
          include: { follower: true },
        });

        const followersByUser = new Map<string, User[]>();
        userIds.forEach(id => followersByUser.set(id, []));
        
        follows.forEach(({ followingId, follower }) => {
          const followers = followersByUser.get(followingId) || [];
          followers.push(follower);
          followersByUser.set(followingId, followers);
        });

        return userIds.map(id => followersByUser.get(id) || []);
      }
    ),
  };
}

// DataLoader context management
class DataLoaderContext {
  private loaders: DataLoaders;
  private requestId: string;

  constructor(db: any, requestId: string) {
    this.loaders = createDataLoaders(db);
    this.requestId = requestId;
  }

  // Get loaders for this request
  getLoaders(): DataLoaders {
    return this.loaders;
  }

  // Clear all caches (useful for mutations)
  clearAllCaches(): void {
    Object.values(this.loaders).forEach(loader => {
      loader.clearAll();
    });
  }

  // Clear specific cache
  clearCache(loaderName: keyof DataLoaders, key?: any): void {
    const loader = this.loaders[loaderName];
    if (key) {
      loader.clear(key);
    } else {
      loader.clearAll();
    }
  }

  // Prime cache with data
  primeCache<T>(loaderName: keyof DataLoaders, key: any, value: T): void {
    const loader = this.loaders[loaderName] as DataLoader<any, T>;
    loader.prime(key, value);
  }
}

// Advanced DataLoader patterns
class AdvancedDataLoader<K, V> extends DataLoader<K, V> {
  private metrics = {
    hits: 0,
    misses: 0,
    batchCount: 0,
    totalRequests: 0,
  };

  constructor(
    batchLoadFn: DataLoader.BatchLoadFn<K, V>,
    options?: DataLoader.Options<K, V>
  ) {
    super(
      async (keys) => {
        this.metrics.batchCount++;
        this.metrics.totalRequests += keys.length;
        
        const startTime = Date.now();
        const results = await batchLoadFn(keys);
        const duration = Date.now() - startTime;
        
        console.log(`DataLoader batch completed: ${keys.length} keys in ${duration}ms`);
        return results;
      },
      {
        ...options,
        cacheMap: new Map(), // Custom cache implementation
      }
    );
  }

  async load(key: K): Promise<V> {
    const cached = this.cache?.get(key);
    if (cached) {
      this.metrics.hits++;
    } else {
      this.metrics.misses++;
    }
    
    return super.load(key);
  }

  getMetrics() {
    const hitRate = this.metrics.totalRequests > 0 
      ? this.metrics.hits / this.metrics.totalRequests 
      : 0;
    
    return {
      ...this.metrics,
      hitRate: Math.round(hitRate * 100) / 100,
    };
  }

  resetMetrics(): void {
    this.metrics = {
      hits: 0,
      misses: 0,
      batchCount: 0,
      totalRequests: 0,
    };
  }
}

// DataLoader with Redis caching
import Redis from 'ioredis';

class RedisDataLoader<K, V> extends DataLoader<K, V> {
  private redis: Redis;
  private keyPrefix: string;
  private ttl: number;

  constructor(
    batchLoadFn: DataLoader.BatchLoadFn<K, V>,
    redis: Redis,
    options: {
      keyPrefix: string;
      ttl?: number;
      cacheKeyFn?: (key: K) => string;
    }
  ) {
    const { keyPrefix, ttl = 300, cacheKeyFn } = options;
    
    super(batchLoadFn, {
      cacheKeyFn,
      cache: false, // Disable in-memory cache
    });

    this.redis = redis;
    this.keyPrefix = keyPrefix;
    this.ttl = ttl;
  }

  async load(key: K): Promise<V> {
    const cacheKey = `${this.keyPrefix}:${this.getCacheKey(key)}`;
    
    // Try Redis cache first
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      try {
        return JSON.parse(cached);
      } catch (error) {
        console.warn('Failed to parse cached value:', error);
      }
    }

    // Load from source
    const value = await super.load(key);
    
    // Cache in Redis
    await this.redis.setex(cacheKey, this.ttl, JSON.stringify(value));
    
    return value;
  }

  private getCacheKey(key: K): string {
    if (typeof key === 'string' || typeof key === 'number') {
      return String(key);
    }
    return JSON.stringify(key);
  }

  async clearKey(key: K): Promise<void> {
    const cacheKey = `${this.keyPrefix}:${this.getCacheKey(key)}`;
    await this.redis.del(cacheKey);
  }

  async clearAll(): Promise<void> {
    const pattern = `${this.keyPrefix}:*`;
    const keys = await this.redis.keys(pattern);
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
  }
}
```

## Query Optimization Techniques

### 1. Query Analysis and Performance Monitoring
```typescript
// GraphQL query complexity analysis
import { createComplexityLimitRule } from 'graphql-query-complexity';
import { parse, validate } from 'graphql';

interface QueryComplexityConfig {
  maximumComplexity: number;
  scalarCost: number;
  objectCost: number;
  listFactor: number;
  introspectionCost: number;
}

class QueryComplexityAnalyzer {
  private config: QueryComplexityConfig;

  constructor(config: QueryComplexityConfig) {
    this.config = config;
  }

  // Calculate query complexity
  calculateComplexity(query: string, schema: any): number {
    const document = parse(query);
    
    const complexityResult = validate(
      schema,
      document,
      [
        createComplexityLimitRule(this.config.maximumComplexity, {
          scalarCost: this.config.scalarCost,
          objectCost: this.config.objectCost,
          listFactor: this.config.listFactor,
          introspectionCost: this.config.introspectionCost,
          createError: (max, actual) => {
            return new Error(
              `Query complexity ${actual} exceeds maximum allowed complexity ${max}`
            );
          },
        }),
      ]
    );

    if (complexityResult.length > 0) {
      throw complexityResult[0];
    }

    return this.extractComplexityFromValidation(document);
  }

  // Analyze query patterns
  analyzeQuery(query: string): {
    depth: number;
    fieldCount: number;
    aliasCount: number;
    fragmentCount: number;
    variables: string[];
    operations: string[];
  } {
    const document = parse(query);
    
    let depth = 0;
    let fieldCount = 0;
    let aliasCount = 0;
    let fragmentCount = 0;
    const variables: string[] = [];
    const operations: string[] = [];

    // Traverse AST to analyze query structure
    const analyzeNode = (node: any, currentDepth = 0): void => {
      depth = Math.max(depth, currentDepth);

      switch (node.kind) {
        case 'OperationDefinition':
          operations.push(node.operation);
          if (node.variableDefinitions) {
            variables.push(...node.variableDefinitions.map((v: any) => v.variable.name.value));
          }
          break;
        case 'Field':
          fieldCount++;
          if (node.alias) aliasCount++;
          break;
        case 'FragmentDefinition':
        case 'InlineFragment':
          fragmentCount++;
          break;
      }

      // Recursively analyze child nodes
      Object.values(node).forEach(value => {
        if (Array.isArray(value)) {
          value.forEach(child => {
            if (child && typeof child === 'object' && child.kind) {
              analyzeNode(child, currentDepth + 1);
            }
          });
        } else if (value && typeof value === 'object' && value.kind) {
          analyzeNode(value, currentDepth + 1);
        }
      });
    };

    document.definitions.forEach(def => analyzeNode(def));

    return {
      depth,
      fieldCount,
      aliasCount,
      fragmentCount,
      variables,
      operations,
    };
  }

  private extractComplexityFromValidation(document: any): number {
    // Implementation depends on specific complexity calculation algorithm
    // This is a simplified version
    let complexity = 0;
    
    const calculateNodeComplexity = (node: any): number => {
      let nodeComplexity = this.config.objectCost;
      
      if (node.selectionSet) {
        node.selectionSet.selections.forEach((selection: any) => {
          if (selection.kind === 'Field') {
            nodeComplexity += this.config.scalarCost;
            if (selection.selectionSet) {
              nodeComplexity += calculateNodeComplexity(selection) * this.config.listFactor;
            }
          }
        });
      }
      
      return nodeComplexity;
    };

    document.definitions.forEach((def: any) => {
      if (def.kind === 'OperationDefinition') {
        complexity += calculateNodeComplexity(def);
      }
    });

    return complexity;
  }
}

// Query performance monitoring
class GraphQLPerformanceMonitor {
  private metrics = new Map<string, QueryMetrics>();

  interface QueryMetrics {
    count: number;
    totalDuration: number;
    avgDuration: number;
    maxDuration: number;
    minDuration: number;
    errorCount: number;
    complexity: number;
  }

  recordQuery(
    queryHash: string,
    duration: number,
    complexity: number,
    hasError = false
  ): void {
    const existing = this.metrics.get(queryHash) || {
      count: 0,
      totalDuration: 0,
      avgDuration: 0,
      maxDuration: 0,
      minDuration: Infinity,
      errorCount: 0,
      complexity: 0,
    };

    existing.count++;
    existing.totalDuration += duration;
    existing.avgDuration = existing.totalDuration / existing.count;
    existing.maxDuration = Math.max(existing.maxDuration, duration);
    existing.minDuration = Math.min(existing.minDuration, duration);
    existing.complexity = complexity;
    
    if (hasError) {
      existing.errorCount++;
    }

    this.metrics.set(queryHash, existing);

    // Alert on slow queries
    if (duration > 1000) { // 1 second threshold
      console.warn(`Slow GraphQL query detected: ${queryHash} (${duration}ms)`);
    }
  }

  getMetrics(queryHash?: string): QueryMetrics | Map<string, QueryMetrics> {
    if (queryHash) {
      return this.metrics.get(queryHash);
    }
    return this.metrics;
  }

  getSlowestQueries(limit = 10): Array<{ hash: string; metrics: QueryMetrics }> {
    return Array.from(this.metrics.entries())
      .sort(([, a], [, b]) => b.avgDuration - a.avgDuration)
      .slice(0, limit)
      .map(([hash, metrics]) => ({ hash, metrics }));
  }

  getMostComplexQueries(limit = 10): Array<{ hash: string; metrics: QueryMetrics }> {
    return Array.from(this.metrics.entries())
      .sort(([, a], [, b]) => b.complexity - a.complexity)
      .slice(0, limit)
      .map(([hash, metrics]) => ({ hash, metrics }));
  }

  resetMetrics(): void {
    this.metrics.clear();
  }
}

// Query optimization middleware
export function createQueryOptimizationMiddleware(
  complexityAnalyzer: QueryComplexityAnalyzer,
  performanceMonitor: GraphQLPerformanceMonitor
) {
  return (req: any, res: any, next: any) => {
    const startTime = Date.now();
    const query = req.body.query;
    
    if (!query) {
      return next();
    }

    try {
      // Generate query hash for caching and monitoring
      const queryHash = require('crypto')
        .createHash('sha256')
        .update(query)
        .digest('hex')
        .substring(0, 16);

      // Analyze query complexity
      const complexity = complexityAnalyzer.calculateComplexity(query, req.schema);
      
      // Add metadata to request
      req.graphql = {
        queryHash,
        complexity,
        startTime,
      };

      // Override res.end to capture metrics
      const originalEnd = res.end;
      res.end = function(...args: any[]) {
        const duration = Date.now() - startTime;
        const hasError = res.statusCode >= 400;
        
        performanceMonitor.recordQuery(queryHash, duration, complexity, hasError);
        
        originalEnd.apply(this, args);
      };

      next();
    } catch (error) {
      // Query complexity exceeded or other validation error
      return res.status(400).json({
        error: 'Query validation failed',
        message: error.message,
      });
    }
  };
}
```

### 2. Advanced Query Optimization Patterns
```typescript
// Query batching and deduplication
class QueryBatcher {
  private pendingQueries = new Map<string, {
    resolve: Function;
    reject: Function;
    query: string;
    variables?: any;
  }[]>();
  private batchTimeout: NodeJS.Timeout | null = null;
  private batchSize = 10;
  private batchDelay = 10; // ms

  constructor(
    private executeQuery: (queries: Array<{ query: string; variables?: any }>) => Promise<any[]>,
    options?: { batchSize?: number; batchDelay?: number }
  ) {
    if (options?.batchSize) this.batchSize = options.batchSize;
    if (options?.batchDelay) this.batchDelay = options.batchDelay;
  }

  async query(query: string, variables?: any): Promise<any> {
    return new Promise((resolve, reject) => {
      const queryKey = this.getQueryKey(query, variables);
      
      if (!this.pendingQueries.has(queryKey)) {
        this.pendingQueries.set(queryKey, []);
      }
      
      this.pendingQueries.get(queryKey)!.push({
        resolve,
        reject,
        query,
        variables,
      });

      this.scheduleBatch();
    });
  }

  private getQueryKey(query: string, variables?: any): string {
    const hash = require('crypto').createHash('sha256');
    hash.update(query);
    if (variables) {
      hash.update(JSON.stringify(variables));
    }
    return hash.digest('hex');
  }

  private scheduleBatch(): void {
    if (this.batchTimeout) return;

    this.batchTimeout = setTimeout(() => {
      this.executeBatch();
    }, this.batchDelay);

    // Execute immediately if batch size reached
    const totalPending = Array.from(this.pendingQueries.values())
      .reduce((sum, pending) => sum + pending.length, 0);
    
    if (totalPending >= this.batchSize) {
      clearTimeout(this.batchTimeout);
      this.executeBatch();
    }
  }

  private async executeBatch(): Promise<void> {
    this.batchTimeout = null;
    
    if (this.pendingQueries.size === 0) return;

    // Group pending queries
    const batchQueries: Array<{ query: string; variables?: any }> = [];
    const pendingPromises: Array<{
      resolve: Function;
      reject: Function;
      index: number;
    }> = [];

    let index = 0;
    for (const [queryKey, pending] of this.pendingQueries) {
      // Use the first query/variables for deduplication
      const { query, variables } = pending[0];
      batchQueries.push({ query, variables });
      
      // All pending queries with same key get same result
      pending.forEach(({ resolve, reject }) => {
        pendingPromises.push({ resolve, reject, index });
      });
      
      index++;
    }

    this.pendingQueries.clear();

    try {
      const results = await this.executeQuery(batchQueries);
      
      // Resolve all pending promises
      pendingPromises.forEach(({ resolve, index }) => {
        resolve(results[index]);
      });
    } catch (error) {
      // Reject all pending promises
      pendingPromises.forEach(({ reject }) => {
        reject(error);
      });
    }
  }
}

// Query result caching with invalidation
interface CacheConfig {
  ttl: number; // Time to live in seconds
  maxSize: number; // Maximum number of cached queries
  invalidationPatterns: string[]; // Patterns for cache invalidation
}

class GraphQLQueryCache {
  private cache = new Map<string, {
    data: any;
    timestamp: number;
    ttl: number;
    tags: string[];
  }>();
  private config: CacheConfig;

  constructor(config: CacheConfig) {
    this.config = config;
    
    // Periodic cleanup of expired entries
    setInterval(() => {
      this.cleanup();
    }, 60000); // Every minute
  }

  // Generate cache key from query and variables
  private getCacheKey(query: string, variables?: any): string {
    const hash = require('crypto').createHash('sha256');
    hash.update(query);
    if (variables) {
      hash.update(JSON.stringify(variables));
    }
    return hash.digest('hex');
  }

  // Extract cache tags from query for invalidation
  private extractTags(query: string): string[] {
    const tags: string[] = [];
    
    // Extract type names from query
    const typeMatches = query.match(/(?:query|mutation|subscription)\s+\w*\s*(?:\([^)]*\))?\s*{([^}]+)}/);
    if (typeMatches) {
      const fields = typeMatches[1];
      const fieldMatches = fields.match(/\b([a-zA-Z_][a-zA-Z0-9_]*)\s*(?:\(|{)/g);
      if (fieldMatches) {
        tags.push(...fieldMatches.map(match => match.replace(/\s*[({].*/, '')));
      }
    }

    return tags;
  }

  // Cache query result
  set(
    query: string,
    variables: any,
    data: any,
    customTTL?: number
  ): void {
    const key = this.getCacheKey(query, variables);
    const ttl = customTTL || this.config.ttl;
    const tags = this.extractTags(query);

    // Enforce maximum cache size
    if (this.cache.size >= this.config.maxSize) {
      this.evictOldest();
    }

    this.cache.set(key, {
      data,
      timestamp: Date.now(),
      ttl: ttl * 1000, // Convert to milliseconds
      tags,
    });
  }

  // Get cached query result
  get(query: string, variables?: any): any | null {
    const key = this.getCacheKey(query, variables);
    const cached = this.cache.get(key);

    if (!cached) return null;

    // Check expiration
    if (Date.now() - cached.timestamp > cached.ttl) {
      this.cache.delete(key);
      return null;
    }

    return cached.data;
  }

  // Invalidate cache by patterns
  invalidate(pattern: string | RegExp): void {
    const keysToDelete: string[] = [];

    for (const [key, cached] of this.cache) {
      // Check if any tag matches the pattern
      const shouldInvalidate = cached.tags.some(tag => {
        if (typeof pattern === 'string') {
          return tag.includes(pattern);
        }
        return pattern.test(tag);
      });

      if (shouldInvalidate) {
        keysToDelete.push(key);
      }
    }

    keysToDelete.forEach(key => this.cache.delete(key));
    
    console.log(`Invalidated ${keysToDelete.length} cache entries for pattern: ${pattern}`);
  }

  // Invalidate all cache entries
  clear(): void {
    this.cache.clear();
  }

  // Clean up expired entries
  private cleanup(): void {
    const now = Date.now();
    const expiredKeys: string[] = [];

    for (const [key, cached] of this.cache) {
      if (now - cached.timestamp > cached.ttl) {
        expiredKeys.push(key);
      }
    }

    expiredKeys.forEach(key => this.cache.delete(key));
    
    if (expiredKeys.length > 0) {
      console.log(`Cleaned up ${expiredKeys.length} expired cache entries`);
    }
  }

  // Evict oldest entry when cache is full
  private evictOldest(): void {
    let oldestKey: string | null = null;
    let oldestTimestamp = Infinity;

    for (const [key, cached] of this.cache) {
      if (cached.timestamp < oldestTimestamp) {
        oldestTimestamp = cached.timestamp;
        oldestKey = key;
      }
    }

    if (oldestKey) {
      this.cache.delete(oldestKey);
    }
  }

  // Get cache statistics
  getStats(): {
    size: number;
    maxSize: number;
    hitRate: number;
    oldestEntry: number;
    newestEntry: number;
  } {
    let oldestTimestamp = Infinity;
    let newestTimestamp = 0;

    for (const cached of this.cache.values()) {
      oldestTimestamp = Math.min(oldestTimestamp, cached.timestamp);
      newestTimestamp = Math.max(newestTimestamp, cached.timestamp);
    }

    return {
      size: this.cache.size,
      maxSize: this.config.maxSize,
      hitRate: 0, // Would need to track hits/misses
      oldestEntry: oldestTimestamp === Infinity ? 0 : oldestTimestamp,
      newestEntry: newestTimestamp,
    };
  }
}

// Automatic persisted queries (APQ)
class AutomaticPersistedQueries {
  private queryStore = new Map<string, string>();
  private redis?: any; // Redis client for distributed storage

  constructor(redis?: any) {
    this.redis = redis;
  }

  // Generate query hash
  private generateHash(query: string): string {
    return require('crypto')
      .createHash('sha256')
      .update(query)
      .digest('hex');
  }

  // Store query with hash
  async storeQuery(query: string): Promise<string> {
    const hash = this.generateHash(query);
    
    // Store in memory
    this.queryStore.set(hash, query);
    
    // Store in Redis if available
    if (this.redis) {
      await this.redis.setex(`apq:${hash}`, 3600, query); // 1 hour TTL
    }

    return hash;
  }

  // Retrieve query by hash
  async getQuery(hash: string): Promise<string | null> {
    // Try memory first
    let query = this.queryStore.get(hash);
    
    if (!query && this.redis) {
      // Try Redis
      query = await this.redis.get(`apq:${hash}`);
      if (query) {
        // Store back in memory for faster access
        this.queryStore.set(hash, query);
      }
    }

    return query || null;
  }

  // Process APQ request
  async processRequest(body: any): Promise<{
    query?: string;
    hash?: string;
    shouldStore?: boolean;
  }> {
    const { query, extensions } = body;
    const persistedQuery = extensions?.persistedQuery;

    if (!persistedQuery) {
      // Regular query, not APQ
      return { query };
    }

    const { sha256Hash, version } = persistedQuery;

    if (version !== 1) {
      throw new Error('Unsupported persisted query version');
    }

    if (query) {
      // Client provided query with hash - store it
      const expectedHash = this.generateHash(query);
      if (expectedHash !== sha256Hash) {
        throw new Error('Query hash mismatch');
      }
      
      await this.storeQuery(query);
      return { query, hash: sha256Hash };
    } else {
      // Client provided only hash - retrieve query
      const storedQuery = await this.getQuery(sha256Hash);
      
      if (!storedQuery) {
        throw new Error('PersistedQueryNotFound');
      }
      
      return { query: storedQuery, hash: sha256Hash };
    }
  }

  // Get APQ statistics
  getStats(): {
    totalQueries: number;
    memoryUsage: number;
    cacheHitRate: number;
  } {
    return {
      totalQueries: this.queryStore.size,
      memoryUsage: JSON.stringify(Array.from(this.queryStore.entries())).length,
      cacheHitRate: 0, // Would need to track hits/misses
    };
  }
}
```

## Interview-Ready Summary

**GraphQL Query Optimization requires:**

1. **Schema Design** - Efficient type definitions, proper field resolvers, connection patterns, input validation
2. **DataLoader Implementation** - N+1 query prevention, request batching, caching strategies, memory management
3. **Query Analysis** - Complexity calculation, depth limiting, performance monitoring, query pattern analysis
4. **Caching Strategies** - Result caching, automatic persisted queries, Redis integration, cache invalidation

**Key optimization techniques:**
- **Batching** - DataLoader for efficient data fetching, query batching, request deduplication
- **Caching** - Query result caching, APQ for bandwidth reduction, smart invalidation
- **Monitoring** - Performance metrics, complexity analysis, slow query detection
- **Security** - Query depth limiting, complexity analysis, rate limiting

**Common performance issues:** N+1 queries, overfetching, complex nested queries, lack of pagination, missing caching.

**Best practices:** Use DataLoader consistently, implement query complexity analysis, cache at multiple levels, monitor query performance, use APQ for mobile apps, implement proper pagination with connections.