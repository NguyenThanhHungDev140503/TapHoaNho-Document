# TanStack Query - Principal Level Patterns

## 1. Enterprise-Scale Query Management

### Centralized API Client với Interceptors

```typescript
// src/lib/api/client.ts
import axios, { AxiosInstance, AxiosRequestConfig } from 'axios'
import { queryClient } from '../queryClient'

class APIClient {
  private client: AxiosInstance
  
  constructor(baseURL: string) {
    this.client = axios.create({
      baseURL,
      timeout: 30000,
      headers: {
        'Content-Type': 'application/json',
      },
    })
    
    this.setupInterceptors()
  }
  
  private setupInterceptors() {
    // Request interceptor
    this.client.interceptors.request.use(
      (config) => {
        // Add auth token
        const token = localStorage.getItem('auth_token')
        if (token) {
          config.headers.Authorization = `Bearer ${token}`
        }
        
        // Add request ID for tracking
        config.headers['X-Request-ID'] = crypto.randomUUID()
        
        // Log request in development
        if (process.env.NODE_ENV === 'development') {
          console.log('API Request:', config.method?.toUpperCase(), config.url)
        }
        
        return config
      },
      (error) => Promise.reject(error)
    )
    
    // Response interceptor
    this.client.interceptors.response.use(
      (response) => {
        // Log response in development
        if (process.env.NODE_ENV === 'development') {
          console.log('API Response:', response.status, response.config.url)
        }
        
        return response
      },
      async (error) => {
        const originalRequest = error.config
        
        // Handle 401 Unauthorized
        if (error.response?.status === 401 && !originalRequest._retry) {
          originalRequest._retry = true
          
          try {
            // Attempt to refresh token
            const newToken = await this.refreshToken()
            originalRequest.headers.Authorization = `Bearer ${newToken}`
            return this.client(originalRequest)
          } catch (refreshError) {
            // Redirect to login
            window.location.href = '/login'
            return Promise.reject(refreshError)
          }
        }
        
        // Handle 403 Forbidden
        if (error.response?.status === 403) {
          // Clear all queries and redirect
          queryClient.clear()
          window.location.href = '/forbidden'
        }
        
        // Handle network errors
        if (!error.response) {
          console.error('Network error:', error.message)
          // Show offline notification
        }
        
        return Promise.reject(error)
      }
    )
  }
  
  private async refreshToken(): Promise<string> {
    const refreshToken = localStorage.getItem('refresh_token')
    const response = await axios.post('/api/auth/refresh', { refreshToken })
    const { token } = response.data
    localStorage.setItem('auth_token', token)
    return token
  }
  
  // Generic request methods
  async get<T>(url: string, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.client.get<T>(url, config)
    return response.data
  }
  
  async post<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.client.post<T>(url, data, config)
    return response.data
  }
  
  async put<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.client.put<T>(url, data, config)
    return response.data
  }
  
  async patch<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.client.patch<T>(url, data, config)
    return response.data
  }
  
  async delete<T>(url: string, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.client.delete<T>(url, config)
    return response.data
  }
}

export const apiClient = new APIClient(process.env.REACT_APP_API_URL || '')
```

### 2. Advanced Prefetching Strategies

#### Router-Based Prefetching (React Router)

```typescript
// src/lib/router/prefetch.ts
import { QueryClient } from '@tanstack/react-query'
import { queryKeys } from '../queryKeys'
import * as postsApi from '@/features/posts/api/posts.api'

export const routePrefetchStrategies = {
  '/posts': async (queryClient: QueryClient) => {
    // Prefetch posts list
    await queryClient.prefetchQuery({
      queryKey: queryKeys.posts.lists(),
      queryFn: postsApi.fetchPosts,
    })
  },
  
  '/posts/:id': async (queryClient: QueryClient, params: { id: string }) => {
    const postId = Number(params.id)
    
    // Prefetch post detail
    await queryClient.prefetchQuery({
      queryKey: queryKeys.posts.detail(postId),
      queryFn: () => postsApi.fetchPostById(postId),
    })
    
    // Prefetch related data
    await Promise.all([
      queryClient.prefetchQuery({
        queryKey: queryKeys.comments.byPost(postId),
        queryFn: () => commentsApi.fetchCommentsByPostId(postId),
      }),
      queryClient.prefetchQuery({
        queryKey: queryKeys.users.detail(postId),
        queryFn: () => usersApi.fetchUserById(postId),
      }),
    ])
  },
}

// src/App.tsx
import { useEffect } from 'react'
import { useLocation, useParams } from 'react-router-dom'
import { useQueryClient } from '@tanstack/react-query'

export function App() {
  const location = useLocation()
  const params = useParams()
  const queryClient = useQueryClient()
  
  useEffect(() => {
    const prefetchStrategy = routePrefetchStrategies[location.pathname]
    if (prefetchStrategy) {
      prefetchStrategy(queryClient, params)
    }
  }, [location.pathname, params, queryClient])
  
  return <Routes />
}
```

