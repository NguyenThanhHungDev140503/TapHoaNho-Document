# TanStack Query - Advanced Patterns (Senior & Principal Level)

> **Version**: TanStack Query v5.90+ (Latest)  
> **Package**: `@tanstack/react-query`  
> **Last Updated**: 2025-01-27

---

## Senior Level - Nâng Cao (tiếp theo)

### 2. Custom Query Client Configuration

```typescript
// src/lib/queryClient.ts
import { QueryClient, QueryCache, MutationCache } from '@tanstack/react-query'

// Custom error handler
const handleQueryError = (error: unknown) => {
  console.error('Query error:', error)
  // Log to error tracking service (Sentry, etc.)
}

const handleMutationError = (error: unknown) => {
  console.error('Mutation error:', error)
  // Show global error notification
}

export const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: handleQueryError,
    onSuccess: (data, query) => {
      console.log('Query success:', query.queryKey)
    },
  }),
  
  mutationCache: new MutationCache({
    onError: handleMutationError,
    onSuccess: (data, variables, context, mutation) => {
      console.log('Mutation success:', mutation.options.mutationKey)
    },
  }),
  
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,
      gcTime: 1000 * 60 * 10,
      retry: (failureCount, error: any) => {
        if (error.status === 404 || error.status === 403) return false
        return failureCount < 3
      },
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
      refetchOnWindowFocus: process.env.NODE_ENV === 'production',
      refetchOnReconnect: true,
      refetchOnMount: true,
    },
    mutations: {
      retry: 1,
      onError: (error: any) => {
        if (error.status === 401) {
          // Redirect to login
          window.location.href = '/login'
        }
      },
    },
  },
})
```

### 3. Query Key Factory Pattern

```typescript
// src/lib/queryKeys.ts
export const queryKeys = {
  // Posts
  posts: {
    all: ['posts'] as const,
    lists: () => [...queryKeys.posts.all, 'list'] as const,
    list: (filters?: PostFilters) => 
      [...queryKeys.posts.lists(), filters] as const,
    details: () => [...queryKeys.posts.all, 'detail'] as const,
    detail: (id: number) => [...queryKeys.posts.details(), id] as const,
    infinite: (filters?: PostFilters) => 
      [...queryKeys.posts.all, 'infinite', filters] as const,
  },
  
  // Users
  users: {
    all: ['users'] as const,
    lists: () => [...queryKeys.users.all, 'list'] as const,
    list: (filters?: UserFilters) => 
      [...queryKeys.users.lists(), filters] as const,
    details: () => [...queryKeys.users.all, 'detail'] as const,
    detail: (id: number) => [...queryKeys.users.details(), id] as const,
    me: () => [...queryKeys.users.all, 'me'] as const,
  },
  
  // Comments
  comments: {
    all: ['comments'] as const,
    byPost: (postId: number) => 
      [...queryKeys.comments.all, 'post', postId] as const,
    detail: (id: number) => 
      [...queryKeys.comments.all, 'detail', id] as const,
  },
}

// Usage
const { data } = useQuery({
  queryKey: queryKeys.posts.detail(postId),
  queryFn: () => fetchPostById(postId),
})

// Invalidate all posts
queryClient.invalidateQueries({ queryKey: queryKeys.posts.all })

// Invalidate specific post
queryClient.invalidateQueries({ queryKey: queryKeys.posts.detail(postId) })
```

### 4. Custom Hooks Architecture

