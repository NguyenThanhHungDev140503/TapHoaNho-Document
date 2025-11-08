# Universal API Hook với TanStack Query + Axios

Tôi sẽ tạo một hệ thống hooks tổng quát có thể tái sử dụng cho mọi entity (Product, User, v.v.).

## 1. Core API Factory (`src/api/apiFactory.ts`)

```typescript
import { AxiosInstance } from 'axios';

export interface ApiConfig<TData = any, TCreate = any, TUpdate = any> {
  endpoint: string;
  axiosInstance: AxiosInstance;
}

export interface QueryParams {
  [key: string]: any;
}

export class ApiService<TData = any, TCreate = any, TUpdate = any> {
  private endpoint: string;
  private axios: AxiosInstance;

  constructor(config: ApiConfig<TData, TCreate, TUpdate>) {
    this.endpoint = config.endpoint;
    this.axios = config.axiosInstance;
  }

  // GET all with optional query params
  async getAll(params?: QueryParams): Promise<TData[]> {
    const response = await this.axios.get<TData[]>(this.endpoint, { params });
    return response.data;
  }

  // GET by ID
  async getById(id: string | number): Promise<TData> {
    const response = await this.axios.get<TData>(`${this.endpoint}/${id}`);
    return response.data;
  }

  // POST - Create
  async create(data: TCreate): Promise<TData> {
    const response = await this.axios.post<TData>(this.endpoint, data);
    return response.data;
  }

  // PUT - Update
  async update(id: string | number, data: TUpdate): Promise<TData> {
    const response = await this.axios.put<TData>(`${this.endpoint}/${id}`, data);
    return response.data;
  }

  // PATCH - Partial Update
  async patch(id: string | number, data: Partial<TUpdate>): Promise<TData> {
    const response = await this.axios.patch<TData>(`${this.endpoint}/${id}`, data);
    return response.data;
  }

  // DELETE
  async delete(id: string | number): Promise<void> {
    await this.axios.delete(`${this.endpoint}/${id}`);
  }

  // Custom method for any additional endpoint
  async custom<TResponse = any>(
    method: 'get' | 'post' | 'put' | 'patch' | 'delete',
    path: string,
    data?: any,
    params?: QueryParams
  ): Promise<TResponse> {
    const url = path.startsWith('/') ? path : `${this.endpoint}/${path}`;
    const response = await this.axios.request<TResponse>({
      method,
      url,
      data,
      params,
    });
    return response.data;
  }
}
```

## 2. Universal CRUD Hooks (`src/hooks/useApi.ts`)

```typescript
import {
  useQuery,
  useMutation,
  useQueryClient,
  UseQueryOptions,
  UseMutationOptions,
  QueryKey,
} from '@tanstack/react-query';
import { ApiService, QueryParams } from '../api/apiFactory';

// ==================== Query Key Factory ====================
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

// ==================== GET BY ID Hook ====================
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
    enabled: !!id,
    ...options,
  });
}

// ==================== CREATE Hook ====================
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
      // Invalidate lists
      queryClient.invalidateQueries({ queryKey: queryKeys.lists() });
      
      // Invalidate additional queries
      invalidateQueries.forEach((key) => {
        queryClient.invalidateQueries({ queryKey: [key] });
      });

      // Call user's onSuccess if provided
      options?.onSuccess?.(data, variables, context);
    },
    ...options,
  });
}

// ==================== UPDATE Hook ====================
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
      // Invalidate lists
      queryClient.invalidateQueries({ queryKey: queryKeys.lists() });
      
      // Invalidate specific detail
      queryClient.invalidateQueries({ queryKey: queryKeys.detail(variables.id) });
      
      // Invalidate additional queries
      invalidateQueries.forEach((key) => {
        queryClient.invalidateQueries({ queryKey: [key] });
      });

      options?.onSuccess?.(data, variables, context);
    },
    ...options,
  });
}

// ==================== PATCH Hook ====================
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
      // Invalidate lists
      queryClient.invalidateQueries({ queryKey: queryKeys.lists() });
      
      // Remove specific detail from cache
      queryClient.removeQueries({ queryKey: queryKeys.detail(variables) });
      
      // Invalidate additional queries
      invalidateQueries.forEach((key) => {
        queryClient.invalidateQueries({ queryKey: [key] });
      });

      options?.onSuccess?.(data, variables, context);
    },
    ...options,
  });
}

// ==================== Custom Query Hook ====================
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
      
      // Invalidate additional queries
      invalidateQueries.forEach((key) => {
        queryClient.invalidateQueries({ queryKey: [key] });
      });

      options?.onSuccess?.(data, variables, context);
    },
    ...options,
  });
}
```

