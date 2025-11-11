# TanStack Query - Hướng Dẫn Toàn Diện

## Mục Lục

1. [Giới Thiệu Tổng Quan](#giới-thiệu-tổng-quan)
2. [Junior Level - Cơ Bản](#junior-level---cơ-bản)
3. [Middle Level - Trung Cấp](#middle-level---trung-cấp)
4. [Senior Level - Nâng Cao](#senior-level---nâng-cao)
5. [Principal Level - Chuyên Gia](#principal-level---chuyên-gia)
6. [Tài Liệu Tham Khảo](#tài-liệu-tham-khảo)

---

## Giới Thiệu Tổng Quan

### TanStack Query là gì?

**TanStack Query** (trước đây là React Query) là một thư viện quản lý trạng thái server-state mạnh mẽ cho các ứng dụng web hiện đại. Được phát triển bởi Tanner Linsley và đội ngũ TanStack, nó giúp đơn giản hóa việc fetching, caching, synchronizing và updating server state trong React applications.

### Tại sao sử dụng TanStack Query?

**Vấn đề truyền thống:**
- Quản lý loading states thủ công
- Xử lý error states phức tạp
- Caching data không hiệu quả
- Refetching và synchronization khó khăn
- Boilerplate code nhiều

**Giải pháp TanStack Query:**
- ✅ Tự động quản lý cache
- ✅ Background refetching thông minh
- ✅ Optimistic updates
- ✅ Pagination và infinite scroll dễ dàng
- ✅ Request deduplication
- ✅ DevTools mạnh mẽ

### Các khái niệm cốt lõi

1. **Queries**: Fetching và caching data từ server
2. **Mutations**: Tạo/cập nhật/xóa data trên server
3. **Query Keys**: Unique identifiers cho queries
4. **Cache**: Lưu trữ data đã fetch
5. **Stale Time**: Thời gian data được coi là "fresh"
6. **Cache Time**: Thời gian data được giữ trong cache

### TanStack Ecosystem

TanStack Query là một phần của TanStack suite:
- **TanStack Table**: Headless UI cho data tables
- **TanStack Router**: Type-safe routing
- **TanStack Form**: Form state management
- **TanStack Virtual**: Virtualization cho large lists

---

## Junior Level - Cơ Bản

### 1. Cài Đặt và Cấu Hình

#### Cài đặt package

```bash
# NPM
npm install @tanstack/react-query

# Yarn
yarn add @tanstack/react-query

# PNPM
pnpm add @tanstack/react-query

# DevTools (optional nhưng highly recommended)
npm install @tanstack/react-query-devtools
```

#### Setup cơ bản

```typescript
// src/main.tsx hoặc src/index.tsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'
import App from './App'

// Tạo QueryClient instance
// QueryClient là trung tâm quản lý tất cả queries và mutations
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // staleTime: Thời gian data được coi là "fresh" (mặc định: 0ms)
      // Trong 5 phút, data sẽ không bị refetch tự động
      staleTime: 1000 * 60 * 5, // 5 minutes
      
      // gcTime (garbage collection time): Thời gian data được giữ trong cache
      // Sau 10 phút không sử dụng, data sẽ bị xóa khỏi cache
      gcTime: 1000 * 60 * 10, // 10 minutes
      
      // retry: Số lần retry khi request fail
      retry: 3,
      
      // refetchOnWindowFocus: Tự động refetch khi user focus vào window
      refetchOnWindowFocus: true,
      
      // refetchOnReconnect: Tự động refetch khi reconnect internet
      refetchOnReconnect: true,
    },
  },
})

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    {/* Wrap app với QueryClientProvider để provide QueryClient cho toàn bộ app */}
    <QueryClientProvider client={queryClient}>
      <App />
      {/* DevTools giúp debug queries dễ dàng - chỉ hiển thị trong development */}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  </React.StrictMode>
)
```

**Giải thích chi tiết:**

1. **QueryClient**: Là core của TanStack Query, quản lý cache và configuration
2. **staleTime**: Quyết định khi nào data cần được refetch. Nếu data "fresh" (chưa stale), sẽ không refetch
3. **gcTime**: Quyết định khi nào data bị xóa khỏi cache sau khi không còn observers
4. **retry**: Tự động retry khi request fail, giúp xử lý network issues
5. **QueryClientProvider**: Context provider để các components con có thể access QueryClient

### 2. useQuery - Fetching Data Cơ Bản

#### Example 1: Fetch danh sách posts

```typescript
// src/types/post.ts
export interface Post {
  id: number
  title: string
  body: string
  userId: number
}

// src/api/posts.ts
export const fetchPosts = async (): Promise<Post[]> => {
  const response = await fetch('https://jsonplaceholder.typicode.com/posts')

  // Kiểm tra response status
  if (!response.ok) {
    throw new Error('Failed to fetch posts')
  }

  return response.json()
}

// src/components/PostsList.tsx
import { useQuery } from '@tanstack/react-query'
import { fetchPosts } from '../api/posts'

export function PostsList() {
  // useQuery hook để fetch data
  const {
    data,           // Data trả về từ queryFn
    error,          // Error object nếu có lỗi
    isLoading,      // true khi đang fetch lần đầu
    isError,        // true khi có error
    isFetching,     // true khi đang fetch (bao gồm cả background refetch)
    isSuccess,      // true khi fetch thành công
  } = useQuery({
    // queryKey: Unique identifier cho query này
    // Phải là array, có thể chứa nhiều elements để tạo unique key
    queryKey: ['posts'],

    // queryFn: Function để fetch data
    // Phải return Promise
    queryFn: fetchPosts,

    // staleTime: Override global config cho query này
    staleTime: 1000 * 60, // 1 minute
  })

  // Hiển thị loading state
  if (isLoading) {
    return <div>Đang tải danh sách bài viết...</div>
  }

  // Hiển thị error state
  if (isError) {
    return <div>Lỗi: {error.message}</div>
  }

  // Hiển thị data khi success
  return (
    <div>
      <h1>Danh Sách Bài Viết</h1>
      {/* isFetching cho biết đang có background refetch */}
      {isFetching && <span>Đang cập nhật...</span>}

      <ul>
        {data.map((post) => (
          <li key={post.id}>
            <h3>{post.title}</h3>
            <p>{post.body}</p>
          </li>
        ))}
      </ul>
    </div>
  )
}
```

**Giải thích chi tiết:**

1. **queryKey**:
   - Là unique identifier cho query
   - TanStack Query sử dụng key này để cache và track query
   - Khi queryKey thay đổi, query sẽ refetch

2. **queryFn**:
   - Function thực hiện fetch data
   - Phải return Promise
   - Nếu throw error, query sẽ chuyển sang error state

3. **isLoading vs isFetching**:
   - `isLoading`: true chỉ khi fetch lần đầu và chưa có cached data
   - `isFetching`: true khi đang fetch (bao gồm cả background refetch)

4. **Automatic Caching**:
   - Data được cache tự động với queryKey
   - Lần mount component tiếp theo sẽ dùng cached data ngay lập tức
   - Background refetch diễn ra nếu data đã stale

#### Example 2: Fetch single post với dynamic parameter

```typescript
// src/api/posts.ts
export const fetchPostById = async (postId: number): Promise<Post> => {
  const response = await fetch(
    `https://jsonplaceholder.typicode.com/posts/${postId}`
  )

  if (!response.ok) {
    throw new Error(`Failed to fetch post ${postId}`)
  }

  return response.json()
}

// src/components/PostDetail.tsx
import { useQuery } from '@tanstack/react-query'
import { fetchPostById } from '../api/posts'

interface PostDetailProps {
  postId: number
}

export function PostDetail({ postId }: PostDetailProps) {
  const { data, isLoading, isError, error } = useQuery({
    // queryKey bao gồm postId để tạo unique key cho mỗi post
    // Khi postId thay đổi, query sẽ tự động refetch
    queryKey: ['post', postId],

    // queryFn nhận postId từ closure
    queryFn: () => fetchPostById(postId),

    // enabled: Chỉ fetch khi condition = true
    // Hữu ích khi cần wait for dependencies
    enabled: !!postId, // Chỉ fetch khi postId tồn tại
  })

  if (!postId) {
    return <div>Vui lòng chọn một bài viết</div>
  }

  if (isLoading) {
    return <div>Đang tải bài viết...</div>
  }

  if (isError) {
    return <div>Lỗi: {error.message}</div>
  }

  return (
    <article>
      <h1>{data.title}</h1>
      <p>{data.body}</p>
    </article>
  )
}
```

**Giải thích chi tiết:**

1. **Dynamic Query Keys**:
   - `['post', postId]` tạo unique key cho mỗi post
   - `['post', 1]` và `['post', 2]` là 2 queries khác nhau
   - Mỗi query có cache riêng

2. **enabled option**:
   - Conditional fetching
   - Query chỉ chạy khi `enabled = true`
   - Hữu ích cho dependent queries

3. **Query Key Best Practices**:
   - Luôn bao gồm tất cả variables ảnh hưởng đến query
   - Thứ tự elements trong array quan trọng
   - `['posts', 1]` ≠ `[1, 'posts']`

### 3. useMutation - Thay Đổi Data

#### Example 1: Tạo post mới

```typescript
// src/api/posts.ts
interface CreatePostInput {
  title: string
  body: string
  userId: number
}

export const createPost = async (newPost: CreatePostInput): Promise<Post> => {
  const response = await fetch('https://jsonplaceholder.typicode.com/posts', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(newPost),
  })

  if (!response.ok) {
    throw new Error('Failed to create post')
  }

  return response.json()
}

// src/components/CreatePostForm.tsx
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { useState } from 'react'
import { createPost } from '../api/posts'

export function CreatePostForm() {
  const [title, setTitle] = useState('')
  const [body, setBody] = useState('')

  // useQueryClient để access QueryClient instance
  const queryClient = useQueryClient()

  // useMutation hook để thực hiện mutations
  const mutation = useMutation({
    // mutationFn: Function thực hiện mutation
    mutationFn: createPost,

    // onSuccess: Callback khi mutation thành công
    onSuccess: (data) => {
      console.log('Post created:', data)

      // Invalidate queries để trigger refetch
      // Tất cả queries với queryKey bắt đầu bằng ['posts'] sẽ bị invalidate
      queryClient.invalidateQueries({ queryKey: ['posts'] })

      // Reset form
      setTitle('')
      setBody('')
    },

    // onError: Callback khi mutation fail
    onError: (error) => {
      console.error('Failed to create post:', error)
      alert('Không thể tạo bài viết. Vui lòng thử lại.')
    },
  })

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault()

    // Trigger mutation với data
    mutation.mutate({
      title,
      body,
      userId: 1,
    })
  }

  return (
    <form onSubmit={handleSubmit}>
      <h2>Tạo Bài Viết Mới</h2>

      <div>
        <label htmlFor="title">Tiêu đề:</label>
        <input
          id="title"
          type="text"
          value={title}
          onChange={(e) => setTitle(e.target.value)}
          disabled={mutation.isPending}
          required
        />
      </div>

      <div>
        <label htmlFor="body">Nội dung:</label>
        <textarea
          id="body"
          value={body}
          onChange={(e) => setBody(e.target.value)}
          disabled={mutation.isPending}
          required
        />
      </div>

      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'Đang tạo...' : 'Tạo Bài Viết'}
      </button>

      {/* Hiển thị status */}
      {mutation.isError && (
        <div style={{ color: 'red' }}>
          Lỗi: {mutation.error.message}
        </div>
      )}

      {mutation.isSuccess && (
        <div style={{ color: 'green' }}>
          Tạo bài viết thành công!
        </div>
      )}
    </form>
  )
}
```

**Giải thích chi tiết:**

1. **useMutation**:
   - Dùng cho operations thay đổi data (POST, PUT, DELETE)
   - Không tự động cache như useQuery
   - Cung cấp callbacks: onSuccess, onError, onSettled

2. **mutation.mutate()**:
   - Trigger mutation với data
   - Async operation nhưng không return Promise
   - Dùng `mutateAsync()` nếu cần await

3. **queryClient.invalidateQueries()**:
   - Đánh dấu queries là stale
   - Trigger refetch nếu query đang active
   - Đảm bảo UI sync với server state

4. **Mutation States**:
   - `isPending`: Đang thực hiện mutation
   - `isSuccess`: Mutation thành công
   - `isError`: Mutation thất bại
   - `data`: Data trả về từ mutation
   - `error`: Error object nếu có

### 4. Common Use Cases

#### Use Case 1: Fetch data với loading và error handling

```typescript
// src/hooks/usePosts.ts
import { useQuery } from '@tanstack/react-query'
import { fetchPosts } from '../api/posts'

export function usePosts() {
  return useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
    // Các options phổ biến
    staleTime: 1000 * 60 * 5, // 5 minutes
    retry: 3,
    retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
  })
}

