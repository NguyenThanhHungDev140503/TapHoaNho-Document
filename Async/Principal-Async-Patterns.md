# Principal-Level Async Patterns - Enterprise Architecture

## Mục Lục

- [1. System Design với Async Operations](#1-system-design-với-async-operations)
  - [Microservices Communication](#microservices-communication)
  - [Event-Driven Architecture](#event-driven-architecture)
  - [CQRS Pattern](#cqrs-pattern)

- [2. Event Loop Deep Dive](#2-event-loop-deep-dive)
  - [Call Stack](#call-stack)
  - [Microtasks vs Macrotasks](#microtasks-vs-macrotasks)
  - [Execution Order](#execution-order)

- [3. Worker Threads và Parallel Processing](#3-worker-threads-và-parallel-processing)
  - [Worker Threads](#worker-threads)
  - [Thread Pool](#thread-pool)
  - [Shared Memory](#shared-memory)

- [4. Streaming và Backpressure](#4-streaming-và-backpressure)
  - [Node.js Streams](#nodejs-streams)
  - [Backpressure Handling](#backpressure-handling)
  - [Transform Streams](#transform-streams)

- [5. Distributed Async Patterns](#5-distributed-async-patterns)
  - [Message Queues](#message-queues)
  - [Saga Pattern](#saga-pattern)
  - [Outbox Pattern](#outbox-pattern)

- [6. Production Monitoring](#6-production-monitoring)
  - [Metrics Collection](#metrics-collection)
  - [Distributed Tracing](#distributed-tracing)
  - [Error Tracking](#error-tracking)

- [7. Migration Strategies](#7-migration-strategies)
  - [Callback to Promise](#callback-to-promise)
  - [Legacy System Integration](#legacy-system-integration)

---

## 1. System Design với Async Operations

### Microservices Communication

```typescript
// Example 1: Service-to-Service communication với retry và circuit breaker
interface ServiceConfig {
  baseUrl: string;
  timeout: number;
  retries: number;
}

class MicroserviceClient {
  private circuitBreaker: CircuitBreaker;
  
  constructor(private config: ServiceConfig) {
    this.circuitBreaker = new CircuitBreaker({
      failureThreshold: 5,
      successThreshold: 2,
      timeout: 60000,
    });
  }
  
  async call<T>(
    endpoint: string,
    options: RequestInit = {}
  ): Promise<T> {
    return this.circuitBreaker.execute(async () => {
      return retryWithBackoff(
        async () => {
          const controller = new AbortController();
          const timeoutId = setTimeout(
            () => controller.abort(),
            this.config.timeout
          );
          
          try {
            const response = await fetch(
              `${this.config.baseUrl}${endpoint}`,
              {
                ...options,
                signal: controller.signal,
              }
            );
            
            if (!response.ok) {
              throw new Error(`HTTP ${response.status}`);
            }
            
            return await response.json();
          } finally {
            clearTimeout(timeoutId);
          }
        },
        { maxRetries: this.config.retries }
      );
    });
  }
}

// Usage
const userService = new MicroserviceClient({
  baseUrl: 'https://user-service.example.com',
  timeout: 5000,
  retries: 3,
});

const orderService = new MicroserviceClient({
  baseUrl: 'https://order-service.example.com',
  timeout: 5000,
  retries: 3,
});

// Parallel calls với fallback
async function getUserDashboard(userId: string) {
  try {
    const [user, orders] = await Promise.all([
      userService.call(`/users/${userId}`),
      orderService.call(`/orders?userId=${userId}`),
    ]);
    
    return { user, orders };
  } catch (error) {
    console.error('Failed to load dashboard:', error);
    
    // Fallback: Load from cache
    return {
      user: await getCachedUser(userId),
      orders: [],
    };
  }
}
```

### Event-Driven Architecture

```typescript
// Example 2: Event Bus implementation
type EventHandler<T = any> = (data: T) => Promise<void>;

class EventBus {
  private handlers = new Map<string, Set<EventHandler>>();
  private eventQueue: Array<{ event: string; data: any }> = [];
  private processing = false;
  
  on<T>(event: string, handler: EventHandler<T>): () => void {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, new Set());
    }
    
    this.handlers.get(event)!.add(handler);
    
    // Return unsubscribe function
    return () => {
      this.handlers.get(event)?.delete(handler);
    };
  }
  
  async emit<T>(event: string, data: T): Promise<void> {
    this.eventQueue.push({ event, data });
    
    if (!this.processing) {
      await this.processQueue();
    }
  }
  
  private async processQueue(): Promise<void> {
    this.processing = true;
    
    while (this.eventQueue.length > 0) {
      const { event, data } = this.eventQueue.shift()!;
      const handlers = this.handlers.get(event);
      
      if (handlers) {
        // Execute all handlers in parallel
        await Promise.allSettled(
          Array.from(handlers).map(handler => handler(data))
        );
      }
    }
    
    this.processing = false;
  }
}

// Usage
const eventBus = new EventBus();

// Subscribe to events
eventBus.on('user.created', async (user) => {
  await sendWelcomeEmail(user);
});

eventBus.on('user.created', async (user) => {
  await createUserProfile(user);
});

eventBus.on('order.placed', async (order) => {
  await processPayment(order);
  await updateInventory(order);
  await notifyWarehouse(order);
});

// Emit events
await eventBus.emit('user.created', { id: '123', email: 'user@example.com' });
await eventBus.emit('order.placed', { id: 'order-1', userId: '123' });
```

### CQRS Pattern

```typescript
// Example 3: Command Query Responsibility Segregation
interface Command {
  type: string;
  payload: any;
}

interface Query {
  type: string;
  params: any;
}

class CommandBus {
  private handlers = new Map<string, (payload: any) => Promise<void>>();

  register(commandType: string, handler: (payload: any) => Promise<void>) {
    this.handlers.set(commandType, handler);
  }

  async execute(command: Command): Promise<void> {
    const handler = this.handlers.get(command.type);
    if (!handler) {
      throw new Error(`No handler for command: ${command.type}`);
    }

    await handler(command.payload);
  }
}

class QueryBus {
  private handlers = new Map<string, (params: any) => Promise<any>>();

  register(queryType: string, handler: (params: any) => Promise<any>) {
    this.handlers.set(queryType, handler);
  }

  async execute<T>(query: Query): Promise<T> {
    const handler = this.handlers.get(query.type);
    if (!handler) {
      throw new Error(`No handler for query: ${query.type}`);
    }

    return await handler(query.params);
  }
}

// Setup
const commandBus = new CommandBus();
const queryBus = new QueryBus();

// Register command handlers (write operations)
commandBus.register('CreateUser', async (payload) => {
  await db.users.insert(payload);
  await eventBus.emit('user.created', payload);
});

commandBus.register('UpdateUser', async (payload) => {
  await db.users.update(payload.id, payload.data);
  await eventBus.emit('user.updated', payload);
});

// Register query handlers (read operations)
queryBus.register('GetUser', async (params) => {
  return await db.users.findById(params.id);
});

queryBus.register('GetUserOrders', async (params) => {
  return await db.orders.findByUserId(params.userId);
});

// Usage
await commandBus.execute({
  type: 'CreateUser',
  payload: { name: 'John', email: 'john@example.com' },
});

const user = await queryBus.execute({
  type: 'GetUser',
  params: { id: '123' },
});
```

## 2. Event Loop Deep Dive

### Call Stack, Microtasks, Macrotasks

```typescript
// Example 1: Execution order demonstration
console.log('1: Synchronous');

setTimeout(() => {
  console.log('2: Macrotask (setTimeout)');
}, 0);

Promise.resolve().then(() => {
  console.log('3: Microtask (Promise)');
});

queueMicrotask(() => {
  console.log('4: Microtask (queueMicrotask)');
});

console.log('5: Synchronous');

// Output:
// 1: Synchronous
// 5: Synchronous
// 3: Microtask (Promise)
// 4: Microtask (queueMicrotask)
// 2: Macrotask (setTimeout)
```

**Execution Order:**
1. **Call Stack**: Synchronous code chạy trước
2. **Microtasks Queue**: Promise callbacks, queueMicrotask
3. **Macrotasks Queue**: setTimeout, setInterval, I/O

```typescript
// Example 2: Complex execution order
async function complexExample() {
  console.log('1');

  setTimeout(() => console.log('2'), 0);

  await Promise.resolve();
  console.log('3');

  queueMicrotask(() => console.log('4'));

  setTimeout(() => console.log('5'), 0);

  console.log('6');
}

complexExample();
console.log('7');

// Output:
// 1
// 7
// 3
// 6
// 4
// 2
// 5
```

**Giải thích:**
- `1`: Synchronous
- `7`: Synchronous (main thread)
- `3`: Sau await (microtask)
- `6`: Synchronous trong async function
- `4`: queueMicrotask
- `2`, `5`: setTimeout (macrotasks)

### Microtask Starvation

```typescript
// Example 3: Microtask starvation problem
function createMicrotaskLoop() {
  queueMicrotask(() => {
    console.log('Microtask');
    createMicrotaskLoop(); // Infinite loop!
  });
}

createMicrotaskLoop();

setTimeout(() => {
  console.log('This will never run!'); // Starved by microtasks
}, 0);
```

**Vấn đề**: Microtasks có priority cao hơn macrotasks, nếu tạo microtasks liên tục sẽ block macrotasks.

**Giải pháp**: Sử dụng setImmediate (Node.js) hoặc setTimeout để yield control.

## 3. Worker Threads và Parallel Processing

### Worker Threads

```typescript
// Example 1: CPU-intensive task với Worker Threads
// worker.ts
import { parentPort, workerData } from 'worker_threads';

function fibonacci(n: number): number {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

const result = fibonacci(workerData.n);
parentPort?.postMessage(result);

// main.ts
import { Worker } from 'worker_threads';

function runWorker(n: number): Promise<number> {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./worker.js', {
      workerData: { n },
    });

    worker.on('message', resolve);
    worker.on('error', reject);
    worker.on('exit', (code) => {
      if (code !== 0) {
        reject(new Error(`Worker stopped with exit code ${code}`));
      }
    });
  });
}

// Usage: Calculate fibonacci in parallel
const results = await Promise.all([
  runWorker(40),
  runWorker(41),
  runWorker(42),
]);
```

### Thread Pool

```typescript
// Example 2: Worker Thread Pool
class WorkerPool {
  private workers: Worker[] = [];
  private queue: Array<{
    data: any;
    resolve: (value: any) => void;
    reject: (error: any) => void;
  }> = [];

  constructor(
    private workerScript: string,
    private poolSize: number
  ) {
    this.initializeWorkers();
  }

  private initializeWorkers() {
    for (let i = 0; i < this.poolSize; i++) {
      const worker = new Worker(this.workerScript);

      worker.on('message', (result) => {
        const task = this.queue.shift();
        if (task) {
          task.resolve(result);
          this.processNext(worker);
        }
      });

      worker.on('error', (error) => {
        const task = this.queue.shift();
        if (task) {
          task.reject(error);
        }
      });

      this.workers.push(worker);
    }
  }

  private processNext(worker: Worker) {
    if (this.queue.length > 0) {
      const task = this.queue[0];
      worker.postMessage(task.data);
    }
  }

  async execute<T>(data: any): Promise<T> {
    return new Promise((resolve, reject) => {
      this.queue.push({ data, resolve, reject });

      // Find idle worker
      const idleWorker = this.workers.find(w => {
        // Check if worker is idle (simplified)
        return true;
      });

      if (idleWorker) {
        this.processNext(idleWorker);
      }
    });
  }

  async terminate() {
    await Promise.all(
      this.workers.map(worker => worker.terminate())
    );
  }
}

// Usage
const pool = new WorkerPool('./worker.js', 4);

const results = await Promise.all(
  [40, 41, 42, 43].map(n => pool.execute({ n }))
);

await pool.terminate();
```

## 4. Streaming và Backpressure

### Node.js Streams

```typescript
// Example 1: Transform stream
import { Transform } from 'stream';

class UpperCaseTransform extends Transform {
  _transform(
    chunk: any,
    encoding: string,
    callback: (error?: Error, data?: any) => void
  ) {
    const upperChunk = chunk.toString().toUpperCase();
    this.push(upperChunk);
    callback();
  }
}

// Usage
const transform = new UpperCaseTransform();
process.stdin.pipe(transform).pipe(process.stdout);
```

### Backpressure Handling

```typescript
// Example 2: Handle backpressure
import { Writable } from 'stream';

class SlowWriter extends Writable {
  async _write(
    chunk: any,
    encoding: string,
    callback: (error?: Error) => void
  ) {
    // Simulate slow processing
    await new Promise(resolve => setTimeout(resolve, 100));
    console.log('Processed:', chunk.toString());
    callback();
  }
}

async function processLargeFile(filePath: string) {
  const readStream = fs.createReadStream(filePath);
  const writer = new SlowWriter();

  readStream.pipe(writer);

  // Handle backpressure
  readStream.on('data', (chunk) => {
    const canContinue = writer.write(chunk);

    if (!canContinue) {
      // Pause reading until drain
      readStream.pause();

      writer.once('drain', () => {
        readStream.resume();
      });
    }
  });

  await new Promise((resolve, reject) => {
    writer.on('finish', resolve);
    writer.on('error', reject);
  });
}
```

## 5. Distributed Async Patterns

### Message Queues

```typescript
// Example 1: Message Queue với RabbitMQ pattern
interface Message<T> {
  id: string;
  data: T;
  timestamp: number;
  retries: number;
}

class MessageQueue<T> {
  private queue: Message<T>[] = [];
  private processing = false;
  private deadLetterQueue: Message<T>[] = [];
  private maxRetries = 3;

  async enqueue(data: T): Promise<void> {
    const message: Message<T> = {
      id: crypto.randomUUID(),
      data,
      timestamp: Date.now(),
      retries: 0,
    };

    this.queue.push(message);

    if (!this.processing) {
      await this.process();
    }
  }

  private async process(): Promise<void> {
    this.processing = true;

    while (this.queue.length > 0) {
      const message = this.queue.shift()!;

      try {
        await this.handleMessage(message);
      } catch (error) {
        await this.handleError(message, error);
      }
    }

    this.processing = false;
  }

  private async handleMessage(message: Message<T>): Promise<void> {
    // Process message
    console.log('Processing message:', message.id);

    // Simulate processing
    await new Promise(resolve => setTimeout(resolve, 100));

    // Acknowledge message
    console.log('Message processed:', message.id);
  }

  private async handleError(message: Message<T>, error: any): Promise<void> {
    message.retries++;

    if (message.retries < this.maxRetries) {
      // Retry với exponential backoff
      const delay = Math.pow(2, message.retries) * 1000;
      console.log(`Retry message ${message.id} after ${delay}ms`);

      await new Promise(resolve => setTimeout(resolve, delay));
      this.queue.unshift(message); // Add back to front
    } else {
      // Move to dead letter queue
      console.error('Message failed after max retries:', message.id);
      this.deadLetterQueue.push(message);
    }
  }

  getDeadLetterQueue(): Message<T>[] {
    return this.deadLetterQueue;
  }
}

// Usage
const queue = new MessageQueue<{ userId: string; action: string }>();

await queue.enqueue({ userId: '123', action: 'send-email' });
await queue.enqueue({ userId: '456', action: 'process-order' });
```

### Saga Pattern

```typescript
// Example 2: Saga Pattern cho distributed transactions
interface SagaStep<T> {
  execute: (context: T) => Promise<void>;
  compensate: (context: T) => Promise<void>;
}

class Saga<T> {
  private steps: SagaStep<T>[] = [];
  private executedSteps: SagaStep<T>[] = [];

  addStep(step: SagaStep<T>): this {
    this.steps.push(step);
    return this;
  }

  async execute(context: T): Promise<void> {
    try {
      for (const step of this.steps) {
        await step.execute(context);
        this.executedSteps.push(step);
      }
    } catch (error) {
      console.error('Saga failed, compensating...', error);
      await this.compensate(context);
      throw error;
    }
  }

  private async compensate(context: T): Promise<void> {
    // Compensate in reverse order
    for (const step of this.executedSteps.reverse()) {
      try {
        await step.compensate(context);
      } catch (error) {
        console.error('Compensation failed:', error);
        // Log but continue compensating
      }
    }
  }
}

// Usage: Order processing saga
interface OrderContext {
  orderId: string;
  userId: string;
  amount: number;
  paymentId?: string;
  inventoryReserved?: boolean;
}

const orderSaga = new Saga<OrderContext>()
  .addStep({
    execute: async (ctx) => {
      console.log('Creating order...');
      await createOrder(ctx.orderId, ctx.userId, ctx.amount);
    },
    compensate: async (ctx) => {
      console.log('Cancelling order...');
      await cancelOrder(ctx.orderId);
    },
  })
  .addStep({
    execute: async (ctx) => {
      console.log('Processing payment...');
      ctx.paymentId = await processPayment(ctx.userId, ctx.amount);
    },
    compensate: async (ctx) => {
      if (ctx.paymentId) {
        console.log('Refunding payment...');
        await refundPayment(ctx.paymentId);
      }
    },
  })
  .addStep({
    execute: async (ctx) => {
      console.log('Reserving inventory...');
      await reserveInventory(ctx.orderId);
      ctx.inventoryReserved = true;
    },
    compensate: async (ctx) => {
      if (ctx.inventoryReserved) {
        console.log('Releasing inventory...');
        await releaseInventory(ctx.orderId);
      }
    },
  });

// Execute saga
try {
  await orderSaga.execute({
    orderId: 'order-123',
    userId: 'user-456',
    amount: 100,
  });
  console.log('Order completed successfully');
} catch (error) {
  console.error('Order failed and compensated');
}
```

### Outbox Pattern

```typescript
// Example 3: Outbox Pattern cho reliable messaging
interface OutboxMessage {
  id: string;
  aggregateId: string;
  eventType: string;
  payload: any;
  createdAt: Date;
  processed: boolean;
}

class OutboxProcessor {
  private processing = false;

  async publishEvent(
    aggregateId: string,
    eventType: string,
    payload: any
  ): Promise<void> {
    // Save to outbox table trong cùng transaction với business logic
    await db.transaction(async (trx) => {
      // Business logic
      await trx.orders.update(aggregateId, payload);

      // Save to outbox
      await trx.outbox.insert({
        id: crypto.randomUUID(),
        aggregateId,
        eventType,
        payload,
        createdAt: new Date(),
        processed: false,
      });
    });

    // Trigger processing
    if (!this.processing) {
      this.processOutbox();
    }
  }

  private async processOutbox(): Promise<void> {
    this.processing = true;

    try {
      const messages = await db.outbox.findUnprocessed();

      for (const message of messages) {
        try {
          // Publish to message broker
          await messageQueue.publish(message.eventType, message.payload);

          // Mark as processed
          await db.outbox.markProcessed(message.id);
        } catch (error) {
          console.error('Failed to process outbox message:', error);
          // Will retry on next run
        }
      }
    } finally {
      this.processing = false;
    }
  }

  // Run periodically
  startPolling(intervalMs: number = 5000) {
    setInterval(() => this.processOutbox(), intervalMs);
  }
}
```

## 6. Production Monitoring

### Metrics Collection

```typescript
// Example 1: Async operation metrics
class AsyncMetrics {
  private metrics = new Map<string, {
    count: number;
    totalDuration: number;
    errors: number;
  }>();

  async measure<T>(
    operation: string,
    fn: () => Promise<T>
  ): Promise<T> {
    const start = Date.now();

    try {
      const result = await fn();
      this.recordSuccess(operation, Date.now() - start);
      return result;
    } catch (error) {
      this.recordError(operation, Date.now() - start);
      throw error;
    }
  }

  private recordSuccess(operation: string, duration: number) {
    const metric = this.getOrCreate(operation);
    metric.count++;
    metric.totalDuration += duration;
  }

  private recordError(operation: string, duration: number) {
    const metric = this.getOrCreate(operation);
    metric.count++;
    metric.totalDuration += duration;
    metric.errors++;
  }

  private getOrCreate(operation: string) {
    if (!this.metrics.has(operation)) {
      this.metrics.set(operation, {
        count: 0,
        totalDuration: 0,
        errors: 0,
      });
    }
    return this.metrics.get(operation)!;
  }

  getMetrics() {
    const result: any = {};

    for (const [operation, metric] of this.metrics) {
      result[operation] = {
        count: metric.count,
        avgDuration: metric.totalDuration / metric.count,
        errorRate: metric.errors / metric.count,
      };
    }

    return result;
  }
}

// Usage
const metrics = new AsyncMetrics();

await metrics.measure('fetchUser', () => getUser('123'));
await metrics.measure('fetchOrders', () => getOrders('123'));

console.log(metrics.getMetrics());
// {
//   fetchUser: { count: 1, avgDuration: 150, errorRate: 0 },
//   fetchOrders: { count: 1, avgDuration: 200, errorRate: 0 }
// }
```

### Distributed Tracing

```typescript
// Example 2: Distributed tracing với correlation IDs
class TraceContext {
  constructor(
    public traceId: string,
    public spanId: string,
    public parentSpanId?: string
  ) {}

  createChildSpan(): TraceContext {
    return new TraceContext(
      this.traceId,
      crypto.randomUUID(),
      this.spanId
    );
  }
}

class Tracer {
  private spans = new Map<string, {
    name: string;
    startTime: number;
    endTime?: number;
    context: TraceContext;
  }>();

  async trace<T>(
    name: string,
    context: TraceContext,
    fn: (ctx: TraceContext) => Promise<T>
  ): Promise<T> {
    const span = {
      name,
      startTime: Date.now(),
      context,
    };

    this.spans.set(context.spanId, span);

    try {
      const result = await fn(context);
      span.endTime = Date.now();
      return result;
    } catch (error) {
      span.endTime = Date.now();
      throw error;
    }
  }

  getTrace(traceId: string) {
    const spans = Array.from(this.spans.values())
      .filter(span => span.context.traceId === traceId);

    return spans.map(span => ({
      name: span.name,
      duration: span.endTime ? span.endTime - span.startTime : null,
      spanId: span.context.spanId,
      parentSpanId: span.context.parentSpanId,
    }));
  }
}

// Usage
const tracer = new Tracer();
const rootContext = new TraceContext(crypto.randomUUID(), crypto.randomUUID());

await tracer.trace('handleRequest', rootContext, async (ctx) => {
  const userCtx = ctx.createChildSpan();
  const user = await tracer.trace('fetchUser', userCtx, async () => {
    return getUser('123');
  });

  const ordersCtx = ctx.createChildSpan();
  const orders = await tracer.trace('fetchOrders', ordersCtx, async () => {
    return getOrders(user.id);
  });

  return { user, orders };
});

console.log(tracer.getTrace(rootContext.traceId));
```

## 7. Migration Strategies

### Callback to Promise

```typescript
// Example 1: Promisify callback-based APIs
function promisify<T>(
  fn: (...args: any[]) => void
): (...args: any[]) => Promise<T> {
  return (...args: any[]) => {
    return new Promise((resolve, reject) => {
      fn(...args, (error: Error | null, result: T) => {
        if (error) {
          reject(error);
        } else {
          resolve(result);
        }
      });
    });
  };
}

// Usage
const readFileAsync = promisify(fs.readFile);
const data = await readFileAsync('file.txt', 'utf-8');
```

### Legacy System Integration

```typescript
// Example 2: Adapter pattern cho legacy systems
class LegacySystemAdapter {
  async callLegacyAPI(endpoint: string, data: any): Promise<any> {
    // Legacy system uses callbacks
    return new Promise((resolve, reject) => {
      legacySystem.call(endpoint, data, (error, result) => {
        if (error) {
          reject(this.transformError(error));
        } else {
          resolve(this.transformResponse(result));
        }
      });
    });
  }

  private transformError(error: any): Error {
    // Transform legacy error format to modern format
    return new Error(error.message || 'Legacy system error');
  }

  private transformResponse(result: any): any {
    // Transform legacy response format
    return {
      data: result.data,
      timestamp: new Date(result.timestamp),
    };
  }
}
```

---

## Best Practices - Principal Level

### 1. Design for Failure
- Implement circuit breakers
- Use saga pattern cho distributed transactions
- Always have compensation logic
- Monitor và alert on failures

### 2. Optimize for Scale
- Use worker threads cho CPU-intensive tasks
- Implement proper backpressure handling
- Use message queues cho async communication
- Cache aggressively

### 3. Observability
- Implement distributed tracing
- Collect metrics on all async operations
- Use correlation IDs
- Log structured data

### 4. Migration Strategy
- Gradual migration từ callbacks sang promises
- Use adapter pattern cho legacy systems
- Maintain backward compatibility
- Test thoroughly

---

## Tài Liệu Tham Khảo

- [Node.js Worker Threads](https://nodejs.org/api/worker_threads.html)
- [Node.js Streams](https://nodejs.org/api/stream.html)
- [Microservices Patterns](https://microservices.io/patterns/)
- [Saga Pattern](https://microservices.io/patterns/data/saga.html)
- [Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)

**Tài liệu này hoàn thành series về Async Programming trong TypeScript!**

