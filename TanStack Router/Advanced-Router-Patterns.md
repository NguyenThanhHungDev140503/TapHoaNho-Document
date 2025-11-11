# TanStack Router - Advanced Patterns (Senior Level)

## Mục Lục

- [1. Advanced Route Patterns](#1-advanced-route-patterns)
  - [Parallel Routes](#parallel-routes)
  - [Catch-All Routes](#catch-all-routes)
  - [Route Masking](#route-masking)
  
- [2. Route Preloading Strategies](#2-route-preloading-strategies)
  - [Intelligent Preloading](#intelligent-preloading)
  - [Prefetch on Viewport](#prefetch-on-viewport)
  - [Cache Management](#cache-management)

- [3. Custom Route Matching](#3-custom-route-matching)
  - [Custom Path Matching](#custom-path-matching)
  - [Route Scoring](#route-scoring)

- [4. Route Transitions](#4-route-transitions)
  - [Page Transitions](#page-transitions)
  - [View Transitions API](#view-transitions-api)

- [5. State Management Integration](#5-state-management-integration)
  - [Zustand Integration](#zustand-integration)
  - [Redux Integration](#redux-integration)

- [6. Performance Optimization](#6-performance-optimization)
  - [Route Caching](#route-caching)
  - [Memoization Strategies](#memoization-strategies)
  - [Bundle Optimization](#bundle-optimization)

- [7. Testing Routing Logic](#7-testing-routing-logic)
  - [Unit Testing Routes](#unit-testing-routes)
  - [Integration Testing](#integration-testing)
  - [E2E Testing](#e2e-testing)

---

## 1. Advanced Route Patterns

### Parallel Routes

Parallel routes cho phép render nhiều routes cùng lúc trong cùng một layout.

```typescript
// Example 1: Dashboard với parallel sections
// File structure:
// routes/
// ├── dashboard.tsx
// ├── dashboard.analytics.tsx
// ├── dashboard.users.tsx
// └── dashboard.settings.tsx

// dashboard.tsx - Parent route
import { createFileRoute, Outlet } from '@tanstack/react-router'

export const Route = createFileRoute('/dashboard')({
  component: DashboardLayout,
})

function DashboardLayout() {
  return (
    <div className="dashboard-grid">
      {/* Main content */}
      <main className="col-span-2">
        <Outlet />
      </main>
      
      {/* Sidebar - always visible */}
      <aside className="sidebar">
        <DashboardSidebar />
      </aside>
    </div>
  )
}
```

```typescript
// Example 2: Modal routes (parallel to main content)
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/products')({
  component: ProductsPage,
})

function ProductsPage() {
  const navigate = Route.useNavigate()
  const search = Route.useSearch()
  
  return (
    <div>
      <h1>Products</h1>
      
      {/* Product list */}
      <div className="products-grid">
        {products.map(product => (
          <Link
            key={product.id}
            to="/products"
            search={{ modal: 'details', productId: product.id }}
          >
            {product.name}
          </Link>
        ))}
      </div>
      
      {/* Modal rendered in parallel */}
      {search.modal === 'details' && search.productId && (
        <ProductModal 
          productId={search.productId}
          onClose={() => navigate({ 
            to: '/products',
            search: { modal: undefined, productId: undefined }
          })}
        />
      )}
    </div>
  )
}
```

**Use Cases:**
- Dashboard với multiple panels
- Modal overlays
- Split views
- Multi-step forms

### Catch-All Routes

```typescript
// Example 1: Wildcard route
// routes/docs.$.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/docs/$')({
  component: DocsPage,
})

function DocsPage() {
  // Get the wildcard path
  const params = Route.useParams()
  const wildcardPath = params['$'] // e.g., "guides/getting-started"
  
  return (
    <div>
      <h1>Documentation</h1>
      <p>Path: {wildcardPath}</p>
    </div>
  )
}

// Matches:
// /docs/guides
// /docs/guides/getting-started
// /docs/api/reference/hooks
```

```typescript
// Example 2: 404 Not Found route
// routes/$.tsx (root level catch-all)
export const Route = createFileRoute('/$')({
  component: NotFoundPage,
})

function NotFoundPage() {
  const params = Route.useParams()
  const attemptedPath = params['$']

  return (
    <div className="not-found">
      <h1>404 - Page Not Found</h1>
      <p>The page "{attemptedPath}" does not exist.</p>
      <Link to="/">Go Home</Link>
    </div>
  )
}
```

### Route Masking

Route masking cho phép hiển thị URL khác với actual route.

```typescript
// Example: Modal với masked URL
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/products/$id')({
  component: ProductDetail,
})

function ProductDetail() {
  const { id } = Route.useParams()
  const navigate = Route.useNavigate()

  const openQuickView = () => {
    // Navigate to detail but mask URL as /products
    navigate({
      to: '/products/$id',
      params: { id },
      mask: {
        to: '/products',
        unmaskOnReload: true, // Show real URL on page reload
      },
    })
  }

  return (
    <div>
      <h1>Product {id}</h1>
      <button onClick={openQuickView}>Quick View</button>
    </div>
  )
}
```

**Use Cases:**
- Modal overlays (show parent URL)
- Preview modes
- Temporary views
- A/B testing different URLs

---

## 2. Route Preloading Strategies

### Intelligent Preloading

```typescript
// Example 1: Preload based on user behavior
import { useRouter } from '@tanstack/react-router'
import { useEffect } from 'react'

function useIntelligentPreload() {
  const router = useRouter()

  useEffect(() => {
    // Preload likely next routes based on current route
    const currentPath = router.state.location.pathname

    if (currentPath === '/') {
      // User on homepage, likely to visit products
      router.preloadRoute({ to: '/products' })
    } else if (currentPath.startsWith('/products')) {
      // User browsing products, preload cart
      router.preloadRoute({ to: '/cart' })
    }
  }, [router.state.location.pathname])
}

function App() {
  useIntelligentPreload()
  return <RouterProvider router={router} />
}
```

```typescript
// Example 2: Preload on idle
import { useIdleCallback } from './hooks/useIdleCallback'

function Navigation() {
  const router = useRouter()

  useIdleCallback(() => {
    // Preload routes when browser is idle
    const routesToPreload = ['/dashboard', '/products', '/settings']

    routesToPreload.forEach(route => {
      router.preloadRoute({ to: route })
    })
  }, { timeout: 2000 })

  return <nav>{/* ... */}</nav>
}
```

### Prefetch on Viewport

```typescript
// Example: Intersection Observer preloading
import { useEffect, useRef } from 'react'
import { useRouter } from '@tanstack/react-router'

function PreloadLink({
  to,
  children,
  ...props
}: {
  to: string
  children: React.ReactNode
}) {
  const router = useRouter()
  const linkRef = useRef<HTMLAnchorElement>(null)

  useEffect(() => {
    const link = linkRef.current
    if (!link) return

    // Create intersection observer
    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            // Preload when link enters viewport
            router.preloadRoute({ to })
            observer.unobserve(link)
          }
        })
      },
      { rootMargin: '50px' } // Preload 50px before visible
    )

    observer.observe(link)

    return () => observer.disconnect()
  }, [to, router])

  return (
    <Link ref={linkRef} to={to} {...props}>
      {children}
    </Link>
  )
}

// Usage
function ProductList() {
  return (
    <div>
      {products.map(product => (
        <PreloadLink
          key={product.id}
          to="/products/$id"
          params={{ id: product.id }}
        >
          {product.name}
        </PreloadLink>
      ))}
    </div>
  )
}
```

### Cache Management

```typescript
// Example: Custom cache control
import { createRouter } from '@tanstack/react-router'

const router = createRouter({
  routeTree,
  defaultPreloadStaleTime: 5000, // Cache for 5 seconds
  defaultPreload: 'intent', // Preload on hover

  // Custom cache invalidation
  onRouteChange: ({ fromLocation, toLocation }) => {
    // Clear cache when navigating away from certain routes
    if (fromLocation.pathname.startsWith('/admin')) {
      router.invalidate()
    }
  },
})

// Manual cache control
function AdminPanel() {
  const router = useRouter()

  const handleDataUpdate = async () => {
    await updateData()

    // Invalidate specific route
    router.invalidate({
      to: '/admin/users',
    })

    // Or invalidate all routes
    router.invalidate()
  }

  return <button onClick={handleDataUpdate}>Update</button>
}
```

---

## 3. Custom Route Matching

### Custom Path Matching

```typescript
// Example: Custom route matcher
import { createRouter, AnyRoute } from '@tanstack/react-router'

// Custom matcher for locale-prefixed routes
function createLocaleMatcher(route: AnyRoute) {
  return (pathname: string) => {
    const localeRegex = /^\/(en|vi|ja)\//
    const match = pathname.match(localeRegex)

    if (match) {
      const locale = match[1]
      const pathWithoutLocale = pathname.replace(localeRegex, '/')

      // Check if path matches route
      if (route.path === pathWithoutLocale) {
        return {
          params: { locale },
          pathname: pathWithoutLocale,
        }
      }
    }

    return null
  }
}

// Usage
export const Route = createFileRoute('/products')({
  // Custom matcher
  matcher: createLocaleMatcher,
  component: ProductsPage,
})

// Matches:
// /en/products
// /vi/products
// /ja/products
```

---

## 4. Route Transitions

### Page Transitions

```typescript
// Example 1: Framer Motion transitions
import { motion, AnimatePresence } from 'framer-motion'
import { useRouter } from '@tanstack/react-router'

function AnimatedOutlet() {
  const router = useRouter()
  const location = router.state.location

  return (
    <AnimatePresence mode="wait">
      <motion.div
        key={location.pathname}
        initial={{ opacity: 0, x: -20 }}
        animate={{ opacity: 1, x: 0 }}
        exit={{ opacity: 0, x: 20 }}
        transition={{ duration: 0.3 }}
      >
        <Outlet />
      </motion.div>
    </AnimatePresence>
  )
}

// Root route
export const Route = createRootRoute({
  component: () => (
    <div>
      <Navigation />
      <AnimatedOutlet />
    </div>
  ),
})
```

### View Transitions API

```typescript
// Example 2: Native View Transitions API
import { useRouter } from '@tanstack/react-router'
import { useEffect } from 'react'

function useViewTransitions() {
  const router = useRouter()

  useEffect(() => {
    // Check if View Transitions API is supported
    if (!document.startViewTransition) return

    const handleNavigation = () => {
      document.startViewTransition(() => {
        // Navigation happens here
      })
    }

    // Listen to navigation events
    return router.subscribe('onBeforeNavigate', handleNavigation)
  }, [router])
}

// CSS for view transitions
/*
::view-transition-old(root),
::view-transition-new(root) {
  animation-duration: 0.3s;
}

::view-transition-old(root) {
  animation-name: fade-out;
}

::view-transition-new(root) {
  animation-name: fade-in;
}
*/
```

---

## 5. State Management Integration

### Zustand Integration

```typescript
// Example 1: Sync router state với Zustand
import { create } from 'zustand'
import { useRouter } from '@tanstack/react-router'
import { useEffect } from 'react'

interface RouterStore {
  currentPath: string
  searchParams: Record<string, any>
  setCurrentPath: (path: string) => void
  setSearchParams: (params: Record<string, any>) => void
}

const useRouterStore = create<RouterStore>((set) => ({
  currentPath: '/',
  searchParams: {},
  setCurrentPath: (path) => set({ currentPath: path }),
  setSearchParams: (params) => set({ searchParams: params }),
}))

// Sync hook
function useSyncRouterWithZustand() {
  const router = useRouter()
  const { setCurrentPath, setSearchParams } = useRouterStore()

  useEffect(() => {
    const unsubscribe = router.subscribe('onResolved', () => {
      setCurrentPath(router.state.location.pathname)
      setSearchParams(router.state.location.search)
    })

    return unsubscribe
  }, [router, setCurrentPath, setSearchParams])
}

// Usage in App
function App() {
  useSyncRouterWithZustand()
  return <RouterProvider router={router} />
}
```

```typescript
// Example 2: Route-based state management
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface ProductFilters {
  category: string | null
  priceRange: [number, number]
  sortBy: 'price' | 'name' | 'date'
}

interface ProductStore {
  filters: ProductFilters
  setFilters: (filters: Partial<ProductFilters>) => void
  resetFilters: () => void
}

const useProductStore = create<ProductStore>()(
  persist(
    (set) => ({
      filters: {
        category: null,
        priceRange: [0, 1000],
        sortBy: 'name',
      },
      setFilters: (newFilters) =>
        set((state) => ({
          filters: { ...state.filters, ...newFilters },
        })),
      resetFilters: () =>
        set({
          filters: {
            category: null,
            priceRange: [0, 1000],
            sortBy: 'name',
          },
        }),
    }),
    { name: 'product-filters' }
  )
)

// Sync với route search params
export const Route = createFileRoute('/products')({
  validateSearch: zodValidator(productSearchSchema),
  beforeLoad: ({ search }) => {
    // Sync search params to Zustand
    const { setFilters } = useProductStore.getState()
    setFilters({
      category: search.category,
      sortBy: search.sortBy,
    })
  },
  component: ProductsPage,
})
```

### Redux Integration

```typescript
// Example: Redux với router
import { configureStore, createSlice } from '@reduxjs/toolkit'
import { useRouter } from '@tanstack/react-router'
import { useEffect } from 'react'

// Router slice
const routerSlice = createSlice({
  name: 'router',
  initialState: {
    pathname: '/',
    search: {},
    params: {},
  },
  reducers: {
    updateLocation: (state, action) => {
      state.pathname = action.payload.pathname
      state.search = action.payload.search
      state.params = action.payload.params
    },
  },
})

const store = configureStore({
  reducer: {
    router: routerSlice.reducer,
  },
})

// Sync hook
function useSyncRouterWithRedux() {
  const router = useRouter()
  const dispatch = useDispatch()

  useEffect(() => {
    return router.subscribe('onResolved', () => {
      dispatch(routerSlice.actions.updateLocation({
        pathname: router.state.location.pathname,
        search: router.state.location.search,
        params: router.state.matches[0]?.params || {},
      }))
    })
  }, [router, dispatch])
}
```

---

## 6. Performance Optimization

### Route Caching

```typescript
// Example 1: Aggressive caching strategy
import { createRouter } from '@tanstack/react-router'

const router = createRouter({
  routeTree,

  // Cache settings
  defaultPreloadStaleTime: 30000, // 30 seconds
  defaultPreload: 'intent',

  // Custom cache key
  defaultLoaderMaxAge: 60000, // 1 minute

  // Prevent unnecessary reloads
  defaultGcMaxAge: 300000, // 5 minutes
})
```

```typescript
// Example 2: Per-route caching
export const Route = createFileRoute('/products/$id')({
  loader: async ({ params }) => {
    const product = await fetchProduct(params.id)
    return { product }
  },

  // Cache for 5 minutes
  staleTime: 300000,

  // Keep in cache for 10 minutes
  gcMaxAge: 600000,

  component: ProductDetail,
})
```

### Memoization Strategies

```typescript
// Example 1: Memoize expensive computations
import { useMemo } from 'react'
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/analytics')({
  loader: async () => {
    const data = await fetchAnalyticsData()
    return { data }
  },
  component: AnalyticsPage,
})

function AnalyticsPage() {
  const { data } = Route.useLoaderData()

  // Memoize expensive calculations
  const processedData = useMemo(() => {
    return data.map(item => ({
      ...item,
      trend: calculateTrend(item.values),
      average: calculateAverage(item.values),
      prediction: predictNextValue(item.values),
    }))
  }, [data])

  return (
    <div>
      {processedData.map(item => (
        <AnalyticsCard key={item.id} data={item} />
      ))}
    </div>
  )
}
```

```typescript
// Example 2: Memoize route components
import { memo } from 'react'

const ProductCard = memo(({ product }: { product: Product }) => {
  return (
    <div className="product-card">
      <img src={product.image} alt={product.name} />
      <h3>{product.name}</h3>
      <p>${product.price}</p>
    </div>
  )
})

function ProductsPage() {
  const { products } = Route.useLoaderData()

  return (
    <div className="products-grid">
      {products.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  )
}
```

### Bundle Optimization

```typescript
// Example: Route-based code splitting
import { createFileRoute, lazyRouteComponent } from '@tanstack/react-router'

// Heavy dashboard with charts
export const Route = createFileRoute('/dashboard')({
  component: lazyRouteComponent(() =>
    import('./Dashboard').then(m => ({
      default: m.Dashboard,
    }))
  ),
})

// Lazy load heavy dependencies
export const Route = createFileRoute('/editor')({
  loader: async () => {
    // Lazy load Monaco Editor
    const monaco = await import('monaco-editor')
    return { monaco }
  },
  component: lazyRouteComponent(() => import('./Editor')),
})
```

---

## 7. Testing Routing Logic

### Unit Testing Routes

```typescript
// Example 1: Test route configuration
import { describe, it, expect } from 'vitest'
import { createMemoryHistory } from '@tanstack/react-router'
import { router } from './router'

describe('Route Configuration', () => {
  it('should match product detail route', () => {
    const history = createMemoryHistory({
      initialEntries: ['/products/123'],
    })

    const testRouter = router.update({
      history,
    })

    const match = testRouter.state.matches[0]
    expect(match?.routeId).toBe('/products/$id')
    expect(match?.params).toEqual({ id: '123' })
  })

  it('should validate search params', () => {
    const history = createMemoryHistory({
      initialEntries: ['/products?page=2&category=electronics'],
    })

    const testRouter = router.update({ history })
    const search = testRouter.state.location.search

    expect(search.page).toBe(2)
    expect(search.category).toBe('electronics')
  })
})
```

### Integration Testing

```typescript
// Example 2: Test navigation flow
import { render, screen, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { RouterProvider } from '@tanstack/react-router'
import { router } from './router'

describe('Navigation Flow', () => {
  it('should navigate from home to product detail', async () => {
    const user = userEvent.setup()

    render(<RouterProvider router={router} />)

    // Click on product link
    const productLink = screen.getByText('Product 1')
    await user.click(productLink)

    // Wait for navigation
    await waitFor(() => {
      expect(screen.getByText('Product Details')).toBeInTheDocument()
    })

    // Check URL
    expect(router.state.location.pathname).toBe('/products/1')
  })

  it('should redirect unauthenticated users', async () => {
    // Mock unauthenticated state
    const mockAuth = { isAuthenticated: false }

    render(
      <RouterProvider
        router={router}
        context={{ auth: mockAuth }}
      />
    )

    // Try to access protected route
    router.navigate({ to: '/dashboard' })

    await waitFor(() => {
      expect(router.state.location.pathname).toBe('/login')
    })
  })
})
```

### E2E Testing

```typescript
// Example 3: Playwright E2E test
import { test, expect } from '@playwright/test'

test.describe('Product Navigation', () => {
  test('should navigate through product pages', async ({ page }) => {
    await page.goto('http://localhost:3000')

    // Click products link
    await page.click('text=Products')
    await expect(page).toHaveURL(/\/products/)

    // Click first product
    await page.click('.product-card:first-child')
    await expect(page).toHaveURL(/\/products\/\d+/)

    // Check product details loaded
    await expect(page.locator('h1')).toContainText('Product')

    // Go back
    await page.goBack()
    await expect(page).toHaveURL(/\/products/)
  })

  test('should handle authentication flow', async ({ page }) => {
    await page.goto('http://localhost:3000/dashboard')

    // Should redirect to login
    await expect(page).toHaveURL(/\/login/)

    // Fill login form
    await page.fill('input[name="username"]', 'testuser')
    await page.fill('input[name="password"]', 'password')
    await page.click('button[type="submit"]')

    // Should redirect back to dashboard
    await expect(page).toHaveURL(/\/dashboard/)
  })
})
```

---

## Best Practices - Senior Level

### 1. Preloading Strategy

```typescript
// ✅ GOOD: Intelligent preloading
<Link to="/dashboard" preload="intent">Dashboard</Link>

// ❌ BAD: Preload everything
<Link to="/dashboard" preload="render">Dashboard</Link>
```

### 2. Error Boundaries

```typescript
// ✅ GOOD: Granular error boundaries
export const Route = createFileRoute('/products')({
  errorComponent: ProductsErrorBoundary,
})

// ❌ BAD: Single global error boundary
```

### 3. Code Splitting

```typescript
// ✅ GOOD: Split heavy routes
component: lazyRouteComponent(() => import('./Dashboard'))

// ❌ BAD: Import everything upfront
```

### 4. Cache Management

```typescript
// ✅ GOOD: Configure cache per route
staleTime: 300000, // 5 minutes

// ❌ BAD: No cache configuration
```

### 5. Testing

```typescript
// ✅ GOOD: Test navigation logic
it('should redirect unauthenticated users', async () => {
  // Test implementation
})

// ❌ BAD: No routing tests
```

---

## Common Pitfalls - Senior Level

### ❌ Pitfall 1: Over-preloading

```typescript
// BAD: Preload too many routes
useEffect(() => {
  allRoutes.forEach(route => router.preloadRoute({ to: route }))
}, [])
```

### ❌ Pitfall 2: Không handle race conditions

```typescript
// BAD: No cancellation
loader: async ({ params }) => {
  const data = await fetchData(params.id)
  return { data }
}

// GOOD: Handle cancellation
loader: async ({ params, abortController }) => {
  const data = await fetchData(params.id, {
    signal: abortController.signal,
  })
  return { data }
}
```

### ❌ Pitfall 3: Memory leaks trong subscriptions

```typescript
// BAD: No cleanup
useEffect(() => {
  router.subscribe('onResolved', handleNavigation)
}, [])

// GOOD: Cleanup subscription
useEffect(() => {
  const unsubscribe = router.subscribe('onResolved', handleNavigation)
  return unsubscribe
}, [])
```

---

**Tiếp tục với Principal Level patterns trong file `Principal-Router-Patterns.md`**

