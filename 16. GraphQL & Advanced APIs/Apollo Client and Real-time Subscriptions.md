# Apollo Client and Real-time Subscriptions

Apollo Client is a comprehensive state management library for JavaScript that enables you to manage both local and remote data with GraphQL. This guide covers advanced Apollo Client patterns, real-time subscriptions, and integration with modern React applications.

## Apollo Client Setup and Configuration

### 1. Advanced Apollo Client Configuration
```typescript
// Apollo Client setup with advanced features
import {
  ApolloClient,
  InMemoryCache,
  createHttpLink,
  from,
  split,
} from '@apollo/client';
import { setContext } from '@apollo/client/link/context';
import { onError } from '@apollo/client/link/error';
import { RetryLink } from '@apollo/client/link/retry';
import { createPersistedQueryLink } from '@apollo/client/link/persisted-queries';
import { GraphQLWsLink } from '@apollo/client/link/subscriptions';
import { getMainDefinition } from '@apollo/client/utilities';
import { createClient } from 'graphql-ws';
import { sha256 } from 'crypto-hash';

// HTTP Link for queries and mutations
const httpLink = createHttpLink({
  uri: process.env.NEXT_PUBLIC_GRAPHQL_ENDPOINT || 'http://localhost:4000/graphql',
  credentials: 'include', // Include cookies for authentication
});

// WebSocket Link for subscriptions
const wsLink = new GraphQLWsLink(
  createClient({
    url: process.env.NEXT_PUBLIC_GRAPHQL_WS_ENDPOINT || 'ws://localhost:4000/graphql',
    connectionParams: () => ({
      authToken: localStorage.getItem('authToken'),
    }),
    retryAttempts: 5,
    shouldRetry: (errOrCloseEvent) => {
      // Retry on connection errors but not on auth failures
      return errOrCloseEvent instanceof CloseEvent && errOrCloseEvent.code !== 4401;
    },
  })
);

// Auth link to add authorization headers
const authLink = setContext((_, { headers }) => {
  const token = localStorage.getItem('authToken');
  
  return {
    headers: {
      ...headers,
      authorization: token ? `Bearer ${token}` : '',
      'x-client-name': 'web-app',
      'x-client-version': process.env.NEXT_PUBLIC_APP_VERSION || '1.0.0',
    },
  };
});

// Error handling link
const errorLink = onError(({ graphQLErrors, networkError, operation, forward }) => {
  if (graphQLErrors) {
    graphQLErrors.forEach(({ message, locations, path, extensions }) => {
      console.error(
        `GraphQL error: Message: ${message}, Location: ${locations}, Path: ${path}`
      );

      // Handle specific error types
      if (extensions?.code === 'UNAUTHENTICATED') {
        // Clear auth token and redirect to login
        localStorage.removeItem('authToken');
        window.location.href = '/login';
      } else if (extensions?.code === 'FORBIDDEN') {
        // Handle authorization errors
        console.warn('Insufficient permissions for operation:', operation.operationName);
      }
    });
  }

  if (networkError) {
    console.error(`Network error: ${networkError}`);
    
    // Handle network errors
    if (networkError.statusCode === 401) {
      localStorage.removeItem('authToken');
      window.location.href = '/login';
    }
  }
});

// Retry link for failed requests
const retryLink = new RetryLink({
  delay: {
    initial: 300,
    max: Infinity,
    jitter: true,
  },
  attempts: {
    max: 3,
    retryIf: (error, _operation) => {
      // Retry on network errors but not on GraphQL errors
      return !!error && !error.result;
    },
  },
});

// Automatic Persisted Queries link
const persistedQueriesLink = createPersistedQueryLink({
  sha256,
  useGETForHashedQueries: true,
});

// Split link to route operations to appropriate transport
const splitLink = split(
  ({ query }) => {
    const definition = getMainDefinition(query);
    return (
      definition.kind === 'OperationDefinition' &&
      definition.operation === 'subscription'
    );
  },
  wsLink,
  from([authLink, errorLink, retryLink, persistedQueriesLink, httpLink])
);

// Advanced cache configuration
const cache = new InMemoryCache({
  typePolicies: {
    Query: {
      fields: {
        posts: {
          // Implement cursor-based pagination
          keyArgs: ['filters'],
          merge(existing = { edges: [], pageInfo: {} }, incoming) {
            return {
              ...incoming,
              edges: [...existing.edges, ...incoming.edges],
              pageInfo: incoming.pageInfo,
            };
          },
        },
        user: {
          // Cache user by ID
          read(_, { args, toReference }) {
            return toReference({
              __typename: 'User',
              id: args?.id,
            });
          },
        },
      },
    },
    User: {
      fields: {
        posts: {
          merge(existing = [], incoming) {
            return [...existing, ...incoming];
          },
        },
      },
    },
    Post: {
      fields: {
        comments: {
          merge(existing = [], incoming) {
            return [...existing, ...incoming];
          },
        },
      },
    },
  },
  possibleTypes: {
    SearchResult: ['User', 'Post', 'Tag'],
    Node: ['User', 'Post', 'Comment', 'Tag'],
  },
});

// Create Apollo Client instance
export const client = new ApolloClient({
  link: splitLink,
  cache,
  defaultOptions: {
    watchQuery: {
      errorPolicy: 'all',
      notifyOnNetworkStatusChange: true,
    },
    query: {
      errorPolicy: 'all',
    },
    mutate: {
      errorPolicy: 'all',
    },
  },
  connectToDevTools: process.env.NODE_ENV === 'development',
});

// Apollo Client provider with error boundary
import React, { Component, ReactNode } from 'react';
import { ApolloProvider } from '@apollo/client';

interface ApolloErrorBoundaryState {
  hasError: boolean;
  error?: Error;
}

class ApolloErrorBoundary extends Component<
  { children: ReactNode },
  ApolloErrorBoundaryState
> {
  constructor(props: { children: ReactNode }) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error): ApolloErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: any) {
    console.error('Apollo Error Boundary caught an error:', error, errorInfo);
    
    // Send error to monitoring service
    if (process.env.NODE_ENV === 'production') {
      // reportError(error, errorInfo);
    }
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-fallback">
          <h2>Something went wrong with the GraphQL client.</h2>
          <button onClick={() => window.location.reload()}>
            Reload Page
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}

export const ApolloAppProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
  return (
    <ApolloErrorBoundary>
      <ApolloProvider client={client}>
        {children}
      </ApolloProvider>
    </ApolloErrorBoundary>
  );
};

// Environment-specific client configuration
interface ApolloConfig {
  httpUri: string;
  wsUri: string;
  enableSubscriptions: boolean;
  enablePersistedQueries: boolean;
  retryAttempts: number;
  cacheSize: number;
}

const apolloConfigs: Record<string, ApolloConfig> = {
  development: {
    httpUri: 'http://localhost:4000/graphql',
    wsUri: 'ws://localhost:4000/graphql',
    enableSubscriptions: true,
    enablePersistedQueries: false,
    retryAttempts: 3,
    cacheSize: 50, // MB
  },
  staging: {
    httpUri: 'https://api-staging.example.com/graphql',
    wsUri: 'wss://api-staging.example.com/graphql',
    enableSubscriptions: true,
    enablePersistedQueries: true,
    retryAttempts: 5,
    cacheSize: 100,
  },
  production: {
    httpUri: 'https://api.example.com/graphql',
    wsUri: 'wss://api.example.com/graphql',
    enableSubscriptions: true,
    enablePersistedQueries: true,
    retryAttempts: 5,
    cacheSize: 200,
  },
};

export function createApolloClient(environment: string = 'development'): ApolloClient<any> {
  const config = apolloConfigs[environment];
  
  // Implementation would use config to create appropriate client
  // This is a simplified example
  return client;
}
```

