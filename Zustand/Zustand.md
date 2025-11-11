# Zustand - State Management cho React

## Mục Lục

- [Junior Level - Cơ Bản](#junior-level---cơ-bản)
  - [1. Giới Thiệu về Zustand](#1-giới-thiệu-về-zustand)
  - [2. Installation và Setup](#2-installation-và-setup)
  - [3. Creating Stores](#3-creating-stores)
  - [4. Reading State](#4-reading-state)
  - [5. Updating State](#5-updating-state)
  - [6. Basic Selectors](#6-basic-selectors)

- [Middle Level - Trung Cấp](#middle-level---trung-cấp)
  - [1. Advanced Selectors](#1-advanced-selectors)
  - [2. Computed Values](#2-computed-values)
  - [3. Async Actions](#3-async-actions)
  - [4. Multiple Stores](#4-multiple-stores)
  - [5. Middleware](#5-middleware)
  - [6. TypeScript Best Practices](#6-typescript-best-practices)

- [Senior Level](#senior-level) - Xem file `Advanced-Zustand-Patterns.md`
- [Principal Level](#principal-level) - Xem file `Principal-Zustand-Patterns.md`

---

## Junior Level - Cơ Bản

### 1. Giới Thiệu về Zustand

#### Zustand là gì?

Zustand là một state management library nhỏ gọn, nhanh và dễ sử dụng cho React với những đặc điểm nổi bật:

- **Minimalistic**: API đơn giản, không cần boilerplate
- **No Provider**: Không cần wrap app với Provider
- **TypeScript First**: Type-safe hoàn toàn
- **Lightweight**: Chỉ ~1KB (gzipped)
- **DevTools Support**: Tích hợp Redux DevTools
- **Middleware**: Persist, devtools, immer, và nhiều hơn

#### So Sánh với Redux và Context API

| Feature | Zustand | Redux Toolkit | Context API |
|---------|---------|---------------|-------------|
| Bundle Size | ~1KB | ~10KB | Built-in |
| Boilerplate | Minimal | Medium | Low |
| Learning Curve | Easy | Medium | Easy |
| TypeScript | Excellent | Good | Good |
| DevTools | ✅ | ✅ | ❌ |
| Middleware | ✅ | ✅ | ❌ |
| Performance | Excellent | Good | Poor (re-renders) |
| Async Support | Built-in | Built-in | Manual |

**Khi nào dùng Zustand:**
- Dự án nhỏ đến trung bình
- Cần state management đơn giản
- Muốn tránh boilerplate của Redux
- Cần performance tốt
- TypeScript project

**Khi nào dùng Redux:**
- Dự án lớn, phức tạp
- Team đã quen với Redux
- Cần ecosystem lớn (middleware, tools)
- Cần time-travel debugging mạnh mẽ

**Khi nào dùng Context API:**
- State đơn giản, ít thay đổi
- Không cần performance cao
- Không muốn thêm dependencies

### 2. Installation và Setup

#### Cài Đặt

```bash
# NPM
npm install zustand

# Yarn
yarn add zustand

# PNPM
pnpm add zustand

# Optional: DevTools typing
npm install -D @redux-devtools/extension
```

#### Setup Cơ Bản

```typescript
// Example 1: Basic store setup
// src/stores/useCountStore.ts
import { create } from 'zustand'

// Define store interface
interface CountStore {
  count: number
  increment: () => void
  decrement: () => void
  reset: () => void
}

// Create store
export const useCountStore = create<CountStore>((set) => ({
  // Initial state
  count: 0,
  
  // Actions
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
}))
```

**Giải thích:**
- `create()`: Function tạo store
- `set()`: Function để update state
- `(state) => ({ ... })`: Callback nhận current state, return new state
- `set({ count: 0 })`: Direct state update (không cần previous state)

#### Sử Dụng Store trong Component

```typescript
// Example 2: Using store in component
// src/components/Counter.tsx
import { useCountStore } from '@/stores/useCountStore'

function Counter() {
  // Get state and actions from store
  const count = useCountStore((state) => state.count)
  const increment = useCountStore((state) => state.increment)
  const decrement = useCountStore((state) => state.decrement)
  const reset = useCountStore((state) => state.reset)
  
  return (
    <div>
      <h1>Count: {count}</h1>
      <button onClick={increment}>+1</button>
      <button onClick={decrement}>-1</button>
      <button onClick={reset}>Reset</button>
    </div>
  )
}
```

**Giải thích:**
- `useCountStore((state) => state.count)`: Selector function để lấy state
- Component chỉ re-render khi selected state thay đổi
- Không cần Provider wrapper

### 3. Creating Stores

#### Simple Store

```typescript
// Example 1: Todo store
import { create } from 'zustand'

interface Todo {
  id: string
  text: string
  completed: boolean
}

interface TodoStore {
  todos: Todo[]
  addTodo: (text: string) => void
  toggleTodo: (id: string) => void
  removeTodo: (id: string) => void
}

export const useTodoStore = create<TodoStore>((set) => ({
  todos: [],
  
  addTodo: (text) =>
    set((state) => ({
      todos: [
        ...state.todos,
        {
          id: crypto.randomUUID(),
          text,
          completed: false,
        },
      ],
    })),
  
  toggleTodo: (id) =>
    set((state) => ({
      todos: state.todos.map((todo) =>
        todo.id === id ? { ...todo, completed: !todo.completed } : todo
      ),
    })),
  
  removeTodo: (id) =>
    set((state) => ({
      todos: state.todos.filter((todo) => todo.id !== id),
    })),
}))
```

**Best Practices:**
- Immutable updates: Luôn return new objects/arrays
- Descriptive action names: `addTodo`, `toggleTodo` thay vì `add`, `toggle`
- Type safety: Sử dụng TypeScript interfaces

#### Store với Nested State

```typescript
// Example 2: User profile store
interface UserProfile {
  id: string
  name: string
  email: string
  preferences: {
    theme: 'light' | 'dark'
    language: 'en' | 'vi'
    notifications: boolean
  }
}

interface UserStore {
  user: UserProfile | null
  setUser: (user: UserProfile) => void
  updatePreferences: (preferences: Partial<UserProfile['preferences']>) => void
  clearUser: () => void
}

export const useUserStore = create<UserStore>((set) => ({
  user: null,

  setUser: (user) => set({ user }),

  updatePreferences: (preferences) =>
    set((state) => ({
      user: state.user
        ? {
            ...state.user,
            preferences: {
              ...state.user.preferences,
              ...preferences,
            },
          }
        : null,
    })),

  clearUser: () => set({ user: null }),
}))
```

### 4. Reading State

#### Basic Selectors

```typescript
// Example 1: Select single value
function UserName() {
  // Only re-renders when user.name changes
  const userName = useUserStore((state) => state.user?.name)

  return <div>Hello, {userName}!</div>
}
```

```typescript
// Example 2: Select multiple values
function UserProfile() {
  // Re-renders when user.name OR user.email changes
  const { name, email } = useUserStore((state) => ({
    name: state.user?.name,
    email: state.user?.email,
  }))

  return (
    <div>
      <p>Name: {name}</p>
      <p>Email: {email}</p>
    </div>
  )
}
```

```typescript
// Example 3: Select entire store (not recommended)
function TodoList() {
  // Re-renders on ANY state change - BAD!
  const store = useTodoStore()

  return (
    <ul>
      {store.todos.map((todo) => (
        <li key={todo.id}>{todo.text}</li>
      ))}
    </ul>
  )
}

// BETTER: Select only what you need
function TodoListOptimized() {
  const todos = useTodoStore((state) => state.todos)

  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>{todo.text}</li>
      ))}
    </ul>
  )
}
```

### 5. Updating State

#### Direct Updates

```typescript
// Example 1: Simple state update
const useCountStore = create<CountStore>((set) => ({
  count: 0,

  // Direct update - không cần previous state
  reset: () => set({ count: 0 }),

  // Update với previous state
  increment: () => set((state) => ({ count: state.count + 1 })),
}))
```

#### Partial Updates

```typescript
// Example 2: Partial state updates
interface FormStore {
  formData: {
    name: string
    email: string
    age: number
  }
  updateField: (field: keyof FormStore['formData'], value: any) => void
  resetForm: () => void
}

const useFormStore = create<FormStore>((set) => ({
  formData: {
    name: '',
    email: '',
    age: 0,
  },

  updateField: (field, value) =>
    set((state) => ({
      formData: {
        ...state.formData,
        [field]: value,
      },
    })),

  resetForm: () =>
    set({
      formData: {
        name: '',
        email: '',
        age: 0,
      },
    }),
}))
```

#### Replace vs Merge

```typescript
// Example 3: Replace vs Merge behavior
const useSettingsStore = create<SettingsStore>((set) => ({
  settings: {
    theme: 'light',
    language: 'en',
  },

  // MERGE: Combines with existing state
  updateTheme: (theme) =>
    set((state) => ({
      settings: {
        ...state.settings,
        theme, // Only updates theme, keeps language
      },
    })),

  // REPLACE: Replaces entire settings object
  replaceSettings: (settings) =>
    set({ settings }), // Replaces entire settings
}))
```

### 6. Basic Selectors

#### Selector Functions

```typescript
// Example 1: Computed selectors
const useTodoStore = create<TodoStore>((set, get) => ({
  todos: [],

  // ... actions ...

  // Getter methods (not reactive)
  getCompletedCount: () => {
    return get().todos.filter((todo) => todo.completed).length
  },

  getTotalCount: () => {
    return get().todos.length
  },
}))

// Usage in component
function TodoStats() {
  const todos = useTodoStore((state) => state.todos)
  const completedCount = useTodoStore((state) =>
    state.todos.filter((t) => t.completed).length
  )

  return (
    <div>
      <p>Total: {todos.length}</p>
      <p>Completed: {completedCount}</p>
    </div>
  )
}
```

#### Equality Functions

```typescript
// Example 2: Custom equality check
import { shallow } from 'zustand/shallow'

function UserInfo() {
  // Default: Reference equality (===)
  // Re-renders even if values are same but object is new
  const user = useUserStore((state) => ({
    name: state.user?.name,
    email: state.user?.email,
  }))

  // Better: Shallow equality
  // Only re-renders if name or email actually changes
  const userShallow = useUserStore(
    (state) => ({
      name: state.user?.name,
      email: state.user?.email,
    }),
    shallow
  )

  return <div>{userShallow.name}</div>
}
```

---

## Middle Level - Trung Cấp

### 1. Advanced Selectors

#### Memoized Selectors

```typescript
// Example 1: Expensive computations
import { useMemo } from 'react'

function TodoAnalytics() {
  const todos = useTodoStore((state) => state.todos)

  // Memoize expensive calculation
  const analytics = useMemo(() => {
    const completed = todos.filter((t) => t.completed).length
    const pending = todos.length - completed
    const completionRate = todos.length > 0
      ? (completed / todos.length) * 100
      : 0

    return { completed, pending, completionRate }
  }, [todos])

  return (
    <div>
      <p>Completed: {analytics.completed}</p>
      <p>Pending: {analytics.pending}</p>
      <p>Rate: {analytics.completionRate.toFixed(2)}%</p>
    </div>
  )
}
```

#### Selector Composition

```typescript
// Example 2: Reusable selectors
// src/stores/selectors/todoSelectors.ts
import { TodoStore } from '../useTodoStore'

export const selectTodos = (state: TodoStore) => state.todos

export const selectCompletedTodos = (state: TodoStore) =>
  state.todos.filter((todo) => todo.completed)

export const selectPendingTodos = (state: TodoStore) =>
  state.todos.filter((todo) => !todo.completed)

export const selectTodoById = (id: string) => (state: TodoStore) =>
  state.todos.find((todo) => todo.id === id)

// Usage
function TodoList() {
  const completedTodos = useTodoStore(selectCompletedTodos)
  const pendingTodos = useTodoStore(selectPendingTodos)

  return (
    <div>
      <h2>Pending ({pendingTodos.length})</h2>
      {/* ... */}

      <h2>Completed ({completedTodos.length})</h2>
      {/* ... */}
    </div>
  )
}
```

### 2. Computed Values

#### Using `combine` Middleware

```typescript
// Example 1: Separate state and actions
import { create } from 'zustand'
import { combine } from 'zustand/middleware'

export const useCartStore = create(
  combine(
    // Initial state
    {
      items: [] as CartItem[],
      discount: 0,
    },
    // Actions
    (set, get) => ({
      addItem: (item: CartItem) =>
        set((state) => ({
          items: [...state.items, item],
        })),

      removeItem: (id: string) =>
        set((state) => ({
          items: state.items.filter((item) => item.id !== id),
        })),

      setDiscount: (discount: number) => set({ discount }),

      // Computed values as getters
      getSubtotal: () => {
        return get().items.reduce((sum, item) => sum + item.price * item.quantity, 0)
      },

      getTotal: () => {
        const subtotal = get().getSubtotal()
        const discount = get().discount
        return subtotal - (subtotal * discount) / 100
      },
    })
  )
)

// Usage
function CartSummary() {
  const items = useCartStore((state) => state.items)
  const discount = useCartStore((state) => state.discount)
  const getSubtotal = useCartStore((state) => state.getSubtotal)
  const getTotal = useCartStore((state) => state.getTotal)

  return (
    <div>
      <p>Items: {items.length}</p>
      <p>Subtotal: ${getSubtotal().toFixed(2)}</p>
      <p>Discount: {discount}%</p>
      <p>Total: ${getTotal().toFixed(2)}</p>
    </div>
  )
}
```

**Giải thích:**
- `combine`: Middleware tách biệt state và actions
- Type inference tự động, không cần define interface
- Getters (`getSubtotal`, `getTotal`) không reactive, gọi mỗi khi cần

#### Derived State in Selectors

```typescript
// Example 2: Compute in selector
function CartSummary() {
  // Compute derived values in selector
  const summary = useCartStore((state) => {
    const subtotal = state.items.reduce(
      (sum, item) => sum + item.price * item.quantity,
      0
    )
    const total = subtotal - (subtotal * state.discount) / 100

    return {
      itemCount: state.items.length,
      subtotal,
      discount: state.discount,
      total,
    }
  })

  return (
    <div>
      <p>Items: {summary.itemCount}</p>
      <p>Subtotal: ${summary.subtotal.toFixed(2)}</p>
      <p>Discount: {summary.discount}%</p>
      <p>Total: ${summary.total.toFixed(2)}</p>
    </div>
  )
}
```

### 3. Async Actions

#### Basic Async Operations

```typescript
// Example 1: Fetch data
interface Product {
  id: string
  name: string
  price: number
}

interface ProductStore {
  products: Product[]
  loading: boolean
  error: string | null
  fetchProducts: () => Promise<void>
}

export const useProductStore = create<ProductStore>((set) => ({
  products: [],
  loading: false,
  error: null,

  fetchProducts: async () => {
    set({ loading: true, error: null })

    try {
      const response = await fetch('/api/products')

      if (!response.ok) {
        throw new Error('Failed to fetch products')
      }

      const products = await response.json()
      set({ products, loading: false })
    } catch (error) {
      set({
        error: error instanceof Error ? error.message : 'Unknown error',
        loading: false,
      })
    }
  },
}))

// Usage
function ProductList() {
  const { products, loading, error, fetchProducts } = useProductStore()

  useEffect(() => {
    fetchProducts()
  }, [fetchProducts])

  if (loading) return <div>Loading...</div>
  if (error) return <div>Error: {error}</div>

  return (
    <ul>
      {products.map((product) => (
        <li key={product.id}>{product.name}</li>
      ))}
    </ul>
  )
}
```

#### Optimistic Updates

```typescript
// Example 2: Optimistic UI updates
interface TodoStore {
  todos: Todo[]
  addTodoOptimistic: (text: string) => Promise<void>
  updateTodoOptimistic: (id: string, updates: Partial<Todo>) => Promise<void>
}

export const useTodoStore = create<TodoStore>((set, get) => ({
  todos: [],

  addTodoOptimistic: async (text) => {
    // Generate temporary ID
    const tempId = `temp-${Date.now()}`
    const optimisticTodo: Todo = {
      id: tempId,
      text,
      completed: false,
    }

    // Immediately add to UI
    set((state) => ({
      todos: [...state.todos, optimisticTodo],
    }))

    try {
      // Send to server
      const response = await fetch('/api/todos', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ text }),
      })

      const newTodo = await response.json()

      // Replace temp todo with real one
      set((state) => ({
        todos: state.todos.map((todo) =>
          todo.id === tempId ? newTodo : todo
        ),
      }))
    } catch (error) {
      // Rollback on error
      set((state) => ({
        todos: state.todos.filter((todo) => todo.id !== tempId),
      }))

      throw error
    }
  },

  updateTodoOptimistic: async (id, updates) => {
    // Save previous state for rollback
    const previousTodos = get().todos

    // Optimistic update
    set((state) => ({
      todos: state.todos.map((todo) =>
        todo.id === id ? { ...todo, ...updates } : todo
      ),
    }))

    try {
      await fetch(`/api/todos/${id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(updates),
      })
    } catch (error) {
      // Rollback on error
      set({ todos: previousTodos })
      throw error
    }
  },
}))
```

### 4. Multiple Stores

#### Separate Concerns

```typescript
// Example 1: Multiple independent stores
// src/stores/useAuthStore.ts
interface AuthStore {
  user: User | null
  token: string | null
  login: (email: string, password: string) => Promise<void>
  logout: () => void
}

export const useAuthStore = create<AuthStore>((set) => ({
  user: null,
  token: null,

  login: async (email, password) => {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
    })

    const { user, token } = await response.json()
    set({ user, token })
  },

  logout: () => set({ user: null, token: null }),
}))

// src/stores/useNotificationStore.ts
interface NotificationStore {
  notifications: Notification[]
  addNotification: (message: string, type: 'success' | 'error') => void
  removeNotification: (id: string) => void
}

export const useNotificationStore = create<NotificationStore>((set) => ({
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
}))
```

#### Cross-Store Communication

```typescript
// Example 2: Stores communicating with each other
// src/stores/useCartStore.ts
export const useCartStore = create<CartStore>((set, get) => ({
  items: [],

  checkout: async () => {
    const items = get().items
    const token = useAuthStore.getState().token

    if (!token) {
      useNotificationStore.getState().addNotification(
        'Please login to checkout',
        'error'
      )
      return
    }

    try {
      await fetch('/api/checkout', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`,
        },
        body: JSON.stringify({ items }),
      })

      set({ items: [] })
      useNotificationStore.getState().addNotification(
        'Checkout successful!',
        'success'
      )
    } catch (error) {
      useNotificationStore.getState().addNotification(
        'Checkout failed',
        'error'
      )
    }
  },
}))
```

### 5. Middleware

#### Persist Middleware

```typescript
// Example 1: Persist to localStorage
import { create } from 'zustand'
import { persist, createJSONStorage } from 'zustand/middleware'

interface SettingsStore {
  theme: 'light' | 'dark'
  language: 'en' | 'vi'
  setTheme: (theme: 'light' | 'dark') => void
  setLanguage: (language: 'en' | 'vi') => void
}

export const useSettingsStore = create<SettingsStore>()(
  persist(
    (set) => ({
      theme: 'light',
      language: 'en',

      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
    }),
    {
      name: 'app-settings', // localStorage key
      storage: createJSONStorage(() => localStorage), // default
    }
  )
)

// Persist to sessionStorage
export const useSessionStore = create<SessionStore>()(
  persist(
    (set) => ({
      // ... state and actions
    }),
    {
      name: 'session-data',
      storage: createJSONStorage(() => sessionStorage),
    }
  )
)
```

**Giải thích:**
- `persist()`: Middleware tự động save/load state
- `name`: Key trong storage
- `storage`: localStorage (default), sessionStorage, hoặc custom
- State được save mỗi khi thay đổi
- Auto-load khi app khởi động

#### DevTools Middleware

```typescript
// Example 2: Redux DevTools integration
import { create } from 'zustand'
import { devtools } from 'zustand/middleware'
import type {} from '@redux-devtools/extension' // Required for typing

interface CountStore {
  count: number
  increment: () => void
  decrement: () => void
}

export const useCountStore = create<CountStore>()(
  devtools(
    (set) => ({
      count: 0,

      increment: () =>
        set(
          (state) => ({ count: state.count + 1 }),
          undefined,
          'counter/increment' // Action name in DevTools
        ),

      decrement: () =>
        set(
          (state) => ({ count: state.count - 1 }),
          undefined,
          'counter/decrement'
        ),
    }),
    {
      name: 'CountStore', // Store name in DevTools
      enabled: process.env.NODE_ENV !== 'production', // Disable in production
    }
  )
)
```

#### Combining Multiple Middleware

```typescript
// Example 3: Persist + DevTools
import { create } from 'zustand'
import { devtools, persist } from 'zustand/middleware'

export const useUserStore = create<UserStore>()(
  devtools(
    persist(
      (set) => ({
        user: null,

        setUser: (user) =>
          set({ user }, undefined, 'user/setUser'),

        clearUser: () =>
          set({ user: null }, undefined, 'user/clearUser'),
      }),
      {
        name: 'user-storage',
      }
    ),
    {
      name: 'UserStore',
    }
  )
)
```

**Middleware Order:**
- Outer middleware wraps inner middleware
- `devtools(persist(...))`: DevTools sees persisted state
- `persist(devtools(...))`: Persist sees DevTools actions

#### Immer Middleware

```typescript
// Example 4: Immer for mutable-style updates
import { create } from 'zustand'
import { immer } from 'zustand/middleware/immer'

interface TodoStore {
  todos: Todo[]
  addTodo: (text: string) => void
  toggleTodo: (id: string) => void
}

export const useTodoStore = create<TodoStore>()(
  immer((set) => ({
    todos: [],

    addTodo: (text) =>
      set((state) => {
        // Mutable-style update (Immer handles immutability)
        state.todos.push({
          id: crypto.randomUUID(),
          text,
          completed: false,
        })
      }),

    toggleTodo: (id) =>
      set((state) => {
        const todo = state.todos.find((t) => t.id === id)
        if (todo) {
          todo.completed = !todo.completed
        }
      }),
  }))
)
```

**Benefits of Immer:**
- Write mutable-style code
- Immer handles immutability automatically
- Cleaner code for nested updates
- No need for spread operators

### 6. TypeScript Best Practices

#### Strict Typing

```typescript
// Example 1: Proper type definitions
import { create } from 'zustand'

// Define state interface
interface BearState {
  bears: number
  fish: number
}

// Define actions interface
interface BearActions {
  increasePopulation: () => void
  removeAllBears: () => void
  eatFish: () => void
}

// Combine into store type
type BearStore = BearState & BearActions

// Create store with full typing
export const useBearStore = create<BearStore>((set) => ({
  bears: 0,
  fish: 0,

  increasePopulation: () => set((state) => ({ bears: state.bears + 1 })),
  removeAllBears: () => set({ bears: 0 }),
  eatFish: () => set((state) => ({ fish: state.fish - 1 })),
}))
```

#### Type Inference with `combine`

```typescript
// Example 2: Auto type inference
import { create } from 'zustand'
import { combine } from 'zustand/middleware'

// No need to define interface - types are inferred!
export const useBearStore = create(
  combine(
    // State
    { bears: 0, fish: 0 },
    // Actions
    (set) => ({
      increasePopulation: () => set((state) => ({ bears: state.bears + 1 })),
      removeAllBears: () => set({ bears: 0 }),
      eatFish: () => set((state) => ({ fish: state.fish - 1 })),
    })
  )
)

// Full type inference in components
function BearCounter() {
  const bears = useBearStore((state) => state.bears) // number
  const increasePopulation = useBearStore((state) => state.increasePopulation) // () => void

  return <button onClick={increasePopulation}>Bears: {bears}</button>
}
```

#### Middleware Type Safety

```typescript
// Example 3: Typed middleware
import { create } from 'zustand'
import { devtools, persist } from 'zustand/middleware'
import type {} from '@redux-devtools/extension'

interface BearState {
  bears: number
  increase: (by: number) => void
}

// Proper typing with middleware
export const useBearStore = create<BearState>()(
  devtools(
    persist(
      (set) => ({
        bears: 0,
        increase: (by) => set((state) => ({ bears: state.bears + by })),
      }),
      { name: 'bear-storage' }
    ),
    { name: 'BearStore' }
  )
)
```

**Important:**
- Use `create<Type>()()` (double call) for middleware
- First `()` for type, second `()` for implementation
- Required for proper type inference with middleware

---

## Best Practices - Junior & Middle Level

### 1. Store Organization

```typescript
// ✅ GOOD: Small, focused stores
const useAuthStore = create<AuthStore>(...)
const useCartStore = create<CartStore>(...)
const useProductStore = create<ProductStore>(...)

// ❌ BAD: One giant store
const useAppStore = create<AppStore>(...)
```

### 2. Selector Performance

```typescript
// ✅ GOOD: Select only what you need
const count = useCountStore((state) => state.count)

// ❌ BAD: Select entire store
const store = useCountStore()
```

### 3. Immutable Updates

```typescript
// ✅ GOOD: Immutable update
set((state) => ({
  todos: [...state.todos, newTodo]
}))

// ❌ BAD: Mutating state directly
set((state) => {
  state.todos.push(newTodo) // Don't do this without Immer!
  return state
})
```

### 4. Async Actions

```typescript
// ✅ GOOD: Handle loading and errors
fetchData: async () => {
  set({ loading: true, error: null })
  try {
    const data = await fetch(...)
    set({ data, loading: false })
  } catch (error) {
    set({ error, loading: false })
  }
}

// ❌ BAD: No error handling
fetchData: async () => {
  const data = await fetch(...)
  set({ data })
}
```

### 5. TypeScript

```typescript
// ✅ GOOD: Proper typing
interface Store {
  count: number
  increment: () => void
}
const useStore = create<Store>(...)

// ❌ BAD: No types
const useStore = create((set) => ...)
```

---

## Common Pitfalls

### ❌ Pitfall 1: Selecting entire store

```typescript
// BAD: Re-renders on any state change
const store = useStore()

// GOOD: Only re-renders when count changes
const count = useStore((state) => state.count)
```

### ❌ Pitfall 2: Creating new objects in selectors

```typescript
// BAD: New object every time, always re-renders
const user = useStore((state) => ({
  name: state.user.name,
  email: state.user.email,
}))

// GOOD: Use shallow equality
import { shallow } from 'zustand/shallow'
const user = useStore(
  (state) => ({
    name: state.user.name,
    email: state.user.email,
  }),
  shallow
)
```

### ❌ Pitfall 3: Mutating state without Immer

```typescript
// BAD: Direct mutation
set((state) => {
  state.count++ // Don't do this!
  return state
})

// GOOD: Immutable update
set((state) => ({ count: state.count + 1 }))

// OR: Use Immer middleware
immer((set) => ({
  increment: () => set((state) => {
    state.count++ // OK with Immer
  })
}))
```

---

## Tài Liệu Tham Khảo

### Official Documentation
- [Zustand Docs](https://zustand-demo.pmnd.rs/)
- [Zustand GitHub](https://github.com/pmndrs/zustand)
- [API Reference](https://github.com/pmndrs/zustand/blob/main/docs/apis/create.md)

### Advanced Topics
- Xem `Advanced-Zustand-Patterns.md` cho Senior level patterns
- Xem `Principal-Zustand-Patterns.md` cho Enterprise patterns

---

**Tài liệu này bao gồm Junior và Middle levels. Để học các patterns nâng cao hơn, hãy tiếp tục với các files khác trong thư mục này.**

