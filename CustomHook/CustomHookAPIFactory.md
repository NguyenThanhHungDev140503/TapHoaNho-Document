# Universal API Hook với TanStack Query + Axios (Hybrid Approach)

Tài liệu này mô tả hệ thống **Hybrid Approach** cho API layer: **BaseApiService** với CRUD cơ bản + **Feature-specific extensions**, phù hợp với `ApiResponse<T>` wrapper và `PagedList<T>` pagination.

## Kiến trúc Hybrid Approach

**Hybrid Approach** kết hợp:
- **ApiServiceInterface** (`src/lib/api/base/ApiServiceInterface.ts`): Định nghĩa contract chung cho mọi API service  
  - Chuẩn hóa: `getAll`, `getPaginated`, `getById`, `create`, `update`, `patch`, `delete`
  - Generic theo `TData`, `TCreate`, `TUpdate`
- **BaseApiService** (`src/lib/api/base/BaseApiService.ts`): Class cơ bản implement `ApiServiceInterface`  
  - Thực thi CRUD operations chuẩn trên `axiosClient`
  - Xử lý `ApiResponse<T>` và `PagedList<T>` ở một chỗ
  - Tích hợp error handling dựa trên `isError` flag
- **apiResponseAdapter** (`src/lib/api/base/apiResponseAdapter.ts`):  
  - Cung cấp helpers dùng lại được:
    - `unwrapResponse<T>(response: ApiResponse<T>): T`
    - `handleApiError(error: unknown): never`
    - (Optional) type guards cho `ApiResponse`
- **API types** (`src/lib/api/types/api.types.ts`):  
  - Re-export `ApiResponse`, `PagedList`, `PagedRequest` từ `src/lib/axios.ts`
  - Định nghĩa thêm `ApiConfig`, `QueryParams` để tránh lặp type giữa các services
- **Feature-specific extensions**: Mỗi feature extend `BaseApiService` và thêm custom methods
- **Type-safe**: Đảm bảo type safety với TypeScript generics
- **Compatible**: Tương thích với cấu trúc hiện tại (ApiResponse wrapper, Axios instance)

## 1. Core API Infrastructure

### 1.1. ApiServiceInterface (`src/lib/api/base/ApiServiceInterface.ts`)

```typescript
// shiny-carnival/frontend/src/lib/api/base/ApiServiceInterface.ts
import type { PagedList, PagedRequest } from '../../axios';

/**
 * ApiServiceInterface - Contract chung cho tất cả API services
 *
 * Mục tiêu:
 * - Chuẩn hóa signature cho CRUD + pagination + custom endpoints
 * - Cho phép BaseApiService và các service khác implement một cách nhất quán
 */
export interface ApiServiceInterface<TData = any, TCreate = any, TUpdate = any> {
  // GET all (không phân trang hoặc có params filter đơn giản)
  getAll(params?: Record<string, any>): Promise<TData[]>;

  // GET với phân trang chuẩn PagedRequest → PagedList<TData>
  getPaginated(params?: PagedRequest): Promise<PagedList<TData>>;

  // GET by ID
  getById(id: string | number): Promise<TData>;

  // POST - Create
  create(data: TCreate): Promise<TData>;

  // PUT - Full update
  update(id: string | number, data: TUpdate): Promise<TData>;

  // PATCH - Partial update (chỉ một số entity/endpoint hỗ trợ)
  patch(id: string | number, data: Partial<TUpdate>): Promise<TData>;

  // DELETE
  delete(id: string | number): Promise<void>;

  // Custom endpoint cho các trường hợp đặc biệt
  custom<TResponse = any>(
    method: 'get' | 'post' | 'put' | 'patch' | 'delete',
    path: string,
    data?: any,
    params?: Record<string, any>
  ): Promise<TResponse>;
}
```

### 1.2. apiResponseAdapter (`src/lib/api/base/apiResponseAdapter.ts`)

```typescript
// shiny-carnival/frontend/src/lib/api/base/apiResponseAdapter.ts
import type { ApiResponse } from '../../axios';

/**
 * Unwrap ApiResponse<T> về T và ném lỗi nếu isError === true hoặc data null.
 * Dùng chung cho BaseApiService và các chỗ cần xử lý ApiResponse thô.
 */
export function unwrapResponse<T>(response: ApiResponse<T>): T {
  if (response.isError || response.data == null) {
    throw new Error(response.message || 'API request failed');
  }
  return response.data;
}

/**
 * Chuẩn hóa error từ Axios/API về dạng Error để upper layers dễ xử lý.
 * Có thể mở rộng sau này để map theo statusCode, validation errors, v.v.
 */
export function handleApiError(error: unknown): never {
  // Nếu error đã là Error và có message rõ ràng → ném lại
  if (
    typeof error === 'object' &&
    error !== null &&
    'message' in error &&
    typeof (error as any).message === 'string'
  ) {
    throw error as Error;
  }

  // Fallback chung
  throw new Error('Unexpected API error');
}
```

### 1.3. BaseApiService (`src/lib/api/base/BaseApiService.ts`)

```typescript
import { AxiosInstance } from 'axios';
import type { ApiResponse, PagedList, PagedRequest } from '../../axios';
import { unwrapResponse } from './apiResponseAdapter';

export interface ApiConfig<TData = any, TCreate = any, TUpdate = any> {
  endpoint: string;
  axiosInstance: AxiosInstance;
}

export interface QueryParams {
  [key: string]: any;
}

/**
 * BaseApiService - Base class cho tất cả API services
 * 
 * Xử lý:
 * - ApiResponse<T> wrapper (unwrap data field, check isError)
 * - PagedList<T> pagination
 * - Error handling tự động
 * 
 * @template TData - Type của entity data
 * @template TCreate - Type của create request
 * @template TUpdate - Type của update request
 */
export class BaseApiService<TData = any, TCreate = any, TUpdate = any> {
  protected endpoint: string;
  protected axios: AxiosInstance;

  constructor(config: ApiConfig<TData, TCreate, TUpdate>) {
    this.endpoint = config.endpoint;
    this.axios = config.axiosInstance;
  }

  /**
   * GET all - Lấy danh sách với query params (không phân trang)
   * Trả về mảng items trực tiếp
   */
  async getAll(params?: QueryParams): Promise<TData[]> {
    const response = await this.axios.get<ApiResponse<TData[]>>(
      this.endpoint, 
      { params }
    );
    return unwrapResponse(response);
  }

  /**
   * GET paginated - Lấy danh sách với phân trang
   * Trả về PagedList<TData> với pagination metadata
   */
  async getPaginated(params?: PagedRequest): Promise<PagedList<TData>> {
    const response = await this.axios.get<ApiResponse<PagedList<TData>>>(
      this.endpoint,
      { params }
    );
    return unwrapResponse(response);
  }

  /**
   * GET by ID - Lấy chi tiết một item
   */
  async getById(id: string | number): Promise<TData> {
    const response = await this.axios.get<ApiResponse<TData>>(
      `${this.endpoint}/${id}`
    );
    return unwrapResponse(response);
  }

  /**
   * POST - Create - Tạo mới item
   */
  async create(data: TCreate): Promise<TData> {
    const response = await this.axios.post<ApiResponse<TData>>(
      this.endpoint,
      data
    );
    return unwrapResponse(response);
  }

  /**
   * PUT - Update - Cập nhật toàn bộ item
   */
  async update(id: string | number, data: TUpdate): Promise<TData> {
    const response = await this.axios.put<ApiResponse<TData>>(
      `${this.endpoint}/${id}`,
      data
    );
    return unwrapResponse(response);
  }

  /**
   * PATCH - Partial Update - Cập nhật một phần item
   * ⚠️ Chỉ dùng cho các entity có endpoint dạng PATCH /{entity}/{id}.
   *    Nếu backend sử dụng đường dẫn khác (vd: /orders/{id}/status),
   *    hãy dùng apiService.custom() hoặc viết custom method thay vì patch().
   */
  async patch(id: string | number, data: Partial<TUpdate>): Promise<TData> {
    const response = await this.axios.patch<ApiResponse<TData>>(
      `${this.endpoint}/${id}`,
      data
    );
    return unwrapResponse(response);
  }

  /**
   * DELETE - Xóa item
   * Backend trả về ApiResponse<object> với data = null
   */
  async delete(id: string | number): Promise<void> {
    const response = await this.axios.delete<ApiResponse<null>>(
      `${this.endpoint}/${id}`
    );
    if (response.isError) {
      throw new Error(response.message || 'Delete failed');
    }
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
    return this.unwrapResponse(response);
  }
}
```

## 1.1. Giải thích chi tiết về `unwrapResponse`

### `unwrapResponse` là gì?

`unwrapResponse` (được định nghĩa trong `apiResponseAdapter.ts`) là helper chung cho toàn bộ API layer, dùng để:

1. **Unwrap `ApiResponse<T>` → `T`**: Lấy `data` field từ wrapper `ApiResponse<T>`
2. **Kiểm tra `isError`**: Throw error nếu `isError === true` hoặc `data === null`
3. **Đảm bảo type safety**: TypeScript đảm bảo return type đúng là `T`, không phải `ApiResponse<T>`

