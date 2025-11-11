# Principal Zustand Patterns - Enterprise Level

## Mục Lục

- [1. Large-Scale Architecture](#1-large-scale-architecture)
- [2. SSR/SSG Support](#2-ssrssg-support)
- [3. Cross-Tab Synchronization](#3-cross-tab-synchronization)
- [4. Performance Monitoring](#4-performance-monitoring)
- [5. Micro-Frontends Integration](#5-micro-frontends-integration)
- [6. Migration Strategies](#6-migration-strategies)
- [7. Production Optimization](#7-production-optimization)

---

## 1. Large-Scale Architecture

### Domain-Driven Store Design

```typescript
// Example 1: Domain-based store architecture
// src/stores/domains/auth/types.ts
export interface AuthDomain {
  // State
  user: User | null
  session: Session | null
  permissions: Permission[]
  
  // Queries
  isAuthenticated: () => boolean
  hasPermission: (permission: string) => boolean
  
  // Commands
  login: (credentials: Credentials) => Promise<void>
  logout: () => Promise<void>
  refreshSession: () => Promise<void>
}

// src/stores/domains/auth/slice.ts
import { StateCreator } from 'zustand'
import { AuthDomain } from './types'
import { authService } from '@/services/auth'

export const createAuthDomain: StateCreator<
  RootStore,
  [],
  [],
  AuthDomain
> = (set, get) => ({
  // State
  user: null,
  session: null,
  permissions: [],
  
  // Queries
  isAuthenticated: () => {
    const { user, session } = get()
    return !!user && !!session && session.expiresAt > Date.now()
  },
  
  hasPermission: (permission) => {
    return get().permissions.some((p) => p.name === permission)
  },
  
  // Commands
  login: async (credentials) => {
    try {
      const { user, session, permissions } = await authService.login(credentials)
      
      set({ user, session, permissions })
      
      // Cross-domain communication
      get().analytics.trackEvent('user_login', { userId: user.id })
      get().notifications.addNotification('Login successful', 'success')
    } catch (error) {
      get().notifications.addNotification('Login failed', 'error')
      throw error
    }
  },
  
  logout: async () => {
    const userId = get().user?.id
    
    await authService.logout()
    set({ user: null, session: null, permissions: [] })
    
    // Cleanup other domains
    get().cart.clearCart()
    get().preferences.resetToDefaults()
    
    get().analytics.trackEvent('user_logout', { userId })
  },
  
  refreshSession: async () => {
    const currentSession = get().session
    if (!currentSession) return
    
    try {
      const newSession = await authService.refreshSession(currentSession.refreshToken)
      set({ session: newSession })
    } catch (error) {
      // Session refresh failed, logout user
      get().logout()
    }
  },
})

// src/stores/domains/cart/types.ts
export interface CartDomain {
  // State
  items: CartItem[]
  
  // Queries
  getTotal: () => number
  getItemCount: () => number
  
  // Commands
  addItem: (item: CartItem) => void
  removeItem: (id: string) => void
  updateQuantity: (id: string, quantity: number) => void
  clearCart: () => void
  checkout: () => Promise<void>
}

// src/stores/domains/cart/slice.ts
export const createCartDomain: StateCreator<
  RootStore,
  [],
  [],
  CartDomain
> = (set, get) => ({
  items: [],
  
  getTotal: () => {
    return get().items.reduce((sum, item) => sum + item.price * item.quantity, 0)
  },
  
  getItemCount: () => {
    return get().items.reduce((sum, item) => sum + item.quantity, 0)
  },
  
  addItem: (item) => {
    const existingItem = get().items.find((i) => i.id === item.id)
    
    if (existingItem) {
      get().updateQuantity(item.id, existingItem.quantity + 1)
    } else {
      set((state) => ({
        items: [...state.items, { ...item, quantity: 1 }],
      }))
      
      get().analytics.trackEvent('cart_add_item', { itemId: item.id })
    }
  },
  
  removeItem: (id) => {
    set((state) => ({
      items: state.items.filter((item) => item.id !== id),
    }))
    
    get().analytics.trackEvent('cart_remove_item', { itemId: id })
  },
  
  updateQuantity: (id, quantity) => {
    if (quantity <= 0) {
      get().removeItem(id)
      return
    }
    
    set((state) => ({
      items: state.items.map((item) =>
        item.id === id ? { ...item, quantity } : item
      ),
    }))
  },
  
  clearCart: () => set({ items: [] }),
  
  checkout: async () => {
    if (!get().isAuthenticated()) {
      get().notifications.addNotification('Please login to checkout', 'error')
      return
    }
    
    const items = get().items
    const total = get().getTotal()
    
    try {
      await fetch('/api/checkout', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${get().session?.accessToken}`,
        },
        body: JSON.stringify({ items, total }),
      })
      
      get().clearCart()
      get().notifications.addNotification('Checkout successful', 'success')
      get().analytics.trackEvent('checkout_complete', { total, itemCount: items.length })
    } catch (error) {
      get().notifications.addNotification('Checkout failed', 'error')
      get().analytics.trackEvent('checkout_failed', { error })
      throw error
    }
  },
})

