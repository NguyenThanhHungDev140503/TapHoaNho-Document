# Nx - Hướng Dẫn Toàn Diện

**Table of Contents**
- [Giới Thiệu Tổng Quan](#giới-thiệu-tổng-quan)
- [Junior Level - Cơ Bản](#junior-level---cơ-bản)
- [Middle Level - Trung Cấp](#middle-level---trung-cấp)
- [Senior Level - Nâng Cao](#senior-level---nâng-cao)
- [Tài Liệu Tham Khảo](#tài-liệu-tham-khảo)

---

## Giới Thiệu Tổng Quan

### Nx là gì?

**Nx** là một build system và monorepo toolkit giúp developers:
- Quản lý nhiều applications và libraries trong một repository
- Build và test code nhanh hơn với caching
- Tự động hóa tasks với code generation
- Tối ưu hóa CI/CD pipelines với distributed execution

### Tại sao sử dụng Nx?

1. **Caching**: 90%+ cache hit rate cho unchanged code
2. **Parallel Execution**: Chạy tasks song song trên nhiều cores/agents
3. **Smart Orchestration**: Auto-detect dependencies và chạy theo đúng thứ tự
4. **Code Generation**: Generators nhanh chóng tạo boilerplate code
5. **Multi-language**: Hỗ trợ React, Angular, .NET, Node.js, Go, Rust, Python, v.v.

### Các Khái Niệu Cốt Lõi

| Khái niệm | Mô tả | Ví dụ |
|---------|-------|--------|
| **Workspace** | Root của monorepo | `TapHoaNho/` với `nx.json` |
| **Project** | App hoặc lib có `project.json` | `apps/frontend/`, `libs/ui/` |
| **App** | Deployable application | React web app, .NET WebAPI |
| **Lib** | Shared library | UI components, utilities |
| **Target** | Named operation | `build`, `test`, `lint`, `serve` |
| **Executor** | Code thực thi target | `@nx/vite:build`, `@nx/dotnet:build` |
| **Generator** | Code tạo boilerplate | `@nx/react:lib`, `@nx/react:app` |

---

## Junior Level - Cơ Bản

### Cài Đặt & Cấu Hình

#### 1. Tạo Workspace Mới

```bash
# Tạo React monorepo mới
npx create-nx-workspace@latest --preset=react-monorepo

# Tạo workspace với nhiều preset (apps + libs)
npx create-nx-workspace@latest --preset=apps

# Tạo workspace với bundler tùy chỉnh
npx create-nx-workspace@latest --preset=react-monorepo --bundler=vite
```

#### 2. Thêm Nx vào Existing Monorepo

```bash
# Nếu bạn đã có npm/yarn/pnpm workspace
cd your-existing-monorepo

# Thêm React plugin
nx add @nx/react

# Thêm .NET plugin
nx add @nx/dotnet

# Thêm Vite plugin
nx add @nx/vite
```

#### 3. Workspace Structure

```
TapHoaNho/
├── apps/                    # Applications (deployable)
│   ├── frontend/             # React app
│   └── webapi/              # .NET WebAPI
├── libs/                    # Shared libraries
│   ├── ui/                  # Shared UI components
│   └── utils/                # Utilities
├── nx.json                  # Nx workspace configuration
├── package.json              # Root package.json (workspaces)
└── tsconfig.base.json        # Base TypeScript config
```

### API/Hook Cơ Bản

#### 1. nx run - Chạy Tasks

```bash
# Chạy task trên project
nx run my-app:build

# Chạy nhiều targets
nx run my-app --target=build --target=test

# Chạy task trên nhiều projects
nx run build my-app another-app --parallel
```

#### 2. nx g - Code Generation

```bash
# Tạo React application mới
nx g @nx/react:app my-new-app

# Tạo React library mới
nx g @nx/react:lib my-new-lib

# Tạo component trong library
nx g @nx/react:component --name=Button --project=my-lib --export=true

# Tạo .NET WebAPI
nx g @nx/dotnet:app my-api --template=webapi

# Tạo .NET library
nx g @nx/dotnet:lib my-lib
```

#### 3. nx affected - Chỉ Chạy Affected Projects

```bash
# List affected projects
nx affected

# Chạy task trên affected projects
nx affected --target=build

# Chạy nhiều tasks trên affected projects
nx affected --target=build --target=test --parallel

# Xem affected projects trong graph
nx affected --graph
```

#### 4. nx graph - Xem Dependency Graph

```bash
# Xem project graph
nx graph

# Xem graph của affected projects
nx graph --affected

# Lưu graph ra file
nx graph --file=output.html
```

### Hello World Example

#### Step 1: Tạo React App

```bash
# Tạo workspace
npx create-nx-workspace@latest myworkspace --preset=react-monorepo

# Chuyển vào workspace
cd myworkspace

# Tạo React app
nx g @nx/react:app my-app
```

#### Step 2: Chạy App

```bash
# Serve app
nx run my-app:serve

# Build app
nx run my-app:build

# Test app
nx run my-app:test
```

#### Step 3: Tạo Shared Library

```bash
# Tạo UI library
nx g @nx/react:lib ui

# Tạo component trong library
nx g @nx/react:component --name=Button --project=ui --export=true
```

#### Step 4: Sử Dụng Library trong App

```tsx
// apps/my-app/src/app/app.tsx
import { Button } from '@myworkspace/ui';

export function App() {
  return (
    <div>
      <h1>Hello Nx!</h1>
      <Button onClick={() => alert('Clicked!')}>
        Click Me
      </Button>
    </div>
  );
}
```

### Common Use Cases

#### Use Case 1: Shared UI Components

**Bài toán**: Multiple React apps cần dùng chung UI components.

**Giải pháp**: Tạo shared library với reusable components.

```bash
# Tạo UI library
nx g @nx/react:lib shared-ui

# Tạo component
nx g @nx/react:component --name=Card --project=shared-ui --export=true

# Sử dụng trong app1
import { Card } from '@myworkspace/shared-ui';

# Sử dụng trong app2
import { Card } from '@myworkspace/shared-ui';
```

#### Use Case 2: Shared Types với .NET

**Bài toán**: React frontend và .NET backend cần chia sẻ DTO types.

**Giải pháp**: Tạo shared types library.

```bash
# Tạo types library
nx g @nx/js:lib shared-types

# Define TypeScript types
// libs/shared-types/src/index.ts
export interface Product {
  id: string;
  name: string;
  price: number;
}

// Sử dụng trong React
import { Product } from '@myworkspace/shared-types';

// Sử dụng trong .NET (convert to C# classes)
// Product.cs
public class Product {
    public string Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
}
```

#### Use Case 3: API Client Layer

**Bài toán**: Multiple apps cần gọi cùng API endpoints.

**Giải pháp**: Tạo centralized API client library.

```bash
# Tạo API client library
nx g @nx/react:lib api-client

// libs/api-client/src/api.ts
export const apiClient = {
  getProducts: () => fetch('/api/products').then(r => r.json()),
  createProduct: (data: Product) => fetch('/api/products', {
      method: 'POST',
      body: JSON.stringify(data)
    }).then(r => r.json())
};

// Sử dụng trong app
import { apiClient } from '@myworkspace/api-client';

const products = await apiClient.getProducts();
```

#### Use Case 4: Testing Strategy

**Bài toán**: Chạy tests trên affected projects để tiết kiệm thời gian.

**Giải pháp**: Sử dụng `nx affected` với target test.

```bash
# Chỉ test affected projects
nx affected --target=test

# Chỉ test projects với changes trong files cụ thể
nx affected --target=test --files=apps/my-app/src/**/*.ts

# Chạy tests parallel
nx affected --target=test --parallel=3
```

#### Use Case 5: Build Optimization

**Bài toán**: Build time quá lâu với nhiều projects.

**Giải pháp**: Sử dụng caching và parallel execution.

```bash
# Build với caching (mặc định)
nx run build

# Build affected projects
nx affected --target=build

# Build parallel với max 3 jobs
nx run build --parallel=3

# Build và skip cache (để test)
nx run build --skip-nx-cache
```

### Common Pitfalls

#### Pitfall 1: Hardcoding Import Paths

❌ **Sai**: Hardcoding relative paths.

```tsx
// apps/my-app/src/components/ProductList.tsx
import { Card } from '../../../libs/shared-ui/src/components/Card';
```

✅ **Đúng**: Sử dụng workspace alias.

```tsx
// apps/my-app/src/components/ProductList.tsx
import { Card } from '@myworkspace/shared-ui';
```

**Giải thích**: Nx tự động cấu hình TypeScript để sử dụng workspace alias. Hardcoding paths sẽ break khi thay đổi cấu trúc folder.

#### Pitfall 2: Không Hiểu Affected Projects

❌ **Sai**: Luôn build/test tất cả projects.

```bash
# Luôn build tất cả (chậm)
nx run build --all
```

✅ **Đúng**: Chỉ build affected projects.

```bash
# Chỉ build affected projects (nhanh)
nx affected --target=build
```

**Giải thích**: `nx affected` chỉ chạy tasks trên projects bị ảnh hưởng bởi changes, tiết kiệm đáng kể thời gian.

#### Pitfall 3: Disable Caching Unnecessarily

❌ **Sai**: Disable cache cho mọi tasks.

```json
// nx.json
{
  "tasksRunnerOptions": {
    "cacheableOperations": "none"
  }
}
```

✅ **Đúng**: Enable caching cho hầu hết tasks.

```json
// nx.json
{
  "tasksRunnerOptions": {
    "cacheableOperations": "default"
  }
}
```

**Giải thích**: Caching là feature mạnh mẽ nhất của Nx. Chỉ disable khi cần thiết (tests, deployments).

#### Pitfall 4: Quá Nhiều Small Libraries

❌ **Sai**: Tạo quá nhiều nhỏ libraries.

```
libs/
├── button/
├── input/
├── card/
├── modal/
├── dropdown/
└── ...
```

✅ **Đúng**: Group components thành libraries lớn.

```
libs/
├── ui/              # Tất cả UI components
└── forms/            # Tất cả form components
```

**Giải thích**: Quá nhiều nhỏ libraries gây:
- Phức tạp dependency management
- Build overhead cao hơn
- Khó maintain

#### Pitfall 5: Không Sử Dụng project.json

❌ **Sai**: Chỉ dùng nx.json cho mọi config.

```json
// nx.json
{
  "targets": {
    "build": {
      "executor": "@nx/vite:build"
    }
  }
}
```

✅ **Đúng**: Định nghĩa targets trong mỗi project.json.

```json
// apps/my-app/project.json
{
  "targets": {
    "build": {
      "executor": "@nx/vite:build",
      "options": {
        "outputPath": "dist/apps/my-app"
      }
    }
  }
}

// nx.json (workspace-level config)
{
  "defaultBase": "my-app"
}
```

**Giải thích**: Mỗi project có `project.json` với targets riêng. `nx.json` chỉ chứa workspace-level config.

### Best Practices Cơ Bản

1. **Sử Dụng Workspace Aliases**
   - Import từ libraries với alias: `@myworkspace/my-lib`
   - Không hardcode relative paths

2. **Chỉ Chạy Affected Projects trong CI**
   ```bash
   nx affected --target=build --target=test
   ```

3. **Enable Caching**
   - Luôn để caching enabled
   - Chỉ disable khi cần thiết

4. **Group Related Code vào Libraries**
   - Tạo libraries theo feature/domain
   - Tránh quá nhiều nhỏ libraries

5. **Sử Dụng Generators**
   - Dùng `nx g` để tạo boilerplate code
   - Tăng consistency và productivity

6. **Maintain Clean Folder Structure**
   - `apps/` cho deployable applications
   - `libs/` cho shared libraries
   - Tránh quá nhiều nested folders

7. **Review Project Graph Thường Xuyên**
   ```bash
   nx graph
   ```
   - Hiểu dependencies giữa projects
   - Xác định circular dependencies

8. **Sử Dụng TypeScript Project References**
   - Cấu hình `tsconfig.base.json` với paths
   - Enable type checking across projects

9. **Test Affected Projects**
   ```bash
   nx affected --target=test
   ```
   - Tiết kiệm CI time

10. **Document Custom Targets**
    - Comment trong `project.json`
    - Giải thích options và usage

---

## Middle Level - Trung Cấp

### Tính Năng Nâng Cao Tầm Trung

#### 1. Task Orchestration

**Affected Projects với Dependencies**

```bash
# Chỉ build affected projects + dependencies của chúng
nx affected --target=build --with-deps

# Build affected projects và tất cả libraries chúng phụ thuộc vào
nx affected --target=build --include-dependencies

# Chỉ test projects bị ảnh hưởng trực tiếp
nx affected --target=test --exclude-dependencies
```

**Parallel Execution**

```bash
# Chạy tasks parallel với max 3 jobs
nx run build --parallel=3

# Chạy tasks trên nhiều projects parallel
nx run build my-app another-app third-app --parallel

# Chạy affected tasks parallel
nx affected --target=build --parallel
```

**Sequential Execution cho Dependencies**

```bash
# Chạy tasks theo thứ tự dependency
nx run build --parallel=false

# Chạy test sau khi build thành công
nx run my-app:build my-app:test
```

#### 2. Caching Strategies

**Local Caching Configuration**

```json
// nx.json
{
  "tasksRunnerOptions": {
    "cacheableOperations": "default",
    "cacheDirectory": "node_modules/.cache/nx",
    "encryptionKey": "some-encryption-key"
  }
}
```

**Cache Inputs Customization**

```json
// apps/my-app/project.json
{
  "targets": {
    "build": {
      "executor": "@nx/vite:build",
      "inputs": [
        "default",
        "{projectRoot}/**/*.md",
        "!{projectRoot}/README.md"
      ],
      "outputs": [
        "{projectRoot}/dist"
      ]
    }
  }
}
```

**Skip Cache cho Specific Tasks**

```bash
# Bỏ qua cache cho task này
nx run my-app:build --skip-nx-cache

# Luôn build không cache (debug)
nx run build --skip-nx-cache
```

#### 3. Pagination và Invalidation

**Cache Invalidation Strategies**

```bash
# Invalidate cache cho tất cả
nx reset

# Invalidate cache cho specific project
nx reset my-app

# Clear remote cache (Nx Cloud)
nx reset --cloud
```

**Cache Health Check**

```bash
# Xem cache statistics
nx show project my-app

# Xem cache hit/miss rate
nx report
```

#### 4. Dependency Management

**Implicit Dependencies**

Nx tự động detect dependencies từ:
- Imports trong TypeScript/JavaScript
- Imports trong C# (.NET)
- Package dependencies trong `package.json`
- Project references trong `.csproj` files

**Explicit Dependencies**

```json
// libs/ui/project.json
{
  "implicitDependencies": ["shared-types"],
  "namedInputs": {
    "my-config": ["my-config.json"]
  }
}
```

**Dependency Visualization**

```bash
# Xem dependency graph
nx graph

# Xem affected projects graph
nx graph --affected

# Focus vào specific project
nx graph --focus=my-app

# Lưu graph ra file
nx graph --file=graph.html
```

### Patterns Thường Dùng

#### Pattern 1: Module Federation với React

**Bài toán**: Micro-frontends với independent deployments.

**Giải pháp**: Module Federation cho React apps.

```bash
# Tạo host app
nx g @nx/react:host my-host-app

# Tạo remote app
nx g @nx/react:remote my-remote-app

# Configure federation trong vite.config.ts
// apps/my-host-app/vite.config.ts
import { federation } from '@softarcuery/federation-plugin';
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [
    react(),
    federation({
      name: 'host',
      remotes: {
        remote: 'http://localhost:4201/assets/remoteEntry.js'
      },
      shared: ['react', 'react-dom']
    })
  ]
});
```

```tsx
// apps/my-host-app/src/app/App.tsx
const RemoteApp = lazy(() => 
  import('remote/RemoteApp')
);

export function App() {
  return (
    <div>
      <h1>Host App</h1>
      <Suspense fallback={<div>Loading...</div>}>
        <RemoteApp />
      </Suspense>
    </div>
  );
}
```

#### Pattern 2: Shared API Client Layer

**Bài toán**: Centralize API calls với error handling.

**Giải pháp**: Tạo API client library với interceptors.

```typescript
// libs/api-client/src/apiClient.ts
import axios, { AxiosError } from 'axios';

interface ApiClientConfig {
  baseUrl: string;
  interceptors?: {
    request?: (config: any) => any;
    response?: (response: any) => any;
    error?: (error: AxiosError) => any;
  };
}

export function createApiClient(config: ApiClientConfig) {
  const client = axios.create({
      baseURL: config.baseUrl,
      timeout: 10000
    });

  // Request interceptor
  if (config.interceptors?.request) {
    client.interceptors.request.use(config.interceptors.request);
  }

  // Response interceptor
  if (config.interceptors?.response) {
    client.interceptors.response.use(config.interceptors.response);
  }

  // Error interceptor
  if (config.interceptors?.error) {
    client.interceptors.response.use(
      (response) => response,
      (error: AxiosError) => {
        if (config.interceptors?.error) {
          return config.interceptors.error(error);
        }
        return Promise.reject(error);
      }
    );
  }

  return client;
}

// libs/api-client/src/index.ts
export const apiClient = createApiClient({
  baseUrl: import.meta.env.VITE_API_URL,
  interceptors: {
      request: (config) => {
        // Add auth token
        const token = localStorage.getItem('token');
        if (token) {
          config.headers.Authorization = `Bearer ${token}`;
        }
        return config;
      },
      error: (error) => {
        // Handle common errors
        if (error.response?.status === 401) {
          window.location.href = '/login';
        }
        return Promise.reject(error);
      }
    }
});

// libs/api-client/src/products.ts
export const productsApi = {
  getAll: () => apiClient.get<Product[]>('/products'),
  getById: (id: string) => apiClient.get<Product>(`/products/${id}`),
  create: (data: CreateProductDto) => apiClient.post<Product>('/products', data),
  update: (id: string, data: UpdateProductDto) => 
    apiClient.put<Product>(`/products/${id}`, data),
  delete: (id: string) => apiClient.delete(`/products/${id}`)
};
```

```tsx
// apps/my-app/src/pages/ProductListPage.tsx
import { productsApi } from '@myworkspace/api-client';

export function ProductListPage() {
  const { data: products, isLoading } = useQuery({
      queryKey: ['products'],
      queryFn: () => productsApi.getAll().then(r => r.data)
    });

  if (isLoading) return <div>Loading...</div>;

  return (
    <div>
      {products?.map(product => (
        <div key={product.id}>
          <h2>{product.name}</h2>
          <p>${product.price}</p>
        </div>
      ))}
    </div>
  );
}
```

#### Pattern 3: .NET Integration

**Bài toán**: Tích hợp .NET backend vào Nx workspace.

**Giải pháp**: Sử dụng @nx/dotnet plugin.

```bash
# Thêm .NET plugin
nx add @nx/dotnet

# Tạo .NET WebAPI
nx g @nx/dotnet:app webapi --template=webapi

# Tạo .NET library
nx g @nx/dotnet:lib shared-domain

// backend/Application/Application.csproj
{
  "targets": {
    "build": {
      "executor": "@nx/dotnet:build",
      "options": {
        "configuration": "Release"
      }
    },
    "test": {
      "executor": "@nx/dotnet:test"
    }
  }
}

// backend/Domain/Domain.csproj
{
  "targets": {
    "build": {
      "executor": "@nx/dotnet:build"
    }
  }
}

// Reference domain library từ application
// backend/Application/Application.csproj
<ItemGroup>
  <ProjectReference Include="..\Domain\Domain.csproj" />
</ItemGroup>

// Shared types giữa React và .NET
// libs/shared-types/src/product.ts
export interface Product {
  id: string;
  name: string;
  price: number;
}

// backend/Domain/Entities/Product.cs
namespace MyWorkspace.Domain.Entities
{
    public class Product
    {
        public string Id { get; set; }
        public string Name { get; set; }
        public decimal Price { get; set; }
    }
}
```

#### Pattern 4: Environment-Specific Configs

**Bài toán**: Khác biệt configs cho dev/staging/prod.

**Giải pháp**: Sử dụng environment variables và Nx config.

```bash
# .env.development
VITE_API_URL=http://localhost:5000/api

# .env.production
VITE_API_URL=https://api.taphoanho.com/api

# apps/my-app/project.json
{
  "targets": {
    "serve": {
      "executor": "@nx/vite:dev-server",
      "options": {
        "buildTarget": "development"
      },
      "configurations": {
        "development": {
          "buildTarget": "development"
        },
        "production": {
          "buildTarget": "production"
        }
      }
    },
    "build": {
      "executor": "@nx/vite:build",
      "configurations": {
        "development": {
          "mode": "development"
        },
        "production": {
          "mode": "production"
        }
      }
    }
  }
}
```

```bash
# Serve development
nx run my-app:serve

# Build production
nx run my-app:build:production
```

### Error Handling & Performance Tầm Trung Cấp

#### 1. Common Errors và Solutions

**Error: Module not found**

```bash
Error: Cannot find module '@myworkspace/ui'
```

**Giải pháp**:
```bash
# 1. Kiểm tra tsconfig.base.json
# Đảm bảo paths được cấu hình
{
  "compilerOptions": {
    "paths": {
      "@myworkspace/*": ["libs/*"]
    }
  }
}

# 2. Rebuild project
nx reset my-app
nx run my-app:build
```

**Error: Cache invalidation issue**

```bash
Error: Task results are inconsistent
```

**Giải pháp**:
```bash
# Reset cache
nx reset

# Re-run task
nx run my-app:build
```

**Error: Dependency cycle**

```bash
Error: Circular dependency detected
```

**Giải pháp**:
```bash
# Xem dependency graph
nx graph

# Tìm circular dependency và refactor
# Ví dụ: lib A phụ thuộc vào B, B phụ thuộc vào A
# Refactor: Tạo lib C mà cả A và B phụ thuộc vào
```

#### 2. Performance Optimization

**Optimize Build Time**

```bash
# Chỉ build affected projects
nx affected --target=build

# Chạy parallel
nx run build --parallel

# Sử dụng Nx Cloud (remote caching)
nx connect-to-nx-cloud
```

**Optimize Test Time**

```bash
# Chỉ test affected projects
nx affected --target=test

# Chạy tests parallel
nx run test --parallel

# Skip slow tests khi needed
nx run my-app:test --testName="^((?!Slow).)*"
```

**Optimize Cache Hit Rate**

```bash
# Xem cache statistics
nx report

# Review cache misses và optimize inputs
# Ví dụ: Bỏ qua unnecessary files từ inputs
```

---

## Senior Level - Nâng Cao

### Chiến Lược Caching/Optimizing Chính

#### 1. Advanced Caching Strategies

**Fine-Grained Cache Inputs**

```json
// apps/my-app/project.json
{
  "targets": {
    "build": {
      "executor": "@nx/vite:build",
      "inputs": [
        "default",
        {
          "fileset": "env",
          "files": [
            "{workspaceRoot}/.env.production"
          ],
          "runtime": "shell"
        }
      ],
      "outputs": [
        "{projectRoot}/dist"
      ]
    }
  }
}
```

**Cache Outputs Customization**

```json
{
  "outputs": [
    "{workspaceRoot}/dist/{projectRoot}",
    "{projectRoot}/.next/cache"
  ]
}
```

**Pipeline Caching**

```json
// nx.json
{
  "tasksRunnerOptions": {
    "pipeline": {
      "my-app:build": {
        "dependsOn": [
          "shared-ui:build",
          "api-client:build"
        ]
      }
    }
  }
}
```

#### 2. Distributed Task Execution

**Nx Cloud Setup**

```bash
# Connect workspace to Nx Cloud
nx connect-to-nx-cloud

# Enable distributed execution
nx run build --parallel --cloud

# Run tasks on remote agents
nx affected --target=build --target=test --cloud
```

**Self-Healing CI**

```yaml
# .github/workflows/ci.yml
name: CI
on: [push]
jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npx nx affected --target=build --target=test --parallel=3
```

### Tích Hợp với State Management/Kiến Trúc Khác

#### 1. Tích hợp với TanStack Query

```bash
# Thêm TanStack Query plugin
nx g @nx/react:lib data-layer

// libs/data-layer/src/hooks/useProducts.ts
import { useQuery } from '@tanstack/react-query';
import { productsApi } from '@myworkspace/api-client';

export function useProducts() {
  return useQuery({
      queryKey: ['products'],
      queryFn: () => productsApi.getAll().then(r => r.data)
    });
}

// apps/my-app/src/pages/ProductListPage.tsx
import { useProducts } from '@myworkspace/data-layer';

export function ProductListPage() {
  const { data: products, isLoading } = useProducts();

  // Render products...
}
```

#### 2. Tích hợp với Zustand

```bash
# Tạo store library
nx g @nx/react:lib state-store

// libs/state-store/src/stores/productStore.ts
import { create } from 'zustand';

interface ProductState {
  products: Product[];
  setProducts: (products: Product[]) => void;
}

export const useProductStore = create<ProductState>((set) => ({
  products: [],
  setProducts: (products) => set({ products })
}));

// apps/my-app/src/pages/ProductListPage.tsx
import { useProductStore } from '@myworkspace/state-store';

export function ProductListPage() {
  const { products, setProducts } = useProductStore();

  // Use products...
}
```

### Monitoring & Logging Cơ Bản

#### 1. Nx Console

```bash
# Cài đặt Nx Console extension trong VS Code
# View: https://marketplace.visualstudio.com/items?itemName=nrwl.angular-console

# Features:
# - Project graph visualization
# - Task dependencies visualization
# - Affected projects view
# - Task runner UI
```

#### 2. Logging Configuration

```json
// nx.json
{
  "tasksRunnerOptions": {
    "logger": "nx-console"
  }
}
```

#### 3. Performance Metrics

```bash
# Xem report chi tiết
nx report

# Xem cache statistics
nx show project my-app --web

# Xem task execution time
nx run my-app:build --verbose
```

---

## Tài Liệu Tham Khảo

### Official Documentation
- **Nx Docs**: https://nx.dev
- **Nx GitHub**: https://github.com/nrwl/nx
- **Nx Cloud**: https://nx.app

### Tài Liệu Liên Quan
- **TanStack Query Documentation**: [TanStack Query.md](../TanStack%20Query/TanStack%20Query.md)
- **.NET Clean Architecture**: [System Analysis](../../CORE-MOBILE-APP/docs/System_Analysis.md)
- **React Patterns**: [TanStack Router](../TanStack%20Router/README.md)

### Community Resources
- **Nx Discord**: https://discord.gg/nx
- **Nx Console**: https://marketplace.visualstudio.com/items?itemName=nrwl.angular-console
- **Nx Examples**: https://github.com/nrwl/nx-examples