### Vấn đề với `ApiResponse<T>` wrapper

Backend của bạn trả về response theo cấu trúc:

```typescript
interface ApiResponse<T> {
  isError: boolean;      // true nếu có lỗi
  message: string;       // Thông báo lỗi hoặc success
  data: T | null;        // Dữ liệu thực tế (null nếu error)
  timestamp: string;     // Thời gian response
}
```

**Ví dụ response từ backend**:

```json
{
  "isError": false,
  "message": "Lấy danh sách sản phẩm thành công",
  "data": [
    { "id": 1, "name": "Coca Cola", "price": 12000 },
    { "id": 2, "name": "Pepsi", "price": 11000 }
  ],
  "timestamp": "2024-01-20T14:30:00Z"
}
```

### Vấn đề khi KHÔNG dùng `unwrapResponse`

Nếu không có `unwrapResponse`, bạn phải xử lý ở **mọi method**:

```typescript
// ❌ KHÔNG DÙNG unwrapResponse - Phải xử lý ở mọi nơi
async getAll(params?: QueryParams): Promise<TData[]> {
  const response = await this.axios.get<ApiResponse<TData[]>>(
    this.endpoint, 
    { params }
  );
  
  // Phải check isError ở mọi method
  if (response.isError || !response.data) {
    throw new Error(response.message || 'API request failed');
  }
  
  return response.data; // Phải unwrap thủ công
}

async getById(id: string | number): Promise<TData> {
  const response = await this.axios.get<ApiResponse<TData>>(
    `${this.endpoint}/${id}`
  );
  
  // Lặp lại code check isError
  if (response.isError || !response.data) {
    throw new Error(response.message || 'API request failed');
  }
  
  return response.data; // Lặp lại code unwrap
}

// ... phải lặp lại ở create, update, patch, delete, custom
```

**Vấn đề**:
- ❌ Code lặp lại ở mọi method (DRY violation)
- ❌ Dễ quên check `isError`
- ❌ Khó maintain khi thay đổi logic error handling

### Giải pháp: Dùng `unwrapResponse` từ `apiResponseAdapter`

```typescript
// ✅ DÙNG unwrapResponse từ apiResponseAdapter - Code tập trung, tái sử dụng
import { unwrapResponse } from './apiResponseAdapter';

async getAll(params?: QueryParams): Promise<TData[]> {
  const response = await this.axios.get<ApiResponse<TData[]>>(
    this.endpoint, 
    { params }
  );
  return unwrapResponse(response); // ✅ Đơn giản, không lặp code
}

async getById(id: string | number): Promise<TData> {
  const response = await this.axios.get<ApiResponse<TData>>(
    `${this.endpoint}/${id}`
  );
  return unwrapResponse(response); // ✅ Tái sử dụng logic
}

async create(data: TCreate): Promise<TData> {
  const response = await this.axios.post<ApiResponse<TData>>(
    this.endpoint,
    data
  );
  return unwrapResponse(response); // ✅ Consistent error handling
}
```

### Lợi ích của `unwrapResponse`

#### 1. **DRY (Don't Repeat Yourself)**
- Logic unwrap và error checking chỉ viết **một lần**
- Tất cả methods tái sử dụng cùng logic

#### 2. **Consistent Error Handling**
- Tất cả methods xử lý error **giống nhau**
- Dễ thay đổi logic error handling (chỉ sửa một chỗ)

#### 3. **Type Safety**
- TypeScript đảm bảo return type đúng
- `unwrapResponse<T>` trả về `T`, không phải `ApiResponse<T>`

#### 4. **Clean API**
- Methods trả về `T` trực tiếp, không phải `ApiResponse<T>`
- Component không cần biết về `ApiResponse` wrapper

### Ví dụ so sánh

#### ❌ Không dùng `unwrapResponse`:
```typescript
// Component phải xử lý ApiResponse wrapper
const { data } = useQuery({
  queryKey: ['products'],
  queryFn: async () => {
    const response = await productApiService.getAll();
    // Phải check isError ở component
    if (response.isError) {
      throw new Error(response.message);
    }
    return response.data; // Phải unwrap thủ công
  }
});
```

#### ✅ Dùng `unwrapResponse`:
```typescript
// Component chỉ nhận data trực tiếp
const { data } = useQuery({
  queryKey: ['products'],
  queryFn: () => productApiService.getAll(), // ✅ Trả về Product[] trực tiếp
});
// data là Product[] chứ không phải ApiResponse<Product[]>
```

### Tóm tắt

`unwrapResponse` là một helper method để:
- ✅ **Unwrap** `ApiResponse<T>` → `T` (lấy `data` field)
- ✅ **Check error** và throw nếu `isError === true`
- ✅ **Tập trung logic** error handling ở một chỗ
- ✅ **Giữ API sạch** - Methods trả về `T` trực tiếp, không phải `ApiResponse<T>`

**Không dùng `unwrapResponse`** sẽ dẫn đến:
- ❌ Code lặp lại ở mọi method
- ❌ Khó maintain và update
- ❌ Dễ quên check error
- ❌ Component phải biết về `ApiResponse` wrapper

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
import { BaseApiService, QueryParams } from '../lib/api/base/BaseApiService';

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

/**
 * QUERY KEY FACTORY - GIẢI THÍCH CHI TIẾT
 * 
 * Query Key Factory là một pattern quan trọng trong TanStack Query giúp:
 * - Tổ chức query keys theo cấu trúc phân cấp
 * - Dễ dàng invalidate/update cache theo nhóm
 * - Tránh lỗi typo và đảm bảo type-safety
 * - Tái sử dụng và mở rộng dễ dàng
 * 
 * CẤU TRÚC PHÂN CẤP:
 * 
 * [entity]                           // all: Base key cho toàn bộ entity
 *   ├── [entity, 'list']             // lists: Base cho tất cả list queries
 *   │   └── [entity, 'list', params] // list: List query với params cụ thể
 *   └── [entity, 'detail']            // details: Base cho tất cả detail queries
 *       └── [entity, 'detail', id]   // detail: Detail query với id cụ thể
 * 
 * VÍ DỤ VỚI ENTITY "products":
 * 
 * const productKeys = createQueryKeys('products');
 * 
 * productKeys.all                    // ['products']
 * productKeys.lists()                 // ['products', 'list']
 * productKeys.list({ search: 'laptop' })  // ['products', 'list', { search: 'laptop' }]
 * productKeys.details()               // ['products', 'detail']
 * productKeys.detail(123)            // ['products', 'detail', 123]
 * 
 * LỢI ÍCH CỦA PATTERN NÀY:
 * 
 * 1. Invalidation linh hoạt:
 *    - Invalidate tất cả: queryClient.invalidateQueries({ queryKey: queryKeys.all })
 *    - Invalidate tất cả lists: queryClient.invalidateQueries({ queryKey: queryKeys.lists() })
 *    - Invalidate một list cụ thể: queryClient.invalidateQueries({ queryKey: queryKeys.list(params) })
 *    - Invalidate một detail cụ thể: queryClient.invalidateQueries({ queryKey: queryKeys.detail(id) })
 * 
 * 2. Type Safety:
 *    - Sử dụng 'as const' đảm bảo TypeScript biết chính xác cấu trúc của query key
 *    - Tránh lỗi typo, autocomplete tốt hơn
 * 
 * 3. Tái sử dụng:
 *    - Chỉ cần truyền entity string, factory tự tạo cấu trúc
 *    - Có thể dùng cho bất kỳ entity nào: products, users, orders, etc.
 * 
 * 4. Dễ mở rộng:
 *    - Có thể thêm custom keys như search, byCategory, etc.
 */

// ==================== Hook Configuration Types ====================
/**
 * HOOK CONFIGURATION TYPES - GIẢI THÍCH CHI TIẾT
 * 
 * Hook Configuration Types là các TypeScript interfaces định nghĩa cấu trúc tham số
 * cho các custom hooks. Chúng đảm bảo:
 * - Type safety khi sử dụng hooks
 * - Tái sử dụng các types từ TanStack Query
 * - Loại trừ các thuộc tính đã được xử lý nội bộ
 * - Cho phép tùy chỉnh các options từ TanStack Query
 */

/**
 * UseApiListConfig - Configuration cho List Query (GET all items)
 * 
 * Generic Types:
 *   - TData: Kiểu dữ liệu trả về (mảng)
 *   - TError: Kiểu lỗi (mặc định Error)
 * 
 * Thuộc tính:
 *   - apiService: Service để gọi API
 *   - entity: Tên entity (ví dụ: 'products', 'users')
 *   - params: Query params tùy chọn (filter, search, pagination...)
 *   - options: Options từ TanStack Query, loại trừ 'queryKey' và 'queryFn'
 *             (vì đã được xử lý nội bộ bởi hook)
 * 
 * Tại sao dùng Omit?
 *   - Ngăn user override queryKey và queryFn (đã được hook xử lý)
 *   - Vẫn cho phép các options khác: staleTime, gcTime, enabled, onSuccess, etc.
 * 
 * Ví dụ sử dụng:
 *   useApiList<Product>({
 *     apiService: productApiService,
 *     entity: 'products',
 *     params: { search: 'laptop' },
 *     options: {
 *       staleTime: 1000 * 60 * 5,
 *       enabled: true,
 *       onSuccess: (data) => console.log(data),
 *     },
 *   });
 */
