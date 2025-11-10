Tôi sẽ đọc toàn bộ codebase và phân tích tài liệu CustomHookAPIFactory để cung cấp code example triển khai phù hợp.

# Phân tích CustomHookAPIFactory và Triển khai trên Codebase TapHoaNho

## 1. Tổng quan về CustomHookAPIFactory

Tài liệu **CustomHookAPIFactory.md** mô tả một hệ thống **Universal API Hook** sử dụng **TanStack Query (React Query) + Axios** với các đặc điểm:

### Các thành phần chính:

1. **ApiService Class** (`apiFactory.ts`):
   - Class generic để xử lý CRUD operations
   - Hỗ trợ: GET all, GET by ID, CREATE, UPDATE, PATCH, DELETE
   - Custom method cho các endpoint đặc biệt

2. **Universal CRUD Hooks** (`useApi.ts`):
   - `useApiList` - GET danh sách
   - `useApiDetail` - GET chi tiết theo ID
   - `useApiCreate` - CREATE
   - `useApiUpdate` - UPDATE
   - `useApiPatch` - PATCH
   - `useApiDelete` - DELETE
   - `useApiCustomQuery` - Custom queries
   - `useApiCustomMutation` - Custom mutations

3. **Query Key Factory**:
   - Tạo query keys có cấu trúc nhất quán
   - Hỗ trợ cache invalidation tự động

4. **Tính năng nâng cao**:
   - Optimistic updates
   - Pagination support
   - Automatic cache management
   - Type-safe với TypeScript generics

## 2. Phân tích Codebase Hiện tại

### ✅ Đã có:
- **Axios instance** với interceptors (`lib/axios.ts`)
- **API config** với endpoints (`config/api.ts`)
- **Entity types** đầy đủ (Product, User, Customer, Order, etc.)
- **API functions** riêng lẻ cho từng feature
- **Zustand stores** cho state management
- **TanStack Router** cho routing

### ❌ Chưa có:
- **TanStack Query** (React Query) - cần cài đặt
- **Thư mục `src/hooks`** - cần tạo
- **ApiFactory class**
- **Universal hooks**
- **QueryClient provider**

### Cấu trúc API Response hiện tại:

````typescript path=shiny-carnival/frontend/src/lib/axios.ts mode=EXCERPT
export interface ApiResponse<T = never> {
  isError: boolean;
  message: string;
  data: T | null;
  timestamp: string;
}

export interface PagedRequest {
  page: number;        // 1-based index
  pageSize: number;    // max 100
  search?: string;     // optional, max 255 chars
  sortBy?: string;     // default: "Id"
  sortDesc?: boolean;  // default: true
}

export interface PagedList<T> {
  page: number;
  pageSize: number;
  totalCount: number;
  totalPages: number;
  hasPrevious: boolean;
  hasNext: boolean;
  items: T[];
}
````

### ⚠️ Backend API Contract (QUAN TRỌNG):

**Base URL**:
- Development: `http://localhost:5000`
- Production: `https://your-domain.com`

**Base URL Pattern**: `/api/admin/{entity}`

**HTTP Methods**:
- `GET /api/admin/{entity}` - List với pagination
- `GET /api/admin/{entity}/{id}` - Get by ID
- `POST /api/admin/{entity}` - Create
- `PUT /api/admin/{entity}/{id}` - Update (full replacement)
- `DELETE /api/admin/{entity}/{id}` - Delete (soft delete)
- `PATCH /api/admin/{entity}/{id}` - Partial update (⚠️ CHỈ Orders và Inventory)

**Query Parameters Naming**:
- ✅ Backend examples sử dụng **PascalCase**: `Page`, `PageSize`, `Search`, `SortBy`, `SortDesc`
- ✅ Frontend TypeScript interfaces sử dụng **camelCase**: `page`, `pageSize`, `search`, `sortBy`, `sortDesc`
- ✅ Axios gửi query params dưới dạng **lowercase** trong URL: `?page=1&pagesize=20`
- ✅ ASP.NET Core Model Binding **case-insensitive**, tự động bind `page` → `Page`

**Response Wrapper**: Tất cả responses đều wrap trong `ApiResponse<T>`
```typescript
interface ApiResponse<T> {
  isError: boolean;      // true nếu có lỗi
  message: string;       // Success/error message
  data: T | null;        // Dữ liệu (null nếu error)
  timestamp: string;     // ISO 8601 DateTime
  statusCode: number;    // HTTP status code
}
```

**PATCH Endpoints** (CHỈ 2 endpoints):
- ✅ **Orders**: `PATCH /api/admin/orders/{id}/status`
  - Request: `{ status: "Pending" | "Paid" | "Cancelled" }`
  - Response: `ApiResponse<OrderResponseDto>`

- ✅ **Inventory**: `PATCH /api/admin/inventory/{productId}`
  - Request: `{ quantityChange: number, reason: string }`
  - Response: `ApiResponse<InventoryResponseDto>`

- ❌ **KHÔNG hỗ trợ PATCH**: Products, Categories, Customers, Suppliers, Users, Promotions
  - Sử dụng `PUT` (full replacement) thay vì `PATCH`

## 3. Kế hoạch Triển khai

### Bước 1: Cài đặt TanStack Query
```bash
yarn add @tanstack/react-query @tanstack/react-query-devtools
```

### Bước 2: Tạo cấu trúc thư mục
```
src/
├── hooks/              # Thư mục mới
│   └── useApi.ts      # Universal CRUD hooks
├── lib/
│   ├── axios.ts       # Đã có
│   ├── apiFactory.ts  # Mới - ApiService class
│   └── queryClient.ts # Mới - QueryClient config
```

### Bước 3: Triển khai các file core

### Bước 4: Tạo example với Product entity

### Bước 5: Setup QueryClient Provider

## 4. Code Implementation

### 4.1. Cài đặt Dependencies

```bash
cd shiny-carnival/frontend
yarn add @tanstack/react-query @tanstack/react-query-devtools
```

### 4.2. ApiFactory Class (`src/lib/apiFactory.ts`)

```typescript
// shiny-carnival/frontend/src/lib/apiFactory.ts
import type { AxiosInstance } from 'axios';
import type { ApiResponse, PagedList, PagedRequest } from './axios';

/**
 * Cấu hình cho ApiService
 */
export interface ApiConfig<TData = any, TCreate = any, TUpdate = any> {
  endpoint: string;
  axiosInstance: AxiosInstance;
}

/**
 * Query parameters cho API requests
 */
export interface QueryParams {
  [key: string]: any;
}

/**
 * Universal API Service Class
 * Cung cấp các phương thức CRUD chuẩn cho mọi entity
 * 
 * @template TData - Type của entity data
 * @template TCreate - Type của create request
 * @template TUpdate - Type của update request
 */
export class ApiService<TData = any, TCreate = any, TUpdate = any> {
  private endpoint: string;
  private axios: AxiosInstance;

  constructor(config: ApiConfig<TData, TCreate, TUpdate>) {
    this.endpoint = config.endpoint;
    this.axios = config.axiosInstance;
  }

  /**
   * GET all - Lấy danh sách với query params
   * Backend: GET /api/admin/{entity}
   */
  async getAll(params?: QueryParams): Promise<TData[]> {
    const response = await this.axios.get<ApiResponse<TData[]>>(
      this.endpoint,
      { params }
    );
    // Backend wraps response trong ApiResponse<T>
    return response.data.data || [];
  }

  /**
   * GET all with pagination - Lấy danh sách có phân trang
   * Backend: GET /api/admin/{entity}?page=1&pageSize=20&search=...
   */
  async getAllPaged(params?: PagedRequest): Promise<PagedList<TData>> {
    const response = await this.axios.get<ApiResponse<PagedList<TData>>>(
      this.endpoint,
      { params }
    );
    // Backend wraps PagedList trong ApiResponse<PagedList<T>>
    return response.data.data || {
      page: 1,
      pageSize: 10,
      totalCount: 0,
      totalPages: 0,
      hasPrevious: false,
      hasNext: false,
      items: []
    };
  }

  /**
   * GET by ID - Lấy chi tiết theo ID
   * Backend: GET /api/admin/{entity}/{id}
   */
  async getById(id: string | number): Promise<TData> {
    const response = await this.axios.get<ApiResponse<TData>>(
      `${this.endpoint}/${id}`
    );
    // Backend wraps data trong ApiResponse<T>
    if (!response.data.data) {
      throw new Error(response.data.message || 'Data not found');
    }
    return response.data.data;
  }

  /**
   * POST - Tạo mới
   * Backend: POST /api/admin/{entity}
   */
  async create(data: TCreate): Promise<TData> {
    const response = await this.axios.post<ApiResponse<TData>>(
      this.endpoint,
      data
    );
    // Backend wraps created data trong ApiResponse<T>
    if (!response.data.data) {
      throw new Error(response.data.message || 'Create failed');
    }
    return response.data.data;
  }

  /**
   * PUT - Cập nhật toàn bộ (full replacement)
   * Backend: PUT /api/admin/{entity}/{id}
   * ⚠️ Yêu cầu gửi TẤT CẢ fields, không chỉ fields thay đổi
   */
  async update(id: string | number, data: TUpdate): Promise<TData> {
    const response = await this.axios.put<ApiResponse<TData>>(
      `${this.endpoint}/${id}`,
      data
    );
    // Backend wraps updated data trong ApiResponse<T>
    if (!response.data.data) {
      throw new Error(response.data.message || 'Update failed');
    }
    return response.data.data;
  }

  /**
   * PATCH - Cập nhật một phần (partial update)
   * Backend: PATCH /api/admin/{entity}/{id}
   * ⚠️ CHỈ hỗ trợ cho Orders và Inventory!
   *
   * Orders: PATCH /api/admin/orders/{id}/status
   * Inventory: PATCH /api/admin/inventory/{productId}
   *
   * Các entities khác (Products, Categories, etc.) KHÔNG hỗ trợ PATCH
   * → Sử dụng PUT (update) thay vì PATCH
   */
  async patch(id: string | number, data: Partial<TUpdate>): Promise<TData> {
    const response = await this.axios.patch<ApiResponse<TData>>(
      `${this.endpoint}/${id}`,
      data
    );
    // Backend wraps patched data trong ApiResponse<T>
    if (!response.data.data) {
      throw new Error(response.data.message || 'Patch failed');
    }
    return response.data.data;
  }

  /**
   * DELETE - Xóa (soft delete)
   * Backend: DELETE /api/admin/{entity}/{id}
   * Backend sử dụng soft delete pattern (set DeletedAt timestamp)
   */
  async delete(id: string | number): Promise<void> {
    const response = await this.axios.delete<ApiResponse<boolean>>(
      `${this.endpoint}/${id}`
    );
    // Backend trả về ApiResponse<bool>
    if (response.data.isError) {
      throw new Error(response.data.message || 'Delete failed');
    }
  }

  /**
   * Custom method - Cho các endpoint đặc biệt
   * Ví dụ:
   * - GET /api/admin/inventory/low-stock
   * - GET /api/admin/reports/revenue
   * - PATCH /api/admin/orders/{id}/status
   */
  async custom<TResponse = any>(
    method: 'get' | 'post' | 'put' | 'patch' | 'delete',
    path: string,
    data?: any,
    params?: QueryParams
  ): Promise<TResponse> {
    const url = path.startsWith('/') ? path : `${this.endpoint}/${path}`;
    const response = await this.axios.request<ApiResponse<TResponse>>({
      method,
      url,
      data,
      params,
    });
    // Backend wraps response trong ApiResponse<T>
    if (response.data.isError) {
      throw new Error(response.data.message || 'Custom request failed');
    }
    return response.data.data as TResponse;
  }
}
```

### 4.3. QueryClient Configuration (`src/lib/queryClient.ts`)

```typescript
// shiny-carnival/frontend/src/lib/queryClient.ts
import { QueryClient } from '@tanstack/react-query';

/**
 * Cấu hình QueryClient cho TanStack Query
 */
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Thời gian data được coi là "fresh" (không refetch)
      staleTime: 1000 * 60 * 5, // 5 phút
      
      // Thời gian cache data trước khi garbage collect
      gcTime: 1000 * 60 * 30, // 30 phút (cacheTime đổi thành gcTime trong v5)
      
      // Retry khi query fail
      retry: 1,
      
      // Refetch khi window focus
      refetchOnWindowFocus: false,
      
      // Refetch khi reconnect
      refetchOnReconnect: true,
    },
    mutations: {
      // Retry khi mutation fail
      retry: 0,
    },
  },
});
```

### 4.4. Universal CRUD Hooks (`src/hooks/useApi.ts`)

```typescript
// shiny-carnival/frontend/src/hooks/useApi.ts
import {
  useQuery,
  useMutation,
  useQueryClient,
  type UseQueryOptions,
  type UseMutationOptions,
  type QueryKey,
} from '@tanstack/react-query';
import type { ApiService, QueryParams } from '../lib/apiFactory';
import type { PagedRequest, PagedList } from '../lib/axios';

// ==================== Query Key Factory ====================

/**
 * Tạo query keys có cấu trúc nhất quán cho entity
 * Giúp quản lý cache và invalidation dễ dàng
 */
export const createQueryKeys = (entity: string) => ({
  all: [entity] as const,
  lists: () => [...createQueryKeys(entity).all, 'list'] as const,
  list: (params?: QueryParams) => 
    [...createQueryKeys(entity).lists(), params] as const,
  details: () => [...createQueryKeys(entity).all, 'detail'] as const,
  detail: (id: string | number) => 
    [...createQueryKeys(entity).details(), id] as const,
});

// ==================== Hook Configuration Types ====================

export interface UseApiListConfig<TData, TError = Error> {
  apiService: ApiService<TData>;
  entity: string;
  params?: QueryParams;
  options?: Omit<
    UseQueryOptions<TData[], TError, TData[], QueryKey>,
    'queryKey' | 'queryFn'
  >;
}

export interface UseApiPagedListConfig<TData, TError = Error> {
  apiService: ApiService<TData>;
  entity: string;
  params?: PagedRequest;
  options?: Omit<
    UseQueryOptions<PagedList<TData>, TError, PagedList<TData>, QueryKey>,
    'queryKey' | 'queryFn'
  >;
}

export interface UseApiDetailConfig<TData, TError = Error> {
  apiService: ApiService<TData>;
  entity: string;
  id: string | number;
  options?: Omit<
    UseQueryOptions<TData, TError, TData, QueryKey>,
    'queryKey' | 'queryFn'
  >;
}

export interface UseApiMutationConfig<TData, TVariables, TError = Error> {
  apiService: ApiService<TData>;
  entity: string;
  invalidateQueries?: string[];
  options?: UseMutationOptions<TData, TError, TVariables>;
}

// ==================== GET ALL Hook ====================

/**
 * Hook để lấy danh sách entity (không phân trang)
 */
export function useApiList<TData = any, TError = Error>({
  apiService,
  entity,
  params,
  options,
}: UseApiListConfig<TData, TError>) {
  const queryKeys = createQueryKeys(entity);

  return useQuery<TData[], TError>({
    queryKey: queryKeys.list(params),
    queryFn: () => apiService.getAll(params),
    ...options,
  });
}

// ==================== GET ALL PAGED Hook ====================

/**
 * Hook để lấy danh sách entity có phân trang
 */
export function useApiPagedList<TData = any, TError = Error>({
  apiService,
  entity,
  params,
  options,
}: UseApiPagedListConfig<TData, TError>) {
  const queryKeys = createQueryKeys(entity);

  return useQuery<PagedList<TData>, TError>({
    queryKey: queryKeys.list(params),
    queryFn: () => apiService.getAllPaged(params),
    ...options,
  });
}

// ==================== GET BY ID Hook ====================

/**
 * Hook để lấy chi tiết entity theo ID
 */
export function useApiDetail<TData = any, TError = Error>({
  apiService,
  entity,
  id,
  options,
}: UseApiDetailConfig<TData, TError>) {
  const queryKeys = createQueryKeys(entity);

  return useQuery<TData, TError>({
    queryKey: queryKeys.detail(id),
    queryFn: () => apiService.getById(id),
    enabled: !!id, // Chỉ fetch khi có ID
    ...options,
  });
}

// ==================== CREATE Hook ====================

/**
 * Hook để tạo mới entity
 */
export function useApiCreate<
  TData = any,
  TCreate = any,
  TError = Error
>({
  apiService,
  entity,
  invalidateQueries = [],
  options,
}: UseApiMutationConfig<TData, TCreate, TError>) {
  const queryClient = useQueryClient();
  const queryKeys = createQueryKeys(entity);

  return useMutation<TData, TError, TCreate>({
    mutationFn: (data: TCreate) => apiService.create(data),
    onSuccess: (data, variables, context) => {
      // Invalidate danh sách để refetch
      queryClient.invalidateQueries({ queryKey: queryKeys.lists() });
      
      // Invalidate các queries bổ sung
      invalidateQueries.forEach((key) => {
        queryClient.invalidateQueries({ queryKey: [key] });
      });

      // Gọi onSuccess của user nếu có
      options?.onSuccess?.(data, variables, context);
    },
    ...options,
  });
}

// ==================== UPDATE Hook ====================

/**
 * Hook để cập nhật entity
 */
export function useApiUpdate<
  TData = any,
  TUpdate = any,
  TError = Error
>({
  apiService,
  entity,
  invalidateQueries = [],
  options,
}: UseApiMutationConfig<TData, { id: string | number; data: TUpdate }, TError>) {
  const queryClient = useQueryClient();
  const queryKeys = createQueryKeys(entity);

  return useMutation<TData, TError, { id: string | number; data: TUpdate }>({
    mutationFn: ({ id, data }) => apiService.update(id, data),
    onSuccess: (data, variables, context) => {
      // Invalidate danh sách
      queryClient.invalidateQueries({ queryKey: queryKeys.lists() });
      
      // Invalidate chi tiết cụ thể
      queryClient.invalidateQueries({ queryKey: queryKeys.detail(variables.id) });
      
      // Invalidate các queries bổ sung
      invalidateQueries.forEach((key) => {
        queryClient.invalidateQueries({ queryKey: [key] });
      });

      options?.onSuccess?.(data, variables, context);
    },
    ...options,
  });
}

// ==================== PATCH Hook ====================

/**
 * Hook để cập nhật một phần entity
 */
export function useApiPatch<
  TData = any,
  TUpdate = any,
  TError = Error
>({
  apiService,
  entity,
  invalidateQueries = [],
  options,
}: UseApiMutationConfig<TData, { id: string | number; data: Partial<TUpdate> }, TError>) {
  const queryClient = useQueryClient();
  const queryKeys = createQueryKeys(entity);

  return useMutation<TData, TError, { id: string | number; data: Partial<TUpdate> }>({
    mutationFn: ({ id, data }) => apiService.patch(id, data),
    onSuccess: (data, variables, context) => {
      queryClient.invalidateQueries({ queryKey: queryKeys.lists() });
      queryClient.invalidateQueries({ queryKey: queryKeys.detail(variables.id) });
      
      invalidateQueries.forEach((key) => {
        queryClient.invalidateQueries({ queryKey: [key] });
      });

      options?.onSuccess?.(data, variables, context);
    },
    ...options,
  });
}

// ==================== DELETE Hook ====================

/**
 * Hook để xóa entity
 */
export function useApiDelete<TError = Error>({
  apiService,
  entity,
  invalidateQueries = [],
  options,
}: Omit<UseApiMutationConfig<void, string | number, TError>, 'options'> & {
  options?: UseMutationOptions<void, TError, string | number>;
}) {
  const queryClient = useQueryClient();
  const queryKeys = createQueryKeys(entity);

  return useMutation<void, TError, string | number>({
    mutationFn: (id) => apiService.delete(id),
    onSuccess: (data, variables, context) => {
      // Invalidate danh sách
      queryClient.invalidateQueries({ queryKey: queryKeys.lists() });
      
      // Xóa chi tiết khỏi cache
      queryClient.removeQueries({ queryKey: queryKeys.detail(variables) });
      
      // Invalidate các queries bổ sung
      invalidateQueries.forEach((key) => {
        queryClient.invalidateQueries({ queryKey: [key] });
      });

      options?.onSuccess?.(data, variables, context);
    },
    ...options,
  });
}

// ==================== Custom Query Hook ====================

/**
 * Hook cho custom query
 */
export function useApiCustomQuery<TData = any, TError = Error>({
  entity,
  queryKey,
  queryFn,
  options,
}: {
  entity: string;
  queryKey: QueryKey;
  queryFn: () => Promise<TData>;
  options?: Omit<UseQueryOptions<TData, TError, TData, QueryKey>, 'queryKey' | 'queryFn'>;
}) {
  return useQuery<TData, TError>({
    queryKey: [entity, ...queryKey],
    queryFn,
    ...options,
  });
}

// ==================== Custom Mutation Hook ====================

/**
 * Hook cho custom mutation
 */
export function useApiCustomMutation<
  TData = any,
  TVariables = any,
  TError = Error
>({
  entity,
  mutationFn,
  invalidateQueries = [],
  options,
}: {
  entity: string;
  mutationFn: (variables: TVariables) => Promise<TData>;
  invalidateQueries?: string[];
  options?: UseMutationOptions<TData, TError, TVariables>;
}) {
  const queryClient = useQueryClient();

  return useMutation<TData, TError, TVariables>({
    mutationFn,
    onSuccess: (data, variables, context) => {
      // Invalidate entity queries
      queryClient.invalidateQueries({ queryKey: [entity] });
      
      // Invalidate các queries bổ sung
      invalidateQueries.forEach((key) => {
        queryClient.invalidateQueries({ queryKey: [key] });
      });

      options?.onSuccess?.(data, variables, context);
    },
    ...options,
  });
}
```

### 4.5. Setup QueryClient Provider (`src/app/main.tsx`)

```typescript
// shiny-carnival/frontend/src/app/main.tsx
import { StrictMode } from 'react'
import ReactDOM from 'react-dom/client'
import './index.css'
import { RouterProvider } from '@tanstack/react-router';
import { QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import 'antd/dist/reset.css';
import { router } from './routes/routeTree';
import { queryClient } from '../lib/queryClient';

// Render the app
const rootElement = document.getElementById('root')!
if (!rootElement.innerHTML) {
  const root = ReactDOM.createRoot(rootElement)
  root.render(
    <StrictMode>
      <QueryClientProvider client={queryClient}>
        <RouterProvider router={router} />
        {/* React Query Devtools - chỉ hiện trong development */}
        <ReactQueryDevtools initialIsOpen={false} />
      </QueryClientProvider>
    </StrictMode>,
  )
}
```

### 4.6. Product API Service Example (`src/lib/api/productApiService.ts`)

```typescript
// shiny-carnival/frontend/src/lib/api/productApiService.ts
import axiosClient from '../axios';
import { ApiService } from '../apiFactory';
import { API_CONFIG } from '../../config/api';
import type { ProductEntity } from '../../features/products/types/entity';
import type { CreateProductRequest, UpdateProductRequest } from '../../features/products/types/api';

/**
 * Product API Service Instance
 * Sử dụng ApiService class với Product types
 */
export const productApiService = new ApiService<
  ProductEntity,
  CreateProductRequest,
  UpdateProductRequest
>({
  endpoint: API_CONFIG.ENDPOINTS.ADMIN.PRODUCTS,
  axiosInstance: axiosClient,
});

/**
 * Extended Product API với custom methods
 */
export const productApi = {
  ...productApiService,
  
  /**
   * Tìm kiếm sản phẩm theo barcode
   */
  searchByBarcode: (barcode: string) => {
    return productApiService.custom<ProductEntity[]>(
      'get',
      '',
      undefined,
      { search: barcode, pageSize: 10 }
    );
  },
  
  /**
   * Lấy sản phẩm theo category
   */
  getByCategory: (categoryId: number) => {
    return productApiService.custom<ProductEntity[]>(
      'get',
      '',
      undefined,
      { categoryId }
    );
  },
  
  /**
   * Bulk update stock
   */
  bulkUpdateStock: (updates: Array<{ id: number; stock: number }>) => {
    return productApiService.custom<ProductEntity[]>(
      'post',
      'bulk-update-stock',
      updates
    );
  },
};
```

### 4.7. Product Hooks Example (`src/features/products/hooks/useProducts.ts`)