// src/components/PostsPage.tsx
import { usePosts } from '../hooks/usePosts'

export function PostsPage() {
  const { data, isLoading, isError, error } = usePosts()

  if (isLoading) return <LoadingSpinner />
  if (isError) return <ErrorMessage error={error} />

  return <PostsList posts={data} />
}
```

#### Use Case 2: Update data với optimistic update đơn giản

```typescript
// src/components/LikeButton.tsx
import { useMutation, useQueryClient } from '@tanstack/react-query'

interface LikeButtonProps {
  postId: number
  initialLikes: number
}

export function LikeButton({ postId, initialLikes }: LikeButtonProps) {
  const queryClient = useQueryClient()

  const likeMutation = useMutation({
    mutationFn: async (postId: number) => {
      const response = await fetch(`/api/posts/${postId}/like`, {
        method: 'POST',
      })
      return response.json()
    },
    onSuccess: () => {
      // Invalidate để refetch post data
      queryClient.invalidateQueries({ queryKey: ['post', postId] })
    },
  })

  return (
    <button
      onClick={() => likeMutation.mutate(postId)}
      disabled={likeMutation.isPending}
    >
      ❤️ {initialLikes} {likeMutation.isPending && '...'}
    </button>
  )
}
```

### 5. Common Pitfalls (Lỗi Thường Gặp)

#### ❌ Pitfall 1: Query key không đúng

```typescript
// SAI: Query key không bao gồm dependencies
const { data } = useQuery({
  queryKey: ['posts'], // Thiếu userId
  queryFn: () => fetchPostsByUser(userId), // userId thay đổi nhưng query không refetch
})