export interface UseApiListConfig<TData, TError = Error> {
  apiService: BaseApiService<TData>;
  entity: string;
  params?: QueryParams;
  options?: Omit<
    UseQueryOptions<TData[], TError, TData[], QueryKey>,
    'queryKey' | 'queryFn'
  >;
}

/**
 * UseApiDetailConfig - Configuration cho Detail Query (GET item by ID)
 * 
 * Khác biệt so với UseApiListConfig:
 *   - id: Bắt buộc để lấy item theo ID
 *   - TData (không phải TData[]): Trả về một object, không phải mảng
 *   - UseQueryOptions<TData, ...>: Type cho single item
 * 
 * Ví dụ sử dụng:
 *   useApiDetail<Product>({
 *     apiService: productApiService,
 *     entity: 'products',
 *     id: 123,
 *     options: {
 *       enabled: !!id,
 *       staleTime: 1000 * 60 * 10,
 *     },
 *   });
 */
export interface UseApiDetailConfig<TData, TError = Error> {
  apiService: BaseApiService<TData>;
  entity: string;
  id: string | number;
  options?: Omit<
    UseQueryOptions<TData, TError, TData, QueryKey>,
    'queryKey' | 'queryFn'
  >;
}

/**
 * UseApiMutationConfig - Configuration cho Mutations (POST/PUT/PATCH/DELETE)
 * 
 * Generic Types:
 *   - TData: Kiểu dữ liệu trả về từ mutation
 *   - TVariables: Kiểu dữ liệu đầu vào (CreateDto, UpdateDto...)
 *   - TError: Kiểu lỗi
 * 
 * Thuộc tính:
 *   - apiService: Service để gọi API
 *   - entity: Tên entity
 *   - invalidateQueries: Danh sách query keys cần invalidate thêm (ngoài các keys tự động)
 *   - options: Options từ TanStack Query cho mutations (không dùng Omit vì mutation
 *             không tự tạo mutationFn từ apiService)
 * 
 * Ví dụ sử dụng:
 *   useApiCreate<Product, CreateProductDto>({
 *     apiService: productApiService,
 *     entity: 'products',
 *     invalidateQueries: ['categories', 'stats'],
 *     options: {
 *       onSuccess: (data) => toast.success('Created!'),
 *       onError: (error) => toast.error('Failed!'),
 *     },
 *   });
 */
export interface UseApiMutationConfig<TData, TVariables, TError = Error> {
  apiService: BaseApiService<TData>;
  entity: string;
  invalidateQueries?: string[];
  options?: UseMutationOptions<TData, TError, TVariables>;
}

