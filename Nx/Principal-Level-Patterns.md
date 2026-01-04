# Nx Principal-Level Patterns

**Table of Contents**
- [Enterprise-Scale Architecture](#enterprise-scale-architecture)
- [Centralized API Client](#centralized-api-client)
- [Advanced Prefetching & Data Loading](#advanced-prefetching--data-loading)
- [Integration với Frameworks Lớn](#integration-với-frameworks-lớn)
- [Migration Strategies](#migration-strategies)
- [Production Monitoring & Metrics](#production-monitoring--metrics)
- [Advanced Error Recovery & Resiliency](#advanced-error-recovery--resiliency)
- [Best Practices Summary](#best-practices-summary)

---

## Enterprise-Scale Architecture

### Multi-Team Monorepo Structure

**Bài toán**: Quản lý monorepo với nhiều teams.

**Giải pháp**: Organize theo bounded contexts/domains.

```
TapHoaNho/
├── apps/
│   ├── billing/              # Billing team
│   ├── inventory/            # Inventory team
│   ├── reporting/             # Reporting team
│   └── shared/               # Platform team
├── libs/
│   ├── ui/                   # Platform team (shared)
│   ├── api-client/            # Platform team (shared)
│   └── domain/               # Domain libraries
│       ├── billing/
│       ├── inventory/
│       └── reporting/
└── tools/
    ├── generators/             # Custom generators
    └── eslint-rules/          # Shared linting rules
```

**Team Responsibilities**:
- **Platform Team**: Shared libs (ui, api-client, utils), Nx config, CI/CD pipelines
- **Domain Teams**: Bounded contexts (billing, inventory, reporting), feature implementations
- **DevOps Team**: Nx Cloud, caching, distributed execution, monitoring

### Nx Configuration cho Enterprise

```json
// nx.json
{
  "targetDefaults": {
    "build": {
      "options": {
        "maxWorkers": 4,
        "parallel": true
      },
      "dependsOn": ["^build"]
    },
    "test": {
      "options": {
        "passWithNoTests": true,
        "codeCoverage": true
      }
    },
    "lint": {
      "options": {
        "fix": true,
        "maxWarnings": 0
      }
    }
  },
  "tasksRunnerOptions": {
    "cacheableOperations": "default",
    "parallel": 5,
    "defaultCacheDir": "/var/cache/nx"
  },
  "nxCloudAccessToken": "${NX_ACCESS_TOKEN}",
  "nxCloudUrl": "https://nx.cloud.taphoanho.com",
  "affected": {
    "defaultBase": "shared"
  }
}
```

### Workspace Governance

**Pattern 1: Architectural Decision Records**

```markdown
// docs/architecture/adr/001-use-nx.md
# ADR-001: Use Nx for Monorepo Management

## Status
Accepted

## Context
We need to manage multiple applications and libraries across teams.

## Decision
Adopt Nx as monorepo management tool with Nx Cloud.

## Consequences
- Build time reduced by 80%
- Team collaboration improved
- Learning curve for new developers

## Alternatives Considered
- Turborepo: Less code generation
- Lerna: No caching, limited features
- pnpm workspaces: No orchestration
```

**Pattern 2: Nx Migration Checklist**

```markdown
// docs/migration/nx-checklist.md
# Nx Migration Checklist

## Planning
- [ ] Define team structure and bounded contexts
- [ ] Identify shared libraries vs domain-specific
- [ ] Plan Nx Cloud integration
- [ ] Define migration phases

## Implementation
- [ ] Set up workspace structure
- [ ] Configure Nx with plugins
- [ ] Create custom generators
- [ ] Set up CI/CD pipelines
- [ ] Enable Nx Cloud caching

## Team Onboarding
- [ ] Create Nx training materials
- [ ] Document team responsibilities
- [ ] Set up Nx Console for team
- [ ] Create migration support channel

## Validation
- [ ] Measure build time improvements
- [ ] Monitor cache hit rates
- [ ] Collect team feedback
- [ ] Adjust configurations
```

---

## Centralized API Client

### Advanced Interceptor Architecture

```typescript
// libs/api-client/src/advancedApiClient.ts
import axios, { AxiosError, AxiosRequestConfig, AxiosResponse } from 'axios';

export interface ApiClientConfig {
  baseUrl: string;
  timeout?: number;
  interceptors?: {
    request?: RequestInterceptor[];
    response?: ResponseInterceptor[];
    error?: ErrorInterceptor[];
  };
  retry?: {
    maxAttempts?: number;
    delay?: number;
    backoff?: 'exponential' | 'linear';
  };
  circuitBreaker?: {
    threshold?: number;
    resetTimeout?: number;
  };
  metrics?: {
    enabled?: boolean;
    collect?: (data: MetricData) => void;
  };
}

export interface MetricData {
  type: 'request' | 'response' | 'error';
  url: string;
  duration?: number;
  status?: number;
  error?: any;
  timestamp: number;
}

export class AdvancedApiClient {
  private client: ReturnType<typeof axios.create>;
  
  constructor(config: ApiClientConfig) {
    this.client = axios.create({
      baseURL: config.baseUrl,
      timeout: config.timeout || 10000
    });
    
    this.setupInterceptors(config);
    this.setupRetry(config);
    this.setupCircuitBreaker(config);
    this.setupMetrics(config);
  }
  
  private setupInterceptors(config: ApiClientConfig) {
    // Request interceptors
    config.interceptors?.request?.forEach(interceptor => {
      this.client.interceptors.request.use(interceptor);
    });
    
    // Response interceptors
    config.interceptors?.response?.forEach(interceptor => {
      this.client.interceptors.response.use(interceptor);
    });
    
    // Error interceptors
    config.interceptors?.error?.forEach(interceptor => {
      this.client.interceptors.response.use(undefined, interceptor);
    });
  }
  
  private setupRetry(config: ApiClientConfig) {
    if (!config.retry) return;
    
    const { maxAttempts = 3, delay = 1000, backoff = 'exponential' } = config.retry;
    
    this.client.interceptors.response.use(
      response => response,
      async (error: AxiosError) => {
        if (!this.shouldRetry(error)) {
          return Promise.reject(error);
        }
        
        const attempt = this.getAttemptCount(error);
        const delay = this.calculateDelay(attempt, delay, backoff);
        
        await this.wait(delay);
        return this.client.request(error.config);
      }
    );
  }
  
  private setupCircuitBreaker(config: ApiClientConfig) {
    if (!config.circuitBreaker) return;
    
    const { threshold = 5, resetTimeout = 60000 } = config.circuitBreaker;
    const failureCount = new Map<string, number>();
    const failureTimestamps = new Map<string, number>();
    
    this.client.interceptors.request.use(request => {
      const url = request.url || '';
      
      if (this.isCircuitOpen(url, failureCount, failureTimestamps, threshold, resetTimeout)) {
        return Promise.reject(new Error('Circuit breaker is open'));
      }
      
      return request;
    });
    
    this.client.interceptors.response.use(
      response => {
        const url = response.config.url || '';
        this.recordSuccess(url, failureCount, failureTimestamps);
        return response;
      },
      error => {
        const url = error.config?.url || '';
        this.recordFailure(url, failureCount, failureTimestamps);
        return Promise.reject(error);
      }
    );
  }
  
  private setupMetrics(config: ApiClientConfig) {
    if (!config.metrics?.enabled) return;
    
    this.client.interceptors.request.use(request => {
      const start = Date.now();
      
      request.metadata = { ...request.metadata, startTime: start };
      
      return request;
    });
    
    this.client.interceptors.response.use(
      response => {
        const start = (response.config as any).metadata?.startTime || Date.now();
        const duration = Date.now() - start;
        
        config.metrics.collect?.({
          type: 'response',
          url: response.config.url || '',
          duration,
          status: response.status,
          timestamp: Date.now()
        });
        
        return response;
      },
      error => {
        const start = (error.config as any).metadata?.startTime || Date.now();
        const duration = Date.now() - start;
        
        config.metrics.collect?.({
          type: 'error',
          url: error.config?.url || '',
          duration,
          error: error.message,
          timestamp: Date.now()
        });
        
        return Promise.reject(error);
      }
    );
  }
  
  // Helper methods...
  private shouldRetry(error: AxiosError): boolean {
    return !!error.config && 
           error.config as any).__retryCount !== undefined &&
           (error.response?.status === 503 || 
            error.code === 'ECONNABORTED' ||
            error.code === 'ETIMEDOUT');
  }
  
  private getAttemptCount(error: AxiosError): number {
    return ((error.config as any).__retryCount || 0) + 1;
  }
  
  private calculateDelay(attempt: number, baseDelay: number, backoff: string): number {
    if (backoff === 'exponential') {
      return baseDelay * Math.pow(2, attempt);
    }
    return baseDelay * attempt;
  }
  
  private wait(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
  
  private isCircuitOpen(
    url: string,
    failureCount: Map<string, number>,
    failureTimestamps: Map<string, number>,
    threshold: number,
    resetTimeout: number
  ): boolean {
    const count = failureCount.get(url) || 0;
    const lastFailure = failureTimestamps.get(url) || 0;
    
    if (count >= threshold && Date.now() - lastFailure < resetTimeout) {
      return true;
    }
    
    return false;
  }
  
  private recordSuccess(
    url: string,
    failureCount: Map<string, number>,
    failureTimestamps: Map<string, number>
  ): void {
    failureCount.delete(url);
    failureTimestamps.delete(url);
  }
  
  private recordFailure(
    url: string,
    failureCount: Map<string, number>,
    failureTimestamps: Map<string, number>
  ): void {
    const count = (failureCount.get(url) || 0) + 1;
    failureCount.set(url, count);
    failureTimestamps.set(url, Date.now());
  }
  
  public get(url: string, config?: AxiosRequestConfig) {
    return this.client.get(url, config);
  }
  
  public post(url: string, data?: any, config?: AxiosRequestConfig) {
    return this.client.post(url, data, config);
  }
  
  public put(url: string, data?: any, config?: AxiosRequestConfig) {
    return this.client.put(url, data, config);
  }
  
  public delete(url: string, config?: AxiosRequestConfig) {
    return this.client.delete(url, config);
  }
}
```

### Correlation ID & Distributed Tracing

```typescript
// libs/api-client/src/tracing.ts
import { v4 as uuidv4 } from 'uuid';

export interface TraceContext {
  correlationId: string;
  userId?: string;
  sessionId?: string;
  timestamp: number;
}

export class DistributedTracer {
  private static instance: DistributedTracer;
  private traceContext: TraceContext | null = null;
  
  private constructor() {}
  
  static getInstance(): DistributedTracer {
    if (!DistributedTracer.instance) {
      DistributedTracer.instance = new DistributedTracer();
    }
    return DistributedTracer.instance;
  }
  
  startTrace(userId?: string, sessionId?: string): TraceContext {
    this.traceContext = {
      correlationId: uuidv4(),
      userId,
      sessionId,
      timestamp: Date.now()
    };
    return this.traceContext;
  }
  
  getTraceContext(): TraceContext | null {
    return this.traceContext;
  }
  
  endTrace(): void {
    this.traceContext = null;
  }
  
  addHeaders(headers: Record<string, string>): Record<string, string> {
    const trace = this.traceContext;
    if (!trace) return headers;
    
    return {
      ...headers,
      'X-Correlation-ID': trace.correlationId,
      'X-Trace-ID': trace.correlationId,
      'X-User-ID': trace.userId || '',
      'X-Session-ID': trace.sessionId || ''
    };
  }
}
```

```typescript
// libs/api-client/src/index.ts
import { AdvancedApiClient } from './advancedApiClient';
import { DistributedTracer } from './tracing';

const tracer = DistributedTracer.getInstance();

export const apiClient = new AdvancedApiClient({
  baseUrl: import.meta.env.VITE_API_URL,
  timeout: 15000,
  interceptors: {
      request: [(config) => {
        const headers = tracer.addHeaders(config.headers || {});
        return { ...config, headers };
      }],
      response: [(response) => {
        // Log success metrics
        return response;
      }],
      error: [(error) => {
        // Log error metrics
        if (error.response?.status === 401) {
          window.location.href = '/login';
        }
        return Promise.reject(error);
      }]
    },
  retry: {
      maxAttempts: 3,
      delay: 1000,
      backoff: 'exponential'
    },
  circuitBreaker: {
      threshold: 5,
      resetTimeout: 60000
    },
  metrics: {
      enabled: true,
      collect: (data) => {
        // Send to monitoring service
        console.log('API Metric:', data);
      }
    }
});

export const productsApi = {
  getAll: () => apiClient.get<Product[]>('/api/products'),
  getById: (id: string) => apiClient.get<Product>(`/api/products/${id}`),
  create: (data: CreateProductDto) => 
    apiClient.post<Product>('/api/products', data),
  update: (id: string, data: UpdateProductDto) => 
    apiClient.put<Product>(`/api/products/${id}`, data),
  delete: (id: string) => apiClient.delete(`/api/products/${id}`)
};
```

---

## Advanced Prefetching & Data Loading

### SSR/SSG Integration với Next.js

```typescript
// apps/nextjs-app/src/pages/products/[id].tsx
import { GetServerSideProps } from 'next';
import { useQuery, dehydrate, QueryClient } from '@tanstack/react-query';
import { QueryClientProvider } from '@tanstack/react-query-next-experimental';
import { productsApi } from '@workspace/api-client';
import { ProductDetail } from '@workspace/ui';

export const getServerSideProps: GetServerSideProps = async (context) => {
  const { id } = context.params!;
  const queryClient = new QueryClient({
      defaultOptions: {
        queries: {
          staleTime: 60 * 1000, // 1 minute
        },
      },
    });
  
  // Prefetch data trên server
  await queryClient.prefetchQuery({
      queryKey: ['product', id],
      queryFn: () => productsApi.getById(id).then(r => r.data)
    });
  
  return {
      props: {
        dehydratedState: dehydrate(queryClient),
      },
    };
};

export default function ProductPage({ dehydratedState }: any) {
  const queryClient = new QueryClient();
  
  return (
    <QueryClientProvider client={queryClient} state={dehydratedState}>
      <ProductDetailPageContent />
    </QueryClientProvider>
  );
}

function ProductDetailPageContent() {
  const router = useRouter();
  const { id } = router.query as { id: string };
  
  const { data: product, isLoading } = useQuery({
      queryKey: ['product', id],
      queryFn: () => productsApi.getById(id).then(r => r.data)
    });
  
  if (isLoading) return <div>Loading...</div>;
  
  return <ProductDetail product={product} />;
}
```

### Streaming/Incremental Data

```typescript
// libs/data-layer/src/hooks/useInfiniteProducts.ts
import { useInfiniteQuery } from '@tanstack/react-query';
import { productsApi } from '@workspace/api-client';

export function useInfiniteProducts(limit: number = 20) {
  return useInfiniteQuery({
      queryKey: ['products', 'infinite'],
      queryFn: ({ pageParam = 0 }) => 
        productsApi.getAll({
          skip: pageParam * limit,
          limit
        }).then(r => r.data),
      initialPageParam: 0,
      getNextPageParam: (lastPage) => {
        if (lastPage.data?.length < limit) return undefined;
        return lastPage.pageParam + 1;
      },
    });
}
```

```tsx
// apps/my-app/src/pages/ProductListPage.tsx
import { useInfiniteProducts } from '@workspace/data-layer';

export function ProductListPage() {
  const {
      data,
      fetchNextPage,
      isFetchingNextPage,
      hasMore
    } = useInfiniteProducts(20);
  
  const products = data?.pages.flat() || [];
  
  return (
    <div>
      {products.map(product => (
        <div key={product.id}>
          <h2>{product.name}</h2>
          <p>${product.price}</p>
        </div>
      ))}
      
      {hasMore && (
        <button 
          onClick={() => fetchNextPage()}
          disabled={isFetchingNextPage}
        >
          {isFetchingNextPage ? 'Loading...' : 'Load More'}
        </button>
      )}
    </div>
  );
}
```

---

## Integration với Frameworks Lớn

### Next.js Integration

**Pattern: App Router với Nx**

```bash
# Tạo Next.js app
nx g @nx/next:app nextjs-app

# Tạo library cho shared types
nx g @nx/next:lib nextjs-shared

// apps/nextjs-app/next.config.js
const { withNx } = require('@nx/next/plugins/with-nx');

module.exports = withNx({
  nx: {
    svgr: false,
  },
});
```

```typescript
// libs/nextjs-shared/src/lib/products.ts
export interface Product {
  id: string;
  name: string;
  price: number;
}

export async function getProducts(): Promise<Product[]> {
  // Fetch products logic...
  return [];
}
```

```typescript
// apps/nextjs-app/app/products/page.tsx
import { getProducts } from '@workspace/nextjs-shared';

export const dynamic = 'force-dynamic';

export default async function ProductsPage() {
  const products = await getProducts();
  
  return (
    <div>
      <h1>Products</h1>
      {products.map(product => (
        <div key={product.id}>
          <h2>{product.name}</h2>
          <p>${product.price}</p>
        </div>
      ))}
    </div>
  );
}
```

### NestJS Integration

**Pattern: NestJS Module Architecture với Nx**

```bash
# Tạo NestJS app
nx g @nx/nest:app api

# Tạo libraries
nx g @nx/nest:lib users
nx g @nx/nest:lib products

// backend/apps/api/src/app.module.ts
import { Module } from '@nestjs/common';
import { UsersModule } from '@workspace/users';
import { ProductsModule } from '@workspace/products';

@Module({
  imports: [UsersModule, ProductsModule],
})
export class AppModule {}
```

```typescript
// backend/libs/users/src/users.module.ts
import { Module } from '@nestjs/common';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

### Microservices Integration

**Pattern 1: Service Discovery với Nx**

```typescript
// libs/service-discovery/src/registry.ts
export interface ServiceInstance {
  name: string;
  url: string;
  health: '/health';
  version: string;
}

export class ServiceRegistry {
  private services = new Map<string, ServiceInstance[]>();
  
  register(service: ServiceInstance) {
    const instances = this.services.get(service.name) || [];
    instances.push(service);
    this.services.set(service.name, instances);
  }
  
  discover(serviceName: string): ServiceInstance | null {
    const instances = this.services.get(serviceName);
    if (!instances || instances.length === 0) {
      return null;
    }
    
    // Load balancing: round-robin
    const index = Math.floor(Math.random() * instances.length);
    return instances[index];
  }
  
  getAllServices(): Map<string, ServiceInstance[]> {
    return this.services;
  }
}
```

**Pattern 2: API Gateway Pattern**

```typescript
// libs/api-gateway/src/gateway.ts
import { ServiceRegistry } from '@workspace/service-discovery';
import { AdvancedApiClient } from '@workspace/api-client';

export class ApiGateway {
  private registry: ServiceRegistry;
  private clients = new Map<string, AdvancedApiClient>();
  
  constructor() {
    this.registry = new ServiceRegistry();
    this.initializeClients();
  }
  
  private initializeClients() {
    const services = this.registry.getAllServices();
    
    services.forEach((instances, serviceName) => {
      const instance = instances[0];
      const client = new AdvancedApiClient({
        baseUrl: instance.url,
        interceptors: {
            request: [(config) => {
              // Add service routing headers
              return {
                ...config,
                headers: {
                  ...config.headers,
                  'X-Target-Service': serviceName,
                  'X-Service-Version': instance.version
                }
              };
            }]
          }
      });
      
      this.clients.set(serviceName, client);
    });
  }
  
  async callService(serviceName: string, endpoint: string, data?: any) {
    const client = this.clients.get(serviceName);
    if (!client) {
      throw new Error(`Service ${serviceName} not found`);
    }
    
    return client.post(endpoint, data);
  }
}
```

---

## Migration Strategies

### Monorepo sang Nx

**Phase 1: Planning**

```markdown
## Migration Plan

### Assessment
- Current structure: Multi-repo
- Number of projects: 15 (8 apps, 7 libs)
- Build tool: npm scripts
- CI/CD: GitHub Actions

### Goals
- Reduce build time by 70%
- Improve code sharing
- Centralize CI/CD
- Enable Nx Cloud caching

### Timeline
- Week 1: Setup and planning
- Week 2-3: Migrate core projects
- Week 4-5: Migrate remaining projects
- Week 6: Optimization and documentation
```

**Phase 2: Initial Setup**

```bash
# Tạo Nx workspace
npx create-nx-workspace@latest tapahonho --preset=apps

# Thêm plugins
nx add @nx/react @nx/dotnet @nx/nextjs @nx/nest

# Import existing projects
# Manually copy projects vào apps/ và libs/
```

**Phase 3: Progressive Migration**

```bash
# Migrate project theo thứ tự dependency
# 1. Migrate shared libs trước
nx g @nx/react:lib shared-ui --directory=../../path/to/old-shared-ui

# 2. Migrate libs phụ thuộc vào shared libs
nx g @nx/react:lib api-client --directory=../../path/to/old-api-client

# 3. Migrate apps
nx g @nx/react:app frontend --directory=../../path/to/old-frontend

# Update imports sang workspace aliases
# Find and replace: "../../../libs/shared-ui" -> "@workspace/shared-ui"
```

**Phase 4: Nx Configuration**

```json
// nx.json
{
  "defaultBase": "shared-ui",
  "tasksRunnerOptions": {
    "cacheableOperations": "default",
    "parallel": 5
  },
  "namedInputs": {
    "default": ["{projectRoot}/**/*", "sharedGlobals"]
  },
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"]
    }
  }
}
```

**Phase 5: CI/CD Migration**

```yaml
# .github/workflows/nx-ci.yml
name: Nx CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [18.x, 20.x]
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci
      - uses: nrwl/nx-set-shas@v3
      - run: npx nx affected --target=build --parallel=3
      - run: npx nx affected --target=test --parallel=3
      - uses: actions/upload-artifact@v3
        if: always()
        with:
          name: nx-logs
          path: logs/
```

### REST sang GraphQL Migration

**Pattern: Incremental Migration với Nx**

```bash
# Tạo GraphQL library
nx g @nx/react:lib graphql-client

// Phase 1: Chạy cả REST và GraphQL song song
// libs/graphql-client/src/adapters.ts
export class ApiAdapter {
  useRest: boolean;
  
  constructor(useRest: boolean = true) {
    this.useRest = useRest;
  }
  
  async getProducts(): Promise<Product[]> {
    if (this.useRest) {
      // Sử dụng REST API
      return fetch('/api/products').then(r => r.json());
    } else {
      // Sử dụng GraphQL API
      const query = `
        query GetProducts {
          products {
            id
            name
            price
          }
        }
      `;
      return fetch('/graphql', {
            method: 'POST',
            body: JSON.stringify({ query }),
            headers: { 'Content-Type': 'application/json' }
          }).then(r => r.json());
    }
  }
}

// Phase 2: Migrate từng feature sang GraphQL
// libs/graphql-client/src/features/products.ts
export const productsQuery = `
  query GetProducts {
    products {
      id
      name
      price
      stock
    }
  }
`;

export const productQuery = `
  query GetProduct($id: ID!) {
    product(id: $id) {
      id
      name
      price
      stock
      description
    }
  }
`;

export const createProductMutation = `
  mutation CreateProduct($input: CreateProductInput!) {
    createProduct(input: $input) {
      id
      name
      price
      stock
    }
  }
`;
```

---

## Production Monitoring & Metrics

### Nx Cloud Analytics

```bash
# Xem Nx Cloud dashboard
nx connect-to-nx-cloud

# Xem cache statistics
nx show project my-app --web

# Xem task execution metrics
nx report --web

# Export metrics
nx report --format=json > metrics.json
```

### Custom Monitoring Integration

**Pattern 1: Prometheus Metrics**

```typescript
// libs/monitoring/src/metrics.ts
import { Counter, Histogram, Registry } from 'prom-client';

export const apiMetrics = {
  requestDuration: new Histogram({
      name: 'api_request_duration_seconds',
      help: 'API request duration in seconds',
      labelNames: ['method', 'endpoint', 'status']
    }),
  requestTotal: new Counter({
      name: 'api_requests_total',
      help: 'Total number of API requests',
      labelNames: ['method', 'endpoint', 'status']
    }),
  errorTotal: new Counter({
      name: 'api_errors_total',
      help: 'Total number of API errors',
      labelNames: ['method', 'endpoint', 'error_type']
    })
};

export function recordRequest(
  method: string,
  endpoint: string,
  duration: number,
  status: number
) {
  apiMetrics.requestDuration
    .labels(method, endpoint, status.toString())
    .observe(duration);
  
  apiMetrics.requestTotal
    .labels(method, endpoint, status.toString())
    .inc();
}

export function recordError(
  method: string,
  endpoint: string,
  errorType: string
) {
  apiMetrics.errorTotal
    .labels(method, endpoint, errorType)
    .inc();
}
```

```typescript
// libs/api-client/src/advancedApiClient.ts
import { recordRequest, recordError } from '@workspace/monitoring';

export class AdvancedApiClient {
  // ... existing code ...
  
  private setupMetrics(config: ApiClientConfig) {
    if (!config.metrics?.enabled) return;
    
    this.client.interceptors.request.use(request => {
      request.metadata = { ...request.metadata, startTime: Date.now() };
      return request;
    });
    
    this.client.interceptors.response.use(
      response => {
        const start = (response.config as any).metadata?.startTime || Date.now();
        const duration = (Date.now() - start) / 1000;
        
        const method = response.config.method?.toUpperCase() || 'GET';
        const endpoint = this.getEndpointName(response.config.url || '');
        const status = response.status || 200;
        
        recordRequest(method, endpoint, duration, status);
        
        return response;
      },
      error => {
        const method = error.config?.method?.toUpperCase() || 'GET';
        const endpoint = this.getEndpointName(error.config?.url || '');
        const errorType = error.code || 'UNKNOWN_ERROR';
        
        recordError(method, endpoint, errorType);
        
        return Promise.reject(error);
      }
    );
  }
  
  private getEndpointName(url: string): string {
    return url.split('/')?.pop() || 'unknown';
  }
}
```

**Pattern 2: OpenTelemetry Integration**

```typescript
// libs/monitoring/src/tracing.ts
import { trace, Span } from '@opentelemetry/api';
import { registerInstrumentations } from '@opentelemetry/instrumentation';
import { ConsoleSpanExporter, SimpleSpanProcessor } from '@opentelemetry/sdk-trace';

const provider = {
  getTracer: () => {
      return trace('workspace', '1.0.0');
    },
};

registerInstrumentations({
  tracerProvider: provider,
});

export class OpenTelemetryTracer {
  static traceFunction<T>(
    name: string,
    fn: () => Promise<T>
  ): Promise<T> {
    const tracer = provider.getTracer();
    
    return tracer.startActiveSpan(name, async (span) => {
      try {
        const result = await fn();
        span.setStatus({ code: 1 });
        return result;
      } catch (error) {
        span.recordException(error as Error);
        span.setStatus({ code: 2, message: (error as Error).message });
        throw error;
      }
    });
  }
}

export function traceApiCall(method: string, endpoint: string) {
  return function<T extends any>(
    target: any,
    propertyKey: string,
    descriptor: TypedPropertyDescriptor<T>
  ): TypedPropertyDescriptor<T> | void {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function (...args: any[]) {
      return OpenTelemetryTracer.traceFunction(
        `${method} ${endpoint}`,
        () => originalMethod.apply(this, args)
      );
    };
    
    return descriptor;
  };
}
```

---

## Advanced Error Recovery & Resiliency

### Self-Healing CI Pipelines

**Pattern 1: Retry with Backoff**

```yaml
# .github/workflows/nx-self-healing.yml
name: Nx Self-Healing CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: 20.x
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Nx Build (with Retry)
        uses: nick-invision/retry@v2
        with:
          timeout_minutes: 30
          max_attempts: 3
          command: npx nx affected --target=build --parallel=3
          retry_wait_seconds: 30
      
      - name: Nx Test (with Retry)
        uses: nick-invision/retry@v2
        with:
          timeout_minutes: 30
          max_attempts: 3
          command: npx nx affected --target=test --parallel=3
          retry_wait_seconds: 30
      
      - name: Cache Miss Analysis
        if: failure()
        run: |
          npx nx report | grep "Cache Miss"
          echo "Cache hit rate: $(npx nx report | grep -oP 'Cache hit rate: [0-9.]*%')"
```

**Pattern 2: Flaky Test Detection**

```bash
# tools/flaky-test-detector/detect.sh
#!/bin/bash

# Chạy tests 3 lần
RUNS=3
FAILED_TESTS=""

for ((i=1; i<=RUNS; i++)); do
  echo "Run $i of $RUNS"
  
  FAILED=$(nx run test --passWithNoTests 2>&1 | grep "FAIL" || echo "")
  
  if [ -n "$FAILED" ]; then
    FAILED_TESTS="$FAILED_TESTS$FAILED\n"
  fi
done

# Phân tích kết quả
if [ -n "$FAILED_TESTS" ]; then
  echo "=== Flaky Tests Detected ==="
  echo "$FAILED_TESTS" | sort | uniq -c | sort -nr | grep -E "^\s*[3-$RUNS]" | head -10
  
  echo ""
  echo "Recommendation: Investigate tests that failed in all $RUNS runs"
else
  echo "No flaky tests detected"
fi
```

### Graceful Degradation

**Pattern 1: Circuit Breaker UI**

```tsx
// libs/ui/src/components/CircuitBreaker.tsx
import { ReactNode } from 'react';

export interface CircuitBreakerProps {
  isOpen: boolean;
  onRetry?: () => void;
  children: ReactNode;
}

export function CircuitBreaker({ isOpen, onRetry, children }: CircuitBreakerProps) {
  if (isOpen) {
    return (
      <div className="circuit-breaker">
        <h3>Service Unavailable</h3>
        <p>We're experiencing technical difficulties. Please try again later.</p>
        {onRetry && (
          <button onClick={onRetry}>
            Retry
          </button>
        )}
      </div>
    );
  }
  
  return <>{children}</>;
}
```

```tsx
// apps/my-app/src/pages/ProductListPage.tsx
import { CircuitBreaker } from '@workspace/ui';
import { useProducts } from '@workspace/data-layer';

export function ProductListPage() {
  const { data: products, isLoading, error, refetch } = useProducts();
  
  const isCircuitOpen = error?.response?.status === 503;
  
  return (
    <CircuitBreaker 
      isOpen={isCircuitOpen}
      onRetry={() => refetch()}
    >
      {isLoading && <div>Loading...</div>}
      {!isLoading && products && (
        <div>
          {products.map(product => (
            <div key={product.id}>
              <h2>{product.name}</h2>
              <p>${product.price}</p>
            </div>
          ))}
        </div>
      )}
    </CircuitBreaker>
  );
}
```

---

## Best Practices Summary

### 1. Architecture Principles

- **Bounded Contexts**: Organize code theo business domains
- **Shared Platform**: Platform team manages shared libs and infrastructure
- **Team Autonomy**: Domain teams have autonomy trong bounded contexts
- **API Gateway**: Centralized gateway cho microservices communication
- **Service Discovery**: Dynamic service discovery cho scalability

### 2. Performance Optimization

- **Distributed Caching**: Nx Cloud cho enterprise teams
- **Parallel Execution**: Maximize parallelism trong CI/CD
- **Prefetching Strategies**: Optimize data loading patterns
- **Circuit Breakers**: Prevent cascading failures
- **Monitoring & Metrics**: Continuous performance tracking

### 3. Reliability & Resiliency

- **Self-Healing CI**: Automatic retry và recovery
- **Graceful Degradation**: Circuit breakers và fallbacks
- **Distributed Tracing**: Correlation IDs cho debugging
- **Error Boundaries**: Graceful error handling trong UI
- **Retry with Backoff**: Exponential backoff cho retries

### 4. Team Enablement

- **Clear Responsibilities**: Define platform vs domain team roles
- **Documentation**: Comprehensive docs cho architecture và migrations
- **Training**: Nx training materials và onboarding
- **Governance**: ADRs và decision records
- **Support Channels**: Migration support và troubleshooting

### 5. Production Readiness

- **Monitoring**: Prometheus, OpenTelemetry integration
- **Alerting**: Threshold-based alerting
- **Dashboards**: Nx Cloud dashboards cho visibility
- **Log Aggregation**: Centralized logging với correlation IDs
- **Incident Response**: Playbooks cho common scenarios

### 6. Migration Strategies

- **Progressive Migration**: Migrate incrementally, không Big Bang
- **Dual Run**: Run old và new systems song song
- **Feature Flags**: Toggle features cho gradual rollout
- **Rollback Plans**: Ability to rollback quickly
- **Validation**: Measure và validate improvements

### 7. Security Considerations

- **Secret Management**: Secure secrets trong CI/CD
- **API Security**: Rate limiting, circuit breakers
- **Audit Logging**: Log API access với correlation IDs
- **Dependency Scanning**: Regular security scans
- **Compliance**: GDPR, SOC2 considerations

### 8. Cost Optimization

- **Resource Utilization**: Monitor Nx Cloud usage
- **Cache Efficiency**: Optimize cache hit rates
- **Build Optimization**: Reduce unnecessary rebuilds
- **CI/CD Optimization**: Minimize job duration
- **Infrastructure Scaling**: Right-size resources

### 9. Continuous Improvement

- **Metrics-Driven Decisions**: Use data để optimize
- **Regular Reviews**: Quarterly architecture reviews
- **Feedback Loops**: Collect và act on team feedback
- **Experimentation**: A/B test configurations
- **Knowledge Sharing**: Document learnings và patterns

### 10. Long-term Vision

- **Scalability**: Design cho growth (10x teams, 100x code)
- **Flexibility**: Enable quick pivots và experiments
- **Standardization**: Consistent patterns cross teams
- **Innovation**: Encourage experimentation với Nx features
- **Sustainability**: Maintainable codebase và processes