// ĐÚNG: Bao gồm tất cả dependencies
const { data } = useQuery({
  queryKey: ['posts', userId], // Khi userId thay đổi, query tự động refetch
  queryFn: () => fetchPostsByUser(userId),
})
```

#### ❌ Pitfall 2: Không handle loading và error states

```typescript
// SAI: Không check loading và error
function PostsList() {
  const { data } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  })

  // data có thể undefined khi loading hoặc error
  return <div>{data.map(...)}</div> // Runtime error!
}

// ĐÚNG: Handle tất cả states
function PostsList() {
  const { data, isLoading, isError, error } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  })

  if (isLoading) return <div>Loading...</div>
  if (isError) return <div>Error: {error.message}</div>

  return <div>{data.map(...)}</div>
}
```

#### ❌ Pitfall 3: Quên invalidate queries sau mutation

```typescript
// SAI: Không invalidate queries
const mutation = useMutation({
  mutationFn: createPost,
  // Không có onSuccess -> UI không update
})

// ĐÚNG: Invalidate để trigger refetch
const mutation = useMutation({
  mutationFn: createPost,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['posts'] })
  },
})
```

#### ❌ Pitfall 4: Sử dụng useQuery trong event handlers

```typescript
// SAI: useQuery trong event handler
function MyComponent() {
  const handleClick = () => {
    // Hooks không được gọi trong callbacks!
    const { data } = useQuery({ ... }) // Error!
  }

  return <button onClick={handleClick}>Click</button>
}

// ĐÚNG: Sử dụng refetch hoặc useMutation
function MyComponent() {
  const { data, refetch } = useQuery({
    queryKey: ['data'],
    queryFn: fetchData,
    enabled: false, // Không fetch tự động
  })

  const handleClick = () => {
    refetch() // Trigger fetch manually
  }

  return <button onClick={handleClick}>Click</button>
}
```

### 6. Best Practices cho Junior Level

1. **Luôn define query keys rõ ràng**
   ```typescript
   // Tốt: Descriptive và bao gồm dependencies
   queryKey: ['posts', 'list', { status: 'published', page: 1 }]

   // Tránh: Quá chung chung
   queryKey: ['data']
   ```

2. **Tách API logic ra khỏi components**
   ```typescript
   // api/posts.ts - API functions
   // hooks/usePosts.ts - Custom hooks với useQuery
   // components/PostsList.tsx - UI components
   ```

3. **Sử dụng TypeScript để type safety**
   ```typescript
   interface Post {
     id: number
     title: string
     body: string
   }

   const { data } = useQuery<Post[]>({
     queryKey: ['posts'],
     queryFn: fetchPosts,
   })
   // data có type Post[] | undefined
   ```

4. **Handle tất cả states (loading, error, success)**
   ```typescript
   if (isLoading) return <Loading />
   if (isError) return <Error error={error} />
   return <Success data={data} />
   ```

5. **Sử dụng DevTools để debug**
   ```typescript
   import { ReactQueryDevtools } from '@tanstack/react-query-devtools'

   <QueryClientProvider client={queryClient}>
     <App />
     <ReactQueryDevtools initialIsOpen={false} />
   </QueryClientProvider>
   ```

---

## Middle Level - Trung Cấp

### 1. Query Invalidation và Refetching Strategies

#### Invalidation Patterns

```typescript
import { useQueryClient } from '@tanstack/react-query'

