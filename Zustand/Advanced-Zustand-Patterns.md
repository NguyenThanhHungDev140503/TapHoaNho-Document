# Advanced Zustand Patterns - Senior Level

## Mục Lục

- [1. Custom Middleware](#1-custom-middleware)
- [2. Advanced TypeScript Patterns](#2-advanced-typescript-patterns)
- [3. Slices Pattern](#3-slices-pattern)
- [4. Performance Optimization](#4-performance-optimization)
- [5. Testing Strategies](#5-testing-strategies)
- [6. State Persistence](#6-state-persistence)
- [7. Integration Patterns](#7-integration-patterns)

---

## 1. Custom Middleware

### Creating Custom Middleware

```typescript
// Example 1: Logger middleware
import { StateCreator, StoreMutatorIdentifier } from 'zustand'

type Logger = <
  T,
  Mps extends [StoreMutatorIdentifier, unknown][] = [],
  Mcs extends [StoreMutatorIdentifier, unknown][] = []
>(
  f: StateCreator<T, Mps, Mcs>,
  name?: string
) => StateCreator<T, Mps, Mcs>

type LoggerImpl = <T>(
  f: StateCreator<T, [], []>,
  name?: string
) => StateCreator<T, [], []>

const loggerImpl: LoggerImpl = (f, name) => (set, get, store) => {
  const loggedSet: typeof set = (...args) => {
    console.log(`[${name || 'Store'}] Previous state:`, get())
    set(...args)
    console.log(`[${name || 'Store'}] New state:`, get())
  }
  
  return f(loggedSet, get, store)
}

export const logger = loggerImpl as Logger

// Usage
import { create } from 'zustand'
import { logger } from './middleware/logger'

interface CountStore {
  count: number
  increment: () => void
}

export const useCountStore = create<CountStore>()(
  logger(
    (set) => ({
      count: 0,
      increment: () => set((state) => ({ count: state.count + 1 })),
    }),
    'CountStore'
  )
)
```

**Giải thích:**
- Custom middleware wraps `set` function
- Logs state before and after changes
- Type-safe với generics
- Reusable across stores

### Validation Middleware

```typescript
// Example 2: Validation middleware
import { StateCreator } from 'zustand'
import { z } from 'zod'

type Validator = <T>(
  schema: z.ZodSchema<T>
) => <Mps extends [StoreMutatorIdentifier, unknown][] = []>(
  f: StateCreator<T, Mps, []>
) => StateCreator<T, Mps, []>

export const validator: Validator = (schema) => (f) => (set, get, store) => {
  const validatedSet: typeof set = (...args) => {
    const nextState = typeof args[0] === 'function' 
      ? args[0](get()) 
      : args[0]
    
    const result = schema.safeParse({ ...get(), ...nextState })
    
    if (!result.success) {
      console.error('Validation failed:', result.error)
      throw new Error('Invalid state update')
    }
    
    set(...args)
  }
  
  return f(validatedSet, get, store)
}

// Usage
const userSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  age: z.number().min(0).max(150),
})

interface UserStore {
  name: string
  email: string
  age: number
  updateUser: (updates: Partial<UserStore>) => void
}

export const useUserStore = create<UserStore>()(
  validator(userSchema)(
    (set) => ({
      name: '',
      email: '',
      age: 0,
      
      updateUser: (updates) => set((state) => ({ ...state, ...updates })),
    })
  )
)
```

### Undo/Redo Middleware

```typescript
// Example 3: Time-travel middleware
interface TemporalState<T> {
  past: T[]
  present: T
  future: T[]
}

type Temporal = <T>(
  initialState: T
) => <Mps extends [StoreMutatorIdentifier, unknown][] = []>(
  f: StateCreator<T, Mps, []>
) => StateCreator<
  TemporalState<T> & {
    undo: () => void
    redo: () => void
    clear: () => void
  },
  Mps,
  []
>

export const temporal: Temporal = (initialState) => (f) => (set, get, store) => {
  const temporalSet: typeof set = (...args) => {
    const currentState = get().present
    
    set((state) => ({
      ...state,
      past: [...state.past, currentState],
      future: [],
    }))
    
    set(...args)
  }
  
  const baseState = f(temporalSet, get, store)
  
  return {
    past: [],
    present: baseState,
    future: [],
    
    undo: () =>
      set((state) => {
        if (state.past.length === 0) return state
        
        const previous = state.past[state.past.length - 1]
        const newPast = state.past.slice(0, -1)
        
        return {
          past: newPast,
          present: previous,
          future: [state.present, ...state.future],
        }
      }),
    
    redo: () =>
      set((state) => {
        if (state.future.length === 0) return state
        
        const next = state.future[0]
        const newFuture = state.future.slice(1)
        
        return {
          past: [...state.past, state.present],
          present: next,
          future: newFuture,
        }
      }),
    
    clear: () =>
      set({
        past: [],
        present: initialState,
        future: [],
      }),
  }
}
```

---

## 2. Advanced TypeScript Patterns

### Slices with TypeScript

```typescript
// Example 1: Type-safe slices
import { StateCreator } from 'zustand'
import { create } from 'zustand'

// Define slice interfaces
interface BearSlice {
  bears: number
  addBear: () => void
  eatFish: () => void
}

interface FishSlice {
  fishes: number
  addFish: () => void
}

interface SharedSlice {
  addBoth: () => void
  getBothCount: () => number
}

// Create bear slice
const createBearSlice: StateCreator<
  BearSlice & FishSlice & SharedSlice,
  [],
  [],
  BearSlice
> = (set, get) => ({
  bears: 0,
  addBear: () => set((state) => ({ bears: state.bears + 1 })),
  eatFish: () => set((state) => ({ fishes: state.fishes - 1 })),
})

// Create fish slice
const createFishSlice: StateCreator<
  BearSlice & FishSlice & SharedSlice,
  [],
  [],
  FishSlice
> = (set) => ({
  fishes: 0,
  addFish: () => set((state) => ({ fishes: state.fishes + 1 })),
})

// Create shared slice
const createSharedSlice: StateCreator<
  BearSlice & FishSlice & SharedSlice,
  [],
  [],
  SharedSlice
> = (set, get) => ({
  addBoth: () => {
    get().addBear()
    get().addFish()
  },
  getBothCount: () => get().bears + get().fishes,
})

// Combine slices
export const useBoundStore = create<BearSlice & FishSlice & SharedSlice>()(
  (...a) => ({
    ...createBearSlice(...a),
    ...createFishSlice(...a),
    ...createSharedSlice(...a),
  })
)
```

**Giải thích:**
- `StateCreator<T, Mps, Mcs, U>`: Type cho slice creator
  - `T`: Full store type (all slices combined)
  - `Mps`: Middleware parameters
  - `Mcs`: Middleware configs
  - `U`: This slice's type
- Each slice can access other slices via `get()`
- Type-safe cross-slice communication

### Extract State Type

```typescript
// Example 2: Type utilities
import { StoreApi, UseBoundStore } from 'zustand'

// Extract state type from store
type ExtractState<S> = S extends {
  getState: () => infer T
}
  ? T
  : never

// Usage
const useStore = create<{ count: number }>(...)
type StoreState = ExtractState<typeof useStore> // { count: number }

// Auto-generate selectors
type WithSelectors<S> = S extends { getState: () => infer T }
  ? S & { use: { [K in keyof T]: () => T[K] } }
  : never

const createSelectors = <S extends UseBoundStore<StoreApi<object>>>(
  _store: S
) => {
  const store = _store as WithSelectors<typeof _store>
  store.use = {} as any

  for (const k of Object.keys(store.getState())) {
    ;(store.use as any)[k] = () => store((s) => s[k as keyof typeof s])
  }

  return store
}

// Usage
const useStoreBase = create<{ count: number; text: string }>(...)
const useStore = createSelectors(useStoreBase)

// Auto-generated selectors
function Component() {
  const count = useStore.use.count() // number
  const text = useStore.use.text()   // string

  return <div>{count} - {text}</div>
}
```

### Middleware Type Mutators

```typescript
// Example 3: Type mutators for middleware
import { create } from 'zustand'
import { devtools, persist, subscribeWithSelector } from 'zustand/middleware'

interface BearState {
  bears: number
  increase: (by: number) => void
}

// Specify middleware types explicitly
export const useBearStore = create<
  BearState,
  [
    ['zustand/devtools', never],
    ['zustand/persist', BearState],
    ['zustand/subscribeWithSelector', never]
  ]
>(
  devtools(
    persist(
      subscribeWithSelector((set) => ({
        bears: 0,
        increase: (by) => set((state) => ({ bears: state.bears + by })),
      })),
      { name: 'bear-storage' }
    ),
    { name: 'BearStore' }
  )
)

// Now subscribe works with proper types
useBearStore.subscribe(
  (state) => state.bears,
  (bears, prevBears) => {
    console.log(`Bears changed from ${prevBears} to ${bears}`)
  }
)
```

---

## 3. Slices Pattern

### Modular Store Architecture

```typescript
// Example 1: Large-scale slices pattern
// src/stores/slices/authSlice.ts
import { StateCreator } from 'zustand'

export interface AuthSlice {
  user: User | null
  token: string | null
  login: (email: string, password: string) => Promise<void>
  logout: () => void
  refreshToken: () => Promise<void>
}

export const createAuthSlice: StateCreator<
  AuthSlice & CartSlice & NotificationSlice,
  [],
  [],
  AuthSlice
> = (set, get) => ({
  user: null,
  token: null,

  login: async (email, password) => {
    try {
      const response = await fetch('/api/auth/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email, password }),
      })

      const { user, token } = await response.json()
      set({ user, token })

      // Cross-slice communication
      get().addNotification('Login successful', 'success')
    } catch (error) {
      get().addNotification('Login failed', 'error')
      throw error
    }
  },

  logout: () => {
    set({ user: null, token: null })
    get().clearCart() // Access cart slice
    get().addNotification('Logged out', 'info')
  },

  refreshToken: async () => {
    const currentToken = get().token
    if (!currentToken) return

    const response = await fetch('/api/auth/refresh', {
      headers: { Authorization: `Bearer ${currentToken}` },
    })

    const { token } = await response.json()
    set({ token })
  },
})

// src/stores/slices/cartSlice.ts
export interface CartSlice {
  items: CartItem[]
  addItem: (item: CartItem) => void
  removeItem: (id: string) => void
  clearCart: () => void
  checkout: () => Promise<void>
}

export const createCartSlice: StateCreator<
  AuthSlice & CartSlice & NotificationSlice,
  [],
  [],
  CartSlice
> = (set, get) => ({
  items: [],

  addItem: (item) =>
    set((state) => ({
      items: [...state.items, item],
    })),

  removeItem: (id) =>
    set((state) => ({
      items: state.items.filter((item) => item.id !== id),
    })),

  clearCart: () => set({ items: [] }),

  checkout: async () => {
    const token = get().token
    if (!token) {
      get().addNotification('Please login first', 'error')
      return
    }

    const items = get().items

    try {
      await fetch('/api/checkout', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${token}`,
        },
        body: JSON.stringify({ items }),
      })

      get().clearCart()
      get().addNotification('Checkout successful', 'success')
    } catch (error) {
      get().addNotification('Checkout failed', 'error')
    }
  },
})