## 3. Example Usage - Product

### 3.1. Product Types (`src/types/product.ts`)

```typescript
export interface Product {
  id: number;
  name: string;
  description: string;
  price: number;
  stock: number;
  createdAt: string;
  updatedAt?: string;
}

export interface CreateProductDto {
  name: string;
  description: string;
  price: number;
  stock: number;
}

export interface UpdateProductDto {
  name: string;
  description: string;
  price: number;
  stock: number;
}
```

### 3.2. Product API Service (`src/api/productApi.ts`)

```typescript
import { apiClient } from './axios';
import { ApiService } from './apiFactory';
import { Product, CreateProductDto, UpdateProductDto } from '../types/product';

export const productApiService = new ApiService<Product, CreateProductDto, UpdateProductDto>({
  endpoint: '/products',
  axiosInstance: apiClient,
});

// Nếu cần custom methods
export const productApi = {
  ...productApiService,
  
  // Custom method example
  getProductsByCategory: (categoryId: number) => {
    return productApiService.custom<Product[]>('get', `category/${categoryId}`);
  },
  
  bulkUpdate: (products: Array<{ id: number; stock: number }>) => {
    return productApiService.custom<Product[]>('post', 'bulk-update', products);
  },
};
```

### 3.3. Product Hooks (`src/hooks/useProducts.ts`)

```typescript
import { 
  useApiList, 
  useApiDetail, 
  useApiCreate, 
  useApiUpdate, 
  useApiDelete,
  useApiCustomQuery,
} from './useApi';
import { productApiService } from '../api/productApi';
import { Product, CreateProductDto, UpdateProductDto } from '../types/product';

const ENTITY = 'products';

// GET all products
export const useProducts = (params?: { search?: string; category?: string }) => {
  return useApiList<Product>({
    apiService: productApiService,
    entity: ENTITY,
    params,
    options: {
      staleTime: 1000 * 60 * 5, // 5 minutes
    },
  });
};

// GET product by ID
export const useProduct = (id: number) => {
  return useApiDetail<Product>({
    apiService: productApiService,
    entity: ENTITY,
    id,
  });
};

// CREATE product
export const useCreateProduct = () => {
  return useApiCreate<Product, CreateProductDto>({
    apiService: productApiService,
    entity: ENTITY,
    options: {
      onSuccess: () => {
        console.log('Product created successfully');
      },
    },
  });
};

// UPDATE product
export const useUpdateProduct = () => {
  return useApiUpdate<Product, UpdateProductDto>({
    apiService: productApiService,
    entity: ENTITY,
  });
};

// DELETE product
export const useDeleteProduct = () => {
  return useApiDelete({
    apiService: productApiService,
    entity: ENTITY,
    options: {
      onSuccess: () => {
        console.log('Product deleted successfully');
      },
    },
  });
};

// Custom: Get products by category
export const useProductsByCategory = (categoryId: number) => {
  return useApiCustomQuery<Product[]>({
    entity: ENTITY,
    queryKey: ['category', categoryId],
    queryFn: () => productApiService.custom<Product[]>('get', `category/${categoryId}`),
    options: {
      enabled: !!categoryId,
    },
  });
};
```

### 3.4. Product List Component