// src/stores/index.ts - Root store composition
import { create } from 'zustand'
import { devtools, persist } from 'zustand/middleware'
import { createAuthDomain, AuthDomain } from './domains/auth/slice'
import { createCartDomain, CartDomain } from './domains/cart/slice'
import { createAnalyticsDomain, AnalyticsDomain } from './domains/analytics/slice'
import { createNotificationsDomain, NotificationsDomain } from './domains/notifications/slice'

type RootStore = AuthDomain & CartDomain & AnalyticsDomain & NotificationsDomain

export const useStore = create<RootStore>()(
  devtools(
    persist(
      (...a) => ({
        ...createAuthDomain(...a),
        ...createCartDomain(...a),
        ...createAnalyticsDomain(...a),
        ...createNotificationsDomain(...a),
      }),
      {
        name: 'app-store',
        partialize: (state) => ({
          // Only persist auth and cart
          user: state.user,
          session: state.session,
          permissions: state.permissions,
          items: state.items,
        }),
      }
    ),
    { name: 'RootStore' }
  )
)

// Domain-specific hooks for better DX
export const useAuth = () => useStore((state) => ({
  user: state.user,
  session: state.session,
  permissions: state.permissions,
  isAuthenticated: state.isAuthenticated,
  hasPermission: state.hasPermission,
  login: state.login,
  logout: state.logout,
  refreshSession: state.refreshSession,
}))

export const useCart = () => useStore((state) => ({
  items: state.items,
  getTotal: state.getTotal,
  getItemCount: state.getItemCount,
  addItem: state.addItem,
  removeItem: state.removeItem,
  updateQuantity: state.updateQuantity,
  clearCart: state.clearCart,
  checkout: state.checkout,
}))
```

**Benefits:**
- Clear domain boundaries
- Type-safe cross-domain communication
- Easy to test individual domains
- Scalable architecture
- Domain-specific hooks for better DX

### Store Factory Pattern

```typescript
// Example 2: Reusable store factory
import { create, StoreApi, UseBoundStore } from 'zustand'
import { devtools, persist } from 'zustand/middleware'

interface EntityState<T> {
  entities: Record<string, T>
  ids: string[]
  loading: boolean
  error: string | null
}

interface EntityActions<T> {
  fetchAll: () => Promise<void>
  fetchById: (id: string) => Promise<void>
  create: (entity: Omit<T, 'id'>) => Promise<void>
  update: (id: string, updates: Partial<T>) => Promise<void>
  delete: (id: string) => Promise<void>
  reset: () => void
}

type EntityStore<T> = EntityState<T> & EntityActions<T>

interface EntityStoreConfig<T> {
  name: string
  apiEndpoint: string
  persist?: boolean
}