// src/stores/slices/notificationSlice.ts
export interface NotificationSlice {
  notifications: Notification[]
  addNotification: (message: string, type: NotificationType) => void
  removeNotification: (id: string) => void
}

export const createNotificationSlice: StateCreator<
  AuthSlice & CartSlice & NotificationSlice,
  [],
  [],
  NotificationSlice
> = (set) => ({
  notifications: [],

  addNotification: (message, type) =>
    set((state) => ({
      notifications: [
        ...state.notifications,
        {
          id: crypto.randomUUID(),
          message,
          type,
          timestamp: Date.now(),
        },
      ],
    })),

  removeNotification: (id) =>
    set((state) => ({
      notifications: state.notifications.filter((n) => n.id !== id),
    })),
})

// src/stores/useAppStore.ts
import { create } from 'zustand'
import { devtools, persist } from 'zustand/middleware'
import { createAuthSlice, AuthSlice } from './slices/authSlice'
import { createCartSlice, CartSlice } from './slices/cartSlice'
import { createNotificationSlice, NotificationSlice } from './slices/notificationSlice'

type AppStore = AuthSlice & CartSlice & NotificationSlice

export const useAppStore = create<AppStore>()(
  devtools(
    persist(
      (...a) => ({
        ...createAuthSlice(...a),
        ...createCartSlice(...a),
        ...createNotificationSlice(...a),
      }),
      {
        name: 'app-storage',
        partialize: (state) => ({
          // Only persist auth and cart
          user: state.user,
          token: state.token,
          items: state.items,
        }),
      }
    ),
    { name: 'AppStore' }
  )
)
```

**Benefits:**
- Modular code organization
- Easy to test individual slices
- Type-safe cross-slice communication
- Selective persistence
- Clear separation of concerns

---

## 4. Performance Optimization

### Shallow Equality

```typescript
// Example 1: Optimize object selectors
import { shallow } from 'zustand/shallow'

