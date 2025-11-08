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

export interface PagedList<T> {
  page: number;
  pageSize: number;
  totalCount: number;
  totalPages: number;
  items: T[];
}
````

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
   */
  async getAll(params?: QueryParams): Promise<TData[]> {
    const response = await this.axios.get<ApiResponse<TData[]>>(
      this.endpoint,
      { params }
    );
    return response.data || [];
  }

  /**
   * GET all with pagination - Lấy danh sách có phân trang
   */
  async getAllPaged(params?: PagedRequest): Promise<PagedList<TData>> {
    const response = await this.axios.get<ApiResponse<PagedList<TData>>>(
      this.endpoint,
      { params }
    );
    return response.data || {
      page: 1,
      pageSize: 20,
      totalCount: 0,
      totalPages: 0,
      hasPrevious: false,
      hasNext: false,
      items: []
    };
  }

  /**
   * GET by ID - Lấy chi tiết theo ID
   */
  async getById(id: string | number): Promise<TData> {
    const response = await this.axios.get<ApiResponse<TData>>(
      `${this.endpoint}/${id}`
    );
    if (!response.data) {
      throw new Error('Data not found');
    }
    return response.data;
  }

  /**
   * POST - Tạo mới
   */
  async create(data: TCreate): Promise<TData> {
    const response = await this.axios.post<ApiResponse<TData>>(
      this.endpoint,
      data
    );
    if (!response.data) {
      throw new Error('Create failed');
    }
    return response.data;
  }

  /**
   * PUT - Cập nhật toàn bộ
   */
  async update(id: string | number, data: TUpdate): Promise<TData> {
    const response = await this.axios.put<ApiResponse<TData>>(
      `${this.endpoint}/${id}`,
      data
    );
    if (!response.data) {
      throw new Error('Update failed');
    }
    return response.data;
  }

  /**
   * PATCH - Cập nhật một phần
   */
  async patch(id: string | number, data: Partial<TUpdate>): Promise<TData> {
    const response = await this.axios.patch<ApiResponse<TData>>(
      `${this.endpoint}/${id}`,
      data
    );
    if (!response.data) {
      throw new Error('Patch failed');
    }
    return response.data;
  }

  /**
   * DELETE - Xóa
   */
  async delete(id: string | number): Promise<void> {
    await this.axios.delete(`${this.endpoint}/${id}`);
  }

  /**
   * Custom method - Cho các endpoint đặc biệt
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
    if (!response.data) {
      throw new Error('Custom request failed');
    }
    return response.data;
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

## 6. Tổng kết

### ✅ Ưu điểm của hệ thống này:

1. **Type-safe**: Đầy đủ TypeScript types
2. **Reusable**: Dùng lại cho mọi entity
3. **Automatic caching**: TanStack Query tự động cache
4. **Optimistic updates**: Hỗ trợ sẵn
5. **Error handling**: Xử lý lỗi tập trung
6. **Devtools**: React Query Devtools để debug
7. **Clean code**: Tách biệt logic API khỏi UI

### 📋 Checklist triển khai:

- [ ] Cài đặt `@tanstack/react-query` và `@tanstack/react-query-devtools`
- [ ] Tạo `src/lib/apiFactory.ts`
- [ ] Tạo `src/lib/queryClient.ts`
- [ ] Tạo `src/hooks/useApi.ts`
- [ ] Cập nhật `src/app/main.tsx` với QueryClientProvider
- [ ] Tạo API Service instances cho từng entity
- [ ] Tạo hooks cho từng entity
- [ ] Cập nhật components để sử dụng hooks mới

### 🚀 Bước tiếp theo:

1. Triển khai cho tất cả entities (User, Customer, Order, Category, Supplier, Promotion)
2. Thêm optimistic updates cho UX tốt hơn
3. Implement infinite scroll với `useInfiniteQuery`
4. Thêm prefetching cho performance
5. Xây dựng error boundaries
