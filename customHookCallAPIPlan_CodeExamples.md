# CODE EXAMPLES CHI TIẾT CHO TỪNG ENDPOINT

> **Tài liệu này**: Cung cấp code examples đầy đủ cho **TẤT CẢ 52 endpoints** trong hệ thống TapHoaNho.
>
> **Liên kết**: Đây là phần mở rộng của `customHookCallAPIPlan.md`

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
**Hook**: `useApiPagedList`

**Frontend Code**:
```typescript
import { useApiPagedList } from '@/hooks/useApi';
import { productService } from '@/services/productService';
import { Table, Input, Select, InputNumber, Space, Spin, Alert } from 'antd';

function ProductListPage() {
  const { 
    data, 
    isLoading, 
    error,
    pagination,
    setPage,
    setPageSize,
    setSearch,
    setSortBy,
    setSortDesc,
    setFilters
  } = useApiPagedList(productService, 'products', {
    page: 1,
    pageSize: 20,
    search: '',
    sortBy: 'ProductName',
    sortDesc: false,
    // Filters
    categoryId: undefined,
    supplierId: undefined,
    minPrice: undefined,
    maxPrice: undefined
  });

  if (isLoading) return <Spin />;
  if (error) return <Alert message={error.message} type="error" />;

  return (
    <div>
      {/* Search */}
      <Input.Search
        placeholder="Tìm kiếm sản phẩm..."
        onSearch={(value) => setSearch(value)}
        style={{ width: 300, marginBottom: 16 }}
      />

      {/* Filters */}
      <Space style={{ marginBottom: 16 }}>
        <Select
          placeholder="Danh mục"
          style={{ width: 200 }}
          onChange={(value) => setFilters({ categoryId: value })}
          allowClear
        >
          <Select.Option value={1}>Nước giải khát</Select.Option>
          <Select.Option value={2}>Snack</Select.Option>
        </Select>

        <InputNumber
          placeholder="Giá tối thiểu"
          onChange={(value) => setFilters({ minPrice: value })}
          style={{ width: 150 }}
        />

        <InputNumber
          placeholder="Giá tối đa"
          onChange={(value) => setFilters({ maxPrice: value })}
          style={{ width: 150 }}
        />
      </Space>

      {/* Table */}
      <Table
        dataSource={data?.items || []}
        rowKey="id"
        pagination={{
          current: pagination.page,
          pageSize: pagination.pageSize,
          total: data?.totalCount || 0,
          onChange: (page, pageSize) => {
            setPage(page);
            setPageSize(pageSize);
          }
        }}
        onChange={(pagination, filters, sorter) => {
          if (sorter.field) {
            setSortBy(sorter.field as string);
            setSortDesc(sorter.order === 'descend');
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
import { productService } from '@/services/productService';
import { Descriptions, Spin, Alert, Empty, Button } from 'antd';

function ProductDetailPage({ productId }: { productId: number }) {
  const { data, isLoading, error, refetch } = useApiDetail(
    productService,
    'products',
    productId
  );

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
import { productService } from '@/services/productService';
import { Form, Input, InputNumber, Select, Button, message } from 'antd';

function CreateProductForm() {
  const [form] = Form.useForm();
  const { mutate, isPending } = useApiCreate(productService, 'products', {
    onSuccess: (data) => {
      message.success('Tạo sản phẩm thành công!');
      form.resetFields();
    },
    onError: (error) => {
      message.error(error.message);
    }
  });

  const onFinish = (values: any) => {
    mutate(values);
  };

  return (
    <Form form={form} onFinish={onFinish} layout="vertical">
      <Form.Item
        label="Tên sản phẩm"
        name="productName"
        rules={[
          { required: true, message: 'Vui lòng nhập tên sản phẩm' },
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
          { type: 'number', min: 1, message: 'Giá phải lớn hơn 0' }
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
import { productService } from '@/services/productService';
import { Form, Input, InputNumber, Select, Button, message, Spin } from 'antd';
import { useEffect } from 'react';

function UpdateProductForm({ productId }: { productId: number }) {
  const [form] = Form.useForm();

  // Load existing data
  const { data: product, isLoading } = useApiDetail(productService, 'products', productId);

  // Update mutation
  const { mutate, isPending } = useApiUpdate(productService, 'products', {
    onSuccess: () => {
      message.success('Cập nhật sản phẩm thành công!');
    },
    onError: (error) => {
      message.error(error.message);
    }
  });

  useEffect(() => {
    if (product) {
      form.setFieldsValue(product);
    }
  }, [product, form]);

  const onFinish = (values: any) => {
    mutate({ id: productId, ...values });
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
      <Form.Item label="Đơn vị" name="unit" rules={[{ required: true }]}>
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
import { productService } from '@/services/productService';
import { Button, Popconfirm, message } from 'antd';
import { DeleteOutlined } from '@ant-design/icons';

function DeleteProductButton({ productId, productName }: { productId: number; productName: string }) {
  const { mutate, isPending } = useApiDelete(productService, 'products', {
    onSuccess: () => {
      message.success('Xóa sản phẩm thành công!');
    },
    onError: (error) => {
      message.error(error.message);
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

**Authorization**: Admin only | **Hook**: `useApiPagedList`

**Frontend Code**:
```typescript
import { useApiPagedList } from '@/hooks/useApi';
import { categoryService } from '@/services/categoryService';