/**
 * SO SÁNH VÀ VÍ DỤ THỰC TẾ
 * 
 * 1. TẠI SAO DÙNG OMIT CHO QUERY OPTIONS?
 * 
 * ❌ Nếu không dùng Omit (cho phép user truyền queryKey và queryFn):
 *    useApiList({
 *      apiService: productApiService,
 *      entity: 'products',
 *      options: {
 *        queryKey: ['wrong-key'],      // ❌ Override key của hook
 *        queryFn: () => fetchWrongData(), // ❌ Override function của hook
 *      },
 *    });
 *    // → Hook sẽ không hoạt động đúng!
 * 
 * ✅ Với Omit (loại bỏ queryKey và queryFn):
 *    useApiList({
 *      apiService: productApiService,
 *      entity: 'products',
 *      options: {
 *        staleTime: 5000,    // ✅ OK
 *        enabled: true,      // ✅ OK
 *        queryKey: ['wrong'], // ❌ TypeScript error!
 *        queryFn: () => {},   // ❌ TypeScript error!
 *      },
 *    });
 * 
 * 2. VÍ DỤ SỬ DỤNG OPTIONS:
 * 
 * // Custom staleTime và gcTime
 * useApiList<Product>({
 *   apiService: productApiService,
 *   entity: 'products',
 *   options: {
 *     staleTime: 1000 * 60 * 5,  // Data fresh trong 5 phút
 *     gcTime: 1000 * 60 * 30,    // Giữ cache 30 phút
 *   },
 * });
 * 
 * // Conditional Fetching
 * useApiDetail<Product>({
 *   apiService: productApiService,
 *   entity: 'products',
 *   id: productId,
 *   options: {
 *     enabled: !!productId && user.isAuthenticated, // Chỉ fetch khi có id và user đã login
 *   },
 * });
 * 
 * // Retry Logic
 * useApiList<Product>({
 *   apiService: productApiService,
 *   entity: 'products',
 *   options: {
 *     retry: 3,
 *     retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
 *   },
 * });
 * 
 * // Callbacks
 * useApiList<Product>({
 *   apiService: productApiService,
 *   entity: 'products',
 *   options: {
 *     onSuccess: (data) => {
 *       console.log('Fetched:', data.length, 'products');
 *       analytics.track('products_loaded', { count: data.length });
 *     },
 *     onError: (error) => {
 *       console.error('Error:', error);
 *       errorReporting.captureException(error);
 *     },
 *   },
 * });
 * 
 *

// ==================== GET ALL Hook ====================
/**
 * useApiList - Hook để fetch danh sách items (GET all)
 * 
 * MỤC ĐÍCH:
 *   - Fetch danh sách tất cả items của một entity
 *   - Hỗ trợ query params để filter, search, pagination
 *   - Tự động quản lý cache và refetching
 * 
 * CÁCH HOẠT ĐỘNG:
 *   1. Tạo query keys từ entity name và params
 *   2. Sử dụng useQuery từ TanStack Query
 *   3. Gọi apiService.getAll(params) để fetch data
 *   4. Cache được quản lý tự động dựa trên queryKey
 * 
 * THAM SỐ:
 *   - apiService: ApiService instance để gọi API
 *   - entity: Tên entity (ví dụ: 'products', 'users')
 *   - params: Query params tùy chọn (filter, search, pagination...)
 *   - options: Options từ TanStack Query (staleTime, gcTime, enabled, onSuccess, etc.)
 * 
 * RETURN VALUES (từ useQuery):
 *   - data: TData[] | undefined - Danh sách items
 *   - isLoading: boolean - Đang fetch lần đầu
 *   - isPending: boolean - Chưa có data
 *   - isFetching: boolean - Đang fetch (bao gồm background refetch)
 *   - isError: boolean - Có lỗi xảy ra
 *   - error: TError | null - Error object
 *   - refetch: () => Promise<QueryObserverResult> - Hàm để refetch thủ công
 *   - ... các properties khác từ useQuery
 * 
 * QUERY KEY STRUCTURE:
 *   - Với params: ['products', 'list', { search: 'laptop', page: 1 }]
 *   - Không params: ['products', 'list', undefined]
 * 
 * VÍ DỤ SỬ DỤNG:
 * 
 * // Fetch tất cả products
 * const { data: products, isLoading } = useApiList<Product>({
 *   apiService: productApiService,
 *   entity: 'products',
 * });
 * 
 * // Fetch với filter
 * const { data: products } = useApiList<Product>({
 *   apiService: productApiService,
 *   entity: 'products',
 *   params: { search: 'laptop', category: 'electronics' },
 *   options: {
 *     staleTime: 1000 * 60 * 5, // 5 phút
 *   },
 * });
 * 
 * // Conditional fetching
 * const { data: products } = useApiList<Product>({
 *   apiService: productApiService,
 *   entity: 'products',
 *   options: {
 *     enabled: user.isAuthenticated, // Chỉ fetch khi user đã login
 *   },
 * });
 * 
 * LƯU Ý:
 *   - Query key phụ thuộc vào params, nên mỗi bộ params khác nhau sẽ tạo cache riêng
 *   - Khi params thay đổi, query sẽ tự động refetch
 *   - Có thể dùng options để customize behavior (retry, staleTime, etc.)
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

// ==================== GET BY ID Hook ====================
/**
 * useApiDetail - Hook để fetch một item cụ thể theo ID (GET by ID)
 * 
 * MỤC ĐÍCH:
 *   - Fetch chi tiết một item dựa trên ID
 *   - Tự động disable query khi không có ID
 *   - Cache riêng cho từng ID
 * 
 * CÁCH HOẠT ĐỘNG:
 *   1. Tạo query key từ entity name và id
 *   2. Sử dụng useQuery với enabled: !!id (chỉ fetch khi có id)
 *   3. Gọi apiService.getById(id) để fetch data
 *   4. Cache được quản lý tự động, mỗi ID có cache riêng
 * 
 * THAM SỐ:
 *   - apiService: ApiService instance để gọi API
 *   - entity: Tên entity (ví dụ: 'products', 'users')
 *   - id: ID của item cần fetch (string | number)
 *   - options: Options từ TanStack Query
 * 
 * RETURN VALUES (từ useQuery):
 *   - data: TData | undefined - Item data
 *   - isLoading: boolean - Đang fetch lần đầu
 *   - isPending: boolean - Chưa có data
 *   - isFetching: boolean - Đang fetch
 *   - isError: boolean - Có lỗi xảy ra
 *   - error: TError | null - Error object
 *   - refetch: () => Promise<QueryObserverResult> - Hàm để refetch thủ công
 * 
 * QUERY KEY STRUCTURE:
 *   - ['products', 'detail', 123]1
 *   - ['users', 'detail', 'user-abc']
 * 
 * VÍ DỤ SỬ DỤNG:
 * 
 * // Fetch product by ID
 * const { data: product, isLoading } = useApiDetail<Product>({
 *   apiService: productApiService,
 *   entity: 'products',
 *   id: 123,
 * });
 * 
 * // Fetch với conditional và custom options
 * const { data: product } = useApiDetail<Product>({
 *   apiService: productApiService,
 *   entity: 'products',
 *   id: productId,
 *   options: {
 *     enabled: !!productId && user.isAuthenticated,
 *     staleTime: 1000 * 60 * 10, // 10 phút cho detail
 *     retry: 2,
 *   },
 * });
 * 
 * // Sử dụng trong component
 * function ProductDetail({ productId }: { productId: number }) {
 *   const { data: product, isLoading, error } = useApiDetail<Product>({
 *     apiService: productApiService,
 *     entity: 'products',
 *     id: productId,
 *   });
 * 
 *   if (isLoading) return <div>Loading...</div>;
 *   if (error) return <div>Error: {error.message}</div>;
 *   if (!product) return <div>Product not found</div>;
 * 
 *   return <div>{product.name}</div>;
 * }
 * 
 * LƯU Ý:
 *   - Hook tự động disable khi id là falsy (null, undefined, 0, '')
 *   - Mỗi ID tạo một cache entry riêng
 *   - Khi id thay đổi, query sẽ tự động refetch với ID mới
 *   - Nên set staleTime cao hơn cho detail queries vì ít thay đổi hơn list
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
    enabled: !!id,
    ...options,
  });
}

// ==================== CREATE Hook ====================
/**
 * useApiCreate - Hook để tạo mới một item (POST)
 * 
 * MỤC ĐÍCH:
 *   - Tạo mới một item trong database
 *   - Tự động invalidate cache sau khi tạo thành công
 *   - Hỗ trợ optimistic updates và error handling
 * 
 * CÁCH HOẠT ĐỘNG:
 *   1. Sử dụng useMutation từ TanStack Query
 *   2. Gọi apiService.create(data) khi mutate
 *   3. Sau khi thành công:
 *      - Tự động invalidate tất cả list queries của entity
 *      - Invalidate các queries bổ sung nếu có
 *      - Gọi onSuccess callback của user nếu có
 * 
 * THAM SỐ:
 *   - apiService: ApiService instance
 *   - entity: Tên entity
 *   - invalidateQueries: Mảng các query keys cần invalidate thêm (optional)
 *   - options: Options từ TanStack Query (onSuccess, onError, onMutate, etc.)
 * 
 * RETURN VALUES (từ useMutation):
 *   - mutate: (variables: TCreate, options?) => void - Hàm để trigger mutation
 *   - mutateAsync: (variables: TCreate) => Promise<TData> - Async version
 *   - isPending: boolean - Đang thực hiện mutation
 *   - isError: boolean - Có lỗi xảy ra
 *   - error: TError | null - Error object
 *   - isSuccess: boolean - Mutation thành công
 *   - data: TData | undefined - Data trả về từ server
 *   - reset: () => void - Reset mutation state
 * 
 * CACHE INVALIDATION:
 *   - Tự động invalidate: ['products', 'list'] và tất cả list queries
 *   - Invalidate thêm: Các queries trong invalidateQueries array
 * 
 * VÍ DỤ SỬ DỤNG:
 * 
 * // Basic usage
 * const createProduct = useApiCreate<Product, CreateProductDto>({
 *   apiService: productApiService,
 *   entity: 'products',
 * });
 * 
 * // Sử dụng trong component
 * function CreateProductForm() {
 *   const createProduct = useApiCreate<Product, CreateProductDto>({
 *     apiService: productApiService,
 *     entity: 'products',
 *     options: {
 *       onSuccess: (data) => {
 *         toast.success('Product created!');
 *         navigate('/products');
 *       },
 *       onError: (error) => {
 *         toast.error('Failed to create product');
 *       },
 *     },
 *   });
 * 
 *   const handleSubmit = (formData: CreateProductDto) => {
 *     createProduct.mutate(formData);
 *   };
 * 
 *   return (
 *     <form onSubmit={handleSubmit}>
 *       {/* form fields */}
 *       <button disabled={createProduct.isPending}>
 *         {createProduct.isPending ? 'Creating...' : 'Create'}
 *       </button>
 *     </form>
 *   );
 * }
 * 
 * // Với invalidateQueries bổ sung
 * const createProduct = useApiCreate<Product, CreateProductDto>({
 *   apiService: productApiService,
 *   entity: 'products',
 *   invalidateQueries: ['categories', 'stats', 'dashboard'],
 * });
 * 
 * // Với async mutate
 * const handleCreate = async (data: CreateProductDto) => {
 *   try {
 *     const newProduct = await createProduct.mutateAsync(data);
 *     console.log('Created:', newProduct);
 *   } catch (error) {
 *     console.error('Error:', error);
 *   }
 * };
 * 
 * LƯU Ý:
 *   - Mutation không tự động refetch, chỉ invalidate cache
 *   - Các list queries sẽ được refetch khi component mount lại hoặc khi refocus
 *   - Có thể dùng optimistic updates trong options.onMutate để cải thiện UX
 *   - invalidateQueries dùng để invalidate các queries liên quan (ví dụ: stats, categories)
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
/**
 * useApiUpdate - Hook để cập nhật toàn bộ một item (PUT)
 * 
 * MỤC ĐÍCH:
 *   - Cập nhật toàn bộ fields của một item (full update)
 *   - Sử dụng HTTP PUT method
 *   - Tự động invalidate cache sau khi update thành công
 * 
 * CÁCH HOẠT ĐỘNG:
 *   1. Nhận { id, data } làm variables
 *   2. Gọi apiService.update(id, data) - HTTP PUT
 *   3. Sau khi thành công:
 *      - Invalidate tất cả list queries
 *      - Invalidate detail query của item vừa update
 *      - Invalidate các queries bổ sung
 * 
 * THAM SỐ:
 *   - apiService: ApiService instance
 *   - entity: Tên entity
 *   - invalidateQueries: Mảng query keys cần invalidate thêm (optional)
 *   - options: Options từ TanStack Query
 * 
 * VARIABLES FORMAT:
 *   { id: string | number, data: TUpdate }
 *   - id: ID của item cần update
 *   - data: Object chứa tất cả fields cần update (full update)
 * 
 * RETURN VALUES (từ useMutation):
 *   - mutate: ({ id, data }, options?) => void
 *   - mutateAsync: ({ id, data }) => Promise<TData>
 *   - isPending, isError, error, isSuccess, data, reset
 * 
 * CACHE INVALIDATION:
 *   - Invalidate: ['products', 'list'] và tất cả list queries
 *   - Invalidate: ['products', 'detail', id] - Detail query của item vừa update
 *   - Invalidate thêm: Các queries trong invalidateQueries
 * 
 * VÍ DỤ SỬ DỤNG:
 * 
 * // Basic usage
 * const updateProduct = useApiUpdate<Product, UpdateProductDto>({
 *   apiService: productApiService,
 *   entity: 'products',
 * });
 * 
 * // Sử dụng trong component
 * function EditProduct({ productId }: { productId: number }) {
 *   const updateProduct = useApiUpdate<Product, UpdateProductDto>({
 *     apiService: productApiService,
 *     entity: 'products',
 *     options: {
 *       onSuccess: () => {
 *         toast.success('Product updated!');
 *         navigate('/products');
 *       },
 *     },
 *   });
 * 
 *   const handleSubmit = (formData: UpdateProductDto) => {
 *     updateProduct.mutate({ id: productId, data: formData });
 *   };
 * 
 *   return (
 *     <form onSubmit={handleSubmit}>
 *       {/* form fields với tất cả fields */}
 *       <button disabled={updateProduct.isPending}>
 *         {updateProduct.isPending ? 'Updating...' : 'Update'}
 *       </button>
 *     </form>
 *   );
 * }
 * 
 * // Với optimistic update
 * const updateProduct = useApiUpdate<Product, UpdateProductDto>({
 *   apiService: productApiService,
 *   entity: 'products',
 *   options: {
 *     onMutate: async ({ id, data }) => {
 *       await queryClient.cancelQueries({ queryKey: ['products', 'detail', id] });
 *       const previous = queryClient.getQueryData(['products', 'detail', id]);
 *       queryClient.setQueryData(['products', 'detail', id], (old: Product) => ({
 *         ...old,
 *         ...data,
 *       }));
 *       return { previous };
 *     },
 *     onError: (err, variables, context) => {
 *       if (context?.previous) {
 *         queryClient.setQueryData(
 *           ['products', 'detail', variables.id],
 *           context.previous
 *         );
 *       }
 *     },
 *   },
 * });
 * 
 * LƯU Ý:
 *   - PUT method yêu cầu gửi toàn bộ object, không chỉ fields thay đổi
 *   - Nếu chỉ muốn update một vài fields, dùng useApiPatch thay vì useApiUpdate
 *   - Detail query của item sẽ được invalidate để refetch data mới nhất
 *   - List queries cũng được invalidate vì item trong list có thể đã thay đổi
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
/**
 * useApiPatch - Hook để cập nhật một phần item (PATCH - Partial Update)
 * 
 * MỤC ĐÍCH:
 *   - Cập nhật chỉ một số fields của item (partial update)
 *   - Sử dụng HTTP PATCH method
 *   - Chỉ cần gửi các fields muốn thay đổi, không cần gửi toàn bộ object
 * 
 * CÁCH HOẠT ĐỘNG:
 *   1. Nhận { id, data } làm variables (data là Partial<TUpdate>)
 *   2. Gọi apiService.patch(id, data) - HTTP PATCH
 *   3. Sau khi thành công, invalidate cache tương tự useApiUpdate
 * 
 * THAM SỐ:
 *   - apiService: ApiService instance
 *   - entity: Tên entity
 *   - invalidateQueries: Mảng query keys cần invalidate thêm (optional)
 *   - options: Options từ TanStack Query
 * 
 * VARIABLES FORMAT:
 *   { id: string | number, data: Partial<TUpdate> }
 *   - id: ID của item cần update
 *   - data: Object chỉ chứa các fields muốn update (partial)
 * 
 * KHÁC BIỆT VỚI useApiUpdate:
 *   - useApiUpdate: Yêu cầu gửi toàn bộ object (TUpdate)
 *   - useApiPatch: Chỉ cần gửi fields thay đổi (Partial<TUpdate>)
 *   - PATCH nhẹ hơn, tiết kiệm bandwidth
 * 
 * RETURN VALUES (từ useMutation):
 *   - mutate: ({ id, data }, options?) => void
 *   - mutateAsync: ({ id, data }) => Promise<TData>
 *   - isPending, isError, error, isSuccess, data, reset
 * 
 * CACHE INVALIDATION:
 *   - Tương tự useApiUpdate: invalidate lists và detail query
 * 
 * VÍ DỤ SỬ DỤNG:
 * 
 * // Update chỉ một field
 * const updateProductStock = useApiPatch<Product, UpdateProductDto>({
 *   apiService: productApiService,
 *   entity: 'products',
 * });
 * 
 * // Chỉ update stock, không cần gửi name, description, price
 * updateProductStock.mutate({
 *   id: 123,
 *   data: { stock: 50 }, // Chỉ gửi field stock
 * });
 * 
 * // Update nhiều fields nhưng không phải tất cả
 * updateProductStock.mutate({
 *   id: 123,
 *   data: {
 *     price: 99.99,
 *     stock: 100,
 *     // Không cần gửi name, description
 *   },
 * });
 * 
 * // Sử dụng trong component - Toggle status
 * function ToggleProductStatus({ product }: { product: Product }) {
 *   const patchProduct = useApiPatch<Product, UpdateProductDto>({
 *     apiService: productApiService,
 *     entity: 'products',
 *   });
 * 
 *   const handleToggle = () => {
 *     patchProduct.mutate({
 *       id: product.id,
 *       data: { isActive: !product.isActive }, // Chỉ update isActive
 *     });
 *   };
 * 
 *   return (
 *     <button onClick={handleToggle} disabled={patchProduct.isPending}>
 *       {product.isActive ? 'Deactivate' : 'Activate'}
 *     </button>
 *   );
 * }
 * 
 * // Với optimistic update
 * const patchProduct = useApiPatch<Product, UpdateProductDto>({
 *   apiService: productApiService,
 *   entity: 'products',
 *   options: {
 *     onMutate: async ({ id, data }) => {
 *       await queryClient.cancelQueries({ queryKey: ['products', 'detail', id] });
 *       const previous = queryClient.getQueryData(['products', 'detail', id]);
 *       queryClient.setQueryData(['products', 'detail', id], (old: Product) => ({
 *         ...old,
 *         ...data, // Merge với data cũ
 *       }));
 *       return { previous };
 *     },
 *   },
 * });
 * 
 * LƯU Ý:
 *   - PATCH method chỉ cần gửi fields thay đổi, tiết kiệm bandwidth
 *   - Partial<TUpdate> cho phép gửi một phần object
 *   - Phù hợp cho các trường hợp: toggle status, update một field, bulk update một số fields
 *   - Nếu cần update toàn bộ object, dùng useApiUpdate (PUT) thay vì useApiPatch
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
 * useApiDelete - Hook để xóa một item (DELETE)
 * 
 * MỤC ĐÍCH:
 *   - Xóa một item khỏi database
 *   - Tự động xóa item khỏi cache sau khi xóa thành công
 *   - Invalidate list queries để cập nhật danh sách
 * 
 * CÁCH HOẠT ĐỘNG:
 *   1. Nhận id làm variable (string | number)
 *   2. Gọi apiService.delete(id) - HTTP DELETE
 *   3. Sau khi thành công:
 *      - Invalidate tất cả list queries (để refetch danh sách mới)
 *      - Remove detail query của item vừa xóa khỏi cache (không còn tồn tại)
 *      - Invalidate các queries bổ sung
 * 
 * THAM SỐ:
 *   - apiService: ApiService instance
 *   - entity: Tên entity
 *   - invalidateQueries: Mảng query keys cần invalidate thêm (optional)
 *   - options: Options từ TanStack Query
 * 
 * VARIABLES FORMAT:
 *   id: string | number - ID của item cần xóa
 * 
 * RETURN VALUES (từ useMutation):
 *   - mutate: (id: string | number, options?) => void
 *   - mutateAsync: (id: string | number) => Promise<void>
 *   - isPending, isError, error, isSuccess, data (void), reset
 * 
 * CACHE MANAGEMENT:
 *   - Invalidate: ['products', 'list'] - Để refetch danh sách mới
 *   - Remove: ['products', 'detail', id] - Xóa detail query khỏi cache (không còn tồn tại)
 *   - Invalidate thêm: Các queries trong invalidateQueries
 * 
 * KHÁC BIỆT VỚI UPDATE/PATCH:
 *   - Dùng removeQueries thay vì invalidateQueries cho detail query
 *   - Vì item đã bị xóa, không cần refetch detail query
 *   - Chỉ cần remove khỏi cache
 * 
 * VÍ DỤ SỬ DỤNG:
 * 
 * // Basic usage
 * const deleteProduct = useApiDelete({
 *   apiService: productApiService,
 *   entity: 'products',
 * });
 * 
 * // Sử dụng trong component
 * function ProductList() {
 *   const deleteProduct = useApiDelete({
 *     apiService: productApiService,
 *     entity: 'products',
 *     options: {
 *       onSuccess: () => {
 *         toast.success('Product deleted!');
 *       },
 *       onError: (error) => {
 *         toast.error('Failed to delete product');
 *       },
 *     },
 *   });
 * 
 *   const handleDelete = (id: number) => {
 *     if (confirm('Are you sure?')) {
 *       deleteProduct.mutate(id);
 *     }
 *   };
 * 
 *   return (
 *     <div>
 *       {products.map((product) => (
 *         <div key={product.id}>
 *           {product.name}
 *           <button onClick={() => handleDelete(product.id)}>
 *             Delete
 *           </button>
 *         </div>
 *       ))}
 *     </div>
 *   );
 * }
 * 
 * // Với async và error handling
 * const deleteProduct = useApiDelete({
 *   apiService: productApiService,
 *   entity: 'products',
 * });
 * 
 * const handleDelete = async (id: number) => {
 *   try {
 *     await deleteProduct.mutateAsync(id);
 *     console.log('Product deleted successfully');
 *   } catch (error) {
 *     console.error('Error deleting product:', error);
 *   }
 * };
 * 
 * // Với optimistic update (remove khỏi UI ngay lập tức)
 * const deleteProduct = useApiDelete({
 *   apiService: productApiService,
 *   entity: 'products',
 *   options: {
 *     onMutate: async (id) => {
 *       // Cancel outgoing queries
 *       await queryClient.cancelQueries({ queryKey: ['products', 'list'] });
 *       
 *       // Snapshot previous value
 *       const previous = queryClient.getQueryData(['products', 'list']);
 *       
 *       // Optimistically remove from list
 *       queryClient.setQueryData(['products', 'list'], (old: Product[]) =>
 *         old?.filter((p) => p.id !== id) ?? []
 *       );
 *       
 *       return { previous };
 *     },
 *     onError: (err, id, context) => {
 *       // Rollback on error
 *       if (context?.previous) {
 *         queryClient.setQueryData(['products', 'list'], context.previous);
 *       }
 *     },
 *   },
 * });
 * 
 * LƯU Ý:
 *   - removeQueries xóa query khỏi cache hoàn toàn (không refetch)
 *   - invalidateQueries đánh dấu query là stale và sẽ refetch khi cần
 *   - Nên dùng removeQueries cho detail query vì item đã không còn tồn tại
 *   - List queries nên dùng invalidateQueries để refetch danh sách mới
 *   - Có thể dùng optimistic update để cải thiện UX (xóa khỏi UI ngay lập tức)
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
/**
 * useApiCustomQuery - Hook để tạo custom query với queryFn tùy chỉnh
 * 
 * MỤC ĐÍCH:
 *   - Tạo query với logic tùy chỉnh không nằm trong CRUD chuẩn
 *   - Hỗ trợ các endpoints đặc biệt (search, filter phức tạp, aggregation, etc.)
 *   - Vẫn tận dụng cache management và query key structure
 * 
 * CÁCH HOẠT ĐỘNG:
 *   1. Nhận queryFn tùy chỉnh từ user
 *   2. Tạo query key từ entity và queryKey được truyền vào
 *   3. Sử dụng useQuery với queryFn tùy chỉnh
 *   4. Query key sẽ có format: [entity, ...queryKey]
 * 
 * THAM SỐ:
 *   - entity: Tên entity (để nhóm queries)
 *   - queryKey: QueryKey array để tạo unique key (ví dụ: ['category', 123])
 *   - queryFn: Function tùy chỉnh để fetch data
 *   - options: Options từ TanStack Query (loại trừ queryKey và queryFn)
 * 
 * QUERY KEY STRUCTURE:
 *   - Format: [entity, ...queryKey]
 *   - Ví dụ: ['products', 'category', 123]
 *   - Ví dụ: ['products', 'search', 'laptop']
 * 
 * RETURN VALUES (từ useQuery):
 *   - data, isLoading, isPending, isFetching, isError, error, refetch, etc.
 * 
 * VÍ DỤ SỬ DỤNG:
 * 
 * // Get products by category
 * const { data: products } = useApiCustomQuery<Product[]>({
 *   entity: 'products',
 *   queryKey: ['category', categoryId],
 *   queryFn: () => productApiService.custom<Product[]>('get', `category/${categoryId}`),
 *   options: {
 *     enabled: !!categoryId,
 *     staleTime: 1000 * 60 * 5,
 *   },
 * });
 * 
 * // Search products
 * const { data: results } = useApiCustomQuery<Product[]>({
 *   entity: 'products',
 *   queryKey: ['search', searchTerm],
 *   queryFn: () => productApiService.custom<Product[]>('get', 'search', undefined, {
 *     q: searchTerm,
 *   }),
 *   options: {
 *     enabled: searchTerm.length > 2, // Chỉ search khi có ít nhất 3 ký tự
 *   },
 * });
 * 
 * // Get product statistics
 * const { data: stats } = useApiCustomQuery<ProductStats>({
 *   entity: 'products',
 *   queryKey: ['stats'],
 *   queryFn: () => productApiService.custom<ProductStats>('get', 'stats'),
 * });
 * 
 * // Get products with complex filter
 * const { data: products } = useApiCustomQuery<Product[]>({
 *   entity: 'products',
 *   queryKey: ['filtered', { minPrice: 100, maxPrice: 500, category: 'electronics' }],
 *   queryFn: () => productApiService.custom<Product[]>('post', 'filter', {
 *     minPrice: 100,
 *     maxPrice: 500,
 *     category: 'electronics',
 *   }),
 * });
 * 
 * // Wrapper hook cho dễ sử dụng
 * export const useProductsByCategory = (categoryId: number) => {
 *   return useApiCustomQuery<Product[]>({
 *     entity: 'products',
 *     queryKey: ['category', categoryId],
 *     queryFn: () => productApiService.custom<Product[]>('get', `category/${categoryId}`),
 *     options: {
 *       enabled: !!categoryId,
 *     },
 *   });
 * };
 * 
 * LƯU Ý:
 *   - Query key phải unique cho mỗi query khác nhau
 *   - Nên include các parameters quan trọng vào queryKey để cache đúng
 *   - QueryFn có thể gọi bất kỳ API endpoint nào, không chỉ CRUD
 *   - Có thể dùng để wrap các queries phức tạp thành custom hooks
 *   - Vẫn tận dụng được tất cả features của TanStack Query (cache, refetch, etc.)
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
 * useApiCustomMutation - Hook để tạo custom mutation với mutationFn tùy chỉnh
 * 
 * MỤC ĐÍCH:
 *   - Tạo mutation với logic tùy chỉnh không nằm trong CRUD chuẩn
 *   - Hỗ trợ các operations đặc biệt (bulk operations, complex updates, etc.)
 *   - Vẫn tận dụng cache invalidation và mutation lifecycle
 * 
 * CÁCH HOẠT ĐỘNG:
 *   1. Nhận mutationFn tùy chỉnh từ user
 *   2. Sử dụng useMutation với mutationFn tùy chỉnh
 *   3. Sau khi thành công:
 *      - Tự động invalidate tất cả queries của entity
 *      - Invalidate các queries bổ sung nếu có
 *      - Gọi onSuccess callback của user nếu có
 * 
 * THAM SỐ:
 *   - entity: Tên entity (để invalidate queries của entity đó)
 *   - mutationFn: Function tùy chỉnh để thực hiện mutation
 *   - invalidateQueries: Mảng query keys cần invalidate thêm (optional)
 *   - options: Options từ TanStack Query (onSuccess, onError, onMutate, etc.)
 * 
 * RETURN VALUES (từ useMutation):
 *   - mutate: (variables: TVariables, options?) => void
 *   - mutateAsync: (variables: TVariables) => Promise<TData>
 *   - isPending, isError, error, isSuccess, data, reset
 * 
 * CACHE INVALIDATION:
 *   - Tự động invalidate: [entity] - Tất cả queries của entity
 *   - Invalidate thêm: Các queries trong invalidateQueries array
 * 
 * VÍ DỤ SỬ DỤNG:
 * 
 * // Bulk update products
 * const bulkUpdateProducts = useApiCustomMutation<Product[], { ids: number[], stock: number }>({
 *   entity: 'products',
 *   mutationFn: ({ ids, stock }) =>
 *     productApiService.custom<Product[]>('post', 'bulk-update', { ids, stock }),
 *   invalidateQueries: ['products', 'stats'],
 * });
 * 
 * bulkUpdateProducts.mutate({
 *   ids: [1, 2, 3],
 *   stock: 100,
 * });
 * 
 * // Upload product image
 * const uploadImage = useApiCustomMutation<{ url: string }, { productId: number, file: File }>({
 *   entity: 'products',
 *   mutationFn: ({ productId, file }) => {
 *     const formData = new FormData();
 *     formData.append('image', file);
 *     return productApiService.custom<{ url: string }>('post', `${productId}/image`, formData);
 *   },
 *   options: {
 *     onSuccess: (data) => {
 *       toast.success('Image uploaded!');
 *     },
 *   },
 * });
 * 
 * // Approve product (custom business logic)
 * const approveProduct = useApiCustomMutation<Product, { id: number, reason: string }>({
 *   entity: 'products',
 *   mutationFn: ({ id, reason }) =>
 *     productApiService.custom<Product>('post', `${id}/approve`, { reason }),
 *   invalidateQueries: ['products', 'pending-products'],
 * });
 * 
 * // Wrapper hook cho dễ sử dụng
 * export const useBulkUpdateProducts = () => {
 *   return useApiCustomMutation<Product[], { ids: number[], stock: number }>({
 *     entity: 'products',
 *     mutationFn: ({ ids, stock }) =>
 *       productApiService.custom<Product[]>('post', 'bulk-update', { ids, stock }),
 *     options: {
 *       onSuccess: () => {
 *         toast.success('Products updated!');
 *       },
 *     },
 *   });
 * };
 * 
 * // Sử dụng trong component
 * function BulkUpdateForm() {
 *   const bulkUpdate = useApiCustomMutation<Product[], { ids: number[], stock: number }>({
 *     entity: 'products',
 *     mutationFn: ({ ids, stock }) =>
 *       productApiService.custom<Product[]>('post', 'bulk-update', { ids, stock }),
 *     invalidateQueries: ['products', 'stats', 'dashboard'],
 *     options: {
 *       onSuccess: () => {
 *         toast.success('Updated successfully!');
 *       },
 *     },
 *   });
 * 
 *   const handleSubmit = (selectedIds: number[], newStock: number) => {
 *     bulkUpdate.mutate({ ids: selectedIds, stock: newStock });
 *   };
 * 
 *   return (
 *     <form>
 *       {/* form fields */}
 *       <button disabled={bulkUpdate.isPending}>
 *         {bulkUpdate.isPending ? 'Updating...' : 'Update'}
 *       </button>
 *     </form>
 *   );
 * }
 * 
 * LƯU Ý:
 *   - mutationFn có thể thực hiện bất kỳ operation nào, không chỉ CRUD
 *   - Tự động invalidate tất cả queries của entity sau khi thành công
 *   - Có thể invalidate thêm các queries liên quan qua invalidateQueries
 *   - Có thể dùng để wrap các mutations phức tạp thành custom hooks
 *   - Vẫn tận dụng được tất cả features của TanStack Query mutations
 *   - Phù hợp cho: bulk operations, file uploads, custom business logic, etc.
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