interface UserStore {
  user: {
    id: string
    name: string
    email: string
    preferences: {
      theme: string
      language: string
    }
  }
  updateUser: (updates: Partial<User>) => void
}

function UserProfile() {
  // ❌ BAD: New object every render, always re-renders
  const user = useUserStore((state) => ({
    name: state.user.name,
    email: state.user.email,
  }))

  // ✅ GOOD: Shallow comparison
  const userShallow = useUserStore(
    (state) => ({
      name: state.user.name,
      email: state.user.email,
    }),
    shallow
  )

  return <div>{userShallow.name}</div>
}
```

### subscribeWithSelector Middleware

```typescript
// Example 2: Subscribe to specific state changes
import { create } from 'zustand'
import { subscribeWithSelector } from 'zustand/middleware'

interface CountStore {
  count: number
  text: string
  increment: () => void
  setText: (text: string) => void
}

export const useCountStore = create<CountStore>()(
  subscribeWithSelector((set) => ({
    count: 0,
    text: '',
    increment: () => set((state) => ({ count: state.count + 1 })),
    setText: (text) => set({ text }),
  }))
)

// Subscribe only to count changes
const unsubscribe = useCountStore.subscribe(
  (state) => state.count,
  (count, prevCount) => {
    console.log(`Count changed from ${prevCount} to ${count}`)

    // Side effects based on count
    if (count > 10) {
      console.log('Count exceeded 10!')
    }
  },
  {
    equalityFn: (a, b) => a === b, // Custom equality
    fireImmediately: false, // Don't fire on subscribe
  }
)

