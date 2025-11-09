# 📘 HƯỚNG DẪN PHÂN TRANG CHO FRONTEND DEVELOPERS

> **Tài liệu tham khảo đầy đủ về cơ chế phân trang, search, và sorting của Backend API**
>
> **Phiên bản:** 2.0
> **Ngày cập nhật:** 2025-01-09
> **Backend Framework:** ASP.NET Core 9.0

---

## 📋 MỤC LỤC

1. [Tổng Quan Cơ Chế Phân Trang](#1-tổng-quan-cơ-chế-phân-trang)
2. [Chi Tiết Từng Endpoint](#2-chi-tiết-từng-endpoint)
3. [TypeScript Interfaces](#3-typescript-interfaces)
4. [Hướng Dẫn Implementation](#4-hướng-dẫn-implementation)
5. [Error Handling](#5-error-handling)
6. [Best Practices](#6-best-practices)
7. [Quick Reference Table](#7-quick-reference-table)

---

## 1. TỔNG QUAN CƠ CHẾ PHÂN TRANG

### 1.1. Hai Loại Request Models

Backend sử dụng **2 loại request models** cho phân trang:

#### **A. BasePagedRequest** (Phân trang cơ bản)

**Sử dụng cho:** Report endpoints (TopProducts, TopCustomers)

**Properties:**
- `Page` (int): Số trang hiện tại (bắt đầu từ 1)
- `PageSize` (int): Số items trên mỗi trang (tối đa 100)

**Đặc điểm:**
- ✅ KHÔNG có `Search`, `SortBy`, `SortDesc`
- ✅ Sorting được hard-code theo business logic
- ✅ Đơn giản, chỉ dùng cho pagination

#### **B. PagedRequest** (Phân trang đầy đủ)

**Sử dụng cho:** Tất cả endpoints quản lý (Products, Categories, Users, v.v.)

**Properties:**
- `Page` (int): Số trang hiện tại (bắt đầu từ 1)
- `PageSize` (int): Số items trên mỗi trang (tối đa 100)
- `Search` (string, optional): Từ khóa tìm kiếm
- `SortBy` (string): Tên property để sắp xếp (mặc định: "Id")
- `SortDesc` (bool): Sắp xếp giảm dần (true) hoặc tăng dần (false)

**Đặc điểm:**
- ✅ Hỗ trợ full-text search
- ✅ Dynamic sorting theo bất kỳ property nào
- ✅ Có thể thêm filters đặc thù (CategoryId, Status, v.v.)

### 1.2. Response Structure (PagedList)

**Tất cả endpoints phân trang** đều trả về cấu trúc `ApiResponse<PagedList<T>>`:

```typescript
interface ApiResponse<T> {
  isError: boolean;
  message: string;
  data: T | null;
  timestamp: string;
}

interface PagedList<T> {
  page: number;           // Trang hiện tại
  pageSize: number;       // Số items trên trang
  totalCount: number;     // Tổng số items
  totalPages: number;     // Tổng số trang
  hasPrevious: boolean;   // Có trang trước không?
  hasNext: boolean;       // Có trang sau không?
  items: T[];            // Danh sách items
}
```

### 1.3. Validation Rules (Chung)

**Áp dụng cho TẤT CẢ endpoints:**

| Parameter | Type | Required | Min | Max | Default | Validation |
|-----------|------|----------|-----|-----|---------|------------|
| `Page` | int | ✅ | 1 | ∞ | 1 | Phải > 0 |
| `PageSize` | int | ✅ | 1 | 100 | 10 | Phải > 0 và ≤ 100 |
| `Search` | string | ❌ | - | 255 chars | null | Tối đa 255 ký tự |
| `SortBy` | string | ❌ | - | - | "Id" | Phải là property hợp lệ |
| `SortDesc` | bool | ❌ | - | - | true | - |

**⚠️ Lưu ý quan trọng:**
- Nếu `PageSize > 100`, backend tự động giới hạn về 100
- `SortBy` phải là tên property **chính xác** (case-insensitive)
- Report endpoints KHÔNG chấp nhận `Search`, `SortBy`, `SortDesc`

---

## 2. CHI TIẾT TỪNG ENDPOINT

### 2.1. 📦 PRODUCTS

**URL:** `GET /api/admin/products`
**Authorization:** Admin only
**Request Model:** `ProductSearchRequest extends PagedRequest`

#### Request Parameters

| Parameter | Type | Required | Description | Validation |
|-----------|------|----------|-------------|------------|
| `Page` | int | ✅ | Số trang | > 0 |
| `PageSize` | int | ✅ | Số items/trang | 1-100 |
| `Search` | string | ❌ | Tìm theo tên, barcode | Max 255 chars |
| `SortBy` | string | ❌ | Property để sort | Xem bảng dưới |
| `SortDesc` | bool | ❌ | Giảm dần? | - |
| `CategoryId` | int | ❌ | Lọc theo category | > 0 |
| `SupplierId` | int | ❌ | Lọc theo supplier | > 0 |
| `MinPrice` | decimal | ❌ | Giá tối thiểu | ≥ 0 |
| `MaxPrice` | decimal | ❌ | Giá tối đa | ≥ 0, ≥ MinPrice |

#### Allowed SortBy Values

```
"Id", "ProductName", "Barcode", "Price", "Unit",
"CategoryId", "SupplierId", "CreatedAt", "UpdatedAt", "DeletedAt"
```

#### Response DTO

```typescript
interface ProductListDto {
  id: number;
  productName: string;
  barcode: string;
  price: number;
  unit: string;
  categoryName: string;
  supplierName: string;
  inventoryQuantity: number;
}
```

#### Example Request

```typescript
// Axios/Fetch
const response = await axios.get('/api/admin/products', {
  params: {
    Page: 1,
    PageSize: 20,
    Search: 'coca',
    SortBy: 'Price',
    SortDesc: false,
    CategoryId: 5,
    MinPrice: 10000,
    MaxPrice: 50000
  }
});
```

---

### 2.2. 📂 CATEGORIES

**URL:** `GET /api/admin/categories`
**Authorization:** Admin only
**Request Model:** `CategorySearchRequest extends PagedRequest`

#### Request Parameters

| Parameter | Type | Required | Description | Validation |
|-----------|------|----------|-------------|------------|
| `Page` | int | ✅ | Số trang | > 0 |
| `PageSize` | int | ✅ | Số items/trang | 1-100 |
| `Search` | string | ❌ | Tìm theo tên category | Max 255 chars |
| `SortBy` | string | ❌ | Property để sort | Xem bảng dưới |
| `SortDesc` | bool | ❌ | Giảm dần? | - |
| `MinProductCount` | int | ❌ | Số sản phẩm tối thiểu | ≥ 0 |
| `MaxProductCount` | int | ❌ | Số sản phẩm tối đa | ≥ 0, ≥ MinProductCount |
| `CreatedAfter` | DateTime | ❌ | Tạo sau ngày | ≤ Now |
| `CreatedBefore` | DateTime | ❌ | Tạo trước ngày | ≤ Now, ≥ CreatedAfter |

#### Allowed SortBy Values

```
"Id", "CategoryName", "CreatedAt", "UpdatedAt", "DeletedAt"
```

#### Response DTO

```typescript
interface CategoryResponseDto {
  id: number;
  categoryName: string;
  productCount: number;
}
```

#### Example Request

```typescript
const response = await axios.get('/api/admin/categories', {
  params: {
    Page: 1,
    PageSize: 10,
    Search: 'Nước',
    SortBy: 'CategoryName',
    SortDesc: false,
    MinProductCount: 5
  }
});
```

---

### 2.3. 👥 CUSTOMERS

**URL:** `GET /api/admin/customers`
**Authorization:** Admin & Staff
**Request Model:** `CustomerSearchRequest extends PagedRequest`

#### Request Parameters

| Parameter | Type | Required | Description | Validation |
|-----------|------|----------|-------------|------------|
| `Page` | int | ✅ | Số trang | > 0 |
| `PageSize` | int | ✅ | Số items/trang | 1-100 |
| `Search` | string | ❌ | Tìm theo tên, SĐT, email | Max 255 chars |
| `SortBy` | string | ❌ | Property để sort | Xem bảng dưới |
| `SortDesc` | bool | ❌ | Giảm dần? | - |

#### Allowed SortBy Values

```
"Id", "Name", "Phone", "Email", "Address", "CreatedAt", "UpdatedAt", "DeletedAt"
```

#### Response DTO

```typescript
interface CustomerListDto {
  id: number;
  name: string;
  phone: string;
  email: string | null;
  lastOrderDate: string | null; // ISO 8601 DateTime
}
```

#### Example Request

```typescript
const response = await axios.get('/api/admin/customers', {
  params: {
    Page: 1,
    PageSize: 20,
    Search: 'Nguyễn',
    SortBy: 'Name',
    SortDesc: false
  }
});
```

---

### 2.4. 🏭 SUPPLIERS

**URL:** `GET /api/admin/suppliers`
**Authorization:** Admin only
**Request Model:** `SupplierSearchRequest extends PagedRequest`

#### Request Parameters

| Parameter | Type | Required | Description | Validation |
|-----------|------|----------|-------------|------------|
| `Page` | int | ✅ | Số trang | > 0 |
| `PageSize` | int | ✅ | Số items/trang | 1-100 |
| `Search` | string | ❌ | Tìm theo tên, SĐT, email | Max 255 chars |
| `SortBy` | string | ❌ | Property để sort | Xem bảng dưới |
| `SortDesc` | bool | ❌ | Giảm dần? | - |

#### Allowed SortBy Values

```
"Id", "Name", "Phone", "Email", "Address", "CreatedAt", "UpdatedAt", "DeletedAt"
```

#### Response DTO

```typescript
interface SupplierResponseDto {
  id: number;
  name: string;
  phone: string;
  email: string | null;
  address: string | null;
  productCount: number;
}
```

#### Example Request

```typescript
const response = await axios.get('/api/admin/suppliers', {
  params: {
    Page: 1,
    PageSize: 15,
    Search: 'Coca',
    SortBy: 'ProductCount',
    SortDesc: true
  }
});
```




---

### 2.5. 📋 ORDERS

**URL:** `GET /api/admin/orders`
**Authorization:** Admin & Staff
**Request Model:** `OrderSearchRequest extends PagedRequest`

#### Request Parameters

| Parameter | Type | Required | Description | Validation |
|-----------|------|----------|-------------|------------|
| `Page` | int | ✅ | Số trang | > 0 |
| `PageSize` | int | ✅ | Số items/trang | 1-100 |
| `Search` | string | ❌ | Tìm theo tên KH, staff | Max 255 chars |
| `SortBy` | string | ❌ | Property để sort | Xem bảng dưới |
| `SortDesc` | bool | ❌ | Giảm dần? | - |
| `Status` | string | ❌ | Lọc theo trạng thái | "Pending", "Paid", "Cancelled" |
| `CustomerId` | int | ❌ | Lọc theo khách hàng | > 0 |
| `UserId` | int | ❌ | Lọc theo nhân viên | > 0 |
| `StartDate` | DateTime | ❌ | Từ ngày | ≤ Now |
| `EndDate` | DateTime | ❌ | Đến ngày | ≤ Now, ≥ StartDate |

#### Allowed SortBy Values

```
"Id", "CustomerId", "UserId", "PromoId", "OrderDate", "Status",
"TotalAmount", "DiscountAmount", "CreatedAt", "UpdatedAt", "DeletedAt"
```

#### Response DTO

```typescript
interface OrderListDto {
  id: number;
  orderDate: string; // ISO 8601 DateTime
  customerName: string;
  staffName: string;
  status: string; // "Pending" | "Paid" | "Cancelled"
  totalAmount: number;
  finalAmount: number;
}
```

#### Example Request

```typescript
const response = await axios.get('/api/admin/orders', {
  params: {
    Page: 1,
    PageSize: 20,
    Status: 'Paid',
    StartDate: '2024-01-01',
    EndDate: '2024-12-31',
    SortBy: 'OrderDate',
    SortDesc: true
  }
});
```

---

### 2.6. 🎁 PROMOTIONS

**URL:** `GET /api/admin/promotions`
**Authorization:** Admin only
**Request Model:** `PromotionSearchRequest extends PagedRequest`

#### Request Parameters

| Parameter | Type | Required | Description | Validation |
|-----------|------|----------|-------------|------------|
| `Page` | int | ✅ | Số trang | > 0 |
| `PageSize` | int | ✅ | Số items/trang | 1-100 |
| `Search` | string | ❌ | Tìm theo mã, mô tả | Max 255 chars |
| `SortBy` | string | ❌ | Property để sort | Xem bảng dưới |
| `SortDesc` | bool | ❌ | Giảm dần? | - |

#### Allowed SortBy Values

```
"Id", "PromoCode", "Description", "DiscountType", "DiscountValue",
"StartDate", "EndDate", "MinOrderAmount", "UsageLimit", "UsedCount",
"Status", "CreatedAt", "UpdatedAt", "DeletedAt"
```

#### Response DTO

```typescript
interface PromotionListDto {
  id: number;
  promoCode: string;
  description: string | null;
  discountType: string; // "percent" | "fixed"
  discountValue: number;
  startDate: string; // ISO 8601 DateTime
  endDate: string; // ISO 8601 DateTime
  status: string; // "active" | "inactive"
  usedCount: number;
  remainingUsage: number;
}
```

#### Example Request

```typescript
const response = await axios.get('/api/admin/promotions', {
  params: {
    Page: 1,
    PageSize: 10,
    Search: 'SALE',
    SortBy: 'StartDate',
    SortDesc: true
  }
});
```

---

### 2.7. 👤 USERS

**URL:** `GET /api/admin/users`
**Authorization:** Admin only
**Request Model:** `UserSearchRequest extends PagedRequest`

#### Request Parameters

| Parameter | Type | Required | Description | Validation |
|-----------|------|----------|-------------|------------|
| `Page` | int | ✅ | Số trang | > 0 |
| `PageSize` | int | ✅ | Số items/trang | 1-100 |
| `Search` | string | ❌ | Tìm theo username, tên | Max 255 chars |
| `SortBy` | string | ❌ | Property để sort | Xem bảng dưới |
| `SortDesc` | bool | ❌ | Giảm dần? | - |
| `Role` | int | ❌ | Lọc theo vai trò | 0 (Admin) hoặc 1 (Staff) |

#### Allowed SortBy Values

```
"Id", "Username", "FullName", "Role", "CreatedAt", "UpdatedAt", "DeletedAt"
```

#### Response DTO

```typescript
interface UserResponseDto {
  id: number;
  username: string;
  fullName: string;
  role: number; // 0: Admin, 1: Staff
  createdAt: string; // ISO 8601 DateTime
}
```

#### Example Request

```typescript
const response = await axios.get('/api/admin/users', {
  params: {
    Page: 1,
    PageSize: 20,
    Role: 1, // Chỉ lấy Staff
    SortBy: 'FullName',
    SortDesc: false
  }
});
```


---

### 2.8. 📦 INVENTORY

**URL:** `GET /api/admin/inventory`
**Authorization:** Admin & Staff
**Request Model:** `InventorySearchRequest extends PagedRequest`

#### Request Parameters

| Parameter | Type | Required | Description | Validation |
|-----------|------|----------|-------------|------------|
| `Page` | int | ✅ | Số trang | > 0 |
| `PageSize` | int | ✅ | Số items/trang | 1-100 |
| `Search` | string | ❌ | Tìm theo tên SP, barcode | Max 255 chars |
| `SortBy` | string | ❌ | Property để sort | Xem bảng dưới |
| `SortDesc` | bool | ❌ | Giảm dần? | - |
| `ProductId` | int | ❌ | Lọc theo sản phẩm | > 0 |
| `MinQuantity` | int | ❌ | Số lượng tối thiểu | ≥ 0 |
| `MaxQuantity` | int | ❌ | Số lượng tối đa | ≥ 0, ≥ MinQuantity |

#### Allowed SortBy Values

```
"Id", "ProductId", "Quantity", "CreatedAt", "UpdatedAt", "DeletedAt"
```

#### Response DTO

```typescript
interface InventoryResponseDto {
  inventoryId: number;
  productId: number;
  productName: string;
  barcode: string;
  quantity: number;
  updatedAt: string; // ISO 8601 DateTime
  status: string; // "in_stock" | "low_stock" | "out_of_stock"
}
```

**Status Logic:**
- `out_of_stock`: quantity = 0
- `low_stock`: quantity < 10
- `in_stock`: quantity ≥ 10

#### Example Request

```typescript
const response = await axios.get('/api/admin/inventory', {
  params: {
    Page: 1,
    PageSize: 20,
    Search: 'Coca',
    MinQuantity: 0,
    MaxQuantity: 10, // Chỉ lấy sản phẩm sắp hết
    SortBy: 'Quantity',
    SortDesc: false
  }
});
```

---

### 2.9. 📊 TOP PRODUCTS (Report)

**URL:** `GET /api/admin/reports/top-products`
**Authorization:** Admin only
**Request Model:** `TopProductsSearchRequest extends BasePagedRequest`

#### Request Parameters

| Parameter | Type | Required | Description | Validation |
|-----------|------|----------|-------------|------------|
| `Page` | int | ✅ | Số trang | > 0 |
| `PageSize` | int | ✅ | Số items/trang | 1-100 |
| `StartDate` | DateTime | ✅ | Từ ngày | ≤ Now |
| `EndDate` | DateTime | ✅ | Đến ngày | ≤ Now, ≥ StartDate |

**⚠️ LƯU Ý QUAN TRỌNG:**
- ❌ KHÔNG hỗ trợ `Search`, `SortBy`, `SortDesc`
- ✅ Sorting FIXED theo `TotalRevenue` (giảm dần)
- ✅ Chỉ có `Page`, `PageSize`, `StartDate`, `EndDate`

#### Response DTO

```typescript
interface TopProductDto {
  productId: number;
  productName: string;
  totalQuantitySold: number;
  totalRevenue: number;
  orderCount: number;
}
```

#### Example Request

```typescript
const response = await axios.get('/api/admin/reports/top-products', {
  params: {
    Page: 1,
    PageSize: 10,
    StartDate: '2024-01-01',
    EndDate: '2024-12-31'
    // ❌ KHÔNG gửi Search, SortBy, SortDesc
  }
});
```

---

### 2.10. 👥 TOP CUSTOMERS (Report)

**URL:** `GET /api/admin/reports/top-customers`
**Authorization:** Admin only
**Request Model:** `TopCustomersSearchRequest extends BasePagedRequest`

#### Request Parameters

| Parameter | Type | Required | Description | Validation |
|-----------|------|----------|-------------|------------|
| `Page` | int | ✅ | Số trang | > 0 |
| `PageSize` | int | ✅ | Số items/trang | 1-100 |
| `StartDate` | DateTime | ✅ | Từ ngày | ≤ Now |
| `EndDate` | DateTime | ✅ | Đến ngày | ≤ Now, ≥ StartDate |

**⚠️ LƯU Ý QUAN TRỌNG:**
- ❌ KHÔNG hỗ trợ `Search`, `SortBy`, `SortDesc`
- ✅ Sorting FIXED theo `TotalSpent` (giảm dần)
- ✅ Chỉ có `Page`, `PageSize`, `StartDate`, `EndDate`

#### Response DTO

```typescript
interface TopCustomerDto {
  customerId: number;
  customerName: string;
  totalOrders: number;
  totalSpent: number;
  lastOrderDate: string; // ISO 8601 DateTime
}
```

#### Example Request

```typescript
const response = await axios.get('/api/admin/reports/top-customers', {
  params: {
    Page: 1,
    PageSize: 10,
    StartDate: '2024-01-01',
    EndDate: '2024-12-31'
    // ❌ KHÔNG gửi Search, SortBy, SortDesc
  }
});
```


---

## 3. TYPESCRIPT INTERFACES

### 3.1. Base Interfaces

```typescript
// ============================================
// BASE REQUEST MODELS
// ============================================

/**
 * Base pagination request (chỉ có Page và PageSize)
 * Sử dụng cho Report endpoints
 */
export interface BasePagedRequest {
  page: number;      // Default: 1
  pageSize: number;  // Default: 10, Max: 100
}

/**
 * Full pagination request (có Search, SortBy, SortDesc)
 * Sử dụng cho tất cả endpoints quản lý
 */
export interface PagedRequest extends BasePagedRequest {
  search?: string;    // Optional, max 255 chars
  sortBy?: string;    // Default: "Id"
  sortDesc?: boolean; // Default: true
}

// ============================================
// RESPONSE MODELS
// ============================================

export interface ApiResponse<T> {
  isError: boolean;
  message: string;
  data: T | null;
  timestamp: string; // ISO 8601 DateTime
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
```

### 3.2. Specialized Request Interfaces

```typescript
// ============================================
// MANAGEMENT ENDPOINTS
// ============================================

export interface ProductSearchRequest extends PagedRequest {
  categoryId?: number;
  supplierId?: number;
  minPrice?: number;
  maxPrice?: number;
}

export interface CategorySearchRequest extends PagedRequest {
  minProductCount?: number;
  maxProductCount?: number;
  createdAfter?: string; // ISO 8601 DateTime
  createdBefore?: string; // ISO 8601 DateTime
}

export interface CustomerSearchRequest extends PagedRequest {
  // Không có filters đặc thù
}

export interface SupplierSearchRequest extends PagedRequest {
  // Không có filters đặc thù
}

export interface OrderSearchRequest extends PagedRequest {
  status?: 'Pending' | 'Paid' | 'Cancelled';
  customerId?: number;
  userId?: number;
  startDate?: string; // ISO 8601 DateTime
  endDate?: string; // ISO 8601 DateTime
}

export interface PromotionSearchRequest extends PagedRequest {
  // Không có filters đặc thù
}

export interface UserSearchRequest extends PagedRequest {
  role?: 0 | 1; // 0: Admin, 1: Staff
}

export interface InventorySearchRequest extends PagedRequest {
  productId?: number;
  minQuantity?: number;
  maxQuantity?: number;
}

// ============================================
// REPORT ENDPOINTS
// ============================================

export interface TopProductsSearchRequest extends BasePagedRequest {
  startDate: string; // ISO 8601 DateTime, Required
  endDate: string; // ISO 8601 DateTime, Required
}

export interface TopCustomersSearchRequest extends BasePagedRequest {
  startDate: string; // ISO 8601 DateTime, Required
  endDate: string; // ISO 8601 DateTime, Required
}
```

### 3.3. Response DTO Interfaces

```typescript
// ============================================
// MANAGEMENT DTOs
// ============================================

export interface ProductListDto {
  id: number;
  productName: string;
  barcode: string;
  price: number;
  unit: string;
  categoryName: string;
  supplierName: string;
  inventoryQuantity: number;
}

export interface CategoryResponseDto {
  id: number;
  categoryName: string;
  productCount: number;
}

export interface CustomerListDto {
  id: number;
  name: string;
  phone: string;
  email: string | null;
  lastOrderDate: string | null; // ISO 8601 DateTime
}

export interface SupplierResponseDto {
  id: number;
  name: string;
  phone: string;
  email: string | null;
  address: string | null;
  productCount: number;
}

export interface OrderListDto {
  id: number;
  orderDate: string; // ISO 8601 DateTime
  customerName: string;
  staffName: string;
  status: 'Pending' | 'Paid' | 'Cancelled';
  totalAmount: number;
  finalAmount: number;
}

export interface PromotionListDto {
  id: number;
  promoCode: string;
  description: string | null;
  discountType: 'percent' | 'fixed';
  discountValue: number;
  startDate: string; // ISO 8601 DateTime
  endDate: string; // ISO 8601 DateTime
  status: 'active' | 'inactive';
  usedCount: number;
  remainingUsage: number;
}

export interface UserResponseDto {
  id: number;
  username: string;
  fullName: string;
  role: 0 | 1; // 0: Admin, 1: Staff
  createdAt: string; // ISO 8601 DateTime
}

export interface InventoryResponseDto {
  inventoryId: number;
  productId: number;
  productName: string;
  barcode: string;
  quantity: number;
  updatedAt: string; // ISO 8601 DateTime
  status: 'in_stock' | 'low_stock' | 'out_of_stock';
}

// ============================================
// REPORT DTOs
// ============================================

export interface TopProductDto {
  productId: number;
  productName: string;
  totalQuantitySold: number;
  totalRevenue: number;
  orderCount: number;
}

export interface TopCustomerDto {
  customerId: number;
  customerName: string;
  totalOrders: number;
  totalSpent: number;
  lastOrderDate: string; // ISO 8601 DateTime
}
```


---

## 4. HƯỚNG DẪN IMPLEMENTATION

### 4.1. React Hook Example (usePagination)

```typescript
import { useState, useCallback } from 'react';
import axios from 'axios';
import type { PagedRequest, PagedList, ApiResponse } from '@/types';

interface UsePaginationOptions<TRequest extends PagedRequest, TDto> {
  endpoint: string;
  initialRequest?: Partial<TRequest>;
  onSuccess?: (data: PagedList<TDto>) => void;
  onError?: (error: string) => void;
}

export function usePagination<TRequest extends PagedRequest, TDto>({
  endpoint,
  initialRequest = {},
  onSuccess,
  onError
}: UsePaginationOptions<TRequest, TDto>) {
  const [request, setRequest] = useState<TRequest>({
    page: 1,
    pageSize: 10,
    sortBy: 'Id',
    sortDesc: true,
    ...initialRequest
  } as TRequest);

  const [data, setData] = useState<PagedList<TDto> | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const fetchData = useCallback(async (customRequest?: Partial<TRequest>) => {
    setLoading(true);
    setError(null);

    try {
      const finalRequest = { ...request, ...customRequest };
      const response = await axios.get<ApiResponse<PagedList<TDto>>>(endpoint, {
        params: finalRequest
      });

      if (response.data.isError) {
        throw new Error(response.data.message);
      }

      setData(response.data.data);
      onSuccess?.(response.data.data!);
    } catch (err: any) {
      const errorMessage = err.response?.data?.message || err.message;
      setError(errorMessage);
      onError?.(errorMessage);
    } finally {
      setLoading(false);
    }
  }, [endpoint, request, onSuccess, onError]);

  const goToPage = useCallback((page: number) => {
    setRequest(prev => ({ ...prev, page }));
    fetchData({ page } as Partial<TRequest>);
  }, [fetchData]);

  const changePageSize = useCallback((pageSize: number) => {
    setRequest(prev => ({ ...prev, pageSize, page: 1 }));
    fetchData({ pageSize, page: 1 } as Partial<TRequest>);
  }, [fetchData]);

  const search = useCallback((searchTerm: string) => {
    setRequest(prev => ({ ...prev, search: searchTerm, page: 1 }));
    fetchData({ search: searchTerm, page: 1 } as Partial<TRequest>);
  }, [fetchData]);

  const sort = useCallback((sortBy: string, sortDesc: boolean) => {
    setRequest(prev => ({ ...prev, sortBy, sortDesc, page: 1 }));
    fetchData({ sortBy, sortDesc, page: 1 } as Partial<TRequest>);
  }, [fetchData]);

  const applyFilters = useCallback((filters: Partial<TRequest>) => {
    setRequest(prev => ({ ...prev, ...filters, page: 1 }));
    fetchData({ ...filters, page: 1 } as Partial<TRequest>);
  }, [fetchData]);

  const refresh = useCallback(() => {
    fetchData();
  }, [fetchData]);

  return {
    // State
    request,
    data,
    loading,
    error,

    // Actions
    fetchData,
    goToPage,
    changePageSize,
    search,
    sort,
    applyFilters,
    refresh
  };
}
```

### 4.2. Usage Example (Products List)

```typescript
import { usePagination } from '@/hooks/usePagination';
import type { ProductSearchRequest, ProductListDto } from '@/types';

function ProductsPage() {
  const {
    data,
    loading,
    error,
    request,
    goToPage,
    changePageSize,
    search,
    sort,
    applyFilters,
    refresh
  } = usePagination<ProductSearchRequest, ProductListDto>({
    endpoint: '/api/admin/products',
    initialRequest: {
      sortBy: 'ProductName',
      sortDesc: false
    },
    onSuccess: (data) => {
      console.log(`Loaded ${data.items.length} products`);
    },
    onError: (error) => {
      console.error('Failed to load products:', error);
    }
  });

  // Load data on mount
  useEffect(() => {
    refresh();
  }, []);

  // Handle search
  const handleSearch = (e: React.ChangeEvent<HTMLInputElement>) => {
    search(e.target.value);
  };

  // Handle filter
  const handleCategoryFilter = (categoryId: number | null) => {
    applyFilters({ categoryId: categoryId || undefined });
  };

  // Handle sort
  const handleSort = (column: string) => {
    const isDesc = request.sortBy === column ? !request.sortDesc : true;
    sort(column, isDesc);
  };

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!data) return null;

  return (
    <div>
      {/* Search */}
      <input
        type="text"
        placeholder="Tìm kiếm sản phẩm..."
        onChange={handleSearch}
      />

      {/* Table */}
      <table>
        <thead>
          <tr>
            <th onClick={() => handleSort('ProductName')}>
              Tên sản phẩm {request.sortBy === 'ProductName' && (request.sortDesc ? '↓' : '↑')}
            </th>
            <th onClick={() => handleSort('Price')}>
              Giá {request.sortBy === 'Price' && (request.sortDesc ? '↓' : '↑')}
            </th>
            <th>Danh mục</th>
            <th>Tồn kho</th>
          </tr>
        </thead>
        <tbody>
          {data.items.map(product => (
            <tr key={product.id}>
              <td>{product.productName}</td>
              <td>{product.price.toLocaleString('vi-VN')} đ</td>
              <td>{product.categoryName}</td>
              <td>{product.inventoryQuantity}</td>
            </tr>
          ))}
        </tbody>
      </table>

      {/* Pagination */}
      <div>
        <button
          disabled={!data.hasPrevious}
          onClick={() => goToPage(request.page - 1)}
        >
          Trang trước
        </button>

        <span>
          Trang {data.page} / {data.totalPages} (Tổng: {data.totalCount} items)
        </span>

        <button
          disabled={!data.hasNext}
          onClick={() => goToPage(request.page + 1)}
        >
          Trang sau
        </button>

        <select
          value={request.pageSize}
          onChange={(e) => changePageSize(Number(e.target.value))}
        >
          <option value={10}>10 / trang</option>
          <option value={20}>20 / trang</option>
          <option value={50}>50 / trang</option>
          <option value={100}>100 / trang</option>
        </select>
      </div>
    </div>
  );
}
```


---

## 5. ERROR HANDLING

### 5.1. Common Validation Errors

Backend sẽ trả về `400 Bad Request` với validation errors:

```typescript
interface ValidationError {
  isError: true;
  message: string;
  data: null;
  timestamp: string;
}
```

**Các lỗi thường gặp:**

| Error Message | Nguyên nhân | Giải pháp |
|---------------|-------------|-----------|
| `'Page' must be greater than '0'` | Page < 1 | Đảm bảo Page >= 1 |
| `'Page Size' must be greater than '0'` | PageSize < 1 | Đảm bảo PageSize >= 1 |
| `The length of 'Search' must be 255 characters or fewer` | Search quá dài | Giới hạn max 255 chars |
| `'Sort By' must not be empty` | SortBy = "" | Không gửi hoặc gửi giá trị hợp lệ |
| `Invalid SortBy value` | SortBy không hợp lệ | Kiểm tra allowed values |
| `'Min Price' must be greater than or equal to '0'` | MinPrice < 0 | Đảm bảo MinPrice >= 0 |
| `MinPrice must be less than or equal to MaxPrice` | MinPrice > MaxPrice | Kiểm tra logic |
| `'Start Date' must be less than or equal to '...'` | StartDate > Now | Không cho phép ngày tương lai |
| `StartDate must be less than or equal to EndDate` | StartDate > EndDate | Kiểm tra logic |

### 5.2. Error Handling Example

```typescript
try {
  const response = await axios.get<ApiResponse<PagedList<ProductListDto>>>('/api/admin/products', {
    params: request
  });

  if (response.data.isError) {
    // Backend trả về error trong ApiResponse
    throw new Error(response.data.message);
  }

  // Success
  setData(response.data.data);
} catch (error: any) {
  if (error.response) {
    // HTTP error (4xx, 5xx)
    const status = error.response.status;
    const message = error.response.data?.message || error.message;

    switch (status) {
      case 400:
        // Validation error
        console.error('Validation error:', message);
        alert(`Lỗi validation: ${message}`);
        break;
      case 401:
        // Unauthorized
        console.error('Unauthorized');
        // Redirect to login
        window.location.href = '/login';
        break;
      case 403:
        // Forbidden
        console.error('Forbidden');
        alert('Bạn không có quyền truy cập tài nguyên này');
        break;
      case 404:
        // Not found
        console.error('Not found');
        alert('Không tìm thấy tài nguyên');
        break;
      case 500:
        // Server error
        console.error('Server error:', message);
        alert('Lỗi server, vui lòng thử lại sau');
        break;
      default:
        console.error('Unknown error:', message);
        alert(`Lỗi: ${message}`);
    }
  } else if (error.request) {
    // Network error
    console.error('Network error:', error.request);
    alert('Lỗi kết nối, vui lòng kiểm tra internet');
  } else {
    // Other error
    console.error('Error:', error.message);
    alert(`Lỗi: ${error.message}`);
  }
}
```

---

## 6. BEST PRACTICES

### 6.1. ✅ DO's

1. **Luôn validate input trước khi gửi request**
   ```typescript
   const validateRequest = (request: PagedRequest): string | null => {
     if (request.page < 1) return 'Page phải >= 1';
     if (request.pageSize < 1 || request.pageSize > 100) return 'PageSize phải từ 1-100';
     if (request.search && request.search.length > 255) return 'Search tối đa 255 ký tự';
     return null;
   };
   ```

2. **Reset về trang 1 khi thay đổi filters/search/sort**
   ```typescript
   const handleSearch = (searchTerm: string) => {
     setRequest(prev => ({ ...prev, search: searchTerm, page: 1 }));
   };
   ```

3. **Sử dụng debounce cho search input**
   ```typescript
   import { debounce } from 'lodash';

   const debouncedSearch = useMemo(
     () => debounce((value: string) => {
       search(value);
     }, 500),
     [search]
   );
   ```

4. **Cache pagination state trong URL query params**
   ```typescript
   const [searchParams, setSearchParams] = useSearchParams();

   useEffect(() => {
     const page = Number(searchParams.get('page')) || 1;
     const pageSize = Number(searchParams.get('pageSize')) || 10;
     const search = searchParams.get('search') || '';

     setRequest({ page, pageSize, search, sortBy: 'Id', sortDesc: true });
   }, [searchParams]);

   const goToPage = (page: number) => {
     setSearchParams({ ...Object.fromEntries(searchParams), page: String(page) });
   };
   ```

5. **Hiển thị loading state và empty state**
   ```typescript
   if (loading) return <Spinner />;
   if (error) return <ErrorMessage error={error} />;
   if (!data || data.items.length === 0) return <EmptyState />;
   ```

6. **Kiểm tra allowed SortBy values**
   ```typescript
   const ALLOWED_SORT_BY = ['Id', 'ProductName', 'Price', 'CreatedAt'];

   const handleSort = (column: string) => {
     if (!ALLOWED_SORT_BY.includes(column)) {
       console.error(`Invalid SortBy: ${column}`);
       return;
     }
     sort(column, !request.sortDesc);
   };
   ```

### 6.2. ❌ DON'Ts

1. **KHÔNG gửi Search/SortBy/SortDesc cho Report endpoints**
   ```typescript
   // ❌ SAI
   axios.get('/api/admin/reports/top-products', {
     params: { page: 1, pageSize: 10, sortBy: 'TotalRevenue' }
   });

   // ✅ ĐÚNG
   axios.get('/api/admin/reports/top-products', {
     params: { page: 1, pageSize: 10, startDate: '2024-01-01', endDate: '2024-12-31' }
   });
   ```

2. **KHÔNG gửi PageSize > 100**
   ```typescript
   // ❌ SAI - Backend sẽ tự động giới hạn về 100 nhưng không nên làm vậy
   const request = { page: 1, pageSize: 500 };

   // ✅ ĐÚNG
   const request = { page: 1, pageSize: Math.min(pageSize, 100) };
   ```

3. **KHÔNG gửi SortBy không hợp lệ**
   ```typescript
   // ❌ SAI
   const request = { sortBy: 'InvalidColumn' };

   // ✅ ĐÚNG - Kiểm tra trước
   const ALLOWED_SORT_BY = ['Id', 'ProductName', 'Price'];
   const sortBy = ALLOWED_SORT_BY.includes(column) ? column : 'Id';
   ```

4. **KHÔNG quên handle pagination metadata**
   ```typescript
   // ❌ SAI - Không kiểm tra hasPrevious/hasNext
   <button onClick={() => goToPage(page - 1)}>Previous</button>

   // ✅ ĐÚNG
   <button disabled={!data.hasPrevious} onClick={() => goToPage(page - 1)}>Previous</button>
   ```

5. **KHÔNG hard-code endpoint URLs**
   ```typescript
   // ❌ SAI
   axios.get('http://localhost:5000/api/admin/products');

   // ✅ ĐÚNG - Sử dụng environment variables
   const API_BASE_URL = import.meta.env.VITE_API_BASE_URL;
   axios.get(`${API_BASE_URL}/api/admin/products`);
   ```


---

## 7. QUICK REFERENCE TABLE

### 7.1. Tổng Hợp Tất Cả Endpoints

| # | Endpoint | URL | Auth | Request Model | Search | Sort | Filters |
|---|----------|-----|------|---------------|--------|------|---------|
| 1 | **Products** | `GET /api/admin/products` | Admin | `ProductSearchRequest` | ✅ | ✅ | CategoryId, SupplierId, MinPrice, MaxPrice |
| 2 | **Categories** | `GET /api/admin/categories` | Admin | `CategorySearchRequest` | ✅ | ✅ | MinProductCount, MaxProductCount, CreatedAfter, CreatedBefore |
| 3 | **Customers** | `GET /api/admin/customers` | Admin, Staff | `CustomerSearchRequest` | ✅ | ✅ | - |
| 4 | **Suppliers** | `GET /api/admin/suppliers` | Admin | `SupplierSearchRequest` | ✅ | ✅ | - |
| 5 | **Orders** | `GET /api/admin/orders` | Admin, Staff | `OrderSearchRequest` | ✅ | ✅ | Status, CustomerId, UserId, StartDate, EndDate |
| 6 | **Promotions** | `GET /api/admin/promotions` | Admin | `PromotionSearchRequest` | ✅ | ✅ | - |
| 7 | **Users** | `GET /api/admin/users` | Admin | `UserSearchRequest` | ✅ | ✅ | Role |
| 8 | **Inventory** | `GET /api/admin/inventory` | Admin, Staff | `InventorySearchRequest` | ✅ | ✅ | ProductId, MinQuantity, MaxQuantity |
| 9 | **Top Products** | `GET /api/admin/reports/top-products` | Admin | `TopProductsSearchRequest` | ❌ | ❌ (Fixed) | StartDate, EndDate |
| 10 | **Top Customers** | `GET /api/admin/reports/top-customers` | Admin | `TopCustomersSearchRequest` | ❌ | ❌ (Fixed) | StartDate, EndDate |

### 7.2. Allowed SortBy Values

| Endpoint | Allowed SortBy Values |
|----------|----------------------|
| **Products** | `Id`, `ProductName`, `Barcode`, `Price`, `Unit`, `CategoryId`, `SupplierId`, `CreatedAt`, `UpdatedAt`, `DeletedAt` |
| **Categories** | `Id`, `CategoryName`, `CreatedAt`, `UpdatedAt`, `DeletedAt` |
| **Customers** | `Id`, `Name`, `Phone`, `Email`, `Address`, `CreatedAt`, `UpdatedAt`, `DeletedAt` |
| **Suppliers** | `Id`, `Name`, `Phone`, `Email`, `Address`, `CreatedAt`, `UpdatedAt`, `DeletedAt` |
| **Orders** | `Id`, `CustomerId`, `UserId`, `PromoId`, `OrderDate`, `Status`, `TotalAmount`, `DiscountAmount`, `CreatedAt`, `UpdatedAt`, `DeletedAt` |
| **Promotions** | `Id`, `PromoCode`, `Description`, `DiscountType`, `DiscountValue`, `StartDate`, `EndDate`, `MinOrderAmount`, `UsageLimit`, `UsedCount`, `Status`, `CreatedAt`, `UpdatedAt`, `DeletedAt` |
| **Users** | `Id`, `Username`, `FullName`, `Role`, `CreatedAt`, `UpdatedAt`, `DeletedAt` |
| **Inventory** | `Id`, `ProductId`, `Quantity`, `CreatedAt`, `UpdatedAt`, `DeletedAt` |
| **Top Products** | ❌ KHÔNG hỗ trợ (Fixed: `TotalRevenue DESC`) |
| **Top Customers** | ❌ KHÔNG hỗ trợ (Fixed: `TotalSpent DESC`) |

### 7.3. Default Values

| Parameter | Default Value | Note |
|-----------|---------------|------|
| `Page` | `1` | Trang đầu tiên |
| `PageSize` | `10` | 10 items/trang |
| `SortBy` | `"Id"` | Sắp xếp theo Id |
| `SortDesc` | `true` | Giảm dần (DESC) |
| `Search` | `null` | Không tìm kiếm |

### 7.4. Validation Limits

| Parameter | Min | Max | Note |
|-----------|-----|-----|------|
| `Page` | 1 | ∞ | Phải > 0 |
| `PageSize` | 1 | 100 | Backend tự động giới hạn về 100 |
| `Search` | - | 255 chars | Tối đa 255 ký tự |
| `MinPrice` | 0 | ∞ | Phải >= 0 |
| `MaxPrice` | 0 | ∞ | Phải >= MinPrice |
| `MinQuantity` | 0 | ∞ | Phải >= 0 |
| `MaxQuantity` | 0 | ∞ | Phải >= MinQuantity |
| `StartDate` | - | Now | Không cho phép ngày tương lai |
| `EndDate` | - | Now | Phải >= StartDate |

---

## 📝 NOTES

### ⚠️ Lưu Ý Quan Trọng

1. **Report Endpoints đặc biệt:**
   - TopProducts và TopCustomers KHÔNG hỗ trợ `Search`, `SortBy`, `SortDesc`
   - Sorting được hard-code theo business logic (TotalRevenue DESC, TotalSpent DESC)
   - Chỉ có `Page`, `PageSize`, `StartDate`, `EndDate`

2. **MaxPageSize = 100:**
   - Backend tự động giới hạn PageSize về 100 nếu vượt quá
   - Frontend nên validate trước khi gửi request

3. **DateTime Format:**
   - Tất cả DateTime đều sử dụng **ISO 8601** format
   - Example: `"2024-01-09T10:30:00Z"`
   - Khi gửi request, có thể dùng format ngắn: `"2024-01-09"`

4. **Search Logic:**
   - Search là **case-insensitive** (không phân biệt hoa thường)
   - Search áp dụng cho nhiều fields (tùy endpoint)
   - Example: Products search trong `ProductName` và `Barcode`

5. **Sorting:**
   - `SortDesc = true` → Giảm dần (DESC)
   - `SortDesc = false` → Tăng dần (ASC)
   - SortBy phải là property **chính xác** (case-insensitive)

6. **Authorization:**
   - Admin: Full access tất cả endpoints
   - Staff: Chỉ access Customers, Orders, Inventory
   - Cần JWT token trong header: `Authorization: Bearer <token>`

---

## 🎯 CHECKLIST IMPLEMENTATION

Khi implement pagination cho một endpoint mới, hãy đảm bảo:

- [ ] Đã tạo TypeScript interfaces cho Request và Response
- [ ] Đã validate tất cả input parameters
- [ ] Đã handle loading, error, và empty states
- [ ] Đã implement debounce cho search input
- [ ] Đã kiểm tra allowed SortBy values
- [ ] Đã reset về page 1 khi thay đổi filters/search/sort
- [ ] Đã disable pagination buttons khi không có previous/next
- [ ] Đã hiển thị pagination metadata (page, totalPages, totalCount)
- [ ] Đã handle validation errors từ backend
- [ ] Đã test với các edge cases (empty results, max page size, invalid sort)

---

## 📞 SUPPORT

Nếu có thắc mắc hoặc phát hiện lỗi, vui lòng liên hệ Backend Team hoặc tạo issue trên repository.

**Happy Coding! 🚀**