## 2.5. Hybrid Approach - Feature Extensions

### Mô hình Hybrid Approach

Thay vì dùng BaseApiService trực tiếp, mỗi feature sẽ **extend BaseApiService** và thêm các **custom methods** riêng:

```
src/lib/api/base/
├── BaseApiService.ts          // Base class với CRUD cơ bản
└── index.ts

src/features/products/api/
├── ProductApiService.ts       // Extends BaseApiService
└── index.ts
```

### Ví dụ: ProductApiService

```typescript
// src/features/products/api/ProductApiService.ts
import { BaseApiService } from '../../../lib/api/base/BaseApiService';
import axiosClient from '../../../lib/axios';
import { API_CONFIG } from '../../../config/api';
import type { ProductEntity } from '../types/entity';
import type { CreateProductRequest, UpdateProductRequest } from '../types/api';
import type { PagedList, PagedRequest } from '../../../lib/axios';

/**
 * ProductApiService - Extends BaseApiService với custom methods
 * 
 * Kế thừa tất cả CRUD methods từ BaseApiService:
 * - getAll(), getPaginated(), getById()
 * - create(), update(), patch(), delete()
 * - custom()
 * 
 * Thêm custom methods riêng cho Products:
 * - searchByBarcode()
 * - getProductsByCategory()
 * - getProductsBySupplier()
 */
export class ProductApiService extends BaseApiService<
  ProductEntity,
  CreateProductRequest,
  UpdateProductRequest
> {
  constructor() {
    super({
      endpoint: API_CONFIG.ENDPOINTS.ADMIN.PRODUCTS,
      axiosInstance: axiosClient,
    });
  }

  /**
   * Tìm kiếm sản phẩm theo barcode
   */
  async searchByBarcode(barcode: string): Promise<ProductEntity[]> {
    return this.custom<ProductEntity[]>('get', this.endpoint, undefined, {
      search: barcode,
      pageSize: 10,
    });
  }

  /**
   * Lấy sản phẩm theo danh mục
   */
  async getProductsByCategory(
    categoryId: number,
    params?: Omit<PagedRequest, 'categoryId'>
  ): Promise<PagedList<ProductEntity>> {
    return this.getPaginated({
      ...params,
      categoryId, // Thêm categoryId vào params
    } as PagedRequest);
  }

  /**
   * Lấy sản phẩm theo nhà cung cấp
   */
  async getProductsBySupplier(
    supplierId: number,
    params?: Omit<PagedRequest, 'supplierId'>
  ): Promise<PagedList<ProductEntity>> {
    return this.getPaginated({
      ...params,
      supplierId, // Thêm supplierId vào params
    } as PagedRequest);
  }
}

// Export singleton instance
export const productApiService = new ProductApiService();
```