// Cleanup
unsubscribe()
```

### Computed Values Optimization

```typescript
// Example 3: Memoized computed values
import { useMemo } from 'react'

interface ProductStore {
  products: Product[]
  filters: {
    category: string
    minPrice: number
    maxPrice: number
    searchQuery: string
  }
}

function ProductList() {
  const products = useProductStore((state) => state.products)
  const filters = useProductStore((state) => state.filters, shallow)

  // Memoize expensive filtering
  const filteredProducts = useMemo(() => {
    return products.filter((product) => {
      const matchesCategory = !filters.category || product.category === filters.category
      const matchesPrice = product.price >= filters.minPrice && product.price <= filters.maxPrice
      const matchesSearch = !filters.searchQuery ||
        product.name.toLowerCase().includes(filters.searchQuery.toLowerCase())

      return matchesCategory && matchesPrice && matchesSearch
    })
  }, [products, filters])

  return (
    <ul>
      {filteredProducts.map((product) => (
        <li key={product.id}>{product.name}</li>
      ))}
    </ul>
  )
}
```

### Selector Splitting

```typescript
// Example 4: Split selectors for better performance
function TodoList() {
  // ❌ BAD: Single selector, re-renders on any todo change
  const { todos, filter } = useTodoStore((state) => ({
    todos: state.todos,
    filter: state.filter,
  }), shallow)

  // ✅ GOOD: Separate selectors
  const todos = useTodoStore((state) => state.todos)
  const filter = useTodoStore((state) => state.filter)

  // Even better: Compute filtered todos in selector
  const filteredTodos = useTodoStore((state) => {
    if (state.filter === 'all') return state.todos
    if (state.filter === 'completed') return state.todos.filter((t) => t.completed)
    return state.todos.filter((t) => !t.completed)
  })

  return (
    <ul>
      {filteredTodos.map((todo) => (
        <TodoItem key={todo.id} todo={todo} />
      ))}
    </ul>
  )
}

// Optimize individual items
const TodoItem = React.memo(({ todo }: { todo: Todo }) => {
  const toggleTodo = useTodoStore((state) => state.toggleTodo)

  return (
    <li>
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={() => toggleTodo(todo.id)}
      />
      {todo.text}
    </li>
  )
})
```

---

## 5. Testing Strategies

### Unit Testing Stores

```typescript
// Example 1: Test store in isolation
// src/stores/__tests__/useCountStore.test.ts
import { renderHook, act } from '@testing-library/react'
import { useCountStore } from '../useCountStore'