export function createEntityStore<T extends { id: string }>(
  config: EntityStoreConfig<T>
): UseBoundStore<StoreApi<EntityStore<T>>> {
  const initialState: EntityState<T> = {
    entities: {},
    ids: [],
    loading: false,
    error: null,
  }

  const storeCreator = (set: any, get: any) => ({
    ...initialState,

    fetchAll: async () => {
      set({ loading: true, error: null })

      try {
        const response = await fetch(config.apiEndpoint)
        const data: T[] = await response.json()

        const entities = data.reduce((acc, item) => {
          acc[item.id] = item
          return acc
        }, {} as Record<string, T>)

        const ids = data.map((item) => item.id)

        set({ entities, ids, loading: false })
      } catch (error) {
        set({
          error: error instanceof Error ? error.message : 'Unknown error',
          loading: false,
        })
      }
    },

    fetchById: async (id: string) => {
      set({ loading: true, error: null })

      try {
        const response = await fetch(`${config.apiEndpoint}/${id}`)
        const entity: T = await response.json()

        set((state: EntityState<T>) => ({
          entities: { ...state.entities, [entity.id]: entity },
          ids: state.ids.includes(entity.id) ? state.ids : [...state.ids, entity.id],
          loading: false,
        }))
      } catch (error) {
        set({
          error: error instanceof Error ? error.message : 'Unknown error',
          loading: false,
        })
      }
    },

    create: async (entity: Omit<T, 'id'>) => {
      set({ loading: true, error: null })

      try {
        const response = await fetch(config.apiEndpoint, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(entity),
        })

        const newEntity: T = await response.json()

        set((state: EntityState<T>) => ({
          entities: { ...state.entities, [newEntity.id]: newEntity },
          ids: [...state.ids, newEntity.id],
          loading: false,
        }))
      } catch (error) {
        set({
          error: error instanceof Error ? error.message : 'Unknown error',
          loading: false,
        })
      }
    },

    update: async (id: string, updates: Partial<T>) => {
      set({ loading: true, error: null })

      try {
        const response = await fetch(`${config.apiEndpoint}/${id}`, {
          method: 'PATCH',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(updates),
        })

        const updatedEntity: T = await response.json()

        set((state: EntityState<T>) => ({
          entities: { ...state.entities, [id]: updatedEntity },
          loading: false,
        }))
      } catch (error) {
        set({
          error: error instanceof Error ? error.message : 'Unknown error',
          loading: false,
        })
      }
    },

    delete: async (id: string) => {
      set({ loading: true, error: null })

      try {
        await fetch(`${config.apiEndpoint}/${id}`, {
          method: 'DELETE',
        })

        set((state: EntityState<T>) => ({
          entities: Object.fromEntries(
            Object.entries(state.entities).filter(([key]) => key !== id)
          ),
          ids: state.ids.filter((entityId) => entityId !== id),
          loading: false,
        }))
      } catch (error) {
        set({
          error: error instanceof Error ? error.message : 'Unknown error',
          loading: false,
        })
      }
    },

    reset: () => set(initialState),
  })

  const store = create<EntityStore<T>>()(
    devtools(
      config.persist
        ? persist(storeCreator, { name: `${config.name}-store` })
        : storeCreator,
      { name: config.name }
    )
  )

  return store
}

// Usage
interface Product {
  id: string
  name: string
  price: number
  category: string
}

export const useProductStore = createEntityStore<Product>({
  name: 'Products',
  apiEndpoint: '/api/products',
  persist: true,
})

interface User {
  id: string
  name: string
  email: string
}

export const useUserStore = createEntityStore<User>({
  name: 'Users',
  apiEndpoint: '/api/users',
  persist: false,
})
```

---

## 2. SSR/SSG Support

### Next.js App Router Integration

```typescript
// Example 1: Server-side store initialization
// src/stores/useProductStore.ts
import { create } from 'zustand'

interface ProductStore {
  products: Product[]
  setProducts: (products: Product[]) => void
}

export const useProductStore = create<ProductStore>((set) => ({
  products: [],
  setProducts: (products) => set({ products }),
}))

// Server Component
// app/products/page.tsx
import { useProductStore } from '@/stores/useProductStore'

async function getProducts() {
  const res = await fetch('https://api.example.com/products', {
    cache: 'no-store',
  })
  return res.json()
}

export default async function ProductsPage() {
  const products = await getProducts()

  // Initialize store on server
  useProductStore.setState({ products })

  return <ProductList />
}

// Client Component
// components/ProductList.tsx
'use client'

import { useProductStore } from '@/stores/useProductStore'