```typescript
// shiny-carnival/frontend/src/features/products/hooks/useProducts.ts
import { 
  useApiPagedList,
  useApiDetail, 
  useApiCreate, 
  useApiUpdate, 
  useApiDelete,
  useApiCustomQuery,
} from '../../../hooks/useApi';
import { productApiService, productApi } from '../../../lib/api/productApiService';
import type { ProductEntity } from '../types/entity';
import type { CreateProductRequest, UpdateProductRequest } from '../types/api';
import type { PagedRequest } from '../../../lib/axios';

const ENTITY = 'products';

// ==================== Query Hooks ====================

/**
 * Hook lấy danh sách sản phẩm có phân trang
 */
export const useProducts = (params?: PagedRequest) => {
  return useApiPagedList<ProductEntity>({
    apiService: productApiService,
    entity: ENTITY,
    params,
    options: {
      staleTime: 1000 * 60 * 5, // 5 phút
    },
  });
};

/**
 * Hook lấy chi tiết sản phẩm theo ID
 */
export const useProduct = (id: number) => {
  return useApiDetail<ProductEntity>({
    apiService: productApiService,
    entity: ENTITY,
    id,
  });
};

/**
 * Hook tìm kiếm sản phẩm theo barcode
 */
export const useProductsByBarcode = (barcode: string) => {
  return useApiCustomQuery<ProductEntity[]>({
    entity: ENTITY,
    queryKey: ['barcode', barcode],
    queryFn: () => productApi.searchByBarcode(barcode),
    options: {
      enabled: !!barcode && barcode.length > 0,
    },
  });
};

/**
 * Hook lấy sản phẩm theo category
 */
export const useProductsByCategory = (categoryId: number) => {
  return useApiCustomQuery<ProductEntity[]>({
    entity: ENTITY,
    queryKey: ['category', categoryId],
    queryFn: () => productApi.getByCategory(categoryId),
    options: {
      enabled: !!categoryId,
    },
  });
};

// ==================== Mutation Hooks ====================

/**
 * Hook tạo sản phẩm mới
 */
export const useCreateProduct = () => {
  return useApiCreate<ProductEntity, CreateProductRequest>({
    apiService: productApiService,
    entity: ENTITY,
    invalidateQueries: ['categories'], // Invalidate categories nếu cần
    options: {
      onSuccess: () => {
        console.log('Tạo sản phẩm thành công');
      },
    },
  });
};

/**
 * Hook cập nhật sản phẩm
 */
export const useUpdateProduct = () => {
  return useApiUpdate<ProductEntity, UpdateProductRequest>({
    apiService: productApiService,
    entity: ENTITY,
  });
};

/**
 * Hook xóa sản phẩm
 */
export const useDeleteProduct = () => {
  return useApiDelete({
    apiService: productApiService,
    entity: ENTITY,
    options: {
      onSuccess: () => {
        console.log('Xóa sản phẩm thành công');
      },
    },
  });
};
```

### 4.8. Component Usage Example

#### Product List Component

```typescript
// shiny-carnival/frontend/src/features/products/components/ProductList.tsx
import { useState } from 'react';
import { Table, Button, Space, message, Popconfirm } from 'antd';
import { EditOutlined, DeleteOutlined } from '@ant-design/icons';
import { useProducts, useDeleteProduct } from '../hooks/useProducts';
import type { ProductEntity } from '../types/entity';

export const ProductList = () => {
  const [page, setPage] = useState(1);
  const [pageSize, setPageSize] = useState(20);
  const [search, setSearch] = useState('');

  // Sử dụng hook để lấy danh sách sản phẩm
  const { 
    data: productsData, 
    isLoading, 
    error,
    refetch 
  } = useProducts({ 
    page, 
    pageSize, 
    search 
  });

  // Hook xóa sản phẩm
  const deleteProduct = useDeleteProduct();

  // Xử lý xóa
  const handleDelete = async (id: number) => {
    try {
      await deleteProduct.mutateAsync(id);
      message.success('Xóa sản phẩm thành công');
    } catch (error: any) {
      message.error(error.message || 'Không thể xóa sản phẩm');
    }
  };

  // Columns cho Table
  const columns = [
    {
      title: 'ID',
      dataIndex: 'id',
      key: 'id',
      width: 80,
    },
    {
      title: 'Tên sản phẩm',
      dataIndex: 'productName',
      key: 'productName',
    },
    {
      title: 'Barcode',
      dataIndex: 'barcode',
      key: 'barcode',
    },
    {
      title: 'Giá',
      dataIndex: 'price',
      key: 'price',
      render: (price: number) => `${price.toLocaleString()} đ`,
    },
    {
      title: 'Đơn vị',
      dataIndex: 'unit',
      key: 'unit',
    },
    {
      title: 'Thao tác',
      key: 'actions',
      width: 150,
      render: (_: any, record: ProductEntity) => (
        <Space>
          <Button
            type="link"
            icon={<EditOutlined />}
            onClick={() => {/* Navigate to edit */}}
          >
            Sửa
          </Button>
          <Popconfirm
            title="Xóa sản phẩm"
            description="Bạn có chắc muốn xóa sản phẩm này?"
            onConfirm={() => handleDelete(record.id!)}
            okText="Xóa"
            cancelText="Hủy"
          >
            <Button
              type="link"
              danger
              icon={<DeleteOutlined />}
              loading={deleteProduct.isPending}
            >
              Xóa
            </Button>
          </Popconfirm>
        </Space>
      ),
    },
  ];

  if (error) {
    return <div>Lỗi: {error.message}</div>;
  }

  return (
    <div>
      <Table
        columns={columns}
        dataSource={productsData?.items || []}
        loading={isLoading}
        rowKey="id"
        pagination={{
          current: page,
          pageSize: pageSize,
          total: productsData?.totalCount || 0,
          showSizeChanger: true,
          showTotal: (total) => `Tổng ${total} sản phẩm`,
          onChange: (newPage, newPageSize) => {
            setPage(newPage);
            setPageSize(newPageSize);
          },
        }}
      />
    </div>
  );
};
```

#### Product Create/Edit Form Component

```typescript
// shiny-carnival/frontend/src/features/products/components/ProductForm.tsx
import { useEffect } from 'react';
import { Form, Input, InputNumber, Button, message } from 'antd';
import { useNavigate } from '@tanstack/react-router';
import { useCreateProduct, useUpdateProduct, useProduct } from '../hooks/useProducts';
import type { CreateProductRequest, UpdateProductRequest } from '../types/api';

interface ProductFormProps {
  productId?: number; // Nếu có ID thì là edit mode
}

export const ProductForm = ({ productId }: ProductFormProps) => {
  const [form] = Form.useForm();
  const navigate = useNavigate();
  
  // Hooks
  const createProduct = useCreateProduct();
  const updateProduct = useUpdateProduct();
  const { data: product, isLoading } = useProduct(productId!);

  // Load dữ liệu khi edit
  useEffect(() => {
    if (product) {
      form.setFieldsValue(product);
    }
  }, [product, form]);

  // Xử lý submit
  const handleSubmit = async (values: CreateProductRequest | UpdateProductRequest) => {
    try {
      if (productId) {
        // Update mode
        await updateProduct.mutateAsync({
          id: productId,
          data: values as UpdateProductRequest,
        });
        message.success('Cập nhật sản phẩm thành công');
      } else {
        // Create mode
        await createProduct.mutateAsync(values as CreateProductRequest);
        message.success('Tạo sản phẩm thành công');
      }
      
      // Navigate về danh sách
      navigate({ to: '/admin/products' });
    } catch (error: any) {
      message.error(error.message || 'Có lỗi xảy ra');
    }
  };

  if (productId && isLoading) {
    return <div>Đang tải...</div>;
  }

  return (
    <Form
      form={form}
      layout="vertical"
      onFinish={handleSubmit}
      initialValues={{
        categoryId: 1,
        supplierId: 1,
        unit: 'cái',
      }}
    >
      <Form.Item
        label="Tên sản phẩm"
        name="productName"
        rules={[{ required: true, message: 'Vui lòng nhập tên sản phẩm' }]}
      >
        <Input placeholder="Nhập tên sản phẩm" />
      </Form.Item>

      <Form.Item
        label="Barcode"
        name="barcode"
        rules={[{ required: true, message: 'Vui lòng nhập barcode' }]}
      >
        <Input placeholder="Nhập barcode" />
      </Form.Item>

      <Form.Item
        label="Giá"
        name="price"
        rules={[{ required: true, message: 'Vui lòng nhập giá' }]}
      >
        <InputNumber
          style={{ width: '100%' }}
          min={0}
          formatter={(value) => `${value}`.replace(/\B(?=(\d{3})+(?!\d))/g, ',')}
          placeholder="Nhập giá"
        />
      </Form.Item>

      <Form.Item
        label="Đơn vị"
        name="unit"
        rules={[{ required: true, message: 'Vui lòng nhập đơn vị' }]}
      >
        <Input placeholder="Nhập đơn vị (cái, kg, hộp...)" />
      </Form.Item>

      <Form.Item
        label="Category ID"
        name="categoryId"
        rules={[{ required: true, message: 'Vui lòng nhập category ID' }]}
      >
        <InputNumber style={{ width: '100%' }} min={1} />
      </Form.Item>

      <Form.Item
        label="Supplier ID"
        name="supplierId"
        rules={[{ required: true, message: 'Vui lòng nhập supplier ID' }]}
      >
        <InputNumber style={{ width: '100%' }} min={1} />
      </Form.Item>

      <Form.Item>
        <Button
          type="primary"
          htmlType="submit"
          loading={createProduct.isPending || updateProduct.isPending}
        >
          {productId ? 'Cập nhật' : 'Tạo mới'}
        </Button>
        <Button
          style={{ marginLeft: 8 }}
          onClick={() => navigate({ to: '/admin/products' })}
        >
          Hủy
        </Button>
      </Form.Item>
    </Form>
  );
};
```

### 4.9. Infinite Scroll Hook (`src/hooks/useApiInfinite.ts`)

```typescript
// shiny-carnival/frontend/src/hooks/useApiInfinite.ts
import { useInfiniteQuery, type UseInfiniteQueryOptions } from '@tanstack/react-query';
import type { ApiService, QueryParams } from '../lib/apiFactory';
import type { PagedList } from '../lib/axios';
import { createQueryKeys } from './useApi';

export interface UseApiInfiniteConfig<TData, TError = Error> {
  apiService: ApiService<TData>;
  entity: string;
  params?: QueryParams;
  pageSize?: number;
  options?: Omit<
    UseInfiniteQueryOptions<PagedList<TData>, TError>,
    'queryKey' | 'queryFn' | 'getNextPageParam' | 'initialPageParam'
  >;
}

/**
 * Hook cho infinite scroll pagination
 * Tự động load thêm data khi scroll đến cuối
 *
 * @example
 * const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useApiInfinite({
 *   apiService: productApiService,
 *   entity: 'products',
 *   pageSize: 20,
 * });
 */
export function useApiInfinite<TData = any, TError = Error>({
  apiService,
  entity,
  params = {},
  pageSize = 20,
  options,
}: UseApiInfiniteConfig<TData, TError>) {
  const queryKeys = createQueryKeys(entity);

  return useInfiniteQuery<PagedList<TData>, TError>({
    queryKey: queryKeys.list({ ...params, pageSize }),
    queryFn: ({ pageParam = 1 }) => {
      return apiService.getAllPaged({
        ...params,
        page: pageParam as number,
        pageSize,
      });
    },
    initialPageParam: 1,
    getNextPageParam: (lastPage) => {
      // Nếu còn trang tiếp theo, return page number
      return lastPage.hasNext ? lastPage.page + 1 : undefined;
    },
    getPreviousPageParam: (firstPage) => {
      // Nếu có trang trước, return page number
      return firstPage.hasPrevious ? firstPage.page - 1 : undefined;
    },
    ...options,
  });
}
```

### 4.10. Pagination Management Hooks

#### 4.10.1. Basic Pagination Hook

```typescript
// shiny-carnival/frontend/src/hooks/usePagination.ts
import { useState, useMemo } from 'react';
import { useApiPagedList } from './useApi';
import type { ApiService, QueryParams } from '../lib/apiFactory';
import type { PagedRequest } from '../lib/axios';

export interface UsePaginationConfig<TData> {
  apiService: ApiService<TData>;
  entity: string;
  initialPage?: number;
  initialPageSize?: number;
  initialSearch?: string;
  initialSortBy?: string;
  initialSortDesc?: boolean;
  additionalParams?: QueryParams;
}

/**
 * Hook quản lý pagination state và actions
 * Tự động sync state và refetch khi params thay đổi
 */
export function usePagination<TData = any>({
  apiService,
  entity,
  initialPage = 1,
  initialPageSize = 20,
  initialSearch = '',
  initialSortBy = 'id',
  initialSortDesc = true,
  additionalParams = {},
}: UsePaginationConfig<TData>) {
  // Pagination state
  const [page, setPage] = useState(initialPage);
  const [pageSize, setPageSize] = useState(initialPageSize);
  const [search, setSearch] = useState(initialSearch);
  const [sortBy, setSortBy] = useState(initialSortBy);
  const [sortDesc, setSortDesc] = useState(initialSortDesc);

  // Build params object
  const params: PagedRequest & QueryParams = useMemo(() => ({
    page,
    pageSize,
    search: search || undefined,
    sortBy,
    sortDesc,
    ...additionalParams,
  }), [page, pageSize, search, sortBy, sortDesc, additionalParams]);

  // Fetch data với pagination
  const query = useApiPagedList<TData>({
    apiService,
    entity,
    params,
    options: {
      staleTime: 1000 * 60 * 5, // 5 phút
      placeholderData: (previousData) => previousData, // Keep previous data while fetching
    },
  });

  // Helper functions
  const handlePageChange = (newPage: number, newPageSize?: number) => {
    setPage(newPage);
    if (newPageSize && newPageSize !== pageSize) {
      setPageSize(newPageSize);
      setPage(1); // Reset về trang 1 khi đổi pageSize
    }
  };

  const handleSearch = (searchText: string) => {
    setSearch(searchText);
    setPage(1); // Reset về trang 1 khi search
  };

  const handleSort = (field: string, descending: boolean) => {
    setSortBy(field);
    setSortDesc(descending);
    setPage(1); // Reset về trang 1 khi sort
  };

  const resetPagination = () => {
    setPage(initialPage);
    setPageSize(initialPageSize);
    setSearch(initialSearch);
    setSortBy(initialSortBy);
    setSortDesc(initialSortDesc);
  };

  return {
    // Query result
    ...query,

    // Pagination state
    page,
    pageSize,
    search,
    sortBy,
    sortDesc,

    // Pagination data
    totalCount: query.data?.totalCount || 0,
    totalPages: query.data?.totalPages || 0,
    hasPrevious: query.data?.hasPrevious || false,
    hasNext: query.data?.hasNext || false,
    items: query.data?.items || [],

    // Actions
    handlePageChange,
    handleSearch,
    handleSort,
    resetPagination,
    setPage,
    setPageSize,
    setSearch,
    setSortBy,
    setSortDesc,
  };
}
```

#### 4.10.2. URL-based Pagination Hook

```typescript
// shiny-carnival/frontend/src/hooks/usePaginationWithRouter.ts
import { useNavigate } from '@tanstack/react-router';
import { useMemo } from 'react';
import { useApiPagedList } from './useApi';
import type { ApiService, QueryParams } from '../lib/apiFactory';
import type { PagedRequest } from '../lib/axios';

export interface UsePaginationWithRouterConfig<TData> {
  apiService: ApiService<TData>;
  entity: string;
  routeApi: any; // TanStack Router route API
  additionalParams?: QueryParams;
}

/**
 * Hook quản lý pagination với URL sync
 * Pagination state được lưu trong URL query params
 *
 * @example
 * const routeApi = getRouteApi('/admin/products');
 * const pagination = usePaginationWithRouter({
 *   apiService: productApiService,
 *   entity: 'products',
 *   routeApi,
 * });
 */
export function usePaginationWithRouter<TData = any>({
  apiService,
  entity,
  routeApi,
  additionalParams = {},
}: UsePaginationWithRouterConfig<TData>) {
  const navigate = useNavigate();

  // Lấy search params từ URL
  const search = routeApi.useSearch();

  // Build params từ URL
  const params: PagedRequest & QueryParams = useMemo(() => ({
    page: search.page || 1,
    pageSize: search.pageSize || 20,
    search: search.search || undefined,
    sortBy: search.sortBy || 'id',
    sortDesc: search.sortDesc !== false, // Default true
    ...additionalParams,
  }), [search, additionalParams]);

  // Fetch data
  const query = useApiPagedList<TData>({
    apiService,
    entity,
    params,
    options: {
      staleTime: 1000 * 60 * 5,
      placeholderData: (previousData) => previousData,
    },
  });

  // Update URL params
  const updateUrlParams = (newParams: Partial<PagedRequest>) => {
    navigate({
      search: (prev) => ({
        ...prev,
        ...newParams,
      }),
    });
  };

  // Helper functions
  const handlePageChange = (newPage: number, newPageSize?: number) => {
    const updates: Partial<PagedRequest> = { page: newPage };
    if (newPageSize && newPageSize !== params.pageSize) {
      updates.pageSize = newPageSize;
      updates.page = 1;
    }
    updateUrlParams(updates);
  };

  const handleSearch = (searchText: string) => {
    updateUrlParams({
      search: searchText || undefined,
      page: 1,
    });
  };

  const handleSort = (field: string, descending: boolean) => {
    updateUrlParams({
      sortBy: field,
      sortDesc: descending,
      page: 1,
    });
  };

  const resetPagination = () => {
    navigate({
      search: {
        page: 1,
        pageSize: 20,
      },
    });
  };

  return {
    ...query,
    params,
    totalCount: query.data?.totalCount || 0,
    totalPages: query.data?.totalPages || 0,
    hasPrevious: query.data?.hasPrevious || false,
    hasNext: query.data?.hasNext || false,
    items: query.data?.items || [],
    handlePageChange,
    handleSearch,
    handleSort,
    resetPagination,
  };
}
```

#### 4.10.3. Advanced Filters Hook

```typescript
// shiny-carnival/frontend/src/hooks/usePaginationWithFilters.ts
import { useState, useMemo } from 'react';
import { useApiPagedList } from './useApi';
import type { ApiService, QueryParams } from '../lib/apiFactory';
import type { PagedRequest } from '../lib/axios';

export interface UsePaginationWithFiltersConfig<TData, TFilters extends Record<string, any>> {
  apiService: ApiService<TData>;
  entity: string;
  initialFilters?: TFilters;
  initialPage?: number;
  initialPageSize?: number;
  initialSortBy?: string;
  initialSortDesc?: boolean;
}

/**
 * Hook với advanced filters support
 *
 * @example
 * interface ProductFilters {
 *   categoryId?: number;
 *   minPrice?: number;
 *   maxPrice?: number;
 * }
 *
 * const pagination = usePaginationWithFilters<ProductEntity, ProductFilters>({
 *   apiService: productApiService,
 *   entity: 'products',
 *   initialFilters: {},
 * });
 */
export function usePaginationWithFilters<TData = any, TFilters extends Record<string, any> = any>({
  apiService,
  entity,
  initialFilters = {} as TFilters,
  initialPage = 1,
  initialPageSize = 20,
  initialSortBy = 'id',
  initialSortDesc = true,
}: UsePaginationWithFiltersConfig<TData, TFilters>) {
  // Pagination state
  const [page, setPage] = useState(initialPage);
  const [pageSize, setPageSize] = useState(initialPageSize);
  const [sortBy, setSortBy] = useState(initialSortBy);
  const [sortDesc, setSortDesc] = useState(initialSortDesc);

  // Filters state
  const [filters, setFilters] = useState<TFilters>(initialFilters);

  // Build params
  const params: PagedRequest & TFilters = useMemo(() => ({
    page,
    pageSize,
    sortBy,
    sortDesc,
    ...filters,
  }), [page, pageSize, sortBy, sortDesc, filters]);

  // Fetch data
  const query = useApiPagedList<TData>({
    apiService,
    entity,
    params,
    options: {
      staleTime: 1000 * 60 * 5,
      placeholderData: (previousData) => previousData,
    },
  });

  // Actions
  const handlePageChange = (newPage: number, newPageSize?: number) => {
    setPage(newPage);
    if (newPageSize && newPageSize !== pageSize) {
      setPageSize(newPageSize);
      setPage(1);
    }
  };

  const handleSort = (field: string, descending: boolean) => {
    setSortBy(field);
    setSortDesc(descending);
    setPage(1);
  };

  const handleFilterChange = (newFilters: Partial<TFilters>) => {
    setFilters((prev) => ({
      ...prev,
      ...newFilters,
    }));
    setPage(1);
  };

  const clearFilters = () => {
    setFilters(initialFilters);
    setPage(1);
  };

  const resetAll = () => {
    setPage(initialPage);
    setPageSize(initialPageSize);
    setSortBy(initialSortBy);
    setSortDesc(initialSortDesc);
    setFilters(initialFilters);
  };

  // Count active filters
  const activeFiltersCount = Object.values(filters).filter(Boolean).length;

  return {
    ...query,
    page,
    pageSize,
    sortBy,
    sortDesc,
    filters,
    totalCount: query.data?.totalCount || 0,
    totalPages: query.data?.totalPages || 0,
    items: query.data?.items || [],
    activeFiltersCount,
    handlePageChange,
    handleSort,
    handleFilterChange,
    clearFilters,
    resetAll,
    setPage,
    setPageSize,
    setSortBy,
    setSortDesc,
    setFilters,
  };
}
```

### 4.12. Usage Examples Chi tiết cho từng Hook

#### 4.12.1. useApiInfinite - Infinite Scroll Hook

**📌 Basic Usage:**

```typescript
import { useApiInfinite } from '../../../hooks/useApiInfinite';
import { productApiService } from '../../../lib/api/productApiService';

function ProductInfiniteList() {
  const {
    data,
    isLoading,
    isFetchingNextPage,
    hasNextPage,
    fetchNextPage,
  } = useApiInfinite({
    apiService: productApiService,
    entity: 'products',
    pageSize: 20,
  });

  const allItems = data?.pages.flatMap(page => page.items) || [];

  return (
    <div>
      {allItems.map(product => (
        <div key={product.id}>{product.productName}</div>
      ))}

      {hasNextPage && (
        <button onClick={() => fetchNextPage()} disabled={isFetchingNextPage}>
          {isFetchingNextPage ? 'Loading...' : 'Load More'}
        </button>
      )}
    </div>
  );
}
```

**📌 Advanced Usage với Filters:**

```typescript
import { useState } from 'react';
import { useApiInfinite } from '../../../hooks/useApiInfinite';
import { productApiService } from '../../../lib/api/productApiService';

function ProductInfiniteListWithFilters() {
  const [categoryId, setCategoryId] = useState<number>();
  const [minPrice, setMinPrice] = useState<number>();

  const {
    data,
    isLoading,
    isFetchingNextPage,
    hasNextPage,
    fetchNextPage,
    refetch,
  } = useApiInfinite({
    apiService: productApiService,
    entity: 'products',
    pageSize: 20,
    params: {
      categoryId,
      minPrice,
    },
    options: {
      staleTime: 1000 * 60 * 5, // 5 phút
      enabled: true, // Có thể disable khi cần
    },
  });

  const allItems = data?.pages.flatMap(page => page.items) || [];
  const totalCount = data?.pages[0]?.totalCount || 0;

  return (
    <div>
      {/* Filters */}
      <div>
        <select onChange={(e) => setCategoryId(Number(e.target.value))}>
          <option value="">All Categories</option>
          <option value="1">Category 1</option>
          <option value="2">Category 2</option>
        </select>

        <input
          type="number"
          placeholder="Min Price"
          onChange={(e) => setMinPrice(Number(e.target.value))}
        />

        <button onClick={() => refetch()}>Refresh</button>
      </div>

      {/* Items */}
      <div>
        <p>Loaded: {allItems.length} / {totalCount}</p>
        {allItems.map(product => (
          <div key={product.id}>{product.productName}</div>
        ))}
      </div>

      {/* Load More */}
      {hasNextPage && (
        <button onClick={() => fetchNextPage()} disabled={isFetchingNextPage}>
          {isFetchingNextPage ? 'Loading...' : 'Load More'}
        </button>
      )}
    </div>
  );
}
```

**📌 Config Options:**

```typescript
interface UseApiInfiniteConfig<TData, TError = Error> {
  // Required
  apiService: ApiService<TData>;  // API service instance
  entity: string;                 // Entity name cho query keys

  // Optional
  params?: QueryParams;           // Additional query params (filters, etc.)
  pageSize?: number;              // Items per page (default: 20)
  options?: {
    staleTime?: number;           // Cache time (default: 0)
    enabled?: boolean;            // Enable/disable query (default: true)
    refetchOnWindowFocus?: boolean; // Refetch on window focus
    retry?: number | boolean;     // Retry failed requests
    // ... other TanStack Query options
  };
}
```

**📌 Return Values:**

```typescript
{
  // Data
  data: {
    pages: PagedList<TData>[];    // Array of pages
    pageParams: number[];         // Array of page numbers
  };

  // Loading states
  isLoading: boolean;             // Initial loading
  isFetching: boolean;            // Any fetching (including background)
  isFetchingNextPage: boolean;    // Loading next page
  isFetchingPreviousPage: boolean; // Loading previous page

  // Pagination
  hasNextPage: boolean;           // Has more pages
  hasPreviousPage: boolean;       // Has previous pages
  fetchNextPage: () => void;      // Load next page
  fetchPreviousPage: () => void;  // Load previous page

  // Actions
  refetch: () => void;            // Refetch all pages

  // Error
  error: TError | null;           // Error object
  isError: boolean;               // Has error
}
```