describe('useCountStore', () => {
  beforeEach(() => {
    // Reset store before each test
    useCountStore.setState({ count: 0 })
  })

  it('should initialize with count 0', () => {
    const { result } = renderHook(() => useCountStore())
    expect(result.current.count).toBe(0)
  })

  it('should increment count', () => {
    const { result } = renderHook(() => useCountStore())

    act(() => {
      result.current.increment()
    })

    expect(result.current.count).toBe(1)
  })

  it('should decrement count', () => {
    const { result } = renderHook(() => useCountStore())

    act(() => {
      result.current.increment()
      result.current.increment()
      result.current.decrement()
    })

    expect(result.current.count).toBe(1)
  })

  it('should reset count', () => {
    const { result } = renderHook(() => useCountStore())

    act(() => {
      result.current.increment()
      result.current.increment()
      result.current.reset()
    })

    expect(result.current.count).toBe(0)
  })
})
```

### Testing Async Actions

```typescript
// Example 2: Test async operations
// src/stores/__tests__/useProductStore.test.ts
import { renderHook, act, waitFor } from '@testing-library/react'
import { useProductStore } from '../useProductStore'

// Mock fetch
global.fetch = jest.fn()

describe('useProductStore', () => {
  beforeEach(() => {
    jest.clearAllMocks()
    useProductStore.setState({
      products: [],
      loading: false,
      error: null,
    })
  })

  it('should fetch products successfully', async () => {
    const mockProducts = [
      { id: '1', name: 'Product 1', price: 100 },
      { id: '2', name: 'Product 2', price: 200 },
    ]

    ;(global.fetch as jest.Mock).mockResolvedValueOnce({
      ok: true,
      json: async () => mockProducts,
    })

    const { result } = renderHook(() => useProductStore())

    expect(result.current.loading).toBe(false)

    act(() => {
      result.current.fetchProducts()
    })

    expect(result.current.loading).toBe(true)

    await waitFor(() => {
      expect(result.current.loading).toBe(false)
    })

    expect(result.current.products).toEqual(mockProducts)
    expect(result.current.error).toBeNull()
  })

  it('should handle fetch error', async () => {
    ;(global.fetch as jest.Mock).mockResolvedValueOnce({
      ok: false,
    })

    const { result } = renderHook(() => useProductStore())

    act(() => {
      result.current.fetchProducts()
    })

    await waitFor(() => {
      expect(result.current.loading).toBe(false)
    })

    expect(result.current.error).toBe('Failed to fetch products')
    expect(result.current.products).toEqual([])
  })
})
```

### Integration Testing

```typescript
// Example 3: Test with React components
// src/components/__tests__/Counter.test.tsx
import { render, screen, fireEvent } from '@testing-library/react'
import { Counter } from '../Counter'
import { useCountStore } from '@/stores/useCountStore'

describe('Counter', () => {
  beforeEach(() => {
    useCountStore.setState({ count: 0 })
  })

  it('should display initial count', () => {
    render(<Counter />)
    expect(screen.getByText(/count: 0/i)).toBeInTheDocument()
  })

  it('should increment on button click', () => {
    render(<Counter />)

    const incrementButton = screen.getByText('+1')
    fireEvent.click(incrementButton)

    expect(screen.getByText(/count: 1/i)).toBeInTheDocument()
  })

  it('should decrement on button click', () => {
    render(<Counter />)

    const incrementButton = screen.getByText('+1')
    const decrementButton = screen.getByText('-1')

    fireEvent.click(incrementButton)
    fireEvent.click(incrementButton)
    fireEvent.click(decrementButton)

    expect(screen.getByText(/count: 1/i)).toBeInTheDocument()
  })
})
```

### Mocking Stores

```typescript
// Example 4: Mock store for component tests
// src/test-utils/mockStores.ts
import { create } from 'zustand'

export const createMockAuthStore = (initialState = {}) => {
  return create(() => ({
    user: null,
    token: null,
    login: jest.fn(),
    logout: jest.fn(),
    ...initialState,
  }))
}

// Usage in tests
import { createMockAuthStore } from '@/test-utils/mockStores'