export function ProductList() {
  const products = useProductStore((state) => state.products)

  return (
    <ul>
      {products.map((product) => (
        <li key={product.id}>{product.name}</li>
      ))}
    </ul>
  )
}
```

### Hydration Pattern

```typescript
// Example 2: Proper hydration handling
// src/stores/useHydratedStore.ts
import { create } from 'zustand'
import { persist, createJSONStorage } from 'zustand/middleware'
import { useEffect, useState } from 'react'

interface Store {
  count: number
  increment: () => void
}

export const useStore = create<Store>()(
  persist(
    (set) => ({
      count: 0,
      increment: () => set((state) => ({ count: state.count + 1 })),
    }),
    {
      name: 'store',
      storage: createJSONStorage(() => localStorage),
    }
  )
)

// Hook to handle hydration
export function useHydratedStore<T, F>(
  store: (callback: (state: T) => unknown) => unknown,
  callback: (state: T) => F
) {
  const result = store(callback) as F
  const [hydrated, setHydrated] = useState(false)

  useEffect(() => {
    setHydrated(true)
  }, [])

  return hydrated ? result : null
}

// Usage
function Counter() {
  const count = useHydratedStore(useStore, (state) => state.count)
  const increment = useStore((state) => state.increment)

  if (count === null) {
    return <div>Loading...</div>
  }

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={increment}>Increment</button>
    </div>
  )
}
```

### SSG with Static Props

```typescript
// Example 3: Static generation with Zustand
// pages/products/[id].tsx (Pages Router)
import { GetStaticProps, GetStaticPaths } from 'next'
import { useProductStore } from '@/stores/useProductStore'

export const getStaticPaths: GetStaticPaths = async () => {
  const products = await fetchAllProducts()

  return {
    paths: products.map((product) => ({
      params: { id: product.id },
    })),
    fallback: 'blocking',
  }
}

export const getStaticProps: GetStaticProps = async ({ params }) => {
  const product = await fetchProduct(params?.id as string)

  return {
    props: {
      product,
    },
    revalidate: 60, // ISR: Revalidate every 60 seconds
  }
}

export default function ProductPage({ product }: { product: Product }) {
  // Initialize store with static data
  useEffect(() => {
    useProductStore.setState({ currentProduct: product })
  }, [product])

  return <ProductDetails />
}
```

---

## 3. Cross-Tab Synchronization

### BroadcastChannel API

```typescript
// Example 1: Sync state across tabs
import { create } from 'zustand'
import { persist, createJSONStorage, StateStorage } from 'zustand/middleware'

// Custom storage with BroadcastChannel
const createBroadcastStorage = (name: string): StateStorage => {
  const channel = typeof window !== 'undefined'
    ? new BroadcastChannel(name)
    : null

  return {
    getItem: (key: string): string | null => {
      const value = localStorage.getItem(key)
      return value
    },

    setItem: (key: string, value: string): void => {
      localStorage.setItem(key, value)

      // Broadcast to other tabs
      channel?.postMessage({
        type: 'STATE_UPDATE',
        key,
        value,
      })
    },

    removeItem: (key: string): void => {
      localStorage.removeItem(key)

      channel?.postMessage({
        type: 'STATE_REMOVE',
        key,
      })
    },
  }
}

interface SyncedStore {
  count: number
  increment: () => void
  decrement: () => void
}

export const useSyncedStore = create<SyncedStore>()(
  persist(
    (set) => ({
      count: 0,
      increment: () => set((state) => ({ count: state.count + 1 })),
      decrement: () => set((state) => ({ count: state.count - 1 })),
    }),
    {
      name: 'synced-store',
      storage: createJSONStorage(() => createBroadcastStorage('app-sync')),
    }
  )
)

// Listen for updates from other tabs
if (typeof window !== 'undefined') {
  const channel = new BroadcastChannel('app-sync')

  channel.addEventListener('message', (event) => {
    if (event.data.type === 'STATE_UPDATE') {
      const state = JSON.parse(event.data.value)
      useSyncedStore.setState(state)
    }
  })
}
```

### Storage Event Listener

```typescript
// Example 2: Alternative sync using storage events
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface SyncStore {
  data: any
  updateData: (data: any) => void
}