```typescript
// src/hooks/api/usePosts.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { queryKeys } from '@/lib/queryKeys'
import * as postsApi from '@/api/posts'

// Query hooks
export function usePosts(filters?: PostFilters) {
  return useQuery({
    queryKey: queryKeys.posts.list(filters),
    queryFn: () => postsApi.fetchPosts(filters),
  })
}

export function usePost(postId: number) {
  return useQuery({
    queryKey: queryKeys.posts.detail(postId),
    queryFn: () => postsApi.fetchPostById(postId),
    enabled: !!postId,
  })
}

export function useInfinitePosts(filters?: PostFilters) {
  return useInfiniteQuery({
    queryKey: queryKeys.posts.infinite(filters),
    queryFn: ({ pageParam = 0 }) => postsApi.fetchPostsPage(pageParam, filters),
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    initialPageParam: 0,
  })
}

// Mutation hooks
export function useCreatePost() {
  const queryClient = useQueryClient()
  
  return useMutation({
    mutationFn: postsApi.createPost,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: queryKeys.posts.lists() })
    },
  })
}

export function useUpdatePost() {
  const queryClient = useQueryClient()
  
  return useMutation({
    mutationFn: ({ id, data }: { id: number; data: Partial<Post> }) =>
      postsApi.updatePost(id, data),
    onMutate: async ({ id, data }) => {
      await queryClient.cancelQueries({ queryKey: queryKeys.posts.detail(id) })
      const previous = queryClient.getQueryData(queryKeys.posts.detail(id))
      
      queryClient.setQueryData(queryKeys.posts.detail(id), (old: Post) => ({
        ...old,
        ...data,
      }))
      
      return { previous }
    },
    onError: (err, { id }, context) => {
      if (context?.previous) {
        queryClient.setQueryData(queryKeys.posts.detail(id), context.previous)
      }
    },
    onSettled: (data, error, { id }) => {
      queryClient.invalidateQueries({ queryKey: queryKeys.posts.detail(id) })
    },
  })
}

export function useDeletePost() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: postsApi.deletePost,
    onSuccess: (data, postId) => {
      // Remove from cache
      queryClient.removeQueries({ queryKey: queryKeys.posts.detail(postId) })
      // Invalidate lists
      queryClient.invalidateQueries({ queryKey: queryKeys.posts.lists() })
    },
  })
}
```

### 5. Performance Optimization Techniques

#### Technique 1: Select - Chỉ subscribe vào data cần thiết

```typescript
// ❌ BAD: Component re-render khi bất kỳ field nào trong post thay đổi
function PostTitle({ postId }: { postId: number }) {
  const { data: post } = useQuery({
    queryKey: queryKeys.posts.detail(postId),
    queryFn: () => fetchPostById(postId),
  })

  return <h1>{post?.title}</h1>
}

// ✅ GOOD: Chỉ re-render khi title thay đổi
function PostTitle({ postId }: { postId: number }) {
  const title = useQuery({
    queryKey: queryKeys.posts.detail(postId),
    queryFn: () => fetchPostById(postId),
    // select: Transform và filter data
    // Component chỉ re-render khi selected data thay đổi
    select: (post) => post.title,
  })

  return <h1>{title.data}</h1>
}

// Advanced select với memoization
import { useCallback } from 'react'

function PostStats({ postId }: { postId: number }) {
  // Memoize selector để tránh re-compute
  const selectStats = useCallback(
    (post: Post) => ({
      wordCount: post.body.split(' ').length,
      readTime: Math.ceil(post.body.split(' ').length / 200),
    }),
    []
  )

  const { data: stats } = useQuery({
    queryKey: queryKeys.posts.detail(postId),
    queryFn: () => fetchPostById(postId),
    select: selectStats,
  })

  return (
    <div>
      <span>{stats?.wordCount} words</span>
      <span>{stats?.readTime} min read</span>
    </div>
  )
}
```

#### Technique 2: Structural Sharing

```typescript
// TanStack Query tự động thực hiện structural sharing
// Chỉ re-render khi data thực sự thay đổi

// Example: Array of objects
const { data: posts } = useQuery({
  queryKey: ['posts'],
  queryFn: fetchPosts,
})

// Nếu refetch trả về cùng data, component KHÔNG re-render
// TanStack Query so sánh deep equality và giữ reference cũ
```

#### Technique 3: Prefetching