function usePostMutations() {
  const queryClient = useQueryClient()

  // Pattern 1: Invalidate exact query
  const invalidateExactPost = (postId: number) => {
    queryClient.invalidateQueries({
      queryKey: ['post', postId],
      exact: true, // Chỉ invalidate query này, không invalidate queries con
    })
  }

  // Pattern 2: Invalidate tất cả queries bắt đầu với prefix
  const invalidateAllPosts = () => {
    queryClient.invalidateQueries({
      queryKey: ['posts'], // Invalidate ['posts'], ['posts', 1], ['posts', 'list'], etc.
    })
  }

  // Pattern 3: Invalidate với predicate function
  const invalidatePublishedPosts = () => {
    queryClient.invalidateQueries({
      predicate: (query) => {
        // Custom logic để quyết định query nào cần invalidate
        const [key, params] = query.queryKey as [string, any]
        return key === 'posts' && params?.status === 'published'
      },
    })
  }

  // Pattern 4: Refetch immediately
  const refetchPosts = async () => {
    await queryClient.refetchQueries({
      queryKey: ['posts'],
      type: 'active', // Chỉ refetch queries đang active (có observers)
    })
  }

  return {
    invalidateExactPost,
    invalidateAllPosts,
    invalidatePublishedPosts,
    refetchPosts,
  }
}
```

**Giải thích chi tiết:**

1. **invalidateQueries vs refetchQueries**:
   - `invalidateQueries`: Đánh dấu queries là stale, refetch khi có observers
   - `refetchQueries`: Force refetch ngay lập tức

2. **exact option**:
   - `exact: true`: Chỉ match query key chính xác
   - `exact: false` (default): Match tất cả queries bắt đầu với key

3. **predicate function**:
   - Custom logic để filter queries
   - Hữu ích cho complex invalidation logic

#### Refetching Strategies

```typescript
// Strategy 1: Refetch on window focus
const { data } = useQuery({
  queryKey: ['posts'],
  queryFn: fetchPosts,
  refetchOnWindowFocus: true, // Default: true
  // User switch tab và quay lại -> auto refetch
})

// Strategy 2: Refetch on mount
const { data } = useQuery({
  queryKey: ['posts'],
  queryFn: fetchPosts,
  refetchOnMount: true, // Default: true
  // Component mount -> refetch nếu data stale
})

// Strategy 3: Refetch on reconnect
const { data } = useQuery({
  queryKey: ['posts'],
  queryFn: fetchPosts,
  refetchOnReconnect: true, // Default: true
  // Internet reconnect -> auto refetch
})

// Strategy 4: Polling (refetch interval)
const { data } = useQuery({
  queryKey: ['live-data'],
  queryFn: fetchLiveData,
  refetchInterval: 5000, // Refetch mỗi 5 giây
  refetchIntervalInBackground: true, // Tiếp tục refetch khi tab không active
})

// Strategy 5: Manual refetch
function MyComponent() {
  const { data, refetch } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  })

  return (
    <div>
      <button onClick={() => refetch()}>Refresh</button>
      {/* ... */}
    </div>
  )
}
```

### 2. Optimistic Updates

Optimistic updates cho phép update UI ngay lập tức trước khi server response, tạo trải nghiệm mượt mà hơn.

#### Example: Optimistic update cho like button

```typescript
// src/hooks/useLikePost.ts
import { useMutation, useQueryClient } from '@tanstack/react-query'

interface Post {
  id: number
  title: string
  likes: number
}

export function useLikePost() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: async (postId: number) => {
      const response = await fetch(`/api/posts/${postId}/like`, {
        method: 'POST',
      })
      if (!response.ok) throw new Error('Failed to like post')
      return response.json()
    },

    // onMutate: Chạy trước khi mutation execute
    // Dùng để optimistic update
    onMutate: async (postId) => {
      // 1. Cancel outgoing refetches
      // Tránh refetch ghi đè optimistic update
      await queryClient.cancelQueries({ queryKey: ['post', postId] })

      // 2. Snapshot previous value
      // Lưu lại data cũ để rollback nếu mutation fail
      const previousPost = queryClient.getQueryData<Post>(['post', postId])

      // 3. Optimistically update cache
      queryClient.setQueryData<Post>(['post', postId], (old) => {
        if (!old) return old
        return {
          ...old,
          likes: old.likes + 1, // Tăng likes ngay lập tức
        }
      })

      // 4. Return context với previous value
      // Context này sẽ được pass vào onError và onSettled
      return { previousPost }
    },

    // onError: Rollback nếu mutation fail
    onError: (err, postId, context) => {
      // Restore previous value từ context
      if (context?.previousPost) {
        queryClient.setQueryData(['post', postId], context.previousPost)
      }

      console.error('Failed to like post:', err)
    },

    // onSettled: Chạy sau khi mutation complete (success hoặc error)
    onSettled: (data, error, postId) => {
      // Refetch để đảm bảo sync với server
      queryClient.invalidateQueries({ queryKey: ['post', postId] })
    },
  })
}

// src/components/PostCard.tsx
import { useLikePost } from '../hooks/useLikePost'

interface PostCardProps {
  post: Post
}

export function PostCard({ post }: PostCardProps) {
  const likeMutation = useLikePost()

  return (
    <div>
      <h3>{post.title}</h3>
      <button
        onClick={() => likeMutation.mutate(post.id)}
        disabled={likeMutation.isPending}
      >
        ❤️ {post.likes}
      </button>
    </div>
  )
}
```

**Giải thích chi tiết:**

1. **onMutate workflow**:
   - Cancel queries để tránh race conditions
   - Snapshot current data để có thể rollback
   - Update cache optimistically
   - Return context cho onError và onSettled

2. **cancelQueries**:
   - Hủy tất cả refetch requests đang pending
   - Đảm bảo optimistic update không bị ghi đè

3. **setQueryData**:
   - Update cache trực tiếp
   - Nhận updater function với old data
   - Phải return immutable data mới

4. **Rollback strategy**:
   - onError restore previous data
   - onSettled refetch để sync với server

#### Example: Optimistic update cho todo list

```typescript
// src/hooks/useTodoMutations.ts
import { useMutation, useQueryClient } from '@tanstack/react-query'