### Lợi ích của Hybrid Approach

1. **Tái sử dụng code CRUD**: Không cần viết lại getAll, getById, create, update, delete
2. **Linh hoạt cho custom methods**: Mỗi feature có thể thêm methods riêng
3. **Phù hợp với Feature-Sliced Design**: Logic vẫn nằm trong feature folder
4. **Xử lý ApiResponse<T> tự động**: BaseApiService đã xử lý unwrap và error checking
5. **Type-safe**: TypeScript generics đảm bảo type safety

### So sánh với cách tiếp cận cũ

**❌ Cách cũ (lặp lại code)**:
```typescript
// Mỗi feature phải viết lại tất cả CRUD
export const productApi = {
  getProducts: async (params) => {
    const response = await axiosClient.get(...);
    if (response.isError) throw new Error(...);
    return response.data;
  },
  getProductById: async (id) => { /* lặp lại code */ },
  createProduct: async (data) => { /* lặp lại code */ },
  // ... 50+ dòng code lặp lại
};
```

**✅ Hybrid Approach (tái sử dụng)**:
```typescript
// Chỉ cần extend và thêm custom methods
export class ProductApiService extends BaseApiService {
  // Tự động có: getAll, getPaginated, getById, create, update, delete
  
  // Chỉ cần thêm custom methods
  async searchByBarcode(barcode: string) { /* custom logic */ }
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

### 3.2. Product API Service (`src/features/products/api/ProductApiService.ts`)

```typescript
import { BaseApiService } from '../../../lib/api/base/BaseApiService';
import axiosClient from '../../../lib/axios';
import { API_CONFIG } from '../../../config/api';
import type { ProductEntity } from '../types/entity';
import type { CreateProductRequest, UpdateProductRequest } from '../types/api';
import type { PagedList, PagedRequest } from '../../../lib/axios';