```typescript
import { useProducts, useDeleteProduct } from '../../hooks/useProducts';
import { Link } from '@tanstack/react-router';

export function ProductsList() {
  const { data: products, isLoading, error } = useProducts();
  const deleteProduct = useDeleteProduct();

  const handleDelete = async (id: number) => {
    if (!confirm('Bạn có chắc muốn xóa?')) return;
    
    try {
      await deleteProduct.mutateAsync(id);
    } catch (error) {
      alert('Không thể xóa sản phẩm');
    }
  };

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div>
      <h1>Products</h1>
      <Link to="/products/create">Create New</Link>
      
      <table>
        <thead>
          <tr>
            <th>Name</th>
            <th>Price</th>
            <th>Stock</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          {products?.map((product) => (
            <tr key={product.id}>
              <td>{product.name}</td>
              <td>${product.price}</td>
              <td>{product.stock}</td>
              <td>
                <Link to="/products/$id" params={{ id: product.id.toString() }}>
                  View
                </Link>
                <Link to="/products/$id/edit" params={{ id: product.id.toString() }}>
                  Edit
                </Link>
                <button onClick={() => handleDelete(product.id)}>
                  Delete
                </button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

### 3.5. Product Create Component

```typescript
import { useCreateProduct } from '../../hooks/useProducts';
import { useNavigate } from '@tanstack/react-router';
import { CreateProductDto } from '../../types/product';
import { useState } from 'react';

export function CreateProduct() {
  const navigate = useNavigate();
  const createProduct = useCreateProduct();
  const [formData, setFormData] = useState<CreateProductDto>({
    name: '',
    description: '',
    price: 0,
    stock: 0,
  });

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    
    try {
      await createProduct.mutateAsync(formData);
      navigate({ to: '/products' });
    } catch (error) {
      alert('Error creating product');
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <h1>Create Product</h1>
      
      <input
        type="text"
        placeholder="Name"
        value={formData.name}
        onChange={(e) => setFormData({ ...formData, name: e.target.value })}
      />
      
      <textarea
        placeholder="Description"
        value={formData.description}
        onChange={(e) => setFormData({ ...formData, description: e.target.value })}
      />
      
      <input
        type="number"
        placeholder="Price"
        value={formData.price}
        onChange={(e) => setFormData({ ...formData, price: parseFloat(e.target.value) })}
      />
      
      <input
        type="number"
        placeholder="Stock"
        value={formData.stock}
        onChange={(e) => setFormData({ ...formData, stock: parseInt(e.target.value) })}
      />
      
      <button type="submit" disabled={createProduct.isPending}>
        {createProduct.isPending ? 'Creating...' : 'Create'}
      </button>
    </form>
  );
}
```

## 4. Example Usage - User

### 4.1. User Types (`src/types/user.ts`)

```typescript
export interface User {
  id: number;
  email: string;
  name: string;
  role: 'admin' | 'user';
  createdAt: string;
}

export interface CreateUserDto {
  email: string;
  name: string;
  password: string;
  role: 'admin' | 'user';
}

export interface UpdateUserDto {
  email: string;
  name: string;
  role: 'admin' | 'user';
}
```

### 4.2. User API Service (`src/api/userApi.ts`)

```typescript
import { apiClient } from './axios';
import { ApiService } from './apiFactory';
import { User, CreateUserDto, UpdateUserDto } from '../types/user';

export const userApiService = new ApiService<User, CreateUserDto, UpdateUserDto>({
  endpoint: '/users',
  axiosInstance: apiClient,
});
```

### 4.3. User Hooks (`src/hooks/useUsers.ts`)

```typescript
import { 
  useApiList, 
  useApiDetail, 
  useApiCreate, 
  useApiUpdate, 
  useApiDelete 
} from './useApi';
import { userApiService } from '../api/userApi';
import { User, CreateUserDto, UpdateUserDto } from '../types/user';

const ENTITY = 'users';

export const useUsers = (params?: { role?: string }) => {
  return useApiList<User>({
    apiService: userApiService,
    entity: ENTITY,
    params,
  });
};

export const useUser = (id: number) => {
  return useApiDetail<User>({
    apiService: userApiService,
    entity: ENTITY,
    id,
  });
};

