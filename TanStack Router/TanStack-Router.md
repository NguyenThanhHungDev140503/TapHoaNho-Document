# TanStack Router - Type-Safe Routing cho React

## Mục Lục

- [Junior Level - Cơ Bản](#junior-level---cơ-bản)
  - [1. Giới Thiệu về TanStack Router](#1-giới-thiệu-về-tanstack-router)
  - [2. Installation và Setup](#2-installation-và-setup)
  - [3. Route Definition](#3-route-definition)
  - [4. Navigation và Link](#4-navigation-và-link)
  - [5. Route Parameters](#5-route-parameters)
  - [6. Basic Layouts](#6-basic-layouts)

- [Middle Level - Trung Cấp](#middle-level---trung-cấp)
  - [1. Nested Routes](#1-nested-routes)
  - [2. Route Loaders](#2-route-loaders)
  - [3. Route Guards](#3-route-guards)
  - [4. Search Params](#4-search-params)
  - [5. Route Context](#5-route-context)
  - [6. Error Handling](#6-error-handling)
  - [7. Code Splitting](#7-code-splitting)

- [Senior Level](#senior-level) - Xem file `Advanced-Router-Patterns.md`
- [Principal Level](#principal-level) - Xem file `Principal-Router-Patterns.md`

---

## Junior Level - Cơ Bản

### 1. Giới Thiệu về TanStack Router

#### TanStack Router là gì?

TanStack Router là một routing library hiện đại cho React với những đặc điểm nổi bật:

- **100% Type-Safe**: TypeScript support hoàn toàn, type inference tự động
- **Nested Routing**: Hỗ trợ routes lồng nhau một cách tự nhiên
- **Built-in Data Loading**: Tích hợp sẵn data fetching với loaders
- **Search Params Management**: Quản lý query parameters type-safe
- **File-Based Routing**: Hỗ trợ cả code-based và file-based routing
- **SSR/SSG Ready**: Tương thích với Server-Side Rendering

#### So Sánh với React Router

| Feature | TanStack Router | React Router v6 |
|---------|----------------|-----------------|
| Type Safety | ✅ 100% type-safe | ⚠️ Partial |
| Search Params | ✅ Type-safe, validated | ❌ String-based |
| Data Loading | ✅ Built-in loaders | ⚠️ Requires loaders |
| Nested Routes | ✅ First-class support | ✅ Supported |
| File-Based Routing | ✅ Built-in | ❌ Requires plugin |
| Bundle Size | ~15KB | ~10KB |
| Learning Curve | Medium | Easy |

**Khi nào dùng TanStack Router:**
- Dự án TypeScript cần type safety cao
- Cần quản lý search params phức tạp
- Muốn data loading tích hợp sẵn
- Dự án lớn cần scalability

**Khi nào dùng React Router:**
- Dự án nhỏ, đơn giản
- Team chưa quen TypeScript
- Cần bundle size nhỏ nhất
- Đã có codebase với React Router

### 2. Installation và Setup

#### Cài Đặt

```bash
# NPM
npm install @tanstack/react-router

# Yarn
yarn add @tanstack/react-router

# PNPM
pnpm add @tanstack/react-router

# Optional: DevTools
npm install @tanstack/router-devtools
```

#### Setup Cơ Bản

```typescript
// Example 1: Basic router setup
// src/router.tsx
import { createRouter, createRootRoute, createRoute } from '@tanstack/react-router'
import { Outlet } from '@tanstack/react-router'

// 1. Create root route
const rootRoute = createRootRoute({
  component: () => (
    <div>
      <nav>
        <h1>My App</h1>
      </nav>
      <hr />
      <Outlet /> {/* Child routes render here */}
    </div>
  ),
})

// 2. Create index route
const indexRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/',
  component: () => <div>Welcome to Home Page!</div>,
})

// 3. Create about route
const aboutRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/about',
  component: () => <div>About Us</div>,
})

// 4. Create route tree
const routeTree = rootRoute.addChildren([indexRoute, aboutRoute])

// 5. Create router
export const router = createRouter({ routeTree })

// 6. Register router for type safety
declare module '@tanstack/react-router' {
  interface Register {
    router: typeof router
  }
}
```

**Giải thích:**
- `createRootRoute()`: Tạo root route, là parent của tất cả routes
- `<Outlet />`: Component đặc biệt để render child routes
- `getParentRoute()`: Xác định parent route
- `path`: URL path cho route
- `addChildren()`: Thêm child routes vào parent
- `declare module`: Register router để TypeScript biết types

#### Integrate với React App

```typescript
// Example 2: App.tsx
import { RouterProvider } from '@tanstack/react-router'
import { router } from './router'

function App() {
  return <RouterProvider router={router} />
}

export default App
```

### 3. Route Definition

#### Code-Based Routes

```typescript
// Example 1: Defining routes with createRoute
import { createRoute } from '@tanstack/react-router'

// Products list route
const productsRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/products',
  component: ProductsPage,
})

function ProductsPage() {
  return (
    <div>
      <h1>Products</h1>
      <ul>
        <li>Product 1</li>
        <li>Product 2</li>
      </ul>
    </div>
  )
}
```

#### File-Based Routes

TanStack Router hỗ trợ file-based routing tự động generate routes từ file structure:

```
src/routes/
├── __root.tsx          # Root route
├── index.tsx           # / route
├── about.tsx           # /about route
├── products.tsx        # /products route
└── products.$id.tsx    # /products/:id route
```

```typescript
// Example 2: File-based route definition
// src/routes/products.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/products')({
  component: ProductsComponent,
})

function ProductsComponent() {
  return <div>Products Page</div>
}
```

**File Naming Conventions:**
- `index.tsx` → `/` (index route)
- `about.tsx` → `/about` (static route)
- `$id.tsx` → `/:id` (dynamic parameter)
- `_layout.tsx` → Layout route (không tạo path)
- `$.tsx` → `/*` (catch-all/wildcard route)

### 4. Navigation và Link

#### Link Component

```typescript
// Example 1: Basic navigation với Link
import { Link } from '@tanstack/react-router'

function Navigation() {
  return (
    <nav>
      {/* Basic link */}
      <Link to="/">Home</Link>

      {/* Link với params */}
      <Link
        to="/products/$id"
        params={{ id: '123' }}
      >
        Product 123
      </Link>

      {/* Link với search params */}
      <Link
        to="/products"
        search={{ category: 'electronics', page: 1 }}
      >
        Electronics
      </Link>

      {/* Active link styling */}
      <Link
        to="/about"
        activeProps={{
          className: 'font-bold text-blue-600',
        }}
        inactiveProps={{
          className: 'text-gray-600',
        }}
      >
        About
      </Link>
    </nav>
  )
}
```

**Giải thích:**
- `to`: Target route path (type-safe!)
- `params`: Route parameters (validated by TypeScript)
- `search`: Query parameters (type-safe)
- `activeProps`: Props khi link active
- `inactiveProps`: Props khi link inactive

#### Programmatic Navigation

```typescript
// Example 2: Navigate programmatically
import { useNavigate } from '@tanstack/react-router'

function ProductCard({ productId }: { productId: string }) {
  const navigate = useNavigate()

  const handleClick = () => {
    // Navigate to product detail
    navigate({
      to: '/products/$id',
      params: { id: productId },
    })
  }

  const handleSearch = () => {
    // Navigate với search params
    navigate({
      to: '/products',
      search: { category: 'electronics' },
    })
  }

  const handleBack = () => {
    // Navigate back
    navigate({ to: '..' }) // Parent route
  }

  return (
    <div>
      <button onClick={handleClick}>View Details</button>
      <button onClick={handleSearch}>Search Electronics</button>
      <button onClick={handleBack}>Go Back</button>
    </div>
  )
}
```

#### Relative Navigation

```typescript
// Example 3: Relative paths
function PostDetail() {
  return (
    <div>
      {/* Current route: /posts/123 */}

      {/* Reload current route */}
      <Link to=".">Reload</Link>

      {/* Navigate to parent (/posts) */}
      <Link to="..">Back to Posts</Link>

      {/* Navigate to sibling route */}
      <Link to="../456">Next Post</Link>

      {/* Navigate to root */}
      <Link to="/">Home</Link>
    </div>
  )
}
```

### 5. Route Parameters

#### Dynamic Route Parameters

```typescript
// Example 1: Single parameter
// Route: /products/$id.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/products/$id')({
  component: ProductDetail,
})

function ProductDetail() {
  // Get route params (type-safe!)
  const { id } = Route.useParams()

  return (
    <div>
      <h1>Product ID: {id}</h1>
    </div>
  )
}
```

```typescript
// Example 2: Multiple parameters
// Route: /users/$userId/posts/$postId.tsx
export const Route = createFileRoute('/users/$userId/posts/$postId')({
  component: UserPost,
})

function UserPost() {
  const { userId, postId } = Route.useParams()

  return (
    <div>
      <h1>User: {userId}</h1>
      <h2>Post: {postId}</h2>
    </div>
  )
}
```

#### Query Parameters (Search Params)

```typescript
// Example 3: Search params với validation
import { createFileRoute } from '@tanstack/react-router'
import { z } from 'zod'
import { zodValidator } from '@tanstack/zod-adapter'

// Define search schema
const productSearchSchema = z.object({
  category: z.string().optional(),
  page: z.number().int().positive().default(1),
  sort: z.enum(['price', 'name', 'date']).default('name'),
})

export const Route = createFileRoute('/products')({
  validateSearch: zodValidator(productSearchSchema),
  component: ProductsPage,
})

function ProductsPage() {
  // Get validated search params (type-safe!)
  const { category, page, sort } = Route.useSearch()

  return (
    <div>
      <h1>Products</h1>
      <p>Category: {category || 'All'}</p>
      <p>Page: {page}</p>
      <p>Sort by: {sort}</p>

      {/* Update search params */}
      <Link
        to="/products"
        search={{ category: 'electronics', page: 1, sort: 'price' }}
      >
        Electronics sorted by price
      </Link>
    </div>
  )
}
```

**Giải thích:**
- `validateSearch`: Validate và parse search params
- `zodValidator`: Adapter cho Zod schema
- `Route.useSearch()`: Get validated search params
- Search params được merge với existing params khi navigate

### 6. Basic Layouts

#### Layout Routes

```typescript
// Example 1: Layout route với shared UI
// src/routes/_layout.tsx
import { createFileRoute, Outlet } from '@tanstack/react-router'

export const Route = createFileRoute('/_layout')({
  component: LayoutComponent,
})

function LayoutComponent() {
  return (
    <div className="min-h-screen">
      {/* Shared header */}
      <header className="bg-blue-600 text-white p-4">
        <nav className="flex gap-4">
          <Link to="/">Home</Link>
          <Link to="/about">About</Link>
          <Link to="/products">Products</Link>
        </nav>
      </header>

      {/* Main content */}
      <main className="container mx-auto p-4">
        <Outlet /> {/* Child routes render here */}
      </main>

      {/* Shared footer */}
      <footer className="bg-gray-800 text-white p-4 text-center">
        © 2024 My App
      </footer>
    </div>
  )
}
```

```typescript
// Example 2: Nested layout
// src/routes/_layout/products.tsx
export const Route = createFileRoute('/_layout/products')({
  component: ProductsPage,
})

// URL: /products
// Renders: LayoutComponent > ProductsPage
```

**Layout Route Naming:**
- `_layout.tsx`: Layout route (underscore prefix)
- Không tạo path segment trong URL
- Chỉ wrap child routes với shared UI

---

## Middle Level - Trung Cấp

### 1. Nested Routes

#### Route Hierarchy

```typescript
// Example 1: Nested route structure
// File structure:
// routes/
// ├── __root.tsx
// ├── posts.tsx           # /posts
// ├── posts.index.tsx     # /posts/ (index)
// └── posts.$id.tsx       # /posts/:id

// posts.tsx - Parent route
import { createFileRoute, Outlet } from '@tanstack/react-router'

export const Route = createFileRoute('/posts')({
  component: PostsLayout,
})

function PostsLayout() {
  return (
    <div className="posts-layout">
      <aside className="sidebar">
        <h2>Posts</h2>
        <nav>
          <Link to="/posts">All Posts</Link>
          <Link to="/posts/new">New Post</Link>
        </nav>
      </aside>

      <main className="content">
        <Outlet /> {/* Child routes render here */}
      </main>
    </div>
  )
}
```

```typescript
// posts.index.tsx - Index route
export const Route = createFileRoute('/posts/')({
  component: PostsList,
})

function PostsList() {
  return (
    <div>
      <h1>All Posts</h1>
      <ul>
        <li><Link to="/posts/$id" params={{ id: '1' }}>Post 1</Link></li>
        <li><Link to="/posts/$id" params={{ id: '2' }}>Post 2</Link></li>
      </ul>
    </div>
  )
}
```

```typescript
// posts.$id.tsx - Dynamic route
export const Route = createFileRoute('/posts/$id')({
  component: PostDetail,
})

function PostDetail() {
  const { id } = Route.useParams()

  return (
    <div>
      <h1>Post {id}</h1>
      <p>Post content...</p>
    </div>
  )
}
```

**Route Matching:**
- `/posts` → PostsLayout
- `/posts/` → PostsLayout > PostsList
- `/posts/123` → PostsLayout > PostDetail

### 2. Route Loaders

#### Data Loading với Loaders

```typescript
// Example 1: Basic loader
import { createFileRoute } from '@tanstack/react-router'

interface Post {
  id: string
  title: string
  content: string
}

async function fetchPost(id: string): Promise<Post> {
  const response = await fetch(`/api/posts/${id}`)
  if (!response.ok) throw new Error('Failed to fetch post')
  return response.json()
}

export const Route = createFileRoute('/posts/$id')({
  // Loader runs before component renders
  loader: async ({ params }) => {
    const post = await fetchPost(params.id)
    return { post }
  },
  component: PostDetail,
})

function PostDetail() {
  // Get loader data (type-safe!)
  const { post } = Route.useLoaderData()

  return (
    <div>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </div>
  )
}
```

**Giải thích:**
- `loader`: Function chạy trước khi render component
- Nhận `params`, `search`, `context` làm arguments
- Return data sẽ available qua `useLoaderData()`
- Loader chạy trên cả server (SSR) và client

#### Loader với Dependencies

```typescript
// Example 2: Loader với search params
const productSearchSchema = z.object({
  category: z.string().optional(),
  page: z.number().default(1),
})

export const Route = createFileRoute('/products')({
  validateSearch: zodValidator(productSearchSchema),
  // Loader có access đến validated search params
  loader: async ({ search }) => {
    const products = await fetchProducts({
      category: search.category,
      page: search.page,
    })

    return { products }
  },
  component: ProductsPage,
})

function ProductsPage() {
  const { products } = Route.useLoaderData()
  const search = Route.useSearch()

  return (
    <div>
      <h1>Products - Page {search.page}</h1>
      {search.category && <p>Category: {search.category}</p>}

      <ul>
        {products.map(product => (
          <li key={product.id}>{product.name}</li>
        ))}
      </ul>

      {/* Pagination */}
      <Link
        to="/products"
        search={{ ...search, page: search.page + 1 }}
      >
        Next Page
      </Link>
    </div>
  )
}
```

#### Parallel Data Loading

```typescript
// Example 3: Load multiple data sources
export const Route = createFileRoute('/dashboard')({
  loader: async () => {
    // Load data in parallel
    const [user, stats, notifications] = await Promise.all([
      fetchUser(),
      fetchStats(),
      fetchNotifications(),
    ])

    return {
      user,
      stats,
      notifications,
    }
  },
  component: Dashboard,
})

function Dashboard() {
  const { user, stats, notifications } = Route.useLoaderData()

  return (
    <div>
      <h1>Welcome, {user.name}!</h1>
      <div>Stats: {stats.total}</div>
      <div>Notifications: {notifications.length}</div>
    </div>
  )
}
```

### 3. Route Guards

#### Authentication Guards với beforeLoad

```typescript
// Example 1: Protected route với redirect
import { createFileRoute, redirect } from '@tanstack/react-router'

// Define auth context type
interface AuthContext {
  isAuthenticated: boolean
  user: { id: string; name: string } | null
}

// Protected layout route
export const Route = createFileRoute('/_authenticated')({
  beforeLoad: async ({ context, location }) => {
    const auth = context.auth as AuthContext

    // Check authentication
    if (!auth.isAuthenticated) {
      // Redirect to login, save current location
      throw redirect({
        to: '/login',
        search: {
          redirect: location.href, // Save for post-login redirect
        },
      })
    }

    // Can return additional context
    return {
      user: auth.user,
    }
  },
  component: () => <Outlet />,
})
```

**Giải thích:**
- `beforeLoad`: Hook chạy trước loader và component
- `throw redirect()`: Redirect user đến route khác
- `location.href`: Current URL để redirect sau khi login
- Return value được merge vào route context

#### Login Route với Redirect

```typescript
// Example 2: Login page
import { createFileRoute } from '@tanstack/react-router'
import { useState } from 'react'

const loginSearchSchema = z.object({
  redirect: z.string().default('/'),
})

export const Route = createFileRoute('/login')({
  validateSearch: zodValidator(loginSearchSchema),
  beforeLoad: ({ context, search }) => {
    // Redirect if already authenticated
    if (context.auth.isAuthenticated) {
      throw redirect({ to: search.redirect })
    }
  },
  component: LoginPage,
})

function LoginPage() {
  const { auth } = Route.useRouteContext()
  const { redirect: redirectUrl } = Route.useSearch()
  const navigate = Route.useNavigate()

  const [username, setUsername] = useState('')
  const [password, setPassword] = useState('')
  const [error, setError] = useState('')

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()

    try {
      await auth.login(username, password)
      // Navigate to redirect URL after successful login
      navigate({ to: redirectUrl })
    } catch (err) {
      setError('Invalid credentials')
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <h1>Login</h1>
      {error && <div className="error">{error}</div>}

      <input
        type="text"
        value={username}
        onChange={(e) => setUsername(e.target.value)}
        placeholder="Username"
      />

      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Password"
      />

      <button type="submit">Sign In</button>
    </form>
  )
}
```

#### Role-Based Access Control

```typescript
// Example 3: Role-based guards
interface User {
  id: string
  name: string
  role: 'admin' | 'user' | 'guest'
}

export const Route = createFileRoute('/_authenticated/admin')({
  beforeLoad: ({ context }) => {
    const user = context.user as User

    // Check role
    if (user.role !== 'admin') {
      throw redirect({
        to: '/unauthorized',
      })
    }
  },
  component: AdminPanel,
})

function AdminPanel() {
  return (
    <div>
      <h1>Admin Panel</h1>
      <p>Only admins can see this</p>
    </div>
  )
}
```

### 4. Search Params

#### Advanced Search Params Management

```typescript
// Example 1: Complex search schema
import { z } from 'zod'
import { zodValidator } from '@tanstack/zod-adapter'

const productFiltersSchema = z.object({
  // Text search
  q: z.string().optional(),

  // Category filter
  category: z.enum(['electronics', 'clothing', 'books']).optional(),

  // Price range
  minPrice: z.number().min(0).optional(),
  maxPrice: z.number().min(0).optional(),

  // Sorting
  sortBy: z.enum(['price', 'name', 'date']).default('name'),
  sortOrder: z.enum(['asc', 'desc']).default('asc'),

  // Pagination
  page: z.number().int().positive().default(1),
  perPage: z.number().int().positive().default(20),

  // Multi-select tags
  tags: z.array(z.string()).default([]),
})

export const Route = createFileRoute('/products')({
  validateSearch: zodValidator(productFiltersSchema),
  loader: async ({ search }) => {
    const products = await fetchProducts(search)
    return { products }
  },
  component: ProductsPage,
})

function ProductsPage() {
  const { products } = Route.useLoaderData()
  const search = Route.useSearch()
  const navigate = Route.useNavigate()

  const updateFilters = (updates: Partial<typeof search>) => {
    navigate({
      to: '/products',
      search: { ...search, ...updates, page: 1 }, // Reset page
    })
  }

  return (
    <div>
      {/* Search input */}
      <input
        type="text"
        value={search.q || ''}
        onChange={(e) => updateFilters({ q: e.target.value })}
        placeholder="Search products..."
      />

      {/* Category filter */}
      <select
        value={search.category || ''}
        onChange={(e) => updateFilters({
          category: e.target.value as any
        })}
      >
        <option value="">All Categories</option>
        <option value="electronics">Electronics</option>
        <option value="clothing">Clothing</option>
        <option value="books">Books</option>
      </select>

      {/* Price range */}
      <input
        type="number"
        value={search.minPrice || ''}
        onChange={(e) => updateFilters({
          minPrice: Number(e.target.value)
        })}
        placeholder="Min price"
      />

      <input
        type="number"
        value={search.maxPrice || ''}
        onChange={(e) => updateFilters({
          maxPrice: Number(e.target.value)
        })}
        placeholder="Max price"
      />

      {/* Sort controls */}
      <select
        value={search.sortBy}
        onChange={(e) => updateFilters({ sortBy: e.target.value as any })}
      >
        <option value="name">Name</option>
        <option value="price">Price</option>
        <option value="date">Date</option>
      </select>

      {/* Products list */}
      <div>
        {products.map(product => (
          <div key={product.id}>{product.name}</div>
        ))}
      </div>

      {/* Pagination */}
      <button
        disabled={search.page === 1}
        onClick={() => updateFilters({ page: search.page - 1 })}
      >
        Previous
      </button>

      <span>Page {search.page}</span>

      <button
        onClick={() => updateFilters({ page: search.page + 1 })}
      >
        Next
      </button>
    </div>
  )
}
```

**Best Practices cho Search Params:**
- Luôn validate với schema (Zod, Yup, etc.)
- Provide default values
- Reset page khi thay đổi filters
- Debounce text input updates
- Preserve search params khi navigate

### 5. Route Context

#### Creating và Using Route Context

```typescript
// Example 1: Root route với context
import { createRootRouteWithContext } from '@tanstack/react-router'
import { QueryClient } from '@tanstack/react-query'

interface RouterContext {
  auth: {
    isAuthenticated: boolean
    user: User | null
    login: (username: string, password: string) => Promise<void>
    logout: () => void
  }
  queryClient: QueryClient
}

export const Route = createRootRouteWithContext<RouterContext>()({
  component: RootComponent,
})

function RootComponent() {
  const { auth } = Route.useRouteContext()

  return (
    <div>
      <header>
        {auth.isAuthenticated ? (
          <div>
            <span>Welcome, {auth.user?.name}</span>
            <button onClick={auth.logout}>Logout</button>
          </div>
        ) : (
          <Link to="/login">Login</Link>
        )}
      </header>
      <Outlet />
    </div>
  )
}
```

```typescript
// Example 2: Provide context to router
import { RouterProvider } from '@tanstack/react-router'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { router } from './router'
import { useAuth } from './hooks/useAuth'

const queryClient = new QueryClient()

function App() {
  const auth = useAuth() // Custom auth hook

  return (
    <QueryClientProvider client={queryClient}>
      <RouterProvider
        router={router}
        context={{
          auth,
          queryClient,
        }}
      />
    </QueryClientProvider>
  )
}
```

#### Extending Context trong beforeLoad

```typescript
// Example 3: Add context in beforeLoad
export const Route = createFileRoute('/dashboard')({
  beforeLoad: async ({ context }) => {
    // Fetch additional data
    const permissions = await fetchUserPermissions(context.auth.user.id)

    // Return additional context
    return {
      permissions,
    }
  },
  loader: async ({ context }) => {
    // Access extended context
    const canViewStats = context.permissions.includes('view_stats')

    if (canViewStats) {
      return { stats: await fetchStats() }
    }

    return { stats: null }
  },
  component: Dashboard,
})

function Dashboard() {
  const { permissions } = Route.useRouteContext()
  const { stats } = Route.useLoaderData()

  return (
    <div>
      <h1>Dashboard</h1>
      {permissions.includes('view_stats') && stats && (
        <div>Stats: {stats.total}</div>
      )}
    </div>
  )
}
```

### 6. Error Handling

#### Error Boundaries

```typescript
// Example 1: Route-level error boundary
import { createFileRoute, ErrorComponent } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/$id')({
  loader: async ({ params }) => {
    const post = await fetchPost(params.id)

    if (!post) {
      throw new Error('Post not found')
    }

    return { post }
  },
  errorComponent: ({ error }) => {
    return (
      <div className="error-container">
        <h1>Error Loading Post</h1>
        <p>{error.message}</p>
        <Link to="/posts">Back to Posts</Link>
      </div>
    )
  },
  component: PostDetail,
})
```

#### Custom Error Components

```typescript
// Example 2: Custom error component
function CustomErrorComponent({ error, reset }: {
  error: Error
  reset: () => void
}) {
  return (
    <div className="error-page">
      <h1>Oops! Something went wrong</h1>
      <p className="error-message">{error.message}</p>

      <div className="error-actions">
        <button onClick={reset}>Try Again</button>
        <Link to="/">Go Home</Link>
      </div>

      {process.env.NODE_ENV === 'development' && (
        <details className="error-details">
          <summary>Error Details</summary>
          <pre>{error.stack}</pre>
        </details>
      )}
    </div>
  )
}

export const Route = createFileRoute('/dashboard')({
  loader: async () => {
    const data = await fetchDashboardData()
    return { data }
  },
  errorComponent: CustomErrorComponent,
  component: Dashboard,
})
```

#### Pending Component (Loading States)

```typescript
// Example 3: Loading states
export const Route = createFileRoute('/posts')({
  loader: async () => {
    // Simulate slow loading
    await new Promise(resolve => setTimeout(resolve, 2000))
    const posts = await fetchPosts()
    return { posts }
  },
  pendingComponent: () => (
    <div className="loading-container">
      <div className="spinner" />
      <p>Loading posts...</p>
    </div>
  ),
  component: PostsList,
})
```

### 7. Code Splitting

#### Lazy Loading Routes

```typescript
// Example 1: Lazy load route component
import { createFileRoute, lazyRouteComponent } from '@tanstack/react-router'

export const Route = createFileRoute('/dashboard')({
  // Lazy load component
  component: lazyRouteComponent(() => import('./Dashboard')),
})

// Dashboard.tsx
export default function Dashboard() {
  return <div>Dashboard Content</div>
}
```

#### Lazy Loading với Loader

```typescript
// Example 2: Lazy load both component and loader
export const Route = createFileRoute('/products')({
  loader: async () => {
    // Loader code is also code-split
    const { fetchProducts } = await import('./api/products')
    const products = await fetchProducts()
    return { products }
  },
  component: lazyRouteComponent(() => import('./ProductsPage')),
})
```

#### Preloading Routes

```typescript
// Example 3: Preload on hover
import { Link, useRouter } from '@tanstack/react-router'

function Navigation() {
  const router = useRouter()

  const handleMouseEnter = () => {
    // Preload route on hover
    router.preloadRoute({ to: '/dashboard' })
  }

  return (
    <nav>
      <Link
        to="/dashboard"
        onMouseEnter={handleMouseEnter}
      >
        Dashboard
      </Link>

      {/* Or use built-in preload prop */}
      <Link
        to="/products"
        preload="intent" // Preload on hover/focus
      >
        Products
      </Link>
    </nav>
  )
}
```

**Preload Strategies:**
- `false`: No preloading (default)
- `"intent"`: Preload on hover/focus
- `"render"`: Preload when link renders
- `"viewport"`: Preload when link enters viewport

---

## Best Practices - Junior & Middle Level

### 1. Type Safety

```typescript
// ✅ GOOD: Use type-safe navigation
<Link to="/products/$id" params={{ id: '123' }}>Product</Link>

// ❌ BAD: String-based navigation
<a href="/products/123">Product</a>
```

### 2. Search Params Validation

```typescript
// ✅ GOOD: Validate search params
const searchSchema = z.object({
  page: z.number().default(1),
})

// ❌ BAD: No validation
const page = Number(searchParams.get('page')) || 1
```

### 3. Error Handling

```typescript
// ✅ GOOD: Provide error boundaries
export const Route = createFileRoute('/posts')({
  errorComponent: ErrorComponent,
  loader: async () => { /* ... */ },
})

// ❌ BAD: No error handling
export const Route = createFileRoute('/posts')({
  loader: async () => { /* ... */ },
})
```

### 4. Loading States

```typescript
// ✅ GOOD: Show loading UI
export const Route = createFileRoute('/posts')({
  pendingComponent: LoadingSpinner,
  loader: async () => { /* ... */ },
})

// ❌ BAD: No loading feedback
```

### 5. Code Splitting

```typescript
// ✅ GOOD: Lazy load heavy routes
component: lazyRouteComponent(() => import('./Dashboard'))

// ❌ BAD: Import everything upfront
import Dashboard from './Dashboard'
```

---

## Common Pitfalls

### ❌ Pitfall 1: Không validate search params

```typescript
// BAD: Unvalidated search params
const page = searchParams.get('page') // string | null
```

### ❌ Pitfall 2: Quên handle loading states

```typescript
// BAD: No loading indicator
// User sees blank screen while loading
```

### ❌ Pitfall 3: Không sử dụng beforeLoad cho guards

```typescript
// BAD: Check auth in component
function Dashboard() {
  if (!auth.isAuthenticated) {
    navigate('/login') // Too late!
  }
}

// GOOD: Use beforeLoad
beforeLoad: ({ context }) => {
  if (!context.auth.isAuthenticated) {
    throw redirect({ to: '/login' })
  }
}
```

---

## Tài Liệu Tham Khảo

### Official Documentation
- [TanStack Router Docs](https://tanstack.com/router/latest)
- [TanStack Router GitHub](https://github.com/tanstack/router)
- [API Reference](https://tanstack.com/router/latest/docs/framework/react/api)

### Advanced Topics
- Xem `Advanced-Router-Patterns.md` cho Senior level patterns
- Xem `Principal-Router-Patterns.md` cho Enterprise patterns

---

**Tài liệu này bao gồm Junior và Middle levels. Để học các patterns nâng cao hơn, hãy tiếp tục với các files khác trong thư mục này.**

