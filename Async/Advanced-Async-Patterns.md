# Advanced Async Patterns - Senior Level

## Mục Lục

- [1. Advanced Async Patterns](#1-advanced-async-patterns)
  - [Retry Logic với Exponential Backoff](#retry-logic-với-exponential-backoff)
  - [Circuit Breaker Pattern](#circuit-breaker-pattern)
  - [Bulkhead Pattern](#bulkhead-pattern)
  
- [2. Custom Promise Implementations](#2-custom-promise-implementations)
  - [Deferred Promise](#deferred-promise)
  - [Promise Pool](#promise-pool)
  - [Lazy Promise](#lazy-promise)

- [3. Async Iterators và Generators](#3-async-iterators-và-generators)
  - [Async Generators](#async-generators)
  - [Async Iteration](#async-iteration)
  - [Stream Processing](#stream-processing)

- [4. Concurrent Execution Control](#4-concurrent-execution-control)
  - [Throttling](#throttling)
  - [Debouncing](#debouncing)
  - [Rate Limiting](#rate-limiting)

- [5. Memory Leaks và Performance](#5-memory-leaks-và-performance)
  - [Common Memory Leaks](#common-memory-leaks)
  - [Performance Optimization](#performance-optimization)

- [6. Integration với Async Libraries](#6-integration-với-async-libraries)
  - [RxJS Integration](#rxjs-integration)
  - [P-Queue](#p-queue)

- [7. Testing Async Code](#7-testing-async-code)
  - [Jest Testing](#jest-testing)
  - [Vitest Testing](#vitest-testing)
  - [Mock Async Operations](#mock-async-operations)

---

## 1. Advanced Async Patterns

### Retry Logic với Exponential Backoff

#### Advanced Retry với Jitter

```typescript
// Example 1: Retry với exponential backoff và jitter
interface RetryOptions {
  maxRetries: number;
  baseDelay: number;
  maxDelay: number;
  factor: number;
  jitter: boolean;
}

class RetryableError extends Error {
  constructor(message: string, public retryable: boolean = true) {
    super(message);
    this.name = 'RetryableError';
  }
}

async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  options: Partial<RetryOptions> = {}
): Promise<T> {
  const {
    maxRetries = 5,
    baseDelay = 1000,
    maxDelay = 30000,
    factor = 2,
    jitter = true,
  } = options;
  
  let lastError: Error;
  
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;
      
      // Check if error is retryable
      if (error instanceof RetryableError && !error.retryable) {
        throw error;
      }
      
      if (attempt === maxRetries - 1) {
        throw lastError;
      }
      
      // Calculate delay với exponential backoff
      let delay = Math.min(baseDelay * Math.pow(factor, attempt), maxDelay);
      
      // Add jitter để tránh thundering herd
      if (jitter) {
        delay = delay * (0.5 + Math.random() * 0.5);
      }
      
      console.log(`Retry attempt ${attempt + 1} after ${delay.toFixed(0)}ms`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
  
  throw lastError!;
}

// Usage
const data = await retryWithBackoff(
  () => fetch('/api/data').then(r => r.json()),
  {
    maxRetries: 5,
    baseDelay: 1000,
    maxDelay: 30000,
    factor: 2,
    jitter: true,
  }
);
```

**Giải thích:**
- **Exponential Backoff**: Delay tăng theo cấp số nhân (1s → 2s → 4s → 8s)
- **Max Delay**: Giới hạn delay tối đa
- **Jitter**: Random delay để tránh nhiều clients retry cùng lúc
- **Retryable Errors**: Chỉ retry những errors có thể recover

### Circuit Breaker Pattern

```typescript
// Example 2: Circuit Breaker implementation
enum CircuitState {
  CLOSED = 'CLOSED',     // Normal operation
  OPEN = 'OPEN',         // Failing, reject immediately
  HALF_OPEN = 'HALF_OPEN' // Testing if service recovered
}

interface CircuitBreakerOptions {
  failureThreshold: number;
  successThreshold: number;
  timeout: number;
}

class CircuitBreaker {
  private state: CircuitState = CircuitState.CLOSED;
  private failureCount = 0;
  private successCount = 0;
  private nextAttempt = Date.now();
  
  constructor(private options: CircuitBreakerOptions) {}
  
  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === CircuitState.OPEN) {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      // Try half-open
      this.state = CircuitState.HALF_OPEN;
    }
    
    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  private onSuccess(): void {
    this.failureCount = 0;
    
    if (this.state === CircuitState.HALF_OPEN) {
      this.successCount++;
      
      if (this.successCount >= this.options.successThreshold) {
        this.state = CircuitState.CLOSED;
        this.successCount = 0;
      }
    }
  }
  
  private onFailure(): void {
    this.failureCount++;
    this.successCount = 0;
    
    if (this.failureCount >= this.options.failureThreshold) {
      this.state = CircuitState.OPEN;
      this.nextAttempt = Date.now() + this.options.timeout;
    }
  }
  
  getState(): CircuitState {
    return this.state;
  }
}

// Usage
const breaker = new CircuitBreaker({
  failureThreshold: 5,    // Open after 5 failures
  successThreshold: 2,    // Close after 2 successes
  timeout: 60000,         // Try again after 60s
});

async function fetchWithCircuitBreaker(url: string) {
  return breaker.execute(() => fetch(url).then(r => r.json()));
}
```

**Giải thích Circuit Breaker:**
- **CLOSED**: Hoạt động bình thường
- **OPEN**: Service đang fail, reject ngay không gọi
- **HALF_OPEN**: Test xem service đã recover chưa
- Tránh overwhelm failing service
- Fast fail thay vì đợi timeout

### Bulkhead Pattern

```typescript
// Example 3: Bulkhead pattern - Isolate resources
class Bulkhead {
  private activeRequests = 0;
  private queue: Array<() => void> = [];

  constructor(private maxConcurrent: number) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    // Wait for slot
    await this.acquireSlot();

    try {
      return await fn();
    } finally {
      this.releaseSlot();
    }
  }

  private async acquireSlot(): Promise<void> {
    if (this.activeRequests < this.maxConcurrent) {
      this.activeRequests++;
      return;
    }

    // Wait in queue
    await new Promise<void>(resolve => {
      this.queue.push(resolve);
    });
  }

  private releaseSlot(): void {
    const next = this.queue.shift();
    if (next) {
      next();
    } else {
      this.activeRequests--;
    }
  }
}

// Usage: Limit concurrent API calls
const apiBulkhead = new Bulkhead(5); // Max 5 concurrent

async function fetchUser(id: string) {
  return apiBulkhead.execute(() =>
    fetch(`/api/users/${id}`).then(r => r.json())
  );
}
```

## 2. Custom Promise Implementations

### Deferred Promise

```typescript
// Example 1: Deferred Promise
class Deferred<T> {
  promise: Promise<T>;
  resolve!: (value: T) => void;
  reject!: (reason?: any) => void;

  constructor() {
    this.promise = new Promise<T>((resolve, reject) => {
      this.resolve = resolve;
      this.reject = reject;
    });
  }
}

// Usage: Control promise resolution externally
const deferred = new Deferred<string>();

setTimeout(() => {
  deferred.resolve('Done!');
}, 1000);

const result = await deferred.promise; // 'Done!'
```

### Promise Pool

```typescript
// Example 2: Promise Pool - Controlled concurrency
class PromisePool<T> {
  private running = 0;
  private queue: Array<() => Promise<T>> = [];

  constructor(private concurrency: number) {}

  async add(fn: () => Promise<T>): Promise<T> {
    if (this.running >= this.concurrency) {
      await new Promise<void>(resolve => {
        this.queue.push(async () => {
          const result = await fn();
          resolve();
          return result;
        });
      });
    }

    this.running++;

    try {
      return await fn();
    } finally {
      this.running--;
      const next = this.queue.shift();
      if (next) {
        next();
      }
    }
  }

  async addAll(fns: Array<() => Promise<T>>): Promise<T[]> {
    return Promise.all(fns.map(fn => this.add(fn)));
  }
}

// Usage
const pool = new PromisePool<User>(3); // Max 3 concurrent

const users = await pool.addAll(
  userIds.map(id => () => getUser(id))
);
```

### Lazy Promise

```typescript
// Example 3: Lazy Promise - Chỉ execute khi cần
class LazyPromise<T> {
  private promise: Promise<T> | null = null;

  constructor(private executor: () => Promise<T>) {}

  then<TResult>(
    onFulfilled?: (value: T) => TResult | PromiseLike<TResult>
  ): Promise<TResult> {
    if (!this.promise) {
      this.promise = this.executor();
    }
    return this.promise.then(onFulfilled);
  }

  catch<TResult>(
    onRejected?: (reason: any) => TResult | PromiseLike<TResult>
  ): Promise<T | TResult> {
    if (!this.promise) {
      this.promise = this.executor();
    }
    return this.promise.catch(onRejected);
  }
}

// Usage
const lazyUser = new LazyPromise(() => getUser('123'));

// Chưa execute ở đây
console.log('Created lazy promise');

// Execute khi gọi .then()
const user = await lazyUser; // Bây giờ mới fetch
```

## 3. Async Iterators và Generators

### Async Generators

```typescript
// Example 1: Async generator cơ bản
async function* numberGenerator(max: number) {
  for (let i = 0; i < max; i++) {
    await new Promise(resolve => setTimeout(resolve, 100));
    yield i;
  }
}

// Usage
for await (const num of numberGenerator(5)) {
  console.log(num); // 0, 1, 2, 3, 4 (mỗi số cách nhau 100ms)
}
```

### Async Iteration

```typescript
// Example 2: Paginated API với async generator
interface Page<T> {
  items: T[];
  nextCursor?: string;
}

async function* fetchAllPages<T>(
  endpoint: string
): AsyncGenerator<T, void, undefined> {
  let cursor: string | undefined;

  while (true) {
    const url = cursor
      ? `${endpoint}?cursor=${cursor}`
      : endpoint;

    const response = await fetch(url);
    const page: Page<T> = await response.json();

    // Yield từng item
    for (const item of page.items) {
      yield item;
    }

    // Dừng nếu không còn page
    if (!page.nextCursor) {
      break;
    }

    cursor = page.nextCursor;
  }
}

// Usage: Process tất cả users từ paginated API
for await (const user of fetchAllPages<User>('/api/users')) {
  console.log('Processing user:', user.id);
  await processUser(user);
}
```

### Stream Processing

```typescript
// Example 3: Transform stream với async generators
async function* mapAsync<T, U>(
  source: AsyncIterable<T>,
  mapper: (item: T) => Promise<U>
): AsyncGenerator<U, void, undefined> {
  for await (const item of source) {
    yield await mapper(item);
  }
}

async function* filterAsync<T>(
  source: AsyncIterable<T>,
  predicate: (item: T) => Promise<boolean>
): AsyncGenerator<T, void, undefined> {
  for await (const item of source) {
    if (await predicate(item)) {
      yield item;
    }
  }
}

// Usage: Pipeline processing
const users = fetchAllPages<User>('/api/users');
const activeUsers = filterAsync(users, async (user) => {
  const status = await getUserStatus(user.id);
  return status === 'active';
});
const enrichedUsers = mapAsync(activeUsers, async (user) => {
  const orders = await getOrders(user.id);
  return { ...user, orderCount: orders.length };
});

for await (const user of enrichedUsers) {
  console.log(`${user.name}: ${user.orderCount} orders`);
}
```

## 4. Concurrent Execution Control

### Throttling

```typescript
// Example 1: Throttle function
function throttle<T extends (...args: any[]) => Promise<any>>(
  fn: T,
  limitMs: number
): T {
  let lastRun = 0;
  let pending: Promise<any> | null = null;

  return (async (...args: Parameters<T>) => {
    const now = Date.now();
    const timeSinceLastRun = now - lastRun;

    if (timeSinceLastRun >= limitMs) {
      lastRun = now;
      return fn(...args);
    }

    // Wait for remaining time
    if (!pending) {
      pending = new Promise(resolve => {
        setTimeout(() => {
          lastRun = Date.now();
          pending = null;
          resolve(fn(...args));
        }, limitMs - timeSinceLastRun);
      });
    }

    return pending;
  }) as T;
}

// Usage
const throttledFetch = throttle(
  async (url: string) => fetch(url).then(r => r.json()),
  1000 // Max 1 call per second
);

// Chỉ 1 request được gửi mỗi giây
throttledFetch('/api/data');
throttledFetch('/api/data'); // Đợi
throttledFetch('/api/data'); // Đợi
```

### Debouncing

```typescript
// Example 2: Debounce function
function debounce<T extends (...args: any[]) => Promise<any>>(
  fn: T,
  delayMs: number
): T {
  let timeoutId: NodeJS.Timeout | null = null;
  let pendingPromise: Promise<any> | null = null;
  let resolve: ((value: any) => void) | null = null;

  return ((...args: Parameters<T>) => {
    if (timeoutId) {
      clearTimeout(timeoutId);
    }

    if (!pendingPromise) {
      pendingPromise = new Promise(res => {
        resolve = res;
      });
    }

    timeoutId = setTimeout(async () => {
      const result = await fn(...args);
      resolve!(result);
      pendingPromise = null;
      resolve = null;
      timeoutId = null;
    }, delayMs);

    return pendingPromise;
  }) as T;
}

// Usage: Search với debounce
const debouncedSearch = debounce(
  async (query: string) => {
    const response = await fetch(`/api/search?q=${query}`);
    return response.json();
  },
  300 // Wait 300ms after last keystroke
);

// Chỉ request cuối cùng được gửi
debouncedSearch('a');
debouncedSearch('ab');
debouncedSearch('abc'); // Chỉ cái này được execute
```

### Rate Limiting

```typescript
// Example 3: Token Bucket Rate Limiter
class RateLimiter {
  private tokens: number;
  private lastRefill: number;

  constructor(
    private maxTokens: number,
    private refillRate: number // tokens per second
  ) {
    this.tokens = maxTokens;
    this.lastRefill = Date.now();
  }

  private refill(): void {
    const now = Date.now();
    const timePassed = (now - this.lastRefill) / 1000;
    const tokensToAdd = timePassed * this.refillRate;

    this.tokens = Math.min(this.maxTokens, this.tokens + tokensToAdd);
    this.lastRefill = now;
  }

  async acquire(): Promise<void> {
    this.refill();

    if (this.tokens >= 1) {
      this.tokens--;
      return;
    }

    // Wait for next token
    const waitTime = (1 - this.tokens) / this.refillRate * 1000;
    await new Promise(resolve => setTimeout(resolve, waitTime));

    this.tokens = 0;
  }

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    await this.acquire();
    return fn();
  }
}

// Usage: Limit API calls to 10 per second
const limiter = new RateLimiter(10, 10);

async function fetchWithRateLimit(url: string) {
  return limiter.execute(() => fetch(url).then(r => r.json()));
}
```

## 5. Memory Leaks và Performance

### Common Memory Leaks

```typescript
// ❌ Example 1: Memory leak với event listeners
class BadComponent {
  private data: any[] = [];

  async startPolling() {
    setInterval(async () => {
      const newData = await fetchData();
      this.data.push(newData); // Memory leak! Array grows forever
    }, 1000);
  }
}

// ✅ Fix: Cleanup và limit size
class GoodComponent {
  private data: any[] = [];
  private intervalId: NodeJS.Timeout | null = null;
  private maxSize = 100;

  startPolling() {
    this.intervalId = setInterval(async () => {
      const newData = await fetchData();
      this.data.push(newData);

      // Limit array size
      if (this.data.length > this.maxSize) {
        this.data.shift();
      }
    }, 1000);
  }

  stopPolling() {
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }

  cleanup() {
    this.stopPolling();
    this.data = [];
  }
}
```

```typescript
// ❌ Example 2: Memory leak với promises
class BadCache {
  private cache = new Map<string, Promise<any>>();

  async get(key: string) {
    if (!this.cache.has(key)) {
      this.cache.set(key, this.fetch(key)); // Never cleaned up!
    }
    return this.cache.get(key);
  }

  private async fetch(key: string) {
    return fetch(`/api/${key}`).then(r => r.json());
  }
}

// ✅ Fix: LRU Cache với size limit
class GoodCache {
  private cache = new Map<string, Promise<any>>();
  private maxSize = 100;

  async get(key: string) {
    if (!this.cache.has(key)) {
      this.cache.set(key, this.fetch(key));
      this.evictIfNeeded();
    } else {
      // Move to end (LRU)
      const value = this.cache.get(key)!;
      this.cache.delete(key);
      this.cache.set(key, value);
    }
    return this.cache.get(key);
  }

  private evictIfNeeded() {
    if (this.cache.size > this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
  }

  private async fetch(key: string) {
    return fetch(`/api/${key}`).then(r => r.json());
  }
}
```

### Performance Optimization

```typescript
// Example 3: Memoization cho async functions
function memoizeAsync<T extends (...args: any[]) => Promise<any>>(
  fn: T,
  keyFn?: (...args: Parameters<T>) => string
): T {
  const cache = new Map<string, Promise<any>>();

  return (async (...args: Parameters<T>) => {
    const key = keyFn ? keyFn(...args) : JSON.stringify(args);

    if (cache.has(key)) {
      return cache.get(key);
    }

    const promise = fn(...args);
    cache.set(key, promise);

    try {
      return await promise;
    } catch (error) {
      // Remove failed promises from cache
      cache.delete(key);
      throw error;
    }
  }) as T;
}

// Usage
const memoizedGetUser = memoizeAsync(
  async (id: string) => {
    console.log('Fetching user', id);
    return fetch(`/api/users/${id}`).then(r => r.json());
  }
);

await memoizedGetUser('123'); // Fetch
await memoizedGetUser('123'); // From cache
```

## 6. Integration với Async Libraries

### RxJS Integration

```typescript
// Example 1: Convert Promise to Observable
import { from, Observable } from 'rxjs';
import { retry, timeout, catchError } from 'rxjs/operators';

function promiseToObservable<T>(promise: Promise<T>): Observable<T> {
  return from(promise);
}

// Usage với operators
const user$ = promiseToObservable(getUser('123')).pipe(
  timeout(5000),
  retry(3),
  catchError(error => {
    console.error('Failed to fetch user:', error);
    throw error;
  })
);

user$.subscribe({
  next: user => console.log('User:', user),
  error: error => console.error('Error:', error),
});
```

### P-Queue

```typescript
// Example 2: Using p-queue library
import PQueue from 'p-queue';

const queue = new PQueue({
  concurrency: 3,
  interval: 1000,
  intervalCap: 10, // Max 10 operations per second
});

// Add tasks to queue
const users = await Promise.all(
  userIds.map(id =>
    queue.add(() => getUser(id))
  )
);

// Priority queue
await queue.add(() => urgentTask(), { priority: 1 });
await queue.add(() => normalTask(), { priority: 0 });
```

## 7. Testing Async Code

### Jest Testing

```typescript
// Example 1: Basic async tests
describe('User API', () => {
  test('should fetch user', async () => {
    const user = await getUser('123');
    expect(user).toHaveProperty('id', '123');
  });

  test('should handle errors', async () => {
    await expect(getUser('invalid')).rejects.toThrow('User not found');
  });

  test('should timeout', async () => {
    jest.setTimeout(1000);
    await expect(slowOperation()).rejects.toThrow('Timeout');
  });
});
```

### Mock Async Operations

```typescript
// Example 2: Mocking với Jest
import { jest } from '@jest/globals';

// Mock fetch
global.fetch = jest.fn();

describe('fetchUser', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('should fetch user successfully', async () => {
    const mockUser = { id: '123', name: 'John' };

    (fetch as jest.Mock).mockResolvedValueOnce({
      ok: true,
      json: async () => mockUser,
    });

    const user = await fetchUser('123');

    expect(fetch).toHaveBeenCalledWith('/api/users/123');
    expect(user).toEqual(mockUser);
  });

  test('should handle fetch error', async () => {
    (fetch as jest.Mock).mockRejectedValueOnce(
      new Error('Network error')
    );

    await expect(fetchUser('123')).rejects.toThrow('Network error');
  });
});
```

### Vitest Testing

```typescript
// Example 3: Vitest với fake timers
import { describe, test, expect, vi, beforeEach } from 'vitest';

describe('Debounce', () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  test('should debounce async function', async () => {
    const fn = vi.fn(async (x: number) => x * 2);
    const debounced = debounce(fn, 100);

    debounced(1);
    debounced(2);
    debounced(3);

    // Fast-forward time
    vi.advanceTimersByTime(100);

    await vi.runAllTimersAsync();

    // Chỉ call cuối cùng được execute
    expect(fn).toHaveBeenCalledTimes(1);
    expect(fn).toHaveBeenCalledWith(3);
  });
});
```

### Integration Testing với MSW

```typescript
// Example 4: Mock Service Worker
import { rest } from 'msw';
import { setupServer } from 'msw/node';

const server = setupServer(
  rest.get('/api/users/:id', (req, res, ctx) => {
    const { id } = req.params;
    return res(
      ctx.json({ id, name: 'John Doe' })
    );
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test('should fetch user from API', async () => {
  const user = await fetchUser('123');
  expect(user).toEqual({ id: '123', name: 'John Doe' });
});

test('should handle API error', async () => {
  server.use(
    rest.get('/api/users/:id', (req, res, ctx) => {
      return res(ctx.status(500));
    })
  );

  await expect(fetchUser('123')).rejects.toThrow();
});
```

---

## Best Practices - Senior Level

### 1. Implement Proper Error Recovery

```typescript
// ✅ Retry với exponential backoff
// ✅ Circuit breaker cho failing services
// ✅ Fallback strategies
// ✅ Graceful degradation
```

### 2. Control Concurrency

```typescript
// ✅ Use promise pools
// ✅ Implement rate limiting
// ✅ Batch processing
// ✅ Bulkhead pattern
```

### 3. Prevent Memory Leaks

```typescript
// ✅ Cleanup timers và listeners
// ✅ Limit cache size
// ✅ Use WeakMap cho object references
// ✅ Cancel pending operations on unmount
```

### 4. Optimize Performance

```typescript
// ✅ Memoize expensive async operations
// ✅ Use async generators cho large datasets
// ✅ Implement proper caching strategies
// ✅ Profile và monitor async operations
```

### 5. Write Testable Code

```typescript
// ✅ Inject dependencies
// ✅ Use interfaces
// ✅ Mock external services
// ✅ Test error cases
```

---

## Tài Liệu Tham Khảo

- [RxJS Documentation](https://rxjs.dev/)
- [p-queue Library](https://github.com/sindresorhus/p-queue)
- [MSW Documentation](https://mswjs.io/)
- [Jest Async Testing](https://jestjs.io/docs/asynchronous)
- [Vitest Documentation](https://vitest.dev/)

**Tiếp theo: Xem `Principal-Async-Patterns.md` cho Enterprise-level patterns**