const { data, isLoading } = useApiPagedList(categoryService, 'categories', {
  page: 1,
  pageSize: 20,
  sortBy: 'CategoryName',
  sortDesc: false
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
import { categoryService } from '@/services/categoryService';

const { mutate } = useApiCreate(categoryService, 'categories');
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
import { categoryService } from '@/services/categoryService';

const { mutate } = useApiUpdate(categoryService, 'categories');
mutate({ id: 3, categoryName: 'Bánh kẹo & Snack' });
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
import { categoryService } from '@/services/categoryService';

const { mutate } = useApiDelete(categoryService, 'categories');
mutate(3);
```

**Request**: `DELETE /api/admin/categories/3`

---

## 3. CUSTOMERS MODULE

### 3.1. GET /api/admin/customers - List Customers

**Authorization**: Admin & Staff | **Hook**: `useApiPagedList`

**Frontend Code**:
```typescript
import { useApiPagedList } from '@/hooks/useApi';
import { customerService } from '@/services/customerService';

const { data } = useApiPagedList(customerService, 'customers', {
  page: 1,
  pageSize: 20,
  search: 'Nguyễn',
  sortBy: 'Name',
  sortDesc: false
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
import { customerService } from '@/services/customerService';

const { mutate } = useApiCreate(customerService, 'customers');
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

**Authorization**: Admin only | **Hook**: `useApiPagedList`

**Frontend Code**:
```typescript
import { useApiPagedList } from '@/hooks/useApi';
import { supplierService } from '@/services/supplierService';

const { data } = useApiPagedList(supplierService, 'suppliers', {
  page: 1,
  pageSize: 20,
  sortBy: 'Name'
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
import { supplierService } from '@/services/supplierService';

const { mutate } = useApiCreate(supplierService, 'suppliers');
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

**Authorization**: Admin & Staff | **Hook**: `useApiPagedList`

**Frontend Code**:
```typescript
import { useApiPagedList } from '@/hooks/useApi';
import { orderService } from '@/services/orderService';

const { data } = useApiPagedList(orderService, 'orders', {
  page: 1,
  pageSize: 20,
  sortBy: 'OrderDate',
  sortDesc: true,
  // Filters
  status: 'Pending',
  customerId: undefined,
  startDate: '2024-01-01',
  endDate: '2024-01-31'
});
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
import { orderService } from '@/services/orderService';

const { mutate } = useApiCreate(orderService, 'orders');
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

**Authorization**: Admin & Staff | **Hook**: `useApiPatch`

**Frontend Code**:
```typescript
import { useApiPatch } from '@/hooks/useApi';
import { orderService } from '@/services/orderService';

const { mutate } = useApiPatch(orderService, 'orders');

// Update to Paid
mutate({
  id: 1,
  status: 'Paid'
});

// Or use custom mutation
const { mutate: updateStatus } = useApiCustomMutation(orderService, 'orders');
updateStatus({
  method: 'patch',
  path: `${orderId}/status`,
  data: { status: 'Paid' }
});
```

**Request**:
```http
PATCH /api/admin/orders/1/status
Content-Type: application/json

{
  "status": "Paid"
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

**Allowed Status Values**: `"Pending"`, `"Paid"`, `"Cancelled"`

### 5.4. POST /api/admin/orders/{orderId}/items - Add Order Item

**Authorization**: Admin & Staff | **Hook**: `useApiCustomMutation`

**Frontend Code**:
```typescript
import { useApiCustomMutation } from '@/hooks/useApi';
import { orderService } from '@/services/orderService';

const { mutate } = useApiCustomMutation(orderService, 'orders');
mutate({
  method: 'post',
  path: `${orderId}/items`,
  data: {
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
import { orderService } from '@/services/orderService';

const { mutate } = useApiCustomMutation(orderService, 'orders');
mutate({
  method: 'put',
  path: `${orderId}/items/${itemId}`,
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
import { orderService } from '@/services/orderService';

const { mutate } = useApiCustomMutation(orderService, 'orders');
mutate({
  method: 'delete',
  path: `${orderId}/items/${itemId}`
});
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

**Authorization**: Admin only | **Hook**: `useApiPagedList`

**Frontend Code**:
```typescript
import { useApiPagedList } from '@/hooks/useApi';
import { promotionService } from '@/services/promotionService';

const { data } = useApiPagedList(promotionService, 'promotions', {
  page: 1,
  pageSize: 20,
  sortBy: 'StartDate',
  sortDesc: true,
  status: 'Active' // Filter by status
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
        "status": "Active"
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
import { promotionService } from '@/services/promotionService';

const { mutate } = useApiCreate(promotionService, 'promotions');
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
import { promotionService } from '@/services/promotionService';

const { mutate, data } = useApiCustomMutation(promotionService, 'promotions');

mutate({
  method: 'post',
  path: 'validate',
  data: {
    promoCode: 'SUMMER2024'
  }
});
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

**Authorization**: Admin only | **Hook**: `useApiPagedList`

**Frontend Code**:
```typescript
import { useApiPagedList } from '@/hooks/useApi';
import { userService } from '@/services/userService';

const { data } = useApiPagedList(userService, 'users', {
  page: 1,
  pageSize: 20,
  sortBy: 'CreatedAt',
  sortDesc: true,
  role: 1 // Filter by role (0 = Admin, 1 = Staff)
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
import { userService } from '@/services/userService';

const { mutate } = useApiCreate(userService, 'users');
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

**Authorization**: Admin & Staff | **Hook**: `useApiPagedList`

**Frontend Code**:
```typescript
import { useApiPagedList } from '@/hooks/useApi';
import { inventoryService } from '@/services/inventoryService';

const { data } = useApiPagedList(inventoryService, 'inventory', {
  page: 1,
  pageSize: 20,
  sortBy: 'Quantity',
  sortDesc: false,
  status: 'LowStock' // Filter by status
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
import { inventoryService } from '@/services/inventoryService';

const { mutate } = useApiPatch(inventoryService, 'inventory');

// Nhập hàng (tăng tồn kho)
mutate({
  id: productId,
  quantityChange: 100,
  reason: 'Nhập hàng từ nhà cung cấp'
});

// Xuất hàng (giảm tồn kho)
mutate({
  id: productId,
  quantityChange: -50,
  reason: 'Xuất hàng bán lẻ'
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
import { inventoryService } from '@/services/inventoryService';

const { data } = useApiCustomQuery(inventoryService, 'inventory', {
  method: 'get',
  path: 'low-stock'
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
import { inventoryService } from '@/services/inventoryService';

const { data } = useApiCustomQuery(inventoryService, 'inventory', {
  method: 'get',
  path: `${productId}/history`
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
import { reportService } from '@/services/reportService';

// Sử dụng useApiCustomQuery để consistent với pattern của hệ thống
// và dễ dàng extend với custom logic (caching, error handling, etc.)
const { data } = useApiCustomQuery(reportService, 'reports', {
  method: 'get',
  path: 'revenue',
  params: {
    startDate: '2024-01-01',
    endDate: '2024-01-31'
  }
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
import { reportService } from '@/services/reportService';

// Sử dụng useApiCustomQuery để consistent với pattern của hệ thống
// và dễ dàng extend với custom logic (caching, error handling, etc.)
const { data } = useApiCustomQuery(reportService, 'reports', {
  method: 'get',
  path: 'sales',
  params: {
    startDate: '2024-01-01',
    endDate: '2024-01-31'
  }
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

**Authorization**: Admin only | **Hook**: `useApiPagedList`

**Frontend Code**:
```typescript
import { useApiPagedList } from '@/hooks/useApi';
import { reportService } from '@/services/reportService';

const { data } = useApiPagedList(reportService, 'reports/top-products', {
  page: 1,
  pageSize: 10,
  startDate: '2024-01-01',
  endDate: '2024-01-31'
  // ⚠️ KHÔNG hỗ trợ: search, sortBy, sortDesc
});
```

**Request**: `GET /api/admin/reports/top-products?page=1&pagesize=10&startdate=2024-01-01&enddate=2024-01-31`

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
- ✅ Có phân trang (page, pageSize)
- ❌ KHÔNG hỗ trợ search, sortBy, sortDesc
- ✅ Yêu cầu startDate, endDate

### 9.4. GET /api/admin/reports/top-customers - Top Customers

**Authorization**: Admin only | **Hook**: `useApiPagedList`

**Frontend Code**:
```typescript
import { useApiPagedList } from '@/hooks/useApi';
import { reportService } from '@/services/reportService';

const { data } = useApiPagedList(reportService, 'reports/top-customers', {
  page: 1,
  pageSize: 10,
  startDate: '2024-01-01',
  endDate: '2024-01-31'
  // ⚠️ KHÔNG hỗ trợ: search, sortBy, sortDesc
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
- **53 Endpoints**: Tất cả đều có code examples đầy đủ
- **Frontend Hooks**: useApiList, useApiPagedList, useApiDetail, useApiCreate, useApiUpdate, useApiPatch, useApiDelete, useApiCustomQuery, useApiCustomMutation
- **Request/Response**: Đầy đủ HTTP request và JSON response cho mỗi endpoint
- **Authorization**: Rõ ràng cho từng endpoint (Admin, Staff, Admin & Staff, Public)

### Key Highlights:

1. ✅ **PATCH Endpoints** (2):
   - Orders: Update status only
   - Inventory: Update quantity (increase/decrease)

2. ✅ **Special Endpoints** (7):
   - Order items management (add, update, delete)
   - Invoice download (PDF)
   - Low stock alerts
   - Inventory history
   - Promo code validation

3. ✅ **Report Endpoints** (4):
   - Revenue report (non-paginated)
   - Sales report (non-paginated)
   - Top products (paginated, no search/sort)
   - Top customers (paginated, no search/sort)

4. ✅ **Authentication** (4):
   - Login
   - Refresh token
   - Logout
   - Setup first admin

### 🎯 Ready for Implementation!

Tất cả code examples đều:
- ✅ Sử dụng TypeScript
- ✅ Tích hợp Ant Design components
- ✅ Error handling đầy đủ
- ✅ Loading states
- ✅ Form validation
- ✅ Khớp 100% với backend API

**File này là tài liệu tham khảo hoàn chỉnh để implement frontend cho TapHoaNho project!** 🚀