/**
 * ProductApiService - Extends BaseApiService với custom methods
 */
export class ProductApiService extends BaseApiService<
  ProductEntity,
  CreateProductRequest,
  UpdateProductRequest
> {
  constructor() {
    super({
      endpoint: API_CONFIG.ENDPOINTS.ADMIN.PRODUCTS,
      axiosInstance: axiosClient,
    });
  }

  /**
   * Tìm kiếm sản phẩm theo barcode
   */
  async searchByBarcode(barcode: string): Promise<ProductEntity[]> {
    const response = await this.axios.get<ApiResponse<PagedList<ProductEntity>>>(
      this.endpoint,
      { params: { search: barcode, pageSize: 10 } }
    );
    const pagedData = this.unwrapResponse(response);
    return pagedData.items;
  }

  /**
   * Lấy sản phẩm theo danh mục
   */
  async getProductsByCategory(
    categoryId: number,
    params?: Omit<PagedRequest, 'categoryId'>
  ): Promise<PagedList<ProductEntity>> {
    return this.getPaginated({
      ...params,
      categoryId,
    } as PagedRequest);
  }

  /**
   * Lấy sản phẩm theo nhà cung cấp
   */
  async getProductsBySupplier(
    supplierId: number,
    params?: Omit<PagedRequest, 'supplierId'>
  ): Promise<PagedList<ProductEntity>> {
    return this.getPaginated({
      ...params,
      supplierId,
    } as PagedRequest);
  }
}

// Export singleton instance
export const productApiService = new ProductApiService();
```

### 3.3. Product Hooks (`src/hooks/useProducts.ts`)

```typescript
import { 
  useApiList, 
  useApiDetail, 
  useApiCreate, 
  useApiUpdate, 
  useApiDelete,
} from './useApi';
import { useQuery } from '@tanstack/react-query';
import { productApiService } from '../api/ProductApiService';
import type { ProductEntity } from '../types/entity';
import type { CreateProductRequest, UpdateProductRequest } from '../types/api';
import type { PagedList, PagedRequest } from '../../../lib/axios';

const ENTITY = 'products';

// GET all products (không phân trang)
export const useProducts = (params?: { search?: string; category?: string }) => {
  return useApiList<ProductEntity>({
    apiService: productApiService,
    entity: ENTITY,
    params,
    options: {
      staleTime: 1000 * 60 * 5, // 5 minutes
    },
  });
};

// GET products với phân trang
export const useProductsPaginated = (params?: PagedRequest) => {
  return useQuery<PagedList<ProductEntity>>({
    queryKey: ['products', 'paginated', params],
    queryFn: () => productApiService.getPaginated(params),
    staleTime: 1000 * 60 * 5,
  });
};

// GET product by ID
export const useProduct = (id: number) => {
  return useApiDetail<ProductEntity>({
    apiService: productApiService,
    entity: ENTITY,
    id,
  });
};

// CREATE product
export const useCreateProduct = () => {
  return useApiCreate<ProductEntity, CreateProductRequest>({
    apiService: productApiService,
    entity: ENTITY,
    options: {
      onSuccess: () => {
        // UI feedback for success is handled here
        // For example: toast.success('Product created successfully');
        console.log('Product created successfully');
      },
      onError: (error) => {
        // Centralized error handling with UI feedback
        try {
          handleApiError(error);
        } catch (finalError: any) {
          // For example: toast.error(`Error: ${finalError.message}`);
          console.error(`Error: ${finalError.message}`);
        }
      },
    },
  });
};

// UPDATE product
export const useUpdateProduct = () => {
  return useApiUpdate<ProductEntity, UpdateProductRequest>({
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
        // For example: toast.success('Product deleted successfully');
        console.log('Product deleted successfully');
      },
      onError: (error) => {
        try {
          handleApiError(error);
        } catch (finalError: any) {
          // For example: toast.error(`Error: ${finalError.message}`);
          console.error(`Error: ${finalError.message}`);
        }
      },
    },
  });
};

// Custom: Get products by category (sử dụng custom method)
export const useProductsByCategory = (
  categoryId: number,
  params?: Omit<PagedRequest, 'categoryId'>
) => {
  return useQuery<PagedList<ProductEntity>>({
    queryKey: ['products', 'category', categoryId, params],
    queryFn: () => productApiService.getProductsByCategory(categoryId, params),
    enabled: !!categoryId,
  });
};