export const useSyncStore = create<SyncStore>()(
  persist(
    (set) => ({
      data: null,
      updateData: (data) => set({ data }),
    }),
    {
      name: 'sync-store',
    }
  )
)

// Listen for storage changes from other tabs
if (typeof window !== 'undefined') {
  window.addEventListener('storage', (event) => {
    if (event.key === 'sync-store' && event.newValue) {
      const newState = JSON.parse(event.newValue)
      useSyncStore.setState(newState.state)
    }
  })
}
```

---

## 4. Performance Monitoring

### Store Performance Tracking

```typescript
// Example 1: Performance monitoring middleware
import { StateCreator, StoreMutatorIdentifier } from 'zustand'

interface PerformanceMetrics {
  actionName: string
  duration: number
  timestamp: number
  stateSize: number
}

type PerformanceMonitor = <
  T,
  Mps extends [StoreMutatorIdentifier, unknown][] = [],
  Mcs extends [StoreMutatorIdentifier, unknown][] = []
>(
  f: StateCreator<T, Mps, Mcs>,
  options?: {
    onMetric?: (metric: PerformanceMetrics) => void
    threshold?: number
  }
) => StateCreator<T, Mps, Mcs>

const performanceMonitorImpl: PerformanceMonitor = (f, options = {}) => (set, get, store) => {
  const { onMetric, threshold = 100 } = options

  const monitoredSet: typeof set = (...args) => {
    const startTime = performance.now()
    const actionName = args[2] as string || 'unknown'

    set(...args)

    const endTime = performance.now()
    const duration = endTime - startTime
    const stateSize = JSON.stringify(get()).length

    const metric: PerformanceMetrics = {
      actionName,
      duration,
      timestamp: Date.now(),
      stateSize,
    }

    // Log slow actions
    if (duration > threshold) {
      console.warn(`Slow action detected: ${actionName} took ${duration.toFixed(2)}ms`)
    }

    onMetric?.(metric)
  }

  return f(monitoredSet, get, store)
}

export const performanceMonitor = performanceMonitorImpl as PerformanceMonitor

// Usage
import { create } from 'zustand'

interface Store {
  items: Item[]
  addItem: (item: Item) => void
}

export const useStore = create<Store>()(
  performanceMonitor(
    (set) => ({
      items: [],
      addItem: (item) =>
        set(
          (state) => ({ items: [...state.items, item] }),
          undefined,
          'addItem'
        ),
    }),
    {
      threshold: 50, // Warn if action takes > 50ms
      onMetric: (metric) => {
        // Send to analytics
        analytics.track('store_performance', metric)
      },
    }
  )
)
```

### Memory Leak Detection

```typescript
// Example 2: Detect memory leaks
import { create } from 'zustand'

interface MemoryMonitor {
  subscriptions: Set<() => void>
  checkLeaks: () => void
}

const memoryMonitor: MemoryMonitor = {
  subscriptions: new Set(),

  checkLeaks: () => {
    if (memoryMonitor.subscriptions.size > 100) {
      console.warn(
        `Potential memory leak: ${memoryMonitor.subscriptions.size} active subscriptions`
      )
    }
  },
}

// Wrap subscribe to track subscriptions
export function createMonitoredStore<T>(storeCreator: any) {
  const store = create<T>(storeCreator)

  const originalSubscribe = store.subscribe

  store.subscribe = (...args: any[]) => {
    const unsubscribe = originalSubscribe(...args)

    memoryMonitor.subscriptions.add(unsubscribe)

    return () => {
      unsubscribe()
      memoryMonitor.subscriptions.delete(unsubscribe)
    }
  }

  // Check for leaks every 10 seconds
  if (typeof window !== 'undefined') {
    setInterval(() => {
      memoryMonitor.checkLeaks()
    }, 10000)
  }

  return store
}
```

### State Change Analytics

```typescript
// Example 3: Track state changes
import { create } from 'zustand'
import { subscribeWithSelector } from 'zustand/middleware'

interface AnalyticsStore {
  events: AnalyticsEvent[]
  trackEvent: (event: AnalyticsEvent) => void
}

export const useAnalyticsStore = create<AnalyticsStore>()(
  subscribeWithSelector((set) => ({
    events: [],
    trackEvent: (event) =>
      set((state) => ({
        events: [...state.events, event],
      })),
  }))
)