### 2. Advanced Query and Mutation Patterns
```typescript
// Advanced Apollo Client hooks and patterns
import {
  useQuery,
  useMutation,
  useSubscription,
  useLazyQuery,
  gql,
  DocumentNode,
  TypedDocumentNode,
  QueryHookOptions,
  MutationHookOptions,
  SubscriptionHookOptions,
} from '@apollo/client';
import { useCallback, useEffect, useMemo, useState } from 'react';

// Type-safe GraphQL operations
const GET_POSTS = gql`
  query GetPosts(
    $filters: PostFilters
    $pagination: PaginationInput
    $sort: SortInput
  ) {
    posts(filters: $filters, pagination: $pagination, sort: $sort) {
      edges {
        node {
          id
          title
          excerpt
          author {
            id
            name
            avatar
          }
          tags {
            id
            name
            color
          }
          publishedAt
          likes
          views
        }
        cursor
      }
      pageInfo {
        hasNextPage
        hasPreviousPage
        startCursor
        endCursor
      }
      totalCount
    }
  }
`;

const CREATE_POST = gql`
  mutation CreatePost($input: CreatePostInput!) {
    createPost(input: $input) {
      id
      title
      content
      status
      author {
        id
        name
      }
      createdAt
    }
  }
`;

const POST_UPDATES = gql`
  subscription PostUpdates {
    postAdded {
      id
      title
      author {
        id
        name
      }
    }
    postUpdated {
      id
      title
      likes
      views
    }
  }