jest.mock('@/stores/useAuthStore', () => ({
  useAuthStore: createMockAuthStore({
    user: { id: '1', name: 'Test User' },
    token: 'test-token',
  }),
}))

describe('ProtectedRoute', () => {
  it('should render children when authenticated', () => {
    render(
      <ProtectedRoute>
        <div>Protected Content</div>
      </ProtectedRoute>
    )

    expect(screen.getByText('Protected Content')).toBeInTheDocument()
  })
})
```

---

## 6. State Persistence

### Custom Storage Implementation

```typescript
// Example 1: IndexedDB storage
import { StateStorage } from 'zustand/middleware'

const indexedDBStorage: StateStorage = {
  getItem: async (name: string): Promise<string | null> => {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open('zustand-storage', 1)

      request.onerror = () => reject(request.error)

      request.onsuccess = () => {
        const db = request.result
        const transaction = db.transaction(['state'], 'readonly')
        const store = transaction.objectStore('state')
        const getRequest = store.get(name)

        getRequest.onsuccess = () => resolve(getRequest.result || null)
        getRequest.onerror = () => reject(getRequest.error)
      }

      request.onupgradeneeded = () => {
        const db = request.result
        if (!db.objectStoreNames.contains('state')) {
          db.createObjectStore('state')
        }
      }
    })
  },

  setItem: async (name: string, value: string): Promise<void> => {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open('zustand-storage', 1)

      request.onsuccess = () => {
        const db = request.result
        const transaction = db.transaction(['state'], 'readwrite')
        const store = transaction.objectStore('state')
        const putRequest = store.put(value, name)

        putRequest.onsuccess = () => resolve()
        putRequest.onerror = () => reject(putRequest.error)
      }
    })
  },

  removeItem: async (name: string): Promise<void> => {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open('zustand-storage', 1)

      request.onsuccess = () => {
        const db = request.result
        const transaction = db.transaction(['state'], 'readwrite')
        const store = transaction.objectStore('state')
        const deleteRequest = store.delete(name)

        deleteRequest.onsuccess = () => resolve()
        deleteRequest.onerror = () => reject(deleteRequest.error)
      }
    })
  },
}

// Usage
import { create } from 'zustand'
import { persist, createJSONStorage } from 'zustand/middleware'

export const useStore = create<Store>()(
  persist(
    (set) => ({
      // ... state and actions
    }),
    {
      name: 'app-state',
      storage: createJSONStorage(() => indexedDBStorage),
    }
  )
)
```

### State Migration

```typescript
// Example 2: Version migration
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface UserStoreV1 {
  name: string
  email: string
}

interface UserStoreV2 {
  profile: {
    firstName: string
    lastName: string
    email: string
  }
}

export const useUserStore = create<UserStoreV2>()(
  persist(
    (set) => ({
      profile: {
        firstName: '',
        lastName: '',
        email: '',
      },

      updateProfile: (updates: Partial<UserStoreV2['profile']>) =>
        set((state) => ({
          profile: { ...state.profile, ...updates },
        })),
    }),
    {
      name: 'user-storage',
      version: 2,

      migrate: (persistedState: any, version: number) => {
        // Migrate from v1 to v2
        if (version === 1) {
          const v1State = persistedState as UserStoreV1
          const [firstName, ...lastNameParts] = v1State.name.split(' ')

          return {
            profile: {
              firstName,
              lastName: lastNameParts.join(' '),
              email: v1State.email,
            },
          } as UserStoreV2
        }

        return persistedState as UserStoreV2
      },
    }
  )
)
```

### Partial Persistence

```typescript
// Example 3: Selective persistence
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface AppStore {
  // Persisted
  user: User | null
  settings: Settings

  // Not persisted (runtime only)
  isLoading: boolean
  error: string | null
  tempData: any[]

  // Actions
  setUser: (user: User) => void
  setSettings: (settings: Settings) => void
  setLoading: (loading: boolean) => void
}

export const useAppStore = create<AppStore>()(
  persist(
    (set) => ({
      user: null,
      settings: { theme: 'light', language: 'en' },
      isLoading: false,
      error: null,
      tempData: [],

      setUser: (user) => set({ user }),
      setSettings: (settings) => set({ settings }),
      setLoading: (loading) => set({ isLoading: loading }),
    }),
    {
      name: 'app-storage',

      // Only persist user and settings
      partialize: (state) => ({
        user: state.user,
        settings: state.settings,
      }),
    }
  )
)
```

### URL State Sync

```typescript
// Example 4: Sync state with URL params
import { create } from 'zustand'
import { persist, createJSONStorage, StateStorage } from 'zustand/middleware'