interface Todo {
  id: string
  text: string
  completed: boolean
}

export function useAddTodo() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: async (text: string) => {
      const response = await fetch('/api/todos', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ text }),
      })
      return response.json()
    },

    onMutate: async (text) => {
      await queryClient.cancelQueries({ queryKey: ['todos'] })

      const previousTodos = queryClient.getQueryData<Todo[]>(['todos'])

      // Tạo optimistic todo với temporary ID
      const optimisticTodo: Todo = {
        id: `temp-${Date.now()}`,
        text,
        completed: false,
      }

      // Add optimistic todo vào list
      queryClient.setQueryData<Todo[]>(['todos'], (old = []) => [
        ...old,
        optimisticTodo,
      ])

      return { previousTodos, optimisticTodo }
    },

    onSuccess: (newTodo, text, context) => {
      // Replace optimistic todo với real todo từ server
      queryClient.setQueryData<Todo[]>(['todos'], (old = []) =>
        old.map((todo) =>
          todo.id === context.optimisticTodo.id ? newTodo : todo
        )
      )
    },

    onError: (err, text, context) => {
      if (context?.previousTodos) {
        queryClient.setQueryData(['todos'], context.previousTodos)
      }
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] })
    },
  })
}

export function useToggleTodo() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: async ({ id, completed }: { id: string; completed: boolean }) => {
      const response = await fetch(`/api/todos/${id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ completed }),
      })
      return response.json()
    },

    onMutate: async ({ id, completed }) => {
      await queryClient.cancelQueries({ queryKey: ['todos'] })

      const previousTodos = queryClient.getQueryData<Todo[]>(['todos'])

      // Toggle completed status optimistically
      queryClient.setQueryData<Todo[]>(['todos'], (old = []) =>
        old.map((todo) =>
          todo.id === id ? { ...todo, completed } : todo
        )
      )

      return { previousTodos }
    },

    onError: (err, variables, context) => {
      if (context?.previousTodos) {
        queryClient.setQueryData(['todos'], context.previousTodos)
      }
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] })
    },
  })
}
```

### 3. Pagination

#### Example 1: Basic pagination

```typescript
// src/hooks/usePaginatedPosts.ts
import { useQuery } from '@tanstack/react-query'
import { useState } from 'react'

interface PaginatedResponse<T> {
  data: T[]
  total: number
  page: number
  pageSize: number
  totalPages: number
}

async function fetchPosts(page: number, pageSize: number): Promise<PaginatedResponse<Post>> {
  const response = await fetch(
    `/api/posts?page=${page}&pageSize=${pageSize}`
  )
  return response.json()
}

export function usePaginatedPosts(pageSize: number = 10) {
  const [page, setPage] = useState(1)

  const query = useQuery({
    // Query key bao gồm page và pageSize
    queryKey: ['posts', 'paginated', { page, pageSize }],
    queryFn: () => fetchPosts(page, pageSize),

    // placeholderData: Giữ previous data khi fetching page mới
    // Tránh UI blink khi chuyển page
    placeholderData: (previousData) => previousData,

    // staleTime: Keep data fresh trong 30 giây
    staleTime: 1000 * 30,
  })

  return {
    ...query,
    page,
    setPage,
    hasNextPage: query.data ? page < query.data.totalPages : false,
    hasPreviousPage: page > 1,
    nextPage: () => setPage((old) => old + 1),
    previousPage: () => setPage((old) => Math.max(old - 1, 1)),
    goToPage: (newPage: number) => setPage(newPage),
  }
}

// src/components/PaginatedPostsList.tsx
export function PaginatedPostsList() {
  const {
    data,
    isLoading,
    isError,
    error,
    page,
    hasNextPage,
    hasPreviousPage,
    nextPage,
    previousPage,
    goToPage,
  } = usePaginatedPosts(10)

  if (isLoading) return <div>Loading...</div>
  if (isError) return <div>Error: {error.message}</div>
  if (!data) return null

  return (
    <div>
      <h1>Posts (Page {page} of {data.totalPages})</h1>

      <ul>
        {data.data.map((post) => (
          <li key={post.id}>
            <h3>{post.title}</h3>
            <p>{post.body}</p>
          </li>
        ))}
      </ul>

      <div className="pagination">
        <button
          onClick={previousPage}
          disabled={!hasPreviousPage}
        >
          Previous
        </button>

        <span>Page {page} of {data.totalPages}</span>

        <button
          onClick={nextPage}
          disabled={!hasNextPage}
        >
          Next
        </button>
      </div>

      {/* Page numbers */}
      <div className="page-numbers">
        {Array.from({ length: data.totalPages }, (_, i) => i + 1).map((pageNum) => (
          <button
            key={pageNum}
            onClick={() => goToPage(pageNum)}
            disabled={pageNum === page}
            className={pageNum === page ? 'active' : ''}
          >
            {pageNum}
          </button>
        ))}
      </div>
    </div>
  )
}
```

**Giải thích chi tiết:**

1. **placeholderData**:
   - Giữ previous data khi fetching new data
   - Tránh UI blink/flash khi chuyển page
   - User vẫn thấy old data cho đến khi new data ready

2. **Query key với pagination**:
   - Bao gồm page và pageSize trong query key
   - Mỗi page có cache riêng
   - Chuyển page nhanh nếu đã cached

3. **Custom hook pattern**:
   - Encapsulate pagination logic
   - Expose helper functions (nextPage, previousPage, etc.)
   - Reusable across components

### 4. Infinite Queries (Load More / Infinite Scroll)

#### Example: Infinite scroll posts

```typescript
// src/hooks/useInfinitePosts.ts
import { useInfiniteQuery } from '@tanstack/react-query'

interface PostsPage {
  posts: Post[]
  nextCursor: number | null
  hasMore: boolean
}

async function fetchPostsPage({ pageParam = 0 }): Promise<PostsPage> {
  const response = await fetch(`/api/posts?cursor=${pageParam}&limit=10`)
  return response.json()
}

export function useInfinitePosts() {
  return useInfiniteQuery({
    queryKey: ['posts', 'infinite'],
    queryFn: fetchPostsPage,

    // getNextPageParam: Xác định cursor cho page tiếp theo
    // Return undefined nếu không còn page nào
    getNextPageParam: (lastPage) => {
      return lastPage.hasMore ? lastPage.nextCursor : undefined
    },

    // getPreviousPageParam: Xác định cursor cho page trước (optional)
    getPreviousPageParam: (firstPage) => {
      // Implement nếu cần bi-directional infinite scroll
      return undefined
    },

    // initialPageParam: Cursor ban đầu
    initialPageParam: 0,

    // maxPages: Giới hạn số pages được cache (optional)
    maxPages: 5, // Chỉ giữ 5 pages gần nhất
  })
}

// src/components/InfinitePostsList.tsx
import { useInfinitePosts } from '../hooks/useInfinitePosts'
import { useEffect, useRef } from 'react'

export function InfinitePostsList() {
  const {
    data,
    isLoading,
    isError,
    error,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfinitePosts()

  // Intersection Observer cho infinite scroll
  const observerTarget = useRef<HTMLDivElement>(null)

  useEffect(() => {
    const observer = new IntersectionObserver(
      (entries) => {
        // Khi scroll đến bottom, fetch next page
        if (entries[0].isIntersecting && hasNextPage && !isFetchingNextPage) {
          fetchNextPage()
        }
      },
      { threshold: 1.0 }
    )

    if (observerTarget.current) {
      observer.observe(observerTarget.current)
    }

    return () => observer.disconnect()
  }, [fetchNextPage, hasNextPage, isFetchingNextPage])

  if (isLoading) return <div>Loading...</div>
  if (isError) return <div>Error: {error.message}</div>
  if (!data) return null

  return (
    <div>
      <h1>Infinite Posts</h1>

      {/* data.pages là array của tất cả pages đã fetch */}
      {data.pages.map((page, pageIndex) => (
        <div key={pageIndex}>
          {page.posts.map((post) => (
            <article key={post.id}>
              <h3>{post.title}</h3>
              <p>{post.body}</p>
            </article>
          ))}
        </div>
      ))}

      {/* Observer target để trigger load more */}
      <div ref={observerTarget} style={{ height: '20px' }} />

      {/* Loading indicator */}
      {isFetchingNextPage && <div>Loading more...</div>}

      {/* End message */}
      {!hasNextPage && <div>No more posts to load</div>}

      {/* Manual load more button (alternative to infinite scroll) */}
      <button
        onClick={() => fetchNextPage()}
        disabled={!hasNextPage || isFetchingNextPage}
      >
        {isFetchingNextPage
          ? 'Loading more...'
          : hasNextPage
          ? 'Load More'
          : 'Nothing more to load'}
      </button>
    </div>
  )
}
```

**Giải thích chi tiết:**

1. **useInfiniteQuery**:
   - Dùng cho infinite scroll hoặc load more patterns
   - Tự động quản lý multiple pages
   - `data.pages` chứa array của tất cả pages

2. **getNextPageParam**:
   - Function xác định cursor/offset cho page tiếp theo
   - Nhận `lastPage` và `allPages` làm arguments
   - Return `undefined` khi không còn pages

3. **fetchNextPage**:
   - Trigger fetch page tiếp theo
   - Chỉ hoạt động khi `hasNextPage = true`
   - Có thể gọi từ button click hoặc scroll event

4. **Intersection Observer**:
   - Detect khi user scroll đến bottom
   - Tự động trigger `fetchNextPage`
   - Performance tốt hơn scroll event listeners

### 5. Parallel Queries

#### Example: Fetch multiple resources đồng thời

```typescript
// src/hooks/usePostWithComments.ts
import { useQuery, useQueries } from '@tanstack/react-query'

// Pattern 1: Multiple useQuery hooks
export function usePostWithComments(postId: number) {
  // Fetch post
  const postQuery = useQuery({
    queryKey: ['post', postId],
    queryFn: () => fetchPostById(postId),
  })

  // Fetch comments (dependent query)
  const commentsQuery = useQuery({
    queryKey: ['comments', postId],
    queryFn: () => fetchCommentsByPostId(postId),
    // Chỉ fetch khi có postId
    enabled: !!postId,
  })

  return {
    post: postQuery.data,
    comments: commentsQuery.data,
    isLoading: postQuery.isLoading || commentsQuery.isLoading,
    isError: postQuery.isError || commentsQuery.isError,
    error: postQuery.error || commentsQuery.error,
  }
}

// Pattern 2: useQueries cho dynamic số lượng queries
export function useMultiplePosts(postIds: number[]) {
  // useQueries fetch multiple queries đồng thời
  const queries = useQueries({
    queries: postIds.map((id) => ({
      queryKey: ['post', id],
      queryFn: () => fetchPostById(id),
      staleTime: 1000 * 60 * 5,
    })),
  })

  return {
    posts: queries.map((q) => q.data).filter(Boolean),
    isLoading: queries.some((q) => q.isLoading),
    isError: queries.some((q) => q.isError),
    errors: queries.map((q) => q.error).filter(Boolean),
  }
}

// src/components/PostWithComments.tsx
export function PostWithComments({ postId }: { postId: number }) {
  const { post, comments, isLoading, isError, error } = usePostWithComments(postId)

  if (isLoading) return <div>Loading...</div>
  if (isError) return <div>Error: {error?.message}</div>

  return (
    <div>
      <article>
        <h1>{post?.title}</h1>
        <p>{post?.body}</p>
      </article>

      <section>
        <h2>Comments ({comments?.length || 0})</h2>
        {comments?.map((comment) => (
          <div key={comment.id}>
            <strong>{comment.name}</strong>
            <p>{comment.body}</p>
          </div>
        ))}
      </section>
    </div>
  )
}
```

### 6. Dependent Queries

```typescript
// src/hooks/useUserPosts.ts
import { useQuery } from '@tanstack/react-query'

export function useUserPosts(email: string) {
  // Query 1: Fetch user by email
  const userQuery = useQuery({
    queryKey: ['user', email],
    queryFn: () => fetchUserByEmail(email),
    enabled: !!email, // Chỉ fetch khi có email
  })

  const userId = userQuery.data?.id

  // Query 2: Fetch posts by userId (dependent on userQuery)
  const postsQuery = useQuery({
    queryKey: ['posts', 'user', userId],
    queryFn: () => fetchPostsByUserId(userId!),
    // enabled: Chỉ fetch khi có userId
    // Query này phụ thuộc vào kết quả của userQuery
    enabled: !!userId,
  })

  return {
    user: userQuery.data,
    posts: postsQuery.data,
    isLoadingUser: userQuery.isLoading,
    isLoadingPosts: postsQuery.isLoading,
    isLoading: userQuery.isLoading || postsQuery.isLoading,
    isError: userQuery.isError || postsQuery.isError,
    error: userQuery.error || postsQuery.error,
  }
}
```

**Giải thích:**
- Query thứ 2 chỉ chạy khi query thứ 1 thành công và có `userId`
- `enabled` option control khi nào query được execute
- Tránh unnecessary requests khi dependencies chưa ready

### 7. Error Handling và Retry Logic

```typescript
// src/hooks/usePostsWithRetry.ts
import { useQuery } from '@tanstack/react-query'

export function usePostsWithRetry() {
  return useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,

    // retry: Số lần retry khi fail
    retry: 3,

    // retryDelay: Delay giữa các retry attempts
    // Exponential backoff: 1s, 2s, 4s, 8s, ...
    retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),

    // retryOnMount: Retry khi component mount lại
    retryOnMount: true,

    // Custom retry logic
    retry: (failureCount, error: any) => {
      // Không retry cho 404 errors
      if (error.status === 404) return false

      // Retry tối đa 3 lần cho các errors khác
      return failureCount < 3
    },

    // onError callback
    onError: (error: any) => {
      console.error('Query failed:', error)

      // Show toast notification
      if (error.status === 500) {
        toast.error('Server error. Please try again later.')
      } else if (error.status === 404) {
        toast.error('Resource not found.')
      } else {
        toast.error('An error occurred. Please try again.')
      }
    },
  })
}

// Global error handling
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: 3,
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),

      // Global error handler
      onError: (error: any) => {
        console.error('Global query error:', error)
      },
    },
    mutations: {
      retry: 1,
      onError: (error: any) => {
        console.error('Global mutation error:', error)
      },
    },
  },
})
```

### 8. Best Practices cho Middle Level

1. **Sử dụng optimistic updates cho better UX**
   ```typescript
   // Update UI ngay lập tức, rollback nếu fail
   onMutate: async (newData) => {
     await queryClient.cancelQueries({ queryKey: ['data'] })
     const previous = queryClient.getQueryData(['data'])
     queryClient.setQueryData(['data'], newData)
     return { previous }
   }
   ```

2. **Implement proper error boundaries**
   ```typescript
   // Component-level error handling
   if (isError) {
     return <ErrorBoundary error={error} />
   }
   ```

3. **Use placeholderData cho smooth transitions**
   ```typescript
   placeholderData: (previousData) => previousData
   ```

4. **Prefetch data cho better performance**
   ```typescript
   // Prefetch khi hover
   const handleMouseEnter = () => {
     queryClient.prefetchQuery({
       queryKey: ['post', postId],
       queryFn: () => fetchPostById(postId),
     })
   }
   ```

5. **Organize query keys systematically**
   ```typescript
   // Query key factory pattern
   const postKeys = {
     all: ['posts'] as const,
     lists: () => [...postKeys.all, 'list'] as const,
     list: (filters: string) => [...postKeys.lists(), { filters }] as const,
     details: () => [...postKeys.all, 'detail'] as const,
     detail: (id: number) => [...postKeys.details(), id] as const,
   }
   ```

---

## Senior Level - Nâng Cao

### 1. Advanced Caching Strategies

#### Cache Time vs Stale Time

```typescript
// Understanding the difference
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // staleTime: Thời gian data được coi là "fresh"
      // Trong thời gian này, query sẽ KHÔNG refetch
      staleTime: 1000 * 60 * 5, // 5 minutes

      // gcTime (garbage collection time): Thời gian data được giữ trong cache
      // Sau thời gian này, data sẽ bị xóa khỏi cache nếu không có observers
      gcTime: 1000 * 60 * 10, // 10 minutes
    },
  },
})
```

**Giải thích chi tiết:**

1. **staleTime**:
   - Thời gian data được coi là "fresh"
   - Trong thời gian này, query KHÔNG tự động refetch
   - Default: 0 (data luôn stale ngay lập tức)

2. **gcTime** (cacheTime trong v4):
   - Thời gian data được giữ trong cache sau khi không còn observers
   - Sau thời gian này, data bị garbage collected
   - Default: 5 minutes

3. **Lifecycle**:
   ```
   Query mounted → Fetch data → Data fresh (staleTime)
   → Data stale → Still in cache (gcTime)
   → Cache expired → Data removed
   ```

#### Cache Strategy Examples

```typescript
// Strategy 1: Static data (rarely changes)
const { data } = useQuery({
  queryKey: ['countries'],
  queryFn: fetchCountries,
  staleTime: Infinity, // Never stale
  gcTime: Infinity, // Never garbage collected
})

// Strategy 2: Real-time data
const { data } = useQuery({
  queryKey: ['live-prices'],
  queryFn: fetchLivePrices,
  staleTime: 0, // Always stale
  refetchInterval: 1000, // Refetch every second
})

// Strategy 3: User-specific data
const { data } = useQuery({
  queryKey: ['user-profile'],
  queryFn: fetchUserProfile,
  staleTime: 1000 * 60 * 5, // Fresh for 5 minutes
  gcTime: 1000 * 60 * 30, // Keep in cache for 30 minutes
})
```

### 2. Integration với State Management (Zustand)

```typescript
// src/stores/usePostsStore.ts
import { create } from 'zustand'
import { useQuery, useQueryClient } from '@tanstack/react-query'
import { queryKeys } from '@/lib/queryKeys'

interface PostsStore {
  selectedPostId: number | null
  setSelectedPostId: (id: number | null) => void
  filters: PostFilters
  setFilters: (filters: PostFilters) => void
}

export const usePostsStore = create<PostsStore>((set) => ({
  selectedPostId: null,
  setSelectedPostId: (id) => set({ selectedPostId: id }),
  filters: {},
  setFilters: (filters) => set({ filters }),
}))

// Usage: Combine Zustand với TanStack Query
export function usePostsWithFilters() {
  const filters = usePostsStore((state) => state.filters)

  return useQuery({
    queryKey: queryKeys.posts.list(filters),
    queryFn: () => fetchPosts(filters),
  })
}

export function useSelectedPost() {
  const selectedPostId = usePostsStore((state) => state.selectedPostId)

  return useQuery({
    queryKey: queryKeys.posts.detail(selectedPostId!),
    queryFn: () => fetchPostById(selectedPostId!),
    enabled: !!selectedPostId,
  })
}
```

**Best Practice**:
- Zustand cho UI state (filters, selections, modals)
- TanStack Query cho server state (API data)
- Không duplicate server state trong Zustand

### 3. Monitoring và Debugging trong Production

#### Custom DevTools Integration

```typescript
// src/lib/monitoring.ts
import { QueryClient } from '@tanstack/react-query'

export function setupQueryMonitoring(queryClient: QueryClient) {
  // Log slow queries
  queryClient.getQueryCache().subscribe((event) => {
    if (event.type === 'updated' && event.action.type === 'success') {
      const query = event.query
      const duration = Date.now() - (query.state.dataUpdatedAt || 0)

      if (duration > 3000) {
        console.warn('Slow query detected:', {
          queryKey: query.queryKey,
          duration,
        })

        // Send to monitoring service
        sendToMonitoring({
          type: 'slow_query',
          queryKey: query.queryKey,
          duration,
        })
      }
    }
  })

  // Track failed queries
  queryClient.getQueryCache().subscribe((event) => {
    if (event.type === 'updated' && event.action.type === 'error') {
      const query = event.query

      console.error('Query failed:', {
        queryKey: query.queryKey,
        error: query.state.error,
      })

      // Send to error tracking (Sentry, etc.)
      sendToErrorTracking({
        type: 'query_error',
        queryKey: query.queryKey,
        error: query.state.error,
      })
    }
  })
}
```

---

## Tài Liệu Tham Khảo

### Official Documentation
- **TanStack Query Official Docs**: https://tanstack.com/query/latest
- **React Query v5 Migration Guide**: https://tanstack.com/query/latest/docs/react/guides/migrating-to-v5
- **API Reference**: https://tanstack.com/query/latest/docs/react/reference/useQuery

### Community Resources
- **TanStack Query GitHub**: https://github.com/TanStack/query
- **Discord Community**: https://discord.com/invite/WrRKjPJ
- **Stack Overflow Tag**: https://stackoverflow.com/questions/tagged/react-query

### Related Libraries
- **TanStack Router**: https://tanstack.com/router/latest
- **TanStack Table**: https://tanstack.com/table/latest
- **TanStack Virtual**: https://tanstack.com/virtual/latest

### Advanced Topics
- **React Query Best Practices**: https://tkdodo.eu/blog/practical-react-query
- **Effective React Query Keys**: https://tkdodo.eu/blog/effective-react-query-keys
- **React Query and TypeScript**: https://tkdodo.eu/blog/react-query-and-type-script

### Video Tutorials
- **TanStack Query Crash Course**: https://www.youtube.com/watch?v=8K1N3fE-cDs
- **Advanced React Query Patterns**: https://www.youtube.com/watch?v=DocXo3gqGdI

---

## Kết Luận

TanStack Query là một thư viện mạnh mẽ giúp quản lý server state trong React applications. Qua tài liệu này, bạn đã học được:

### Junior Level
- ✅ Cài đặt và cấu hình cơ bản
- ✅ Sử dụng useQuery và useMutation
- ✅ Hiểu về caching và query keys
- ✅ Handle loading và error states
- ✅ Tránh các pitfalls phổ biến

### Middle Level
- ✅ Query invalidation strategies
- ✅ Optimistic updates
- ✅ Pagination và infinite queries
- ✅ Parallel và dependent queries
- ✅ Advanced error handling

### Senior Level
- ✅ Advanced caching strategies
- ✅ Custom hooks architecture
- ✅ Performance optimization
- ✅ Testing strategies
- ✅ Integration với state management

### Principal Level
- ✅ Enterprise-scale architecture
- ✅ SSR/SSG integration
- ✅ Monitoring và debugging
- ✅ Migration strategies
- ✅ Production best practices

**Lưu ý quan trọng:**
- Luôn bắt đầu đơn giản và tăng dần complexity
- Sử dụng TypeScript để type safety
- Test thoroughly trước khi deploy
- Monitor performance trong production
- Keep learning và cập nhật với latest versions

**Next Steps:**
1. Thực hành với các examples trong tài liệu
2. Đọc thêm advanced patterns trong file `Advanced-Patterns.md`
3. Tham khảo Principal Level patterns trong `Principal-Level-Patterns.md`
4. Join TanStack Discord community để học hỏi thêm
5. Contribute back to the community

---

**Tài liệu được tạo bởi:** AI Assistant
**Ngày tạo:** 2025-11-10
**Version:** TanStack Query v5
**Ngôn ngữ:** Tiếng Việt với TypeScript examples
```