`;

// TypeScript interfaces for GraphQL types
interface Post {
  id: string;
  title: string;
  excerpt?: string;
  content?: string;
  author: User;
  tags: Tag[];
  publishedAt?: string;
  likes: number;
  views: number;
}

interface User {
  id: string;
  name: string;
  avatar?: string;
}

interface Tag {
  id: string;
  name: string;
  color?: string;
}

interface PostsConnection {
  edges: Array<{
    node: Post;
    cursor: string;
  }>;
  pageInfo: {
    hasNextPage: boolean;
    hasPreviousPage: boolean;
    startCursor?: string;
    endCursor?: string;
  };
  totalCount: number;
}

// Advanced query hook with error handling and caching
interface UsePostsOptions {
  filters?: any;
  sort?: any;
  pageSize?: number;
  fetchPolicy?: 'cache-first' | 'cache-and-network' | 'network-only';
  errorPolicy?: 'none' | 'ignore' | 'all';
}

export function usePosts(options: UsePostsOptions = {}) {
  const {
    filters,
    sort,
    pageSize = 10,
    fetchPolicy = 'cache-first',
    errorPolicy = 'all',
  } = options;

  const [cursor, setCursor] = useState<string | null>(null);
  const [allPosts, setAllPosts] = useState<Post[]>([]);

  const { data, loading, error, fetchMore, refetch, networkStatus } = useQuery(
    GET_POSTS,
    {
      variables: {
        filters,
        pagination: { limit: pageSize, cursor },
        sort,
      },
      fetchPolicy,
      errorPolicy,
      notifyOnNetworkStatusChange: true,
    }
  );

  // Memoized posts data
  const posts = useMemo(() => {
    if (!data?.posts) return [];
    return data.posts.edges.map(edge => edge.node);
  }, [data]);

  const pageInfo = data?.posts?.pageInfo;
  const totalCount = data?.posts?.totalCount || 0;

  // Load more function for infinite scrolling
  const loadMore = useCallback(async () => {
    if (!pageInfo?.hasNextPage || loading) return;

    try {
      const result = await fetchMore({
        variables: {
          pagination: {
            limit: pageSize,
            cursor: pageInfo.endCursor,
          },
        },
      });

      // Update accumulated posts
      if (result.data?.posts) {
        const newPosts = result.data.posts.edges.map(edge => edge.node);
        setAllPosts(prev => [...prev, ...newPosts]);
      }
    } catch (error) {
      console.error('Failed to load more posts:', error);
    }
  }, [pageInfo, loading, fetchMore, pageSize]);

  // Refresh function
  const refresh = useCallback(async () => {
    setAllPosts([]);
    setCursor(null);
    try {
      await refetch({
        filters,
        pagination: { limit: pageSize, cursor: null },
        sort,
      });
    } catch (error) {
      console.error('Failed to refresh posts:', error);
    }
  }, [refetch, filters, sort, pageSize]);

  return {
    posts,
    allPosts,
    loading,
    error,
    networkStatus,
    pageInfo,
    totalCount,
    loadMore,
    refresh,
    hasMore: pageInfo?.hasNextPage || false,
  };
}

// Advanced mutation hook with optimistic updates
interface UseCreatePostOptions {
  onCompleted?: (data: any) => void;
  onError?: (error: any) => void;
  optimisticResponse?: boolean;
}

export function useCreatePost(options: UseCreatePostOptions = {}) {
  const { onCompleted, onError, optimisticResponse = true } = options;

  const [createPost, { loading, error, data }] = useMutation(CREATE_POST, {
    onCompleted,
    onError,
    
    // Optimistic response for immediate UI updates
    optimisticResponse: optimisticResponse
      ? (variables) => ({
          createPost: {
            __typename: 'Post',
            id: `temp-${Date.now()}`,
            title: variables.input.title,
            content: variables.input.content,
            status: 'DRAFT',
            author: {
              __typename: 'User',
              id: 'current-user', // Would be actual user ID
              name: 'Current User',
            },
            createdAt: new Date().toISOString(),
          },
        })
      : undefined,

    // Update cache after mutation
    update: (cache, { data }) => {
      if (!data?.createPost) return;

      // Add new post to posts query cache
      const existingPosts = cache.readQuery({
        query: GET_POSTS,
        variables: { pagination: { limit: 10 } },
      });

      if (existingPosts) {
        cache.writeQuery({
          query: GET_POSTS,
          variables: { pagination: { limit: 10 } },
          data: {
            posts: {
              ...existingPosts.posts,
              edges: [
                {
                  __typename: 'PostEdge',
                  node: data.createPost,
                  cursor: data.createPost.id,
                },
                ...existingPosts.posts.edges,
              ],
              totalCount: existingPosts.posts.totalCount + 1,
            },
          },
        });
      }
    },

    // Error handling
    errorPolicy: 'all',
  });

  const createPostWithValidation = useCallback(
    async (input: any) => {
      try {
        // Client-side validation
        if (!input.title?.trim()) {
          throw new Error('Title is required');
        }
        if (!input.content?.trim()) {
          throw new Error('Content is required');
        }

        const result = await createPost({
          variables: { input },
        });

        return result.data?.createPost;
      } catch (error) {
        console.error('Failed to create post:', error);
        throw error;
      }
    },
    [createPost]
  );

  return {
    createPost: createPostWithValidation,
    loading,
    error,
    data: data?.createPost,
  };
}

// Advanced lazy query hook
export function useSearchPosts() {
  const [searchPosts, { data, loading, error, called }] = useLazyQuery(
    gql`
      query SearchPosts($query: String!, $pagination: PaginationInput) {
        search(query: $query, type: POST, pagination: $pagination) {
          ... on Post {
            id
            title
            excerpt
            author {
              id
              name
            }
            publishedAt
          }
        }
      }
    `,
    {
      fetchPolicy: 'cache-and-network',
      errorPolicy: 'all',
    }
  );

  const search = useCallback(
    async (query: string, options?: { pageSize?: number; cursor?: string }) => {
      if (!query.trim()) return;

      try {
        await searchPosts({
          variables: {
            query: query.trim(),
            pagination: {
              limit: options?.pageSize || 20,
              cursor: options?.cursor,
            },
          },
        });
      } catch (error) {
        console.error('Search failed:', error);
      }
    },
    [searchPosts]
  );

  const results = useMemo(() => {
    if (!data?.search) return [];
    return data.search.filter((result: any) => result.__typename === 'Post');
  }, [data]);

  return {
    search,
    results,
    loading,
    error,
    called,
  };
}

// Custom hook for polling queries
export function usePollingQuery<T = any>(
  query: DocumentNode,
  options: QueryHookOptions & { pollInterval: number }
) {
  const [isPolling, setIsPolling] = useState(false);
  
  const queryResult = useQuery<T>(query, {
    ...options,
    pollInterval: isPolling ? options.pollInterval : 0,
  });

  const startPolling = useCallback(() => {
    setIsPolling(true);
  }, []);

  const stopPolling = useCallback(() => {
    setIsPolling(false);
    queryResult.stopPolling?.();
  }, [queryResult]);

  // Auto-stop polling when component unmounts
  useEffect(() => {
    return () => {
      stopPolling();
    };
  }, [stopPolling]);

  return {
    ...queryResult,
    isPolling,
    startPolling,
    stopPolling,
  };
}
```

