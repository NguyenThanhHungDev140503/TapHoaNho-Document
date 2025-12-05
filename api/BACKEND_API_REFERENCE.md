app# 📚 TÀI LIỆU THAM KHẢO API BACKEND

> **Tài liệu đầy đủ về tất cả endpoints của Backend API**
> 
> **Phiên bản:** 1.1  
> **Ngày cập nhật:** 2025-01-09  
> **Backend Framework:** ASP.NET Core 9.0  
> **Database:** PostgreSQL (Neon)
> 
> **Cập nhật lần này:**
> - ✅ Bổ sung thông tin về Query Parameters Naming Convention (PascalCase)
> - ✅ Bổ sung thông tin về Request Validation Rules (additionalProperties: false)
> - ✅ Làm rõ sự khác biệt giữa Nullable vs Optional Fields
> - ✅ Bổ sung đầy đủ min/max constraints vào Validation Rules
> - ✅ Ghi chú về Promotions Status filter chưa được triển khai

---

## 📋 MỤC LỤC

1. [Tổng Quan API](#1-tổng-quan-api)
2. [Authentication Module](#2-authentication-module)
3. [Products Module](#3-products-module)
4. [Categories Module](#4-categories-module)
5. [Customers Module](#5-customers-module)
6. [Suppliers Module](#6-suppliers-module)
7. [Orders Module](#7-orders-module)
8. [Promotions Module](#8-promotions-module)
9. [Users Module](#9-users-module)
10. [Inventory Module](#10-inventory-module)
11. [Reports Module](#11-reports-module)
12. [TypeScript Interfaces](#12-typescript-interfaces)
13. [Error Handling](#13-error-handling)
14. [Quick Reference](#14-quick-reference)

**⚠️ LƯU Ý QUAN TRỌNG:**
- Xem [Query Parameters Naming Convention](#17-query-parameters-naming-convention) để biết cách sử dụng PascalCase
- Xem [Request Validation Rules](#18-request-validation-rules) để biết về `additionalProperties: false`
- Xem [Nullable vs Optional Fields](#19-nullable-vs-optional-fields) để hiểu sự khác biệt

---

## 1. TỔNG QUAN API

### 1.1. Base URL

```
Development: http://localhost:5000
Production: https://your-domain.com
```

### 1.2. Authentication

**Mechanism:** JWT (JSON Web Token)

**Header Format:**
```http
Authorization: Bearer <access_token>
```

**Token Expiration:**
- Access Token: 1 giờ
- Refresh Token: 7 ngày

### 1.3. Common Response Format

Tất cả endpoints đều trả về cấu trúc `ApiResponse<T>`:

```typescript
interface ApiResponse<T> {
  isError: boolean;      // true nếu có lỗi
  message: string;       // Thông báo (success/error message)
  data: T | null;        // Dữ liệu trả về (null nếu có lỗi)
  timestamp: string;     // ISO 8601 DateTime
  statusCode: number;    // HTTP status code (200, 201, 400, 401, 403, 404, 500)
}
```

**Example Success Response:**
```json
{
  "isError": false,
  "message": "Product created successfully",
  "data": {
    "id": 1,
    "productName": "Coca Cola",
    "price": 15000
  },
  "timestamp": "2025-01-09T10:30:00Z",
  "statusCode": 201
}
```

**Example Error Response:**
```json
{
  "isError": true,
  "message": "Product not found",
  "data": null,
  "timestamp": "2025-01-09T10:30:00Z",
  "statusCode": 404
}
```

### 1.4. Pagination Response Format

Endpoints có phân trang trả về `PagedList<T>`:

```typescript
interface PagedList<T> {
  page: number;          // Trang hiện tại
  pageSize: number;      // Số items trên trang
  totalCount: number;    // Tổng số items
  totalPages: number;    // Tổng số trang
  hasPrevious: boolean;  // Có trang trước không?
  hasNext: boolean;      // Có trang sau không?
  items: T[];           // Danh sách items
}
```

### 1.5. Common HTTP Status Codes

| Code | Meaning | Description |
|------|---------|-------------|
| 200 | OK | Request thành công |
| 201 | Created | Tạo resource thành công |
| 400 | Bad Request | Validation error hoặc invalid request |
| 401 | Unauthorized | Chưa đăng nhập hoặc token không hợp lệ |
| 403 | Forbidden | Không có quyền truy cập |
| 404 | Not Found | Resource không tồn tại |
| 409 | Conflict | Duplicate resource (ví dụ: barcode đã tồn tại) |
| 500 | Internal Server Error | Lỗi server |

### 1.6. Authorization Roles

| Role | Value | Description |
|------|-------|-------------|
| Admin | 0 | Toàn quyền truy cập tất cả endpoints |
| Staff | 1 | Truy cập giới hạn (Customers, Orders, Inventory) |

### 1.7. Query Parameters Naming Convention

**⚠️ QUAN TRỌNG:** Backend API sử dụng **PascalCase** cho tất cả query parameters.

**Ví dụ:**
- ✅ Đúng: `Page`, `PageSize`, `Search`, `SortBy`, `SortDesc`, `CategoryId`, `MinPrice`
- ❌ Sai: `page`, `pageSize`, `search`, `sortBy`, `sortDesc`, `categoryId`, `minPrice`

**Lưu ý khi implement TypeScript:**
- TypeScript interfaces có thể sử dụng camelCase để tuân theo convention của TypeScript
- Khi gọi API, **PHẢI** convert sang PascalCase trong query parameters
- Ví dụ mapping:

```typescript
// TypeScript interface (camelCase - internal)
interface ProductSearchRequest {
  page?: number;
  pageSize?: number;
  search?: string;
  sortBy?: string;
  sortDesc?: boolean;
  categoryId?: number;
  minPrice?: number;
}

// Khi gọi API (PascalCase - external)
const response = await axios.get('/api/admin/products', {
  params: {
    Page: request.page,           // Convert camelCase → PascalCase
    PageSize: request.pageSize,
    Search: request.search,
    SortBy: request.sortBy,
    SortDesc: request.sortDesc,
    CategoryId: request.categoryId,
    MinPrice: request.minPrice
  }
});
```

**Hoặc sử dụng helper function:**
```typescript
function toPascalCaseParams(params: Record<string, any>): Record<string, any> {
  const result: Record<string, any> = {};
  for (const [key, value] of Object.entries(params)) {
    if (value !== undefined && value !== null) {
      const pascalKey = key.charAt(0).toUpperCase() + key.slice(1);
      result[pascalKey] = value;
    }
  }
  return result;
}
```

### 1.8. Request Validation Rules

**⚠️ QUAN TRỌNG:** Tất cả request schemas có `additionalProperties: false`, nghĩa là:

- Backend sẽ **reject** request nếu có thêm properties không được định nghĩa trong schema
- Chỉ gửi các properties được định nghĩa trong interface
- Không được gửi thêm properties tùy ý

**Ví dụ:**
```typescript
// ✅ Đúng - chỉ gửi các properties được định nghĩa
const request = {
  categoryId: 1,
  supplierId: 2,
  productName: 'Coca Cola',
  barcode: '123456',
  price: 15000,
  unit: 'can'
};

// ❌ Sai - có thêm property không được định nghĩa
const request = {
  categoryId: 1,
  supplierId: 2,
  productName: 'Coca Cola',
  barcode: '123456',
  price: 15000,
  unit: 'can',
  extraField: 'value'  // ❌ Backend sẽ reject request này
};
```

### 1.9. Nullable vs Optional Fields

**Sự khác biệt:**
- **Optional (`?`):** Field có thể không gửi trong request (không có trong object)
- **Nullable (`| null`):** Field có thể gửi giá trị `null` trong request

**Ví dụ:**
```typescript
interface CreateCustomerRequest {
  name: string;           // Required
  phone: string;          // Required
  email?: string | null;   // Optional AND nullable - có thể không gửi hoặc gửi null
  address?: string | null; // Optional AND nullable
}

// ✅ Tất cả đều hợp lệ:
{ name: 'John', phone: '123' }                    // email và address không gửi
{ name: 'John', phone: '123', email: null }      // email = null
{ name: 'John', phone: '123', email: 'test@example.com' } // email có giá trị
```

**Lưu ý:** Trong tài liệu này, `field?: type` thường có nghĩa là "optional và có thể nullable" (`field?: type | null`).

---

## 2. AUTHENTICATION MODULE

**Base URL:** `/api/Auth`
**Authorization:** AllowAnonymous (Public)

### 2.1. 🔐 Login

**Endpoint:** `POST /api/Auth/login`
**Authorization:** Public
**Description:** Đăng nhập và nhận access token + refresh token

#### Request Body

```typescript
interface LoginRequest {
  username?: string;  // Optional, nullable
  password?: string;  // Optional, nullable
}
```

#### Response

```typescript
interface LoginResponse {
  token: string;         // Access token (JWT)
  refreshToken: string;  // Refresh token
  user: UserDto;
}

interface UserDto {
  id: number;
  username: string;
  fullName: string;
  role: number;  // 0: Admin, 1: Staff
}
```

#### Example Request

```typescript
const response = await axios.post('/api/Auth/login', {
  username: 'admin',
  password: 'Admin@123'
});
```

#### Example Response

```json
{
  "isError": false,
  "message": "Login successful",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "user": {
      "id": 1,
      "username": "admin",
      "fullName": "Administrator",
      "role": 0
    }
  },
  "timestamp": "2025-01-09T10:30:00Z",
  "statusCode": 200
}
```

#### Status Codes

- **200 OK:** Login thành công
- **400 Bad Request:** Username hoặc password không hợp lệ
- **401 Unauthorized:** Sai username hoặc password

---

### 2.2. 🔧 Setup Admin

**Endpoint:** `POST /api/Auth/setup-admin`
**Authorization:** Public
**Description:** Tạo tài khoản Admin đầu tiên (chỉ hoạt động khi chưa có Admin nào)

#### Request Body

```typescript
interface CreateUserRequest {
  username: string;    // Required, max 50 chars
  password: string;    // Required, min 6 chars
  fullName: string;    // Required, max 100 chars
  role: number;        // Required, 0: Admin, 1: Staff
}
```

#### Validation Rules

- `username`: Required, string, min 1 character, max 50 characters
- `password`: Required, string, min 6 characters
- `fullName`: Required, string, min 1 character, max 100 characters
- `role`: Required, integer, minimum 0, maximum 1 (0: Admin, 1: Staff)

#### Response

```typescript
interface UserResponseDto {
  id: number;
  username: string;
  fullName: string;
  role: number;
}
```

#### Example Request

```typescript
const response = await axios.post('/api/Auth/setup-admin', {
  username: 'admin',
  password: 'Admin@123',
  fullName: 'System Administrator',
  role: 0
});
```

#### Status Codes

- **201 Created:** Admin được tạo thành công
- **400 Bad Request:** Validation error hoặc đã có Admin
- **500 Internal Server Error:** Lỗi server

---

### 2.3. 🔄 Refresh Token

**Endpoint:** `POST /api/Auth/refresh`
**Authorization:** Public
**Description:** Làm mới access token bằng refresh token

#### Request Body

```typescript
interface RefreshTokenRequest {
  accessToken: string;   // Required, access token cũ
  refreshToken: string;  // Required, refresh token
}
```

#### Response

```typescript
interface LoginResponse {
  token: string;         // Access token mới
  refreshToken: string;  // Refresh token mới
  user: UserDto;
}
```

#### Example Request

```typescript
const response = await axios.post('/api/Auth/refresh', {
  accessToken: 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...',
  refreshToken: 'a1b2c3d4-e5f6-7890-abcd-ef1234567890'
});
```

#### Status Codes

- **200 OK:** Token được làm mới thành công
- **400 Bad Request:** Invalid tokens
- **401 Unauthorized:** Refresh token hết hạn hoặc đã bị revoke

---

### 2.4. 🚪 Logout

**Endpoint:** `POST /api/Auth/logout`
**Authorization:** Public
**Description:** Đăng xuất và revoke refresh token

#### Request Body

```typescript
interface LogoutRequest {
  refreshToken: string;  // Required
}
```

#### Response

```typescript
// ApiResponse<object> với data = null
```

#### Example Request

```typescript
const response = await axios.post('/api/Auth/logout', {
  refreshToken: 'a1b2c3d4-e5f6-7890-abcd-ef1234567890'
});
```

#### Status Codes

- **200 OK:** Logout thành công
- **400 Bad Request:** Invalid refresh token

---

## 3. PRODUCTS MODULE

**Base URL:** `/api/admin/products`
**Authorization:** Admin only

### 3.1. 📋 Get Products List (Paginated)

**Endpoint:** `GET /api/admin/products`
**Authorization:** Admin
**Description:** Lấy danh sách sản phẩm có phân trang, tìm kiếm, và sắp xếp

#### Query Parameters

```typescript
interface ProductSearchRequest {
  // Pagination (từ PagedRequest)
  page?: number;          // Default: 1
  pageSize?: number;      // Default: 10, Max: 100
  search?: string;        // Tìm kiếm theo ProductName hoặc Barcode
  sortBy?: string;        // Default: "Id"
  sortDesc?: boolean;     // Default: true

  // Filters
  categoryId?: number;    // Lọc theo category
  supplierId?: number;    // Lọc theo supplier
  minPrice?: number;      // Giá tối thiểu
  maxPrice?: number;      // Giá tối đa
}
```

#### Allowed SortBy Values

- `Id` (default)
- `ProductName`
- `Price`
- `CategoryName`
- `SupplierName`
- `InventoryQuantity`
- `CreatedAt`

#### Response

```typescript
interface PagedList<ProductListDto> {
  page: number;
  pageSize: number;
  totalCount: number;
  totalPages: number;
  hasPrevious: boolean;
  hasNext: boolean;
  items: ProductListDto[];
}

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
const response = await axios.get('/api/admin/products', {
  params: {
    Page: 1,
    PageSize: 20,
    Search: 'Coca',
    SortBy: 'Price',
    SortDesc: false,
    CategoryId: 1,
    MinPrice: 10000,
    MaxPrice: 50000
  },
  headers: {
    Authorization: `Bearer ${accessToken}`
  }
});
```

#### Status Codes

- **200 OK:** Thành công
- **401 Unauthorized:** Chưa đăng nhập
- **403 Forbidden:** Không có quyền Admin

---

### 3.2. 🔍 Get Product By ID

**Endpoint:** `GET /api/admin/products/{id}`
**Authorization:** Admin
**Description:** Lấy thông tin chi tiết một sản phẩm

#### Path Parameters

- `id` (int, required): Product ID

#### Response

```typescript
interface ProductResponseDto {
  id: number;
  categoryId: number;
  categoryName: string;
  supplierId: number;
  supplierName: string;
  productName: string;
  barcode: string;
  price: number;
  unit: string;
  inventoryQuantity: number;
  createdAt: string;  // ISO 8601 DateTime
}
```

#### Example Request

```typescript
const response = await axios.get('/api/admin/products/1', {
  headers: {
    Authorization: `Bearer ${accessToken}`
  }
});
```

#### Status Codes

- **200 OK:** Thành công
- **401 Unauthorized:** Chưa đăng nhập
- **403 Forbidden:** Không có quyền Admin
- **404 Not Found:** Product không tồn tại

---

### 3.3. ➕ Create Product

**Endpoint:** `POST /api/admin/products`
**Authorization:** Admin
**Description:** Tạo sản phẩm mới

#### Request Body

```typescript
interface CreateProductRequest {
  categoryId: number;    // Required
  supplierId: number;    // Required
  productName: string;   // Required, max 100 chars
  barcode: string;       // Required, max 50 chars, unique
  price: number;         // Required, > 0
  unit: string;          // Required, max 20 chars, default "pcs"
}
```

#### Validation Rules

- `categoryId`: Required, integer, phải tồn tại trong database
- `supplierId`: Required, integer, phải tồn tại trong database
- `productName`: Required, string, min 1 character, max 100 characters
- `barcode`: Required, string, min 1 character, max 50 characters, **unique** (không được trùng)
- `price`: Required, number (double), minimum 0.01 (phải > 0)
- `unit`: Required, string, min 1 character, max 20 characters, default "pcs"

#### Response

```typescript
interface ProductResponseDto {
  id: number;
  categoryId: number;
  categoryName: string;
  supplierId: number;
  supplierName: string;
  productName: string;
  barcode: string;
  price: number;
  unit: string;
  inventoryQuantity: number;
  createdAt: string;
}
```

#### Example Request

```typescript
const response = await axios.post('/api/admin/products', {
  categoryId: 1,
  supplierId: 2,
  productName: 'Coca Cola 330ml',
  barcode: '8934588123456',
  price: 15000,
  unit: 'can'
}, {
  headers: {
    Authorization: `Bearer ${accessToken}`
  }
});
```

#### Status Codes

- **201 Created:** Product được tạo thành công
- **400 Bad Request:** Validation error
- **401 Unauthorized:** Chưa đăng nhập
- **403 Forbidden:** Không có quyền Admin
- **409 Conflict:** Barcode đã tồn tại

---

### 3.4. ✏️ Update Product

**Endpoint:** `PUT /api/admin/products/{id}`
**Authorization:** Admin
**Description:** Cập nhật thông tin sản phẩm

#### Path Parameters

- `id` (int, required): Product ID

#### Request Body

```typescript
interface UpdateProductRequest {
  id: number;            // Required, phải khớp với path parameter
  categoryId: number;    // Required
  supplierId: number;    // Required
  productName: string;   // Required, max 100 chars
  barcode: string;       // Required, max 50 chars, unique
  price: number;         // Required, > 0
  unit: string;          // Required, max 20 chars
}
```

#### Validation Rules

- Giống `CreateProductRequest` + thêm `id` phải khớp với path parameter

#### Response

```typescript
interface ProductResponseDto {
  // Same as Create Product response
}
```

#### Example Request

```typescript
const response = await axios.put('/api/admin/products/1', {
  id: 1,
  categoryId: 1,
  supplierId: 2,
  productName: 'Coca Cola 330ml (Updated)',
  barcode: '8934588123456',
  price: 16000,
  unit: 'can'
}, {
  headers: {
    Authorization: `Bearer ${accessToken}`
  }
});
```

#### Status Codes

- **200 OK:** Cập nhật thành công
- **400 Bad Request:** Validation error hoặc ID mismatch
- **401 Unauthorized:** Chưa đăng nhập
- **403 Forbidden:** Không có quyền Admin
- **404 Not Found:** Product không tồn tại
- **409 Conflict:** Barcode đã tồn tại (cho product khác)

---

### 3.5. 🗑️ Delete Product

**Endpoint:** `DELETE /api/admin/products/{id}`
**Authorization:** Admin
**Description:** Xóa sản phẩm (soft delete)

#### Path Parameters

- `id` (int, required): Product ID

#### Response

```typescript
// ApiResponse<object> với data = null
```

#### Example Request

```typescript
const response = await axios.delete('/api/admin/products/1', {
  headers: {
    Authorization: `Bearer ${accessToken}`
  }
});
```

#### Status Codes

- **200 OK:** Xóa thành công
- **401 Unauthorized:** Chưa đăng nhập
- **403 Forbidden:** Không có quyền Admin
- **404 Not Found:** Product không tồn tại

---

## 4. CATEGORIES MODULE

**Base URL:** `/api/admin/categories`
**Authorization:** Admin only

### 4.1. 📋 Get Categories List (Paginated)

**Endpoint:** `GET /api/admin/categories`
**Authorization:** Admin
**Description:** Lấy danh sách danh mục có phân trang, tìm kiếm, và sắp xếp

#### Query Parameters

```typescript
interface CategorySearchRequest {
  // Pagination (từ PagedRequest)
  page?: number;          // Default: 1
  pageSize?: number;      // Default: 10, Max: 100
  search?: string;        // Tìm kiếm theo CategoryName
  sortBy?: string;        // Default: "Id"
  sortDesc?: boolean;     // Default: true

  // Filters
  minProductCount?: number;    // Số lượng sản phẩm tối thiểu
  maxProductCount?: number;    // Số lượng sản phẩm tối đa
  createdAfter?: string;       // ISO 8601 DateTime - Lọc category tạo sau ngày này
  createdBefore?: string;      // ISO 8601 DateTime - Lọc category tạo trước ngày này
}
```

#### Allowed SortBy Values

- `Id` (default)
- `CategoryName`
- `ProductCount`

#### Response

```typescript
interface PagedList<CategoryResponseDto> {
  page: number;
  pageSize: number;
  totalCount: number;
  totalPages: number;
  hasPrevious: boolean;
  hasNext: boolean;
  items: CategoryResponseDto[];
}

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
    PageSize: 20,
    Search: 'Beverage',
    SortBy: 'ProductCount',
    SortDesc: true
  },
  headers: {
    Authorization: `Bearer ${accessToken}`
  }
});
```

---

### 4.2. 🔍 Get Category By ID

**Endpoint:** `GET /api/admin/categories/{id}`
**Authorization:** Admin

#### Response

```typescript
interface CategoryResponseDto {
  id: number;
  categoryName: string;
  productCount: number;
}
```

---

### 4.3. ➕ Create Category

**Endpoint:** `POST /api/admin/categories`
**Authorization:** Admin

#### Request Body

```typescript
interface CreateCategoryRequest {
  categoryName: string;  // Required, max 100 chars
}
```

#### Validation Rules

- `categoryName`: Required, string, min 1 character, max 100 characters

---

### 4.4. ✏️ Update Category

**Endpoint:** `PUT /api/admin/categories/{id}`
**Authorization:** Admin

#### Request Body

```typescript
interface UpdateCategoryRequest {
  id: number;            // Required
  categoryName: string;  // Required, max 100 chars
}
```

---

### 4.5. 🗑️ Delete Category

**Endpoint:** `DELETE /api/admin/categories/{id}`
**Authorization:** Admin

---

## 5. CUSTOMERS MODULE

**Base URL:** `/api/admin/customers`
**Authorization:** Admin & Staff

### 5.1. 📋 Get Customers List (Paginated)

**Endpoint:** `GET /api/admin/customers`
**Authorization:** Admin & Staff

#### Query Parameters

```typescript
interface CustomerSearchRequest {
  page?: number;
  pageSize?: number;
  search?: string;        // Tìm theo Name, Phone, Email
  sortBy?: string;        // Default: "Id"
  sortDesc?: boolean;
}
```

#### Allowed SortBy Values

- `Id` (default)
- `Name`
- `Phone`
- `Email`
- `LastOrderDate`

#### Response

```typescript
interface CustomerListDto {
  id: number;
  name: string;
  phone: string;
  email: string | null;
  lastOrderDate: string | null;  // ISO 8601 DateTime
}
```

---

### 5.2. 🔍 Get Customer By ID

**Endpoint:** `GET /api/admin/customers/{id}`
**Authorization:** Admin & Staff

#### Response

```typescript
interface CustomerResponseDto {
  id: number;
  name: string;
  phone: string;
  email: string | null;
  address: string | null;
  totalOrders: number;
  totalSpent: number;
  createdAt: string;  // ISO 8601 DateTime
}
```

---

### 5.3. ➕ Create Customer

**Endpoint:** `POST /api/admin/customers`
**Authorization:** Admin & Staff

#### Request Body

```typescript
interface CreateCustomerRequest {
  name: string;      // Required, max 100 chars
  phone: string;     // Required, max 20 chars
  email?: string;    // Optional, email format, max 100 chars
  address?: string;  // Optional, max 255 chars
}
```

#### Validation Rules

- `name`: Required, string, min 1 character, max 100 characters
- `phone`: Required, string, min 1 character, max 20 characters
- `email`: Optional, nullable, string, max 100 characters, must be valid email format
- `address`: Optional, nullable, string, max 255 characters

---

### 5.4. ✏️ Update Customer

**Endpoint:** `PUT /api/admin/customers/{id}`
**Authorization:** Admin & Staff

#### Request Body

```typescript
interface UpdateCustomerRequest {
  id: number;        // Required
  name: string;      // Required, max 100 chars
  phone: string;     // Required, max 20 chars
  email?: string;    // Optional, email format, max 100 chars
  address?: string;  // Optional, max 255 chars
}
```

---

### 5.5. 🗑️ Delete Customer

**Endpoint:** `DELETE /api/admin/customers/{id}`
**Authorization:** Admin & Staff

---

## 6. SUPPLIERS MODULE

**Base URL:** `/api/admin/suppliers`
**Authorization:** Admin only

### 6.1. 📋 Get Suppliers List (Paginated)

**Endpoint:** `GET /api/admin/suppliers`
**Authorization:** Admin

#### Query Parameters

```typescript
interface SupplierSearchRequest {
  page?: number;
  pageSize?: number;
  search?: string;        // Tìm theo Name, Phone, Email
  sortBy?: string;        // Default: "Id"
  sortDesc?: boolean;
}
```

#### Allowed SortBy Values

- `Id` (default)
- `Name`
- `Phone`
- `Email`
- `ProductCount`

#### Response

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

---

### 6.2. 🔍 Get Supplier By ID

**Endpoint:** `GET /api/admin/suppliers/{id}`
**Authorization:** Admin

---

### 6.3. ➕ Create Supplier

**Endpoint:** `POST /api/admin/suppliers`
**Authorization:** Admin

#### Request Body

```typescript
interface CreateSupplierRequest {
  name: string;      // Required, max 100 chars
  phone: string;     // Required, max 20 chars
  email?: string;    // Optional, max 100 chars
  address?: string;  // Optional, max 255 chars
}
```

---

### 6.4. ✏️ Update Supplier

**Endpoint:** `PUT /api/admin/suppliers/{id}`
**Authorization:** Admin

#### Request Body

```typescript
interface UpdateSupplierRequest {
  id: number;        // Required
  name: string;      // Required, max 100 chars
  phone: string;     // Required, max 20 chars
  email?: string;    // Optional, max 100 chars
  address?: string;  // Optional, max 255 chars
}
```

---

### 6.5. 🗑️ Delete Supplier

**Endpoint:** `DELETE /api/admin/suppliers/{id}`
**Authorization:** Admin

---

## 7. ORDERS MODULE

**Base URL:** `/api/admin/orders`
**Authorization:** Admin & Staff

### 7.1. 📋 Get Orders List (Paginated)

**Endpoint:** `GET /api/admin/orders`
**Authorization:** Admin & Staff

#### Query Parameters

```typescript
interface OrderSearchRequest {
  page?: number;
  pageSize?: number;
  search?: string;        // Tìm theo CustomerName, StaffName
  sortBy?: string;        // Default: "Id"
  sortDesc?: boolean;

  // Filters
  status?: string;        // | "Paid" | "Cancelled"
  customerId?: number;
  userId?: number;        // Staff ID
  startDate?: string;     // ISO 8601 DateTime
  endDate?: string;       // ISO 8601 DateTime
}
```

#### Allowed SortBy Values

- `Id` (default)
- `OrderDate`
- `CustomerName`
- `StaffName`
- `Status`
- `TotalAmount`
- `FinalAmount`

#### Response

```typescript
interface OrderListDto {
  id: number;
  orderDate: string;      // ISO 8601 DateTime
  customerName: string;
  staffName: string;
  status: string;         // "Pending" | "Paid" | "Cancelled"
  totalAmount: number;
  finalAmount: number;
}
```

---

### 7.2. 🔍 Get Order By ID

**Endpoint:** `GET /api/admin/orders/{id}`
**Authorization:** Admin & Staff

#### Response

```typescript
interface OrderDetailsDto {
  id: number;
  customerId: number;
  customerName: string;
  customerPhone: string;
  userId: number;
  staffName: string;
  promoId: number | null;
  promoCode: string | null;
  orderDate: string;      // ISO 8601 DateTime
  status: string;         // "Pending" | "Paid" | "Cancelled"
  totalAmount: number;
  discountAmount: number;
  finalAmount: number;
  orderItems: OrderItemDto[];
}

interface OrderItemDto {
  orderItemId: number;
  productId: number;
  productName: string;
  barcode: string;
  quantity: number;
  price: number;
  subtotal: number;
}
```

---

### 7.3. ➕ Create Order

**Endpoint:** `POST /api/admin/orders`
**Authorization:** Admin & Staff
**Description:** Tạo đơn hàng mới. UserId được lấy từ JWT token (user đang đăng nhập)

#### Request Body

```typescript
interface CreateOrderRequest {
  customerId: number;              // Required
  promoCode?: string;              // Optional, max 50 chars
  orderItems: OrderItemInput[];    // Required, min 1 item
}

interface OrderItemInput {
  productId: number;  // Required
  quantity: number;   // Required, > 0
}
```

#### Validation Rules

- `customerId`: Required, integer, phải tồn tại trong database
- `promoCode`: Optional, nullable, string, max 50 characters, phải valid và active nếu được cung cấp
- `orderItems`: Required, array, min 1 item (minItems: 1)
- `orderItems[].productId`: Required, integer, phải tồn tại trong database
- `orderItems[].quantity`: Required, integer, minimum 1, maximum 2147483647 (phải > 0)

#### Response

```typescript
interface OrderResponseDto {
  id: number;
  customerId: number;
  customerName: string;
  customerPhone: string;
  userId: number;
  staffName: string;
  promoId: number | null;
  promoCode: string | null;
  orderDate: string;
  status: string;
  totalAmount: number;
  discountAmount: number;
  finalAmount: number;
}
```

#### Example Request

```typescript
const response = await axios.post('/api/admin/orders', {
  customerId: 1,
  promoCode: 'SUMMER2025',
  orderItems: [
    { productId: 1, quantity: 2 },
    { productId: 3, quantity: 1 }
  ]
}, {
  headers: {
    Authorization: `Bearer ${accessToken}`
  }
});
```

---

### 7.4. 🔄 Update Order Status

**Endpoint:** `PATCH /api/admin/orders/{id}/status`
**Authorization:** Admin & Staff

#### Request Body

```typescript
interface UpdateOrderStatusRequest {
  status: string;  // Required, "paid" | "canceled"
}
```

#### Validation Rules

- `status`: Required, string, min 1 character, pattern: `^(paid|canceled)$` (lowercase)

---

### 7.5. ➕ Add Order Item

**Endpoint:** `POST /api/admin/orders/{orderId}/items`
**Authorization:** Admin & Staff

#### Request Body

```typescript
interface AddOrderItemRequest {
  productId: number;  // Required, integer
  quantity: number;   // Required, integer, minimum 1, maximum 2147483647
}
```

#### Validation Rules

- `productId`: Required, integer, phải tồn tại trong database
- `quantity`: Required, integer, minimum 1, maximum 2147483647

#### Response

```typescript
interface OrderItemResponseDto {
  orderItemId: number;
  orderId: number;
  productId: number;
  productName: string;
  quantity: number;
  price: number;
  subtotal: number;
}
```

---

### 7.6. ✏️ Update Order Item

**Endpoint:** `PUT /api/admin/orders/{orderId}/items/{itemId}`
**Authorization:** Admin & Staff

#### Request Body

```typescript
interface UpdateOrderItemRequest {
  quantity: number;  // Required, integer, minimum 1, maximum 2147483647
}
```

#### Validation Rules

- `quantity`: Required, integer, minimum 1, maximum 2147483647

---

### 7.7. 🗑️ Delete Order Item

**Endpoint:** `DELETE /api/admin/orders/{orderId}/items/{itemId}`
**Authorization:** Admin & Staff

---

### 7.8. 📄 Get Order Invoice (PDF)

**Endpoint:** `GET /api/admin/orders/{id}/invoice`
**Authorization:** Admin & Staff
**Description:** Tải hóa đơn dạng PDF

#### Response

- **Content-Type:** `application/pdf`
- **File:** PDF binary data

#### Example Request

```typescript
const response = await axios.get(`/api/admin/orders/${orderId}/invoice`, {
  headers: {
    Authorization: `Bearer ${accessToken}`
  },
  responseType: 'blob'  // Important for PDF download
});

// Download file
const url = window.URL.createObjectURL(new Blob([response.data]));
const link = document.createElement('a');
link.href = url;
link.setAttribute('download', `invoice-${orderId}.pdf`);
document.body.appendChild(link);
link.click();
```

---

## 8. PROMOTIONS MODULE

**Base URL:** `/api/admin/promotions`
**Authorization:** Admin only

### 8.1. 📋 Get Promotions List (Paginated)

**Endpoint:** `GET /api/admin/promotions`
**Authorization:** Admin

#### Query Parameters

```typescript
interface PromotionSearchRequest {
  page?: number;
  pageSize?: number;
  search?: string;        // Tìm theo PromoCode, Description
  sortBy?: string;        // Default: "Id"
  sortDesc?: boolean;

  // Filters
  // ⚠️ LƯU Ý: Filter Status chưa được backend triển khai, cần thực hiện triển khai
  // status?: string;        // "active" | "inactive" - CHƯA ĐƯỢC HỖ TRỢ
}
```

**⚠️ Lưu ý:** Query parameter `Status` để lọc theo trạng thái promotion hiện tại **chưa được backend triển khai**. Cần thực hiện triển khai tính năng này trong backend trước khi sử dụng.

#### Allowed SortBy Values

- `Id` (default)
- `PromoCode`
- `DiscountValue`
- `StartDate`
- `EndDate`
- `UsedCount`
- `Status`

#### Response

```typescript
interface PromotionListDto {
  id: number;
  promoCode: string;
  description: string | null;
  discountType: string;      // "percent" | "fixed"
  discountValue: number;
  startDate: string;         // ISO 8601 DateTime
  endDate: string;           // ISO 8601 DateTime
  status: string;            // "active" | "inactive"
  usedCount: number;
  remainingUsage: number;
}
```

---

### 8.2. 🔍 Get Promotion By ID

**Endpoint:** `GET /api/admin/promotions/{id}`
**Authorization:** Admin

#### Response

```typescript
interface PromotionResponseDto {
  id: number;
  promoCode: string;
  description: string | null;
  discountType: string;      // "percent" | "fixed"
  discountValue: number;
  startDate: string;         // ISO 8601 DateTime
  endDate: string;           // ISO 8601 DateTime
  minOrderAmount: number;
  usageLimit: number;
  usedCount: number;
  remainingUsage: number;
  status: string;            // "active" | "inactive"
  isActive: boolean;
}
```

---

### 8.3. ➕ Create Promotion

**Endpoint:** `POST /api/admin/promotions`
**Authorization:** Admin

#### Request Body

```typescript
interface CreatePromotionRequest {
  promoCode: string;       // Required, max 50 chars
  description?: string;    // Optional, max 255 chars
  discountType: string;    // Required, "percent" | "fixed"
  discountValue: number;   // Required, > 0
  startDate: string;       // Required, ISO 8601 DateTime
  endDate: string;         // Required, ISO 8601 DateTime
  minOrderAmount?: number; // Optional, >= 0, default 0
  usageLimit?: number;     // Optional, >= 1, default 1
  status: string;          // Required, "active" | "inactive"
}
```

#### Validation Rules

- `promoCode`: Required, string, min 1 character, max 50 characters
- `description`: Optional, nullable, string, max 255 characters
- `discountType`: Required, string, min 1 character, pattern: `^(percent|fixed)$`
- `discountValue`: Required, number (double), minimum 0.01 (phải > 0)
- `startDate`: Required, string, ISO 8601 DateTime format
- `endDate`: Required, string, ISO 8601 DateTime format, phải sau startDate
- `minOrderAmount`: Optional, number (double), minimum 0, default 0
- `usageLimit`: Optional, integer, minimum 1, maximum 2147483647, default 1
- `status`: Required, string, min 1 character, pattern: `^(active|inactive)$`

---

### 8.4. ✏️ Update Promotion

**Endpoint:** `PUT /api/admin/promotions/{id}`
**Authorization:** Admin

#### Request Body

```typescript
interface UpdatePromotionRequest {
  id: number;              // Required
  promoCode: string;       // Required, max 50 chars
  description?: string;    // Optional, max 255 chars
  discountType: string;    // Required, "percent" | "fixed"
  discountValue: number;   // Required, > 0
  startDate: string;       // Required, ISO 8601 DateTime
  endDate: string;         // Required, ISO 8601 DateTime
  minOrderAmount?: number; // Optional, >= 0, default 0
  usageLimit?: number;     // Optional, >= 1, default 1
  status: string;          // Required, "active" | "inactive"
}
```

---

### 8.5. 🗑️ Delete Promotion

**Endpoint:** `DELETE /api/admin/promotions/{id}`
**Authorization:** Admin

---

### 8.6. ✅ Validate Promo Code

**Endpoint:** `POST /api/admin/promotions/validate`
**Authorization:** Admin
**Description:** Kiểm tra promo code có hợp lệ không và tính discount amount

#### Request Body

```typescript
interface ValidatePromoRequest {
  promoCode: string;   // Required, min 1 character
  orderAmount: number; // Required, number (double), minimum 0.01
}
```

#### Validation Rules

- `promoCode`: Required, string, min 1 character
- `orderAmount`: Required, number (double), minimum 0.01 (phải > 0)

#### Response

```typescript
interface ValidatePromoResponse {
  isValid: boolean;
  message: string;
  discountAmount: number;
  promoId: number | null;
}
```

#### Example Request

```typescript
const response = await axios.post('/api/admin/promotions/validate', {
  promoCode: 'SUMMER2025',
  orderAmount: 100000
}, {
  headers: {
    Authorization: `Bearer ${accessToken}`
  }
});
```

#### Example Response (Valid)

```json
{
  "isError": false,
  "message": "Promo code validated successfully",
  "data": {
    "isValid": true,
    "message": "Promo code is valid",
    "discountAmount": 20000,
    "promoId": 1
  },
  "timestamp": "2025-01-09T10:30:00Z",
  "statusCode": 200
}
```

#### Example Response (Invalid)

```json
{
  "isError": false,
  "message": "Promo code validated successfully",
  "data": {
    "isValid": false,
    "message": "Promo code has expired",
    "discountAmount": 0,
    "promoId": null
  },
  "timestamp": "2025-01-09T10:30:00Z",
  "statusCode": 200
}
```

---

## 9. USERS MODULE

**Base URL:** `/api/admin/users`
**Authorization:** Admin only

### 9.1. 📋 Get Users List (Paginated)

**Endpoint:** `GET /api/admin/users`
**Authorization:** Admin

#### Query Parameters

```typescript
interface UserSearchRequest {
  page?: number;
  pageSize?: number;
  search?: string;        // Tìm theo Username, FullName
  sortBy?: string;        // Default: "Id"
  sortDesc?: boolean;

  // Filters
  role?: number;          // 0: Admin, 1: Staff
}
```

#### Allowed SortBy Values

- `Id` (default)
- `Username`
- `FullName`
- `Role`
- `CreatedAt`

#### Response

```typescript
interface UserResponseDto {
  id: number;
  username: string;
  fullName: string;
  role: number;  // 0: Admin, 1: Staff
}
```

---

### 9.2. 🔍 Get User By ID

**Endpoint:** `GET /api/admin/users/{id}`
**Authorization:** Admin

---

### 9.3. ➕ Create User

**Endpoint:** `POST /api/admin/users`
**Authorization:** Admin

#### Request Body

```typescript
interface CreateUserRequest {
  username: string;  // Required, max 50 chars
  password: string;  // Required, min 6 chars
  fullName: string;  // Required, max 100 chars
  role: number;      // Required, 0: Admin, 1: Staff
}
```

#### Validation Rules

- `username`: Required, string, min 1 character, max 50 characters, unique
- `password`: Required, string, min 6 characters
- `fullName`: Required, string, min 1 character, max 100 characters
- `role`: Required, integer, minimum 0, maximum 1 (0: Admin, 1: Staff)

---

### 9.4. ✏️ Update User

**Endpoint:** `PUT /api/admin/users/{id}`
**Authorization:** Admin

#### Request Body

```typescript
interface UpdateUserRequest {
  id: number;         // Required
  username: string;   // Required, max 50 chars
  password?: string;  // Optional, max 255 chars, null = không đổi password
  fullName: string;   // Required, max 100 chars
  role: number;       // Required, 0: Admin, 1: Staff
}
```

#### Validation Rules

- `id`: Required, integer, phải khớp với path parameter
- `username`: Required, string, min 1 character, max 50 characters
- `password`: Optional, nullable, string, max 255 characters - nếu null hoặc empty string thì không đổi password
- `fullName`: Required, string, min 1 character, max 100 characters
- `role`: Required, integer, minimum 0, maximum 1 (0: Admin, 1: Staff)

---

### 9.5. 🗑️ Delete User

**Endpoint:** `DELETE /api/admin/users/{id}`
**Authorization:** Admin

---

## 10. INVENTORY MODULE

**Base URL:** `/api/admin/inventory`
**Authorization:** Admin & Staff

### 10.1. 📋 Get Inventory List (Paginated)

**Endpoint:** `GET /api/admin/inventory`
**Authorization:** Admin & Staff

#### Query Parameters

```typescript
interface InventorySearchRequest {
  page?: number;
  pageSize?: number;
  search?: string;        // Tìm theo ProductName, Barcode
  sortBy?: string;        // Default: "Id"
  sortDesc?: boolean;

  // Filters
  productId?: number;     // Lọc theo Product ID
  minQuantity?: number;   // Số lượng tồn kho tối thiểu
  maxQuantity?: number;   // Số lượng tồn kho tối đa
}
```

#### Allowed SortBy Values

- `Id` (default)
- `ProductName`
- `Barcode`
- `Quantity`
- `UpdatedAt`
- `Status`

#### Response

```typescript
interface InventoryResponseDto {
  inventoryId: number;
  productId: number;
  productName: string;
  barcode: string;
  quantity: number;
  updatedAt: string;      // ISO 8601 DateTime
  status: string;         // "in_stock" | "low_stock" | "out_of_stock"
}
```

---

### 10.2. 🔍 Get Inventory By Product ID

**Endpoint:** `GET /api/admin/inventory/{productId}`
**Authorization:** Admin & Staff

---

### 10.3. 🔄 Update Inventory

**Endpoint:** `PATCH /api/admin/inventory/{productId}`
**Authorization:** Admin & Staff
**Description:** Cập nhật số lượng tồn kho. UserId được lấy từ JWT token

#### Request Body

```typescript
interface UpdateInventoryRequest {
  quantityChange: number;  // Required, positive = tăng, negative = giảm
  reason: string;          // Required, max 255 chars
}
```

#### Validation Rules

- `quantityChange`: Required, integer (int32), có thể âm (giảm) hoặc dương (tăng)
- `reason`: Required, string, min 1 character, max 255 characters

#### Example Request

```typescript
// Nhập hàng (tăng 100)
const response = await axios.patch('/api/admin/inventory/1', {
  quantityChange: 100,
  reason: 'Nhập hàng từ nhà cung cấp'
}, {
  headers: {
    Authorization: `Bearer ${accessToken}`
  }
});

// Xuất hàng (giảm 50)
const response = await axios.patch('/api/admin/inventory/1', {
  quantityChange: -50,
  reason: 'Xuất hàng bán lẻ'
}, {
  headers: {
    Authorization: `Bearer ${accessToken}`
  }
});
```

---

### 10.4. ⚠️ Get Low Stock Alerts

**Endpoint:** `GET /api/admin/inventory/low-stock`
**Authorization:** Admin & Staff
**Description:** Lấy danh sách sản phẩm sắp hết hàng (không phân trang)

#### Response

```typescript
interface LowStockAlertDto {
  productId: number;
  productName: string;
  barcode: string;
  currentQuantity: number;
  threshold: number;
}[]
```

---

### 10.5. 📜 Get Inventory History

**Endpoint:** `GET /api/admin/inventory/{productId}/history`
**Authorization:** Admin & Staff
**Description:** Lấy lịch sử thay đổi tồn kho của một sản phẩm (có phân trang)

#### Query Parameters

```typescript
interface PagedRequest {
  page?: number;          // Default: 1
  pageSize?: number;      // Default: 10, Max: 100
  search?: string;        // Tìm theo Reason, UpdatedBy
  sortBy?: string;        // Default: "Id"
  sortDesc?: boolean;     // Default: true
}
```

#### Response

```typescript
interface InventoryHistoryDto {
  id: number;
  productId: number;
  productName: string;
  quantityChange: number;
  quantityAfter: number;
  reason: string;
  updatedBy: string;      // Staff name
  updatedAt: string;      // ISO 8601 DateTime
}
```

---

## 11. REPORTS MODULE

**Base URL:** `/api/admin/reports`
**Authorization:** Admin only

### 11.1. 💰 Revenue Report

**Endpoint:** `GET /api/admin/reports/revenue`
**Authorization:** Admin

#### Query Parameters

```typescript
interface RevenueReportRequest {
  startDate: string;  // Required, ISO 8601 DateTime
  endDate: string;    // Required, ISO 8601 DateTime
  groupBy?: string;   // Optional, "day" | "week" | "month", default "day"
}
```

#### Validation Rules

- `startDate`: Required, ISO 8601 DateTime
- `endDate`: Required, ISO 8601 DateTime
- `groupBy`: Optional, phải là "day", "week", hoặc "month", default "day"

#### Response

```typescript
interface RevenueReportDto {
  summary: RevenueSummaryDto;
  details: RevenueDetailDto[];
}

interface RevenueSummaryDto {
  overallRevenue: number;
  overallOrders: number;
  overallDiscount: number;
  averageOrderValue: number;
  period: string;
}

interface RevenueDetailDto {
  period: string;           // "2025-01-09", "2025-W02", "2025-01"
  totalRevenue: number;
  totalOrders: number;
  totalDiscount: number;
  averageOrderValue: number;
  date: string;             // ISO 8601 DateTime
}
```

#### Example Request

```typescript
const response = await axios.get('/api/admin/reports/revenue', {
  params: {
    StartDate: '2025-01-01T00:00:00Z',
    EndDate: '2025-01-31T23:59:59Z',
    GroupBy: 'day'
  },
  headers: {
    Authorization: `Bearer ${accessToken}`
  }
});
```

---

### 11.2. 📊 Sales Report

**Endpoint:** `GET /api/admin/reports/sales`
**Authorization:** Admin

#### Query Parameters

```typescript
interface SalesReportRequest {
  startDate: string;   // Required, ISO 8601 DateTime
  endDate: string;     // Required, ISO 8601 DateTime
  groupBy?: string;    // Optional, "day" | "week" | "month"
  categoryId?: number; // Optional, lọc theo category
}
```

#### Response

```typescript
interface SalesReportDto {
  topProducts: TopProductDto[];
  topCustomers: TopCustomerDto[];
  categoryBreakdown: CategorySalesDto[];
}

interface TopProductDto {
  productId: number;
  productName: string;
  totalQuantitySold: number;
  totalRevenue: number;
  orderCount: number;
}

interface TopCustomerDto {
  customerId: number;
  customerName: string;
  totalOrders: number;
  totalSpent: number;
  lastOrderDate: string;  // ISO 8601 DateTime
}

interface CategorySalesDto {
  categoryId: number;
  categoryName: string;
  totalRevenue: number;
  totalQuantitySold: number;
  productCount: number;
}
```

---

### 11.3. 🏆 Top Products Report (Paginated)

**Endpoint:** `GET /api/admin/reports/top-products`
**Authorization:** Admin
**Description:** Báo cáo sản phẩm bán chạy nhất (có phân trang, fixed sorting theo TotalRevenue DESC)

#### Query Parameters

```typescript
interface TopProductsSearchRequest {
  // Pagination (từ BasePagedRequest)
  page?: number;          // Default: 1
  pageSize?: number;      // Default: 10, Max: 100

  // Filters
  startDate: string;      // Required, ISO 8601 DateTime
  endDate: string;        // Required, ISO 8601 DateTime
}
```

**LƯU Ý:** Endpoint này **KHÔNG** hỗ trợ `Search`, `SortBy`, `SortDesc` vì sử dụng fixed sorting theo `TotalRevenue DESC`.

#### Response

```typescript
interface PagedList<TopProductDto> {
  page: number;
  pageSize: number;
  totalCount: number;
  totalPages: number;
  hasPrevious: boolean;
  hasNext: boolean;
  items: TopProductDto[];
}

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
    StartDate: '2025-01-01T00:00:00Z',
    EndDate: '2025-01-31T23:59:59Z'
  },
  headers: {
    Authorization: `Bearer ${accessToken}`
  }
});
```

---

### 11.4. 👥 Top Customers Report (Paginated)

**Endpoint:** `GET /api/admin/reports/top-customers`
**Authorization:** Admin
**Description:** Báo cáo khách hàng chi tiêu nhiều nhất (có phân trang, fixed sorting theo TotalSpent DESC)

#### Query Parameters

```typescript
interface TopCustomersSearchRequest {
  // Pagination (từ BasePagedRequest)
  page?: number;          // Default: 1
  pageSize?: number;      // Default: 10, Max: 100

  // Filters
  startDate: string;      // Required, ISO 8601 DateTime
  endDate: string;        // Required, ISO 8601 DateTime
}
```

**LƯU Ý:** Endpoint này **KHÔNG** hỗ trợ `Search`, `SortBy`, `SortDesc` vì sử dụng fixed sorting theo `TotalSpent DESC`.

#### Response

```typescript
interface PagedList<TopCustomerDto> {
  page: number;
  pageSize: number;
  totalCount: number;
  totalPages: number;
  hasPrevious: boolean;
  hasNext: boolean;
  items: TopCustomerDto[];
}

interface TopCustomerDto {
  customerId: number;
  customerName: string;
  totalOrders: number;
  totalSpent: number;
  lastOrderDate: string;  // ISO 8601 DateTime
}
```

---

## 12. TYPESCRIPT INTERFACES

### 12.1. Common Interfaces

```typescript
// ============================================
// API RESPONSE WRAPPER
// ============================================

export interface ApiResponse<T> {
  isError: boolean;
  message: string;
  data: T | null;
  timestamp: string;
  statusCode: number;
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

// ============================================
// BASE REQUEST MODELS
// ============================================

export interface BasePagedRequest {
  page?: number;          // Default: 1
  pageSize?: number;      // Default: 10, Max: 100
}

export interface PagedRequest extends BasePagedRequest {
  search?: string;
  sortBy?: string;        // Default: "Id"
  sortDesc?: boolean;     // Default: true
}
```

### 12.2. Authentication Interfaces

```typescript
// ============================================
// AUTHENTICATION
// ============================================

export interface LoginRequest {
  username?: string;  // Optional, nullable
  password?: string;  // Optional, nullable
}

export interface LoginResponse {
  token: string;
  refreshToken: string;
  user: UserDto;
}

export interface UserDto {
  id: number;
  username: string;
  fullName: string;
  role: number;  // 0: Admin, 1: Staff
}

export interface RefreshTokenRequest {
  accessToken: string;
  refreshToken: string;
}

export interface LogoutRequest {
  refreshToken: string;
}
```

### 12.3. Product Interfaces

```typescript
// ============================================
// PRODUCTS
// ============================================

export interface ProductSearchRequest extends PagedRequest {
  categoryId?: number;
  supplierId?: number;
  minPrice?: number;
  maxPrice?: number;
}

export interface CreateProductRequest {
  categoryId: number;
  supplierId: number;
  productName: string;
  barcode: string;
  price: number;
  unit: string;
}

export interface UpdateProductRequest {
  id: number;
  categoryId: number;
  supplierId: number;
  productName: string;
  barcode: string;
  price: number;
  unit: string;
}

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

export interface ProductResponseDto {
  id: number;
  categoryId: number;
  categoryName: string;
  supplierId: number;
  supplierName: string;
  productName: string;
  barcode: string;
  price: number;
  unit: string;
  inventoryQuantity: number;
  createdAt: string;
}
```

### 12.4. Category Interfaces

```typescript
// ============================================
// CATEGORIES
// ============================================

export interface CategorySearchRequest extends PagedRequest {
  minProductCount?: number;
  maxProductCount?: number;
  createdAfter?: string;    // ISO 8601 DateTime
  createdBefore?: string;   // ISO 8601 DateTime
}

export interface CreateCategoryRequest {
  categoryName: string;
}

export interface UpdateCategoryRequest {
  id: number;
  categoryName: string;
}

export interface CategoryResponseDto {
  id: number;
  categoryName: string;
  productCount: number;
}
```

### 12.5. Customer Interfaces

```typescript
// ============================================
// CUSTOMERS
// ============================================

export interface CustomerSearchRequest extends PagedRequest {}

export interface CreateCustomerRequest {
  name: string;              // Required, min 1, max 100 chars
  phone: string;             // Required, min 1, max 20 chars
  email?: string | null;     // Optional, nullable, max 100 chars, email format
  address?: string | null;   // Optional, nullable, max 255 chars
}

export interface UpdateCustomerRequest {
  id: number;                // Required, integer
  name: string;              // Required, min 1, max 100 chars
  phone: string;             // Required, min 1, max 20 chars
  email?: string | null;     // Optional, nullable, max 100 chars, email format
  address?: string | null;   // Optional, nullable, max 255 chars
}

export interface CustomerListDto {
  id: number;
  name: string;
  phone: string;
  email: string | null;
  lastOrderDate: string | null;
}

export interface CustomerResponseDto {
  id: number;
  name: string;
  phone: string;
  email: string | null;
  address: string | null;
  totalOrders: number;
  totalSpent: number;
  createdAt: string;
}
```

### 12.6. Supplier Interfaces

```typescript
// ============================================
// SUPPLIERS
// ============================================

export interface SupplierSearchRequest extends PagedRequest {}

export interface CreateSupplierRequest {
  name: string;              // Required, min 1, max 100 chars
  phone: string;             // Required, min 1, max 20 chars
  email?: string | null;     // Optional, nullable, max 100 chars
  address?: string | null;   // Optional, nullable, max 255 chars
}

export interface UpdateSupplierRequest {
  id: number;                // Required, integer
  name: string;              // Required, min 1, max 100 chars
  phone: string;             // Required, min 1, max 20 chars
  email?: string | null;     // Optional, nullable, max 100 chars
  address?: string | null;   // Optional, nullable, max 255 chars
}

export interface SupplierResponseDto {
  id: number;
  name: string;
  phone: string;
  email: string | null;
  address: string | null;
  productCount: number;
}
```

### 12.7. Order Interfaces

```typescript
// ============================================
// ORDERS
// ============================================

export interface OrderSearchRequest extends PagedRequest {
  status?: string;        // "Pending" | "Paid" | "Cancelled"
  customerId?: number;
  userId?: number;
  startDate?: string;
  endDate?: string;
}

export interface CreateOrderRequest {
  customerId: number;        // Required, integer
  promoCode?: string | null; // Optional, nullable, max 50 chars
  orderItems: OrderItemInput[]; // Required, min 1 item
}

export interface OrderItemInput {
  productId: number;  // Required, integer
  quantity: number;   // Required, integer, min 1, max 2147483647
}

export interface UpdateOrderStatusRequest {
  status: string;  // "paid" | "canceled"
}

export interface AddOrderItemRequest {
  productId: number;  // Required, integer
  quantity: number;   // Required, integer, min 1, max 2147483647
}

export interface UpdateOrderItemRequest {
  quantity: number;  // Required, integer, min 1, max 2147483647
}

export interface OrderListDto {
  id: number;
  orderDate: string;
  customerName: string;
  staffName: string;
  status: string;
  totalAmount: number;
  finalAmount: number;
}

export interface OrderResponseDto {
  id: number;
  customerId: number;
  customerName: string;
  customerPhone: string;
  userId: number;
  staffName: string;
  promoId: number | null;
  promoCode: string | null;
  orderDate: string;
  status: string;
  totalAmount: number;
  discountAmount: number;
  finalAmount: number;
}

export interface OrderDetailsDto extends OrderResponseDto {
  orderItems: OrderItemDto[];
}

export interface OrderItemDto {
  orderItemId: number;
  productId: number;
  productName: string;
  barcode: string;
  quantity: number;
  price: number;
  subtotal: number;
}

export interface OrderItemResponseDto {
  orderItemId: number;
  orderId: number;
  productId: number;
  productName: string;
  quantity: number;
  price: number;
  subtotal: number;
}
```

### 12.8. Promotion Interfaces

```typescript
// ============================================
// PROMOTIONS
// ============================================

export interface PromotionSearchRequest extends PagedRequest {
  // ⚠️ LƯU Ý: Filter Status chưa được backend triển khai, cần thực hiện triển khai
  // status?: string;  // "active" | "inactive" - CHƯA ĐƯỢC HỖ TRỢ
}

export interface CreatePromotionRequest {
  promoCode: string;        // Required, min 1, max 50 chars
  description?: string | null; // Optional, nullable, max 255 chars
  discountType: string;      // Required, "percent" | "fixed"
  discountValue: number;     // Required, min 0.01
  startDate: string;         // Required, ISO 8601 DateTime
  endDate: string;           // Required, ISO 8601 DateTime
  minOrderAmount?: number;   // Optional, min 0, default 0
  usageLimit?: number;       // Optional, min 1, max 2147483647, default 1
  status: string;            // Required, "active" | "inactive"
}

export interface UpdatePromotionRequest {
  id: number;                // Required, integer
  promoCode: string;         // Required, min 1, max 50 chars
  description?: string | null; // Optional, nullable, max 255 chars
  discountType: string;       // Required, "percent" | "fixed"
  discountValue: number;     // Required, min 0.01
  startDate: string;         // Required, ISO 8601 DateTime
  endDate: string;           // Required, ISO 8601 DateTime
  minOrderAmount?: number;   // Optional, min 0, default 0
  usageLimit?: number;        // Optional, min 1, max 2147483647, default 1
  status: string;            // Required, "active" | "inactive"
}

export interface ValidatePromoRequest {
  promoCode: string;   // Required, min 1 character
  orderAmount: number; // Required, min 0.01
}

export interface ValidatePromoResponse {
  isValid: boolean;
  message: string;
  discountAmount: number;
  promoId: number | null;
}

export interface PromotionListDto {
  id: number;
  promoCode: string;
  description: string | null;
  discountType: string;
  discountValue: number;
  startDate: string;
  endDate: string;
  status: string;
  usedCount: number;
  remainingUsage: number;
}

export interface PromotionResponseDto {
  id: number;
  promoCode: string;
  description: string | null;
  discountType: string;
  discountValue: number;
  startDate: string;
  endDate: string;
  minOrderAmount: number;
  usageLimit: number;
  usedCount: number;
  remainingUsage: number;
  status: string;
  isActive: boolean;
}
```

### 12.9. User Interfaces

```typescript
// ============================================
// USERS
// ============================================

export interface UserSearchRequest extends PagedRequest {
  role?: number;  // 0: Admin, 1: Staff
}

export interface CreateUserRequest {
  username: string;
  password: string;
  fullName: string;
  role: number;
}

export interface UpdateUserRequest {
  id: number;                // Required, integer
  username: string;          // Required, min 1, max 50 chars
  password?: string | null;  // Optional, nullable, max 255 chars - null = không đổi password
  fullName: string;          // Required, min 1, max 100 chars
  role: number;              // Required, 0 (Admin) or 1 (Staff)
}

export interface UserResponseDto {
  id: number;
  username: string;
  fullName: string;
  role: number;
}
```

### 12.10. Inventory Interfaces

```typescript
// ============================================
// INVENTORY
// ============================================

export interface InventorySearchRequest extends PagedRequest {
  productId?: number;
  minQuantity?: number;
  maxQuantity?: number;
}

export interface UpdateInventoryRequest {
  quantityChange: number;
  reason: string;
}

export interface InventoryResponseDto {
  inventoryId: number;
  productId: number;
  productName: string;
  barcode: string;
  quantity: number;
  updatedAt: string;
  status: string;
}

export interface InventoryHistoryDto {
  id: number;
  productId: number;
  productName: string;
  quantityChange: number;
  quantityAfter: number;
  reason: string;
  updatedBy: string;
  updatedAt: string;
}

export interface LowStockAlertDto {
  productId: number;
  productName: string;
  barcode: string;
  currentQuantity: number;
  threshold: number;
}
```

### 12.11. Report Interfaces

```typescript
// ============================================
// REPORTS
// ============================================

export interface RevenueReportRequest {
  startDate: string;
  endDate: string;
  groupBy?: string;  // "day" | "week" | "month"
}

export interface RevenueReportDto {
  summary: RevenueSummaryDto;
  details: RevenueDetailDto[];
}

export interface RevenueSummaryDto {
  overallRevenue: number;
  overallOrders: number;
  overallDiscount: number;
  averageOrderValue: number;
  period: string;
}

export interface RevenueDetailDto {
  period: string;
  totalRevenue: number;
  totalOrders: number;
  totalDiscount: number;
  averageOrderValue: number;
  date: string;
}

export interface SalesReportRequest {
  startDate: string;
  endDate: string;
  groupBy?: string;
  categoryId?: number;
}

export interface SalesReportDto {
  topProducts: TopProductDto[];
  topCustomers: TopCustomerDto[];
  categoryBreakdown: CategorySalesDto[];
}

export interface TopProductsSearchRequest extends BasePagedRequest {
  startDate: string;
  endDate: string;
}

export interface TopCustomersSearchRequest extends BasePagedRequest {
  startDate: string;
  endDate: string;
}

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
  lastOrderDate: string;
}

export interface CategorySalesDto {
  categoryId: number;
  categoryName: string;
  totalRevenue: number;
  totalQuantitySold: number;
  productCount: number;
}
```

### 12.12. Enums

```typescript
// ============================================
// ENUMS
// ============================================

export enum UserRole {
  Admin = 0,
  Staff = 1
}

export enum OrderStatus {
  Pending = 'Pending',
  Paid = 'Paid',
  Cancelled = 'Cancelled'
}

export enum PromotionStatus {
  Active = 'active',
  Inactive = 'inactive'
}

export enum DiscountType {
  Percent = 'percent',
  Fixed = 'fixed'
}

export enum InventoryStatus {
  InStock = 'in_stock',
  LowStock = 'low_stock',
  OutOfStock = 'out_of_stock'
}

export enum ReportGroupBy {
  Day = 'day',
  Week = 'week',
  Month = 'month'
}
```

---

## 13. ERROR HANDLING

### 13.1. Error Response Format

Tất cả errors đều trả về cấu trúc `ApiResponse<T>` với `isError = true`:

```typescript
interface ErrorResponse {
  isError: true;
  message: string;
  data: null;
  timestamp: string;
  statusCode: number;
}
```

### 13.2. Validation Errors

**Status Code:** 400 Bad Request

```json
{
  "isError": true,
  "message": "Validation failed",
  "data": null,
  "timestamp": "2025-01-09T10:30:00Z",
  "statusCode": 400
}
```

### 13.3. Authentication Errors

**Status Code:** 401 Unauthorized

```json
{
  "isError": true,
  "message": "Unauthorized. Please login.",
  "data": null,
  "timestamp": "2025-01-09T10:30:00Z",
  "statusCode": 401
}
```

### 13.4. Authorization Errors

**Status Code:** 403 Forbidden

```json
{
  "isError": true,
  "message": "You do not have permission to access this resource.",
  "data": null,
  "timestamp": "2025-01-09T10:30:00Z",
  "statusCode": 403
}
```

### 13.5. Not Found Errors

**Status Code:** 404 Not Found

```json
{
  "isError": true,
  "message": "Product not found",
  "data": null,
  "timestamp": "2025-01-09T10:30:00Z",
  "statusCode": 404
}
```

### 13.6. Conflict Errors

**Status Code:** 409 Conflict

```json
{
  "isError": true,
  "message": "Barcode already exists",
  "data": null,
  "timestamp": "2025-01-09T10:30:00Z",
  "statusCode": 409
}
```

### 13.7. Server Errors

**Status Code:** 500 Internal Server Error

```json
{
  "isError": true,
  "message": "An error occurred while processing your request",
  "data": null,
  "timestamp": "2025-01-09T10:30:00Z",
  "statusCode": 500
}
```

### 13.8. Error Handling Best Practices

```typescript
// Example: Axios error handling
try {
  const response = await axios.get('/api/admin/products/1', {
    headers: {
      Authorization: `Bearer ${accessToken}`
    }
  });

  if (response.data.isError) {
    // Handle API error
    console.error(response.data.message);
    return;
  }

  // Success
  const product = response.data.data;

} catch (error) {
  if (axios.isAxiosError(error)) {
    if (error.response) {
      // Server responded with error
      const apiError = error.response.data;

      switch (apiError.statusCode) {
        case 400:
          console.error('Validation error:', apiError.message);
          break;
        case 401:
          console.error('Unauthorized. Redirecting to login...');
          // Redirect to login
          break;
        case 403:
          console.error('Forbidden:', apiError.message);
          break;
        case 404:
          console.error('Not found:', apiError.message);
          break;
        case 409:
          console.error('Conflict:', apiError.message);
          break;
        case 500:
          console.error('Server error:', apiError.message);
          break;
        default:
          console.error('Unknown error:', apiError.message);
      }
    } else if (error.request) {
      // Request was made but no response
      console.error('No response from server');
    } else {
      // Error setting up request
      console.error('Request error:', error.message);
    }
  }
}
```

---

## 14. QUICK REFERENCE

### 14.1. Endpoints Summary

| Module | Endpoint | Method | Auth | Description |
|--------|----------|--------|------|-------------|
| **Auth** | `/api/Auth/login` | POST | Public | Đăng nhập |
| | `/api/Auth/setup-admin` | POST | Public | Tạo Admin đầu tiên |
| | `/api/Auth/refresh` | POST | Public | Làm mới token |
| | `/api/Auth/logout` | POST | Public | Đăng xuất |
| **Products** | `/api/admin/products` | GET | Admin | Danh sách sản phẩm |
| | `/api/admin/products/{id}` | GET | Admin | Chi tiết sản phẩm |
| | `/api/admin/products` | POST | Admin | Tạo sản phẩm |
| | `/api/admin/products/{id}` | PUT | Admin | Cập nhật sản phẩm |
| | `/api/admin/products/{id}` | DELETE | Admin | Xóa sản phẩm |
| **Categories** | `/api/admin/categories` | GET | Admin | Danh sách danh mục |
| | `/api/admin/categories/{id}` | GET | Admin | Chi tiết danh mục |
| | `/api/admin/categories` | POST | Admin | Tạo danh mục |
| | `/api/admin/categories/{id}` | PUT | Admin | Cập nhật danh mục |
| | `/api/admin/categories/{id}` | DELETE | Admin | Xóa danh mục |
| **Customers** | `/api/admin/customers` | GET | Admin & Staff | Danh sách khách hàng |
| | `/api/admin/customers/{id}` | GET | Admin & Staff | Chi tiết khách hàng |
| | `/api/admin/customers` | POST | Admin & Staff | Tạo khách hàng |
| | `/api/admin/customers/{id}` | PUT | Admin & Staff | Cập nhật khách hàng |
| | `/api/admin/customers/{id}` | DELETE | Admin & Staff | Xóa khách hàng |
| **Suppliers** | `/api/admin/suppliers` | GET | Admin | Danh sách nhà cung cấp |
| | `/api/admin/suppliers/{id}` | GET | Admin | Chi tiết nhà cung cấp |
| | `/api/admin/suppliers` | POST | Admin | Tạo nhà cung cấp |
| | `/api/admin/suppliers/{id}` | PUT | Admin | Cập nhật nhà cung cấp |
| | `/api/admin/suppliers/{id}` | DELETE | Admin | Xóa nhà cung cấp |
| **Orders** | `/api/admin/orders` | GET | Admin & Staff | Danh sách đơn hàng |
| | `/api/admin/orders/{id}` | GET | Admin & Staff | Chi tiết đơn hàng |
| | `/api/admin/orders` | POST | Admin & Staff | Tạo đơn hàng |
| | `/api/admin/orders/{id}/status` | PATCH | Admin & Staff | Cập nhật trạng thái |
| | `/api/admin/orders/{orderId}/items` | POST | Admin & Staff | Thêm item vào order |
| | `/api/admin/orders/{orderId}/items/{itemId}` | PUT | Admin & Staff | Cập nhật order item |
| | `/api/admin/orders/{orderId}/items/{itemId}` | DELETE | Admin & Staff | Xóa order item |
| | `/api/admin/orders/{id}/invoice` | GET | Admin & Staff | Tải hóa đơn PDF |
| **Promotions** | `/api/admin/promotions` | GET | Admin | Danh sách khuyến mãi |
| | `/api/admin/promotions/{id}` | GET | Admin | Chi tiết khuyến mãi |
| | `/api/admin/promotions` | POST | Admin | Tạo khuyến mãi |
| | `/api/admin/promotions/{id}` | PUT | Admin | Cập nhật khuyến mãi |
| | `/api/admin/promotions/{id}` | DELETE | Admin | Xóa khuyến mãi |
| | `/api/admin/promotions/validate` | POST | Admin | Validate promo code |
| **Users** | `/api/admin/users` | GET | Admin | Danh sách người dùng |
| | `/api/admin/users/{id}` | GET | Admin | Chi tiết người dùng |
| | `/api/admin/users` | POST | Admin | Tạo người dùng |
| | `/api/admin/users/{id}` | PUT | Admin | Cập nhật người dùng |
| | `/api/admin/users/{id}` | DELETE | Admin | Xóa người dùng |
| **Inventory** | `/api/admin/inventory` | GET | Admin & Staff | Danh sách tồn kho |
| | `/api/admin/inventory/{productId}` | GET | Admin & Staff | Chi tiết tồn kho |
| | `/api/admin/inventory/{productId}` | PATCH | Admin & Staff | Cập nhật tồn kho |
| | `/api/admin/inventory/low-stock` | GET | Admin & Staff | Cảnh báo sắp hết hàng |
| | `/api/admin/inventory/{productId}/history` | GET | Admin & Staff | Lịch sử tồn kho |
| **Reports** | `/api/admin/reports/revenue` | GET | Admin | Báo cáo doanh thu |
| | `/api/admin/reports/sales` | GET | Admin | Báo cáo bán hàng |
| | `/api/admin/reports/top-products` | GET | Admin | Top sản phẩm bán chạy |
| | `/api/admin/reports/top-customers` | GET | Admin | Top khách hàng chi tiêu |

### 14.2. Pagination Endpoints

| Endpoint | Search Fields | SortBy Options | Filters |
|----------|---------------|----------------|---------|
| Products | ProductName, Barcode | Id, ProductName, Price, CategoryName, SupplierName, InventoryQuantity, CreatedAt | CategoryId, SupplierId, MinPrice, MaxPrice |
| Categories | CategoryName | Id, CategoryName, ProductCount | MinProductCount, MaxProductCount, CreatedAfter, CreatedBefore |
| Customers | Name, Phone, Email | Id, Name, Phone, Email, LastOrderDate | - |
| Suppliers | Name, Phone, Email | Id, Name, Phone, Email, ProductCount | - |
| Orders | CustomerName, StaffName | Id, OrderDate, CustomerName, StaffName, Status, TotalAmount, FinalAmount | Status, CustomerId, UserId, StartDate, EndDate |
| Promotions | PromoCode, Description | Id, PromoCode, DiscountValue, StartDate, EndDate, UsedCount, Status | Status |
| Users | Username, FullName | Id, Username, FullName, Role, CreatedAt | Role |
| Inventory | ProductName, Barcode | Id, ProductName, Barcode, Quantity, UpdatedAt, Status | ProductId, MinQuantity, MaxQuantity |
| Top Products | - | Fixed: TotalRevenue DESC | StartDate, EndDate |
| Top Customers | - | Fixed: TotalSpent DESC | StartDate, EndDate |

### 14.3. Authorization Matrix

| Module | Admin | Staff |
|--------|-------|-------|
| Authentication | ✅ | ✅ |
| Products | ✅ | ❌ |
| Categories | ✅ | ❌ |
| Customers | ✅ | ✅ |
| Suppliers | ✅ | ❌ |
| Orders | ✅ | ✅ |
| Promotions | ✅ | ❌ |
| Users | ✅ | ❌ |
| Inventory | ✅ | ✅ |
| Reports | ✅ | ❌ |

---

## 🎉 KẾT LUẬN

Tài liệu này cung cấp **TẤT CẢ** thông tin cần thiết để frontend team có thể implement các tính năng một cách chính xác và đầy đủ.

**Tổng kết:**
- ✅ **4 Authentication endpoints** (Login, Setup Admin, Refresh, Logout)
- ✅ **5 Products endpoints** (List, Get, Create, Update, Delete)
- ✅ **5 Categories endpoints** (List, Get, Create, Update, Delete)
- ✅ **5 Customers endpoints** (List, Get, Create, Update, Delete)
- ✅ **5 Suppliers endpoints** (List, Get, Create, Update, Delete)
- ✅ **8 Orders endpoints** (List, Get, Create, Update Status, Add/Update/Delete Items, Invoice)
- ✅ **6 Promotions endpoints** (List, Get, Create, Update, Delete, Validate)
- ✅ **5 Users endpoints** (List, Get, Create, Update, Delete)
- ✅ **5 Inventory endpoints** (List, Get, Update, Low Stock, History)
- ✅ **4 Reports endpoints** (Revenue, Sales, Top Products, Top Customers)

**Tổng cộng: 52 endpoints**

**Nếu có thắc mắc hoặc cần thêm thông tin, vui lòng liên hệ Backend Team!** 🚀

