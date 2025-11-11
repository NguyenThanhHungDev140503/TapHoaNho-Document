# Principal Refine Patterns - Enterprise Level

## Mục Lục

- [1. Micro-Frontend Architecture](#1-micro-frontend-architecture)
- [2. Multi-Tenancy at Scale](#2-multi-tenancy-at-scale)
- [3. Advanced Caching Strategies](#3-advanced-caching-strategies)
- [4. Performance Monitoring](#4-performance-monitoring)
- [5. SSR/SSG with Next.js](#5-ssrssg-with-nextjs)
- [6. Testing Strategies](#6-testing-strategies)
- [7. CI/CD Integration](#7-cicd-integration)
- [8. Production Optimization](#8-production-optimization)

---

## 1. Micro-Frontend Architecture

### Module Federation Setup

```typescript
// Example 1: Host application with Module Federation
// apps/host/webpack.config.js
const ModuleFederationPlugin = require("webpack/lib/container/ModuleFederationPlugin");

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: "host",
      remotes: {
        products: "products@http://localhost:3001/remoteEntry.js",
        orders: "orders@http://localhost:3002/remoteEntry.js",
        users: "users@http://localhost:3003/remoteEntry.js",
      },
      shared: {
        react: { singleton: true, requiredVersion: "^18.0.0" },
        "react-dom": { singleton: true, requiredVersion: "^18.0.0" },
        "@refinedev/core": { singleton: true },
        "@refinedev/antd": { singleton: true },
      },
    }),
  ],
};

// apps/host/src/App.tsx
import { Refine } from "@refinedev/core";
import { lazy, Suspense } from "react";

// Lazy load remote modules
const ProductsModule = lazy(() => import("products/Module"));
const OrdersModule = lazy(() => import("orders/Module"));
const UsersModule = lazy(() => import("users/Module"));

function App() {
  return (
    <Refine
      dataProvider={dataProvider(API_URL)}
      authProvider={authProvider}
      resources={[
        {
          name: "products",
          list: "/products",
          create: "/products/create",
          edit: "/products/edit/:id",
        },
        {
          name: "orders",
          list: "/orders",
          show: "/orders/show/:id",
        },
        {
          name: "users",
          list: "/users",
          create: "/users/create",
          edit: "/users/edit/:id",
        },
      ]}
    >
      <Routes>
        <Route
          path="/products/*"
          element={
            <Suspense fallback={<div>Loading Products...</div>}>
              <ProductsModule />
            </Suspense>
          }
        />
        <Route
          path="/orders/*"
          element={
            <Suspense fallback={<div>Loading Orders...</div>}>
              <OrdersModule />
            </Suspense>
          }
        />
        <Route
          path="/users/*"
          element={
            <Suspense fallback={<div>Loading Users...</div>}>
              <UsersModule />
            </Suspense>
          }
        />
      </Routes>
    </Refine>
  );
}

// apps/products/webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: "products",
      filename: "remoteEntry.js",
      exposes: {
        "./Module": "./src/Module",
      },
      shared: {
        react: { singleton: true },
        "react-dom": { singleton: true },
        "@refinedev/core": { singleton: true },
      },
    }),
  ],
};

// apps/products/src/Module.tsx
import { Route, Routes } from "react-router-dom";
import { ProductList, ProductCreate, ProductEdit } from "./pages";

export default function ProductsModule() {
  return (
    <Routes>
      <Route index element={<ProductList />} />
      <Route path="create" element={<ProductCreate />} />
      <Route path="edit/:id" element={<ProductEdit />} />
    </Routes>
  );
}
```

### Shared State Across Micro-Frontends

```typescript
// Example 2: Cross-app state management
// packages/shared-state/src/index.ts
import { create } from "zustand";
import { persist } from "zustand/middleware";

interface SharedState {
  user: User | null;
  tenant: Tenant | null;
  theme: "light" | "dark";
  setUser: (user: User | null) => void;
  setTenant: (tenant: Tenant | null) => void;
  setTheme: (theme: "light" | "dark") => void;
}

export const useSharedState = create<SharedState>()(
  persist(
    (set) => ({
      user: null,
      tenant: null,
      theme: "light",
      setUser: (user) => set({ user }),
      setTenant: (tenant) => set({ tenant }),
      setTheme: (theme) => set({ theme }),
    }),
    {
      name: "shared-state",
    }
  )
);

// Usage in any micro-frontend
import { useSharedState } from "@shared/state";

export const Header = () => {
  const user = useSharedState((state) => state.user);
  const theme = useSharedState((state) => state.theme);
  const setTheme = useSharedState((state) => state.setTheme);
  
  return (
    <header>
      <div>Welcome, {user?.name}</div>
      <button onClick={() => setTheme(theme === "light" ? "dark" : "light")}>
        Toggle Theme
      </button>
    </header>
  );
};
```

---

## 2. Multi-Tenancy at Scale

### Tenant Isolation Strategy

```typescript
// Example 3: Complete multi-tenant architecture
// src/providers/tenantDataProvider.ts
import { DataProvider } from "@refinedev/core";
import axios, { AxiosInstance } from "axios";

interface TenantContext {
  tenantId: string;
  tenantSlug: string;
  database: string;
}

export const createTenantDataProvider = (
  apiUrl: string,
  getTenantContext: () => TenantContext | null
): DataProvider => {
  const axiosInstance = axios.create({
    baseURL: apiUrl,
  });
  
  // Add tenant headers to all requests
  axiosInstance.interceptors.request.use((config) => {
    const context = getTenantContext();
    
    if (context) {
      config.headers["X-Tenant-ID"] = context.tenantId;
      config.headers["X-Tenant-Slug"] = context.tenantSlug;
      config.headers["X-Tenant-Database"] = context.database;
    }
    
    const token = localStorage.getItem("token");
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    
    return config;
  });
  
  return {
    getList: async ({ resource, pagination, filters, sorters }) => {
      const context = getTenantContext();
      
      if (!context) {
        throw new Error("Tenant context not found");
      }
      
      const { current = 1, pageSize = 10 } = pagination ?? {};
      
      const query = {
        _start: (current - 1) * pageSize,
        _end: current * pageSize,
      };
      
      // Add tenant filter
      const tenantFilters = [
        ...(filters || []),
        {
          field: "tenantId",
          operator: "eq" as const,
          value: context.tenantId,
        },
      ];
      
      const { data, headers } = await axiosInstance.get(`/${resource}`, {
        params: { ...query, filters: tenantFilters },
      });
      
      return {
        data,
        total: Number(headers["x-total-count"]),
      };
    },
    
    create: async ({ resource, variables }) => {
      const context = getTenantContext();
      
      if (!context) {
        throw new Error("Tenant context not found");
      }
      
      const { data } = await axiosInstance.post(`/${resource}`, {
        ...variables,
        tenantId: context.tenantId,
      });
      
      return { data };
    },
    
    // Similar for other methods...
    getOne: async ({ resource, id }) => {
      const { data } = await axiosInstance.get(`/${resource}/${id}`);
      return { data };
    },
    
    update: async ({ resource, id, variables }) => {
      const { data } = await axiosInstance.patch(`/${resource}/${id}`, variables);
      return { data };
    },
    
    deleteOne: async ({ resource, id }) => {
      const { data } = await axiosInstance.delete(`/${resource}/${id}`);
      return { data };
    },
    
    getApiUrl: () => apiUrl,
  };
};

// Tenant context provider
import { createContext, useContext, useState, useEffect } from "react";

const TenantContext = createContext<TenantContext | null>(null);

export const TenantProvider = ({ children }: { children: React.ReactNode }) => {
  const [tenant, setTenant] = useState<TenantContext | null>(null);

  useEffect(() => {
    // Get tenant from subdomain or path
    const hostname = window.location.hostname;
    const subdomain = hostname.split(".")[0];

    // Fetch tenant configuration
    fetch(`/api/tenants/${subdomain}`)
      .then((res) => res.json())
      .then((data) => {
        setTenant({
          tenantId: data.id,
          tenantSlug: data.slug,
          database: data.database,
        });
      });
  }, []);

  if (!tenant) {
    return <div>Loading tenant...</div>;
  }

  return (
    <TenantContext.Provider value={tenant}>
      {children}
    </TenantContext.Provider>
  );
};

export const useTenant = () => {
  const context = useContext(TenantContext);

  if (!context) {
    throw new Error("useTenant must be used within TenantProvider");
  }

  return context;
};

// Usage in App
import { TenantProvider, useTenant } from "./providers/TenantProvider";

function App() {
  return (
    <TenantProvider>
      <RefineApp />
    </TenantProvider>
  );
}

function RefineApp() {
  const tenant = useTenant();

  return (
    <Refine
      dataProvider={createTenantDataProvider(API_URL, () => tenant)}
      // ...
    />
  );
}
```

### Database-per-Tenant Strategy

```typescript
// Example 4: Dynamic database connection per tenant
// backend/src/middleware/tenantDatabase.ts
import { Request, Response, NextFunction } from "express";
import { createConnection, Connection } from "typeorm";

const connections = new Map<string, Connection>();

export const tenantDatabaseMiddleware = async (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  const tenantId = req.headers["x-tenant-id"] as string;

  if (!tenantId) {
    return res.status(400).json({ error: "Tenant ID required" });
  }

  try {
    // Get or create connection for tenant
    let connection = connections.get(tenantId);

    if (!connection || !connection.isConnected) {
      // Fetch tenant database config
      const tenantConfig = await getTenantDatabaseConfig(tenantId);

      connection = await createConnection({
        type: "postgres",
        host: tenantConfig.host,
        port: tenantConfig.port,
        username: tenantConfig.username,
        password: tenantConfig.password,
        database: tenantConfig.database,
        entities: [__dirname + "/../entities/*.ts"],
        synchronize: false,
        name: tenantId,
      });

      connections.set(tenantId, connection);
    }

    // Attach connection to request
    req.tenantConnection = connection;

    next();
  } catch (error) {
    console.error("Database connection error:", error);
    res.status(500).json({ error: "Database connection failed" });
  }
};

// Usage in routes
app.use("/api", tenantDatabaseMiddleware);

app.get("/api/products", async (req, res) => {
  const connection = req.tenantConnection;
  const productRepository = connection.getRepository(Product);

  const products = await productRepository.find();

  res.json(products);
});
```

---

## 3. Advanced Caching Strategies

### Multi-Layer Caching

```typescript
// Example 5: Redis + React Query caching
import { QueryClient } from "@tanstack/react-query";
import { createSyncStoragePersister } from "@tanstack/query-sync-storage-persister";
import { PersistQueryClientProvider } from "@tanstack/react-query-persist-client";
import Redis from "ioredis";

// Redis client for server-side caching
const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: Number(process.env.REDIS_PORT),
  password: process.env.REDIS_PASSWORD,
});

// Custom data provider with Redis caching
export const cachedDataProvider = (baseProvider: DataProvider): DataProvider => {
  return {
    ...baseProvider,

    getList: async (params) => {
      const cacheKey = `list:${params.resource}:${JSON.stringify(params)}`;

      // Try to get from Redis cache
      const cached = await redis.get(cacheKey);

      if (cached) {
        return JSON.parse(cached);
      }

      // Fetch from API
      const result = await baseProvider.getList(params);

      // Cache for 5 minutes
      await redis.setex(cacheKey, 300, JSON.stringify(result));

      return result;
    },

    getOne: async (params) => {
      const cacheKey = `one:${params.resource}:${params.id}`;

      const cached = await redis.get(cacheKey);

      if (cached) {
        return JSON.parse(cached);
      }

      const result = await baseProvider.getOne(params);

      // Cache for 10 minutes
      await redis.setex(cacheKey, 600, JSON.stringify(result));

      return result;
    },

    create: async (params) => {
      const result = await baseProvider.create(params);

      // Invalidate list cache
      const pattern = `list:${params.resource}:*`;
      const keys = await redis.keys(pattern);

      if (keys.length > 0) {
        await redis.del(...keys);
      }

      return result;
    },

    update: async (params) => {
      const result = await baseProvider.update(params);

      // Invalidate specific item and list caches
      await redis.del(`one:${params.resource}:${params.id}`);

      const pattern = `list:${params.resource}:*`;
      const keys = await redis.keys(pattern);

      if (keys.length > 0) {
        await redis.del(...keys);
      }

      return result;
    },

    deleteOne: async (params) => {
      const result = await baseProvider.deleteOne(params);

      // Invalidate caches
      await redis.del(`one:${params.resource}:${params.id}`);

      const pattern = `list:${params.resource}:*`;
      const keys = await redis.keys(pattern);

      if (keys.length > 0) {
        await redis.del(...keys);
      }

      return result;
    },
  };
};

// Client-side persistent cache
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5 minutes
      gcTime: 1000 * 60 * 60 * 24, // 24 hours
    },
  },
});

const persister = createSyncStoragePersister({
  storage: window.localStorage,
});

// Usage in App
function App() {
  return (
    <PersistQueryClientProvider
      client={queryClient}
      persistOptions={{ persister }}
    >
      <Refine
        dataProvider={cachedDataProvider(baseDataProvider)}
        // ...
      />
    </PersistQueryClientProvider>
  );
}
```

### Cache Invalidation Strategy

```typescript
// Example 6: Smart cache invalidation
import { useInvalidate } from "@refinedev/core";
import { useQueryClient } from "@tanstack/react-query";

export const useCacheInvalidation = () => {
  const invalidate = useInvalidate();
  const queryClient = useQueryClient();

  const invalidateResource = (resource: string, options?: {
    id?: string;
    invalidateList?: boolean;
    invalidateDetail?: boolean;
    invalidateRelated?: string[];
  }) => {
    const {
      id,
      invalidateList = true,
      invalidateDetail = true,
      invalidateRelated = [],
    } = options || {};

    // Invalidate list
    if (invalidateList) {
      invalidate({
        resource,
        invalidates: ["list"],
      });
    }

    // Invalidate specific detail
    if (invalidateDetail && id) {
      invalidate({
        resource,
        invalidates: ["detail"],
        id,
      });
    }

    // Invalidate related resources
    invalidateRelated.forEach((relatedResource) => {
      invalidate({
        resource: relatedResource,
        invalidates: ["list"],
      });
    });

    // Clear Redis cache (via API call)
    fetch("/api/cache/invalidate", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ resource, id }),
    });
  };

  return { invalidateResource };
};

// Usage
export const ProductActions = ({ id }: { id: string }) => {
  const { mutate: deleteProduct } = useDelete();
  const { invalidateResource } = useCacheInvalidation();

  const handleDelete = () => {
    deleteProduct(
      {
        resource: "products",
        id,
      },
      {
        onSuccess: () => {
          // Invalidate products and related categories
          invalidateResource("products", {
            id,
            invalidateList: true,
            invalidateDetail: true,
            invalidateRelated: ["categories", "orders"],
          });
        },
      }
    );
  };

  return <button onClick={handleDelete}>Delete</button>;
};
```

---

## 4. Performance Monitoring

### Custom Performance Metrics

```typescript
// Example 7: Performance monitoring with Web Vitals
import { onCLS, onFID, onFCP, onLCP, onTTFB } from "web-vitals";
import { DataProvider } from "@refinedev/core";

// Track Web Vitals
export const initPerformanceMonitoring = () => {
  const sendToAnalytics = (metric: any) => {
    // Send to your analytics service
    fetch("/api/analytics/vitals", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(metric),
    });
  };

  onCLS(sendToAnalytics);
  onFID(sendToAnalytics);
  onFCP(sendToAnalytics);
  onLCP(sendToAnalytics);
  onTTFB(sendToAnalytics);
};

// Data provider with performance tracking
export const monitoredDataProvider = (
  baseProvider: DataProvider
): DataProvider => {
  const trackOperation = async <T,>(
    operation: string,
    resource: string,
    fn: () => Promise<T>
  ): Promise<T> => {
    const startTime = performance.now();

    try {
      const result = await fn();
      const duration = performance.now() - startTime;

      // Send metrics
      fetch("/api/analytics/operations", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          operation,
          resource,
          duration,
          success: true,
          timestamp: new Date().toISOString(),
        }),
      });

      return result;
    } catch (error) {
      const duration = performance.now() - startTime;

      // Track error
      fetch("/api/analytics/operations", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          operation,
          resource,
          duration,
          success: false,
          error: error.message,
          timestamp: new Date().toISOString(),
        }),
      });

      throw error;
    }
  };

  return {
    getList: (params) =>
      trackOperation("getList", params.resource, () =>
        baseProvider.getList(params)
      ),

    getOne: (params) =>
      trackOperation("getOne", params.resource, () =>
        baseProvider.getOne(params)
      ),

    create: (params) =>
      trackOperation("create", params.resource, () =>
        baseProvider.create(params)
      ),

    update: (params) =>
      trackOperation("update", params.resource, () =>
        baseProvider.update(params)
      ),

    deleteOne: (params) =>
      trackOperation("deleteOne", params.resource, () =>
        baseProvider.deleteOne(params)
      ),

    getApiUrl: () => baseProvider.getApiUrl(),
  };
};

// Usage in App
import { useEffect } from "react";

function App() {
  useEffect(() => {
    initPerformanceMonitoring();
  }, []);

  return (
    <Refine
      dataProvider={monitoredDataProvider(baseDataProvider)}
      // ...
    />
  );
}
```

### Error Tracking Integration

```typescript
// Example 8: Sentry integration
import * as Sentry from "@sentry/react";
import { BrowserTracing } from "@sentry/tracing";

Sentry.init({
  dsn: process.env.REACT_APP_SENTRY_DSN,
  integrations: [new BrowserTracing()],
  tracesSampleRate: 1.0,
  environment: process.env.NODE_ENV,
});

// Error boundary for Refine
export const RefineErrorBoundary = ({ children }: { children: React.ReactNode }) => {
  return (
    <Sentry.ErrorBoundary
      fallback={({ error, resetError }) => (
        <div>
          <h1>Something went wrong</h1>
          <p>{error.message}</p>
          <button onClick={resetError}>Try again</button>
        </div>
      )}
      showDialog
    >
      {children}
    </Sentry.ErrorBoundary>
  );
};

// Track custom events
import { useEffect } from "react";
import { useResource } from "@refinedev/core";

export const useResourceTracking = () => {
  const { resource } = useResource();

  useEffect(() => {
    if (resource) {
      Sentry.addBreadcrumb({
        category: "navigation",
        message: `Navigated to ${resource.name}`,
        level: "info",
      });
    }
  }, [resource]);
};
```

---

## 5. SSR/SSG with Next.js

### Server-Side Rendering

```typescript
// Example 9: Refine with Next.js App Router
// app/layout.tsx
import { Refine } from "@refinedev/core";
import routerProvider from "@refinedev/nextjs-router/app";
import dataProvider from "@refinedev/simple-rest";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <Refine
          routerProvider={routerProvider}
          dataProvider={dataProvider(process.env.NEXT_PUBLIC_API_URL!)}
          resources={[
            {
              name: "products",
              list: "/products",
              show: "/products/:id",
              create: "/products/create",
              edit: "/products/:id/edit",
            },
          ]}
          options={{
            syncWithLocation: true,
            warnWhenUnsavedChanges: true,
          }}
        >
          {children}
        </Refine>
      </body>
    </html>
  );
}

// app/products/page.tsx
import { Suspense } from "react";
import dataProvider from "@refinedev/simple-rest";

const API_URL = process.env.NEXT_PUBLIC_API_URL!;

async function getProducts() {
  const { data } = await dataProvider(API_URL).getList({
    resource: "products",
    pagination: { current: 1, pageSize: 10 },
  });

  return data;
}

export default async function ProductsPage() {
  const products = await getProducts();

  return (
    <div>
      <h1>Products</h1>
      <Suspense fallback={<div>Loading...</div>}>
        <ProductList initialData={products} />
      </Suspense>
    </div>
  );
}

// components/ProductList.tsx
"use client";

import { useList } from "@refinedev/core";

export function ProductList({ initialData }: { initialData: any[] }) {
  const { data } = useList({
    resource: "products",
    queryOptions: {
      initialData: {
        data: initialData,
        total: initialData.length,
      },
    },
  });

  return (
    <ul>
      {data?.data.map((product) => (
        <li key={product.id}>{product.name}</li>
      ))}
    </ul>
  );
}
```

### Static Site Generation

```typescript
// Example 10: ISR with Next.js
// app/products/[id]/page.tsx
import dataProvider from "@refinedev/simple-rest";

const API_URL = process.env.NEXT_PUBLIC_API_URL!;

// Generate static params
export async function generateStaticParams() {
  const { data } = await dataProvider(API_URL).getList({
    resource: "products",
    pagination: { current: 1, pageSize: 100 },
  });

  return data.map((product) => ({
    id: product.id.toString(),
  }));
}

// Fetch product data
async function getProduct(id: string) {
  const { data } = await dataProvider(API_URL).getOne({
    resource: "products",
    id,
  });

  return data;
}

export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await getProduct(params.id);

  return (
    <div>
      <h1>{product.name}</h1>
      <p>Price: ${product.price}</p>
      <p>{product.description}</p>
    </div>
  );
}

// Revalidate every 60 seconds
export const revalidate = 60;
```

---

## 6. Testing Strategies

### Integration Testing

```typescript
// Example 11: Testing with React Testing Library
import { render, screen, waitFor } from "@testing-library/react";
import { Refine } from "@refinedev/core";
import { ProductList } from "./ProductList";

const mockDataProvider = {
  getList: jest.fn(() =>
    Promise.resolve({
      data: [
        { id: "1", name: "Product 1", price: 100 },
        { id: "2", name: "Product 2", price: 200 },
      ],
      total: 2,
    })
  ),
  getOne: jest.fn(),
  create: jest.fn(),
  update: jest.fn(),
  deleteOne: jest.fn(),
  getApiUrl: () => "http://localhost:3000",
};

const TestWrapper = ({ children }: { children: React.ReactNode }) => (
  <Refine
    dataProvider={mockDataProvider}
    resources={[{ name: "products" }]}
  >
    {children}
  </Refine>
);

describe("ProductList", () => {
  it("renders product list", async () => {
    render(
      <TestWrapper>
        <ProductList />
      </TestWrapper>
    );

    await waitFor(() => {
      expect(screen.getByText("Product 1")).toBeInTheDocument();
      expect(screen.getByText("Product 2")).toBeInTheDocument();
    });

    expect(mockDataProvider.getList).toHaveBeenCalledWith({
      resource: "products",
      pagination: expect.any(Object),
    });
  });

  it("handles loading state", () => {
    mockDataProvider.getList.mockImplementation(
      () => new Promise(() => {}) // Never resolves
    );

    render(
      <TestWrapper>
        <ProductList />
      </TestWrapper>
    );

    expect(screen.getByText("Loading...")).toBeInTheDocument();
  });

  it("handles error state", async () => {
    mockDataProvider.getList.mockRejectedValue(new Error("API Error"));

    render(
      <TestWrapper>
        <ProductList />
      </TestWrapper>
    );

    await waitFor(() => {
      expect(screen.getByText(/error/i)).toBeInTheDocument();
    });
  });
});
```

### E2E Testing with Playwright

```typescript
// Example 12: E2E tests
// tests/products.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Product Management", () => {
  test.beforeEach(async ({ page }) => {
    await page.goto("http://localhost:3000/products");
  });

  test("should display product list", async ({ page }) => {
    await expect(page.locator("h1")).toContainText("Products");

    const productRows = page.locator("table tbody tr");
    await expect(productRows).toHaveCount(10);
  });

  test("should create new product", async ({ page }) => {
    await page.click('button:has-text("Create")');

    await page.fill('input[name="name"]', "Test Product");
    await page.fill('input[name="price"]', "99.99");
    await page.selectOption('select[name="category"]', "electronics");

    await page.click('button[type="submit"]');

    await expect(page.locator(".success-message")).toBeVisible();
    await expect(page.locator("table")).toContainText("Test Product");
  });

  test("should edit product", async ({ page }) => {
    await page.click('table tbody tr:first-child button:has-text("Edit")');

    await page.fill('input[name="price"]', "149.99");
    await page.click('button[type="submit"]');

    await expect(page.locator(".success-message")).toBeVisible();
  });

  test("should delete product", async ({ page }) => {
    page.on("dialog", (dialog) => dialog.accept());

    await page.click('table tbody tr:first-child button:has-text("Delete")');

    await expect(page.locator(".success-message")).toBeVisible();
  });

  test("should filter products", async ({ page }) => {
    await page.fill('input[placeholder="Search..."]', "laptop");

    await page.waitForTimeout(500); // Debounce

    const productRows = page.locator("table tbody tr");
    await expect(productRows).toHaveCount(3);
  });

  test("should sort products", async ({ page }) => {
    await page.click('th:has-text("Price")');

    const firstPrice = await page.locator("table tbody tr:first-child td:nth-child(3)").textContent();
    expect(Number(firstPrice?.replace("$", ""))).toBeLessThan(100);
  });
});
```

---

## 7. CI/CD Integration

### GitHub Actions Workflow

```yaml
# Example 13: CI/CD pipeline
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run type check
        run: npm run type-check

      - name: Run unit tests
        run: npm run test:unit

      - name: Run integration tests
        run: npm run test:integration

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json

  e2e:
    runs-on: ubuntu-latest
    needs: test

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright
        run: npx playwright install --with-deps

      - name: Build application
        run: npm run build

      - name: Start application
        run: npm run start &

      - name: Wait for application
        run: npx wait-on http://localhost:3000

      - name: Run E2E tests
        run: npm run test:e2e

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: playwright-report/

  build:
    runs-on: ubuntu-latest
    needs: [test, e2e]
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Build application
        run: npm run build
        env:
          NEXT_PUBLIC_API_URL: ${{ secrets.API_URL }}

      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Push to registry
        run: |
          echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
          docker tag myapp:${{ github.sha }} myregistry/myapp:latest
          docker push myregistry/myapp:latest

  deploy:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Deploy to production
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USERNAME }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            cd /app
            docker-compose pull
            docker-compose up -d
            docker system prune -f
```

---

## 8. Production Optimization

### Code Splitting Strategy

```typescript
// Example 14: Advanced code splitting
// src/App.tsx
import { lazy, Suspense } from "react";
import { Refine } from "@refinedev/core";

// Lazy load pages
const ProductList = lazy(() => import("./pages/products/list"));
const ProductCreate = lazy(() => import("./pages/products/create"));
const ProductEdit = lazy(() => import("./pages/products/edit"));
const ProductShow = lazy(() => import("./pages/products/show"));

// Lazy load UI framework
const AntdLayout = lazy(() =>
  import("@refinedev/antd").then((module) => ({
    default: module.ThemedLayoutV2,
  }))
);

// Loading component
const PageLoader = () => (
  <div style={{ padding: "20px", textAlign: "center" }}>
    <div>Loading...</div>
  </div>
);

function App() {
  return (
    <Refine
      dataProvider={dataProvider(API_URL)}
      resources={[
        {
          name: "products",
          list: "/products",
          create: "/products/create",
          edit: "/products/edit/:id",
          show: "/products/show/:id",
        },
      ]}
    >
      <Suspense fallback={<PageLoader />}>
        <AntdLayout>
          <Routes>
            <Route path="/products">
              <Route
                index
                element={
                  <Suspense fallback={<PageLoader />}>
                    <ProductList />
                  </Suspense>
                }
              />
              <Route
                path="create"
                element={
                  <Suspense fallback={<PageLoader />}>
                    <ProductCreate />
                  </Suspense>
                }
              />
              <Route
                path="edit/:id"
                element={
                  <Suspense fallback={<PageLoader />}>
                    <ProductEdit />
                  </Suspense>
                }
              />
              <Route
                path="show/:id"
                element={
                  <Suspense fallback={<PageLoader />}>
                    <ProductShow />
                  </Suspense>
                }
              />
            </Route>
          </Routes>
        </AntdLayout>
      </Suspense>
    </Refine>
  );
}
```

### Bundle Optimization

```javascript
// Example 15: Webpack optimization
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: "all",
      cacheGroups: {
        // Vendor chunks
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name(module) {
            const packageName = module.context.match(
              /[\\/]node_modules[\\/](.*?)([\\/]|$)/
            )[1];
            return `vendor.${packageName.replace("@", "")}`;
          },
          priority: 10,
        },

        // Refine core
        refineCore: {
          test: /[\\/]node_modules[\\/]@refinedev[\\/]core/,
          name: "refine.core",
          priority: 20,
        },

        // UI framework
        antd: {
          test: /[\\/]node_modules[\\/](antd|@ant-design)/,
          name: "ui.antd",
          priority: 20,
        },

        // Common chunks
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true,
        },
      },
    },

    // Runtime chunk
    runtimeChunk: "single",

    // Minimize
    minimize: true,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            drop_console: true,
          },
        },
      }),
    ],
  },

  // Performance hints
  performance: {
    maxEntrypointSize: 512000,
    maxAssetSize: 512000,
  },
};
```

---

## Best Practices - Principal Level

### 1. Architecture Decisions

```typescript
// ✅ GOOD: Modular architecture
apps/
  admin/          # Admin dashboard
  customer/       # Customer portal
  mobile/         # Mobile app
packages/
  shared/         # Shared utilities
  ui/             # Shared UI components
  data-provider/  # Custom data providers
  auth/           # Authentication logic

// ❌ BAD: Monolithic structure
src/
  everything-in-one-place/
```

### 2. Security Best Practices

```typescript
// ✅ GOOD: Secure configuration
// - Use environment variables
// - Implement CSP headers
// - Enable HTTPS only
// - Sanitize user inputs
// - Implement rate limiting
// - Use secure cookies

// ❌ BAD: Hardcoded secrets
const API_KEY = "sk_live_123456789";
```

### 3. Monitoring and Observability

```typescript
// ✅ GOOD: Comprehensive monitoring
// - Application performance monitoring (APM)
// - Error tracking (Sentry)
// - User analytics
// - Server metrics
// - Database performance
// - Cache hit rates

// ❌ BAD: No monitoring
```

---

## Tài Liệu Tham Khảo

### Official Documentation
- [Refine Enterprise Guide](https://refine.dev/docs/enterprise/)
- [Next.js Integration](https://refine.dev/docs/packages/list-of-packages/#nextjs)
- [Performance Best Practices](https://refine.dev/docs/guides-concepts/performance/)

### Tools & Services
- **Monitoring**: Sentry, DataDog, New Relic
- **Caching**: Redis, Memcached
- **Testing**: Jest, Playwright, Cypress
- **CI/CD**: GitHub Actions, GitLab CI, Jenkins

---

**Tài liệu này bao gồm Principal level patterns cho enterprise applications. Kết hợp với Junior, Middle, và Senior levels để có kiến thức toàn diện về Refine.**