## Real-time Subscriptions

### 1. Subscription Implementation Patterns
```typescript
// Real-time subscription hooks and components
import {
  useSubscription,
  gql,
  useApolloClient,
  SubscriptionHookOptions,
} from '@apollo/client';
import { useEffect, useCallback, useRef, useState } from 'react';

// Subscription definitions
const POST_UPDATES_SUBSCRIPTION = gql`
  subscription PostUpdates {
    postAdded {
      id
      title
      excerpt
      author {
        id
        name
        avatar
      }
      publishedAt
    }
    postUpdated {
      id
      title
      likes
      views
    }
  }
`;

const COMMENT_UPDATES_SUBSCRIPTION = gql`
  subscription CommentUpdates($postId: ID!) {
    commentAdded(postId: $postId) {
      id
      content
      author {
        id
        name
        avatar
      }
      createdAt
    }
  }
`;

const USER_PRESENCE_SUBSCRIPTION = gql`
  subscription UserPresence {
    userOnline {
      id
      name
      lastSeen
    }
    userOffline {
      id
      name
      lastSeen
    }
  }
`;

// Advanced subscription hook with reconnection logic
interface UseSubscriptionWithReconnectOptions<T>
  extends SubscriptionHookOptions<T> {
  reconnectAttempts?: number;
  reconnectInterval?: number;
  onReconnect?: () => void;
  onConnectionLost?: () => void;
}

export function useSubscriptionWithReconnect<T = any>(
  subscription: DocumentNode,
  options: UseSubscriptionWithReconnectOptions<T> = {}
) {
  const {
    reconnectAttempts = 5,
    reconnectInterval = 2000,
    onReconnect,
    onConnectionLost,
    ...subscriptionOptions
  } = options;

  const [connectionStatus, setConnectionStatus] = useState<
    'connected' | 'connecting' | 'disconnected' | 'error'
  >('connecting');
  const [reconnectCount, setReconnectCount] = useState(0);
  const reconnectTimeoutRef = useRef<NodeJS.Timeout>();
  const client = useApolloClient();

  const subscriptionResult = useSubscription<T>(subscription, {
    ...subscriptionOptions,
    onSubscriptionComplete: () => {
      setConnectionStatus('disconnected');
      onConnectionLost?.();
      
      // Attempt reconnection
      if (reconnectCount < reconnectAttempts) {
        reconnectTimeoutRef.current = setTimeout(() => {
          setReconnectCount(prev => prev + 1);
          setConnectionStatus('connecting');
          // Trigger subscription restart by changing a variable
        }, reconnectInterval * Math.pow(2, reconnectCount)); // Exponential backoff
      }
    },
    onSubscriptionData: (data) => {
      if (connectionStatus !== 'connected') {
        setConnectionStatus('connected');
        setReconnectCount(0);
        onReconnect?.();
      }
      subscriptionOptions.onSubscriptionData?.(data);
    },
  });

  // Cleanup timeout on unmount
  useEffect(() => {
    return () => {
      if (reconnectTimeoutRef.current) {
        clearTimeout(reconnectTimeoutRef.current);
      }
    };
  }, []);

  // Manual reconnect function
  const reconnect = useCallback(() => {
    setConnectionStatus('connecting');
    setReconnectCount(0);
    // Force subscription restart
    client.resetStore();
  }, [client]);

  return {
    ...subscriptionResult,
    connectionStatus,
    reconnectCount,
    reconnect,
  };
}

// Post updates subscription hook
export function usePostUpdates() {
  const [recentUpdates, setRecentUpdates] = useState<any[]>([]);
  const [notificationCount, setNotificationCount] = useState(0);

  const { data, error, connectionStatus } = useSubscriptionWithReconnect(
    POST_UPDATES_SUBSCRIPTION,
    {
      onSubscriptionData: ({ subscriptionData }) => {
        if (!subscriptionData.data) return;

        const { postAdded, postUpdated } = subscriptionData.data;
        
        if (postAdded) {
          setRecentUpdates(prev => [
            { type: 'added', post: postAdded, timestamp: Date.now() },
            ...prev.slice(0, 9), // Keep only 10 recent updates
          ]);
          setNotificationCount(prev => prev + 1);
        }

        if (postUpdated) {
          setRecentUpdates(prev => [
            { type: 'updated', post: postUpdated, timestamp: Date.now() },
            ...prev.slice(0, 9),
          ]);
        }
      },
      onReconnect: () => {
        console.log('Post updates subscription reconnected');
      },
      onConnectionLost: () => {
        console.warn('Post updates subscription connection lost');
      },
    }
  );

  const clearNotifications = useCallback(() => {
    setNotificationCount(0);
  }, []);

  const clearRecentUpdates = useCallback(() => {
    setRecentUpdates([]);
  }, []);

  return {
    recentUpdates,
    notificationCount,
    connectionStatus,
    error,
    clearNotifications,
    clearRecentUpdates,
  };
}

// Comments subscription for specific post
export function usePostComments(postId: string) {
  const [comments, setComments] = useState<any[]>([]);

  const { data, loading, error } = useSubscription(
    COMMENT_UPDATES_SUBSCRIPTION,
    {
      variables: { postId },
      onSubscriptionData: ({ subscriptionData }) => {
        if (!subscriptionData.data?.commentAdded) return;

        const newComment = subscriptionData.data.commentAdded;
        setComments(prev => [...prev, newComment]);

        // Show toast notification for new comments
        if (typeof window !== 'undefined' && 'Notification' in window) {
          if (Notification.permission === 'granted') {
            new Notification('New Comment', {
              body: `${newComment.author.name}: ${newComment.content.substring(0, 50)}...`,
              icon: newComment.author.avatar,
            });
          }
        }
      },
      skip: !postId,
    }
  );

  return {
    comments,
    loading,
    error,
  };
}

// User presence tracking
export function useUserPresence() {
  const [onlineUsers, setOnlineUsers] = useState<Map<string, any>>(new Map());

  const { data, error } = useSubscription(USER_PRESENCE_SUBSCRIPTION, {
    onSubscriptionData: ({ subscriptionData }) => {
      if (!subscriptionData.data) return;

      const { userOnline, userOffline } = subscriptionData.data;

      if (userOnline) {
        setOnlineUsers(prev => new Map(prev.set(userOnline.id, userOnline)));
      }

      if (userOffline) {
        setOnlineUsers(prev => {
          const updated = new Map(prev);
          updated.delete(userOffline.id);
          return updated;
        });
      }
    },
  });

  const isUserOnline = useCallback(
    (userId: string) => onlineUsers.has(userId),
    [onlineUsers]
  );

  const getOnlineCount = useCallback(
    () => onlineUsers.size,
    [onlineUsers]
  );

  return {
    onlineUsers: Array.from(onlineUsers.values()),
    isUserOnline,
    getOnlineCount,
    error,
  };
}
```