export const useCreateUser = () => {
  return useApiCreate<User, CreateUserDto>({
    apiService: userApiService,
    entity: ENTITY,
  });
};

export const useUpdateUser = () => {
  return useApiUpdate<User, UpdateUserDto>({
    apiService: userApiService,
    entity: ENTITY,
  });
};

export const useDeleteUser = () => {
  return useApiDelete({
    apiService: userApiService,
    entity: ENTITY,
  });
};
```

### 4.4. User List Component

```typescript
import { useUsers, useDeleteUser } from '../../hooks/useUsers';

export function UsersList() {
  const { data: users, isLoading } = useUsers();
  const deleteUser = useDeleteUser();

  if (isLoading) return <div>Loading...</div>;

  return (
    <div>
      <h1>Users</h1>
      {users?.map((user) => (
        <div key={user.id}>
          <h3>{user.name}</h3>
          <p>{user.email}</p>
          <p>Role: {user.role}</p>
          <button onClick={() => deleteUser.mutate(user.id)}>
            Delete
          </button>
        </div>
      ))}
    </div>
  );
}
```

## 5. Advanced Features

### 5.1. Optimistic Updates

```typescript
export const useUpdateProduct = () => {
  const queryClient = useQueryClient();
  
  return useApiUpdate<Product, UpdateProductDto>({
    apiService: productApiService,
    entity: 'products',
    options: {
      // Optimistic update
      onMutate: async ({ id, data }) => {
        // Cancel outgoing refetches
        await queryClient.cancelQueries({ queryKey: ['products', 'detail', id] });
        
        // Snapshot previous value
        const previousProduct = queryClient.getQueryData(['products', 'detail', id]);
        
        // Optimistically update
        queryClient.setQueryData(['products', 'detail', id], (old: Product | undefined) => {
          if (!old) return old;
          return { ...old, ...data };
        });
        
        return { previousProduct };
      },
      
      // Rollback on error
      onError: (err, variables, context) => {
        if (context?.previousProduct) {
          queryClient.setQueryData(
            ['products', 'detail', variables.id],
            context.previousProduct
          );
        }
      },
    },
  });
};
```

### 5.2. Pagination Support

```typescript
// Add to apiFactory.ts
export interface PaginatedResponse<T> {
  data: T[];
  page: number;
  pageSize: number;
  total: number;
  totalPages: number;
}

// In ApiService class
async getPaginated(page: number = 1, pageSize: number = 10): Promise<PaginatedResponse<TData>> {
  const response = await this.axios.get<PaginatedResponse<TData>>(this.endpoint, {
    params: { page, pageSize },
  });
  return response.data;
}

// Hook for pagination
export function useApiPaginated<TData = any>({
  apiService,
  entity,
  page = 1,
  pageSize = 10,
  options,
}: {
  apiService: ApiService<TData>;
  entity: string;
  page?: number;
  pageSize?: number;
  options?: Omit<UseQueryOptions<PaginatedResponse<TData>>, 'queryKey' | 'queryFn'>;
}) {
  const queryKeys = createQueryKeys(entity);

  return useQuery<PaginatedResponse<TData>>({
    queryKey: [...queryKeys.lists(), 'paginated', { page, pageSize }],
    queryFn: () => apiService.getPaginated(page, pageSize),
    keepPreviousData: true,
    ...options,
  });
}
```

## Tổng kết

Hệ thống này cung cấp:

✅ **Universal hooks** có thể dùng cho bất kỳ entity nào
✅ **Type-safe** với TypeScript generics
✅ **Reusable** - chỉ cần tạo ApiService và sử dụng hooks
✅ **Flexible** - hỗ trợ custom queries và mutations
✅ **Automatic cache management** - invalidation tự động
✅ **Easy to extend** - thêm features như pagination, optimistic updates
✅ **Clean separation** - API logic tách biệt khỏi components

Để sử dụng cho entity mới, chỉ cần:
1. Tạo types
2. Tạo ApiService instance
3. Tạo hooks wrapper (optional, có thể dùng trực tiếp `useApiList`, `useApiCreate`, etc.)
4. Sử dụng trong components