### 3. SSR/SSG Integration

#### Next.js App Router Integration

```typescript
// app/posts/[id]/page.tsx
import { QueryClient, dehydrate, HydrationBoundary } from '@tanstack/react-query'
import { queryKeys } from '@/lib/queryKeys'
import { fetchPostById } from '@/features/posts/api/posts.api'
import { PostDetail } from '@/features/posts/components/PostDetail'

export async function generateStaticParams() {
  // Generate static paths
  const posts = await fetchPosts()
  return posts.map((post) => ({
    id: post.id.toString(),
  }))
}

export default async function PostPage({ params }: { params: { id: string } }) {
  const postId = Number(params.id)
  const queryClient = new QueryClient()

  // Prefetch on server
  await queryClient.prefetchQuery({
    queryKey: queryKeys.posts.detail(postId),
    queryFn: () => fetchPostById(postId),
  })

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <PostDetail postId={postId} />
    </HydrationBoundary>
  )
}
```

### 4. Streaming Queries (Experimental)

```typescript
// src/hooks/useStreamingQuery.ts
import { useQuery } from '@tanstack/react-query'
import { useState, useEffect } from 'react'

interface StreamingOptions<T> {
  queryKey: any[]
  streamUrl: string
  onChunk: (chunk: T) => void
}

export function useStreamingQuery<T>({ queryKey, streamUrl, onChunk }: StreamingOptions<T>) {
  const [chunks, setChunks] = useState<T[]>([])
  const [isStreaming, setIsStreaming] = useState(false)

  useEffect(() => {
    let controller: AbortController

    const startStreaming = async () => {
      controller = new AbortController()
      setIsStreaming(true)

      try {
        const response = await fetch(streamUrl, {
          signal: controller.signal,
        })

        const reader = response.body?.getReader()
        const decoder = new TextDecoder()

        while (true) {
          const { done, value } = await reader!.read()
          if (done) break

          const chunk = JSON.parse(decoder.decode(value))
          setChunks((prev) => [...prev, chunk])
          onChunk(chunk)
        }
      } catch (error) {
        if (error.name !== 'AbortError') {
          console.error('Streaming error:', error)
        }
      } finally {
        setIsStreaming(false)
      }
    }

    startStreaming()

    return () => {
      controller?.abort()
    }
  }, [streamUrl])

  return {
    chunks,
    isStreaming,
  }
}
```

### 5. Migration Strategies

#### Migrating from Redux to TanStack Query

```typescript
// Before: Redux approach
// store/postsSlice.ts
const postsSlice = createSlice({
  name: 'posts',
  initialState: {
    data: [],
    loading: false,
    error: null,
  },
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchPosts.pending, (state) => {
        state.loading = true
      })
      .addCase(fetchPosts.fulfilled, (state, action) => {
        state.loading = false
        state.data = action.payload
      })
      .addCase(fetchPosts.rejected, (state, action) => {
        state.loading = false
        state.error = action.error.message
      })
  },
})

// After: TanStack Query approach
// hooks/usePosts.ts
export function usePosts() {
  return useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  })
}

// Migration steps:
// 1. Identify server state in Redux
// 2. Create TanStack Query hooks
// 3. Replace Redux selectors with query hooks
// 4. Remove Redux actions and reducers
// 5. Keep Redux only for UI state
```

#### Coexistence Strategy