### 2. Real-time UI Components
```typescript
// Real-time components using subscriptions
import React, { useState, useEffect, useCallback } from 'react';
import { motion, AnimatePresence } from 'framer-motion';

// Real-time notifications component
interface NotificationProps {
  id: string;
  type: 'info' | 'success' | 'warning' | 'error';
  title: string;
  message: string;
  duration?: number;
  onDismiss: (id: string) => void;
}

const Notification: React.FC<NotificationProps> = ({
  id,
  type,
  title,
  message,
  duration = 5000,
  onDismiss,
}) => {
  useEffect(() => {
    const timer = setTimeout(() => {
      onDismiss(id);
    }, duration);

    return () => clearTimeout(timer);
  }, [id, duration, onDismiss]);

  const typeStyles = {
    info: 'bg-blue-50 border-blue-200 text-blue-800',
    success: 'bg-green-50 border-green-200 text-green-800',
    warning: 'bg-yellow-50 border-yellow-200 text-yellow-800',
    error: 'bg-red-50 border-red-200 text-red-800',
  };

  return (
    <motion.div
      initial={{ opacity: 0, y: -50, scale: 0.3 }}
      animate={{ opacity: 1, y: 0, scale: 1 }}
      exit={{ opacity: 0, y: -50, scale: 0.3 }}
      className={`border rounded-lg p-4 shadow-lg ${typeStyles[type]}`}
    >
      <div className="flex justify-between items-start">
        <div>
          <h4 className="font-semibold">{title}</h4>
          <p className="text-sm mt-1">{message}</p>
        </div>
        <button
          onClick={() => onDismiss(id)}
          className="ml-4 text-gray-400 hover:text-gray-600"
        >
          ×
        </button>
      </div>
    </motion.div>
  );
};

// Real-time notifications container
export const RealtimeNotifications: React.FC = () => {
  const [notifications, setNotifications] = useState<NotificationProps[]>([]);
  const { recentUpdates, clearNotifications } = usePostUpdates();

  // Create notifications from subscription updates
  useEffect(() => {
    recentUpdates.forEach(update => {
      const notification: NotificationProps = {
        id: `${update.type}-${update.post.id}-${update.timestamp}`,
        type: update.type === 'added' ? 'success' : 'info',
        title: update.type === 'added' ? 'New Post' : 'Post Updated',
        message:
          update.type === 'added'
            ? `${update.post.author.name} published "${update.post.title}"`
            : `"${update.post.title}" has been updated`,
        onDismiss: dismissNotification,
      };

      setNotifications(prev => [notification, ...prev.slice(0, 4)]); // Keep max 5
    });
  }, [recentUpdates]);

  const dismissNotification = useCallback((id: string) => {
    setNotifications(prev => prev.filter(notification => notification.id !== id));
  }, []);

  return (
    <div className="fixed top-4 right-4 z-50 space-y-2 max-w-sm">
      <AnimatePresence>
        {notifications.map(notification => (
          <Notification key={notification.id} {...notification} />
        ))}
      </AnimatePresence>
    </div>
  );
};

// Live comments component
interface LiveCommentsProps {
  postId: string;
}

export const LiveComments: React.FC<LiveCommentsProps> = ({ postId }) => {
  const { comments, loading, error } = usePostComments(postId);
  const [showNewCommentIndicator, setShowNewCommentIndicator] = useState(false);

  // Show indicator when new comments arrive
  useEffect(() => {
    if (comments.length > 0) {
      setShowNewCommentIndicator(true);
      const timer = setTimeout(() => {
        setShowNewCommentIndicator(false);
      }, 3000);
      return () => clearTimeout(timer);
    }
  }, [comments.length]);

  if (loading) return <div>Loading comments...</div>;
  if (error) return <div>Error loading comments: {error.message}</div>;

  return (
    <div className="space-y-4">
      <div className="flex items-center justify-between">
        <h3 className="text-lg font-semibold">Live Comments</h3>
        {showNewCommentIndicator && (
          <motion.div
            initial={{ scale: 0 }}
            animate={{ scale: 1 }}
            className="bg-green-500 text-white px-2 py-1 rounded-full text-xs"
          >
            New comments!
          </motion.div>
        )}
      </div>

      <div className="space-y-3 max-h-96 overflow-y-auto">
        <AnimatePresence>
          {comments.map(comment => (
            <motion.div
              key={comment.id}
              initial={{ opacity: 0, x: -20 }}
              animate={{ opacity: 1, x: 0 }}
              exit={{ opacity: 0, x: 20 }}
              className="flex space-x-3 p-3 bg-gray-50 rounded-lg"
            >
              <img
                src={comment.author.avatar || '/default-avatar.png'}
                alt={comment.author.name}
                className="w-8 h-8 rounded-full"
              />
              <div className="flex-1">
                <div className="flex items-center space-x-2">
                  <span className="font-medium text-sm">{comment.author.name}</span>
                  <span className="text-xs text-gray-500">
                    {new Date(comment.createdAt).toLocaleTimeString()}
                  </span>
                </div>
                <p className="text-sm mt-1">{comment.content}</p>
              </div>
            </motion.div>
          ))}
        </AnimatePresence>
      </div>
    </div>
  );
};

// User presence indicator
export const UserPresenceIndicator: React.FC = () => {
  const { onlineUsers, getOnlineCount, error } = useUserPresence();

  if (error) return null;

  return (
    <div className="flex items-center space-x-2 text-sm text-gray-600">
      <div className="flex -space-x-1">
        {onlineUsers.slice(0, 5).map(user => (
          <img
            key={user.id}
            src={user.avatar || '/default-avatar.png'}
            alt={user.name}
            className="w-6 h-6 rounded-full border-2 border-white"
            title={user.name}
          />
        ))}
      </div>
      <span>
        {getOnlineCount()} user{getOnlineCount() !== 1 ? 's' : ''} online
      </span>
      <div className="w-2 h-2 bg-green-500 rounded-full animate-pulse" />
    </div>
  );
};

// Live activity feed
export const LiveActivityFeed: React.FC = () => {
  const { recentUpdates } = usePostUpdates();
  const [visibleCount, setVisibleCount] = useState(3);

  const showMore = () => setVisibleCount(prev => prev + 3);
  const showLess = () => setVisibleCount(3);

  return (
    <div className="bg-white rounded-lg shadow p-6">
      <h2 className="text-xl font-semibold mb-4">Live Activity</h2>
      
      <div className="space-y-3">
        <AnimatePresence>
          {recentUpdates.slice(0, visibleCount).map((update, index) => (
            <motion.div
              key={`${update.type}-${update.post.id}-${update.timestamp}`}
              initial={{ opacity: 0, y: 20 }}
              animate={{ opacity: 1, y: 0 }}
              exit={{ opacity: 0, y: -20 }}
              transition={{ delay: index * 0.1 }}
              className="flex items-start space-x-3 p-3 border rounded-lg"
            >
              <div className="flex-shrink-0">
                {update.type === 'added' ? (
                  <div className="w-8 h-8 bg-green-100 rounded-full flex items-center justify-center">
                    <span className="text-green-600 text-sm">+</span>
                  </div>
                ) : (
                  <div className="w-8 h-8 bg-blue-100 rounded-full flex items-center justify-center">
                    <span className="text-blue-600 text-sm">↻</span>
                  </div>
                )}
              </div>
              
              <div className="flex-1">
                <p className="text-sm">
                  <span className="font-medium">{update.post.author.name}</span>
                  {update.type === 'added' ? ' published ' : ' updated '}
                  <span className="font-medium">"{update.post.title}"</span>
                </p>
                <p className="text-xs text-gray-500">
                  {new Date(update.timestamp).toLocaleTimeString()}
                </p>
              </div>
            </motion.div>
          ))}
        </AnimatePresence>
      </div>

      {recentUpdates.length > visibleCount && (
        <button
          onClick={showMore}
          className="mt-3 text-blue-600 hover:text-blue-800 text-sm"
        >
          Show more
        </button>
      )}

      {visibleCount > 3 && (
        <button
          onClick={showLess}
          className="mt-3 ml-4 text-gray-600 hover:text-gray-800 text-sm"
        >
          Show less
        </button>
      )}
    </div>
  );
};
```