const urlStorage: StateStorage = {
  getItem: (name: string): string | null => {
    const searchParams = new URLSearchParams(window.location.search)
    const value = searchParams.get(name)
    return value
  },

  setItem: (name: string, value: string): void => {
    const searchParams = new URLSearchParams(window.location.search)
    searchParams.set(name, value)

    const newUrl = `${window.location.pathname}?${searchParams.toString()}`
    window.history.replaceState({}, '', newUrl)
  },

  removeItem: (name: string): void => {
    const searchParams = new URLSearchParams(window.location.search)
    searchParams.delete(name)

    const newUrl = searchParams.toString()
      ? `${window.location.pathname}?${searchParams.toString()}`
      : window.location.pathname

    window.history.replaceState({}, '', newUrl)
  },
}

interface FilterStore {
  category: string
  minPrice: number
  maxPrice: number
  setCategory: (category: string) => void
  setPriceRange: (min: number, max: number) => void
}

export const useFilterStore = create<FilterStore>()(
  persist(
    (set) => ({
      category: 'all',
      minPrice: 0,
      maxPrice: 1000,

      setCategory: (category) => set({ category }),
      setPriceRange: (min, max) => set({ minPrice: min, maxPrice: max }),
    }),
    {
      name: 'filters',
      storage: createJSONStorage(() => urlStorage),
    }
  )
)
```

---

## 7. Integration Patterns

### React Query Integration

```typescript
// Example 1: Combine Zustand with React Query
import { create } from 'zustand'
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'

interface TodoStore {
  selectedTodoId: string | null
  setSelectedTodoId: (id: string | null) => void
}

export const useTodoStore = create<TodoStore>((set) => ({
  selectedTodoId: null,
  setSelectedTodoId: (id) => set({ selectedTodoId: id }),
}))

// Use React Query for server state
function TodoList() {
  const selectedTodoId = useTodoStore((state) => state.selectedTodoId)
  const setSelectedTodoId = useTodoStore((state) => state.setSelectedTodoId)

  // Server state with React Query
  const { data: todos, isLoading } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  })

  const queryClient = useQueryClient()

  const addTodoMutation = useMutation({
    mutationFn: addTodo,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] })
    },
  })

  if (isLoading) return <div>Loading...</div>

  return (
    <div>
      <ul>
        {todos?.map((todo) => (
          <li
            key={todo.id}
            onClick={() => setSelectedTodoId(todo.id)}
            style={{
              fontWeight: selectedTodoId === todo.id ? 'bold' : 'normal',
            }}
          >
            {todo.text}
          </li>
        ))}
      </ul>
    </div>
  )
}
```

**Pattern:**
- Zustand: Client state (UI state, selections, filters)
- React Query: Server state (API data, caching, mutations)

### TanStack Router Integration

```typescript
// Example 2: Zustand with TanStack Router
import { create } from 'zustand'
import { createRootRoute, createRoute, createRouter } from '@tanstack/react-router'

interface NavigationStore {
  breadcrumbs: string[]
  addBreadcrumb: (label: string) => void
  clearBreadcrumbs: () => void
}

export const useNavigationStore = create<NavigationStore>((set) => ({
  breadcrumbs: [],
  addBreadcrumb: (label) =>
    set((state) => ({
      breadcrumbs: [...state.breadcrumbs, label],
    })),
  clearBreadcrumbs: () => set({ breadcrumbs: [] }),
}))

// Use in route loader
const productRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/products/$productId',

  beforeLoad: ({ params }) => {
    const store = useNavigationStore.getState()
    store.clearBreadcrumbs()
    store.addBreadcrumb('Products')
    store.addBreadcrumb(params.productId)
  },

  component: ProductPage,
})
```

### Form State Management

```typescript
// Example 3: Complex form with Zustand
import { create } from 'zustand'
import { immer } from 'zustand/middleware/immer'
import { z } from 'zod'

const formSchema = z.object({
  personalInfo: z.object({
    firstName: z.string().min(1),
    lastName: z.string().min(1),
    email: z.string().email(),
  }),
  address: z.object({
    street: z.string().min(1),
    city: z.string().min(1),
    zipCode: z.string().regex(/^\d{5}$/),
  }),
  preferences: z.object({
    newsletter: z.boolean(),
    notifications: z.boolean(),
  }),
})