// Track specific state changes
export function setupStateTracking<T>(
  store: any,
  selector: (state: T) => any,
  eventName: string
) {
  store.subscribe(
    selector,
    (value: any, previousValue: any) => {
      useAnalyticsStore.getState().trackEvent({
        name: eventName,
        timestamp: Date.now(),
        data: {
          previous: previousValue,
          current: value,
        },
      })
    }
  )
}

// Usage
setupStateTracking(
  useCartStore,
  (state) => state.items.length,
  'cart_item_count_changed'
)

setupStateTracking(
  useAuthStore,
  (state) => state.user,
  'user_changed'
)
```

---

## 5. Micro-Frontends Integration

### Shared State Across Micro-Frontends

```typescript
// Example 1: Cross-app state sharing
// packages/shared-store/src/index.ts
import { create } from 'zustand'
import { persist, createJSONStorage } from 'zustand/middleware'

interface SharedStore {
  user: User | null
  theme: 'light' | 'dark'
  setUser: (user: User | null) => void
  setTheme: (theme: 'light' | 'dark') => void
}

// Shared store accessible by all micro-frontends
export const useSharedStore = create<SharedStore>()(
  persist(
    (set) => ({
      user: null,
      theme: 'light',
      setUser: (user) => set({ user }),
      setTheme: (theme) => set({ theme }),
    }),
    {
      name: 'shared-store',
      storage: createJSONStorage(() => localStorage),
    }
  )
)

// App 1 (Header micro-frontend)
// apps/header/src/components/UserMenu.tsx
import { useSharedStore } from '@shared/store'

export function UserMenu() {
  const user = useSharedStore((state) => state.user)
  const setUser = useSharedStore((state) => state.setUser)

  const handleLogout = () => {
    setUser(null)
  }

  return (
    <div>
      {user ? (
        <>
          <span>{user.name}</span>
          <button onClick={handleLogout}>Logout</button>
        </>
      ) : (
        <button>Login</button>
      )}
    </div>
  )
}

// App 2 (Dashboard micro-frontend)
// apps/dashboard/src/components/Welcome.tsx
import { useSharedStore } from '@shared/store'

export function Welcome() {
  const user = useSharedStore((state) => state.user)

  return <h1>Welcome, {user?.name || 'Guest'}!</h1>
}
```

### Module Federation

```typescript
// Example 2: Zustand with Webpack Module Federation
// host/webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host',
      remotes: {
        app1: 'app1@http://localhost:3001/remoteEntry.js',
        app2: 'app2@http://localhost:3002/remoteEntry.js',
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true },
        zustand: { singleton: true }, // Share Zustand instance
      },
    }),
  ],
}

// remote/webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'app1',
      filename: 'remoteEntry.js',
      exposes: {
        './Store': './src/store',
        './Component': './src/Component',
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true },
        zustand: { singleton: true },
      },
    }),
  ],
}
```

---

## 6. Migration Strategies

### From Redux to Zustand

```typescript
// Example 1: Redux to Zustand migration
// Before (Redux)
// store/slices/counterSlice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit'

interface CounterState {
  value: number
}

const initialState: CounterState = {
  value: 0,
}

export const counterSlice = createSlice({
  name: 'counter',
  initialState,
  reducers: {
    increment: (state) => {
      state.value += 1
    },
    decrement: (state) => {
      state.value -= 1
    },
    incrementByAmount: (state, action: PayloadAction<number>) => {
      state.value += action.payload
    },
  },
})

export const { increment, decrement, incrementByAmount } = counterSlice.actions
export default counterSlice.reducer

// After (Zustand)
// stores/useCounterStore.ts
import { create } from 'zustand'
import { devtools } from 'zustand/middleware'

interface CounterStore {
  value: number
  increment: () => void
  decrement: () => void
  incrementByAmount: (amount: number) => void
}

