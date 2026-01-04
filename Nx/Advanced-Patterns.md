# Nx Advanced Patterns

**Table of Contents**
- [Custom Configuration](#custom-configuration)
- [Caching Strategies](#caching-strategies)
- [Performance Optimization](#performance-optimization)
- [Testing Nâng Cao](#testing-nâng-cao)
- [Custom Plugins và Generators](#custom-plugins-và-generators)

---

## Custom Configuration

### Custom Executors

**Bài toán**: Tạo custom executor cho tasks đặc biệt.

**Giải pháp**: Định nghĩa custom executor trong project.

```json
// tools/custom-executors/src/build-custom.js
const { exec } = require('@nx/devkit');

exports.default = exec('custom-build', (opts) => {
  console.log('Building with custom executor...');
  return exec('npx tsc', {
      cwd: opts.cwd,
      env: process.env
    });
});
```

```json
// project.json
{
  "targets": {
    "build": {
      "executor": "./tools/custom-executors/build-custom"
    }
  }
}
```

### Global Error Handling

**Bài toán**: Centralize error handling cho tất cả tasks.

**Giải pháp**: Configure global error handler trong nx.json.

```json
// nx.json
{
  "targetDefaults": {
    "build": {
      "options": {
        "maxWorkers": 3,
        "parallel": false
      },
      "dependsOn": ["^build"]
    },
    "test": {
      "options": {
        "passWithNoTests": true
      }
    }
  },
  "tasksRunnerOptions": {
    "cacheableOperations": "default",
    "parallel": 3
  }
}
```

### Retry Logic

**Bài toán**: Auto-retry failed tasks.

**Giải pháp**: Configure retry trong task options.

```json
// apps/my-app/project.json
{
  "targets": {
    "test": {
      "executor": "@nx/vitest:test",
      "options": {
        "passWithNoTests": true,
        "retries": 2,
        "timeout": 30000
      }
    }
  }
}
```

### Custom Middleware

**Bài toán**: Inject custom logic vào task execution.

**Giải pháp**: Sử dụng Nx middleware trong executor.

```typescript
// tools/custom-middleware/index.ts
import { createProjectGraphAsync } from '@nx/devkit';
import { workspaceRoot } from 'nx/src/utils/workspace-root';

export async function customMiddleware() {
  const graph = await createProjectGraphAsync();
  console.log('Custom middleware: Processing project graph...');
  
  // Custom logic here
  return graph;
}
```

```json
// nx.json
{
  "plugins": [
    "./tools/custom-middleware"
  ]
}
```

---

## Caching Strategies

### Cache Key Factory

**Bài toán**: Tối ưu cache keys cho complex dependencies.

**Giải pháp**: Define custom cache inputs.

```json
// apps/my-app/project.json
{
  "targets": {
    "build": {
      "executor": "@nx/vite:build",
      "inputs": [
        "default",
        {
          "runtime": "node",
          "dependencies": ["{workspaceRoot}/config/build.config.js"]
        },
        {
          "runtime": "shell",
          "executeCommand": "git rev-parse HEAD",
          "cacheName": "git-sha"
        },
        {
          "fileset": "env",
          "files": [
            "{workspaceRoot}/.env.production",
            "{workspaceRoot}/.env.development"
          ]
        }
      ],
      "outputs": [
        "{projectRoot}/dist",
        "{projectRoot}/.nx/cache"
      ]
    }
  }
}
```

### Centralized Cache Management

**Bài toán**: Share cache giữa team members.

**Giải pháp**: Sử dụng Nx Cloud hoặc custom cache server.

```bash
# Connect to Nx Cloud
nx connect-to-nx-cloud

# Setup custom cache server (alternative)
# nx.json
{
  "tasksRunnerOptions": {
    "cacheDirectory": "/path/to/shared/cache",
    "cacheableOperations": "default"
  }
}
```

```json
// nx.json với Nx Cloud
{
  "nxCloudAccessToken": "your-access-token",
  "nxCloudUrl": "https://cloud.nx.app",
  "tasksRunnerOptions": {
    "cacheableOperations": "default",
    "parallel": 3
  }
}
```

### Cache Invalidation Patterns

**Pattern 1: Time-based Invalidation**

```bash
# Invalidate cache mỗi ngày
nx reset --cache && nx run build

# Script trong package.json
{
  "scripts": {
    "build:clean": "nx reset && nx run build"
  }
}
```

**Pattern 2: Dependency-based Invalidation**

```json
// nx.json
{
  "targetDefaults": {
    "build": {
      "inputs": [
        "default",
        "{workspaceRoot}/package.json",
        "{workspaceRoot}/tsconfig.base.json"
      ]
    }
  }
}
```

**Pattern 3: Environment-based Invalidation**

```bash
# Invalidate cache khi environment thay đổi
if [ "$ENV" != "$LAST_ENV" ]; then
  nx reset
  export LAST_ENV=$ENV
fi

nx run build
```

### Cache Monitoring

```bash
# Xem cache statistics
nx report

# Xem chi tiết cho project
nx show project my-app --web

# Xem cache hit/miss rate
nx show project my-app | grep "Cache"
```

---

## Performance Optimization

### Structural Sharing

**Bài toán**: Optimize cache efficiency với structural sharing.

**Giải pháp**: Configure outputs với shared paths.

```json
// apps/my-app/project.json
{
  "targets": {
    "build": {
      "executor": "@nx/vite:build",
      "outputs": [
        "{workspaceRoot}/dist/apps/my-app",
        "{projectRoot}/node_modules/.cache/vite"
      ]
    }
  }
}
```

### Select Optimization

**Bài toán**: Reduce cache size với selective outputs.

**Giải pháp**: Chỉ cache necessary files.

```json
{
  "outputs": [
    "{projectRoot}/dist/**/*.{js,css,html}",
    "!{projectRoot}/dist/**/*.map"
  ]
}
```

### Prefetching Strategies

**Pattern 1: Prefetch Dependencies**

```bash
# Prefetch dependencies khi build project
nx run my-app:prefetch-deps && nx run my-app:build
```

```json
// apps/my-app/project.json
{
  "targets": {
    "prefetch-deps": {
      "executor": "nx:run-commands",
      "options": {
        "commands": [
          "npm install",
          "npm list --depth=0"
        ],
        "parallel": false
      }
    }
  }
}
```

**Pattern 2: Prefetch Build Artifacts**

```typescript
// tools/prefetcher/index.ts
import { exec } from '@nx/devkit';

export async function prefetchExecutor(options) {
  console.log('Prefetching build artifacts...');
  
  // Prefetch từ remote cache
  await exec('npx nx show project my-app --web');
  
  return { success: true };
}
```

### Initial Data Patterns

**Pattern 1: Server-Side Data Injection**

```typescript
// apps/my-app/src/main.tsx
import { hydrateRoot } from '@tanstack/react-query-next-experimental';
import { dehydrate } from '@tanstack/react-query';
import { queryClient } from './lib/query-client';

export default function Root() {
  // Inject initial data từ server
  const dehydratedState = dehydrate(queryClient);
  
  return (
    <HydrationBoundary state={dehydratedState}>
      <App />
    </HydrationBoundary>
  );
}
```

**Pattern 2: Client-Side Data Prefetching**

```typescript
// apps/my-app/src/pages/ProductListPage.tsx
import { useQuery, useQueryClient } from '@tanstack/react-query';
import { productsApi } from '@myworkspace/api-client';

export function ProductListPage() {
  const queryClient = useQueryClient();
  const { data: products } = useQuery({
      queryKey: ['products'],
      queryFn: () => productsApi.getAll().then(r => r.data)
    });
  
  // Prefetch detail page khi hover
  const prefetchProduct = (id: string) => {
    queryClient.prefetchQuery({
      queryKey: ['product', id],
      queryFn: () => productsApi.getById(id).then(r => r.data)
    });
  };
  
  return (
    <div>
      {products?.map(product => (
        <div 
          key={product.id}
          onMouseEnter={() => prefetchProduct(product.id)}
        >
          <h2>{product.name}</h2>
        </div>
      ))}
    </div>
  );
}
```

---

## Testing Nâng Cao

### Unit Tests cho Hooks

**Pattern 1: Test React Hooks với Vitest**

```typescript
// libs/api-client/src/__tests__/useProducts.test.ts
import { renderHook, act, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useProducts } from '../src/hooks/useProducts';

describe('useProducts', () => {
  let queryClient: QueryClient;
  
  beforeEach(() => {
    queryClient = new QueryClient({
      defaultOptions: {
        queries: {
          retry: false,
        },
      },
    });
  });
  
  it('should fetch products', async () => {
    const { result } = renderHook(() => useProducts(), {
      wrapper: ({ children }) => (
        <QueryClientProvider client={queryClient}>
          {children}
        </QueryClientProvider>
      ),
    });
    
    await waitFor(() => result.current.isSuccess);
    expect(result.current.data).toHaveLength(10);
  });
  
  it('should handle error', async () => {
    // Mock failed request...
    const { result } = renderHook(() => useProducts(), {
      wrapper: ({ children }) => (
        <QueryClientProvider client={queryClient}>
          {children}
        </QueryClientProvider>
      ),
    });
    
    await waitFor(() => result.current.isError);
    expect(result.current.error).toBeDefined();
  });
});
```

**Pattern 2: Test Zustand Stores**

```typescript
// libs/state-store/src/__tests__/productStore.test.ts
import { renderHook, act } from '@testing-library/react';
import { useProductStore } from '../src/stores/productStore';

describe('useProductStore', () => {
  it('should set products', () => {
    const { result } = renderHook(() => useProductStore());
    
    act(() => {
      result.current.setProducts([
        { id: '1', name: 'Product 1', price: 10 }
      ]);
    });
    
    expect(result.current.products).toHaveLength(1);
    expect(result.current.products[0].name).toBe('Product 1');
  });
  
  it('should persist state', () => {
    const { result: store1 } = renderHook(() => useProductStore());
    const { result: store2 } = renderHook(() => useProductStore());
    
    act(() => {
      store1.current.setProducts([
        { id: '1', name: 'Product 1', price: 10 }
      ]);
    });
    
    expect(store2.current.products).toEqual(store1.current.products);
  });
});
```

### Integration Tests với MSW

**Pattern 1: Mock API Responses**

```typescript
// apps/my-app/src/__tests__/api.test.ts
import { setupServer } from 'msw/node';
import { rest } from 'msw';
import { productsApi } from '@myworkspace/api-client';

const server = setupServer(
  rest.get('/api/products', (req, res, ctx) => {
    return res(
      ctx.status(200),
      ctx.json([
        { id: '1', name: 'Product 1', price: 10 },
        { id: '2', name: 'Product 2', price: 20 }
      ])
    );
  })
);

describe('productsApi', () => {
  afterAll(() => server.close());
  
  it('should fetch products', async () => {
    const products = await productsApi.getAll();
    expect(products).toHaveLength(2);
  });
  
  it('should handle errors', async () => {
    // Mock error response...
    await expect(productsApi.getById('999')).rejects.toThrow();
  });
});
```

**Pattern 2: Test Error Handling**

```typescript
// libs/api-client/src/__tests__/errorHandling.test.ts
import { productsApi } from '../src/apiClient';
import { setupServer } from 'msw/node';
import { rest } from 'msw';

const server = setupServer(
  rest.get('/api/products', (req, res, ctx) => {
    return res(ctx.status(500), ctx.json({ error: 'Internal Server Error' }));
  })
);

describe('Error Handling', () => {
  afterAll(() => server.close());
  
  it('should retry failed requests', async () => {
    // Test retry logic...
  });
  
  it('should log errors', async () => {
    const consoleSpy = jest.spyOn(console, 'error');
    await expect(productsApi.getAll()).rejects.toThrow();
    expect(consoleSpy).toHaveBeenCalled();
  });
});
```

### Component Tests với Testing Library

**Pattern 1: Test với React Testing Library**

```typescript
// apps/my-app/src/components/__tests__/ProductList.test.tsx
import { render, screen } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ProductList } from '../ProductList';

describe('ProductList', () => {
  const queryClient = new QueryClient({
      defaultOptions: {
        queries: {
          retry: false,
          staleTime: Infinity,
        },
      },
    });
  
  it('should render products', async () => {
    render(
      <QueryClientProvider client={queryClient}>
        <ProductList products={[
          { id: '1', name: 'Product 1', price: 10 }
        ]} />
      </QueryClientProvider>
    );
    
    expect(screen.getByText('Product 1')).toBeInTheDocument();
  });
  
  it('should show loading state', () => {
    render(
      <QueryClientProvider client={queryClient}>
        <ProductList products={null} isLoading={true} />
      </QueryClientProvider>
    );
    
    expect(screen.getByText('Loading...')).toBeInTheDocument();
  });
});
```

**Pattern 2: Test User Interactions**

```typescript
// libs/ui/src/components/__tests__/Button.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { Button } from '../Button';

describe('Button', () => {
  it('should call onClick', () => {
    const handleClick = jest.fn();
    render(<Button onClick={handleClick}>Click Me</Button>);
    
    fireEvent.click(screen.getByText('Click Me'));
    expect(handleClick).toHaveBeenCalledTimes(1);
  });
  
  it('should be disabled', () => {
    render(<Button disabled>Click Me</Button>);
    
    const button = screen.getByText('Click Me');
    expect(button).toBeDisabled();
  });
});
```

### Test Orchestration với Nx

```bash
# Chạy tất cả tests
nx run test

# Chạy tests parallel
nx run test --parallel

# Chỉ test affected projects
nx affected --target=test

# Chạy tests với coverage
nx run test --codeCoverage

# Chạy e2e tests sau unit tests
nx run test && nx run e2e
```

---

## Custom Plugins và Generators

### Tạo Custom Generator

**Pattern 1: CRUD Generator**

```typescript
// tools/generators/crud/index.ts
import {
  addProjectConfiguration,
  formatFiles,
  generateFiles,
  installPackagesTask,
  names,
  offsetFromRoot,
  readProjectConfiguration,
  Tree,
  updateJson
} from '@nx/devkit';
import * as path from 'path';

export async function crudGenerator(tree: Tree, options: CrudGeneratorSchema) {
  const { name, project } = options;
  const projectConfig = readProjectConfiguration(tree, project);
  const sourceRoot = projectConfig.sourceRoot;
  const projectRoot = projectConfig.root;
  const sourceDir = `${offsetFromRoot(projectRoot)}/${sourceRoot}/${projectConfig.projectType === 'application' ? 'app' : 'lib'}/${project}`;
  const templateDir = path.join(__dirname, './files');
  
  // Generate files từ template
  generateFiles(tree, templateDir, sourceDir, {
    name,
    className: names(name).className,
    fileName: names(name).fileName,
    propertyName: names(name).propertyName,
    constantName: names(name).constantName,
  });
  
  // Add exports to index
  if (projectConfig.projectType === 'library') {
    const indexFilePath = path.join(sourceRoot, projectConfig.projectType === 'application' ? 'app' : 'lib', project, 'src/index.ts');
    const content = tree.read(indexFilePath);
    if (!content.includes(`export * from './${names(name).fileName}'`)) {
      tree.write(
        indexFilePath,
        `${content}\nexport * from './${names(name).fileName}';\n`
      );
    }
  }
  
  await formatFiles(tree);
  
  return () => {
    installPackagesTask(tree, 'npm', ['axios', '@tanstack/react-query']);
  };
}

export default crudGenerator;

interface CrudGeneratorSchema {
  name: string;
  project: string;
}
```

```typescript
// tools/generators/crud/schema.json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "cli": "nx",
  "$id": "Crud",
  "title": "Create CRUD operations",
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "description": "The name of the resource",
      "$default": "product",
      "x-prompt": "What is the resource name?"
    },
    "project": {
      "type": "string",
      "description": "The project to generate into",
      "$default": "my-lib",
      "x-prompt": "Which project should this be generated in?"
    }
  },
  "required": ["name", "project"]
}
```

```json
// nx.json
{
  "generators": {
    "@nx/react:component": {
      "package": "@nx/react",
      "schema": "./node_modules/@nx/react/generators/component/schema.json"
    },
    "crud": {
      "package": "./tools/generators/crud",
      "schema": "./tools/generators/crud/schema.json"
    }
  }
}
```

**Usage:**

```bash
# Sử dụng custom CRUD generator
nx g crud --name=product --project=my-lib

# Với alias
nx g @workspace/crud --name=category --project=my-lib
```

### Custom Hooks Architecture

**Pattern 1: Feature-based Hooks Organization**

```
libs/
├── features/
│   ├── products/
│   │   ├── src/
│   │   │   ├── hooks/
│   │   │   │   ├── useProducts.ts
│   │   │   │   ├── useProductDetail.ts
│   │   │   │   └── useCreateProduct.ts
│   │   │   ├── services/
│   │   │   │   └── productsApi.ts
│   │   │   └── index.ts
│   └── users/
│       ├── src/
│       │   ├── hooks/
│       │   │   └── services/
│       └── index.ts
```

**Pattern 2: Tách UI và Data Logic**

```typescript
// libs/features/products/src/hooks/useProducts.ts
// Data-only logic
import { useQuery } from '@tanstack/react-query';
import { productsApi } from '../services/productsApi';

export function useProducts() {
  return useQuery({
      queryKey: ['products'],
      queryFn: () => productsApi.getAll().then(r => r.data)
    });
}

// apps/my-app/src/pages/ProductListPage.tsx
// UI-only logic
import { useProducts } from '@workspace/features-products';
import { ProductList } from '@workspace/ui';

export function ProductListPage() {
  const { data: products, isLoading, error } = useProducts();
  
  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  
  return <ProductList products={products} />;
}
```

---

## Best Practices cho Advanced Patterns

1. **Monitor Cache Performance**
   - Theo dõi cache hit/miss rate
   - Optimize inputs/outputs để tăng hit rate

2. **Use Distributed Caching**
   - Enable Nx Cloud cho team collaboration
   - Chia sẻ cache giữa team members

3. **Customize Executors judiciously**
   - Chỉ tạo custom executor khi cần thiết
   - Ưu tiên sử dụng built-in executors

4. **Test Integration Points**
   - Test API integration với MSW
   - Test state management với realistic scenarios

5. **Organize Code by Feature**
   - Group hooks, services, components theo feature
   - Tách UI và data logic

6. **Use Generators for Consistency**
   - Tạo generators cho CRUD operations
   - Maintain consistent code structure

7. **Profile Performance**
   - Sử dụng `nx report` để xem bottlenecks
   - Optimize tasks tốn nhiều thời gian

8. **Document Custom Patterns**
   - Comment trong custom executors
   - Write documentation cho custom generators

9. **Leverage Nx Cloud**
   - Sử dụng remote caching cho large teams
   - Enable distributed execution cho CI/CD

10. **Keep Dependencies Minimal**
    - Chỉ add dependencies khi cần thiết
    - Review và clean up dependencies thường xuyên