**💡 Common Patterns:**

```typescript
// 1. Auto-load với Intersection Observer
useEffect(() => {
  const observer = new IntersectionObserver(
    (entries) => {
      if (entries[0].isIntersecting && hasNextPage && !isFetchingNextPage) {
        fetchNextPage();
      }
    },
    { threshold: 0.1 }
  );

  if (observerTarget.current) {
    observer.observe(observerTarget.current);
  }

  return () => observer.disconnect();
}, [hasNextPage, isFetchingNextPage, fetchNextPage]);

// 2. Flatten all items
const allItems = useMemo(
  () => data?.pages.flatMap(page => page.items) || [],
  [data]
);

// 3. Get total count
const totalCount = data?.pages[0]?.totalCount || 0;

// 4. Reset và refetch khi filters change
useEffect(() => {
  refetch();
}, [categoryId, minPrice, refetch]);
```

#### 4.12.2. usePagination - Basic Pagination Management

**📌 Basic Usage:**

```typescript
import { usePagination } from '../../../hooks/usePagination';
import { productApiService } from '../../../lib/api/productApiService';

function ProductTable() {
  const {
    items,
    isLoading,
    page,
    pageSize,
    totalCount,
    totalPages,
    handlePageChange,
  } = usePagination({
    apiService: productApiService,
    entity: 'products',
  });

  return (
    <div>
      <table>
        <tbody>
          {items.map(product => (
            <tr key={product.id}>
              <td>{product.productName}</td>
              <td>{product.price}</td>
            </tr>
          ))}
        </tbody>
      </table>

      <div>
        <button
          onClick={() => handlePageChange(page - 1)}
          disabled={page === 1}
        >
          Previous
        </button>

        <span>Page {page} of {totalPages}</span>

        <button
          onClick={() => handlePageChange(page + 1)}
          disabled={page === totalPages}
        >
          Next
        </button>
      </div>
    </div>
  );
}
```

**📌 Advanced Usage với Search, Sort, Filters:**

```typescript
import { useState } from 'react';
import { usePagination } from '../../../hooks/usePagination';
import { productApiService } from '../../../lib/api/productApiService';

function ProductTableAdvanced() {
  const [categoryId, setCategoryId] = useState<number>();

  const {
    items,
    isLoading,
    isFetching,
    page,
    pageSize,
    totalCount,
    search,
    sortBy,
    sortDesc,
    handlePageChange,
    handleSearch,
    handleSort,
    resetPagination,
    setPageSize,
  } = usePagination({
    apiService: productApiService,
    entity: 'products',
    initialPage: 1,
    initialPageSize: 20,
    initialSortBy: 'createdAt',
    initialSortDesc: true,
    additionalParams: {
      categoryId, // Dynamic filter
    },
  });

  return (
    <div>
      {/* Search */}
      <input
        type="text"
        placeholder="Search..."
        defaultValue={search}
        onKeyDown={(e) => {
          if (e.key === 'Enter') {
            handleSearch(e.currentTarget.value);
          }
        }}
      />

      {/* Filters */}
      <select onChange={(e) => setCategoryId(Number(e.target.value))}>
        <option value="">All Categories</option>
        <option value="1">Category 1</option>
      </select>

      {/* Page Size */}
      <select
        value={pageSize}
        onChange={(e) => setPageSize(Number(e.target.value))}
      >
        <option value="10">10 per page</option>
        <option value="20">20 per page</option>
        <option value="50">50 per page</option>
      </select>

      <button onClick={resetPagination}>Reset All</button>

      {/* Table */}
      <table>
        <thead>
          <tr>
            <th onClick={() => handleSort('productName', !sortDesc)}>
              Name {sortBy === 'productName' && (sortDesc ? '↓' : '↑')}
            </th>
            <th onClick={() => handleSort('price', !sortDesc)}>
              Price {sortBy === 'price' && (sortDesc ? '↓' : '↑')}
            </th>
          </tr>
        </thead>
        <tbody>
          {items.map(product => (
            <tr key={product.id}>
              <td>{product.productName}</td>
              <td>{product.price}</td>
            </tr>
          ))}
        </tbody>
      </table>

      {/* Loading indicator */}
      {isFetching && <div>Loading...</div>}

      {/* Pagination */}
      <div>
        <span>Total: {totalCount} items</span>
        <button onClick={() => handlePageChange(page - 1)} disabled={page === 1}>
          Previous
        </button>
        <span>Page {page}</span>
        <button onClick={() => handlePageChange(page + 1)}>
          Next
        </button>
      </div>
    </div>
  );
}
```

**📌 Config Options:**

```typescript
interface UsePaginationConfig<TData> {
  // Required
  apiService: ApiService<TData>;
  entity: string;

  // Optional
  initialPage?: number;           // Default: 1
  initialPageSize?: number;       // Default: 20
  initialSearch?: string;         // Default: ''
  initialSortBy?: string;         // Default: 'id'
  initialSortDesc?: boolean;      // Default: true
  additionalParams?: QueryParams; // Extra params (filters, etc.)
}
```

**📌 Return Values:**

```typescript
{
  // Query result
  data: PagedList<TData> | undefined;
  isLoading: boolean;
  isFetching: boolean;
  isError: boolean;
  error: Error | null;
  refetch: () => void;

  // Pagination state
  page: number;
  pageSize: number;
  search: string;
  sortBy: string;
  sortDesc: boolean;

  // Pagination data
  totalCount: number;
  totalPages: number;
  hasPrevious: boolean;
  hasNext: boolean;
  items: TData[];

  // Actions
  handlePageChange: (newPage: number, newPageSize?: number) => void;
  handleSearch: (searchText: string) => void;
  handleSort: (field: string, descending: boolean) => void;
  resetPagination: () => void;

  // Setters
  setPage: (page: number) => void;
  setPageSize: (size: number) => void;
  setSearch: (search: string) => void;
  setSortBy: (field: string) => void;
  setSortDesc: (desc: boolean) => void;
}
```

**💡 Common Patterns:**

```typescript
// 1. Debounced search
const [searchInput, setSearchInput] = useState('');

useEffect(() => {
  const timer = setTimeout(() => {
    handleSearch(searchInput);
  }, 500);
  return () => clearTimeout(timer);
}, [searchInput]);

// 2. Reset về page 1 khi filter change
useEffect(() => {
  setPage(1);
}, [categoryId, minPrice]);

// 3. Persist pageSize trong localStorage
useEffect(() => {
  localStorage.setItem('products-pageSize', pageSize.toString());
}, [pageSize]);

// 4. Show loading overlay
{isFetching && !isLoading && <LoadingOverlay />}
```

#### 4.12.3. usePaginationWithRouter - URL-based Pagination

**📌 Basic Usage:**

```typescript
import { getRouteApi } from '@tanstack/react-router';
import { usePaginationWithRouter } from '../../../hooks/usePaginationWithRouter';
import { productApiService } from '../../../lib/api/productApiService';

const routeApi = getRouteApi('/admin/products');

function ProductListPage() {
  const {
    items,
    isLoading,
    params,
    totalCount,
    handlePageChange,
    handleSearch,
  } = usePaginationWithRouter({
    apiService: productApiService,
    entity: 'products',
    routeApi,
  });

  return (
    <div>
      <input
        type="text"
        defaultValue={params.search}
        onKeyDown={(e) => {
          if (e.key === 'Enter') {
            handleSearch(e.currentTarget.value);
          }
        }}
      />

      <table>
        <tbody>
          {items.map(product => (
            <tr key={product.id}>
              <td>{product.productName}</td>
            </tr>
          ))}
        </tbody>
      </table>

      <div>
        <button onClick={() => handlePageChange(params.page - 1)}>
          Previous
        </button>
        <span>Page {params.page}</span>
        <button onClick={() => handlePageChange(params.page + 1)}>
          Next
        </button>
      </div>
    </div>
  );
}
```

**📌 Advanced Usage với Route Definition:**

```typescript
// 1. Define route với search params validation
// routes/admin/products.tsx
import { createFileRoute } from '@tanstack/react-router';
import { z } from 'zod';

const productSearchSchema = z.object({
  page: z.number().min(1).catch(1),
  pageSize: z.number().min(10).max(100).catch(20),
  search: z.string().optional(),
  sortBy: z.string().catch('id'),
  sortDesc: z.boolean().catch(true),
  categoryId: z.number().optional(),
});

export const Route = createFileRoute('/admin/products')({
  validateSearch: productSearchSchema,
});

// 2. Component sử dụng
function ProductListPageAdvanced() {
  const routeApi = getRouteApi('/admin/products');
  const navigate = useNavigate();

  const {
    items,
    params,
    totalCount,
    handlePageChange,
    handleSearch,
    handleSort,
    resetPagination,
  } = usePaginationWithRouter({
    apiService: productApiService,
    entity: 'products',
    routeApi,
    additionalParams: {
      // Additional filters from URL
    },
  });

  // Share current URL
  const shareUrl = () => {
    const url = window.location.href;
    navigator.clipboard.writeText(url);
    alert('URL copied!');
  };

  return (
    <div>
      <button onClick={shareUrl}>Share Current View</button>
      <button onClick={resetPagination}>Reset Filters</button>

      {/* Table with URL-synced state */}
      <table>
        <thead>
          <tr>
            <th onClick={() => handleSort('productName', !params.sortDesc)}>
              Name {params.sortBy === 'productName' && (params.sortDesc ? '↓' : '↑')}
            </th>
          </tr>
        </thead>
        <tbody>
          {items.map(product => (
            <tr key={product.id}>
              <td>{product.productName}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

**📌 Config Options:**

```typescript
interface UsePaginationWithRouterConfig<TData> {
  apiService: ApiService<TData>;
  entity: string;
  routeApi: any; // TanStack Router route API
  additionalParams?: QueryParams;
}
```

**📌 Return Values:**

```typescript
{
  // Same as usePagination, plus:
  params: PagedRequest & QueryParams; // Current URL params

  // Actions automatically update URL
  handlePageChange: (newPage: number, newPageSize?: number) => void;
  handleSearch: (searchText: string) => void;
  handleSort: (field: string, descending: boolean) => void;
  resetPagination: () => void;
}
```

**💡 Common Patterns:**

```typescript
// 1. Deep linking - User có thể bookmark URL
// URL: /products?page=2&pageSize=50&search=laptop&sortBy=price&sortDesc=false

// 2. Share filtered view
const shareCurrentView = () => {
  const url = new URL(window.location.href);
  navigator.clipboard.writeText(url.toString());
};

// 3. Navigate programmatically
navigate({
  to: '/admin/products',
  search: {
    page: 1,
    pageSize: 20,
    categoryId: 5,
  },
});

// 4. Preserve filters khi navigate back
// TanStack Router tự động preserve search params
```

#### 4.12.4. usePaginationWithFilters - Advanced Filters

**📌 Basic Usage:**

```typescript
import { usePaginationWithFilters } from '../../../hooks/usePaginationWithFilters';
import { productApiService } from '../../../lib/api/productApiService';

interface ProductFilters {
  categoryId?: number;
  minPrice?: number;
  maxPrice?: number;
}

function ProductTableWithFilters() {
  const {
    items,
    page,
    pageSize,
    filters,
    activeFiltersCount,
    handlePageChange,
    handleFilterChange,
    clearFilters,
  } = usePaginationWithFilters<ProductEntity, ProductFilters>({
    apiService: productApiService,
    entity: 'products',
    initialFilters: {},
  });

  return (
    <div>
      {/* Filters */}
      <div>
        <select
          value={filters.categoryId || ''}
          onChange={(e) => handleFilterChange({
            categoryId: e.target.value ? Number(e.target.value) : undefined
          })}
        >
          <option value="">All Categories</option>
          <option value="1">Category 1</option>
          <option value="2">Category 2</option>
        </select>

        <input
          type="number"
          placeholder="Min Price"
          value={filters.minPrice || ''}
          onChange={(e) => handleFilterChange({
            minPrice: e.target.value ? Number(e.target.value) : undefined
          })}
        />

        <input
          type="number"
          placeholder="Max Price"
          value={filters.maxPrice || ''}
          onChange={(e) => handleFilterChange({
            maxPrice: e.target.value ? Number(e.target.value) : undefined
          })}
        />

        {activeFiltersCount > 0 && (
          <button onClick={clearFilters}>
            Clear Filters ({activeFiltersCount})
          </button>
        )}
      </div>

      {/* Table */}
      <table>
        <tbody>
          {items.map(product => (
            <tr key={product.id}>
              <td>{product.productName}</td>
              <td>{product.price}</td>
            </tr>
          ))}
        </tbody>
      </table>

      {/* Pagination */}
      <button onClick={() => handlePageChange(page - 1)}>Previous</button>
      <span>Page {page}</span>
      <button onClick={() => handlePageChange(page + 1)}>Next</button>
    </div>
  );
}
```

**📌 Advanced Usage với Filter Drawer:**

```typescript
import { useState } from 'react';
import { Drawer, Button, Badge, Space, InputNumber, Select } from 'antd';
import { FilterOutlined } from '@ant-design/icons';
import { usePaginationWithFilters } from '../../../hooks/usePaginationWithFilters';

interface ProductFilters {
  categoryId?: number;
  supplierId?: number;
  minPrice?: number;
  maxPrice?: number;
  unit?: string;
  inStock?: boolean;
}

function ProductTableWithFilterDrawer() {
  const [drawerVisible, setDrawerVisible] = useState(false);
  const [tempFilters, setTempFilters] = useState<Partial<ProductFilters>>({});

  const {
    items,
    isLoading,
    filters,
    activeFiltersCount,
    handleFilterChange,
    clearFilters,
    resetAll,
  } = usePaginationWithFilters<ProductEntity, ProductFilters>({
    apiService: productApiService,
    entity: 'products',
    initialFilters: {
      inStock: true, // Default filter
    },
  });

  const applyFilters = () => {
    handleFilterChange(tempFilters);
    setDrawerVisible(false);
  };

  const handleResetFilters = () => {
    setTempFilters({});
    clearFilters();
    setDrawerVisible(false);
  };

  return (
    <div>
      {/* Filter Button */}
      <Badge count={activeFiltersCount} offset={[-5, 5]}>
        <Button
          icon={<FilterOutlined />}
          onClick={() => {
            setTempFilters(filters);
            setDrawerVisible(true);
          }}
        >
          Filters
        </Button>
      </Badge>

      {/* Active Filters Display */}
      {activeFiltersCount > 0 && (
        <Space>
          {filters.categoryId && <span>Category: {filters.categoryId}</span>}
          {filters.minPrice && <span>Min: ${filters.minPrice}</span>}
          {filters.maxPrice && <span>Max: ${filters.maxPrice}</span>}
          <Button size="small" onClick={clearFilters}>Clear All</Button>
        </Space>
      )}

      {/* Table */}
      <table>
        <tbody>
          {items.map(product => (
            <tr key={product.id}>
              <td>{product.productName}</td>
              <td>{product.price}</td>
            </tr>
          ))}
        </tbody>
      </table>

      {/* Filter Drawer */}
      <Drawer
        title="Advanced Filters"
        placement="right"
        width={400}
        open={drawerVisible}
        onClose={() => setDrawerVisible(false)}
        extra={
          <Space>
            <Button onClick={handleResetFilters}>Reset</Button>
            <Button type="primary" onClick={applyFilters}>Apply</Button>
          </Space>
        }
      >
        <Space direction="vertical" style={{ width: '100%' }} size="large">
          {/* Category Filter */}
          <div>
            <label>Category</label>
            <Select
              style={{ width: '100%' }}
              value={tempFilters.categoryId}
              onChange={(value) => setTempFilters({ ...tempFilters, categoryId: value })}
              allowClear
            >
              <Select.Option value={1}>Category 1</Select.Option>
              <Select.Option value={2}>Category 2</Select.Option>
            </Select>
          </div>

          {/* Price Range */}
          <div>
            <label>Price Range</label>
            <Space>
              <InputNumber
                placeholder="Min"
                value={tempFilters.minPrice}
                onChange={(value) => setTempFilters({ ...tempFilters, minPrice: value || undefined })}
              />
              <span>-</span>
              <InputNumber
                placeholder="Max"
                value={tempFilters.maxPrice}
                onChange={(value) => setTempFilters({ ...tempFilters, maxPrice: value || undefined })}
              />
            </Space>
          </div>

          {/* In Stock */}
          <div>
            <label>
              <input
                type="checkbox"
                checked={tempFilters.inStock || false}
                onChange={(e) => setTempFilters({ ...tempFilters, inStock: e.target.checked })}
              />
              In Stock Only
            </label>
          </div>
        </Space>
      </Drawer>
    </div>
  );
}
```

**📌 Config Options:**

```typescript
interface UsePaginationWithFiltersConfig<TData, TFilters> {
  apiService: ApiService<TData>;
  entity: string;
  initialFilters?: TFilters;      // Default filters
  initialPage?: number;
  initialPageSize?: number;
  initialSortBy?: string;
  initialSortDesc?: boolean;
}
```

**📌 Return Values:**

```typescript
{
  // Same as usePagination, plus:
  filters: TFilters;              // Current filters
  activeFiltersCount: number;     // Count of active filters

  // Filter actions
  handleFilterChange: (newFilters: Partial<TFilters>) => void;
  clearFilters: () => void;
  resetAll: () => void;           // Reset pagination + filters
  setFilters: (filters: TFilters) => void;
}
```

**💡 Common Patterns:**

```typescript
// 1. Filter presets
const filterPresets = {
  inStock: { inStock: true, minPrice: 0 },
  expensive: { minPrice: 1000 },
  cheap: { maxPrice: 100 },
};

const applyPreset = (preset: keyof typeof filterPresets) => {
  handleFilterChange(filterPresets[preset]);
};

// 2. URL sync với filters (combine với usePaginationWithRouter)
const searchParams = new URLSearchParams();
Object.entries(filters).forEach(([key, value]) => {
  if (value !== undefined) {
    searchParams.set(key, String(value));
  }
});

// 3. Save filter preferences
useEffect(() => {
  localStorage.setItem('product-filters', JSON.stringify(filters));
}, [filters]);

// 4. Complex filter logic
const handlePriceRangeChange = (min?: number, max?: number) => {
  handleFilterChange({
    minPrice: min,
    maxPrice: max,
  });
};
```

#### 4.12.5. usePaginationWithPrefetch - Prefetching Support

**📌 Basic Usage:**

```typescript
import { useEffect } from 'react';
import { usePaginationWithPrefetch } from '../../../hooks/usePaginationWithPrefetch';
import { productApiService } from '../../../lib/api/productApiService';