type FormData = z.infer<typeof formSchema>

interface FormStore {
  data: FormData
  errors: Partial<Record<keyof FormData, string>>
  touched: Partial<Record<keyof FormData, boolean>>

  updateField: <K extends keyof FormData>(
    section: K,
    field: keyof FormData[K],
    value: any
  ) => void

  validateField: <K extends keyof FormData>(
    section: K,
    field: keyof FormData[K]
  ) => void

  validateForm: () => boolean
  resetForm: () => void
  submitForm: () => Promise<void>
}

export const useFormStore = create<FormStore>()(
  immer((set, get) => ({
    data: {
      personalInfo: { firstName: '', lastName: '', email: '' },
      address: { street: '', city: '', zipCode: '' },
      preferences: { newsletter: false, notifications: false },
    },
    errors: {},
    touched: {},

    updateField: (section, field, value) =>
      set((state) => {
        state.data[section][field] = value
        state.touched[section] = true
      }),

    validateField: (section, field) =>
      set((state) => {
        const result = formSchema.shape[section].safeParse(get().data[section])

        if (!result.success) {
          const error = result.error.errors.find((e) => e.path[0] === field)
          if (error) {
            state.errors[section] = error.message
          }
        } else {
          delete state.errors[section]
        }
      }),

    validateForm: () => {
      const result = formSchema.safeParse(get().data)

      if (!result.success) {
        set((state) => {
          result.error.errors.forEach((error) => {
            const [section] = error.path as [keyof FormData]
            state.errors[section] = error.message
          })
        })
        return false
      }

      return true
    },

    resetForm: () =>
      set({
        data: {
          personalInfo: { firstName: '', lastName: '', email: '' },
          address: { street: '', city: '', zipCode: '' },
          preferences: { newsletter: false, notifications: false },
        },
        errors: {},
        touched: {},
      }),

    submitForm: async () => {
      if (!get().validateForm()) {
        throw new Error('Form validation failed')
      }

      const response = await fetch('/api/submit', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(get().data),
      })

      if (!response.ok) {
        throw new Error('Form submission failed')
      }

      get().resetForm()
    },
  }))
)
```

---

## Best Practices - Senior Level

### 1. Middleware Composition

```typescript
// ✅ GOOD: Logical middleware order
create<Store>()(
  devtools(
    persist(
      immer((set) => ({
        // ... store
      })),
      { name: 'storage' }
    ),
    { name: 'Store' }
  )
)

// ❌ BAD: Illogical order
create<Store>()(
  persist(
    devtools(
      immer((set) => ({
        // ... store
      })),
      { name: 'Store' }
    ),
    { name: 'storage' }
  )
)
```

### 2. Type Safety

```typescript
// ✅ GOOD: Full type safety
interface Store {
  count: number
  increment: () => void
}

const useStore = create<Store>()(
  devtools((set) => ({
    count: 0,
    increment: () => set((state) => ({ count: state.count + 1 })),
  }))
)

// ❌ BAD: No types
const useStore = create(
  devtools((set) => ({
    count: 0,
    increment: () => set((state) => ({ count: state.count + 1 })),
  }))
)
```

### 3. Slices Organization

```typescript
// ✅ GOOD: Modular slices
const useStore = create<AuthSlice & CartSlice>()((...a) => ({
  ...createAuthSlice(...a),
  ...createCartSlice(...a),
}))

// ❌ BAD: Monolithic store
const useStore = create((set) => ({
  // 1000+ lines of state and actions
}))
```

---

## Tài Liệu Tham Khảo

### Official Documentation
- [Zustand Middleware](https://github.com/pmndrs/zustand/blob/main/docs/guides/typescript.md)
- [Testing Guide](https://github.com/pmndrs/zustand/blob/main/docs/guides/testing.md)
- [Slices Pattern](https://github.com/pmndrs/zustand/blob/main/docs/guides/slices-pattern.md)

### Next Level
- Xem `Principal-Zustand-Patterns.md` cho Enterprise patterns

---

**Tài liệu này bao gồm Senior level patterns. Để học các patterns enterprise, hãy tiếp tục với Principal-Zustand-Patterns.md.**