```typescript
// src/hooks/useHybridState.ts
// Sử dụng cả Redux và TanStack Query trong quá trình migration

import { useSelector } from 'react-redux'
import { useQuery } from '@tanstack/react-query'

export function useHybridPosts() {
  // Redux state (legacy)
  const reduxPosts = useSelector((state) => state.posts.data)
  const isReduxLoading = useSelector((state) => state.posts.loading)

  // TanStack Query (new)
  const { data: queryPosts, isLoading: isQueryLoading } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
    enabled: false, // Disable auto-fetch initially
  })

  // Feature flag để switch giữa Redux và TanStack Query
  const useTanStackQuery = useFeatureFlag('use-tanstack-query')

  return {
    posts: useTanStackQuery ? queryPosts : reduxPosts,
    isLoading: useTanStackQuery ? isQueryLoading : isReduxLoading,
  }
}
```

### 6. Production Monitoring

#### Performance Metrics

```typescript
// src/lib/monitoring/queryMetrics.ts
import { QueryClient } from '@tanstack/react-query'

interface QueryMetrics {
  queryKey: string
  duration: number
  status: 'success' | 'error'
  cacheHit: boolean
  timestamp: number
}

export class QueryMonitor {
  private metrics: QueryMetrics[] = []

  constructor(private queryClient: QueryClient) {
    this.setupMonitoring()
  }

  private setupMonitoring() {
    // Monitor query performance
    this.queryClient.getQueryCache().subscribe((event) => {
      if (event.type === 'updated') {
        const query = event.query
        const state = query.state

        const metric: QueryMetrics = {
          queryKey: JSON.stringify(query.queryKey),
          duration: state.dataUpdatedAt - state.fetchMeta?.startedAt || 0,
          status: state.status === 'success' ? 'success' : 'error',
          cacheHit: state.fetchStatus === 'idle',
          timestamp: Date.now(),
        }

        this.metrics.push(metric)
        this.analyzeMetrics(metric)
      }
    })
  }

  private analyzeMetrics(metric: QueryMetrics) {
    // Alert on slow queries
    if (metric.duration > 5000) {
      this.sendAlert({
        type: 'slow_query',
        message: `Query ${metric.queryKey} took ${metric.duration}ms`,
        severity: 'warning',
      })
    }

    // Track error rate
    const recentMetrics = this.metrics.slice(-100)
    const errorRate = recentMetrics.filter((m) => m.status === 'error').length / recentMetrics.length

    if (errorRate > 0.1) {
      this.sendAlert({
        type: 'high_error_rate',
        message: `Query error rate: ${(errorRate * 100).toFixed(2)}%`,
        severity: 'critical',
      })
    }

    // Track cache hit rate
    const cacheHitRate = recentMetrics.filter((m) => m.cacheHit).length / recentMetrics.length

    if (cacheHitRate < 0.5) {
      this.sendAlert({
        type: 'low_cache_hit_rate',
        message: `Cache hit rate: ${(cacheHitRate * 100).toFixed(2)}%`,
        severity: 'info',
      })
    }
  }

  private sendAlert(alert: any) {
    // Send to monitoring service (Datadog, New Relic, etc.)
    console.warn('Query Alert:', alert)

    // Send to external service
    fetch('/api/monitoring/alerts', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(alert),
    })
  }

  getMetrics() {
    return this.metrics
  }

  getAverageQueryTime() {
    const total = this.metrics.reduce((sum, m) => sum + m.duration, 0)
    return total / this.metrics.length
  }

  getCacheHitRate() {
    const hits = this.metrics.filter((m) => m.cacheHit).length
    return hits / this.metrics.length
  }
}

// Usage
const queryMonitor = new QueryMonitor(queryClient)

// Get metrics
console.log('Average query time:', queryMonitor.getAverageQueryTime())
console.log('Cache hit rate:', queryMonitor.getCacheHitRate())
```

### 7. Advanced Error Recovery