```typescript
// src/hooks/usePrefetchPost.ts
import { useQueryClient } from '@tanstack/react-query'
import { queryKeys } from '@/lib/queryKeys'

export function usePrefetchPost() {
  const queryClient = useQueryClient()

  return {
    // Prefetch on hover
    prefetchOnHover: (postId: number) => {
      queryClient.prefetchQuery({
        queryKey: queryKeys.posts.detail(postId),
        queryFn: () => fetchPostById(postId),
        staleTime: 1000 * 60 * 5, // Cache for 5 minutes
      })
    },

    // Prefetch multiple posts
    prefetchPosts: async (postIds: number[]) => {
      await Promise.all(
        postIds.map((id) =>
          queryClient.prefetchQuery({
            queryKey: queryKeys.posts.detail(id),
            queryFn: () => fetchPostById(id),
          })
        )
      )
    },
  }
}

// Usage in component
function PostLink({ postId, title }: { postId: number; title: string }) {
  const { prefetchOnHover } = usePrefetchPost()

  return (
    <Link
      to={`/posts/${postId}`}
      onMouseEnter={() => prefetchOnHover(postId)}
    >
      {title}
    </Link>
  )
}
```

#### Technique 4: Initial Data

```typescript
// Pattern 1: Use data from list query
function usePost(postId: number) {
  const queryClient = useQueryClient()

  return useQuery({
    queryKey: queryKeys.posts.detail(postId),
    queryFn: () => fetchPostById(postId),
    // Sử dụng data từ list query làm initial data
    initialData: () => {
      const posts = queryClient.getQueryData<Post[]>(queryKeys.posts.lists())
      return posts?.find((post) => post.id === postId)
    },
    // initialDataUpdatedAt: Timestamp của list query
    initialDataUpdatedAt: () =>
      queryClient.getQueryState(queryKeys.posts.lists())?.dataUpdatedAt,
  })
}

// Pattern 2: SSR initial data
function PostDetail({ postId, initialPost }: { postId: number; initialPost?: Post }) {
  const { data: post } = useQuery({
    queryKey: queryKeys.posts.detail(postId),
    queryFn: () => fetchPostById(postId),
    initialData: initialPost, // Data từ SSR
  })

  return <div>{post.title}</div>
}
```

### 6. Testing Strategies

#### Unit Testing Hooks

```typescript
// src/hooks/__tests__/usePosts.test.ts
import { renderHook, waitFor } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { usePosts } from '../usePosts'
import * as postsApi from '@/api/posts'

// Mock API
jest.mock('@/api/posts')

describe('usePosts', () => {
  let queryClient: QueryClient

  beforeEach(() => {
    // Tạo QueryClient mới cho mỗi test
    queryClient = new QueryClient({
      defaultOptions: {
        queries: {
          retry: false, // Tắt retry trong tests
        },
      },
    })
  })

  afterEach(() => {
    queryClient.clear()
  })

  const wrapper = ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )

  it('should fetch posts successfully', async () => {
    const mockPosts = [
      { id: 1, title: 'Post 1', body: 'Body 1' },
      { id: 2, title: 'Post 2', body: 'Body 2' },
    ]

    ;(postsApi.fetchPosts as jest.Mock).mockResolvedValue(mockPosts)

    const { result } = renderHook(() => usePosts(), { wrapper })

    // Initial state
    expect(result.current.isLoading).toBe(true)
    expect(result.current.data).toBeUndefined()

    // Wait for query to complete
    await waitFor(() => expect(result.current.isSuccess).toBe(true))

    // Assert final state
    expect(result.current.data).toEqual(mockPosts)
    expect(result.current.isLoading).toBe(false)
    expect(postsApi.fetchPosts).toHaveBeenCalledTimes(1)
  })

  it('should handle errors', async () => {
    const mockError = new Error('Failed to fetch')
    ;(postsApi.fetchPosts as jest.Mock).mockRejectedValue(mockError)

    const { result } = renderHook(() => usePosts(), { wrapper })

    await waitFor(() => expect(result.current.isError).toBe(true))

    expect(result.current.error).toEqual(mockError)
    expect(result.current.data).toBeUndefined()
  })
})
```