export const useCounterStore = create<CounterStore>()(
  devtools(
    (set) => ({
      value: 0,

      increment: () =>
        set(
          (state) => ({ value: state.value + 1 }),
          undefined,
          'counter/increment'
        ),

      decrement: () =>
        set(
          (state) => ({ value: state.value - 1 }),
          undefined,
          'counter/decrement'
        ),

      incrementByAmount: (amount) =>
        set(
          (state) => ({ value: state.value + amount }),
          undefined,
          'counter/incrementByAmount'
        ),
    }),
    { name: 'CounterStore' }
  )
)

// Migration helper for gradual migration
export function createReduxCompatibleStore<T>(
  storeCreator: any
) {
  const store = create<T>(storeCreator)

  // Redux-like dispatch function
  const dispatch = (action: { type: string; payload?: any }) => {
    const state = store.getState() as any
    const actionFn = state[action.type]

    if (typeof actionFn === 'function') {
      actionFn(action.payload)
    }
  }

  return { ...store, dispatch }
}
```

### From Context API to Zustand

```typescript
// Example 2: Context API to Zustand migration
// Before (Context API)
// contexts/AuthContext.tsx
import { createContext, useContext, useState, ReactNode } from 'react'

interface AuthContextType {
  user: User | null
  login: (user: User) => void
  logout: () => void
}

const AuthContext = createContext<AuthContextType | undefined>(undefined)

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null)

  const login = (user: User) => setUser(user)
  const logout = () => setUser(null)

  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  )
}

export function useAuth() {
  const context = useContext(AuthContext)
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider')
  }
  return context
}

// After (Zustand)
// stores/useAuthStore.ts
import { create } from 'zustand'

interface AuthStore {
  user: User | null
  login: (user: User) => void
  logout: () => void
}

export const useAuthStore = create<AuthStore>((set) => ({
  user: null,
  login: (user) => set({ user }),
  logout: () => set({ user: null }),
}))

// No Provider needed!
// Just use the hook directly in components
```

### Coexistence Strategy

```typescript
// Example 3: Run Redux and Zustand side-by-side
// Gradually migrate features one by one

// Feature still in Redux
import { useSelector, useDispatch } from 'react-redux'

function LegacyFeature() {
  const data = useSelector((state) => state.legacy.data)
  const dispatch = useDispatch()

  return <div>{data}</div>
}

// New feature in Zustand
import { useNewFeatureStore } from '@/stores/useNewFeatureStore'

function NewFeature() {
  const data = useNewFeatureStore((state) => state.data)

  return <div>{data}</div>
}

// Bridge between Redux and Zustand
import { useEffect } from 'react'
import { useSelector } from 'react-redux'
import { useAuthStore } from '@/stores/useAuthStore'

export function ReduxZustandBridge() {
  const reduxUser = useSelector((state) => state.auth.user)
  const setZustandUser = useAuthStore((state) => state.setUser)

  useEffect(() => {
    // Sync Redux user to Zustand
    setZustandUser(reduxUser)
  }, [reduxUser, setZustandUser])

  return null
}
```

---

## 7. Production Optimization

### Code Splitting

```typescript
// Example 1: Lazy load store slices
// stores/index.ts
import { create } from 'zustand'
import { devtools } from 'zustand/middleware'

interface CoreStore {
  // Core state always loaded
  user: User | null
  theme: string
}

export const useCoreStore = create<CoreStore>()(
  devtools((set) => ({
    user: null,
    theme: 'light',
  }))
)

// Lazy-loaded feature stores
export const useAdminStore = () =>
  import('./slices/adminSlice').then((m) => m.useAdminStore)

export const useAnalyticsStore = () =>
  import('./slices/analyticsSlice').then((m) => m.useAnalyticsStore)

// Usage
function AdminPanel() {
  const [AdminStore, setAdminStore] = useState<any>(null)

  useEffect(() => {
    useAdminStore().then(setAdminStore)
  }, [])

  if (!AdminStore) return <div>Loading...</div>

  return <AdminPanelContent store={AdminStore} />
}
```

### Bundle Size Optimization

```typescript
// Example 2: Tree-shakeable exports
// stores/index.ts
export { useAuthStore } from './auth'
export { useCartStore } from './cart'
export { useProductStore } from './product'

// Only import what you need
import { useAuthStore } from '@/stores' // Tree-shaken

// Avoid barrel exports for large stores
// ❌ BAD
export * from './stores'

