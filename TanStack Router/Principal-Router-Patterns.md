# TanStack Router - Principal Level Patterns

## Mục Lục

- [1. Large-Scale Routing Architecture](#1-large-scale-routing-architecture)
  - [Modular Route Organization](#modular-route-organization)
  - [Route Registry Pattern](#route-registry-pattern)
  - [Feature-Based Routing](#feature-based-routing)

- [2. File-Based Routing Systems](#2-file-based-routing-systems)
  - [Auto-Generated Routes](#auto-generated-routes)
  - [Route Tree Generation](#route-tree-generation)
  - [Convention-Based Routing](#convention-based-routing)

- [3. SSR/SSG với TanStack Router](#3-ssrssg-với-tanstack-router)
  - [Server-Side Rendering](#server-side-rendering)
  - [Static Site Generation](#static-site-generation)
  - [Streaming SSR](#streaming-ssr)

- [4. Route-Based Code Splitting](#4-route-based-code-splitting)
  - [Dynamic Imports Strategy](#dynamic-imports-strategy)
  - [Vendor Splitting](#vendor-splitting)
  - [Critical Path Optimization](#critical-path-optimization)

- [5. Micro-Frontends Routing](#5-micro-frontends-routing)
  - [Module Federation](#module-federation)
  - [Cross-App Navigation](#cross-app-navigation)
  - [Shared State Management](#shared-state-management)

- [6. Route Analytics](#6-route-analytics)
  - [Performance Monitoring](#performance-monitoring)
  - [User Journey Tracking](#user-journey-tracking)
  - [Error Tracking](#error-tracking)

- [7. Migration Strategies](#7-migration-strategies)
  - [React Router Migration](#react-router-migration)
  - [Incremental Adoption](#incremental-adoption)
  - [Backward Compatibility](#backward-compatibility)

- [8. Performance Tuning](#8-performance-tuning)
  - [Bundle Size Optimization](#bundle-size-optimization)
  - [Runtime Performance](#runtime-performance)
  - [Memory Management](#memory-management)

---

## 1. Large-Scale Routing Architecture

### Modular Route Organization

```typescript
// Example 1: Feature-based route modules
// src/features/products/routes.ts
import { createRoute } from '@tanstack/react-router'
import { rootRoute } from '@/router'

// Products feature routes
const productsRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: 'products',
  component: () => import('./ProductsLayout'),
})

const productsIndexRoute = createRoute({
  getParentRoute: () => productsRoute,
  path: '/',
  component: () => import('./ProductsList'),
})

const productDetailRoute = createRoute({
  getParentRoute: () => productsRoute,
  path: '$id',
  component: () => import('./ProductDetail'),
})

export const productRoutes = productsRoute.addChildren([
  productsIndexRoute,
  productDetailRoute,
])
```

```typescript
// src/features/auth/routes.ts
export const authRoutes = createRoute({
  getParentRoute: () => rootRoute,
  path: 'auth',
  component: () => import('./AuthLayout'),
}).addChildren([
  createRoute({
    getParentRoute: () => authRoutes,
    path: 'login',
    component: () => import('./Login'),
  }),
  createRoute({
    getParentRoute: () => authRoutes,
    path: 'register',
    component: () => import('./Register'),
  }),
])
```

```typescript
// src/router/index.ts - Combine all feature routes
import { createRouter } from '@tanstack/react-router'
import { productRoutes } from '@/features/products/routes'
import { authRoutes } from '@/features/auth/routes'
import { dashboardRoutes } from '@/features/dashboard/routes'

const routeTree = rootRoute.addChildren([
  indexRoute,
  productRoutes,
  authRoutes,
  dashboardRoutes,
])

export const router = createRouter({ routeTree })
```

**Benefits:**
- Clear separation of concerns
- Easy to add/remove features
- Better code organization
- Team can work independently on features

### Route Registry Pattern

```typescript
// Example 2: Centralized route registry
// src/router/registry.ts
interface RouteConfig {
  path: string
  loader: () => Promise<any>
  component: () => Promise<any>
  guards?: Array<(context: any) => boolean>
  meta?: {
    title: string
    description?: string
    requiresAuth?: boolean
    roles?: string[]
  }
}

class RouteRegistry {
  private routes = new Map<string, RouteConfig>()
  
  register(id: string, config: RouteConfig) {
    this.routes.set(id, config)
  }
  
  get(id: string): RouteConfig | undefined {
    return this.routes.get(id)
  }
  
  getAll(): RouteConfig[] {
    return Array.from(this.routes.values())
  }
  
  // Generate sitemap
  generateSitemap(): string[] {
    return Array.from(this.routes.keys())
  }
  
  // Find routes by meta
  findByMeta(predicate: (meta: RouteConfig['meta']) => boolean): RouteConfig[] {
    return this.getAll().filter(route => 
      route.meta && predicate(route.meta)
    )
  }
}

export const routeRegistry = new RouteRegistry()

// Register routes
routeRegistry.register('products.list', {
  path: '/products',
  loader: () => import('./features/products/loader'),
  component: () => import('./features/products/ProductsList'),
  meta: {
    title: 'Products',
    requiresAuth: false,
  },
})

routeRegistry.register('products.detail', {
  path: '/products/:id',
  loader: () => import('./features/products/detail-loader'),
  component: () => import('./features/products/ProductDetail'),
  meta: {
    title: 'Product Detail',
    requiresAuth: false,
  },
})
```

### Feature-Based Routing

```typescript
// Example 3: Complete feature module
// src/features/products/index.ts
export interface ProductsFeature {
  routes: RouteConfig[]
  services: ProductService
  store: ProductStore
}

export function createProductsFeature(): ProductsFeature {
  const store = createProductStore()
  const services = createProductService(store)

  const routes = [
    {
      path: '/products',
      loader: () => services.loadProducts(),
      component: () => import('./components/ProductsList'),
    },
    {
      path: '/products/:id',
      loader: ({ params }) => services.loadProduct(params.id),
      component: () => import('./components/ProductDetail'),
    },
  ]

  return { routes, services, store }
}

// src/app.ts - Bootstrap features
const features = [
  createProductsFeature(),
  createAuthFeature(),
  createDashboardFeature(),
]

const allRoutes = features.flatMap(f => f.routes)
const router = createRouter({ routes: allRoutes })
```

---

## 2. File-Based Routing Systems

### Auto-Generated Routes

```typescript
// Example 1: Vite plugin for route generation
// vite.config.ts
import { defineConfig } from 'vite'
import { TanStackRouterVite } from '@tanstack/router-vite-plugin'

export default defineConfig({
  plugins: [
    TanStackRouterVite({
      // Auto-generate routes from files
      routesDirectory: './src/routes',
      generatedRouteTree: './src/routeTree.gen.ts',

      // Route naming conventions
      routeFilePrefix: '',
      routeFileIgnorePrefix: '_',

      // Enable experimental features
      experimental: {
        enableCodeSplitting: true,
      },
    }),
  ],
})
```

**Generated Route Tree:**
```typescript
// src/routeTree.gen.ts (auto-generated)
import { Route as rootRoute } from './routes/__root'
import { Route as IndexRoute } from './routes/index'
import { Route as ProductsRoute } from './routes/products'
import { Route as ProductsIdRoute } from './routes/products.$id'

const routeTree = rootRoute.addChildren([
  IndexRoute,
  ProductsRoute.addChildren([ProductsIdRoute]),
])

export { routeTree }
```

### Route Tree Generation

```typescript
// Example 2: Custom route generator
// scripts/generate-routes.ts
import fs from 'fs'
import path from 'path'
import { glob } from 'glob'

interface RouteNode {
  path: string
  filePath: string
  children: RouteNode[]
}

async function generateRouteTree() {
  const routesDir = path.join(process.cwd(), 'src/routes')
  const files = await glob('**/*.tsx', { cwd: routesDir })

  const tree: RouteNode[] = []

  for (const file of files) {
    const segments = file.replace('.tsx', '').split('/')
    let currentLevel = tree

    for (let i = 0; i < segments.length; i++) {
      const segment = segments[i]
      const isLast = i === segments.length - 1

      let node = currentLevel.find(n => n.path === segment)

      if (!node) {
        node = {
          path: segment,
          filePath: isLast ? file : '',
          children: [],
        }
        currentLevel.push(node)
      }

      currentLevel = node.children
    }
  }

  // Generate TypeScript code
  const code = generateRouteCode(tree)

  fs.writeFileSync(
    path.join(process.cwd(), 'src/routeTree.gen.ts'),
    code
  )
}

function generateRouteCode(nodes: RouteNode[]): string {
  let imports = ''
  let routes = ''

  // Generate imports and route definitions
  nodes.forEach(node => {
    if (node.filePath) {
      const routeName = node.path.replace(/[.$]/g, '_')
      imports += `import { Route as ${routeName}Route } from './routes/${node.filePath}'\n`
      routes += `  ${routeName}Route,\n`
    }
  })

  return `
${imports}

export const routeTree = rootRoute.addChildren([
${routes}
])
  `.trim()
}

generateRouteTree()
```

### Convention-Based Routing

```typescript
// Example 3: Route conventions
/*
File Structure Conventions:

routes/
├── __root.tsx              → Root layout
├── index.tsx               → / (home)
├── about.tsx               → /about
├── _auth.tsx               → Layout route (no path)
├── _auth.login.tsx         → /login (with _auth layout)
├── _auth.register.tsx      → /register (with _auth layout)
├── products.tsx            → /products (layout)
├── products.index.tsx      → /products/ (index)
├── products.$id.tsx        → /products/:id
├── products.$id.edit.tsx   → /products/:id/edit
├── blog.$.tsx              → /blog/* (catch-all)
└── $.tsx                   → /* (404)

Naming Conventions:
- __root.tsx: Root route
- index.tsx: Index route
- _prefix: Layout route (no path segment)
- $param: Dynamic parameter
- $.tsx: Catch-all/wildcard
*/
```

---

## 3. SSR/SSG với TanStack Router

### Server-Side Rendering

```typescript
// Example 1: SSR với Express
// server.ts
import express from 'express'
import { renderToString } from 'react-dom/server'
import { createMemoryHistory, RouterProvider } from '@tanstack/react-router'
import { router } from './router'

const app = express()

app.get('*', async (req, res) => {
  // Create memory history for SSR
  const history = createMemoryHistory({
    initialEntries: [req.url],
  })

  // Update router with SSR history
  const ssrRouter = router.update({
    history,
    context: {
      // Pass server-side context
      req,
      res,
    },
  })

  // Wait for all loaders to complete
  await ssrRouter.load()

  // Render to string
  const html = renderToString(
    <RouterProvider router={ssrRouter} />
  )

  // Get dehydrated state for client hydration
  const dehydratedState = ssrRouter.dehydrate()

  res.send(`
    <!DOCTYPE html>
    <html>
      <head>
        <title>My App</title>
      </head>
      <body>
        <div id="root">${html}</div>
        <script>
          window.__ROUTER_STATE__ = ${JSON.stringify(dehydratedState)}
        </script>
        <script src="/client.js"></script>
      </body>
    </html>
  `)
})

app.listen(3000)
```

```typescript
// client.ts - Hydration
import { hydrateRoot } from 'react-dom/client'
import { RouterProvider } from '@tanstack/react-router'
import { router } from './router'

// Hydrate with server state
const dehydratedState = window.__ROUTER_STATE__

router.hydrate(dehydratedState)

hydrateRoot(
  document.getElementById('root')!,
  <RouterProvider router={router} />
)
```

### Static Site Generation

```typescript
// Example 2: SSG build script
// scripts/build-ssg.ts
import fs from 'fs'
import path from 'path'
import { renderToString } from 'react-dom/server'
import { createMemoryHistory, RouterProvider } from '@tanstack/react-router'
import { router } from '../src/router'

interface StaticRoute {
  path: string
  params?: Record<string, string>
}

async function generateStaticSite() {
  // Define routes to pre-render
  const staticRoutes: StaticRoute[] = [
    { path: '/' },
    { path: '/about' },
    { path: '/products' },
    // Dynamic routes
    { path: '/products/$id', params: { id: '1' } },
    { path: '/products/$id', params: { id: '2' } },
  ]

  const outputDir = path.join(process.cwd(), 'dist')

  for (const route of staticRoutes) {
    const url = route.params
      ? route.path.replace(/\$(\w+)/g, (_, key) => route.params![key])
      : route.path

    // Create memory history
    const history = createMemoryHistory({
      initialEntries: [url],
    })

    const ssrRouter = router.update({ history })

    // Load data
    await ssrRouter.load()

    // Render
    const html = renderToString(
      <RouterProvider router={ssrRouter} />
    )

    // Write to file
    const filePath = path.join(
      outputDir,
      url === '/' ? 'index.html' : `${url}/index.html`
    )

    fs.mkdirSync(path.dirname(filePath), { recursive: true })
    fs.writeFileSync(filePath, generateHTML(html, ssrRouter.dehydrate()))
  }
}

function generateHTML(content: string, state: any): string {
  return `
    <!DOCTYPE html>
    <html>
      <head>
        <title>My App</title>
        <link rel="stylesheet" href="/styles.css">
      </head>
      <body>
        <div id="root">${content}</div>
        <script>
          window.__ROUTER_STATE__ = ${JSON.stringify(state)}
        </script>
        <script src="/client.js"></script>
      </body>
    </html>
  `
}

generateStaticSite()
```

### Streaming SSR

```typescript
// Example 3: Streaming SSR với React 18
import { renderToPipeableStream } from 'react-dom/server'
import { PassThrough } from 'stream'

app.get('*', async (req, res) => {
  const history = createMemoryHistory({
    initialEntries: [req.url],
  })

  const ssrRouter = router.update({ history })

  // Start loading (don't await)
  ssrRouter.load()

  let didError = false

  const { pipe } = renderToPipeableStream(
    <RouterProvider router={ssrRouter} />,
    {
      onShellReady() {
        res.statusCode = didError ? 500 : 200
        res.setHeader('Content-Type', 'text/html')

        // Send shell immediately
        res.write('<!DOCTYPE html><html><head><title>My App</title></head><body><div id="root">')

        // Stream content
        pipe(res)
      },
      onShellError(error) {
        didError = true
        res.statusCode = 500
        res.send('Internal Server Error')
      },
      onAllReady() {
        // Close HTML
        res.write('</div></body></html>')
        res.end()
      },
    }
  )
})
```

---

## 4. Route-Based Code Splitting

### Dynamic Imports Strategy

```typescript
// Example 1: Granular code splitting
// webpack.config.js
export default {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        // Vendor splitting
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: 10,
        },
        // Route-specific chunks
        routes: {
          test: /[\\/]routes[\\/]/,
          name(module) {
            // Generate chunk name from route path
            const match = module.context.match(/routes[\\/](.*)/)
            return match ? `route-${match[1].replace(/[\\/]/g, '-')}` : 'route'
          },
          priority: 5,
        },
        // Common components
        common: {
          minChunks: 2,
          priority: 1,
          reuseExistingChunk: true,
        },
      },
    },
  },
}
```

```typescript
// Example 2: Route-level splitting
export const Route = createFileRoute('/dashboard')({
  // Split component
  component: lazyRouteComponent(() => import('./Dashboard')),

  // Split loader dependencies
  loader: async ({ params }) => {
    // Lazy load heavy dependencies
    const [{ fetchDashboardData }, { processData }] = await Promise.all([
      import('./api/dashboard'),
      import('./utils/dataProcessor'),
    ])

    const data = await fetchDashboardData()
    const processed = processData(data)

    return { data: processed }
  },
})
```

### Vendor Splitting

```typescript
// Example 3: Strategic vendor splitting
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // React core
          'react-vendor': ['react', 'react-dom'],

          // Router
          'router-vendor': ['@tanstack/react-router'],

          // UI libraries
          'ui-vendor': ['@radix-ui/react-dialog', '@radix-ui/react-dropdown-menu'],

          // Charts (heavy)
          'charts-vendor': ['recharts', 'd3'],

          // Utils
          'utils-vendor': ['date-fns', 'lodash-es'],
        },
      },
    },
  },
})
```

### Critical Path Optimization

```typescript
// Example 4: Optimize critical rendering path
// src/routes/__root.tsx
import { createRootRoute } from '@tanstack/react-router'
import { Suspense, lazy } from 'react'

// Critical: Load immediately
import { Header } from '@/components/Header'
import { Footer } from '@/components/Footer'

// Non-critical: Lazy load
const DevTools = lazy(() => import('@tanstack/router-devtools'))
const Analytics = lazy(() => import('@/components/Analytics'))

export const Route = createRootRoute({
  component: () => (
    <div>
      {/* Critical content */}
      <Header />

      <main>
        <Outlet />
      </main>

      <Footer />

      {/* Non-critical content */}
      <Suspense fallback={null}>
        {process.env.NODE_ENV === 'development' && <DevTools />}
        <Analytics />
      </Suspense>
    </div>
  ),
})
```

---

## 5. Micro-Frontends Routing

### Module Federation

```typescript
// Example 1: Host app configuration
// webpack.config.js (Host)
import { ModuleFederationPlugin } from 'webpack/lib/container/ModuleFederationPlugin'

export default {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host',
      remotes: {
        products: 'products@http://localhost:3001/remoteEntry.js',
        checkout: 'checkout@http://localhost:3002/remoteEntry.js',
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true },
        '@tanstack/react-router': { singleton: true },
      },
    }),
  ],
}
```

```typescript
// Host router
import { createRouter } from '@tanstack/react-router'
import { lazy } from 'react'

// Lazy load remote modules
const ProductsApp = lazy(() => import('products/App'))
const CheckoutApp = lazy(() => import('checkout/App'))

const productsRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: 'products',
  component: () => (
    <Suspense fallback={<div>Loading Products...</div>}>
      <ProductsApp />
    </Suspense>
  ),
})

const checkoutRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: 'checkout',
  component: () => (
    <Suspense fallback={<div>Loading Checkout...</div>}>
      <CheckoutApp />
    </Suspense>
  ),
})
```

### Cross-App Navigation

```typescript
// Example 2: Shared navigation service
// packages/shared/navigation.ts
interface NavigationEvent {
  type: 'navigate'
  path: string
  params?: Record<string, any>
}

class CrossAppNavigation {
  private listeners = new Set<(event: NavigationEvent) => void>()

  subscribe(listener: (event: NavigationEvent) => void) {
    this.listeners.add(listener)
    return () => this.listeners.delete(listener)
  }

  navigate(path: string, params?: Record<string, any>) {
    const event: NavigationEvent = { type: 'navigate', path, params }
    this.listeners.forEach(listener => listener(event))
  }
}

export const crossAppNav = new CrossAppNavigation()

// Host app
function HostApp() {
  const router = useRouter()

  useEffect(() => {
    return crossAppNav.subscribe((event) => {
      if (event.type === 'navigate') {
        router.navigate({ to: event.path, params: event.params })
      }
    })
  }, [router])

  return <RouterProvider router={router} />
}

// Remote app (Products)
function ProductCard({ productId }: { productId: string }) {
  const handleClick = () => {
    // Navigate in host app
    crossAppNav.navigate('/checkout', { productId })
  }

  return <button onClick={handleClick}>Buy Now</button>
}
```

### Shared State Management

```typescript
// Example 3: Cross-app state sync
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

// Shared store (in shared package)
interface SharedStore {
  user: User | null
  cart: CartItem[]
  setUser: (user: User | null) => void
  addToCart: (item: CartItem) => void
}

export const useSharedStore = create<SharedStore>()(
  persist(
    (set) => ({
      user: null,
      cart: [],
      setUser: (user) => set({ user }),
      addToCart: (item) => set((state) => ({
        cart: [...state.cart, item],
      })),
    }),
    {
      name: 'shared-store',
      // Use localStorage for cross-app sync
      storage: {
        getItem: (name) => {
          const str = localStorage.getItem(name)
          return str ? JSON.parse(str) : null
        },
        setItem: (name, value) => {
          localStorage.setItem(name, JSON.stringify(value))
        },
        removeItem: (name) => {
          localStorage.removeItem(name)
        },
      },
    }
  )
)

// Use in any micro-frontend
function ProductsApp() {
  const { user, addToCart } = useSharedStore()

  return (
    <div>
      {user && <p>Welcome, {user.name}</p>}
      <button onClick={() => addToCart(product)}>
        Add to Cart
      </button>
    </div>
  )
}
```

---

## 6. Route Analytics

### Performance Monitoring

```typescript
// Example 1: Route performance tracking
import { useRouter } from '@tanstack/react-router'
import { useEffect } from 'react'

interface RouteMetrics {
  path: string
  loadTime: number
  renderTime: number
  timestamp: number
}

function useRoutePerformance() {
  const router = useRouter()

  useEffect(() => {
    let navigationStart = 0
    let loadComplete = 0

    const unsubscribeBeforeLoad = router.subscribe('onBeforeLoad', () => {
      navigationStart = performance.now()
    })

    const unsubscribeResolved = router.subscribe('onResolved', () => {
      loadComplete = performance.now()

      const metrics: RouteMetrics = {
        path: router.state.location.pathname,
        loadTime: loadComplete - navigationStart,
        renderTime: performance.now() - loadComplete,
        timestamp: Date.now(),
      }

      // Send to analytics
      sendMetrics(metrics)
    })

    return () => {
      unsubscribeBeforeLoad()
      unsubscribeResolved()
    }
  }, [router])
}

function sendMetrics(metrics: RouteMetrics) {
  // Send to analytics service
  if (window.gtag) {
    window.gtag('event', 'route_performance', {
      page_path: metrics.path,
      load_time: metrics.loadTime,
      render_time: metrics.renderTime,
    })
  }

  // Or custom analytics
  fetch('/api/analytics/route-performance', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(metrics),
  })
}
```

### User Journey Tracking

```typescript
// Example 2: Track user navigation flow
interface JourneyStep {
  path: string
  timestamp: number
  duration: number
  params?: Record<string, any>
}

class UserJourneyTracker {
  private journey: JourneyStep[] = []
  private currentStepStart = Date.now()

  trackNavigation(path: string, params?: Record<string, any>) {
    const now = Date.now()
    const duration = now - this.currentStepStart

    this.journey.push({
      path,
      timestamp: now,
      duration,
      params,
    })

    this.currentStepStart = now

    // Send to analytics every 5 steps
    if (this.journey.length % 5 === 0) {
      this.flush()
    }
  }

  flush() {
    if (this.journey.length === 0) return

    fetch('/api/analytics/user-journey', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        journey: this.journey,
        sessionId: getSessionId(),
      }),
    })

    this.journey = []
  }

  getJourney() {
    return [...this.journey]
  }
}

const journeyTracker = new UserJourneyTracker()

// Use in app
function App() {
  const router = useRouter()

  useEffect(() => {
    return router.subscribe('onResolved', () => {
      journeyTracker.trackNavigation(
        router.state.location.pathname,
        router.state.matches[0]?.params
      )
    })
  }, [router])

  // Flush on page unload
  useEffect(() => {
    const handleUnload = () => journeyTracker.flush()
    window.addEventListener('beforeunload', handleUnload)
    return () => window.removeEventListener('beforeunload', handleUnload)
  }, [])

  return <RouterProvider router={router} />
}
```

### Error Tracking

```typescript
// Example 3: Route error monitoring
function useRouteErrorTracking() {
  const router = useRouter()

  useEffect(() => {
    return router.subscribe('onError', ({ error, location }) => {
      // Log to error tracking service
      if (window.Sentry) {
        window.Sentry.captureException(error, {
          tags: {
            route: location.pathname,
            type: 'route_error',
          },
          extra: {
            search: location.search,
            params: router.state.matches[0]?.params,
          },
        })
      }

      // Custom error logging
      fetch('/api/errors', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          error: {
            message: error.message,
            stack: error.stack,
          },
          route: location.pathname,
          timestamp: Date.now(),
        }),
      })
    })
  }, [router])
}
```

---

## 7. Migration Strategies

### React Router Migration

```typescript
// Example 1: Gradual migration from React Router
// Step 1: Install both routers
import { BrowserRouter, Routes, Route } from 'react-router-dom'
import { RouterProvider, createRouter } from '@tanstack/react-router'

// Step 2: Create TanStack Router for new routes
const tanstackRouter = createRouter({
  routeTree: newRoutesTree,
})

// Step 3: Hybrid router setup
function App() {
  const [useTanStackRouter, setUseTanStackRouter] = useState(false)

  // Feature flag to switch routers
  useEffect(() => {
    const shouldUseTanStack = localStorage.getItem('use-tanstack-router') === 'true'
    setUseTanStackRouter(shouldUseTanStack)
  }, [])

  if (useTanStackRouter) {
    return <RouterProvider router={tanstackRouter} />
  }

  return (
    <BrowserRouter>
      <Routes>
        {/* Existing React Router routes */}
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />

        {/* Gradually migrate routes */}
        <Route
          path="/products/*"
          element={<TanStackRouterBridge router={tanstackRouter} />}
        />
      </Routes>
    </BrowserRouter>
  )
}

// Bridge component
function TanStackRouterBridge({ router }: { router: Router }) {
  return <RouterProvider router={router} />
}
```

```typescript
// Example 2: Route mapping helper
interface RouteMapping {
  reactRouter: string
  tanstackRouter: string
  params?: Record<string, string>
}

const routeMappings: RouteMapping[] = [
  {
    reactRouter: '/products/:id',
    tanstackRouter: '/products/$id',
  },
  {
    reactRouter: '/users/:userId/posts/:postId',
    tanstackRouter: '/users/$userId/posts/$postId',
  },
]

function convertReactRouterPath(path: string): string {
  return path.replace(/:(\w+)/g, '$$1')
}

// Migration script
function migrateRoutes() {
  routeMappings.forEach(mapping => {
    console.log(`Migrating ${mapping.reactRouter} → ${mapping.tanstackRouter}`)
    // Generate new route file
    generateTanStackRoute(mapping)
  })
}
```

### Incremental Adoption

```typescript
// Example 3: Feature-by-feature migration
// Phase 1: New features use TanStack Router
const newFeaturesRouter = createRouter({
  routeTree: createRoute({
    path: '/new-features',
    component: NewFeaturesLayout,
  }).addChildren([
    createRoute({
      path: 'dashboard',
      component: NewDashboard,
    }),
  ]),
})

// Phase 2: Migrate high-traffic routes
const migratedRouter = createRouter({
  routeTree: rootRoute.addChildren([
    // Migrated routes
    productsRoute,
    checkoutRoute,

    // Legacy fallback
    createRoute({
      path: '*',
      component: () => <LegacyRouterFallback />,
    }),
  ]),
})

// Phase 3: Complete migration
const finalRouter = createRouter({
  routeTree: completeRouteTree,
})
```

### Backward Compatibility

```typescript
// Example 4: Maintain URL compatibility
// Old React Router: /products/:id/edit
// New TanStack Router: /products/$id/edit

export const Route = createFileRoute('/products/$id/edit')({
  // Handle old URL format
  beforeLoad: ({ params, location }) => {
    // Check if using old format
    const oldFormatMatch = location.pathname.match(/\/products\/(\d+)\/edit/)

    if (oldFormatMatch && !params.id) {
      // Redirect to new format
      throw redirect({
        to: '/products/$id/edit',
        params: { id: oldFormatMatch[1] },
      })
    }
  },
  component: ProductEdit,
})
```

---

## 8. Performance Tuning

### Bundle Size Optimization

```typescript
// Example 1: Analyze and optimize bundles
// vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer'

export default defineConfig({
  plugins: [
    visualizer({
      open: true,
      gzipSize: true,
      brotliSize: true,
    }),
  ],
  build: {
    rollupOptions: {
      output: {
        manualChunks: (id) => {
          // Optimize chunking strategy
          if (id.includes('node_modules')) {
            if (id.includes('@tanstack/react-router')) {
              return 'router'
            }
            if (id.includes('react')) {
              return 'react-vendor'
            }
            return 'vendor'
          }

          // Route-based chunks
          if (id.includes('/routes/')) {
            const match = id.match(/routes\/(.+?)\//)
            return match ? `route-${match[1]}` : 'route'
          }
        },
      },
    },
  },
})
```

### Runtime Performance

```typescript
// Example 2: Optimize route matching
import { createRouter } from '@tanstack/react-router'
import { useMemo } from 'react'

// Memoize router creation
const router = useMemo(
  () => createRouter({
    routeTree,

    // Optimize route matching
    defaultPreload: 'intent',
    defaultPreloadDelay: 100,

    // Reduce re-renders
    defaultPendingMs: 1000,
    defaultPendingMinMs: 500,
  }),
  []
)

// Optimize route components
const ProductsList = memo(function ProductsList() {
  const { products } = Route.useLoaderData()

  // Virtualize long lists
  return (
    <VirtualList
      items={products}
      height={600}
      itemHeight={100}
      renderItem={(product) => <ProductCard product={product} />}
    />
  )
})
```

### Memory Management

```typescript
// Example 3: Prevent memory leaks
function useRouterCleanup() {
  const router = useRouter()

  useEffect(() => {
    // Cleanup old route data
    const interval = setInterval(() => {
      const now = Date.now()
      const maxAge = 5 * 60 * 1000 // 5 minutes

      // Clear old cached routes
      router.state.matches.forEach(match => {
        if (match.updatedAt && now - match.updatedAt > maxAge) {
          router.invalidate({ to: match.routeId })
        }
      })
    }, 60000) // Check every minute

    return () => clearInterval(interval)
  }, [router])
}

// Cleanup on unmount
function App() {
  useRouterCleanup()

  useEffect(() => {
    return () => {
      // Clear all router state on unmount
      router.invalidate()
    }
  }, [])

  return <RouterProvider router={router} />
}
```

---

## Best Practices - Principal Level

### 1. Architecture

```typescript
// ✅ GOOD: Modular, scalable architecture
features/
├── products/
│   ├── routes.ts
│   ├── services.ts
│   └── store.ts
└── auth/
    ├── routes.ts
    └── services.ts

// ❌ BAD: Monolithic route file
routes.ts (5000+ lines)
```

### 2. Code Splitting

```typescript
// ✅ GOOD: Strategic splitting
manualChunks: {
  'react-vendor': ['react', 'react-dom'],
  'router-vendor': ['@tanstack/react-router'],
}

// ❌ BAD: No splitting strategy
```

### 3. SSR/SSG

```typescript
// ✅ GOOD: Proper hydration
router.hydrate(dehydratedState)

// ❌ BAD: Client-only rendering for static content
```

### 4. Monitoring

```typescript
// ✅ GOOD: Comprehensive monitoring
- Route performance tracking
- Error tracking
- User journey analytics

// ❌ BAD: No monitoring
```

### 5. Migration

```typescript
// ✅ GOOD: Incremental migration
- Feature flags
- Gradual rollout
- Backward compatibility

// ❌ BAD: Big bang migration
```

---

## Common Pitfalls - Principal Level

### ❌ Pitfall 1: Over-engineering

```typescript
// BAD: Unnecessary complexity
class RouteManager {
  private registry: Map<string, Route>
  private middleware: Middleware[]
  private interceptors: Interceptor[]
  // ... 500 more lines
}

// GOOD: Use built-in features
const router = createRouter({ routeTree })
```

### ❌ Pitfall 2: Không optimize bundle

```typescript
// BAD: Import everything
import * as TanStackRouter from '@tanstack/react-router'

// GOOD: Import only what you need
import { createRouter, RouterProvider } from '@tanstack/react-router'
```

### ❌ Pitfall 3: Memory leaks trong SSR

```typescript
// BAD: Reuse router instance
const router = createRouter({ routeTree })

app.get('*', (req, res) => {
  // Reusing same router causes memory leaks
  renderToString(<RouterProvider router={router} />)
})

// GOOD: Create new router per request
app.get('*', (req, res) => {
  const ssrRouter = router.update({
    history: createMemoryHistory({ initialEntries: [req.url] }),
  })
  renderToString(<RouterProvider router={ssrRouter} />)
})
```

---

## Tài Liệu Tham Khảo

### Official Documentation
- [TanStack Router Docs](https://tanstack.com/router/latest)
- [TanStack Router GitHub](https://github.com/tanstack/router)
- [SSR Guide](https://tanstack.com/router/latest/docs/framework/react/guide/ssr)

### Tools & Libraries
- [Vite Plugin](https://tanstack.com/router/latest/docs/framework/react/guide/file-based-routing)
- [Module Federation](https://webpack.js.org/concepts/module-federation/)
- [Sentry](https://sentry.io/) - Error tracking
- [Datadog](https://www.datadoghq.com/) - Performance monitoring

### Community Resources
- [TanStack Discord](https://discord.com/invite/tanstack)
- [GitHub Discussions](https://github.com/TanStack/router/discussions)

---

**Tài liệu này bao gồm Principal level patterns. Để học các patterns cơ bản hơn, hãy xem các files khác trong thư mục này.**