#### Integration Testing với MSW

```typescript
// src/mocks/handlers.ts
import { rest } from 'msw'

export const handlers = [
  rest.get('/api/posts', (req, res, ctx) => {
    return res(
      ctx.status(200),
      ctx.json([
        { id: 1, title: 'Post 1', body: 'Body 1' },
        { id: 2, title: 'Post 2', body: 'Body 2' },
      ])
    )
  }),

  rest.get('/api/posts/:id', (req, res, ctx) => {
    const { id } = req.params
    return res(
      ctx.status(200),
      ctx.json({ id: Number(id), title: `Post ${id}`, body: `Body ${id}` })
    )
  }),

  rest.post('/api/posts', async (req, res, ctx) => {
    const body = await req.json()
    return res(
      ctx.status(201),
      ctx.json({ id: 3, ...body })
    )
  }),
]

// src/mocks/server.ts
import { setupServer } from 'msw/node'
import { handlers } from './handlers'

export const server = setupServer(...handlers)

// src/setupTests.ts
import { server } from './mocks/server'

beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

#### Component Testing

```typescript
// src/components/__tests__/PostsList.test.tsx
import { render, screen, waitFor } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { PostsList } from '../PostsList'

describe('PostsList', () => {
  let queryClient: QueryClient

  beforeEach(() => {
    queryClient = new QueryClient({
      defaultOptions: {
        queries: { retry: false },
      },
    })
  })

  const renderWithClient = (component: React.ReactElement) => {
    return render(
      <QueryClientProvider client={queryClient}>
        {component}
      </QueryClientProvider>
    )
  }

  it('should display posts', async () => {
    renderWithClient(<PostsList />)

    // Loading state
    expect(screen.getByText(/loading/i)).toBeInTheDocument()

    // Wait for posts to load
    await waitFor(() => {
      expect(screen.getByText('Post 1')).toBeInTheDocument()
      expect(screen.getByText('Post 2')).toBeInTheDocument()
    })
  })

  it('should handle errors', async () => {
    // Override handler to return error
    server.use(
      rest.get('/api/posts', (req, res, ctx) => {
        return res(ctx.status(500), ctx.json({ message: 'Server error' }))
      })
    )

    renderWithClient(<PostsList />)

    await waitFor(() => {
      expect(screen.getByText(/error/i)).toBeInTheDocument()
    })
  })
})
```

---

## Principal Level - Chuyên Gia

### 1. System Design với TanStack Query ở Quy Mô Lớn

#### Architecture Pattern: Feature-Based Structure

```
src/
├── features/
│   ├── posts/
│   │   ├── api/
│   │   │   ├── posts.api.ts          # API functions
│   │   │   └── posts.types.ts        # TypeScript types
│   │   ├── hooks/
│   │   │   ├── usePosts.ts           # Query hooks
│   │   │   ├── useCreatePost.ts      # Mutation hooks
│   │   │   └── usePostMutations.ts   # Combined mutations
│   │   ├── components/
│   │   │   ├── PostsList.tsx
│   │   │   ├── PostDetail.tsx
│   │   │   └── CreatePostForm.tsx
│   │   ├── utils/
│   │   │   └── postHelpers.ts
│   │   └── index.ts                  # Public exports
│   │
│   ├── users/
│   │   └── ... (similar structure)
│   │
│   └── comments/
│       └── ... (similar structure)
│
├── lib/
│   ├── queryClient.ts                # QueryClient configuration
│   ├── queryKeys.ts                  # Centralized query keys
│   └── api/
│       ├── client.ts                 # API client (axios/fetch)
│       └── interceptors.ts           # Request/response interceptors
│
└── shared/
    ├── hooks/
    │   ├── useAuth.ts
    │   └── usePermissions.ts
    └── components/
        ├── ErrorBoundary.tsx
        └── LoadingSpinner.tsx
```