// ✅ GOOD
export { useAuthStore } from './stores/auth'
export { useCartStore } from './stores/cart'
```

### Production Error Handling

```typescript
// Example 3: Robust error handling
import { create } from 'zustand'
import { devtools } from 'zustand/middleware'

interface ErrorBoundaryStore {
  error: Error | null
  setError: (error: Error | null) => void
  clearError: () => void
}

export const useErrorStore = create<ErrorBoundaryStore>()(
  devtools((set) => ({
    error: null,
    setError: (error) => {
      set({ error })

      // Log to error tracking service
      if (process.env.NODE_ENV === 'production') {
        // Sentry, LogRocket, etc.
        console.error('Store error:', error)
      }
    },
    clearError: () => set({ error: null }),
  }))
)

// Global error handler
if (typeof window !== 'undefined') {
  window.addEventListener('error', (event) => {
    useErrorStore.getState().setError(event.error)
  })

  window.addEventListener('unhandledrejection', (event) => {
    useErrorStore.getState().setError(new Error(event.reason))
  })
}
```

### Performance Budgets

```typescript
// Example 4: Enforce performance budgets
import { create } from 'zustand'

const MAX_STATE_SIZE = 1024 * 1024 // 1MB
const MAX_ACTION_TIME = 100 // 100ms

function enforcePerformanceBudgets<T>(storeCreator: any) {
  return (set: any, get: any, store: any) => {
    const wrappedSet: typeof set = (...args) => {
      const startTime = performance.now()

      set(...args)

      const endTime = performance.now()
      const duration = endTime - startTime

      // Check action time
      if (duration > MAX_ACTION_TIME) {
        console.warn(
          `Action exceeded time budget: ${duration.toFixed(2)}ms > ${MAX_ACTION_TIME}ms`
        )
      }

      // Check state size
      const stateSize = JSON.stringify(get()).length
      if (stateSize > MAX_STATE_SIZE) {
        console.error(
          `State exceeded size budget: ${(stateSize / 1024).toFixed(2)}KB > ${MAX_STATE_SIZE / 1024}KB`
        )
      }
    }

    return storeCreator(wrappedSet, get, store)
  }
}

// Usage
export const useStore = create<Store>()(
  enforcePerformanceBudgets((set) => ({
    // ... store
  }))
)
```

---

## Best Practices - Principal Level

### 1. Architecture

```typescript
// ✅ GOOD: Domain-driven design
stores/
  domains/
    auth/
      types.ts
      slice.ts
      selectors.ts
      __tests__/
    cart/
      types.ts
      slice.ts
      selectors.ts
      __tests__/
  index.ts

// ❌ BAD: Flat structure
stores/
  authStore.ts
  cartStore.ts
  productStore.ts
  userStore.ts
```

### 2. Performance

```typescript
// ✅ GOOD: Selective persistence
persist(
  (set) => ({ /* ... */ }),
  {
    name: 'store',
    partialize: (state) => ({
      user: state.user,
      settings: state.settings,
    }),
  }
)

// ❌ BAD: Persist everything
persist(
  (set) => ({ /* ... */ }),
  { name: 'store' }
)
```

### 3. Type Safety

```typescript
// ✅ GOOD: Strict typing
interface Store {
  count: number
  increment: () => void
}

const useStore = create<Store>()(...)

// ❌ BAD: Any types
const useStore = create((set: any) => ...)
```

---

## Tài Liệu Tham Khảo

### Official Documentation
- [Zustand Advanced Patterns](https://github.com/pmndrs/zustand/blob/main/docs/guides/practice-with-no-store-actions.md)
- [SSR Guide](https://github.com/pmndrs/zustand/blob/main/docs/guides/ssr-and-hydration.md)
- [Performance](https://github.com/pmndrs/zustand/blob/main/docs/guides/performance.md)

### Community Resources
- [Zustand Examples](https://github.com/pmndrs/zustand/tree/main/examples)
- [Best Practices](https://tkdodo.eu/blog/working-with-zustand)

---

**Tài liệu này bao gồm Principal level patterns cho enterprise applications. Kết hợp với Junior, Middle và Senior levels để có kiến thức toàn diện về Zustand.**