```typescript
// src/lib/errorRecovery.ts
import { QueryClient } from '@tanstack/react-query'

export function setupErrorRecovery(queryClient: QueryClient) {
  // Automatic retry with exponential backoff
  queryClient.setDefaultOptions({
    queries: {
      retry: (failureCount, error: any) => {
        // Don't retry on 4xx errors (client errors)
        if (error.status >= 400 && error.status < 500) {
          return false
        }

        // Retry up to 3 times for 5xx errors
        return failureCount < 3
      },

      retryDelay: (attemptIndex) => {
        // Exponential backoff: 1s, 2s, 4s, 8s, ...
        return Math.min(1000 * 2 ** attemptIndex, 30000)
      },
    },
  })

  // Global error handler
  queryClient.getQueryCache().subscribe((event) => {
    if (event.type === 'updated' && event.action.type === 'error') {
      const error = event.query.state.error as any

      // Handle specific error types
      if (error.status === 401) {
        // Unauthorized - redirect to login
        window.location.href = '/login'
      } else if (error.status === 403) {
        // Forbidden - show permission error
        showNotification({
          type: 'error',
          message: 'You do not have permission to access this resource',
        })
      } else if (error.status === 503) {
        // Service unavailable - show maintenance message
        showNotification({
          type: 'warning',
          message: 'Service is temporarily unavailable. Please try again later.',
        })
      } else if (!error.status) {
        // Network error - show offline message
        showNotification({
          type: 'error',
          message: 'Network error. Please check your internet connection.',
        })
      }
    }
  })
}
```

### 8. Best Practices Summary

#### Do's ✅

1. **Use Query Key Factory**
   ```typescript
   const queryKeys = {
     posts: {
       all: ['posts'] as const,
       detail: (id: number) => [...queryKeys.posts.all, id] as const,
     },
   }
   ```

2. **Implement Optimistic Updates**
   ```typescript
   onMutate: async (newData) => {
     await queryClient.cancelQueries({ queryKey })
     const previous = queryClient.getQueryData(queryKey)
     queryClient.setQueryData(queryKey, newData)
     return { previous }
   }
   ```

3. **Use TypeScript**
   ```typescript
   const { data } = useQuery<Post[], Error>({
     queryKey: ['posts'],
     queryFn: fetchPosts,
   })
   ```

4. **Monitor Performance**
   ```typescript
   setupQueryMonitoring(queryClient)
   ```

5. **Test Thoroughly**
   ```typescript
   renderHook(() => usePosts(), { wrapper: QueryClientProvider })
   ```

#### Don'ts ❌

1. **Don't use useQuery in event handlers**
   ```typescript
   // ❌ Wrong
   const handleClick = () => {
     const { data } = useQuery({ ... })
   }

   // ✅ Correct
   const { data, refetch } = useQuery({ ... })
   const handleClick = () => refetch()
   ```

2. **Don't forget to include dependencies in query keys**
   ```typescript
   // ❌ Wrong
   queryKey: ['posts']
   queryFn: () => fetchPosts(userId)

   // ✅ Correct
   queryKey: ['posts', userId]
   queryFn: () => fetchPosts(userId)
   ```

3. **Don't duplicate server state in other state managers**
   ```typescript
   // ❌ Wrong: Storing API data in Redux
   // ✅ Correct: Use TanStack Query for server state, Redux for UI state
   ```

4. **Don't ignore error handling**
   ```typescript
   // ❌ Wrong
   const { data } = useQuery({ ... })

   // ✅ Correct
   const { data, isError, error } = useQuery({ ... })
   if (isError) return <Error error={error} />
   ```

5. **Don't set staleTime too high for dynamic data**
   ```typescript
   // ❌ Wrong for frequently changing data
   staleTime: Infinity

   // ✅ Correct
   staleTime: 1000 * 60 * 5 // 5 minutes
   ```

---

## Kết Luận

Tài liệu này cung cấp comprehensive guide về TanStack Query từ Junior đến Principal level. Để thành công với TanStack Query trong production:

1. **Start Simple**: Bắt đầu với basic queries và mutations
2. **Iterate**: Thêm optimistic updates, prefetching khi cần
3. **Monitor**: Track performance và errors trong production
4. **Test**: Viết tests cho tất cả query và mutation logic
5. **Document**: Maintain clear documentation cho team

**Resources:**
- Main documentation: `TanStack Query.md`
- Advanced patterns: `Advanced-Patterns.md`
- This file: `Principal-Level-Patterns.md`

Happy coding! 🚀