// Custom: Search by barcode
export const useProductsByBarcode = (barcode: string) => {
  return useQuery<ProductEntity[]>({
    queryKey: ['products', 'barcode', barcode],
    queryFn: () => productApiService.searchByBarcode(barcode),
    enabled: barcode.length > 0,
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

  const handleDelete = (id: number) => {
    if (!confirm('Bạn có chắc muốn xóa?')) return;

    // Call mutate. onSuccess and onError are handled by the hook's options.
    deleteProduct.mutate(id);
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

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    
    // Call mutate. onSuccess and onError are handled by the hook's options.
    // We can pass component-specific logic here if needed.
    createProduct.mutate(formData, {
      onSuccess: () => {
        navigate({ to: '/products' });
      },
    });
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

### 4.1. User Types (`src/features/users/types/entity.ts`)

```typescript
export interface UserEntity {
  id: number;
  username: string;
  fullName: string;
  role: 0 | 1; // 0 = Admin, 1 = Staff
  createdAt: string;
  updatedAt?: string;
}
```

### 4.2. User API Service (`src/features/users/api/UserApiService.ts`)

```typescript
import { BaseApiService } from '../../../lib/api/base/BaseApiService';
import axiosClient from '../../../lib/axios';
import { API_CONFIG } from '../../../config/api';
import type { UserEntity } from '../types/entity';
import type { CreateUserRequest, UpdateUserRequest } from '../types/api';
import type { PagedList, PagedRequest } from '../../../lib/axios';

/**
 * UserApiService - Extends BaseApiService với custom methods
 */
export class UserApiService extends BaseApiService<
  UserEntity,
  CreateUserRequest,
  UpdateUserRequest
> {
  constructor() {
    super({
      endpoint: API_CONFIG.ENDPOINTS.ADMIN.USERS,
      axiosInstance: axiosClient,
    });
  }

  /**
   * Lấy danh sách Staff (dùng cho dropdown)
   */
  async getStaffUsers(): Promise<UserEntity[]> {
    const pagedData = await this.getPaginated({
      page: 1,
      pageSize: 1000,
    });
    // Filter staff users ở client side
    return pagedData.items.filter(user => user.role === API_CONFIG.USER_ROLES.STAFF);
  }

  /**
   * Kiểm tra username có tồn tại không
   */
  async checkUsernameExists(username: string): Promise<boolean> {
    try {
      const pagedData = await this.getPaginated({
        search: username,
        pageSize: 1,
      });
      return pagedData.items.some(
        user => user.username.toLowerCase() === username.toLowerCase()
      );
    } catch {
      return false;
    }
  }
}

// Export singleton instance
export const userApiService = new UserApiService();
```

### 4.3. User Hooks (`src/features/users/hooks/useUsers.ts`)

```typescript
import { 
  useApiList, 
  useApiDetail, 
  useApiCreate, 
  useApiUpdate, 
  useApiDelete 
} from '../../../hooks/useApi';
import { useQuery } from '@tanstack/react-query';
import { userApiService } from '../api/UserApiService';
import type { UserEntity } from '../types/entity';
import type { CreateUserRequest, UpdateUserRequest } from '../types/api';
import type { PagedList, PagedRequest } from '../../../lib/axios';

const ENTITY = 'users';

// GET users với phân trang
export const useUsers = (params?: PagedRequest) => {
  return useQuery<PagedList<UserEntity>>({
    queryKey: ['users', 'paginated', params],
    queryFn: () => userApiService.getPaginated(params),
  });
};

export const useUser = (id: number) => {
  return useApiDetail<UserEntity>({
    apiService: userApiService,
    entity: ENTITY,
    id,
  });
};

export const useCreateUser = () => {
  return useApiCreate<UserEntity, CreateUserRequest>({
    apiService: userApiService,
    entity: ENTITY,
  });
};

export const useUpdateUser = () => {
  return useApiUpdate<UserEntity, UpdateUserRequest>({
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

// Custom: Get staff users
export const useStaffUsers = () => {
  return useQuery<UserEntity[]>({
    queryKey: ['users', 'staff'],
    queryFn: () => userApiService.getStaffUsers(),
  });
};

// Custom: Check username exists
export const useCheckUsernameExists = (username: string) => {
  return useQuery<boolean>({
    queryKey: ['users', 'check-username', username],
    queryFn: () => userApiService.checkUsernameExists(username),
    enabled: username.length > 0,
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

### 5.2. Pagination Support với PagedList<T>

BaseApiService đã có sẵn method `getPaginated()` sử dụng `PagedList<T>` từ `src/lib/axios.ts`:

```typescript
// PagedList<T> đã được định nghĩa trong src/lib/axios.ts
export interface PagedList<T> {
  page: number;
  pageSize: number;
  totalCount: number;
  totalPages: number;
  hasPrevious: boolean;
  hasNext: boolean;
  items: T[];
}

// BaseApiService.getPaginated() đã xử lý ApiResponse<PagedList<T>>
// ⚠️ LƯU Ý: Backend API sử dụng PascalCase cho query parameters
// Nếu params là camelCase (page, pageSize), cần convert sang PascalCase (Page, PageSize) trước khi gửi
async getPaginated(params?: PagedRequest): Promise<PagedList<TData>> {
  // TODO: Implement helper function để convert camelCase → PascalCase
  // const pascalParams = toPascalCaseParams(params || {});
  const response = await this.axios.get<ApiResponse<PagedList<TData>>>(
    this.endpoint,
    { params } // ⚠️ Cần đảm bảo params đã được convert sang PascalCase
  );
  return this.unwrapResponse(response);
}
```

**Hook cho pagination**:

```typescript
import { useQuery } from '@tanstack/react-query';
import type { PagedList, PagedRequest } from '../../../lib/axios';

export function useApiPaginated<TData = any>({
  apiService,
  entity,
  params,
  options,
}: {
  apiService: BaseApiService<TData>;
  entity: string;
  params?: PagedRequest;
  options?: Omit<UseQueryOptions<PagedList<TData>>, 'queryKey' | 'queryFn'>;
}) {
  const queryKeys = createQueryKeys(entity);

  return useQuery<PagedList<TData>>({
    queryKey: [...queryKeys.lists(), 'paginated', params],
    queryFn: () => apiService.getPaginated(params),
    keepPreviousData: true, // Giữ data cũ khi đang fetch trang mới
    ...options,
  });
}
```

**Ví dụ sử dụng**:

```typescript
// Trong component
function ProductListPage() {
  const [pagination, setPagination] = useState<PagedRequest>({
    page: 1,
    pageSize: 20,
    search: '',
    sortBy: 'ProductName',
    sortDesc: false,
  });

  const { data, isLoading } = useApiPaginated<ProductEntity>({
    apiService: productApiService,
    entity: 'products',
    params: pagination,
  });

  return (
    <div>
      <Table
        dataSource={data?.items || []}
        pagination={{
          current: data?.page || 1,
          pageSize: data?.pageSize || 20,
          total: data?.totalCount || 0,
          onChange: (page, pageSize) => {
            setPagination(prev => ({ ...prev, page, pageSize }));
          },
        }}
      />
    </div>
  );
}
```

## Tổng kết

Hệ thống **Hybrid Approach** này cung cấp:

✅ **BaseApiService** - CRUD cơ bản với xử lý `ApiResponse<T>` và `PagedList<T>` tự động
✅ **Feature Extensions** - Mỗi feature extend BaseApiService và thêm custom methods
✅ **Universal hooks** - Có thể dùng cho bất kỳ entity nào
✅ **Type-safe** - TypeScript generics đảm bảo type safety
✅ **Reusable** - Tái sử dụng code CRUD, không lặp lại
✅ **Flexible** - Linh hoạt cho custom methods và endpoints đặc biệt
✅ **Automatic cache management** - Invalidation tự động với TanStack Query
✅ **Easy to extend** - Dễ dàng thêm features như pagination, optimistic updates
✅ **Clean separation** - API logic tách biệt khỏi components, phù hợp với Feature-Sliced Design
✅ **Compatible** - Tương thích với cấu trúc hiện tại (ApiResponse wrapper, Axios instance)

### Quy trình sử dụng cho entity mới:

1. **Tạo types** (`src/features/[entity]/types/entity.ts` và `api.ts`)
2. **Tạo [Entity]ApiService** extend BaseApiService (`src/features/[entity]/api/[Entity]ApiService.ts`)
3. **Thêm custom methods** nếu cần (searchByBarcode, getByCategory, etc.)
4. **Tạo hooks wrapper** (optional, có thể dùng trực tiếp `useApiList`, `useApiCreate`, etc.)
5. **Sử dụng trong components**

### Lợi ích của Hybrid Approach:

- **DRY (Don't Repeat Yourself)**: Code CRUD chỉ viết một lần trong BaseApiService
- **Flexibility**: Mỗi feature có thể thêm custom methods riêng
- **Maintainability**: Dễ maintain và update vì logic tập trung
- **Type Safety**: TypeScript generics đảm bảo type safety
- **Feature-Sliced Design**: Phù hợp với kiến trúc FSD của project
