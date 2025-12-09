# CODE EXAMPLES CHI TIẾT CHO TỪNG ENDPOINT

> **Tài liệu này**: Cung cấp code examples đầy đủ cho **TẤT CẢ 52 endpoints** trong hệ thống TapHoaNho.
>
> **Liên kết**: Đây là phần mở rộng của `customHookCallAPIPlan.md`
>
> **Hybrid Approach**: Tất cả examples sử dụng **Core API Infrastructure** (ApiServiceInterface + BaseApiService + apiResponseAdapter + API types) với xử lý `ApiResponse<T>` wrapper và `PagedList<T>` pagination tự động.

## Lưu ý về Hybrid Approach

Tất cả code examples trong tài liệu này sử dụng:

1. **ApiServiceInterface**: Định nghĩa contract cho tất cả API services (getAll, getPaginated, getById, create, update, patch, delete, custom)
2. **BaseApiService**: Base class implement `ApiServiceInterface`, xử lý `ApiResponse<T>` và `PagedList<T>` ở một chỗ
3. **apiResponseAdapter**: Helper (ví dụ `unwrapResponse`) được BaseApiService dùng để unwrap và check `isError`
4. **API types (`api.types.ts`)**: Re-export `ApiResponse`, `PagedList`, `PagedRequest` để dùng thống nhất trong toàn app
5. **Feature-specific services**: Mỗi feature extend BaseApiService (ví dụ: `ProductApiService`, `UserApiService`) và thêm custom methods

**Import pattern**:
```typescript
// Core API imports
import type { ApiResponse, PagedList, PagedRequest } from '@/lib/api/types/api.types';
import { BaseApiService } from '@/lib/api/base/BaseApiService';

// Universal hooks
import { 
  useApiList, 
  useApiPaginated, 
  useApiDetail, 
  useApiCreate, 
  useApiUpdate, 
  useApiPatch, 
  useApiDelete,
  useApiCustomQuery,
  useApiCustomMutation
} from '@/hooks/useApi';

// Pagination hooks
import { usePaginationWithRouter } from '@/hooks/usePaginationWithRouter';
import { usePaginationLocal } from '@/hooks/usePaginationLocal';

// Feature-specific service
import { productApiService } from '@/features/products/api/ProductApiService';
```

---

## MỤC LỤC