### 3. Subscription Management and Performance
```typescript
// Advanced subscription management
class SubscriptionManager {
  private subscriptions = new Map<string, any>();
  private client: ApolloClient<any>;

  constructor(client: ApolloClient<any>) {
    this.client = client;
  }

  // Subscribe with automatic cleanup
  subscribe(
    key: string,
    subscription: DocumentNode,
    options: SubscriptionHookOptions = {}
  ) {
    // Clean up existing subscription with same key
    this.unsubscribe(key);

    const observable = this.client.subscribe({
      query: subscription,
      ...options,
    });

    const subscriptionObject = observable.subscribe({
      next: options.onSubscriptionData,
      error: options.onError,
      complete: options.onSubscriptionComplete,
    });

    this.subscriptions.set(key, subscriptionObject);

    return () => this.unsubscribe(key);
  }

  // Unsubscribe from specific subscription
  unsubscribe(key: string) {
    const subscription = this.subscriptions.get(key);
    if (subscription) {
      subscription.unsubscribe();
      this.subscriptions.delete(key);
    }
  }

  // Unsubscribe from all subscriptions
  unsubscribeAll() {
    this.subscriptions.forEach(subscription => {
      subscription.unsubscribe();
    });
    this.subscriptions.clear();
  }

  // Get active subscription count
  getActiveCount() {
    return this.subscriptions.size;
  }

  // Check if subscription is active
  isActive(key: string) {
    return this.subscriptions.has(key);
  }
}

// React hook for subscription management
export function useSubscriptionManager() {
  const client = useApolloClient();
  const managerRef = useRef<SubscriptionManager>();

  if (!managerRef.current) {
    managerRef.current = new SubscriptionManager(client);
  }

  // Cleanup on unmount
  useEffect(() => {
    return () => {
      managerRef.current?.unsubscribeAll();
    };
  }, []);

  return managerRef.current;
}

// Subscription batching for performance
class SubscriptionBatcher {
  private batchedSubscriptions = new Map<string, {
    subscription: DocumentNode;
    variables: any[];
    callbacks: Function[];
  }>();
  private batchTimeout: NodeJS.Timeout | null = null;
  private client: ApolloClient<any>;

  constructor(client: ApolloClient<any>) {
    this.client = client;
  }

  // Add subscription to batch
  addToBatch(
    subscriptionKey: string,
    subscription: DocumentNode,
    variables: any,
    callback: Function
  ) {
    if (!this.batchedSubscriptions.has(subscriptionKey)) {
      this.batchedSubscriptions.set(subscriptionKey, {
        subscription,
        variables: [],
        callbacks: [],
      });
    }

    const batch = this.batchedSubscriptions.get(subscriptionKey)!;
    batch.variables.push(variables);
    batch.callbacks.push(callback);

    this.scheduleBatchExecution();
  }

  private scheduleBatchExecution() {
    if (this.batchTimeout) return;

    this.batchTimeout = setTimeout(() => {
      this.executeBatch();
    }, 100); // Batch for 100ms
  }

  private executeBatch() {
    this.batchTimeout = null;

    for (const [key, batch] of this.batchedSubscriptions) {
      // Create multiplexed subscription
      const multiplexedSubscription = gql`
        subscription BatchedSubscription {
          ${batch.variables.map((_, index) => `
            sub${index}: ${batch.subscription.loc?.source.body.replace('subscription', '').trim()}
          `).join('\n')}
        }
      `;

      this.client.subscribe({
        query: multiplexedSubscription,
      }).subscribe({
        next: (data) => {
          // Distribute data to individual callbacks
          batch.callbacks.forEach((callback, index) => {
            const subData = data.data[`sub${index}`];
            if (subData) {
              callback({ data: subData });
            }
          });
        },
      });
    }

    this.batchedSubscriptions.clear();
  }
}

// Performance monitoring for subscriptions
export function useSubscriptionPerformance() {
  const [metrics, setMetrics] = useState({
    subscriptionCount: 0,
    messageCount: 0,
    errorCount: 0,
    averageLatency: 0,
    connectionUptime: 0,
  });

  const startTime = useRef(Date.now());
  const messageTimestamps = useRef<number[]>([]);

  const recordMessage = useCallback(() => {
    messageTimestamps.current.push(Date.now());
    
    setMetrics(prev => ({
      ...prev,
      messageCount: prev.messageCount + 1,
      connectionUptime: Date.now() - startTime.current,
    }));
  }, []);

  const recordError = useCallback(() => {
    setMetrics(prev => ({
      ...prev,
      errorCount: prev.errorCount + 1,
    }));
  }, []);

  const recordLatency = useCallback((latency: number) => {
    setMetrics(prev => {
      const newAverage = prev.messageCount > 0
        ? (prev.averageLatency * prev.messageCount + latency) / (prev.messageCount + 1)
        : latency;
      
      return {
        ...prev,
        averageLatency: newAverage,
      };
    });
  }, []);

  return {
    metrics,
    recordMessage,
    recordError,
    recordLatency,
  };
}
```

## Interview-Ready Summary

**Apollo Client and Real-time Subscriptions require:**

1. **Advanced Client Configuration** - HTTP/WS links, error handling, retry logic, persisted queries, auth integration
2. **Query/Mutation Patterns** - Type-safe operations, optimistic updates, cache management, error boundaries
3. **Real-time Subscriptions** - WebSocket connections, automatic reconnection, subscription batching, performance monitoring
4. **Cache Management** - Type policies, cache updates, invalidation strategies, optimistic responses

**Key implementation patterns:**
- **Connection Management** - Automatic reconnection, connection status tracking, error recovery
- **Performance Optimization** - Query batching, subscription multiplexing, cache optimization
- **Real-time UI** - Live notifications, presence indicators, activity feeds, optimistic updates
- **Error Handling** - Comprehensive error boundaries, retry mechanisms, graceful degradation

**Common challenges:** Connection drops, cache inconsistency, subscription memory leaks, query complexity, authentication with WebSockets.

**Best practices:** Use TypeScript for type safety, implement connection monitoring, batch subscription operations, handle offline scenarios, optimize cache policies, implement proper error boundaries, monitor subscription performance.