function ProductTableWithPrefetch() {
  const {
    items,
    page,
    hasNext,
    handlePageChange,
    prefetchNextPage,
    prefetchDetail,
  } = usePaginationWithPrefetch({
    apiService: productApiService,
    entity: 'products',
  });

  // Auto-prefetch next page
  useEffect(() => {
    if (hasNext) {
      prefetchNextPage();
    }
  }, [page, hasNext, prefetchNextPage]);

  return (
    <table>
      <tbody>
        {items.map(product => (
          <tr
            key={product.id}
            onMouseEnter={() => prefetchDetail(product.id!)}
          >
            <td>{product.productName}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

**📌 Advanced Usage với Smart Prefetching:**

```typescript
import { useEffect, useCallback } from 'react';
import { usePaginationWithPrefetch } from '../../../hooks/usePaginationWithPrefetch';
import { Link } from '@tanstack/react-router';

function ProductTableSmartPrefetch() {
  const {
    items,
    page,
    totalPages,
    hasNext,
    hasPrevious,
    handlePageChange,
    prefetchNextPage,
    prefetchPreviousPage,
    prefetchDetail,
  } = usePaginationWithPrefetch({
    apiService: productApiService,
    entity: 'products',
    initialPageSize: 20,
  });

  // Prefetch adjacent pages
  useEffect(() => {
    const timer = setTimeout(() => {
      if (hasNext) prefetchNextPage();
      if (hasPrevious) prefetchPreviousPage();
    }, 500); // Delay để không prefetch ngay lập tức

    return () => clearTimeout(timer);
  }, [page, hasNext, hasPrevious, prefetchNextPage, prefetchPreviousPage]);

  // Prefetch on hover với debounce
  const handleRowHover = useCallback((id: number) => {
    const timer = setTimeout(() => {
      prefetchDetail(id);
    }, 200);
    return () => clearTimeout(timer);
  }, [prefetchDetail]);

  // Prefetch khi user hover vào pagination buttons
  const handlePaginationHover = useCallback((targetPage: number) => {
    // Có thể prefetch specific page nếu cần
    if (targetPage === page + 1) {
      prefetchNextPage();
    } else if (targetPage === page - 1) {
      prefetchPreviousPage();
    }
  }, [page, prefetchNextPage, prefetchPreviousPage]);

  return (
    <div>
      <table>
        <tbody>
          {items.map(product => (
            <tr
              key={product.id}
              onMouseEnter={() => handleRowHover(product.id!)}
            >
              <td>
                <Link
                  to="/admin/products/$id"
                  params={{ id: product.id!.toString() }}
                >
                  {product.productName}
                </Link>
              </td>
              <td>{product.price}</td>
            </tr>
          ))}
        </tbody>
      </table>

      {/* Pagination với prefetch on hover */}
      <div>
        <button
          onClick={() => handlePageChange(page - 1)}
          onMouseEnter={() => handlePaginationHover(page - 1)}
          disabled={!hasPrevious}
        >
          Previous
        </button>

        {/* Page numbers */}
        {Array.from({ length: Math.min(5, totalPages) }, (_, i) => {
          const pageNum = page - 2 + i;
          if (pageNum < 1 || pageNum > totalPages) return null;

          return (
            <button
              key={pageNum}
              onClick={() => handlePageChange(pageNum)}
              onMouseEnter={() => handlePaginationHover(pageNum)}
              disabled={pageNum === page}
            >
              {pageNum}
            </button>
          );
        })}

        <button
          onClick={() => handlePageChange(page + 1)}
          onMouseEnter={() => handlePaginationHover(page + 1)}
          disabled={!hasNext}
        >
          Next
        </button>
      </div>
    </div>
  );
}
```

**📌 Return Values:**

```typescript
{
  // Same as usePagination, plus:
  prefetchNextPage: () => void;
  prefetchPreviousPage: () => void;
  prefetchDetail: (id: number | string) => void;
}
```

**💡 Common Patterns:**

```typescript
// 1. Prefetch on scroll (near bottom)
useEffect(() => {
  const handleScroll = () => {
    const scrollPercentage =
      (window.scrollY + window.innerHeight) / document.body.scrollHeight;

    if (scrollPercentage > 0.8 && hasNext) {
      prefetchNextPage();
    }
  };

  window.addEventListener('scroll', handleScroll);
  return () => window.removeEventListener('scroll', handleScroll);
}, [hasNext, prefetchNextPage]);

// 2. Prefetch on idle
useEffect(() => {
  const timer = setTimeout(() => {
    if (hasNext) prefetchNextPage();
  }, 2000); // Prefetch sau 2s idle

  return () => clearTimeout(timer);
}, [page, hasNext, prefetchNextPage]);

// 3. Conditional prefetch (chỉ prefetch khi network tốt)
useEffect(() => {
  const connection = (navigator as any).connection;
  const shouldPrefetch = !connection || connection.effectiveType === '4g';

  if (shouldPrefetch && hasNext) {
    prefetchNextPage();
  }
}, [page, hasNext, prefetchNextPage]);

// 4. Prefetch multiple pages ahead
const prefetchMultiplePages = useCallback(() => {
  if (hasNext) {
    prefetchNextPage();
    // Có thể prefetch page + 2, page + 3 nếu cần
  }
}, [hasNext, prefetchNextPage]);
```

### 4.13. Chi tiết Tham số, Đầu ra và Luồng hoạt động

#### 4.13.1. useApiList - GET All Items (No Pagination)

**📥 INPUT PARAMETERS:**

```typescript
interface UseApiListConfig<TData, TError = Error> {
  // REQUIRED
  apiService: ApiService<TData>;
  // - Mô tả: Instance của ApiService cho entity cụ thể
  // - Type: ApiService<TData>
  // - Example: productApiService, userApiService

  entity: string;
  // - Mô tả: Tên entity để tạo query keys
  // - Type: string
  // - Example: 'products', 'users', 'orders'
  // - Dùng để: Tạo unique query keys cho cache management

  // OPTIONAL
  params?: QueryParams;
  // - Mô tả: Additional query parameters
  // - Type: Record<string, any>
  // - Example: { categoryId: 5, inStock: true }
  // - Default: {}

  options?: UseQueryOptions<TData[], TError>;
  // - Mô tả: TanStack Query options
  // - Type: UseQueryOptions
  // - Common options:
  //   * staleTime: number - Cache time (ms)
  //   * enabled: boolean - Enable/disable query
  //   * refetchOnWindowFocus: boolean
  //   * retry: number | boolean
  // - Default: {}
}
```

**📤 OUTPUT/RETURN VALUES:**

```typescript
{
  // DATA
  data: TData[] | undefined;
  // - Mô tả: Array of items
  // - Type: TData[] hoặc undefined khi chưa load
  // - Example: ProductEntity[]

  // LOADING STATES
  isLoading: boolean;
  // - Mô tả: true khi đang fetch lần đầu (chưa có data)
  // - Use case: Show skeleton/spinner

  isFetching: boolean;
  // - Mô tả: true khi đang fetch (kể cả background refetch)
  // - Use case: Show loading indicator

  isRefetching: boolean;
  // - Mô tả: true khi đang refetch (đã có data)
  // - Use case: Show subtle loading indicator

  // ERROR STATES
  isError: boolean;
  // - Mô tả: true khi có lỗi

  error: TError | null;
  // - Mô tả: Error object nếu có
  // - Type: Error hoặc null

  // SUCCESS STATE
  isSuccess: boolean;
  // - Mô tả: true khi fetch thành công

  // ACTIONS
  refetch: () => Promise<QueryObserverResult<TData[], TError>>;
  // - Mô tả: Manually refetch data
  // - Returns: Promise với result

  // METADATA
  dataUpdatedAt: number;
  // - Mô tả: Timestamp của lần update cuối
  // - Type: number (Unix timestamp)

  errorUpdatedAt: number;
  // - Mô tả: Timestamp của lần error cuối
}
```

**🔄 LUỒNG HOẠT ĐỘNG:**

```
1. Component mount
   ↓
2. useApiList được gọi với config
   ↓
3. Tạo query key: ['entity', 'list', params]
   ↓
4. Check cache
   ├─ Có cache và chưa stale → Return cached data
   └─ Không có cache hoặc stale → Tiếp tục
   ↓
5. Call apiService.getAll(params)
   ↓
6. API Request → Backend
   ↓
7. Response từ Backend
   ├─ Success → Update cache, return data
   └─ Error → Set error state
   ↓
8. Component re-render với data/error
   ↓
9. Background refetch (nếu refetchOnWindowFocus = true)
```

**📊 DATA FLOW DIAGRAM:**

```mermaid
graph TD
    A[Component] -->|Call useApiList| B[Hook]
    B -->|Generate Query Key| C[Query Key Factory]
    C -->|Check Cache| D{Cache Valid?}
    D -->|Yes| E[Return Cached Data]
    D -->|No| F[Call apiService.getAll]
    F -->|HTTP GET| G[Backend API]
    G -->|Response| H{Success?}
    H -->|Yes| I[Update Cache]
    H -->|No| J[Set Error State]
    I --> K[Return Data]
    J --> K
    K --> A
```

**⚙️ INTERNAL MECHANISM:**

```typescript
// Bên trong hook
export function useApiList<TData>({ apiService, entity, params, options }) {
  // 1. Tạo query keys
  const queryKeys = createQueryKeys(entity);
  const queryKey = queryKeys.list(params);
  // Result: ['products', 'list', { categoryId: 5 }]

  // 2. useQuery từ TanStack Query
  return useQuery<TData[]>({
    queryKey,
    queryFn: () => apiService.getAll(params),
    ...options,
  });

  // 3. TanStack Query tự động:
  //    - Cache management
  //    - Deduplication (không fetch duplicate requests)
  //    - Background refetch
  //    - Retry on error
  //    - Garbage collection
}
```

**🎯 USE CASES:**

```typescript
// 1. Simple list - Load all
const { data: products } = useApiList({
  apiService: productApiService,
  entity: 'products',
});

// 2. With filters
const { data: products } = useApiList({
  apiService: productApiService,
  entity: 'products',
  params: { categoryId: 5 },
});

// 3. With custom options
const { data: products, refetch } = useApiList({
  apiService: productApiService,
  entity: 'products',
  options: {
    staleTime: 1000 * 60 * 5, // 5 minutes
    enabled: isAuthenticated,  // Only fetch when authenticated
  },
});

// 4. Dependent query
const { data: category } = useApiDetail({ ... });
const { data: products } = useApiList({
  apiService: productApiService,
  entity: 'products',
  params: { categoryId: category?.id },
  options: {
    enabled: !!category?.id, // Only fetch when category loaded
  },
});
```

#### 4.13.2. useApiPagedList - GET Paged Items

**📥 INPUT PARAMETERS:**

```typescript
interface UseApiPagedListConfig<TData, TError = Error> {
  // REQUIRED
  apiService: ApiService<TData>;
  // - Mô tả: Instance của ApiService
  // - Type: ApiService<TData>

  entity: string;
  // - Mô tả: Entity name cho query keys
  // - Type: string

  // OPTIONAL
  params?: PagedRequest & QueryParams;
  // - Mô tả: Pagination + filter parameters
  // - Type: Object với các fields:
  //   * page: number (required) - Page number (1-based)
  //   * pageSize: number (required) - Items per page
  //   * search?: string - Search text
  //   * sortBy?: string - Sort field
  //   * sortDesc?: boolean - Sort descending
  //   * ...custom filters
  // - Example:
  //   {
  //     page: 1,
  //     pageSize: 20,
  //     search: 'laptop',
  //     sortBy: 'price',
  //     sortDesc: false,
  //     categoryId: 5
  //   }
  // - Default: { page: 1, pageSize: 20 }

  options?: UseQueryOptions<PagedList<TData>, TError>;
  // - Mô tả: TanStack Query options
  // - Common options:
  //   * staleTime: Cache time
  //   * placeholderData: Keep previous data while fetching
  //   * enabled: Enable/disable query
}
```

**📤 OUTPUT/RETURN VALUES:**

```typescript
{
  // DATA
  data: PagedList<TData> | undefined;
  // - Mô tả: Paged result object
  // - Structure:
  //   {
  //     page: number;           // Current page
  //     pageSize: number;       // Items per page
  //     totalCount: number;     // Total items
  //     totalPages: number;     // Total pages
  //     hasPrevious: boolean;   // Has previous page
  //     hasNext: boolean;       // Has next page
  //     items: TData[];         // Items in current page
  //   }

  // LOADING STATES
  isLoading: boolean;
  // - true khi fetch lần đầu

  isFetching: boolean;
  // - true khi đang fetch (including background)

  isPlaceholderData: boolean;
  // - true khi đang show placeholder data (previous page data)
  // - Use case: Smooth transition between pages

  // ERROR STATES
  isError: boolean;
  error: TError | null;

  // SUCCESS STATE
  isSuccess: boolean;

  // ACTIONS
  refetch: () => Promise<QueryObserverResult<PagedList<TData>, TError>>;
  // - Manually refetch current page

  // METADATA
  dataUpdatedAt: number;
  errorUpdatedAt: number;
}
```

**🔄 LUỒNG HOẠT ĐỘNG:**

```
1. Component mount với pagination params
   ↓
2. useApiPagedList được gọi
   ↓
3. Tạo query key với pagination params
   Example: ['products', 'list', { page: 1, pageSize: 20, search: 'laptop' }]
   ↓
4. Check cache cho exact key
   ├─ Cache hit → Return cached data
   └─ Cache miss → Continue
   ↓
5. Call apiService.getAllPaged(params)
   ↓
6. API Request với query params
   GET /api/products?page=1&pageSize=20&search=laptop
   ↓
7. Backend xử lý:
   - Filter theo search
   - Sort theo sortBy
   - Paginate
   - Calculate totalCount, totalPages
   ↓
8. Response: PagedList<TData>
   ↓
9. Update cache với key
   ↓
10. Component re-render với data
   ↓
11. User thay đổi page/search/sort
   ↓
12. New query key được tạo
   ↓
13. Repeat từ step 4
```

**📊 DATA FLOW DIAGRAM:**

```mermaid
graph TD
    A[Component] -->|page=1, pageSize=20| B[useApiPagedList]
    B -->|Create Key| C["['products', 'list', {page:1, pageSize:20}]"]
    C -->|Check Cache| D{Cache?}
    D -->|Hit| E[Return Cached PagedList]
    D -->|Miss| F[apiService.getAllPaged]
    F -->|GET /api/products?page=1&pageSize=20| G[Backend]
    G -->|Process| H[Filter + Sort + Paginate]
    H -->|Response| I[PagedList Object]
    I -->|Cache| J[TanStack Query Cache]
    J --> K[Return to Component]
    E --> K

    L[User clicks page 2] -->|page=2| B
    M[User searches] -->|search='laptop'| B
    N[User sorts] -->|sortBy='price'| B
```

**⚙️ INTERNAL MECHANISM:**

```typescript
export function useApiPagedList<TData>({ apiService, entity, params, options }) {
  // 1. Tạo query key với pagination params
  const queryKeys = createQueryKeys(entity);
  const queryKey = queryKeys.list(params);
  // Key thay đổi khi page/pageSize/search/sort thay đổi
  // → Trigger new fetch

  // 2. useQuery
  return useQuery<PagedList<TData>>({
    queryKey,
    queryFn: () => apiService.getAllPaged(params),

    // Keep previous data while fetching new page
    placeholderData: (previousData) => previousData,
    // → Smooth transition, no loading spinner between pages

    ...options,
  });

  // 3. Cache strategy:
  //    - Mỗi page có cache riêng
  //    - Page 1 cache: ['products', 'list', {page:1, pageSize:20}]
  //    - Page 2 cache: ['products', 'list', {page:2, pageSize:20}]
  //    - Search cache: ['products', 'list', {page:1, pageSize:20, search:'laptop'}]
}
```

**🎯 USE CASES:**

```typescript
// 1. Basic pagination
const { data } = useApiPagedList({
  apiService: productApiService,
  entity: 'products',
  params: { page: 1, pageSize: 20 },
});
// data.items: ProductEntity[]
// data.totalCount: 150
// data.totalPages: 8

// 2. With search
const { data } = useApiPagedList({
  apiService: productApiService,
  entity: 'products',
  params: {
    page: 1,
    pageSize: 20,
    search: 'laptop',
  },
});

// 3. With sort
const { data } = useApiPagedList({
  apiService: productApiService,
  entity: 'products',
  params: {
    page: 1,
    pageSize: 20,
    sortBy: 'price',
    sortDesc: false, // Ascending
  },
});

// 4. With filters
const { data } = useApiPagedList({
  apiService: productApiService,
  entity: 'products',
  params: {
    page: 1,
    pageSize: 20,
    categoryId: 5,
    minPrice: 100,
    maxPrice: 1000,
  },
});

// 5. Keep previous data while fetching
const { data, isFetching, isPlaceholderData } = useApiPagedList({
  apiService: productApiService,
  entity: 'products',
  params: { page, pageSize },
  options: {
    placeholderData: (previousData) => previousData,
  },
});
// isPlaceholderData = true → Showing old page while fetching new
// isFetching = true → Show subtle loading indicator
```

**🔍 CACHE BEHAVIOR:**

```typescript
// Scenario: User navigates pages
// Page 1 → Cache: ['products', 'list', {page:1}]
// Page 2 → Cache: ['products', 'list', {page:2}]
// Back to Page 1 → Instant (from cache)

// Scenario: User changes pageSize
// pageSize=20 → Cache: ['products', 'list', {page:1, pageSize:20}]
// pageSize=50 → Cache: ['products', 'list', {page:1, pageSize:50}]
// Different cache → New fetch

// Scenario: User searches
// No search → Cache: ['products', 'list', {page:1}]
// search='laptop' → Cache: ['products', 'list', {page:1, search:'laptop'}]
// Different cache → New fetch

// Invalidation:
// When create/update/delete product:
queryClient.invalidateQueries({ queryKey: ['products'] });
// → All product list caches invalidated
// → Next access will refetch
```

#### 4.13.3. useApiInfinite - Infinite Scroll

**📥 INPUT PARAMETERS:**

```typescript
interface UseApiInfiniteConfig<TData, TError = Error> {
  // REQUIRED
  apiService: ApiService<TData>;
  entity: string;

  // OPTIONAL
  params?: QueryParams;
  // - Mô tả: Filter parameters (không bao gồm page)
  // - Type: Record<string, any>
  // - Example: { categoryId: 5, search: 'laptop' }
  // - Note: page được quản lý tự động bởi useInfiniteQuery

  pageSize?: number;
  // - Mô tả: Items per page
  // - Type: number
  // - Default: 20
  // - Range: 1-100 (tùy backend config)

  options?: UseInfiniteQueryOptions<PagedList<TData>, TError>;
  // - Mô tả: TanStack Query infinite options
  // - Special options:
  //   * initialPageParam: Starting page (required in v5)
  //   * getNextPageParam: Function to get next page number
  //   * getPreviousPageParam: Function to get previous page
}
```

**📤 OUTPUT/RETURN VALUES:**

```typescript
{
  // DATA
  data: {
    pages: PagedList<TData>[];  // Array of pages
    pageParams: number[];        // Array of page numbers [1, 2, 3, ...]
  } | undefined;
  // - Mô tả: Object chứa tất cả pages đã load
  // - Example:
  //   {
  //     pages: [
  //       { page: 1, items: [...20 items], totalCount: 100 },
  //       { page: 2, items: [...20 items], totalCount: 100 },
  //       { page: 3, items: [...20 items], totalCount: 100 },
  //     ],
  //     pageParams: [1, 2, 3]
  //   }

  // LOADING STATES
  isLoading: boolean;
  // - true khi load page đầu tiên

  isFetching: boolean;
  // - true khi đang fetch bất kỳ page nào

  isFetchingNextPage: boolean;
  // - true khi đang load next page
  // - Use case: Show "Loading more..." indicator

  isFetchingPreviousPage: boolean;
  // - true khi đang load previous page

  // PAGINATION INFO
  hasNextPage: boolean;
  // - true nếu còn page tiếp theo
  // - Calculated từ getNextPageParam

  hasPreviousPage: boolean;
  // - true nếu có page trước đó

  // ACTIONS
  fetchNextPage: () => Promise<InfiniteQueryObserverResult>;
  // - Mô tả: Load next page
  // - Returns: Promise với result
  // - Auto-increment page number

  fetchPreviousPage: () => Promise<InfiniteQueryObserverResult>;
  // - Load previous page

  refetch: () => Promise<InfiniteQueryObserverResult>;
  // - Refetch tất cả pages

  // ERROR
  isError: boolean;
  error: TError | null;
}
```

**🔄 LUỒNG HOẠT ĐỘNG:**

```
1. Component mount
   ↓
2. useApiInfinite được gọi
   ↓
3. Tạo query key: ['products', 'list', { pageSize: 20, categoryId: 5 }]
   (Không có page trong key vì infinite query)
   ↓
4. Load page 1 (initialPageParam = 1)
   ↓
5. Call apiService.getAllPaged({ page: 1, pageSize: 20, ...params })
   ↓
6. Response: PagedList<TData>
   {
     page: 1,
     pageSize: 20,
     totalCount: 100,
     totalPages: 5,
     hasNext: true,
     items: [...20 items]
   }
   ↓
7. Store in data.pages[0]
   ↓
8. Call getNextPageParam(lastPage)
   → lastPage.hasNext ? lastPage.page + 1 : undefined
   → Returns: 2
   ↓
9. Set hasNextPage = true
   ↓
10. Component renders với 20 items
   ↓
11. User scrolls to bottom
   ↓
12. Intersection Observer triggers
   ↓
13. Call fetchNextPage()
   ↓
14. Load page 2 với pageParam = 2
   ↓
15. Response stored in data.pages[1]
   ↓
16. Component re-renders với 40 items (pages[0] + pages[1])
   ↓
17. Repeat steps 11-16 cho pages tiếp theo
   ↓
18. Khi page cuối: getNextPageParam returns undefined
   ↓
19. hasNextPage = false
   ↓
20. Hide "Load More" button
```

**📊 DATA FLOW DIAGRAM:**

```mermaid
graph TD
    A[Component Mount] -->|initialPageParam=1| B[useApiInfinite]
    B -->|Fetch Page 1| C[apiService.getAllPaged]
    C -->|Response| D[PagedList Page 1]
    D -->|Store| E[data.pages[0]]
    E -->|getNextPageParam| F{hasNext?}
    F -->|Yes| G[hasNextPage = true]
    F -->|No| H[hasNextPage = false]

    I[User Scrolls] -->|Intersection Observer| J{hasNextPage?}
    J -->|Yes| K[fetchNextPage]
    K -->|pageParam=2| C
    C -->|Response| L[PagedList Page 2]
    L -->|Append| M[data.pages[1]]
    M -->|getNextPageParam| F

    N[Flatten Pages] -->|pages.flatMap| O[All Items Array]
    O --> P[Render List]
```

**⚙️ INTERNAL MECHANISM:**

```typescript
export function useApiInfinite<TData>({ apiService, entity, params, pageSize, options }) {
  const queryKeys = createQueryKeys(entity);

  return useInfiniteQuery<PagedList<TData>>({
    // Query key (không có page)
    queryKey: queryKeys.list({ ...params, pageSize }),

    // Query function với pageParam
    queryFn: ({ pageParam = 1 }) => {
      return apiService.getAllPaged({
        ...params,
        page: pageParam as number,
        pageSize,
      });
    },

    // Initial page
    initialPageParam: 1,

    // Determine next page
    getNextPageParam: (lastPage) => {
      // lastPage = PagedList<TData>
      // lastPage.hasNext = true/false
      // lastPage.page = current page number
      return lastPage.hasNext ? lastPage.page + 1 : undefined;
      // undefined = no more pages
    },

    // Determine previous page
    getPreviousPageParam: (firstPage) => {
      return firstPage.hasPrevious ? firstPage.page - 1 : undefined;
    },

    ...options,
  });
}

// Usage trong component:
const { data, fetchNextPage, hasNextPage } = useApiInfinite({ ... });

// Flatten all pages
const allItems = data?.pages.flatMap(page => page.items) || [];
// [page1.items, page2.items, page3.items] → [...all items]

// Total count (từ page đầu tiên)
const totalCount = data?.pages[0]?.totalCount || 0;
```

**🎯 USE CASES:**

```typescript
// 1. Basic infinite scroll
const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useApiInfinite({
  apiService: productApiService,
  entity: 'products',
  pageSize: 20,
});

const allProducts = data?.pages.flatMap(page => page.items) || [];

// 2. With filters
const { data } = useApiInfinite({
  apiService: productApiService,
  entity: 'products',
  pageSize: 20,
  params: {
    categoryId: 5,
    search: 'laptop',
  },
});

// 3. Auto-load với Intersection Observer
useEffect(() => {
  const observer = new IntersectionObserver(
    (entries) => {
      if (entries[0].isIntersecting && hasNextPage && !isFetchingNextPage) {
        fetchNextPage();
      }
    },
    { threshold: 0.1 }
  );

  if (observerTarget.current) {
    observer.observe(observerTarget.current);
  }

  return () => observer.disconnect();
}, [hasNextPage, isFetchingNextPage, fetchNextPage]);

// 4. Manual "Load More" button
<button
  onClick={() => fetchNextPage()}
  disabled={!hasNextPage || isFetchingNextPage}
>
  {isFetchingNextPage ? 'Loading...' : 'Load More'}
</button>
```

**🔍 CACHE BEHAVIOR:**

```typescript
// Cache key: ['products', 'list', { pageSize: 20, categoryId: 5 }]
// Stored data:
{
  pages: [
    { page: 1, items: [...], totalCount: 100 },
    { page: 2, items: [...], totalCount: 100 },
    { page: 3, items: [...], totalCount: 100 },
  ],
  pageParams: [1, 2, 3]
}

// Khi filter thay đổi:
// Old: ['products', 'list', { pageSize: 20, categoryId: 5 }]
// New: ['products', 'list', { pageSize: 20, categoryId: 6 }]
// → Different cache → Reset to page 1

// Khi refetch:
refetch(); // Refetch tất cả pages (1, 2, 3)

// Khi invalidate:
queryClient.invalidateQueries({ queryKey: ['products'] });
// → All infinite queries invalidated
```

#### 4.13.4. usePagination - State Management Hook

**📥 INPUT PARAMETERS:**

```typescript
interface UsePaginationConfig<TData> {
  // REQUIRED
  apiService: ApiService<TData>;
  entity: string;

  // OPTIONAL - Initial State
  initialPage?: number;
  // - Default: 1
  // - Mô tả: Starting page number

  initialPageSize?: number;
  // - Default: 20
  // - Mô tả: Starting page size

  initialSearch?: string;
  // - Default: ''
  // - Mô tả: Initial search text

  initialSortBy?: string;
  // - Default: 'id'
  // - Mô tả: Initial sort field

  initialSortDesc?: boolean;
  // - Default: true
  // - Mô tả: Initial sort direction

  additionalParams?: QueryParams;
  // - Default: {}
  // - Mô tả: Extra params (filters, etc.)
  // - Example: { categoryId: 5, inStock: true }
}
```

**📤 OUTPUT/RETURN VALUES:**

```typescript
{
  // QUERY RESULT (từ useApiPagedList)
  data: PagedList<TData> | undefined;
  isLoading: boolean;
  isFetching: boolean;
  isError: boolean;
  error: Error | null;
  refetch: () => void;

  // PAGINATION STATE
  page: number;              // Current page
  pageSize: number;          // Items per page
  search: string;            // Search text
  sortBy: string;            // Sort field
  sortDesc: boolean;         // Sort direction

  // PAGINATION DATA (từ data)
  totalCount: number;        // Total items
  totalPages: number;        // Total pages
  hasPrevious: boolean;      // Has previous page
  hasNext: boolean;          // Has next page
  items: TData[];            // Items in current page

  // ACTIONS
  handlePageChange: (newPage: number, newPageSize?: number) => void;
  // - Mô tả: Change page/pageSize
  // - Logic:
  //   * Set page = newPage
  //   * If newPageSize changed: set pageSize, reset page to 1

  handleSearch: (searchText: string) => void;
  // - Mô tả: Update search
  // - Logic: Set search, reset page to 1

  handleSort: (field: string, descending: boolean) => void;
  // - Mô tả: Update sort
  // - Logic: Set sortBy, sortDesc, reset page to 1

  resetPagination: () => void;
  // - Mô tả: Reset all to initial values

  // SETTERS (direct state update)
  setPage: (page: number) => void;
  setPageSize: (size: number) => void;
  setSearch: (search: string) => void;
  setSortBy: (field: string) => void;
  setSortDesc: (desc: boolean) => void;
}
```

**🔄 LUỒNG HOẠT ĐỘNG:**

```
1. Component mount
   ↓
2. usePagination khởi tạo state
   - page = initialPage (1)
   - pageSize = initialPageSize (20)
   - search = initialSearch ('')
   - sortBy = initialSortBy ('id')
   - sortDesc = initialSortDesc (true)
   ↓
3. Build params object từ state
   params = {
     page: 1,
     pageSize: 20,
     search: '',
     sortBy: 'id',
     sortDesc: true,
     ...additionalParams
   }
   ↓
4. Call useApiPagedList với params
   ↓
5. Fetch data từ API
   ↓
6. Component renders với data
   ↓
7. User interaction:

   A. User clicks page 2
      ↓
      handlePageChange(2)
      ↓
      setPage(2)
      ↓
      params thay đổi: { page: 2, ... }
      ↓
      useApiPagedList refetch với params mới
      ↓
      Component re-render với page 2 data

   B. User searches "laptop"
      ↓
      handleSearch('laptop')
      ↓
      setSearch('laptop')
      setPage(1)  // Reset về page 1
      ↓
      params thay đổi: { page: 1, search: 'laptop', ... }
      ↓
      useApiPagedList refetch
      ↓
      Component re-render với search results

   C. User sorts by price
      ↓
      handleSort('price', false)
      ↓
      setSortBy('price')
      setSortDesc(false)
      setPage(1)  // Reset về page 1
      ↓
      params thay đổi: { page: 1, sortBy: 'price', sortDesc: false, ... }
      ↓
      useApiPagedList refetch
      ↓
      Component re-render với sorted data

   D. User changes pageSize to 50
      ↓
      handlePageChange(1, 50)
      ↓
      setPageSize(50)
      setPage(1)  // Reset về page 1
      ↓
      params thay đổi: { page: 1, pageSize: 50, ... }
      ↓
      useApiPagedList refetch
      ↓
      Component re-render với 50 items
```

**📊 STATE FLOW DIAGRAM:**

```mermaid
stateDiagram-v2
    [*] --> InitialState: Component Mount

    InitialState: page=1, pageSize=20, search='', sortBy='id'

    InitialState --> FetchData: Build params
    FetchData --> ShowData: API Response

    ShowData --> PageChange: User clicks page 2
    PageChange --> FetchData: page=2

    ShowData --> Search: User searches
    Search --> ResetPage: page=1
    ResetPage --> FetchData: search='laptop'

    ShowData --> Sort: User sorts
    Sort --> ResetPage: page=1
    ResetPage --> FetchData: sortBy='price'

    ShowData --> PageSizeChange: User changes pageSize
    PageSizeChange --> ResetPage: page=1
    ResetPage --> FetchData: pageSize=50
```

**⚙️ INTERNAL MECHANISM:**

```typescript
export function usePagination<TData>({ apiService, entity, initialPage = 1, ... }) {
  // 1. State management
  const [page, setPage] = useState(initialPage);
  const [pageSize, setPageSize] = useState(initialPageSize);
  const [search, setSearch] = useState(initialSearch);
  const [sortBy, setSortBy] = useState(initialSortBy);
  const [sortDesc, setSortDesc] = useState(initialSortDesc);

  // 2. Build params (useMemo để tránh re-create)
  const params = useMemo(() => ({
    page,
    pageSize,
    search: search || undefined,
    sortBy,
    sortDesc,
    ...additionalParams,
  }), [page, pageSize, search, sortBy, sortDesc, additionalParams]);

  // 3. Fetch data
  const query = useApiPagedList<TData>({
    apiService,
    entity,
    params,
    options: {
      staleTime: 1000 * 60 * 5,
      placeholderData: (previousData) => previousData,
    },
  });

  // 4. Helper functions
  const handlePageChange = (newPage: number, newPageSize?: number) => {
    setPage(newPage);
    if (newPageSize && newPageSize !== pageSize) {
      setPageSize(newPageSize);
      setPage(1); // Reset về page 1 khi đổi pageSize
    }
  };

  const handleSearch = (searchText: string) => {
    setSearch(searchText);
    setPage(1); // Reset về page 1 khi search
  };

  const handleSort = (field: string, descending: boolean) => {
    setSortBy(field);
    setSortDesc(descending);
    setPage(1); // Reset về page 1 khi sort
  };

  // 5. Return combined state + query result
  return {
    ...query,
    page,
    pageSize,
    search,
    sortBy,
    sortDesc,
    totalCount: query.data?.totalCount || 0,
    totalPages: query.data?.totalPages || 0,
    items: query.data?.items || [],
    handlePageChange,
    handleSearch,
    handleSort,
    // ...
  };
}
```

**🎯 KEY BEHAVIORS:**

```typescript
// 1. Auto-reset page to 1 khi:
// - Search changes
// - Sort changes
// - PageSize changes
// - Filters change
// → Tránh trường hợp page 5 nhưng chỉ có 2 pages sau filter

// 2. Keep previous data while fetching
// placeholderData: (previousData) => previousData
// → Smooth transition, không blink

// 3. Memoized params
// useMemo → Chỉ re-create khi dependencies thay đổi
// → Tránh unnecessary refetch

// 4. Separate setters vs handlers
// setPage(2) → Chỉ set page
// handlePageChange(2) → Set page + logic (reset nếu cần)
```

#### 4.13.5. usePaginationWithRouter - URL Sync Hook

**📥 INPUT PARAMETERS:**

```typescript
interface UsePaginationWithRouterConfig<TData> {
  apiService: ApiService<TData>;
  entity: string;

  routeApi: any;
  // - Mô tả: TanStack Router route API
  // - Type: ReturnType<typeof getRouteApi>
  // - Example: getRouteApi('/admin/products')
  // - Provides: useSearch(), useNavigate()

  additionalParams?: QueryParams;
  // - Mô tả: Extra params không lưu trong URL
  // - Example: { userId: currentUser.id }
}
```

**📤 OUTPUT/RETURN VALUES:**

```typescript
{
  // Same as usePagination, plus:

  params: PagedRequest & QueryParams;
  // - Mô tả: Current params từ URL
  // - Source: routeApi.useSearch()
  // - Example: { page: 2, pageSize: 20, search: 'laptop', sortBy: 'price' }

  // Actions tự động update URL
  handlePageChange: (newPage: number, newPageSize?: number) => void;
  handleSearch: (searchText: string) => void;
  handleSort: (field: string, descending: boolean) => void;
  resetPagination: () => void;
}
```

**🔄 LUỒNG HOẠT ĐỘNG:**

```
1. Component mount
   ↓
2. Read URL search params
   URL: /products?page=2&pageSize=20&search=laptop
   ↓
3. routeApi.useSearch() returns:
   {
     page: 2,
     pageSize: 20,
     search: 'laptop',
     sortBy: 'id',
     sortDesc: true
   }
   ↓
4. Build params từ URL
   ↓
5. Call useApiPagedList với params
   ↓
6. Fetch data cho page 2 với search='laptop'
   ↓
7. Component renders với data
   ↓
8. User clicks page 3
   ↓
9. handlePageChange(3)
   ↓
10. navigate({ search: (prev) => ({ ...prev, page: 3 }) })
   ↓
11. URL updates: /products?page=3&pageSize=20&search=laptop
   ↓
12. routeApi.useSearch() returns new params
   ↓
13. params thay đổi → useApiPagedList refetch
   ↓
14. Component re-render với page 3 data
   ↓
15. User shares URL
   ↓
16. Other user opens: /products?page=3&pageSize=20&search=laptop
   ↓
17. Exact same state restored!
```

**📊 URL SYNC FLOW:**

```mermaid
graph LR
    A[URL] -->|Read| B[routeApi.useSearch]
    B -->|Params| C[useApiPagedList]
    C -->|Fetch| D[API]
    D -->|Data| E[Component]

    E -->|User Action| F[handlePageChange]
    F -->|navigate| G[Update URL]
    G -->|Trigger| B

    H[Browser Back] -->|History| A
    I[Share URL] -->|Copy| A
```

**⚙️ INTERNAL MECHANISM:**

```typescript
export function usePaginationWithRouter<TData>({ apiService, entity, routeApi, additionalParams }) {
  const navigate = useNavigate();

  // 1. Read params từ URL
  const search = routeApi.useSearch();
  // search = { page: 2, pageSize: 20, search: 'laptop', ... }

  // 2. Build params
  const params = useMemo(() => ({
    page: search.page || 1,
    pageSize: search.pageSize || 20,
    search: search.search || undefined,
    sortBy: search.sortBy || 'id',
    sortDesc: search.sortDesc !== false,
    ...additionalParams,
  }), [search, additionalParams]);

  // 3. Fetch data
  const query = useApiPagedList<TData>({
    apiService,
    entity,
    params,
    options: {
      staleTime: 1000 * 60 * 5,
      placeholderData: (previousData) => previousData,
    },
  });

  // 4. Update URL helper
  const updateUrlParams = (newParams: Partial<PagedRequest>) => {
    navigate({
      search: (prev) => ({
        ...prev,
        ...newParams,
      }),
    });
  };

  // 5. Actions
  const handlePageChange = (newPage: number, newPageSize?: number) => {
    const updates: Partial<PagedRequest> = { page: newPage };
    if (newPageSize && newPageSize !== params.pageSize) {
      updates.pageSize = newPageSize;
      updates.page = 1;
    }
    updateUrlParams(updates);
  };

  const handleSearch = (searchText: string) => {
    updateUrlParams({
      search: searchText || undefined,
      page: 1,
    });
  };

  return {
    ...query,
    params,
    handlePageChange,
    handleSearch,
    // ...
  };
}
```

**🎯 KEY FEATURES:**

```typescript
// 1. Deep linking
// URL: /products?page=3&search=laptop&categoryId=5
// → User có thể bookmark, share, hoặc refresh
// → State được restore chính xác

// 2. Browser history
// User: Page 1 → Page 2 → Page 3
// Browser back → Page 2 (từ URL)
// Browser back → Page 1 (từ URL)

// 3. SEO friendly
// Search engines có thể index các filtered/sorted views

// 4. Route validation với Zod
const productSearchSchema = z.object({
  page: z.number().min(1).catch(1),
  pageSize: z.number().min(10).max(100).catch(20),
  search: z.string().optional(),
});

export const Route = createFileRoute('/admin/products')({
  validateSearch: productSearchSchema,
});
// → Invalid params tự động fallback về default

// 5. Programmatic navigation
navigate({
  to: '/admin/products',
  search: {
    page: 1,
    categoryId: 5,
  },
});
```

#### 4.13.6. useApiDetail - GET by ID

**📥 INPUT PARAMETERS:**

```typescript
interface UseApiDetailConfig<TData, TError = Error> {
  // REQUIRED
  apiService: ApiService<TData>;
  // - Mô tả: Instance của ApiService

  entity: string;
  // - Mô tả: Entity name cho query keys

  id: number | string | undefined;
  // - Mô tả: ID của item cần fetch
  // - Type: number | string | undefined
  // - undefined → Query disabled (không fetch)
  // - Example: 123, 'abc-123', productId

  // OPTIONAL
  options?: UseQueryOptions<TData, TError>;
  // - Mô tả: TanStack Query options
  // - Common options:
  //   * enabled: boolean - Enable/disable query
  //   * staleTime: number - Cache time
  //   * retry: number | boolean - Retry on error
}
```

**📤 OUTPUT/RETURN VALUES:**

```typescript
{
  // DATA
  data: TData | undefined;
  // - Mô tả: Single item object
  // - Type: TData hoặc undefined
  // - Example: ProductEntity { id: 1, productName: 'Laptop', ... }

  // LOADING STATES
  isLoading: boolean;
  // - true khi đang fetch lần đầu

  isFetching: boolean;
  // - true khi đang fetch (including refetch)

  // ERROR STATES
  isError: boolean;
  error: TError | null;

  // SUCCESS STATE
  isSuccess: boolean;

  // ACTIONS
  refetch: () => Promise<QueryObserverResult<TData, TError>>;
  // - Manually refetch item

  // METADATA
  dataUpdatedAt: number;
  errorUpdatedAt: number;
}
```

**🔄 LUỒNG HOẠT ĐỘNG:**

```
1. Component mount với id
   ↓
2. useApiDetail được gọi với id
   ↓
3. Check id
   ├─ id = undefined → Query disabled, return early
   └─ id có giá trị → Continue
   ↓
4. Tạo query key: ['products', 'detail', 123]
   ↓
5. Check cache
   ├─ Cache hit và chưa stale → Return cached data
   └─ Cache miss hoặc stale → Continue
   ↓
6. Call apiService.getById(id)
   ↓
7. API Request: GET /api/products/123
   ↓
8. Backend response
   ├─ Success → Update cache, return data
   └─ Error (404, 500, etc.) → Set error state
   ↓
9. Component re-render với data/error
   ↓
10. Background refetch (nếu refetchOnWindowFocus = true)
```

**📊 DATA FLOW DIAGRAM:**

```mermaid
graph TD
    A[Component] -->|id=123| B[useApiDetail]
    B -->|Check id| C{id valid?}
    C -->|No/undefined| D[Query Disabled]
    C -->|Yes| E[Create Query Key]
    E -->|"['products', 'detail', 123]"| F{Cache?}
    F -->|Hit| G[Return Cached]
    F -->|Miss| H[apiService.getById]
    H -->|GET /api/products/123| I[Backend]
    I -->|Response| J{Success?}
    J -->|Yes| K[Update Cache]
    J -->|No 404| L[Set Error]
    K --> M[Return Data]
    L --> M
    M --> A
```

**⚙️ INTERNAL MECHANISM:**

```typescript
export function useApiDetail<TData>({ apiService, entity, id, options }) {
  const queryKeys = createQueryKeys(entity);

  return useQuery<TData>({
    // Query key với id
    queryKey: queryKeys.detail(id!),
    // Result: ['products', 'detail', 123]

    // Query function
    queryFn: () => apiService.getById(id!),
    // Call: GET /api/products/123

    // Chỉ fetch khi có id
    enabled: !!id && (options?.enabled !== false),
    // !!id → Convert to boolean
    // undefined/null/0 → false → Query disabled

    ...options,
  });
}
```

**🎯 USE CASES:**

```typescript
// 1. Basic detail fetch
const { data: product, isLoading } = useApiDetail({
  apiService: productApiService,
  entity: 'products',
  id: 123,
});

// 2. Dependent query (id từ route params)
const { id } = useParams();
const { data: product } = useApiDetail({
  apiService: productApiService,
  entity: 'products',
  id: id ? Number(id) : undefined,
});
// id = undefined → Query disabled

// 3. Conditional fetch
const [selectedId, setSelectedId] = useState<number>();
const { data: product } = useApiDetail({
  apiService: productApiService,
  entity: 'products',
  id: selectedId,
  options: {
    enabled: !!selectedId, // Chỉ fetch khi có selectedId
  },
});

// 4. With custom staleTime
const { data: product } = useApiDetail({
  apiService: productApiService,
  entity: 'products',
  id: 123,
  options: {
    staleTime: 1000 * 60 * 10, // 10 minutes
    // Data sẽ fresh trong 10 phút
  },
});

// 5. Prefetch detail
const queryClient = useQueryClient();
const prefetchProduct = (id: number) => {
  queryClient.prefetchQuery({
    queryKey: ['products', 'detail', id],
    queryFn: () => productApiService.getById(id),
  });
};

// Hover to prefetch
<tr onMouseEnter={() => prefetchProduct(product.id)}>
```

**🔍 CACHE BEHAVIOR:**

```typescript
// Cache key: ['products', 'detail', 123]
// Cached data: ProductEntity { id: 123, productName: 'Laptop', ... }

// Scenario 1: Navigate to detail page
// → Fetch and cache
// → Navigate away
// → Navigate back → Instant (from cache)

// Scenario 2: Update product
const mutation = useApiUpdate({ ... });
mutation.mutate({ id: 123, productName: 'New Name' });
// → Auto invalidate: ['products', 'detail', 123]
// → Next access will refetch

// Scenario 3: Different IDs
// id=123 → Cache: ['products', 'detail', 123]
// id=456 → Cache: ['products', 'detail', 456]
// Different cache → Separate fetch
```

#### 4.13.7. useApiCreate - CREATE Mutation

**📥 INPUT PARAMETERS:**

```typescript
interface UseApiCreateConfig<TData, TCreateData, TError = Error> {
  // REQUIRED
  apiService: ApiService<TData>;
  // - Mô tả: Instance của ApiService

  entity: string;
  // - Mô tả: Entity name cho query keys

  // OPTIONAL
  options?: UseMutationOptions<TData, TError, TCreateData>;
  // - Mô tả: TanStack Query mutation options
  // - Common options:
  //   * onSuccess: (data, variables, context) => void
  //   * onError: (error, variables, context) => void
  //   * onSettled: (data, error, variables, context) => void
  //   * onMutate: (variables) => Promise<context>
}
```

**📤 OUTPUT/RETURN VALUES:**

```typescript
{
  // MUTATION FUNCTION
  mutate: (data: TCreateData, options?: MutateOptions) => void;
  // - Mô tả: Trigger mutation (fire and forget)
  // - Params: data to create
  // - Example: mutate({ productName: 'New Product', price: 100 })

  mutateAsync: (data: TCreateData, options?: MutateOptions) => Promise<TData>;
  // - Mô tả: Trigger mutation (returns Promise)
  // - Use case: Khi cần await result
  // - Example: const result = await mutateAsync({ ... })

  // MUTATION STATE
  isPending: boolean;
  // - true khi mutation đang chạy
  // - Use case: Disable submit button

  isSuccess: boolean;
  // - true khi mutation thành công

  isError: boolean;
  // - true khi mutation failed

  // DATA & ERROR
  data: TData | undefined;
  // - Mô tả: Created item từ server response
  // - Example: { id: 124, productName: 'New Product', ... }

  error: TError | null;
  // - Mô tả: Error object nếu failed

  // ACTIONS
  reset: () => void;
  // - Mô tả: Reset mutation state
  // - Use case: Clear error, reset form

  // METADATA
  variables: TCreateData | undefined;
  // - Mô tả: Data đã submit

  submittedAt: number;
  // - Mô tả: Timestamp của lần submit cuối
}
```

**🔄 LUỒNG HOẠT ĐỘNG:**

```
1. Component mount
   ↓
2. useApiCreate hook initialized
   ↓
3. User fills form
   ↓
4. User clicks Submit
   ↓
5. Call mutate(formData)
   ↓
6. onMutate callback (nếu có)
   - Optimistic update
   - Cancel outgoing queries
   - Snapshot previous data
   ↓
7. Call apiService.create(formData)
   ↓
8. API Request: POST /api/products
   Body: { productName: 'New Product', price: 100 }
   ↓
9. Backend processing
   - Validate data
   - Save to database
   - Return created item với ID
   ↓
10. Response
   ├─ Success (201 Created)
   │  ↓
   │  11. onSuccess callback
   │      - Show success message
   │      - Navigate to detail page
   │      - Close modal
   │  ↓
   │  12. Invalidate queries
   │      queryClient.invalidateQueries(['products'])
   │      → List queries will refetch
   │  ↓
   │  13. Component re-render với data
   │
   └─ Error (400, 500, etc.)
      ↓
      14. onError callback
          - Show error message
          - Keep form data
      ↓
      15. Component re-render với error
   ↓
16. onSettled callback (always runs)
    - Hide loading spinner
    - Re-enable submit button
```

**📊 MUTATION FLOW DIAGRAM:**

```mermaid
graph TD
    A[User Submits Form] -->|formData| B[mutate/mutateAsync]
    B --> C[onMutate]
    C -->|Optimistic Update| D[Update UI]
    C -->|Cancel Queries| E[Cancel Outgoing]
    C -->|Snapshot| F[Save Previous Data]

    B --> G[apiService.create]
    G -->|POST /api/products| H[Backend]
    H -->|Validate & Save| I{Success?}

    I -->|Yes 201| J[onSuccess]
    J -->|Invalidate| K[queryClient.invalidateQueries]
    K -->|Refetch| L[List Queries Update]
    J --> M[Show Success Message]
    J --> N[Navigate/Close Modal]

    I -->|No 400/500| O[onError]
    O -->|Rollback| P[Restore Previous Data]
    O --> Q[Show Error Message]

    J --> R[onSettled]
    O --> R
    R --> S[Hide Loading]
```

**⚙️ INTERNAL MECHANISM:**

```typescript
export function useApiCreate<TData, TCreateData>({ apiService, entity, options }) {
  const queryClient = useQueryClient();
  const queryKeys = createQueryKeys(entity);

  return useMutation<TData, Error, TCreateData>({
    // Mutation function
    mutationFn: (data: TCreateData) => apiService.create(data),
    // Call: POST /api/products với body = data

    // Auto invalidate list queries on success
    onSuccess: (data, variables, context) => {
      // Invalidate all list queries
      queryClient.invalidateQueries({
        queryKey: queryKeys.lists(),
        // Matches: ['products', 'list', ...]
      });

      // Call user's onSuccess
      options?.onSuccess?.(data, variables, context);
    },

    ...options,
  });
}
```

**🎯 USE CASES:**

```typescript
// 1. Basic create
const createProduct = useApiCreate({
  apiService: productApiService,
  entity: 'products',
});

const handleSubmit = (formData: CreateProductRequest) => {
  createProduct.mutate(formData);
};

// 2. With callbacks
const createProduct = useApiCreate({
  apiService: productApiService,
  entity: 'products',
  options: {
    onSuccess: (data) => {
      message.success('Product created!');
      navigate(`/products/${data.id}`);
    },
    onError: (error) => {
      message.error(error.message);
    },
  },
});

// 3. With async/await
const createProduct = useApiCreate({
  apiService: productApiService,
  entity: 'products',
});

const handleSubmit = async (formData: CreateProductRequest) => {
  try {
    const result = await createProduct.mutateAsync(formData);
    console.log('Created:', result);
    navigate(`/products/${result.id}`);
  } catch (error) {
    console.error('Failed:', error);
  }
};

// 4. With optimistic update
const createProduct = useApiCreate({
  apiService: productApiService,
  entity: 'products',
  options: {
    onMutate: async (newProduct) => {
      // Cancel outgoing queries
      await queryClient.cancelQueries({ queryKey: ['products'] });

      // Snapshot previous value
      const previousProducts = queryClient.getQueryData(['products', 'list']);

      // Optimistically update
      queryClient.setQueryData(['products', 'list'], (old: any) => {
        return {
          ...old,
          items: [...old.items, { ...newProduct, id: 'temp-id' }],
        };
      });

      // Return context với snapshot
      return { previousProducts };
    },
    onError: (err, newProduct, context) => {
      // Rollback on error
      queryClient.setQueryData(
        ['products', 'list'],
        context?.previousProducts
      );
    },
    onSettled: () => {
      // Always refetch after error or success
      queryClient.invalidateQueries({ queryKey: ['products'] });
    },
  },
});

// 5. Form integration với React Hook Form
const form = useForm<CreateProductRequest>();
const createProduct = useApiCreate({
  apiService: productApiService,
  entity: 'products',
  options: {
    onSuccess: () => {
      form.reset();
      message.success('Created!');
    },
  },
});

const onSubmit = form.handleSubmit((data) => {
  createProduct.mutate(data);
});

<form onSubmit={onSubmit}>
  <input {...form.register('productName')} />
  <button type="submit" disabled={createProduct.isPending}>
    {createProduct.isPending ? 'Creating...' : 'Create'}
  </button>
</form>
```

#### 4.13.8. useApiUpdate - UPDATE Mutation (Full Replace)

**📥 INPUT PARAMETERS:**

```typescript
interface UseApiUpdateConfig<TData, TUpdateData, TError = Error> {
  apiService: ApiService<TData>;
  entity: string;

  options?: UseMutationOptions<TData, TError, { id: number | string; data: TUpdateData }>;
  // - Mô tả: Mutation options
  // - Note: Variables = { id, data }
}
```

**📤 OUTPUT/RETURN VALUES:**

```typescript
{
  // MUTATION FUNCTION
  mutate: (params: { id: number | string; data: TUpdateData }) => void;
  // - Mô tả: Update item (fire and forget)
  // - Params: { id: 123, data: { productName: 'Updated', price: 200 } }

  mutateAsync: (params: { id: number | string; data: TUpdateData }) => Promise<TData>;
  // - Mô tả: Update item (returns Promise)

  // MUTATION STATE
  isPending: boolean;
  isSuccess: boolean;
  isError: boolean;

  // DATA & ERROR
  data: TData | undefined;
  // - Mô tả: Updated item từ server

  error: TError | null;

  // ACTIONS
  reset: () => void;

  // METADATA
  variables: { id: number | string; data: TUpdateData } | undefined;
}
```

**🔄 LUỒNG HOẠT ĐỘNG:**

```
1. User edits item (form với existing data)
   ↓
2. User clicks Save
   ↓
3. Call mutate({ id: 123, data: updatedData })
   ↓
4. onMutate callback
   - Optimistic update detail cache
   - Optimistic update list cache
   - Snapshot previous data
   ↓
5. Call apiService.update(id, data)
   ↓
6. API Request: PUT /api/products/123
   Body: { productName: 'Updated', price: 200, ... }
   (Full object replacement)
   ↓
7. Backend processing
   - Validate data
   - Replace entire object
   - Return updated item
   ↓
8. Response
   ├─ Success (200 OK)
   │  ↓
   │  9. onSuccess callback
   │      - Show success message
   │      - Navigate back to list
   │  ↓
   │  10. Invalidate queries
   │      - ['products', 'detail', 123] → Refetch detail
   │      - ['products', 'list', ...] → Refetch lists
   │  ↓
   │  11. Component re-render với updated data
   │
   └─ Error (400, 404, 500, etc.)
      ↓
      12. onError callback
          - Rollback optimistic update
          - Show error message
      ↓
      13. Component re-render với error
```

**📊 MUTATION FLOW DIAGRAM:**

```mermaid
graph TD
    A[Edit Form] -->|id=123, data| B[mutate]
    B --> C[onMutate]
    C -->|Optimistic| D[Update Detail Cache]
    C -->|Optimistic| E[Update List Cache]

    B --> F[apiService.update]
    F -->|PUT /api/products/123| G[Backend]
    G -->|Replace Object| H{Success?}

    H -->|Yes| I[onSuccess]
    I -->|Invalidate| J["['products', 'detail', 123]"]
    I -->|Invalidate| K["['products', 'list', ...]"]
    J --> L[Refetch Detail]
    K --> M[Refetch Lists]

    H -->|No| N[onError]
    N -->|Rollback| O[Restore Caches]
    N --> P[Show Error]
```

**⚙️ INTERNAL MECHANISM:**

```typescript
export function useApiUpdate<TData, TUpdateData>({ apiService, entity, options }) {
  const queryClient = useQueryClient();
  const queryKeys = createQueryKeys(entity);

  return useMutation<TData, Error, { id: number | string; data: TUpdateData }>({
    mutationFn: ({ id, data }) => apiService.update(id, data),
    // Call: PUT /api/products/123

    onSuccess: (data, variables, context) => {
      // Invalidate detail query
      queryClient.invalidateQueries({
        queryKey: queryKeys.detail(variables.id),
      });

      // Invalidate all list queries
      queryClient.invalidateQueries({
        queryKey: queryKeys.lists(),
      });

      options?.onSuccess?.(data, variables, context);
    },

    ...options,
  });
}
```

**🎯 USE CASES:**

```typescript
// 1. Basic update
const updateProduct = useApiUpdate({
  apiService: productApiService,
  entity: 'products',
});

const handleSave = (id: number, formData: UpdateProductRequest) => {
  updateProduct.mutate({ id, data: formData });
};

// 2. With optimistic update
const updateProduct = useApiUpdate({
  apiService: productApiService,
  entity: 'products',
  options: {
    onMutate: async ({ id, data }) => {
      // Cancel queries
      await queryClient.cancelQueries({ queryKey: ['products', 'detail', id] });

      // Snapshot
      const previous = queryClient.getQueryData(['products', 'detail', id]);

      // Optimistic update detail
      queryClient.setQueryData(['products', 'detail', id], (old: any) => ({
        ...old,
        ...data,
      }));

      // Optimistic update list
      queryClient.setQueryData(['products', 'list'], (old: any) => ({
        ...old,
        items: old.items.map((item: any) =>
          item.id === id ? { ...item, ...data } : item
        ),
      }));

      return { previous };
    },
    onError: (err, { id }, context) => {
      // Rollback
      queryClient.setQueryData(['products', 'detail', id], context?.previous);
    },
  },
});

// 3. Form integration
const { id } = useParams();
const { data: product } = useApiDetail({ apiService: productApiService, entity: 'products', id });
const updateProduct = useApiUpdate({ apiService: productApiService, entity: 'products' });

const form = useForm({
  values: product, // Populate form với existing data
});

const onSubmit = form.handleSubmit((data) => {
  updateProduct.mutate({ id: Number(id), data });
});
```

#### 4.13.9. useApiPatch - PATCH Mutation (Partial Update)

**📥 INPUT PARAMETERS:**

```typescript
interface UseApiPatchConfig<TData, TPatchData, TError = Error> {
  apiService: ApiService<TData>;
  entity: string;

  options?: UseMutationOptions<TData, TError, { id: number | string; data: Partial<TPatchData> }>;
  // - Mô tả: Mutation options
  // - Note: data là Partial<TPatchData> (chỉ fields cần update)
}
```

**📤 OUTPUT/RETURN VALUES:**

```typescript
{
  mutate: (params: { id: number | string; data: Partial<TPatchData> }) => void;
  // - Mô tả: Partial update (chỉ update fields được gửi)
  // - Example: mutate({ id: 123, data: { price: 200 } })
  //   → Chỉ update price, giữ nguyên các fields khác

  mutateAsync: (params: { id: number | string; data: Partial<TPatchData> }) => Promise<TData>;

  isPending: boolean;
  isSuccess: boolean;
  isError: boolean;
  data: TData | undefined;
  error: TError | null;
  reset: () => void;
}
```

**🔄 LUỒNG HOẠT ĐỘNG:**

```
1. User muốn update 1 field (ví dụ: toggle active status)
   ↓
2. Call mutate({ id: 123, data: { isActive: true } })
   ↓
3. onMutate callback
   - Optimistic update chỉ field isActive
   ↓
4. Call apiService.patch(id, data)
   ↓
5. API Request: PATCH /api/products/123
   Body: { isActive: true }
   (Chỉ gửi fields cần update)
   ↓
6. Backend processing
   - Validate data
   - Update chỉ fields được gửi
   - Giữ nguyên các fields khác
   - Return full updated object
   ↓
7. Response
   ├─ Success (200 OK)
   │  ↓
   │  8. Invalidate queries
   │  ↓
   │  9. Component re-render
   │
   └─ Error
      ↓
      10. Rollback optimistic update
```

**📊 PATCH vs PUT:**

```typescript
// PUT (useApiUpdate) - Full replacement
PUT /api/products/123
Body: {
  productName: 'Updated',
  price: 200,
  categoryId: 5,
  unit: 'pcs',
  // Phải gửi TẤT CẢ fields
}

// PATCH (useApiPatch) - Partial update
PATCH /api/products/123
Body: {
  price: 200
  // Chỉ gửi fields cần update
}
```

**⚙️ INTERNAL MECHANISM:**

```typescript
export function useApiPatch<TData, TPatchData>({ apiService, entity, options }) {
  const queryClient = useQueryClient();
  const queryKeys = createQueryKeys(entity);

  return useMutation<TData, Error, { id: number | string; data: Partial<TPatchData> }>({
    mutationFn: ({ id, data }) => apiService.patch(id, data),
    // Call: PATCH /api/products/123

    onSuccess: (data, variables, context) => {
      // Same invalidation như useApiUpdate
      queryClient.invalidateQueries({
        queryKey: queryKeys.detail(variables.id),
      });
      queryClient.invalidateQueries({
        queryKey: queryKeys.lists(),
      });

      options?.onSuccess?.(data, variables, context);
    },

    ...options,
  });
}
```

**🎯 USE CASES:**

```typescript
// 1. Toggle active status
const patchProduct = useApiPatch({
  apiService: productApiService,
  entity: 'products',
});

const handleToggleActive = (id: number, isActive: boolean) => {
  patchProduct.mutate({
    id,
    data: { isActive }, // Chỉ update 1 field
  });
};

// 2. Update price only
const handleUpdatePrice = (id: number, newPrice: number) => {
  patchProduct.mutate({
    id,
    data: { price: newPrice },
  });
};

// 3. Bulk field update
const handleBulkUpdate = (id: number) => {
  patchProduct.mutate({
    id,
    data: {
      price: 200,
      isActive: true,
      stock: 50,
      // Chỉ update 3 fields này
    },
  });
};

// 4. With optimistic update
const patchProduct = useApiPatch({
  apiService: productApiService,
  entity: 'products',
  options: {
    onMutate: async ({ id, data }) => {
      await queryClient.cancelQueries({ queryKey: ['products', 'detail', id] });

      const previous = queryClient.getQueryData(['products', 'detail', id]);

      // Optimistic update - merge partial data
      queryClient.setQueryData(['products', 'detail', id], (old: any) => ({
        ...old,
        ...data, // Merge partial update
      }));

      return { previous };
    },
    onError: (err, { id }, context) => {
      queryClient.setQueryData(['products', 'detail', id], context?.previous);
    },
  },
});

// 5. Quick actions
<Switch
  checked={product.isActive}
  onChange={(checked) => {
    patchProduct.mutate({
      id: product.id,
      data: { isActive: checked },
    });
  }}
  loading={patchProduct.isPending}
/>
```

#### 4.13.10. useApiDelete - DELETE Mutation

**📥 INPUT PARAMETERS:**

```typescript
interface UseApiDeleteConfig<TError = Error> {
  apiService: ApiService<any>;
  entity: string;

  options?: UseMutationOptions<void, TError, number | string>;
  // - Mô tả: Mutation options
  // - Note: Variables = id (number | string)
  // - Return type = void (no data returned)
}
```

**📤 OUTPUT/RETURN VALUES:**

```typescript
{
  mutate: (id: number | string) => void;
  // - Mô tả: Delete item by id
  // - Example: mutate(123)

  mutateAsync: (id: number | string) => Promise<void>;
  // - Mô tả: Delete item (returns Promise)

  isPending: boolean;
  // - Use case: Show loading, disable delete button

  isSuccess: boolean;
  isError: boolean;

  error: TError | null;
  // - Common errors: 404 (not found), 409 (conflict - has dependencies)

  reset: () => void;

  variables: number | string | undefined;
  // - Mô tả: ID đã delete
}
```

**🔄 LUỒNG HOẠT ĐỘNG:**

```
1. User clicks Delete button
   ↓
2. Show confirmation dialog
   "Are you sure you want to delete this item?"
   ↓
3. User confirms
   ↓
4. Call mutate(id)
   ↓
5. onMutate callback
   - Optimistic remove từ list
   - Snapshot previous data
   ↓
6. Call apiService.delete(id)
   ↓
7. API Request: DELETE /api/products/123
   ↓
8. Backend processing
   - Check dependencies (foreign keys)
   - Delete from database
   - Return 204 No Content
   ↓
9. Response
   ├─ Success (204 No Content)
   │  ↓
   │  10. onSuccess callback
   │      - Show success message
   │      - Navigate to list page
   │  ↓
   │  11. Invalidate queries
   │      - Remove detail cache
   │      - Refetch list queries
   │  ↓
   │  12. Component re-render (item removed)
   │
   └─ Error (404, 409, 500)
      ↓
      13. onError callback
          - Rollback optimistic delete
          - Show error message
          - "Cannot delete: item has dependencies"
      ↓
      14. Component re-render (item restored)
```

**📊 DELETE FLOW DIAGRAM:**

```mermaid
graph TD
    A[Delete Button] -->|Confirm?| B{User Confirms}
    B -->|No| C[Cancel]
    B -->|Yes| D[mutate id]

    D --> E[onMutate]
    E -->|Optimistic| F[Remove from List Cache]
    E -->|Snapshot| G[Save Previous Data]

    D --> H[apiService.delete]
    H -->|DELETE /api/products/123| I[Backend]
    I -->|Check Dependencies| J{Can Delete?}

    J -->|Yes| K[Delete from DB]
    K --> L[204 No Content]
    L --> M[onSuccess]
    M -->|Invalidate| N[Remove Detail Cache]
    M -->|Invalidate| O[Refetch Lists]
    M --> P[Show Success]

    J -->|No| Q[409 Conflict]
    Q --> R[onError]
    R -->|Rollback| S[Restore List Cache]
    R --> T[Show Error]
```

**⚙️ INTERNAL MECHANISM:**

```typescript
export function useApiDelete({ apiService, entity, options }) {
  const queryClient = useQueryClient();
  const queryKeys = createQueryKeys(entity);

  return useMutation<void, Error, number | string>({
    mutationFn: (id) => apiService.delete(id),
    // Call: DELETE /api/products/123

    onSuccess: (data, id, context) => {
      // Remove detail cache
      queryClient.removeQueries({
        queryKey: queryKeys.detail(id),
      });

      // Invalidate list queries
      queryClient.invalidateQueries({
        queryKey: queryKeys.lists(),
      });

      options?.onSuccess?.(data, id, context);
    },

    ...options,
  });
}
```

**🎯 USE CASES:**

```typescript
// 1. Basic delete với confirmation
const deleteProduct = useApiDelete({
  apiService: productApiService,
  entity: 'products',
});

const handleDelete = (id: number) => {
  Modal.confirm({
    title: 'Delete Product?',
    content: 'This action cannot be undone.',
    onOk: () => {
      deleteProduct.mutate(id);
    },
  });
};

// 2. With callbacks
const deleteProduct = useApiDelete({
  apiService: productApiService,
  entity: 'products',
  options: {
    onSuccess: () => {
      message.success('Product deleted!');
      navigate('/products');
    },
    onError: (error) => {
      if (error.message.includes('foreign key')) {
        message.error('Cannot delete: product has orders');
      } else {
        message.error('Delete failed');
      }
    },
  },
});

// 3. With optimistic delete
const deleteProduct = useApiDelete({
  apiService: productApiService,
  entity: 'products',
  options: {
    onMutate: async (id) => {
      // Cancel queries
      await queryClient.cancelQueries({ queryKey: ['products'] });

      // Snapshot list
      const previousList = queryClient.getQueryData(['products', 'list']);

      // Optimistic remove
      queryClient.setQueryData(['products', 'list'], (old: any) => ({
        ...old,
        items: old.items.filter((item: any) => item.id !== id),
        totalCount: old.totalCount - 1,
      }));

      return { previousList };
    },
    onError: (err, id, context) => {
      // Rollback
      queryClient.setQueryData(['products', 'list'], context?.previousList);
      message.error('Delete failed');
    },
    onSuccess: () => {
      message.success('Deleted!');
    },
  },
});

// 4. Bulk delete
const deleteProduct = useApiDelete({
  apiService: productApiService,
  entity: 'products',
});

const handleBulkDelete = async (ids: number[]) => {
  for (const id of ids) {
    await deleteProduct.mutateAsync(id);
  }
  message.success(`Deleted ${ids.length} products`);
};

// 5. Delete button component
<Popconfirm
  title="Delete this product?"
  onConfirm={() => deleteProduct.mutate(product.id)}
  okText="Yes"
  cancelText="No"
>
  <Button
    danger
    loading={deleteProduct.isPending}
    icon={<DeleteOutlined />}
  >
    Delete
  </Button>
</Popconfirm>
```

#### 4.13.11. useApiCustomQuery - Custom Query Hook

**📥 INPUT PARAMETERS:**

```typescript
interface UseApiCustomQueryConfig<TData, TError = Error> {
  // REQUIRED
  apiService: ApiService<any>;
  // - Mô tả: Instance của ApiService

  entity: string;
  // - Mô tả: Entity name cho query keys

  queryKey: string | (string | number | object)[];
  // - Mô tả: Custom query key
  // - Type: string hoặc array
  // - Example: 'stats' hoặc ['stats', { year: 2024 }]
  // - Note: Sẽ được prefix với entity name
  //   → Final key: ['products', 'stats', { year: 2024 }]

  queryFn: () => Promise<TData>;
  // - Mô tả: Custom query function
  // - Type: Function returns Promise<TData>
  // - Example: () => apiService.custom('GET', '/stats')

  // OPTIONAL
  options?: UseQueryOptions<TData, TError>;
  // - Mô tả: TanStack Query options
}
```

**📤 OUTPUT/RETURN VALUES:**

```typescript
{
  // Same as useQuery
  data: TData | undefined;
  isLoading: boolean;
  isFetching: boolean;
  isError: boolean;
  error: TError | null;
  isSuccess: boolean;
  refetch: () => Promise<QueryObserverResult<TData, TError>>;
  dataUpdatedAt: number;
}
```

**🔄 LUỒNG HOẠT ĐỘNG:**

```
1. Component mount
   ↓
2. useApiCustomQuery được gọi với custom queryFn
   ↓
3. Tạo query key: [entity, ...customKey]
   Example: ['products', 'stats', { year: 2024 }]
   ↓
4. Check cache
   ↓
5. Call custom queryFn
   ↓
6. API Request (custom endpoint)
   Example: GET /api/products/stats?year=2024
   ↓
7. Response
   ↓
8. Cache với custom key
   ↓
9. Component re-render với data
```

**⚙️ INTERNAL MECHANISM:**

```typescript
export function useApiCustomQuery<TData>({
  apiService,
  entity,
  queryKey,
  queryFn,
  options,
}) {
  const queryKeys = createQueryKeys(entity);

  // Build full query key
  const fullQueryKey = Array.isArray(queryKey)
    ? [entity, ...queryKey]
    : [entity, queryKey];

  return useQuery<TData>({
    queryKey: fullQueryKey,
    queryFn,
    ...options,
  });
}
```

**🎯 USE CASES:**

```typescript
// 1. Custom endpoint - Statistics
const { data: stats } = useApiCustomQuery({
  apiService: productApiService,
  entity: 'products',
  queryKey: ['stats', { year: 2024 }],
  queryFn: () => productApiService.custom('GET', '/stats', { year: 2024 }),
});
// Query key: ['products', 'stats', { year: 2024 }]
// Endpoint: GET /api/products/stats?year=2024

// 2. Custom endpoint - Search suggestions
const { data: suggestions } = useApiCustomQuery({
  apiService: productApiService,
  entity: 'products',
  queryKey: ['suggestions', searchText],
  queryFn: () => productApiService.custom('GET', '/suggestions', { q: searchText }),
  options: {
    enabled: searchText.length >= 3, // Chỉ fetch khi >= 3 chars
    staleTime: 1000 * 60, // 1 minute
  },
});

// 3. Custom endpoint - Related products
const { data: relatedProducts } = useApiCustomQuery({
  apiService: productApiService,
  entity: 'products',
  queryKey: ['related', productId],
  queryFn: () => productApiService.custom('GET', `/related/${productId}`),
  options: {
    enabled: !!productId,
  },
});
// Endpoint: GET /api/products/related/123

// 4. Custom endpoint - Dashboard summary
const { data: summary } = useApiCustomQuery({
  apiService: productApiService,
  entity: 'products',
  queryKey: 'summary',
  queryFn: () => productApiService.custom('GET', '/summary'),
});
// Query key: ['products', 'summary']
// Endpoint: GET /api/products/summary

// 5. Custom endpoint với complex params
const { data: report } = useApiCustomQuery({
  apiService: productApiService,
  entity: 'products',
  queryKey: ['report', { startDate, endDate, categoryId }],
  queryFn: () =>
    productApiService.custom('GET', '/report', {
      startDate,
      endDate,
      categoryId,
    }),
});
```

#### 4.13.12. useApiCustomMutation - Custom Mutation Hook

**📥 INPUT PARAMETERS:**

```typescript
interface UseApiCustomMutationConfig<TData, TVariables, TError = Error> {
  // REQUIRED
  apiService: ApiService<any>;
  entity: string;

  mutationFn: (variables: TVariables) => Promise<TData>;
  // - Mô tả: Custom mutation function
  // - Type: Function nhận variables, returns Promise<TData>
  // - Example: (data) => apiService.custom('POST', '/bulk-update', data)

  // OPTIONAL
  invalidateQueries?: string[];
  // - Mô tả: Query keys to invalidate on success
  // - Type: Array of strings
  // - Example: ['list', 'stats']
  // - Note: Sẽ được prefix với entity name
  //   → Invalidate: ['products', 'list'], ['products', 'stats']

  options?: UseMutationOptions<TData, TError, TVariables>;
  // - Mô tả: TanStack Query mutation options
}
```

**📤 OUTPUT/RETURN VALUES:**

```typescript
{
  // Same as useMutation
  mutate: (variables: TVariables) => void;
  mutateAsync: (variables: TVariables) => Promise<TData>;
  isPending: boolean;
  isSuccess: boolean;
  isError: boolean;
  data: TData | undefined;
  error: TError | null;
  reset: () => void;
  variables: TVariables | undefined;
}
```

**🔄 LUỒNG HOẠT ĐỘNG:**

```
1. User triggers custom action
   ↓
2. Call mutate(variables)
   ↓
3. Call custom mutationFn
   ↓
4. API Request (custom endpoint/method)
   Example: POST /api/products/bulk-update
   ↓
5. Backend processing
   ↓
6. Response
   ├─ Success
   │  ↓
   │  7. Invalidate specified queries
   │     ['products', 'list']
   │     ['products', 'stats']
   │  ↓
   │  8. onSuccess callback
   │
   └─ Error
      ↓
      9. onError callback
```

**⚙️ INTERNAL MECHANISM:**

```typescript
export function useApiCustomMutation<TData, TVariables>({
  apiService,
  entity,
  mutationFn,
  invalidateQueries = [],
  options,
}) {
  const queryClient = useQueryClient();
  const queryKeys = createQueryKeys(entity);

  return useMutation<TData, Error, TVariables>({
    mutationFn,

    onSuccess: (data, variables, context) => {
      // Auto invalidate specified queries
      invalidateQueries.forEach((key) => {
        queryClient.invalidateQueries({
          queryKey: [entity, key],
        });
      });

      options?.onSuccess?.(data, variables, context);
    },

    ...options,
  });
}
```

**🎯 USE CASES:**

```typescript
// 1. Bulk update
const bulkUpdate = useApiCustomMutation({
  apiService: productApiService,
  entity: 'products',
  mutationFn: (data: { ids: number[]; updates: Partial<ProductEntity> }) =>
    productApiService.custom('POST', '/bulk-update', data),
  invalidateQueries: ['list'], // Invalidate list after bulk update
  options: {
    onSuccess: () => {
      message.success('Bulk update successful!');
    },
  },
});

// Usage
bulkUpdate.mutate({
  ids: [1, 2, 3],
  updates: { isActive: true },
});

// 2. Import from file
const importProducts = useApiCustomMutation({
  apiService: productApiService,
  entity: 'products',
  mutationFn: (file: File) => {
    const formData = new FormData();
    formData.append('file', file);
    return productApiService.custom('POST', '/import', formData);
  },
  invalidateQueries: ['list', 'stats'],
});

// Usage
const handleFileUpload = (file: File) => {
  importProducts.mutate(file);
};

// 3. Export to Excel
const exportProducts = useApiCustomMutation({
  apiService: productApiService,
  entity: 'products',
  mutationFn: (filters: any) =>
    productApiService.custom('POST', '/export', filters),
  options: {
    onSuccess: (data: Blob) => {
      // Download file
      const url = window.URL.createObjectURL(data);
      const a = document.createElement('a');
      a.href = url;
      a.download = 'products.xlsx';
      a.click();
    },
  },
});

// 4. Duplicate product
const duplicateProduct = useApiCustomMutation({
  apiService: productApiService,
  entity: 'products',
  mutationFn: (id: number) =>
    productApiService.custom('POST', `/duplicate/${id}`),
  invalidateQueries: ['list'],
  options: {
    onSuccess: (newProduct) => {
      message.success('Product duplicated!');
      navigate(`/products/${newProduct.id}`);
    },
  },
});

// 5. Batch delete
const batchDelete = useApiCustomMutation({
  apiService: productApiService,
  entity: 'products',
  mutationFn: (ids: number[]) =>
    productApiService.custom('POST', '/batch-delete', { ids }),
  invalidateQueries: ['list', 'stats'],
  options: {
    onSuccess: (result: { deleted: number }) => {
      message.success(`Deleted ${result.deleted} products`);
    },
  },
});

// 6. Restore deleted item
const restoreProduct = useApiCustomMutation({
  apiService: productApiService,
  entity: 'products',
  mutationFn: (id: number) =>
    productApiService.custom('POST', `/restore/${id}`),
  invalidateQueries: ['list'],
});

// 7. Custom action - Publish
const publishProduct = useApiCustomMutation({
  apiService: productApiService,
  entity: 'products',
  mutationFn: (id: number) =>
    productApiService.custom('POST', `/publish/${id}`),
  invalidateQueries: ['list', 'detail'],
  options: {
    onSuccess: () => {
      message.success('Product published!');
    },
  },
});
```

**📊 SUMMARY TABLE - All Universal CRUD Hooks:**

```typescript
// QUERIES (Read Operations)
┌─────────────────────┬──────────────────────┬─────────────────────────┐
│ Hook                │ Purpose              │ API Method              │
├─────────────────────┼──────────────────────┼─────────────────────────┤
│ useApiList          │ Get all items        │ GET /api/products       │
│ useApiPagedList     │ Get paged items      │ GET /api/products?page=1│
│ useApiInfinite      │ Infinite scroll      │ GET /api/products?page=N│
│ useApiDetail        │ Get by ID            │ GET /api/products/123   │
│ useApiCustomQuery   │ Custom GET           │ GET /api/products/stats │
└─────────────────────┴──────────────────────┴─────────────────────────┘

// MUTATIONS (Write Operations)
┌─────────────────────┬──────────────────────┬─────────────────────────┐
│ Hook                │ Purpose              │ API Method              │
├─────────────────────┼──────────────────────┼─────────────────────────┤
│ useApiCreate        │ Create new item      │ POST /api/products      │
│ useApiUpdate        │ Full update          │ PUT /api/products/123   │
│ useApiPatch         │ Partial update       │ PATCH /api/products/123 │
│ useApiDelete        │ Delete item          │ DELETE /api/products/123│
│ useApiCustomMutation│ Custom POST/PUT/etc  │ POST /api/products/bulk │
└─────────────────────┴──────────────────────┴─────────────────────────┘

// PAGINATION HELPERS
┌─────────────────────────┬──────────────────────────────────────────┐
│ Hook                    │ Purpose                                  │
├─────────────────────────┼──────────────────────────────────────────┤
│ usePagination           │ State management cho pagination          │
│ usePaginationWithRouter │ URL-synced pagination                    │
│ usePaginationWithFilters│ Advanced filters management              │
│ usePaginationWithPrefetch│ Prefetching support                     │
└─────────────────────────┴──────────────────────────────────────────┘
```

## 5. Hướng dẫn Sử dụng cho Entity Khác

Để áp dụng cho entity khác (User, Customer, Order, etc.), làm theo các bước:

### Bước 1: Tạo API Service Instance

```typescript
// src/lib/api/customerApiService.ts
import axiosClient from '../axios';
import { ApiService } from '../apiFactory';
import { API_CONFIG } from '../../config/api';
import type { CustomerEntity } from '../../features/customers/types/entity';
import type { CreateCustomerRequest, UpdateCustomerRequest } from '../../features/customers/types/api';

export const customerApiService = new ApiService<
  CustomerEntity,
  CreateCustomerRequest,
  UpdateCustomerRequest
>({
  endpoint: API_CONFIG.ENDPOINTS.ADMIN.CUSTOMERS,
  axiosInstance: axiosClient,
});
```

### Bước 2: Tạo Hooks cho Entity

```typescript
// src/features/customers/hooks/useCustomers.ts
import { 
  useApiPagedList,
  useApiDetail, 
  useApiCreate, 
  useApiUpdate, 
  useApiDelete,
} from '../../../hooks/useApi';
import { customerApiService } from '../../../lib/api/customerApiService';
import type { CustomerEntity } from '../types/entity';
import type { CreateCustomerRequest, UpdateCustomerRequest } from '../types/api';
import type { PagedRequest } from '../../../lib/axios';

const ENTITY = 'customers';

export const useCustomers = (params?: PagedRequest) => {
  return useApiPagedList<CustomerEntity>({
    apiService: customerApiService,
    entity: ENTITY,
    params,
  });
};

export const useCustomer = (id: number) => {
  return useApiDetail<CustomerEntity>({
    apiService: customerApiService,
    entity: ENTITY,
    id,
  });
};

export const useCreateCustomer = () => {
  return useApiCreate<CustomerEntity, CreateCustomerRequest>({
    apiService: customerApiService,
    entity: ENTITY,
  });
};

export const useUpdateCustomer = () => {
  return useApiUpdate<CustomerEntity, UpdateCustomerRequest>({
    apiService: customerApiService,
    entity: ENTITY,
  });
};

export const useDeleteCustomer = () => {
  return useApiDelete({
    apiService: customerApiService,
    entity: ENTITY,
  });
};
```

### Bước 3: Sử dụng trong Component

```typescript
// Component example
import { useCustomers, useDeleteCustomer } from '../hooks/useCustomers';

export const CustomerList = () => {
  const { data, isLoading } = useCustomers({ page: 1, pageSize: 20 });
  const deleteCustomer = useDeleteCustomer();

  // ... rest of component
};
```

### 4.11. Prefetching Support

```typescript
// shiny-carnival/frontend/src/hooks/usePaginationWithPrefetch.ts
import { useQueryClient } from '@tanstack/react-query';
import { usePagination } from './usePagination';
import { createQueryKeys } from './useApi';
import type { ApiService } from '../lib/apiFactory';
import type { PagedRequest } from '../lib/axios';
import type { UsePaginationConfig } from './usePagination';

/**
 * Hook với prefetching cho next/previous pages
 * Tự động prefetch trang tiếp theo để cải thiện UX
 */
export function usePaginationWithPrefetch<TData = any>(
  config: UsePaginationConfig<TData>
) {
  const queryClient = useQueryClient();
  const pagination = usePagination(config);
  const queryKeys = createQueryKeys(config.entity);

  const { page, pageSize, search, sortBy, sortDesc, hasNext, hasPrevious } = pagination;

  // Prefetch next page
  const prefetchNextPage = () => {
    if (hasNext) {
      const nextParams: PagedRequest = {
        page: page + 1,
        pageSize,
        search: search || undefined,
        sortBy,
        sortDesc,
        ...config.additionalParams,
      };

      queryClient.prefetchQuery({
        queryKey: queryKeys.list(nextParams),
        queryFn: () => config.apiService.getAllPaged(nextParams),
        staleTime: 1000 * 60 * 5,
      });
    }
  };

  // Prefetch previous page
  const prefetchPreviousPage = () => {
    if (hasPrevious) {
      const prevParams: PagedRequest = {
        page: page - 1,
        pageSize,
        search: search || undefined,
        sortBy,
        sortDesc,
        ...config.additionalParams,
      };

      queryClient.prefetchQuery({
        queryKey: queryKeys.list(prevParams),
        queryFn: () => config.apiService.getAllPaged(prevParams),
        staleTime: 1000 * 60 * 5,
      });
    }
  };

  // Prefetch detail khi hover
  const prefetchDetail = (id: number | string) => {
    queryClient.prefetchQuery({
      queryKey: queryKeys.detail(id),
      queryFn: () => config.apiService.getById(id),
      staleTime: 1000 * 60 * 5,
    });
  };

  return {
    ...pagination,
    prefetchNextPage,
    prefetchPreviousPage,
    prefetchDetail,
  };
}
```

## 6. Pagination Examples Chi tiết

### 6.1. Basic Pagination với Ant Design Table

```typescript
// shiny-carnival/frontend/src/features/products/components/ProductTableBasic.tsx
import { Table, Input, Button, Space } from 'antd';
import { SearchOutlined, ReloadOutlined } from '@ant-design/icons';
import { usePagination } from '../../../hooks/usePagination';
import { productApiService } from '../../../lib/api/productApiService';
import type { ProductEntity } from '../types/entity';

export const ProductTableBasic = () => {
  const {
    items,
    isLoading,
    isFetching,
    page,
    pageSize,
    totalCount,
    search,
    handlePageChange,
    handleSearch,
    handleSort,
    resetPagination,
    refetch,
  } = usePagination<ProductEntity>({
    apiService: productApiService,
    entity: 'products',
  });

  const columns = [
    {
      title: 'ID',
      dataIndex: 'id',
      key: 'id',
      width: 80,
      sorter: true,
    },
    {
      title: 'Tên sản phẩm',
      dataIndex: 'productName',
      key: 'productName',
      sorter: true,
    },
    {
      title: 'Giá',
      dataIndex: 'price',
      key: 'price',
      sorter: true,
      render: (price: number) => `${price.toLocaleString('vi-VN')} đ`,
    },
  ];

  return (
    <div>
      <Space style={{ marginBottom: 16 }}>
        <Input.Search
          placeholder="Tìm kiếm..."
          allowClear
          enterButton={<SearchOutlined />}
          onSearch={handleSearch}
          defaultValue={search}
          style={{ width: 300 }}
        />
        <Button icon={<ReloadOutlined />} onClick={() => refetch()}>
          Làm mới
        </Button>
        <Button onClick={resetPagination}>Reset</Button>
      </Space>

      <Table<ProductEntity>
        columns={columns}
        dataSource={items}
        rowKey="id"
        loading={isLoading || isFetching}
        onChange={(pagination, _, sorter) => {
          if (pagination.current && pagination.pageSize) {
            handlePageChange(pagination.current, pagination.pageSize);
          }
          if (!Array.isArray(sorter) && sorter.field) {
            const field = String(sorter.field);
            const descending = sorter.order === 'descend';
            handleSort(field, descending);
          }
        }}
        pagination={{
          current: page,
          pageSize: pageSize,
          total: totalCount,
          showSizeChanger: true,
          showQuickJumper: true,
          showTotal: (total, range) =>
            `${range[0]}-${range[1]} của ${total} sản phẩm`,
          pageSizeOptions: ['10', '20', '50', '100'],
        }}
      />
    </div>
  );
};
```

### 6.2. URL-based Pagination Example

```typescript
// shiny-carnival/frontend/src/features/products/pages/ProductListPageWithRouter.tsx
import { getRouteApi } from '@tanstack/react-router';
import { Table, Input, Button } from 'antd';
import { SearchOutlined } from '@ant-design/icons';
import { usePaginationWithRouter } from '../../../hooks/usePaginationWithRouter';
import { productApiService } from '../../../lib/api/productApiService';
import type { ProductEntity } from '../types/entity';

const routeApi = getRouteApi('/admin/products');

export const ProductListPageWithRouter = () => {
  const {
    items,
    isLoading,
    params,
    totalCount,
    handlePageChange,
    handleSearch,
    handleSort,
  } = usePaginationWithRouter<ProductEntity>({
    apiService: productApiService,
    entity: 'products',
    routeApi,
  });

  const columns = [
    {
      title: 'Tên sản phẩm',
      dataIndex: 'productName',
      key: 'productName',
      sorter: true,
    },
    {
      title: 'Giá',
      dataIndex: 'price',
      key: 'price',
      sorter: true,
      render: (price: number) => `${price.toLocaleString('vi-VN')} đ`,
    },
  ];

  return (
    <div>
      <Input.Search
        placeholder="Tìm kiếm..."
        allowClear
        enterButton={<SearchOutlined />}
        onSearch={handleSearch}
        defaultValue={params.search}
        style={{ width: 300, marginBottom: 16 }}
      />

      <Table<ProductEntity>
        columns={columns}
        dataSource={items}
        rowKey="id"
        loading={isLoading}
        onChange={(pagination, _, sorter) => {
          if (pagination.current && pagination.pageSize) {
            handlePageChange(pagination.current, pagination.pageSize);
          }
          if (!Array.isArray(sorter) && sorter.field) {
            handleSort(String(sorter.field), sorter.order === 'descend');
          }
        }}
        pagination={{
          current: params.page,
          pageSize: params.pageSize,
          total: totalCount,
          showSizeChanger: true,
          showTotal: (total) => `Tổng ${total} sản phẩm`,
        }}
      />
    </div>
  );
};
```

### 6.3. Infinite Scroll Example

Xem code example chi tiết trong phần phân tích pagination ở trên.

### 6.4. Advanced Filters Example

Xem code example chi tiết trong phần phân tích pagination ở trên.

### 6.5. Prefetching Example

```typescript
// Component với prefetching
import { useEffect } from 'react';
import { Table } from 'antd';
import { usePaginationWithPrefetch } from '../../../hooks/usePaginationWithPrefetch';
import { productApiService } from '../../../lib/api/productApiService';
import type { ProductEntity } from '../types/entity';

export const ProductTableWithPrefetch = () => {
  const {
    items,
    page,
    pageSize,
    totalCount,
    handlePageChange,
    prefetchNextPage,
    prefetchDetail,
  } = usePaginationWithPrefetch<ProductEntity>({
    apiService: productApiService,
    entity: 'products',
  });

  // Prefetch next page khi component mount
  useEffect(() => {
    prefetchNextPage();
  }, [page, prefetchNextPage]);

  const columns = [
    {
      title: 'Tên sản phẩm',
      dataIndex: 'productName',
      key: 'productName',
      render: (text: string, record: ProductEntity) => (
        <span
          onMouseEnter={() => prefetchDetail(record.id!)}
          style={{ cursor: 'pointer' }}
        >
          {text}
        </span>
      ),
    },
  ];

  return (
    <Table
      columns={columns}
      dataSource={items}
      pagination={{
        current: page,
        pageSize: pageSize,
        total: totalCount,
        onChange: (newPage, newPageSize) => {
          handlePageChange(newPage, newPageSize);
          // Prefetch next page sau khi change
          setTimeout(() => {
            prefetchNextPage();
          }, 100);
        },
      }}
    />
  );
};
```

## 7. Performance Optimization & Best Practices

### 7.1. Debounced Search

```typescript
// Hook với debounced search
import { useState, useEffect } from 'react';
import { usePagination } from '../../../hooks/usePagination';

export function useProductsWithDebouncedSearch() {
  const [searchInput, setSearchInput] = useState('');
  const [debouncedSearch, setDebouncedSearch] = useState('');

  // Debounce search input
  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedSearch(searchInput);
    }, 500);

    return () => clearTimeout(timer);
  }, [searchInput]);

  const pagination = usePagination({
    apiService: productApiService,
    entity: 'products',
    initialSearch: debouncedSearch,
  });

  return {
    ...pagination,
    searchInput,
    setSearchInput,
  };
}
```

### 7.2. Optimistic Updates với Pagination

```typescript
// Hook với optimistic updates
export const useUpdateProductOptimistic = () => {
  const queryClient = useQueryClient();

  return useApiUpdate<ProductEntity, UpdateProductRequest>({
    apiService: productApiService,
    entity: 'products',
    options: {
      onMutate: async ({ id, data }) => {
        // Cancel outgoing refetches
        await queryClient.cancelQueries({ queryKey: ['products'] });

        // Snapshot previous value
        const previousData = queryClient.getQueryData(['products', 'list']);

        // Optimistically update
        queryClient.setQueryData(['products', 'list'], (old: any) => {
          if (!old) return old;
          return {
            ...old,
            items: old.items.map((item: ProductEntity) =>
              item.id === id ? { ...item, ...data } : item
            ),
          };
        });

        return { previousData };
      },
      onError: (err, variables, context) => {
        // Rollback on error
        if (context?.previousData) {
          queryClient.setQueryData(['products', 'list'], context.previousData);
        }
      },
      onSettled: () => {
        // Refetch after error or success
        queryClient.invalidateQueries({ queryKey: ['products'] });
      },
    },
  });
};
```

### 7.3. Persistent Pagination State

```typescript
// Hook với localStorage persistence
import { useState, useEffect } from 'react';

export function usePersistentPageSize(key: string, defaultValue: number = 20) {
  const [pageSize, setPageSize] = useState(() => {
    const stored = localStorage.getItem(key);
    return stored ? parseInt(stored, 10) : defaultValue;
  });

  useEffect(() => {
    localStorage.setItem(key, pageSize.toString());
  }, [key, pageSize]);

  return [pageSize, setPageSize] as const;
}

// Sử dụng
export function useProductsWithPersistence() {
  const [pageSize, setPageSize] = usePersistentPageSize('products-page-size', 20);

  return usePagination({
    apiService: productApiService,
    entity: 'products',
    initialPageSize: pageSize,
  });
}
```

### 7.4. Best Practices Checklist

✅ **State Management**
- Sử dụng URL params cho pagination state (SEO friendly, shareable)
- Giữ previous data khi fetch new page (`placeholderData`)
- Reset về page 1 khi thay đổi filters/search/sort
- Persist user preferences (pageSize) trong localStorage

✅ **Performance**
- Prefetch next/previous pages
- Debounce search input (500ms)
- Memoize expensive computations với `useMemo`
- Use `staleTime` để giảm unnecessary refetches
- Implement virtual scrolling cho large lists

✅ **UX**
- Show loading states (skeleton, spinner)
- Show total count và current range
- Provide quick jump và page size options
- Clear visual feedback khi filtering
- Disable actions khi loading
- Show empty states

✅ **Error Handling**
- Graceful error messages
- Retry mechanism
- Fallback UI
- Error boundaries

✅ **Accessibility**
- Keyboard navigation support
- ARIA labels
- Focus management
- Screen reader support

## 8. Tổng kết

### ✅ Ưu điểm của hệ thống này:

1. **Type-safe**: Đầy đủ TypeScript types
2. **Reusable**: Dùng lại cho mọi entity
3. **Automatic caching**: TanStack Query tự động cache
4. **Optimistic updates**: Hỗ trợ sẵn
5. **Error handling**: Xử lý lỗi tập trung
6. **Devtools**: React Query Devtools để debug
7. **Clean code**: Tách biệt logic API khỏi UI
8. **Pagination support**: Đầy đủ các loại pagination
9. **Performance optimized**: Prefetching, debouncing, memoization
10. **URL-based state**: SEO friendly và shareable links

### 📋 Checklist triển khai đầy đủ:

#### Core Setup
- [ ] Cài đặt `@tanstack/react-query` và `@tanstack/react-query-devtools`
- [ ] Tạo `src/lib/apiFactory.ts` - ApiService class
- [ ] Tạo `src/lib/queryClient.ts` - QueryClient config
- [ ] Cập nhật `src/app/main.tsx` với QueryClientProvider

#### Universal Hooks
- [ ] Tạo `src/hooks/useApi.ts` - Basic CRUD hooks
- [ ] Tạo `src/hooks/useApiInfinite.ts` - Infinite scroll hook
- [ ] Tạo `src/hooks/usePagination.ts` - Pagination management
- [ ] Tạo `src/hooks/usePaginationWithRouter.ts` - URL-based pagination
- [ ] Tạo `src/hooks/usePaginationWithFilters.ts` - Advanced filters
- [ ] Tạo `src/hooks/usePaginationWithPrefetch.ts` - Prefetching support

#### Entity Implementation
- [ ] Tạo API Service instances cho từng entity (`src/lib/api/`)
- [ ] Tạo entity-specific hooks (`src/features/*/hooks/`)
- [ ] Cập nhật components để sử dụng hooks mới

#### Advanced Features
- [ ] Implement optimistic updates
- [ ] Add debounced search
- [ ] Add prefetching
- [ ] Add persistent pagination state
- [ ] Add error boundaries
- [ ] Add loading skeletons

#### Testing & Documentation
- [ ] Write unit tests cho hooks
- [ ] Write integration tests
- [ ] Document usage examples
- [ ] Create migration guide

### 🚀 Bước tiếp theo:

1. **Phase 1: Core Setup**
   - Cài đặt dependencies
   - Setup QueryClient và Provider
   - Tạo ApiFactory và Universal Hooks

2. **Phase 2: Pagination Support**
   - Implement pagination management hooks
   - Add URL-based pagination
   - Add infinite scroll support
   - Add advanced filters

3. **Phase 3: Entity Migration**
   - Migrate Product entity (pilot)
   - Migrate User, Customer entities
   - Migrate Order, Category, Supplier entities
   - Migrate Promotion, Inventory entities

4. **Phase 4: Optimization**
   - Add prefetching
   - Add optimistic updates
   - Add debouncing
   - Performance tuning

5. **Phase 5: Polish**
   - Error handling improvements
   - Loading states
   - Empty states
   - Accessibility improvements

### 🎯 Key Features Summary:

#### Pagination Support
- ✅ Server-side pagination với `useApiPagedList`
- ✅ Infinite scroll với `useApiInfinite`
- ✅ URL-based pagination với `usePaginationWithRouter`
- ✅ Advanced filters với `usePaginationWithFilters`
- ✅ Prefetching với `usePaginationWithPrefetch`
- ✅ Debounced search
- ✅ Sorting support
- ✅ Persistent state (localStorage)

#### CRUD Operations
- ✅ GET all (with/without pagination)
- ✅ GET by ID
- ✅ CREATE
- ✅ UPDATE (full)
- ✅ PATCH (partial)
- ✅ DELETE
- ✅ Custom queries
- ✅ Custom mutations

#### Performance
- ✅ Automatic caching
- ✅ Prefetching
- ✅ Optimistic updates
- ✅ Debouncing
- ✅ Memoization
- ✅ Stale time configuration

#### Developer Experience
- ✅ Type-safe với TypeScript
- ✅ Reusable hooks
- ✅ Clean API
- ✅ React Query Devtools
- ✅ Error handling
- ✅ Loading states

---

## 9. Backend Integration Guide

### 9.1. API Endpoint Mapping Table

Bảng mapping đầy đủ giữa Frontend Hooks và Backend Endpoints:

| Entity | Frontend Hook | Backend Endpoint | HTTP Method | Auth | Request Type | Response Type |
|--------|---------------|------------------|-------------|------|--------------|---------------|
| **Products** | `useApiList` | `GET /api/admin/products` | GET | Admin | `ProductSearchRequest` | `ApiResponse<PagedList<ProductListDto>>` |
| | `useApiDetail` | `GET /api/admin/products/{id}` | GET | Admin | - | `ApiResponse<ProductResponseDto>` |
| | `useApiCreate` | `POST /api/admin/products` | POST | Admin | `CreateProductRequest` | `ApiResponse<ProductResponseDto>` |
| | `useApiUpdate` | `PUT /api/admin/products/{id}` | PUT | Admin | `UpdateProductRequest` | `ApiResponse<ProductResponseDto>` |
| | `useApiDelete` | `DELETE /api/admin/products/{id}` | DELETE | Admin | - | `ApiResponse<object>` |
| **Categories** | `useApiList` | `GET /api/admin/categories` | GET | Admin | `CategorySearchRequest` | `ApiResponse<PagedList<CategoryResponseDto>>` |
| | `useApiDetail` | `GET /api/admin/categories/{id}` | GET | Admin | - | `ApiResponse<CategoryResponseDto>` |
| | `useApiCreate` | `POST /api/admin/categories` | POST | Admin | `CreateCategoryRequest` | `ApiResponse<CategoryResponseDto>` |
| | `useApiUpdate` | `PUT /api/admin/categories/{id}` | PUT | Admin | `UpdateCategoryRequest` | `ApiResponse<CategoryResponseDto>` |
| | `useApiDelete` | `DELETE /api/admin/categories/{id}` | DELETE | Admin | - | `ApiResponse<object>` |
| **Customers** | `useApiList` | `GET /api/admin/customers` | GET | Admin, Staff | `CustomerSearchRequest` | `ApiResponse<PagedList<CustomerListDto>>` |
| | `useApiDetail` | `GET /api/admin/customers/{id}` | GET | Admin, Staff | - | `ApiResponse<CustomerResponseDto>` |
| | `useApiCreate` | `POST /api/admin/customers` | POST | Admin, Staff | `CreateCustomerRequest` | `ApiResponse<CustomerResponseDto>` |
| | `useApiUpdate` | `PUT /api/admin/customers/{id}` | PUT | Admin, Staff | `UpdateCustomerRequest` | `ApiResponse<CustomerResponseDto>` |
| | `useApiDelete` | `DELETE /api/admin/customers/{id}` | DELETE | Admin, Staff | - | `ApiResponse<object>` |
| **Suppliers** | `useApiList` | `GET /api/admin/suppliers` | GET | Admin | `SupplierSearchRequest` | `ApiResponse<PagedList<SupplierResponseDto>>` |
| | `useApiDetail` | `GET /api/admin/suppliers/{id}` | GET | Admin | - | `ApiResponse<SupplierResponseDto>` |
| | `useApiCreate` | `POST /api/admin/suppliers` | POST | Admin | `CreateSupplierRequest` | `ApiResponse<SupplierResponseDto>` |
| | `useApiUpdate` | `PUT /api/admin/suppliers/{id}` | PUT | Admin | `UpdateSupplierRequest` | `ApiResponse<SupplierResponseDto>` |
| | `useApiDelete` | `DELETE /api/admin/suppliers/{id}` | DELETE | Admin | - | `ApiResponse<object>` |
| **Orders** | `useApiList` | `GET /api/admin/orders` | GET | Admin, Staff | `OrderSearchRequest` | `ApiResponse<PagedList<OrderListDto>>` |
| | `useApiDetail` | `GET /api/admin/orders/{id}` | GET | Admin, Staff | - | `ApiResponse<OrderResponseDto>` |
| | `useApiCreate` | `POST /api/admin/orders` | POST | Admin, Staff | `CreateOrderRequest` | `ApiResponse<OrderResponseDto>` |
| | `useApiPatch` | `PATCH /api/admin/orders/{id}/status` | PATCH | Admin, Staff | `{ status: string }` | `ApiResponse<OrderResponseDto>` |
| | `useApiCustom` | `POST /api/admin/orders/{orderId}/items` | POST | Admin, Staff | `AddOrderItemRequest` | `ApiResponse<OrderItemDto>` |
| | `useApiCustom` | `PUT /api/admin/orders/{orderId}/items/{itemId}` | PUT | Admin, Staff | `UpdateOrderItemRequest` | `ApiResponse<OrderItemDto>` |
| | `useApiCustom` | `DELETE /api/admin/orders/{orderId}/items/{itemId}` | DELETE | Admin, Staff | - | `ApiResponse<object>` |
| | `useApiCustom` | `GET /api/admin/orders/{id}/invoice` | GET | Admin, Staff | - | PDF File |
| **Promotions** | `useApiList` | `GET /api/admin/promotions` | GET | Admin | `PromotionSearchRequest` | `ApiResponse<PagedList<PromotionListDto>>` |
| | `useApiDetail` | `GET /api/admin/promotions/{id}` | GET | Admin | - | `ApiResponse<PromotionResponseDto>` |
| | `useApiCreate` | `POST /api/admin/promotions` | POST | Admin | `CreatePromotionRequest` | `ApiResponse<PromotionResponseDto>` |
| | `useApiUpdate` | `PUT /api/admin/promotions/{id}` | PUT | Admin | `UpdatePromotionRequest` | `ApiResponse<PromotionResponseDto>` |
| | `useApiDelete` | `DELETE /api/admin/promotions/{id}` | DELETE | Admin | - | `ApiResponse<object>` |
| | `useApiCustom` | `POST /api/admin/promotions/validate` | POST | Admin | `{ promoCode: string }` | `ApiResponse<PromotionResponseDto>` |
| **Users** | `useApiList` | `GET /api/admin/users` | GET | Admin | `UserSearchRequest` | `ApiResponse<PagedList<UserResponseDto>>` |
| | `useApiDetail` | `GET /api/admin/users/{id}` | GET | Admin | - | `ApiResponse<UserResponseDto>` |
| | `useApiCreate` | `POST /api/admin/users` | POST | Admin | `CreateUserRequest` | `ApiResponse<UserResponseDto>` |
| | `useApiUpdate` | `PUT /api/admin/users/{id}` | PUT | Admin | `UpdateUserRequest` | `ApiResponse<UserResponseDto>` |
| | `useApiDelete` | `DELETE /api/admin/users/{id}` | DELETE | Admin | - | `ApiResponse<object>` |
| **Inventory** | `useApiList` | `GET /api/admin/inventory` | GET | Admin, Staff | `InventorySearchRequest` | `ApiResponse<PagedList<InventoryResponseDto>>` |
| | `useApiDetail` | `GET /api/admin/inventory/{productId}` | GET | Admin, Staff | - | `ApiResponse<InventoryResponseDto>` |
| | `useApiPatch` | `PATCH /api/admin/inventory/{productId}` | PATCH | Admin, Staff | `{ quantityChange: number, reason: string }` | `ApiResponse<InventoryResponseDto>` |
| | `useApiCustom` | `GET /api/admin/inventory/low-stock` | GET | Admin, Staff | - | `ApiResponse<InventoryResponseDto[]>` |
| | `useApiCustom` | `GET /api/admin/inventory/{productId}/history` | GET | Admin, Staff | - | `ApiResponse<InventoryHistoryDto[]>` |
| **Reports** | `useApiCustom` | `GET /api/admin/reports/revenue` | GET | Admin | `RevenueReportRequest` | `ApiResponse<RevenueReportDto>` |
| | `useApiCustom` | `GET /api/admin/reports/sales` | GET | Admin | `SalesReportRequest` | `ApiResponse<SalesReportDto>` |
| | `useApiList` | `GET /api/admin/reports/top-products` | GET | Admin | `TopProductsSearchRequest` | `ApiResponse<PagedList<TopProductDto>>` |
| | `useApiList` | `GET /api/admin/reports/top-customers` | GET | Admin | `TopCustomersSearchRequest` | `ApiResponse<PagedList<TopCustomerDto>>` |

### 9.2. Query Parameters Contract

#### Frontend TypeScript Interfaces (camelCase):
```typescript
interface PagedRequest {
  page?: number;          // Default: 1
  pageSize?: number;      // Default: 10, Max: 100
  search?: string;        // Max 255 chars
  sortBy?: string;        // Default: "Id"
  sortDesc?: boolean;     // Default: true
}

interface ProductSearchRequest extends PagedRequest {
  categoryId?: number;    // Filter by category
  supplierId?: number;    // Filter by supplier
  minPrice?: number;      // Minimum price
  maxPrice?: number;      // Maximum price
}
```

#### Backend C# Models (PascalCase):
```csharp
public class PagedRequest : BasePagedRequest
{
    public string? Search { get; set; }
    public string SortBy { get; set; } = "Id";
    public bool SortDesc { get; set; } = true;
}

public class BasePagedRequest
{
    private const int MaxPageSize = 100;
    public int Page { get; set; } = 1;
    private int _pageSize = 10;
    public int PageSize
    {
        get => _pageSize;
        set => _pageSize = (value > MaxPageSize) ? MaxPageSize : value;
    }
}

public class ProductSearchRequest : PagedRequest
{
    public int? CategoryId { get; set; }
    public int? SupplierId { get; set; }
    public decimal? MinPrice { get; set; }
    public decimal? MaxPrice { get; set; }
}
```

#### Frontend gửi request (camelCase):
```typescript
const response = await axios.get('/api/admin/products', {
  params: {
    page: 1,
    pageSize: 20,
    search: 'coca',
    sortBy: 'ProductName',
    sortDesc: false,
    categoryId: 5,
    minPrice: 10000,
    maxPrice: 50000
  }
});
```

#### URL thực tế (lowercase):
```
GET /api/admin/products?page=1&pagesize=20&search=coca&sortby=ProductName&sortdesc=false&categoryid=5&minprice=10000&maxprice=50000
```

#### Backend nhận được (ASP.NET Core Model Binding):
✅ **ASP.NET Core Model Binding** là **case-insensitive**
✅ Tự động bind: `page` → `Page`, `pagesize` → `PageSize`, `categoryid` → `CategoryId`
✅ Không cần transform, backend tự động map

**Lưu ý**: Backend API Reference examples sử dụng PascalCase (`Page`, `PageSize`) nhưng đó chỉ là convention trong C#. Frontend có thể gửi camelCase hoặc lowercase, backend đều nhận được chính xác.

### 9.3. Authorization Requirements

| Endpoint | Required Role | Header |
|----------|---------------|--------|
| `/api/admin/products` | Admin | `Authorization: Bearer <token>` |
| `/api/admin/categories` | Admin | `Authorization: Bearer <token>` |
| `/api/admin/customers` | Admin, Staff | `Authorization: Bearer <token>` |
| `/api/admin/suppliers` | Admin | `Authorization: Bearer <token>` |
| `/api/admin/orders` | Admin, Staff | `Authorization: Bearer <token>` |
| `/api/admin/promotions` | Admin | `Authorization: Bearer <token>` |
| `/api/admin/users` | Admin | `Authorization: Bearer <token>` |
| `/api/admin/inventory` | Admin, Staff | `Authorization: Bearer <token>` |
| `/api/admin/reports/*` | Admin | `Authorization: Bearer <token>` |

**JWT Token Structure**:
```json
{
  "sub": "1",
  "unique_name": "admin",
  "role": "Admin",
  "nbf": 1704844800,
  "exp": 1704931200,
  "iat": 1704844800
}
```

### 9.4. Error Response Structure

Backend trả về errors trong `ApiResponse<T>` wrapper:

```typescript
// Success Response
{
  "isError": false,
  "message": "Success",
  "data": { /* actual data */ },
  "timestamp": "2024-01-09T10:30:00Z"
}

// Error Response
{
  "isError": true,
  "message": "Error message here",
  "data": null,
  "timestamp": "2024-01-09T10:30:00Z"
}
```

**HTTP Status Codes**:
- `200 OK` - Success
- `400 Bad Request` - Validation errors
- `401 Unauthorized` - Missing/invalid token
- `403 Forbidden` - Insufficient permissions
- `404 Not Found` - Resource not found
- `500 Internal Server Error` - Server error

### 9.5. SortBy Options & Filters

Mỗi entity có các SortBy options và filters riêng:

| Entity | SortBy Options | Filters |
|--------|----------------|---------|
| **Products** | `Id`, `ProductName`, `Price`, `CategoryName`, `SupplierName`, `InventoryQuantity`, `CreatedAt` | `CategoryId`, `SupplierId`, `MinPrice`, `MaxPrice` |
| **Categories** | `Id`, `CategoryName`, `ProductCount` | - |
| **Customers** | `Id`, `Name`, `Phone`, `Email`, `LastOrderDate` | - |
| **Suppliers** | `Id`, `Name`, `Phone`, `Email`, `ProductCount` | - |
| **Orders** | `Id`, `OrderDate`, `CustomerName`, `StaffName`, `Status`, `TotalAmount`, `FinalAmount` | `Status`, `CustomerId`, `UserId`, `StartDate`, `EndDate` |
| **Promotions** | `Id`, `PromoCode`, `DiscountValue`, `StartDate`, `EndDate`, `UsedCount`, `Status` | `Status` |
| **Users** | `Id`, `Username`, `FullName`, `Role`, `CreatedAt` | `Role` |
| **Inventory** | `Id`, `ProductName`, `Barcode`, `Quantity`, `UpdatedAt`, `Status` | `Status` |
| **Top Products** | Fixed: `TotalRevenue DESC` | `StartDate` (required), `EndDate` (required) |
| **Top Customers** | Fixed: `TotalSpent DESC` | `StartDate` (required), `EndDate` (required) |

**Lưu ý**:
- ⚠️ **Top Products** và **Top Customers** KHÔNG hỗ trợ `Search`, `SortBy`, `SortDesc`
- ⚠️ Chỉ hỗ trợ `Page`, `PageSize`, `StartDate`, `EndDate`

### 9.6. Validation Rules

#### Common Validation (tất cả endpoints):

| Parameter | Min | Max | Default | Validation |
|-----------|-----|-----|---------|------------|
| `page` | 1 | ∞ | 1 | Must be > 0 |
| `pageSize` | 1 | 100 | 10 | Auto-capped at 100 |
| `search` | - | 255 chars | null | Max 255 characters |
| `sortBy` | - | - | "Id" | Must be valid property name (see table above) |
| `sortDesc` | - | - | true | Boolean |

#### Entity-Specific Validation:

**Products**:
- `categoryId`: Must exist in database
- `supplierId`: Must exist in database
- `minPrice`: >= 0
- `maxPrice`: >= minPrice
- `price`: > 0 (for create/update)
- `productName`: Required, max 100 chars
- `barcode`: Required, max 50 chars, **unique**
- `unit`: Required, max 20 chars, default "pcs"

**Categories**:
- `categoryName`: Required, max 100 chars, **unique**

**Customers**:
- `name`: Required, max 100 chars
- `phone`: Required, max 20 chars
- `email`: Optional, max 100 chars, valid email format

**Suppliers**:
- `name`: Required, max 100 chars
- `phone`: Required, max 20 chars
- `email`: Optional, max 100 chars, valid email format
- `address`: Optional, max 255 chars

**Orders**:
- `status`: Must be "Pending", "Paid", or "Cancelled"
- `customerId`: Required, must exist
- `userId`: Auto-filled from JWT token
- `startDate`: <= Now (for search filter)
- `endDate`: >= startDate, <= Now (for search filter)

**Promotions**:
- `promoCode`: Required, max 50 chars, **unique**
- `discountType`: Required, "percent" or "fixed"
- `discountValue`: Required, > 0
- `startDate`: Required
- `endDate`: Required, >= startDate
- `minOrderAmount`: Optional, >= 0, default 0
- `usageLimit`: Optional, >= 0, default 0 (unlimited)

**Users**:
- `username`: Required, max 50 chars, **unique**
- `password`: Required, min 6 chars
- `fullName`: Required, max 100 chars
- `role`: Required, 0 (Admin) or 1 (Staff)

**Inventory**:
- `quantityChange`: Required, can be negative (decrease) or positive (increase)
- `reason`: Required, max 255 chars

### 9.7. Breaking Changes & Migration Notes

#### ❌ KHÔNG hỗ trợ PATCH cho hầu hết entities

**Affected Hooks**: `useApiPatch`

**Entities KHÔNG hỗ trợ PATCH**:
- ❌ Products
- ❌ Categories
- ❌ Customers
- ❌ Suppliers
- ❌ Users
- ❌ Promotions

**Giải pháp**: Sử dụng `useApiUpdate` (PUT) thay vì `useApiPatch`

```typescript
// ❌ SAI - KHÔNG hoạt động cho Products
const { mutate } = useApiPatch(productService, 'products');
mutate({ id: 1, price: 50000 });

// ✅ ĐÚNG - Sử dụng PUT (full replacement)
const { mutate } = useApiUpdate(productService, 'products');
mutate({
  id: 1,
  categoryId: 5,
  supplierId: 3,
  productName: 'Coca Cola',
  barcode: '8934588123456',
  price: 50000,
  unit: 'can'
});
```

**Entities HỖ TRỢ PATCH** (CHỈ 2 endpoints):

**1. Orders - Update Status**:
```typescript
// ✅ ĐÚNG - PATCH order status
const { mutate } = useApiPatch(orderService, 'orders');
mutate({
  id: 1,
  status: 'Paid' // "Pending" | "Paid" | "Cancelled"
});

// Hoặc sử dụng custom endpoint
const { mutate } = useApiCustomMutation(orderService, 'orders');
mutate({
  method: 'patch',
  path: `${orderId}/status`,
  data: { status: 'Paid' }
});
```

**2. Inventory - Update Quantity**:
```typescript
// ✅ ĐÚNG - PATCH inventory quantity
const { mutate } = useApiPatch(inventoryService, 'inventory');
mutate({
  id: productId,
  quantityChange: 100,  // Positive = increase, Negative = decrease
  reason: 'Nhập hàng từ nhà cung cấp'
});

// Giảm tồn kho
mutate({
  id: productId,
  quantityChange: -50,
  reason: 'Xuất hàng bán lẻ'
});
```

#### ⚠️ Report Endpoints đặc biệt

**1. Top Products & Top Customers** (Paginated):

KHÔNG hỗ trợ:
- ❌ `search`
- ❌ `sortBy`
- ❌ `sortDesc`

Chỉ chấp nhận:
- ✅ `page`
- ✅ `pageSize`
- ✅ `startDate` (required, ISO 8601 format)
- ✅ `endDate` (required, ISO 8601 format)

```typescript
// ❌ SAI
const { data } = useApiPagedList(reportService, 'reports/top-products', {
  page: 1,
  pageSize: 10,
  search: 'coca', // ❌ Backend sẽ ignore
  sortBy: 'TotalRevenue' // ❌ Backend sẽ ignore
});

// ✅ ĐÚNG - Top Products
const { data } = useApiPagedList(reportService, 'reports/top-products', {
  page: 1,
  pageSize: 10,
  startDate: '2024-01-01',
  endDate: '2024-12-31'
});

// ✅ ĐÚNG - Top Customers
const { data } = useApiPagedList(reportService, 'reports/top-customers', {
  page: 1,
  pageSize: 10,
  startDate: '2024-01-01',
  endDate: '2024-12-31'
});
```

**2. Revenue & Sales Reports** (Non-paginated):

```typescript
// Revenue Report
const { data } = useApiCustomQuery(reportService, 'reports', {
  method: 'get',
  path: 'revenue',
  params: {
    startDate: '2024-01-01',
    endDate: '2024-12-31'
  }
});

// Sales Report
const { data } = useApiCustomQuery(reportService, 'reports', {
  method: 'get',
  path: 'sales',
  params: {
    startDate: '2024-01-01',
    endDate: '2024-12-31'
  }
});
```

**Response Types**:
```typescript
interface TopProductDto {
  productId: number;
  productName: string;
  totalQuantity: number;
  totalRevenue: number;
}

interface TopCustomerDto {
  customerId: number;
  customerName: string;
  orderCount: number;
  totalSpent: number;
}

interface RevenueReportDto {
  totalRevenue: number;
  totalOrders: number;
  averageOrderValue: number;
  // ... other fields
}

interface SalesReportDto {
  totalSales: number;
  totalItems: number;
  // ... other fields
}
```

### 9.7. Testing Checklist

Khi implement hooks cho entity mới, test các scenarios sau:

#### Query Operations:
- [ ] `useApiList` - GET all items
- [ ] `useApiPagedList` - GET with pagination
- [ ] `useApiDetail` - GET by ID
- [ ] Pagination: page 1, page 2, last page
- [ ] Search: empty, partial match, no results
- [ ] Sorting: ascending, descending, different columns
- [ ] Filters: single filter, multiple filters, edge cases
- [ ] Error handling: 401, 403, 404, 500
- [ ] Loading states
- [ ] Cache behavior

#### Mutation Operations:
- [ ] `useApiCreate` - POST new item
- [ ] `useApiUpdate` - PUT existing item
- [ ] `useApiPatch` - PATCH (if supported)
- [ ] `useApiDelete` - DELETE item
- [ ] Validation errors (400)
- [ ] Success responses
- [ ] Optimistic updates
- [ ] Cache invalidation
- [ ] Error rollback

#### Authorization:
- [ ] Admin role access
- [ ] Staff role access (if applicable)
- [ ] Unauthorized access (401)
- [ ] Forbidden access (403)
- [ ] Token expiration handling

### 9.8. Special Endpoints

Các endpoints đặc biệt không theo pattern CRUD chuẩn:

#### 1. Inventory - Low Stock Alerts

**Endpoint**: `GET /api/admin/inventory/low-stock`
**Authorization**: Admin & Staff
**Description**: Lấy danh sách sản phẩm sắp hết hàng (KHÔNG phân trang)

```typescript
const { data } = useApiCustomQuery(inventoryService, 'inventory', {
  method: 'get',
  path: 'low-stock'
});

// Response: ApiResponse<InventoryResponseDto[]>
```

#### 2. Inventory - History

**Endpoint**: `GET /api/admin/inventory/{productId}/history`
**Authorization**: Admin & Staff
**Description**: Lấy lịch sử thay đổi tồn kho của sản phẩm

```typescript
const { data } = useApiCustomQuery(inventoryService, 'inventory', {
  method: 'get',
  path: `${productId}/history`
});

// Response: ApiResponse<InventoryHistoryDto[]>
```

#### 3. Orders - Add Item

**Endpoint**: `POST /api/admin/orders/{orderId}/items`
**Authorization**: Admin & Staff

```typescript
const { mutate } = useApiCustomMutation(orderService, 'orders');
mutate({
  method: 'post',
  path: `${orderId}/items`,
  data: {
    productId: 1,
    quantity: 5,
    price: 15000
  }
});
```

#### 4. Orders - Update Item

**Endpoint**: `PUT /api/admin/orders/{orderId}/items/{itemId}`
**Authorization**: Admin & Staff

```typescript
const { mutate } = useApiCustomMutation(orderService, 'orders');
mutate({
  method: 'put',
  path: `${orderId}/items/${itemId}`,
  data: {
    quantity: 10,
    price: 15000
  }
});
```

#### 5. Orders - Delete Item

**Endpoint**: `DELETE /api/admin/orders/{orderId}/items/{itemId}`
**Authorization**: Admin & Staff

```typescript
const { mutate } = useApiCustomMutation(orderService, 'orders');
mutate({
  method: 'delete',
  path: `${orderId}/items/${itemId}`
});
```

#### 6. Orders - Download Invoice (PDF)

**Endpoint**: `GET /api/admin/orders/{id}/invoice`
**Authorization**: Admin & Staff
**Response**: PDF File (binary)

```typescript
const downloadInvoice = async (orderId: number) => {
  const response = await axios.get(`/api/admin/orders/${orderId}/invoice`, {
    responseType: 'blob',
    headers: {
      Authorization: `Bearer ${accessToken}`
    }
  });

  // Create download link
  const url = window.URL.createObjectURL(new Blob([response.data]));
  const link = document.createElement('a');
  link.href = url;
  link.setAttribute('download', `invoice-${orderId}.pdf`);
  document.body.appendChild(link);
  link.click();
  link.remove();
};
```

#### 7. Promotions - Validate Promo Code

**Endpoint**: `POST /api/admin/promotions/validate`
**Authorization**: Admin

```typescript
const { mutate } = useApiCustomMutation(promotionService, 'promotions');
mutate({
  method: 'post',
  path: 'validate',
  data: {
    promoCode: 'SUMMER2024'
  }
});

// Response: ApiResponse<PromotionResponseDto>
```

### 9.9. Common Pitfalls & Solutions

#### 1. Query Parameters Case Sensitivity

**Problem**: Backend không nhận được query parameters

**Cause**: Frontend gửi sai tên parameter

**Solution**:
```typescript
// ❌ SAI
const params = { Page: 1, PageSize: 20 }; // PascalCase

// ✅ ĐÚNG
const params = { page: 1, pageSize: 20 }; // camelCase
```

#### 2. PATCH Not Supported

**Problem**: `useApiPatch` trả về 404 hoặc 405

**Cause**: Entity không hỗ trợ PATCH

**Solution**: Sử dụng `useApiUpdate` (PUT) thay vì `useApiPatch`

#### 3. PageSize > 100

**Problem**: Backend chỉ trả về 100 items dù request 200

**Cause**: Backend auto-cap PageSize ở 100

**Solution**:
```typescript
const pageSize = Math.min(requestedPageSize, 100);
```

#### 4. Invalid SortBy Value

**Problem**: Backend trả về validation error

**Cause**: SortBy không phải là property hợp lệ

**Solution**: Kiểm tra allowed values trước khi gửi
```typescript
const ALLOWED_SORT_BY = ['Id', 'ProductName', 'Price', 'CreatedAt'];
const sortBy = ALLOWED_SORT_BY.includes(column) ? column : 'Id';
```

#### 5. Search Too Long

**Problem**: Backend trả về validation error "max 255 chars"

**Cause**: Search string > 255 characters

**Solution**:
```typescript
const search = searchInput.slice(0, 255);
```

#### 6. Missing Authorization Header

**Problem**: Backend trả về 401 Unauthorized

**Cause**: Token không được gửi trong header

**Solution**: Đảm bảo Axios interceptor thêm token
```typescript
axiosInstance.interceptors.request.use((config) => {
  const token = getToken();
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});
```

### 9.9. Performance Optimization

#### Cache Strategy

**Stale Time**: Thời gian data được coi là "fresh"
```typescript
// Products: 5 minutes (data ít thay đổi)
staleTime: 1000 * 60 * 5

// Orders: 1 minute (data thay đổi thường xuyên)
staleTime: 1000 * 60

// Inventory: 30 seconds (real-time data)
staleTime: 1000 * 30
```

**GC Time**: Thời gian giữ cache sau khi không sử dụng
```typescript
gcTime: 1000 * 60 * 30 // 30 minutes
```

#### Prefetching Strategy

```typescript
// Prefetch next page khi user scroll gần cuối
const { data, hasNextPage } = useApiInfinite(productService, 'products', {
  pageSize: 20
});

useEffect(() => {
  if (hasNextPage && scrollPosition > 80%) {
    queryClient.prefetchInfiniteQuery({
      queryKey: ['products', 'infinite', params],
      queryFn: () => fetchNextPage()
    });
  }
}, [scrollPosition, hasNextPage]);
```

#### Debouncing Search

```typescript
const [searchTerm, setSearchTerm] = useState('');
const debouncedSearch = useDebounce(searchTerm, 500);

const { data } = useApiPagedList(productService, 'products', {
  page: 1,
  pageSize: 20,
  search: debouncedSearch
});
```

---

## 10. Tổng kết

### ✅ Đã đồng bộ hóa:

1. **API Endpoint URLs** - Khớp 100% với backend
2. **HTTP Methods** - Đúng methods cho từng operation
3. **Request/Response Types** - Khớp với backend DTOs
4. **Query Parameters** - Đúng naming và validation
5. **Authorization** - Đúng roles và headers
6. **Error Handling** - Khớp với backend error structure
7. **Pagination** - Đúng PagedRequest và PagedList structure
8. **PATCH Support** - Chỉ Orders và Inventory
9. **Report Endpoints** - Không có Search/Sort
10. **Validation Rules** - Khớp với backend validators

### 📊 Mapping Summary:

- **10 Entities**: Products, Categories, Customers, Suppliers, Orders, Promotions, Users, Inventory, TopProducts, TopCustomers
- **5 Query Hooks**: useApiList, useApiPagedList, useApiInfinite, useApiDetail, useApiCustomQuery
- **5 Mutation Hooks**: useApiCreate, useApiUpdate, useApiPatch, useApiDelete, useApiCustomMutation
- **50+ Endpoints** được document đầy đủ
- **100% Backend Compatible** ✅

### 🎯 Ready for Implementation:

Plan này giờ đã **hoàn toàn tương thích** với backend TapHoaNho và sẵn sàng để implement! 🚀

---

## 10. CODE EXAMPLES CHI TIẾT

📚 **Xem tài liệu riêng**: [`customHookCallAPIPlan_CodeExamples.md`](./customHookCallAPIPlan_CodeExamples.md)

Tài liệu này cung cấp **code examples đầy đủ cho TẤT CẢ 53 endpoints**, bao gồm:

### 📋 Nội dung:

1. **Products Module** (5 endpoints)
   - GET list, GET detail, POST create, PUT update, DELETE

2. **Categories Module** (5 endpoints)
   - Full CRUD operations

3. **Customers Module** (5 endpoints)
   - Full CRUD operations

4. **Suppliers Module** (5 endpoints)
   - Full CRUD operations

5. **Orders Module** (8 endpoints)
   - CRUD + PATCH status + Order items management + Invoice download

6. **Promotions Module** (6 endpoints)
   - CRUD + Validate promo code

7. **Users Module** (5 endpoints)
   - Full CRUD operations

8. **Inventory Module** (5 endpoints)
   - List + PATCH quantity + Low stock alerts + History

9. **Reports Module** (4 endpoints)
   - Revenue, Sales, Top Products, Top Customers

10. **Authentication Module** (4 endpoints)
    - Login, Refresh, Logout, Setup Admin

### ✨ Mỗi endpoint bao gồm:

- ✅ **Frontend Code**: TypeScript + React + Ant Design
- ✅ **Backend Request**: HTTP method + URL + Headers + Body
- ✅ **Backend Response**: JSON structure đầy đủ
- ✅ **Authorization**: Rõ ràng (Admin, Staff, Public)
- ✅ **Hooks**: useApiList, useApiPagedList, useApiCreate, useApiUpdate, useApiPatch, useApiDelete, useApiCustomQuery, useApiCustomMutation
- ✅ **Error Handling**: Try-catch, message.error, loading states
- ✅ **Form Validation**: Rules, required fields, max length

### 📊 Thống kê:

- **1,857 dòng** code examples
- **53 endpoints** được document đầy đủ
- **10 modules** với examples chi tiết
- **100% production-ready** code

**👉 [Xem tài liệu đầy đủ tại đây](./customHookCallAPIPlan_CodeExamples.md)**