1. [Products Module (5 endpoints)](#1-products-module)
2. [Categories Module (5 endpoints)](#2-categories-module)
3. [Customers Module (5 endpoints)](#3-customers-module)
4. [Suppliers Module (5 endpoints)](#4-suppliers-module)
5. [Orders Module (8 endpoints)](#5-orders-module)
6. [Promotions Module (6 endpoints)](#6-promotions-module)
7. [Users Module (5 endpoints)](#7-users-module)
8. [Inventory Module (5 endpoints)](#8-inventory-module)
9. [Reports Module (4 endpoints)](#9-reports-module)
10. [Authentication Module (4 endpoints)](#10-authentication-module)

**Tổng cộng: 52 endpoints** (4 Auth + 48 Admin endpoints)

---

## 1. PRODUCTS MODULE

### 1.1. GET /api/admin/products - List Products

**Authorization**: Admin only  
**Hook**: `usePaginationWithRouter` (Page components) hoặc `usePaginationLocal` (Modal/Drawer)

**Frontend Code (Page Component với URL sync)**:
```typescript
import { getRouteApi } from '@tanstack/react-router';
import { usePaginationWithRouter } from '@/hooks/usePaginationWithRouter';
import { productApiService } from '@/features/products/api/ProductApiService';
import type { ProductEntity } from '@/features/products/types/entity';
import { Table, Input, Select, InputNumber, Space, Button, Badge } from 'antd';

const routeApi = getRouteApi('/admin/products');

function ProductListPage() {
  const pagination = usePaginationWithRouter<ProductEntity>({
    apiService: productApiService,
    entity: 'products',
    routeApi,
  });

  return (
    <div>
      {/* Search */}
      <Input.Search
        placeholder="Tìm kiếm sản phẩm..."
        defaultValue={pagination.params.search}
        onSearch={pagination.handleSearch}
        allowClear
        style={{ width: 300, marginBottom: 16 }}
      />

      {/* Filters */}
      <Space style={{ marginBottom: 16 }} wrap>
        <Select
          placeholder="Danh mục"
          style={{ width: 200 }}
          value={pagination.filters.categoryId}
          onChange={(value) => pagination.handleFilterChange({ categoryId: value || undefined })}
          allowClear
        >
          <Select.Option value={1}>Nước giải khát</Select.Option>
          <Select.Option value={2}>Snack</Select.Option>
        </Select>

        <Select
          placeholder="Nhà cung cấp"
          style={{ width: 200 }}
          value={pagination.filters.supplierId}
          onChange={(value) => pagination.handleFilterChange({ supplierId: value || undefined })}
          allowClear
        >
          <Select.Option value={1}>Coca Cola Vietnam</Select.Option>
          <Select.Option value={2}>Pepsi Vietnam</Select.Option>
        </Select>

        <InputNumber
          placeholder="Giá tối thiểu"
          value={pagination.filters.minPrice}
          onChange={(value) => pagination.handleFilterChange({ minPrice: value || undefined })}
          style={{ width: 150 }}
          min={0}
        />

        <InputNumber
          placeholder="Giá tối đa"
          value={pagination.filters.maxPrice}
          onChange={(value) => pagination.handleFilterChange({ maxPrice: value || undefined })}
          style={{ width: 150 }}
          min={0}
        />

        {/* Badge hiển thị số filters đang active */}
        {pagination.activeFiltersCount > 0 && (
          <Badge count={pagination.activeFiltersCount} offset={[-5, 5]}>
            <Button onClick={() => pagination.clearFilters()}>
              Xóa bộ lọc
            </Button>
          </Badge>
        )}
      </Space>

      {/* Table */}
      <Table
        dataSource={pagination.items}
        rowKey="id"
        loading={pagination.isLoading}
        pagination={{
          current: pagination.params.page,
          pageSize: pagination.params.pageSize,
          total: pagination.totalCount,
          onChange: pagination.handlePageChange,
          showSizeChanger: true,
        }}
        onChange={(_, __, sorter) => {
          if (sorter && 'field' in sorter && sorter.field) {
            pagination.handleSort(
              sorter.field as string,
              sorter.order === 'descend'
            );
          }
        }}
      >
        <Table.Column title="ID" dataIndex="id" sorter />
        <Table.Column title="Tên sản phẩm" dataIndex="productName" sorter />
        <Table.Column 
          title="Giá" 
          dataIndex="price" 
          sorter 
          render={(price) => `${price.toLocaleString()} đ`}
        />
        <Table.Column title="Danh mục" dataIndex="categoryName" sorter />
        <Table.Column title="Nhà cung cấp" dataIndex="supplierName" sorter />
        <Table.Column title="Tồn kho" dataIndex="inventoryQuantity" sorter />
      </Table>
    </div>
  );
}
```

**Frontend Code (Modal/Drawer với Local State)**:
```typescript
import { usePaginationLocal } from '@/hooks/usePaginationLocal';
import { productApiService } from '@/features/products/api/ProductApiService';
import type { ProductEntity } from '@/features/products/types/entity';
import { Modal, Table, Input, Select, InputNumber, Space, Button, Badge } from 'antd';

interface ProductFilters {
  categoryId?: number;
  supplierId?: number;
  minPrice?: number;
  maxPrice?: number;
}

function ProductSelectionModal({ visible, onSelect, onCancel }: Props) {
  const pagination = usePaginationLocal<ProductEntity, ProductFilters>({
    apiService: productApiService,
    entity: 'products',
    initialFilters: {
      inStock: true, // Default filter
    },
    initialParams: {
      sortBy: 'ProductName',
      sortDesc: false,
    },
  });

  return (
    <Modal
      title="Chọn sản phẩm"
      visible={visible}
      onCancel={onCancel}
      footer={null}
      width={1000}
    >
      {/* Search và Filters tương tự như trên */}
      <Table
        dataSource={pagination.items}
        loading={pagination.isLoading}
        rowKey="id"
        pagination={{
          current: pagination.page,
          pageSize: pagination.pageSize,
          total: pagination.totalCount,
          onChange: pagination.handlePageChange,
        }}
        onRow={(record) => ({
          onClick: () => onSelect(record),
          style: { cursor: 'pointer' },
        })}
      />
    </Modal>
  );
}
```

**Backend Request**:
```http
GET /api/admin/products?page=1&pagesize=20&search=coca&sortby=ProductName&sortdesc=false&categoryid=1&minprice=10000&maxprice=50000
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Backend Response**:
```json
{
  "isError": false,
  "message": "Lấy danh sách sản phẩm thành công",
  "data": {
    "page": 1,
    "pageSize": 20,
    "totalCount": 45,
    "totalPages": 3,
    "hasPrevious": false,
    "hasNext": true,
    "items": [
      {
        "id": 1,
        "productName": "Coca Cola 330ml",
        "barcode": "8934588123456",
        "price": 12000,
        "unit": "can",
        "categoryId": 1,
        "categoryName": "Nước giải khát",
        "supplierId": 1,
        "supplierName": "Coca Cola Vietnam",
        "inventoryQuantity": 150,
        "createdAt": "2024-01-15T10:30:00Z"
      }
    ]
  },
  "timestamp": "2024-01-20T14:30:00Z",
  "statusCode": 200
}
```

---

### 1.2. GET /api/admin/products/{id} - Get Product Detail

**Authorization**: Admin only
**Hook**: `useApiDetail`

**Frontend Code**:
```typescript
import { useApiDetail } from '@/hooks/useApi';
import { productApiService } from '@/features/products/api/ProductApiService';
import type { ProductEntity } from '@/features/products/types/entity';
import { Descriptions, Spin, Alert, Empty, Button } from 'antd';

function ProductDetailPage({ productId }: { productId: number }) {
  const { data, isLoading, error, refetch } = useApiDetail<ProductEntity>({
    apiService: productApiService,
    entity: 'products',
    id: productId,
    options: {
      staleTime: 1000 * 60 * 10, // 10 phút cho detail
    },
  });

  if (isLoading) return <Spin />;
  if (error) return <Alert message={error.message} type="error" />;
  if (!data) return <Empty />;

  return (
    <div>
      <Descriptions title="Chi tiết sản phẩm" bordered extra={<Button onClick={refetch}>Refresh</Button>}>
        <Descriptions.Item label="ID">{data.id}</Descriptions.Item>
        <Descriptions.Item label="Tên sản phẩm">{data.productName}</Descriptions.Item>
        <Descriptions.Item label="Mã vạch">{data.barcode}</Descriptions.Item>
        <Descriptions.Item label="Giá">{data.price.toLocaleString()} đ</Descriptions.Item>
        <Descriptions.Item label="Đơn vị">{data.unit}</Descriptions.Item>
        <Descriptions.Item label="Danh mục">{data.categoryName}</Descriptions.Item>
        <Descriptions.Item label="Nhà cung cấp">{data.supplierName}</Descriptions.Item>
        <Descriptions.Item label="Tồn kho">{data.inventoryQuantity}</Descriptions.Item>
        <Descriptions.Item label="Ngày tạo">{new Date(data.createdAt).toLocaleString()}</Descriptions.Item>
      </Descriptions>
    </div>
  );
}
```

**Backend Request**:
```http
GET /api/admin/products/1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Backend Response**:
```json
{
  "isError": false,
  "message": "Lấy thông tin sản phẩm thành công",
  "data": {
    "id": 1,
    "productName": "Coca Cola 330ml",
    "barcode": "8934588123456",
    "price": 12000,
    "unit": "can",
    "categoryId": 1,
    "categoryName": "Nước giải khát",
    "supplierId": 1,
    "supplierName": "Coca Cola Vietnam",
    "inventoryQuantity": 150,
    "createdAt": "2024-01-15T10:30:00Z",
    "updatedAt": "2024-01-20T14:00:00Z"
  },
  "timestamp": "2024-01-20T14:30:00Z",
  "statusCode": 200
}
```

---

### 1.3. POST /api/admin/products - Create Product

**Authorization**: Admin only
**Hook**: `useApiCreate`

**Frontend Code**:
```typescript
import { useApiCreate } from '@/hooks/useApi';
import { productApiService } from '@/features/products/api/ProductApiService';
import type { ProductEntity } from '@/features/products/types/entity';
import type { CreateProductRequest } from '@/features/products/types/api';
import { Form, Input, InputNumber, Select, Button, message } from 'antd';

function CreateProductForm() {
  const [form] = Form.useForm();
  const { mutate, isPending } = useApiCreate<ProductEntity, CreateProductRequest>({
    apiService: productApiService,
    entity: 'products',
    invalidateQueries: ['categories', 'stats'], // Invalidate thêm các queries liên quan
    options: {
      onSuccess: (data) => {
        message.success('Tạo sản phẩm thành công!');
        form.resetFields();
      },
      onError: (error) => {
        message.error(error.message);
      }
    }
  });

  const onFinish = (values: CreateProductRequest) => {
    mutate(values);
  };

  return (
    <Form form={form} onFinish={onFinish} layout="vertical">
      <Form.Item
        label="Tên sản phẩm"
        name="productName"
        rules={[
          { required: true, message: 'Vui lòng nhập tên sản phẩm' },
          { min: 1, message: 'Tên sản phẩm phải có ít nhất 1 ký tự' },
          { max: 100, message: 'Tên sản phẩm tối đa 100 ký tự' }
        ]}
      >
        <Input placeholder="Coca Cola 330ml" />
      </Form.Item>

      <Form.Item
        label="Mã vạch"
        name="barcode"
        rules={[
          { required: true, message: 'Vui lòng nhập mã vạch' },
          { min: 1, message: 'Mã vạch phải có ít nhất 1 ký tự' },
          { max: 50, message: 'Mã vạch tối đa 50 ký tự' }
        ]}
      >
        <Input placeholder="8934588123456" />
      </Form.Item>

      <Form.Item
        label="Giá"
        name="price"
        rules={[
          { required: true, message: 'Vui lòng nhập giá' },
          { type: 'number', min: 0.01, message: 'Giá phải lớn hơn hoặc bằng 0.01' }
        ]}
      >
        <InputNumber style={{ width: '100%' }} placeholder="12000" />
      </Form.Item>

      <Form.Item
        label="Đơn vị"
        name="unit"
        rules={[
          { required: true, message: 'Vui lòng nhập đơn vị' },
          { max: 20, message: 'Đơn vị tối đa 20 ký tự' }
        ]}
      >
        <Input placeholder="can" />
      </Form.Item>

      <Form.Item
        label="Danh mục"
        name="categoryId"
        rules={[{ required: true, message: 'Vui lòng chọn danh mục' }]}
      >
        <Select placeholder="Chọn danh mục">
          <Select.Option value={1}>Nước giải khát</Select.Option>
          <Select.Option value={2}>Snack</Select.Option>
        </Select>
      </Form.Item>

      <Form.Item
        label="Nhà cung cấp"
        name="supplierId"
        rules={[{ required: true, message: 'Vui lòng chọn nhà cung cấp' }]}
      >
        <Select placeholder="Chọn nhà cung cấp">
          <Select.Option value={1}>Coca Cola Vietnam</Select.Option>
          <Select.Option value={2}>Pepsi Vietnam</Select.Option>
        </Select>
      </Form.Item>

      <Form.Item>
        <Button type="primary" htmlType="submit" loading={isPending}>
          Tạo sản phẩm
        </Button>
      </Form.Item>
    </Form>
  );
}
```

**Backend Request**:
```http
POST /api/admin/products
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "productName": "Coca Cola 330ml",
  "barcode": "8934588123456",
  "price": 12000,
  "unit": "can",
  "categoryId": 1,
  "supplierId": 1
}
```

**Backend Response**:
```json
{
  "isError": false,
  "message": "Tạo sản phẩm thành công",
  "data": {
    "id": 1,
    "productName": "Coca Cola 330ml",
    "barcode": "8934588123456",
    "price": 12000,
    "unit": "can",
    "categoryId": 1,
    "categoryName": "Nước giải khát",
    "supplierId": 1,
    "supplierName": "Coca Cola Vietnam",
    "inventoryQuantity": 0,
    "createdAt": "2024-01-20T14:30:00Z",
    "updatedAt": "2024-01-20T14:30:00Z"
  },
  "timestamp": "2024-01-20T14:30:00Z",
  "statusCode": 201
}
```

---

### 1.4. PUT /api/admin/products/{id} - Update Product

**Authorization**: Admin only
**Hook**: `useApiUpdate`

**Frontend Code**:
```typescript
import { useApiUpdate, useApiDetail } from '@/hooks/useApi';
import { productApiService } from '@/features/products/api/ProductApiService';
import type { ProductEntity } from '@/features/products/types/entity';
import type { UpdateProductRequest } from '@/features/products/types/api';
import { Form, Input, InputNumber, Select, Button, message, Spin } from 'antd';
import { useEffect } from 'react';

function UpdateProductForm({ productId }: { productId: number }) {
  const [form] = Form.useForm();

  // Load existing data
  const { data: product, isLoading } = useApiDetail<ProductEntity>({
    apiService: productApiService,
    entity: 'products',
    id: productId,
  });

  // Update mutation
  const { mutate, isPending } = useApiUpdate<ProductEntity, UpdateProductRequest>({
    apiService: productApiService,
    entity: 'products',
    invalidateQueries: ['categories', 'stats'],
    options: {
      onSuccess: () => {
        message.success('Cập nhật sản phẩm thành công!');
      },
      onError: (error) => {
        message.error(error.message);
      }
    }
  });

  useEffect(() => {
    if (product) {
      form.setFieldsValue(product);
    }
  }, [product, form]);

  const onFinish = (values: UpdateProductRequest) => {
    mutate({ id: productId, data: values });
  };

  if (isLoading) return <Spin />;

  return (
    <Form form={form} onFinish={onFinish} layout="vertical">
      <Form.Item label="Tên sản phẩm" name="productName" rules={[{ required: true }]}>
        <Input />
      </Form.Item>
      <Form.Item label="Mã vạch" name="barcode" rules={[{ required: true }]}>
        <Input />
      </Form.Item>
      <Form.Item label="Giá" name="price" rules={[{ required: true }]}>
        <InputNumber style={{ width: '100%' }} />
      </Form.Item>
      <Form.Item 
        label="Đơn vị" 
        name="unit" 
        rules={[
          { required: true, message: 'Vui lòng nhập đơn vị' },
          { min: 1, message: 'Đơn vị phải có ít nhất 1 ký tự' },
          { max: 20, message: 'Đơn vị tối đa 20 ký tự' }
        ]}
      >
        <Input />
      </Form.Item>
      <Form.Item label="Danh mục" name="categoryId" rules={[{ required: true }]}>
        <Select>
          <Select.Option value={1}>Nước giải khát</Select.Option>
          <Select.Option value={2}>Snack</Select.Option>
        </Select>
      </Form.Item>
      <Form.Item label="Nhà cung cấp" name="supplierId" rules={[{ required: true }]}>
        <Select>
          <Select.Option value={1}>Coca Cola Vietnam</Select.Option>
          <Select.Option value={2}>Pepsi Vietnam</Select.Option>
        </Select>
      </Form.Item>
      <Form.Item>
        <Button type="primary" htmlType="submit" loading={isPending}>
          Cập nhật
        </Button>
      </Form.Item>
    </Form>
  );
}
```

**Backend Request**:
```http
PUT /api/admin/products/1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "productName": "Coca Cola 330ml (Updated)",
  "barcode": "8934588123456",
  "price": 13000,
  "unit": "can",
  "categoryId": 1,
  "supplierId": 1
}
```

**Backend Response**:
```json
{
  "isError": false,
  "message": "Cập nhật sản phẩm thành công",
  "data": {
    "id": 1,
    "productName": "Coca Cola 330ml (Updated)",
    "barcode": "8934588123456",
    "price": 13000,
    "unit": "can",
    "categoryId": 1,
    "categoryName": "Nước giải khát",
    "supplierId": 1,
    "supplierName": "Coca Cola Vietnam",
    "inventoryQuantity": 150,
    "createdAt": "2024-01-15T10:30:00Z",
    "updatedAt": "2024-01-20T15:00:00Z"
  },
  "timestamp": "2024-01-20T15:00:00Z",
  "statusCode": 200
}
```

---

### 1.5. DELETE /api/admin/products/{id} - Delete Product

**Authorization**: Admin only
**Hook**: `useApiDelete`

**Frontend Code**:
```typescript
import { useApiDelete } from '@/hooks/useApi';
import { productApiService } from '@/features/products/api/ProductApiService';
import { Button, Popconfirm, message } from 'antd';
import { DeleteOutlined } from '@ant-design/icons';

function DeleteProductButton({ productId, productName }: { productId: number; productName: string }) {
  const { mutate, isPending } = useApiDelete({
    apiService: productApiService,
    entity: 'products',
    invalidateQueries: ['categories', 'stats'],
    options: {
      onSuccess: () => {
        message.success('Xóa sản phẩm thành công!');
      },
      onError: (error) => {
        message.error(error.message);
      }
    }
  });

  const handleDelete = () => {
    mutate(productId);
  };

  return (
    <Popconfirm
      title="Xóa sản phẩm"
      description={`Bạn có chắc chắn muốn xóa "${productName}"?`}
      onConfirm={handleDelete}
      okText="Xóa"
      cancelText="Hủy"
      okButtonProps={{ danger: true }}
    >
      <Button
        danger
        icon={<DeleteOutlined />}
        loading={isPending}
      >
        Xóa
      </Button>
    </Popconfirm>
  );
}
```

**Backend Request**:
```http
DELETE /api/admin/products/1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Backend Response**:
```json
{
  "isError": false,
  "message": "Xóa sản phẩm thành công",
  "data": {},
  "timestamp": "2024-01-20T15:10:00Z",
  "statusCode": 200
}
```

**Lưu ý**: Backend thực hiện **soft delete**, sản phẩm không bị xóa vĩnh viễn khỏi database mà chỉ được đánh dấu là đã xóa.

---

## 2. CATEGORIES MODULE

### 2.1. GET /api/admin/categories - List Categories

**Authorization**: Admin only | **Hook**: `useApiPaginated` hoặc `usePaginationWithRouter`

**Frontend Code**:
```typescript
import { useApiPaginated } from '@/hooks/useApi';
import { categoryApiService } from '@/features/categories/api/CategoryApiService';
import type { CategoryEntity } from '@/features/categories/types/entity';
import type { PagedRequest } from '@/lib/axios';

const { data, isLoading } = useApiPaginated<CategoryEntity>({
  apiService: categoryApiService,
  entity: 'categories',
  params: {
    page: 1,
    pageSize: 20,
    sortBy: 'CategoryName',
    sortDesc: false,
  } as PagedRequest,
  options: {
    staleTime: 1000 * 60 * 5, // 5 phút
  },
});
```

**Request**: `GET /api/admin/categories?page=1&pagesize=20&sortby=CategoryName&sortdesc=false`

**Response**:
```json
{
  "isError": false,
  "message": "Lấy danh sách danh mục thành công",
  "data": {
    "page": 1,
    "pageSize": 20,
    "totalCount": 10,
    "items": [
      {
        "id": 1,
        "categoryName": "Nước giải khát",
        "productCount": 25,
        "createdAt": "2024-01-10T10:00:00Z"
      }
    ]
  },
  "statusCode": 200
}
```

### 2.2. POST /api/admin/categories - Create Category

**Authorization**: Admin only | **Hook**: `useApiCreate`

**Frontend Code**:
```typescript
import { useApiCreate } from '@/hooks/useApi';
import { categoryApiService } from '@/features/categories/api/CategoryApiService';
import type { CategoryEntity } from '@/features/categories/types/entity';
import type { CreateCategoryRequest } from '@/features/categories/types/api';

const { mutate, isPending } = useApiCreate<CategoryEntity, CreateCategoryRequest>({
  apiService: categoryApiService,
  entity: 'categories',
  options: {
    onSuccess: () => {
      message.success('Tạo danh mục thành công!');
    },
  },
});

mutate({ categoryName: 'Bánh kẹo' });
```

**Request**:
```http
POST /api/admin/categories
Content-Type: application/json

{
  "categoryName": "Bánh kẹo"
}
```

**Response**:
```json
{
  "isError": false,
  "message": "Tạo danh mục thành công",
  "data": {
    "id": 3,
    "categoryName": "Bánh kẹo",
    "productCount": 0,
    "createdAt": "2024-01-20T15:30:00Z"
  },
  "statusCode": 201
}
```

### 2.3. PUT /api/admin/categories/{id} - Update Category

**Authorization**: Admin only | **Hook**: `useApiUpdate`

**Frontend Code**:
```typescript
import { useApiUpdate } from '@/hooks/useApi';
import { categoryApiService } from '@/features/categories/api/CategoryApiService';
import type { CategoryEntity } from '@/features/categories/types/entity';
import type { UpdateCategoryRequest } from '@/features/categories/types/api';

const { mutate, isPending } = useApiUpdate<CategoryEntity, UpdateCategoryRequest>({
  apiService: categoryApiService,
  entity: 'categories',
  options: {
    onSuccess: () => {
      message.success('Cập nhật danh mục thành công!');
    },
  },
});

mutate({ id: 3, data: { categoryName: 'Bánh kẹo & Snack' } });
```

**Request**:
```http
PUT /api/admin/categories/3
Content-Type: application/json

{
  "categoryName": "Bánh kẹo & Snack"
}
```

### 2.4. DELETE /api/admin/categories/{id} - Delete Category

**Authorization**: Admin only | **Hook**: `useApiDelete`

**Frontend Code**:
```typescript
import { useApiDelete } from '@/hooks/useApi';
import { categoryApiService } from '@/features/categories/api/CategoryApiService';

const { mutate, isPending } = useApiDelete({
  apiService: categoryApiService,
  entity: 'categories',
  options: {
    onSuccess: () => {
      message.success('Xóa danh mục thành công!');
    },
  },
});

mutate(3);
```

**Request**: `DELETE /api/admin/categories/3`

---

## 3. CUSTOMERS MODULE

### 3.1. GET /api/admin/customers - List Customers

**Authorization**: Admin & Staff | **Hook**: `useApiPaginated` hoặc `usePaginationWithRouter`

**Frontend Code**:
```typescript
import { useApiPaginated } from '@/hooks/useApi';
import { customerApiService } from '@/features/customers/api/CustomerApiService';
import type { CustomerEntity } from '@/features/customers/types/entity';
import type { PagedRequest } from '@/lib/axios';

const { data, isLoading } = useApiPaginated<CustomerEntity>({
  apiService: customerApiService,
  entity: 'customers',
  params: {
    page: 1,
    pageSize: 20,
    search: 'Nguyễn',
    sortBy: 'Name',
    sortDesc: false,
  } as PagedRequest,
});
```

**Request**: `GET /api/admin/customers?page=1&pagesize=20&search=Nguyễn&sortby=Name`

**Response**:
```json
{
  "isError": false,
  "data": {
    "items": [
      {
        "id": 1,
        "name": "Nguyễn Văn A",
        "phone": "0901234567",
        "email": "nguyenvana@gmail.com",
        "address": "123 Lê Lợi, Q1, TP.HCM",
        "lastOrderDate": "2024-01-15T10:00:00Z",
        "totalOrders": 15,
        "totalSpent": 1500000
      }
    ]
  },
  "statusCode": 200
}
```

### 3.2. POST /api/admin/customers - Create Customer

**Authorization**: Admin & Staff | **Hook**: `useApiCreate`

**Frontend Code**:
```typescript
import { useApiCreate } from '@/hooks/useApi';
import { customerApiService } from '@/features/customers/api/CustomerApiService';
import type { CustomerEntity } from '@/features/customers/types/entity';
import type { CreateCustomerRequest } from '@/features/customers/types/api';

const { mutate, isPending } = useApiCreate<CustomerEntity, CreateCustomerRequest>({
  apiService: customerApiService,
  entity: 'customers',
  options: {
    onSuccess: () => {
      message.success('Tạo khách hàng thành công!');
    },
  },
});

mutate({
  name: 'Trần Thị B',
  phone: '0912345678',
  email: 'tranthib@gmail.com',
  address: '456 Nguyễn Huệ, Q1, TP.HCM'
});
```

**Request**:
```http
POST /api/admin/customers
Content-Type: application/json

{
  "name": "Trần Thị B",
  "phone": "0912345678",
  "email": "tranthib@gmail.com",
  "address": "456 Nguyễn Huệ, Q1, TP.HCM"
}
```

---

## 4. SUPPLIERS MODULE

### 4.1. GET /api/admin/suppliers - List Suppliers

**Authorization**: Admin only | **Hook**: `useApiPaginated` hoặc `usePaginationWithRouter`

**Frontend Code**:
```typescript
import { useApiPaginated } from '@/hooks/useApi';
import { supplierApiService } from '@/features/suppliers/api/SupplierApiService';
import type { SupplierEntity } from '@/features/suppliers/types/entity';
import type { PagedRequest } from '@/lib/axios';

const { data, isLoading } = useApiPaginated<SupplierEntity>({
  apiService: supplierApiService,
  entity: 'suppliers',
  params: {
    page: 1,
    pageSize: 20,
    sortBy: 'Name',
    sortDesc: false,
  } as PagedRequest,
});
```

**Response**:
```json
{
  "isError": false,
  "data": {
    "items": [
      {
        "id": 1,
        "name": "Coca Cola Vietnam",
        "phone": "0281234567",
        "email": "contact@cocacola.vn",
        "address": "Tòa nhà ABC, Q1, TP.HCM",
        "productCount": 15
      }
    ]
  },
  "statusCode": 200
}
```

### 4.2. POST /api/admin/suppliers - Create Supplier

**Authorization**: Admin only | **Hook**: `useApiCreate`

**Frontend Code**:
```typescript
import { useApiCreate } from '@/hooks/useApi';
import { supplierApiService } from '@/features/suppliers/api/SupplierApiService';
import type { SupplierEntity } from '@/features/suppliers/types/entity';
import type { CreateSupplierRequest } from '@/features/suppliers/types/api';

const { mutate, isPending } = useApiCreate<SupplierEntity, CreateSupplierRequest>({
  apiService: supplierApiService,
  entity: 'suppliers',
  options: {
    onSuccess: () => {
      message.success('Tạo nhà cung cấp thành công!');
    },
  },
});

mutate({
  name: 'Pepsi Vietnam',
  phone: '0287654321',
  email: 'contact@pepsi.vn',
  address: 'Tòa nhà XYZ, Q3, TP.HCM'
});
```

---

## 5. ORDERS MODULE

### 5.1. GET /api/admin/orders - List Orders

**Authorization**: Admin & Staff | **Hook**: `usePaginationWithRouter` (Page components) hoặc `useApiPaginated`

**Frontend Code**:
```typescript
import { getRouteApi } from '@tanstack/react-router';
import { usePaginationWithRouter } from '@/hooks/usePaginationWithRouter';
import { orderApiService } from '@/features/orders/api/OrderApiService';
import type { OrderEntity } from '@/features/orders/types/entity';

const routeApi = getRouteApi('/admin/orders');

function OrderListPage() {
  const pagination = usePaginationWithRouter<OrderEntity>({
    apiService: orderApiService,
    entity: 'orders',
    routeApi,
  });

  // Filters được tự động quản lý qua URL
  // pagination.filters.status, pagination.filters.customerId, etc.
  
  return (
    <div>
      {/* Sử dụng pagination.items, pagination.handlePageChange, etc. */}
    </div>
  );
}
```

**Request**: `GET /api/admin/orders?page=1&pagesize=20&sortby=OrderDate&sortdesc=true&status=Pending&startdate=2024-01-01&enddate=2024-01-31`

**Response**:
```json
{
  "isError": false,
  "data": {
    "items": [
      {
        "id": 1,
        "orderDate": "2024-01-20T14:30:00Z",
        "customerId": 1,
        "customerName": "Nguyễn Văn A",
        "userId": 1,
        "staffName": "Admin",
        "status": "Pending",
        "totalAmount": 150000,
        "discountAmount": 15000,
        "finalAmount": 135000,
        "itemCount": 5
      }
    ]
  },
  "statusCode": 200
}
```

### 5.2. POST /api/admin/orders - Create Order

**Authorization**: Admin & Staff | **Hook**: `useApiCreate`

**Frontend Code**:
```typescript
import { useApiCreate } from '@/hooks/useApi';
import { orderApiService } from '@/features/orders/api/OrderApiService';
import type { OrderEntity } from '@/features/orders/types/entity';
import type { CreateOrderRequest } from '@/features/orders/types/api';

const { mutate, isPending } = useApiCreate<OrderEntity, CreateOrderRequest>({
  apiService: orderApiService,
  entity: 'orders',
  invalidateQueries: ['customers', 'inventory', 'reports'],
  options: {
    onSuccess: (data) => {
      message.success('Tạo đơn hàng thành công!');
      navigate({ to: '/admin/orders/$id', params: { id: data.id.toString() } });
    },
  },
});

mutate({
  customerId: 1,
  promotionId: 2,
  items: [
    { productId: 1, quantity: 5, price: 12000 },
    { productId: 2, quantity: 3, price: 25000 }
  ]
});
```

**Request**:
```http
POST /api/admin/orders
Content-Type: application/json

{
  "customerId": 1,
  "promotionId": 2,
  "items": [
    { "productId": 1, "quantity": 5, "price": 12000 },
    { "productId": 2, "quantity": 3, "price": 25000 }
  ]
}
```

**Response**:
```json
{
  "isError": false,
  "message": "Tạo đơn hàng thành công",
  "data": {
    "id": 1,
    "orderDate": "2024-01-20T14:30:00Z",
    "customerId": 1,
    "customerName": "Nguyễn Văn A",
    "userId": 1,
    "staffName": "Admin",
    "status": "Pending",
    "totalAmount": 135000,
    "discountAmount": 13500,
    "finalAmount": 121500,
    "items": [
      {
        "id": 1,
        "productId": 1,
        "productName": "Coca Cola 330ml",
        "quantity": 5,
        "price": 12000,
        "totalPrice": 60000
      },
      {
        "id": 2,
        "productId": 2,
        "productName": "Coca Cola 1.5L",
        "quantity": 3,
        "price": 25000,
        "totalPrice": 75000
      }
    ]
  },
  "statusCode": 201
}
```

### 5.3. PATCH /api/admin/orders/{id}/status - Update Order Status

**Authorization**: Admin & Staff | **Hook**: `useApiCustomMutation`

**Frontend Code**:
```typescript
import { useApiCustomMutation } from '@/hooks/useApi';
import { orderApiService } from '@/features/orders/api/OrderApiService';
import type { OrderEntity } from '@/features/orders/types/entity';

// Update to Paid - Sử dụng custom method vì endpoint là /orders/{id}/status
// ⚠️ LƯU Ý: Khi update order status, phải dùng lowercase "paid" hoặc "canceled"
const { mutate: updateStatus, isPending } = useApiCustomMutation<OrderEntity, { orderId: number; status: string }>({
  entity: 'orders',
  mutationFn: ({ orderId, status }) =>
    orderApiService.custom<OrderEntity>('patch', `${orderId}/status`, { status }),
  invalidateQueries: ['orders', 'reports'],
  options: {
    onSuccess: () => {
      message.success('Cập nhật trạng thái đơn hàng thành công!');
    },
  },
});

updateStatus({ orderId: 1, status: 'paid' }); // lowercase: "paid" | "canceled"
```

**Request**:
```http
PATCH /api/admin/orders/1/status
Content-Type: application/json

{
  "status": "paid"  // ⚠️ PHẢI dùng lowercase: "paid" | "canceled"
}
```

**Response**:
```json
{
  "isError": false,
  "message": "Cập nhật trạng thái đơn hàng thành công",
  "data": {
    "id": 1,
    "status": "Paid",
    "orderDate": "2024-01-20T14:30:00Z",
    "finalAmount": 121500
  },
  "statusCode": 200
}
```

**Status Values**:
- **For search/filter**: `"Pending"`, `"Paid"`, `"Cancelled"` (PascalCase)
- **For update status**: `"paid"`, `"canceled"` (lowercase) - ⚠️ **QUAN TRỌNG**: Phải dùng lowercase khi update
- **Response**: Backend có thể trả về PascalCase `"Pending"`, `"Paid"`, `"Cancelled"` trong response data

### 5.4. POST /api/admin/orders/{orderId}/items - Add Order Item

**Authorization**: Admin & Staff | **Hook**: `useApiCustomMutation`

**Frontend Code**:
```typescript
import { useApiCustomMutation } from '@/hooks/useApi';
import { orderApiService } from '@/features/orders/api/OrderApiService';
import type { OrderItemEntity } from '@/features/orders/types/entity';

const { mutate, isPending } = useApiCustomMutation<OrderItemEntity, { orderId: number; item: any }>({
  entity: 'orders',
  mutationFn: ({ orderId, item }) =>
    orderApiService.custom<OrderItemEntity>('post', `${orderId}/items`, item),
  invalidateQueries: ['orders'],
  options: {
    onSuccess: () => {
      message.success('Thêm sản phẩm vào đơn hàng thành công!');
    },
  },
});

mutate({
  orderId: 1,
  item: {
    productId: 3,
    quantity: 2,
    price: 15000
  }
});
```

**Request**:
```http
POST /api/admin/orders/1/items
Content-Type: application/json

{
  "productId": 3,
  "quantity": 2,
  "price": 15000
}
```

**Response**:
```json
{
  "isError": false,
  "message": "Thêm sản phẩm vào đơn hàng thành công",
  "data": {
    "id": 3,
    "productId": 3,
    "productName": "Pepsi 330ml",
    "quantity": 2,
    "price": 15000,
    "totalPrice": 30000
  },
  "statusCode": 201
}
```

### 5.5. PUT /api/admin/orders/{orderId}/items/{itemId} - Update Order Item

**Authorization**: Admin & Staff | **Hook**: `useApiCustomMutation`

**Frontend Code**:
```typescript
import { useApiCustomMutation } from '@/hooks/useApi';
import { orderApiService } from '@/features/orders/api/OrderApiService';
import type { OrderItemEntity } from '@/features/orders/types/entity';

const { mutate, isPending } = useApiCustomMutation<OrderItemEntity, { orderId: number; itemId: number; data: any }>({
  entity: 'orders',
  mutationFn: ({ orderId, itemId, data }) =>
    orderApiService.custom<OrderItemEntity>('put', `${orderId}/items/${itemId}`, data),
  invalidateQueries: ['orders'],
  options: {
    onSuccess: () => {
      message.success('Cập nhật sản phẩm trong đơn hàng thành công!');
    },
  },
});

mutate({
  orderId: 1,
  itemId: 2,
  data: {
    quantity: 5,
    price: 15000
  }
});
```

### 5.6. DELETE /api/admin/orders/{orderId}/items/{itemId} - Delete Order Item

**Authorization**: Admin & Staff | **Hook**: `useApiCustomMutation`

**Frontend Code**:
```typescript
import { useApiCustomMutation } from '@/hooks/useApi';
import { orderApiService } from '@/features/orders/api/OrderApiService';

const { mutate, isPending } = useApiCustomMutation<void, { orderId: number; itemId: number }>({
  entity: 'orders',
  mutationFn: ({ orderId, itemId }) =>
    orderApiService.custom<void>('delete', `${orderId}/items/${itemId}`),
  invalidateQueries: ['orders'],
  options: {
    onSuccess: () => {
      message.success('Xóa sản phẩm khỏi đơn hàng thành công!');
    },
  },
});

mutate({ orderId: 1, itemId: 2 });
```

### 5.7. GET /api/admin/orders/{id}/invoice - Download Invoice PDF

**Authorization**: Admin & Staff | **Hook**: Custom Axios call

**Frontend Code**:
```typescript
import axios from 'axios';

const downloadInvoice = async (orderId: number) => {
  try {
    const response = await axios.get(`/api/admin/orders/${orderId}/invoice`, {
      responseType: 'blob',
      headers: {
        Authorization: `Bearer ${localStorage.getItem('accessToken')}`
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
    window.URL.revokeObjectURL(url);
  } catch (error) {
    message.error('Tải hóa đơn thất bại');
  }
};
```

**Request**: `GET /api/admin/orders/1/invoice`

**Response**: Binary PDF file

---

## 6. PROMOTIONS MODULE

### 6.1. GET /api/admin/promotions - List Promotions

**Authorization**: Admin only | **Hook**: `useApiPaginated` hoặc `usePaginationWithRouter`

**Frontend Code**:
```typescript
import { useApiPaginated } from '@/hooks/useApi';
import { promotionApiService } from '@/features/promotions/api/PromotionApiService';
import type { PromotionEntity } from '@/features/promotions/types/entity';
import type { PagedRequest } from '@/lib/axios';

const { data, isLoading } = useApiPaginated<PromotionEntity>({
  apiService: promotionApiService,
  entity: 'promotions',
  params: {
    page: 1,
    pageSize: 20,
    sortBy: 'StartDate',
    sortDesc: true,
    // ⚠️ LƯU Ý: Filter Status chưa được backend triển khai, cần thực hiện triển khai
    // status: 'active', // CHƯA ĐƯỢC HỖ TRỢ - Comment out cho đến khi backend triển khai
  } as PagedRequest,
});
```

**Response**:
```json
{
  "isError": false,
  "data": {
    "items": [
      {
        "id": 1,
        "promoCode": "SUMMER2024",
        "discountType": "percent",
        "discountValue": 10,
        "startDate": "2024-06-01T00:00:00Z",
        "endDate": "2024-08-31T23:59:59Z",
        "minOrderAmount": 100000,
        "usageLimit": 100,
        "usedCount": 25,
        "status": "active"  // lowercase: "active" | "inactive"
      }
    ]
  },
  "statusCode": 200
}
```

### 6.2. POST /api/admin/promotions - Create Promotion

**Authorization**: Admin only | **Hook**: `useApiCreate`

**Frontend Code**:
```typescript
import { useApiCreate } from '@/hooks/useApi';
import { promotionApiService } from '@/features/promotions/api/PromotionApiService';
import type { PromotionEntity } from '@/features/promotions/types/entity';
import type { CreatePromotionRequest } from '@/features/promotions/types/api';

const { mutate, isPending } = useApiCreate<PromotionEntity, CreatePromotionRequest>({
  apiService: promotionApiService,
  entity: 'promotions',
  options: {
    onSuccess: () => {
      message.success('Tạo khuyến mãi thành công!');
    },
  },
});

mutate({
  promoCode: 'NEWYEAR2025',
  discountType: 'fixed',
  discountValue: 50000,
  startDate: '2025-01-01T00:00:00Z',
  endDate: '2025-01-31T23:59:59Z',
  minOrderAmount: 200000,
  usageLimit: 50
});
```

### 6.3. POST /api/admin/promotions/validate - Validate Promo Code

**Authorization**: Admin only | **Hook**: `useApiCustomMutation`

**Frontend Code**:
```typescript
import { useApiCustomMutation } from '@/hooks/useApi';
import { promotionApiService } from '@/features/promotions/api/PromotionApiService';
import type { PromotionValidationResponse } from '@/features/promotions/types/api';

const { mutate, isPending, data } = useApiCustomMutation<PromotionValidationResponse, { promoCode: string }>({
  entity: 'promotions',
  mutationFn: ({ promoCode }) =>
    promotionApiService.custom<PromotionValidationResponse>('post', 'validate', { promoCode }),
  options: {
    onSuccess: (data) => {
      if (data.isValid) {
        message.success('Mã khuyến mãi hợp lệ!');
      } else {
        message.error('Mã khuyến mãi không hợp lệ hoặc đã hết hạn!');
      }
    },
  },
});

mutate({ promoCode: 'SUMMER2024' });
```

**Request**:
```http
POST /api/admin/promotions/validate
Content-Type: application/json

{
  "promoCode": "SUMMER2024"
}
```

**Response**:
```json
{
  "isError": false,
  "message": "Mã khuyến mãi hợp lệ",
  "data": {
    "id": 1,
    "promoCode": "SUMMER2024",
    "discountType": "percent",
    "discountValue": 10,
    "isValid": true
  },
  "statusCode": 200
}
```

---

## 7. USERS MODULE

### 7.1. GET /api/admin/users - List Users

**Authorization**: Admin only | **Hook**: `useApiPaginated` hoặc `usePaginationWithRouter`

**Frontend Code**:
```typescript
import { useApiPaginated } from '@/hooks/useApi';
import { userApiService } from '@/features/users/api/UserApiService';
import type { UserEntity } from '@/features/users/types/entity';
import type { PagedRequest } from '@/lib/axios';

const { data, isLoading } = useApiPaginated<UserEntity>({
  apiService: userApiService,
  entity: 'users',
  params: {
    page: 1,
    pageSize: 20,
    sortBy: 'CreatedAt',
    sortDesc: true,
    role: 1, // Filter by role (0 = Admin, 1 = Staff)
  } as PagedRequest,
});
```

**Response**:
```json
{
  "isError": false,
  "data": {
    "items": [
      {
        "id": 1,
        "username": "admin",
        "fullName": "Administrator",
        "role": 0,
        "createdAt": "2024-01-01T00:00:00Z"
      },
      {
        "id": 2,
        "username": "staff01",
        "fullName": "Nguyễn Văn A",
        "role": 1,
        "createdAt": "2024-01-10T10:00:00Z"
      }
    ]
  },
  "statusCode": 200
}
```

### 7.2. POST /api/admin/users - Create User

**Authorization**: Admin only | **Hook**: `useApiCreate`

**Frontend Code**:
```typescript
import { useApiCreate } from '@/hooks/useApi';
import { userApiService } from '@/features/users/api/UserApiService';
import type { UserEntity } from '@/features/users/types/entity';
import type { CreateUserRequest } from '@/features/users/types/api';

const { mutate, isPending } = useApiCreate<UserEntity, CreateUserRequest>({
  apiService: userApiService,
  entity: 'users',
  options: {
    onSuccess: () => {
      message.success('Tạo người dùng thành công!');
    },
  },
});

mutate({
  username: 'staff02',
  password: 'password123',
  fullName: 'Trần Thị B',
  role: 1 // 0 = Admin, 1 = Staff
});
```

---

## 8. INVENTORY MODULE

### 8.1. GET /api/admin/inventory - List Inventory

**Authorization**: Admin & Staff | **Hook**: `useApiPaginated` hoặc `usePaginationWithRouter`

**Frontend Code**:
```typescript
import { useApiPaginated } from '@/hooks/useApi';
import { inventoryApiService } from '@/features/inventory/api/InventoryApiService';
import type { InventoryEntity } from '@/features/inventory/types/entity';
import type { PagedRequest } from '@/lib/axios';

const { data, isLoading } = useApiPaginated<InventoryEntity>({
  apiService: inventoryApiService,
  entity: 'inventory',
  params: {
    page: 1,
    pageSize: 20,
    sortBy: 'Quantity',
    sortDesc: false,
    status: 'LowStock', // Filter by status
  } as PagedRequest,
});
```

**Response**:
```json
{
  "isError": false,
  "data": {
    "items": [
      {
        "id": 1,
        "productId": 1,
        "productName": "Coca Cola 330ml",
        "barcode": "8934588123456",
        "quantity": 15,
        "status": "LowStock",
        "updatedAt": "2024-01-20T14:00:00Z"
      }
    ]
  },
  "statusCode": 200
}
```

### 8.2. PATCH /api/admin/inventory/{productId} - Update Inventory Quantity

**Authorization**: Admin & Staff | **Hook**: `useApiPatch`

**Frontend Code**:
```typescript
import { useApiPatch } from '@/hooks/useApi';
import { inventoryApiService } from '@/features/inventory/api/InventoryApiService';
import type { InventoryEntity } from '@/features/inventory/types/entity';
import type { UpdateInventoryRequest } from '@/features/inventory/types/api';

const { mutate, isPending } = useApiPatch<InventoryEntity, UpdateInventoryRequest>({
  apiService: inventoryApiService,
  entity: 'inventory',
  invalidateQueries: ['products', 'reports'],
  options: {
    onSuccess: () => {
      message.success('Cập nhật tồn kho thành công!');
    },
  },
});

// Nhập hàng (tăng tồn kho)
mutate({
  id: productId,
  data: {
    quantityChange: 100,
    reason: 'Nhập hàng từ nhà cung cấp'
  }
});

// Xuất hàng (giảm tồn kho)
mutate({
  id: productId,
  data: {
    quantityChange: -50,
    reason: 'Xuất hàng bán lẻ'
  }
});
```

**Request (Increase)**:
```http
PATCH /api/admin/inventory/1
Content-Type: application/json

{
  "quantityChange": 100,
  "reason": "Nhập hàng từ nhà cung cấp"
}
```

**Request (Decrease)**:
```http
PATCH /api/admin/inventory/1
Content-Type: application/json

{
  "quantityChange": -50,
  "reason": "Xuất hàng bán lẻ"
}
```

**Response**:
```json
{
  "isError": false,
  "message": "Cập nhật tồn kho thành công",
  "data": {
    "id": 1,
    "productId": 1,
    "productName": "Coca Cola 330ml",
    "quantity": 200,
    "previousQuantity": 150,
    "quantityChange": 50,
    "updatedAt": "2024-01-20T15:00:00Z"
  },
  "statusCode": 200
}
```

### 8.3. GET /api/admin/inventory/low-stock - Low Stock Alerts

**Authorization**: Admin & Staff | **Hook**: `useApiCustomQuery`

**Frontend Code**:
```typescript
import { useApiCustomQuery } from '@/hooks/useApi';
import { inventoryApiService } from '@/features/inventory/api/InventoryApiService';
import type { InventoryEntity } from '@/features/inventory/types/entity';

const { data, isLoading } = useApiCustomQuery<InventoryEntity[]>({
  entity: 'inventory',
  queryKey: ['low-stock'],
  queryFn: () => inventoryApiService.custom<InventoryEntity[]>('get', 'low-stock'),
  options: {
    staleTime: 1000 * 60 * 5, // 5 phút
  },
});
```

**Request**: `GET /api/admin/inventory/low-stock`

**Response**:
```json
{
  "isError": false,
  "message": "Lấy danh sách sản phẩm sắp hết hàng thành công",
  "data": [
    {
      "id": 1,
      "productId": 1,
      "productName": "Coca Cola 330ml",
      "quantity": 15,
      "status": "LowStock"
    },
    {
      "id": 2,
      "productId": 5,
      "productName": "Pepsi 1.5L",
      "quantity": 8,
      "status": "LowStock"
    }
  ],
  "statusCode": 200
}
```

**Lưu ý**: Endpoint này KHÔNG phân trang, trả về array trực tiếp.

### 8.4. GET /api/admin/inventory/{productId}/history - Inventory History

**Authorization**: Admin & Staff | **Hook**: `useApiCustomQuery`

**Frontend Code**:
```typescript
import { useApiCustomQuery } from '@/hooks/useApi';
import { inventoryApiService } from '@/features/inventory/api/InventoryApiService';
import type { InventoryHistoryEntity } from '@/features/inventory/types/entity';

const { data, isLoading } = useApiCustomQuery<InventoryHistoryEntity[]>({
  entity: 'inventory',
  queryKey: ['history', productId],
  queryFn: () => inventoryApiService.custom<InventoryHistoryEntity[]>('get', `${productId}/history`),
  options: {
    enabled: !!productId,
    staleTime: 1000 * 60 * 5,
  },
});
```

**Request**: `GET /api/admin/inventory/1/history`

**Response**:
```json
{
  "isError": false,
  "data": [
    {
      "id": 1,
      "productId": 1,
      "quantityChange": 100,
      "reason": "Nhập hàng từ nhà cung cấp",
      "userId": 1,
      "userName": "Admin",
      "createdAt": "2024-01-20T10:00:00Z"
    },
    {
      "id": 2,
      "productId": 1,
      "quantityChange": -50,
      "reason": "Xuất hàng bán lẻ",
      "userId": 2,
      "userName": "Staff01",
      "createdAt": "2024-01-20T14:00:00Z"
    }
  ],
  "statusCode": 200
}
```

---

## 9. REPORTS MODULE

### 9.1. GET /api/admin/reports/revenue - Revenue Report

**Authorization**: Admin only | **Hook**: `useApiCustomQuery`

**Frontend Code**:
```typescript
import { useApiCustomQuery } from '@/hooks/useApi';
import { reportApiService } from '@/features/reports/api/ReportApiService';
import type { RevenueReportResponse } from '@/features/reports/types/api';

const { data, isLoading } = useApiCustomQuery<RevenueReportResponse>({
  entity: 'reports',
  queryKey: ['revenue', { startDate: '2024-01-01', endDate: '2024-01-31' }],
  queryFn: () => reportApiService.custom<RevenueReportResponse>('get', 'revenue', undefined, {
    startDate: '2024-01-01',
    endDate: '2024-01-31'
  }),
  options: {
    staleTime: 1000 * 60 * 10, // 10 phút cho reports
  },
});
```

**Request**: `GET /api/admin/reports/revenue?startdate=2024-01-01&enddate=2024-01-31`

**Response**:
```json
{
  "isError": false,
  "data": {
    "totalRevenue": 15000000,
    "totalOrders": 150,
    "averageOrderValue": 100000,
    "totalDiscount": 1500000,
    "netRevenue": 13500000,
    "period": {
      "startDate": "2024-01-01T00:00:00Z",
      "endDate": "2024-01-31T23:59:59Z"
    }
  },
  "statusCode": 200
}
```

**Lưu ý**: KHÔNG phân trang, trả về single object.

### 9.2. GET /api/admin/reports/sales - Sales Report

**Authorization**: Admin only | **Hook**: `useApiCustomQuery`

**Frontend Code**:
```typescript
import { useApiCustomQuery } from '@/hooks/useApi';
import { reportApiService } from '@/features/reports/api/ReportApiService';
import type { SalesReportResponse } from '@/features/reports/types/api';

const { data, isLoading } = useApiCustomQuery<SalesReportResponse>({
  entity: 'reports',
  queryKey: ['sales', { startDate: '2024-01-01', endDate: '2024-01-31' }],
  queryFn: () => reportApiService.custom<SalesReportResponse>('get', 'sales', undefined, {
    startDate: '2024-01-01',
    endDate: '2024-01-31'
  }),
  options: {
    staleTime: 1000 * 60 * 10,
  },
});
```

**Response**:
```json
{
  "isError": false,
  "data": {
    "totalSales": 15000000,
    "totalItems": 500,
    "totalOrders": 150,
    "averageItemsPerOrder": 3.33,
    "topSellingProducts": [
      {
        "productId": 1,
        "productName": "Coca Cola 330ml",
        "quantitySold": 200,
        "revenue": 2400000
      }
    ]
  },
  "statusCode": 200
}
```

### 9.3. GET /api/admin/reports/top-products - Top Products

**Authorization**: Admin only | **Hook**: `useApiPaginated`

**Frontend Code**:
```typescript
import { useApiPaginated } from '@/hooks/useApi';
import { reportApiService } from '@/features/reports/api/ReportApiService';
import type { TopProductEntity } from '@/features/reports/types/entity';
import type { PagedRequest } from '@/lib/axios';

const { data, isLoading } = useApiPaginated<TopProductEntity>({
  apiService: reportApiService,
  entity: 'reports',
  params: {
    page: 1,
    pageSize: 10,
    startDate: '2024-01-01',
    endDate: '2024-01-31',
    // ⚠️ KHÔNG hỗ trợ: search, sortBy, sortDesc
  } as PagedRequest,
});
```

**Request**: `GET /api/admin/reports/top-products?Page=1&PageSize=10&StartDate=2024-01-01&EndDate=2024-01-31`

**Response**:
```json
{
  "isError": false,
  "data": {
    "page": 1,
    "pageSize": 10,
    "totalCount": 25,
    "items": [
      {
        "productId": 1,
        "productName": "Coca Cola 330ml",
        "totalQuantity": 200,
        "totalRevenue": 2400000
      },
      {
        "productId": 2,
        "productName": "Pepsi 330ml",
        "totalQuantity": 150,
        "totalRevenue": 2250000
      }
    ]
  },
  "statusCode": 200
}
```

**Lưu ý**:
- ✅ Có phân trang (Page, PageSize) — gửi PascalCase khi call API
- ❌ KHÔNG hỗ trợ search, sortBy, sortDesc
- ℹ️ `StartDate`, `EndDate` là optional (PascalCase khi gửi)

### 9.4. GET /api/admin/reports/top-customers - Top Customers

**Authorization**: Admin only | **Hook**: `useApiPaginated`

**Frontend Code**:
```typescript
import { useApiPaginated } from '@/hooks/useApi';
import { reportApiService } from '@/features/reports/api/ReportApiService';
import type { TopCustomerEntity } from '@/features/reports/types/entity';
import type { PagedRequest } from '@/lib/axios';

const { data, isLoading } = useApiPaginated<TopCustomerEntity>({
  apiService: reportApiService,
  entity: 'reports',
  params: {
    page: 1,
    pageSize: 10,
    startDate: '2024-01-01',
    endDate: '2024-01-31',
    // ⚠️ KHÔNG hỗ trợ: search, sortBy, sortDesc
  } as PagedRequest,
});
```

**Response**:
```json
{
  "isError": false,
  "data": {
    "items": [
      {
        "customerId": 1,
        "customerName": "Nguyễn Văn A",
        "orderCount": 15,
        "totalSpent": 1500000
      },
      {
        "customerId": 2,
        "customerName": "Trần Thị B",
        "orderCount": 12,
        "totalSpent": 1200000
      }
    ]
  },
  "statusCode": 200
}
```

---

## 10. AUTHENTICATION MODULE

### 10.1. POST /api/auth/login - Login

**Authorization**: Public (No token required)

**Frontend Code**:
```typescript
import axios from 'axios';

const login = async (username: string, password: string) => {
  try {
    const response = await axios.post('/api/auth/login', {
      username,
      password
    });

    const { accessToken, refreshToken, user } = response.data.data;

    // Store tokens
    localStorage.setItem('accessToken', accessToken);
    localStorage.setItem('refreshToken', refreshToken);

    return user;
  } catch (error) {
    throw error;
  }
};
```

**Request**:
```http
POST /api/auth/login
Content-Type: application/json

{
  "username": "admin",
  "password": "admin123"
}
```

**Response**:
```json
{
  "isError": false,
  "message": "Đăng nhập thành công",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": 1,
      "username": "admin",
      "fullName": "Administrator",
      "role": 0
    }
  },
  "timestamp": "2024-01-20T15:00:00Z",
  "statusCode": 200
}
```

**Token Expiration**:
- Access Token: 1 hour
- Refresh Token: 7 days

### 10.2. POST /api/auth/refresh - Refresh Token

**Authorization**: Public (No token required)

**Frontend Code**:
```typescript
const refreshAccessToken = async () => {
  const refreshToken = localStorage.getItem('refreshToken');

  const response = await axios.post('/api/auth/refresh', {
    refreshToken
  });

  const { accessToken } = response.data.data;
  localStorage.setItem('accessToken', accessToken);

  return accessToken;
};
```

**Request**:
```http
POST /api/auth/refresh
Content-Type: application/json

{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response**:
```json
{
  "isError": false,
  "message": "Làm mới token thành công",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  },
  "statusCode": 200
}
```

### 10.3. POST /api/auth/logout - Logout

**Authorization**: Public (No token required)

**Frontend Code**:
```typescript
const logout = async () => {
  const refreshToken = localStorage.getItem('refreshToken');

  await axios.post('/api/auth/logout', {
    refreshToken
  });

  // Clear tokens
  localStorage.removeItem('accessToken');
  localStorage.removeItem('refreshToken');
};
```

**Request**:
```http
POST /api/auth/logout
Content-Type: application/json

{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response**:
```json
{
  "isError": false,
  "message": "Đăng xuất thành công",
  "data": {},
  "statusCode": 200
}
```

### 10.4. POST /api/auth/setup-admin - Setup First Admin

**Authorization**: Public (No token required)
**Note**: Chỉ hoạt động khi chưa có admin nào trong hệ thống

**Frontend Code**:
```typescript
const setupAdmin = async (username: string, password: string, fullName: string) => {
  const response = await axios.post('/api/auth/setup-admin', {
    username,
    password,
    fullName
  });

  return response.data.data;
};
```

**Request**:
```http
POST /api/auth/setup-admin
Content-Type: application/json

{
  "username": "admin",
  "password": "admin123",
  "fullName": "Administrator"
}
```

**Response**:
```json
{
  "isError": false,
  "message": "Tạo tài khoản admin thành công",
  "data": {
    "id": 1,
    "username": "admin",
    "fullName": "Administrator",
    "role": 0
  },
  "statusCode": 201
}
```

---

## 📊 TỔNG KẾT

### Thống kê Code Examples:

- **10 Modules**: Products, Categories, Customers, Suppliers, Orders, Promotions, Users, Inventory, Reports, Authentication
- **52 Endpoints**: Tất cả đều có code examples đầy đủ
- **Universal Hooks** (từ `@/hooks/useApi`):
  - `useApiList` - GET danh sách không phân trang
  - `useApiPaginated` - GET danh sách có phân trang (core hook)
  - `useApiDetail` - GET chi tiết theo ID
  - `useApiCreate` - POST tạo mới
  - `useApiUpdate` - PUT cập nhật toàn bộ
  - `useApiPatch` - PATCH cập nhật một phần
  - `useApiDelete` - DELETE xóa
  - `useApiCustomQuery` - Custom queries
  - `useApiCustomMutation` - Custom mutations
- **Pagination Hooks**:
  - `usePaginationWithRouter` - URL-based pagination cho Page components (hỗ trợ filters)
  - `usePaginationLocal` - Local state pagination cho Modal/Drawer (hỗ trợ filters)
- **Request/Response**: Đầy đủ HTTP request và JSON response cho mỗi endpoint
- **Authorization**: Rõ ràng cho từng endpoint (Admin, Staff, Admin & Staff, Public)

### Key Highlights:

1. ✅ **Hybrid Approach Pattern**:
   - Tất cả examples sử dụng `BaseApiService` với xử lý `ApiResponse<T>` và `PagedList<T>` tự động
   - Feature-specific services extend `BaseApiService` và thêm custom methods
   - Type-safe với TypeScript generics

2. ✅ **Pagination Strategy**:
   - **Page components**: Sử dụng `usePaginationWithRouter` (URL sync, shareable links, filters support)
   - **Modal/Drawer**: Sử dụng `usePaginationLocal` (local state, filters support)
   - **Nested components**: Sử dụng `useApiPaginated` (nhận params từ props)

3. ✅ **PATCH Endpoints** (2):
   - Orders: Update status only (`useApiCustomMutation`)
   - Inventory: Update quantity (`useApiPatch`)

4. ✅ **Special Endpoints** (7):
   - Order items management (add, update, delete) - `useApiCustomMutation`
   - Invoice download (PDF) - Custom Axios call
   - Low stock alerts - `useApiCustomQuery`
   - Inventory history - `useApiCustomQuery`
   - Promo code validation - `useApiCustomMutation`

5. ✅ **Report Endpoints** (4):
   - Revenue report (non-paginated) - `useApiCustomQuery`
   - Sales report (non-paginated) - `useApiCustomQuery`
   - Top products (paginated, no search/sort) - `useApiPaginated`
   - Top customers (paginated, no search/sort) - `useApiPaginated`

6. ✅ **Authentication** (4):
   - Login - Custom Axios call (không dùng hooks vì public endpoint)
   - Refresh token - Custom Axios call
   - Logout - Custom Axios call
   - Setup first admin - Custom Axios call

### 🎯 Ready for Implementation!

Tất cả code examples đều:
- ✅ Sử dụng **Universal Hooks** từ `@/hooks/useApi`
- ✅ Sử dụng **Pagination Hooks** phù hợp với use case (Page vs Modal/Drawer)
- ✅ Type-safe với TypeScript generics (`ProductEntity`, `CreateProductRequest`, etc.)
- ✅ Tích hợp Ant Design components
- ✅ Error handling đầy đủ với `onSuccess` và `onError` callbacks
- ✅ Loading states (`isPending`, `isLoading`)
- ✅ Form validation
- ✅ Cache invalidation tự động với `invalidateQueries`
- ✅ Khớp 100% với backend API contract

### 📚 Tài liệu liên quan:

- **CustomHookAPIFactory.md**: Kiến trúc Hybrid Approach, BaseApiService, Universal Hooks
- **customHookCallAPIPlan.md**: Hướng dẫn chi tiết về từng hook, pagination patterns, examples

**File này là tài liệu tham khảo hoàn chỉnh để implement frontend cho TapHoaNho project!** 🚀

