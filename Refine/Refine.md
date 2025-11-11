# Refine Framework - Junior & Middle Levels

## Mục Lục

### Junior Level - Cơ Bản
- [1. Giới Thiệu về Refine](#1-giới-thiệu-về-refine)
- [2. Installation và Setup](#2-installation-và-setup)
- [3. Data Providers](#3-data-providers)
- [4. Resources và CRUD Operations](#4-resources-và-crud-operations)
- [5. Basic Hooks](#5-basic-hooks)
- [6. Routing Basics](#6-routing-basics)

### Middle Level - Trung Cấp
- [7. Advanced Data Hooks](#7-advanced-data-hooks)
- [8. Authentication](#8-authentication)
- [9. Access Control](#9-access-control)
- [10. Multiple Data Providers](#10-multiple-data-providers)
- [11. UI Framework Integration](#11-ui-framework-integration)
- [12. Forms và Validation](#12-forms-và-validation)

---

# Junior Level - Cơ Bản

## 1. Giới Thiệu về Refine

### Refine là gì?

**Refine** là một React meta-framework mã nguồn mở được thiết kế đặc biệt cho việc xây dựng các ứng dụng CRUD-heavy như:
- Admin panels
- Dashboards
- Internal tools
- B2B applications
- Data-intensive applications

### Đặc điểm chính

**Headless Architecture:**
- Không bị ràng buộc bởi UI components cụ thể
- Tự do chọn UI framework (Ant Design, Material UI, Chakra UI, hoặc custom)
- Tách biệt logic và presentation

**Hooks-based API:**
- Sử dụng React hooks cho mọi operations
- Type-safe với TypeScript
- Dễ dàng compose và reuse

**Built-in Features:**
- ✅ Data fetching và caching
- ✅ Authentication & Authorization
- ✅ Routing
- ✅ Form management
- ✅ Real-time updates
- ✅ Audit logs
- ✅ i18n support

### So Sánh với Các Framework Khác

| Feature | Refine | React Admin | Admin Bro | Retool |
|---------|--------|-------------|-----------|--------|
| **Open Source** | ✅ | ✅ | ✅ | ❌ |
| **Headless** | ✅ | ❌ | ❌ | ❌ |
| **TypeScript** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **Learning Curve** | Medium | Medium | Easy | Easy |
| **Customization** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ |
| **Data Providers** | 15+ | 10+ | Limited | Built-in |
| **UI Flexibility** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐ |
| **Bundle Size** | Small | Medium | Medium | N/A |

### Khi Nào Dùng Refine?

**✅ Nên dùng khi:**
- Xây dựng admin panels hoặc dashboards
- Cần tích hợp với nhiều backend khác nhau
- Muốn tự do về UI/UX
- Cần type safety với TypeScript
- Xây dựng internal tools phức tạp

**❌ Không nên dùng khi:**
- Ứng dụng đơn giản không cần CRUD
- Chỉ cần static website
- Team không quen với React hooks

---

## 2. Installation và Setup

### Cài Đặt

```bash
# NPM
npm create refine-app@latest my-refine-app

# Yarn
yarn create refine-app my-refine-app

# PNPM
pnpm create refine-app@latest my-refine-app
```

### Interactive Setup

CLI sẽ hỏi các câu hỏi:

```
? Select your project type:
  ❯ Vite
    Next.js
    Remix

? Select your UI framework:
  ❯ Ant Design
    Material UI
    Chakra UI
    Headless

? Select your data provider:
  ❯ REST API
    GraphQL
    Supabase
    Strapi
    NestJS CRUD

? Do you want to add authentication?
  ❯ Yes
    No

? Select your router:
  ❯ React Router v6
    Next.js Router
    Remix Router
```

### Manual Installation

```bash
# Core package
npm install @refinedev/core

# UI framework (choose one)
npm install @refinedev/antd antd
# or
npm install @refinedev/mui @mui/material @emotion/react @emotion/styled

# Router
npm install @refinedev/react-router-v6 react-router-dom

# Data provider
npm install @refinedev/simple-rest
```

### Basic App Structure

```typescript
// Example 1: Minimal Refine app
import { Refine } from "@refinedev/core";
import { BrowserRouter, Routes, Route, Outlet } from "react-router-dom";
import routerProvider from "@refinedev/react-router-v6";
import dataProvider from "@refinedev/simple-rest";

function App() {
  return (
    <BrowserRouter>
      <Refine
        dataProvider={dataProvider("https://api.fake-rest.refine.dev")}
        routerProvider={routerProvider}
        resources={[
          {
            name: "products",
            list: "/products",
          },
        ]}
      >
        <Routes>
          <Route path="/products" element={<ProductList />} />
        </Routes>
      </Refine>
    </BrowserRouter>
  );
}
```

**Giải thích:**
- `<Refine>`: Component chính, wrap toàn bộ app
- `dataProvider`: Kết nối với API backend
- `routerProvider`: Quản lý routing
- `resources`: Định nghĩa các resources (products, users, etc.)

---

## 3. Data Providers

### Data Provider là gì?

Data Provider là một adapter kết nối Refine với backend API. Nó định nghĩa cách thức:
- Fetch data (GET)
- Create records (POST)
- Update records (PUT/PATCH)
- Delete records (DELETE)

### Built-in Data Providers

Refine hỗ trợ 15+ data providers:

```typescript
// REST API
import dataProvider from "@refinedev/simple-rest";

// GraphQL
import { GraphQLClient } from "graphql-request";
import dataProvider from "@refinedev/graphql";

// Supabase
import { dataProvider } from "@refinedev/supabase";
import { supabaseClient } from "./supabaseClient";

// Strapi
import { DataProvider } from "@refinedev/strapi-v4";

// NestJS CRUD
import dataProvider from "@refinedev/nestjsx-crud";
```

### Simple REST Data Provider

```typescript
// Example 2: Using REST API data provider
import { Refine } from "@refinedev/core";
import dataProvider from "@refinedev/simple-rest";

const API_URL = "https://api.fake-rest.refine.dev";

function App() {
  return (
    <Refine
      dataProvider={dataProvider(API_URL)}
      resources={[
        {
          name: "posts",
          list: "/posts",
          show: "/posts/show/:id",
          create: "/posts/create",
          edit: "/posts/edit/:id",
        },
      ]}
    >
      {/* Routes */}
    </Refine>
  );
}
```

**Data Provider Methods:**

Data provider phải implement các methods sau:

```typescript
interface DataProvider {
  // Fetch list of records
  getList: (params) => Promise<{ data: any[]; total: number }>;

  // Fetch single record
  getOne: (params) => Promise<{ data: any }>;

  // Fetch multiple records by IDs
  getMany: (params) => Promise<{ data: any[] }>;

  // Create new record
  create: (params) => Promise<{ data: any }>;

  // Update existing record
  update: (params) => Promise<{ data: any }>;

  // Delete record
  deleteOne: (params) => Promise<{ data: any }>;

  // Get API URL
  getApiUrl: () => string;

  // Custom requests
  custom?: (params) => Promise<{ data: any }>;
}
```

### Custom Data Provider

```typescript
// Example 3: Creating custom data provider
import { DataProvider } from "@refinedev/core";
import axios from "axios";

export const customDataProvider = (apiUrl: string): DataProvider => ({
  getList: async ({ resource, pagination, filters, sorters }) => {
    const { current = 1, pageSize = 10 } = pagination ?? {};

    const query = {
      _start: (current - 1) * pageSize,
      _end: current * pageSize,
    };

    const url = `${apiUrl}/${resource}`;
    const { data, headers } = await axios.get(url, { params: query });

    const total = Number(headers["x-total-count"]);

    return {
      data,
      total,
    };
  },

  getOne: async ({ resource, id }) => {
    const url = `${apiUrl}/${resource}/${id}`;
    const { data } = await axios.get(url);

    return { data };
  },

  create: async ({ resource, variables }) => {
    const url = `${apiUrl}/${resource}`;
    const { data } = await axios.post(url, variables);

    return { data };
  },

  update: async ({ resource, id, variables }) => {
    const url = `${apiUrl}/${resource}/${id}`;
    const { data } = await axios.patch(url, variables);

    return { data };
  },

  deleteOne: async ({ resource, id }) => {
    const url = `${apiUrl}/${resource}/${id}`;
    const { data } = await axios.delete(url);

    return { data };
  },

  getApiUrl: () => apiUrl,
});
```

**Giải thích:**
- `getList`: Fetch danh sách với pagination, filters, sorting
- `getOne`: Fetch 1 record theo ID
- `create`: Tạo record mới
- `update`: Cập nhật record
- `deleteOne`: Xóa record
- `getApiUrl`: Trả về base URL của API

---

## 4. Resources và CRUD Operations

### Định Nghĩa Resources

Resource là một entity trong ứng dụng (products, users, posts, etc.)

```typescript
// Example 4: Defining resources
<Refine
  dataProvider={dataProvider(API_URL)}
  resources={[
    {
      name: "products",           // Resource name (matches API endpoint)
      list: "/products",          // List page route
      show: "/products/show/:id", // Detail page route
      create: "/products/create", // Create page route
      edit: "/products/edit/:id", // Edit page route
      meta: {
        label: "Products",        // Display name
        icon: <ShoppingOutlined />, // Icon (optional)
      },
    },
    {
      name: "categories",
      list: "/categories",
      create: "/categories/create",
      edit: "/categories/edit/:id",
    },
  ]}
>
  {/* Routes */}
</Refine>
```

### CRUD Pages

```typescript
// Example 5: Basic CRUD pages
import { useTable, useMany } from "@refinedev/core";

// LIST PAGE
export const ProductList = () => {
  const { tableQuery } = useTable<IProduct>({
    resource: "products",
  });

  const { data, isLoading } = tableQuery;

  if (isLoading) return <div>Loading...</div>;

  return (
    <div>
      <h1>Products</h1>
      <table>
        <thead>
          <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Price</th>
          </tr>
        </thead>
        <tbody>
          {data?.data.map((product) => (
            <tr key={product.id}>
              <td>{product.id}</td>
              <td>{product.name}</td>
              <td>${product.price}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
};

// SHOW PAGE
import { useShow } from "@refinedev/core";

export const ProductShow = () => {
  const { query } = useShow<IProduct>();
  const { data, isLoading } = query;

  if (isLoading) return <div>Loading...</div>;

  const product = data?.data;

  return (
    <div>
      <h1>{product?.name}</h1>
      <p>Price: ${product?.price}</p>
      <p>Description: {product?.description}</p>
    </div>
  );
};

// CREATE PAGE
import { useForm } from "@refinedev/react-hook-form";

export const ProductCreate = () => {
  const {
    refineCore: { onFinish },
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<IProduct>();

  return (
    <form onSubmit={handleSubmit(onFinish)}>
      <label>Name:</label>
      <input {...register("name", { required: true })} />
      {errors.name && <span>This field is required</span>}

      <label>Price:</label>
      <input type="number" {...register("price", { required: true })} />
      {errors.price && <span>This field is required</span>}

      <button type="submit">Create</button>
    </form>
  );
};

// EDIT PAGE
export const ProductEdit = () => {
  const {
    refineCore: { onFinish, query },
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<IProduct>();

  const { isLoading } = query;

  if (isLoading) return <div>Loading...</div>;

  return (
    <form onSubmit={handleSubmit(onFinish)}>
      <label>Name:</label>
      <input {...register("name", { required: true })} />

      <label>Price:</label>
      <input type="number" {...register("price", { required: true })} />

      <button type="submit">Update</button>
    </form>
  );
};
```

**Giải thích:**
- `useTable`: Hook để fetch list data với pagination, sorting, filtering
- `useShow`: Hook để fetch single record
- `useForm`: Hook để handle form (create/edit)
- `onFinish`: Function để submit form data

---

## 5. Basic Hooks

### useList Hook

```typescript
// Example 6: Using useList hook
import { useList } from "@refinedev/core";

export const ProductList = () => {
  const { data, isLoading, isError } = useList<IProduct>({
    resource: "products",
    pagination: {
      current: 1,
      pageSize: 10,
    },
    sorters: [
      {
        field: "createdAt",
        order: "desc",
      },
    ],
    filters: [
      {
        field: "category",
        operator: "eq",
        value: "electronics",
      },
    ],
  });

  if (isLoading) return <div>Loading...</div>;
  if (isError) return <div>Error loading products</div>;

  return (
    <ul>
      {data?.data.map((product) => (
        <li key={product.id}>{product.name}</li>
      ))}
    </ul>
  );
};
```

### useOne Hook

```typescript
// Example 7: Using useOne hook
import { useOne } from "@refinedev/core";

export const ProductDetail = ({ id }: { id: string }) => {
  const { data, isLoading } = useOne<IProduct>({
    resource: "products",
    id,
  });

  if (isLoading) return <div>Loading...</div>;

  const product = data?.data;

  return (
    <div>
      <h2>{product?.name}</h2>
      <p>Price: ${product?.price}</p>
    </div>
  );
};
```

### useCreate Hook

```typescript
// Example 8: Using useCreate hook
import { useCreate } from "@refinedev/core";

export const CreateProductButton = () => {
  const { mutate, isLoading } = useCreate<IProduct>();

  const handleCreate = () => {
    mutate({
      resource: "products",
      values: {
        name: "New Product",
        price: 99.99,
        category: "electronics",
      },
    });
  };

  return (
    <button onClick={handleCreate} disabled={isLoading}>
      {isLoading ? "Creating..." : "Create Product"}
    </button>
  );
};
```

### useUpdate Hook

```typescript
// Example 9: Using useUpdate hook
import { useUpdate } from "@refinedev/core";

export const UpdateProductButton = ({ id }: { id: string }) => {
  const { mutate, isLoading } = useUpdate<IProduct>();

  const handleUpdate = () => {
    mutate({
      resource: "products",
      id,
      values: {
        price: 79.99,
      },
    });
  };

  return (
    <button onClick={handleUpdate} disabled={isLoading}>
      {isLoading ? "Updating..." : "Update Price"}
    </button>
  );
};
```

### useDelete Hook

```typescript
// Example 10: Using useDelete hook
import { useDelete } from "@refinedev/core";

export const DeleteProductButton = ({ id }: { id: string }) => {
  const { mutate, isLoading } = useDelete();

  const handleDelete = () => {
    if (confirm("Are you sure?")) {
      mutate({
        resource: "products",
        id,
      });
    }
  };

  return (
    <button onClick={handleDelete} disabled={isLoading}>
      {isLoading ? "Deleting..." : "Delete"}
    </button>
  );
};
```

---

## 6. Routing Basics

### React Router Integration

```typescript
// Example 11: Complete routing setup
import { Refine } from "@refinedev/core";
import { BrowserRouter, Routes, Route, Outlet } from "react-router-dom";
import routerProvider from "@refinedev/react-router-v6";
import dataProvider from "@refinedev/simple-rest";

import { ProductList, ProductShow, ProductCreate, ProductEdit } from "./pages/products";
import { Layout } from "./components/Layout";

function App() {
  return (
    <BrowserRouter>
      <Refine
        dataProvider={dataProvider("https://api.fake-rest.refine.dev")}
        routerProvider={routerProvider}
        resources={[
          {
            name: "products",
            list: "/products",
            show: "/products/show/:id",
            create: "/products/create",
            edit: "/products/edit/:id",
          },
        ]}
      >
        <Routes>
          <Route
            element={
              <Layout>
                <Outlet />
              </Layout>
            }
          >
            <Route path="/products">
              <Route index element={<ProductList />} />
              <Route path="show/:id" element={<ProductShow />} />
              <Route path="create" element={<ProductCreate />} />
              <Route path="edit/:id" element={<ProductEdit />} />
            </Route>
          </Route>
        </Routes>
      </Refine>
    </BrowserRouter>
  );
}
```

### Navigation Hooks

```typescript
// Example 12: Using navigation hooks
import { useNavigation, useGo } from "@refinedev/core";

export const ProductActions = ({ id }: { id: string }) => {
  const { show, edit, list } = useNavigation();
  const go = useGo();

  return (
    <div>
      {/* Navigate to show page */}
      <button onClick={() => show("products", id)}>
        View Details
      </button>

      {/* Navigate to edit page */}
      <button onClick={() => edit("products", id)}>
        Edit
      </button>

      {/* Navigate to list page */}
      <button onClick={() => list("products")}>
        Back to List
      </button>

      {/* Custom navigation */}
      <button onClick={() => go({ to: "/custom-page" })}>
        Custom Page
      </button>
    </div>
  );
};
```

---

# Middle Level - Trung Cấp

## 7. Advanced Data Hooks

### useTable Hook

```typescript
// Example 13: Advanced useTable with filters and sorting
import { useTable } from "@refinedev/core";
import { useState } from "react";

export const ProductTable = () => {
  const {
    tableQuery: { data, isLoading },
    current,
    setCurrent,
    pageSize,
    setPageSize,
    sorters,
    setSorters,
    filters,
    setFilters,
  } = useTable<IProduct>({
    resource: "products",
    pagination: {
      current: 1,
      pageSize: 10,
    },
    sorters: {
      initial: [
        {
          field: "createdAt",
          order: "desc",
        },
      ],
    },
    filters: {
      initial: [
        {
          field: "category",
          operator: "eq",
          value: "electronics",
        },
      ],
    },
  });

  const handleSort = (field: string) => {
    const currentSorter = sorters.find((s) => s.field === field);
    const newOrder = currentSorter?.order === "asc" ? "desc" : "asc";

    setSorters([{ field, order: newOrder }]);
  };

  const handleFilter = (field: string, value: string) => {
    setFilters([
      {
        field,
        operator: "contains",
        value,
      },
    ]);
  };

  if (isLoading) return <div>Loading...</div>;

  return (
    <div>
      {/* Search filter */}
      <input
        type="text"
        placeholder="Search by name..."
        onChange={(e) => handleFilter("name", e.target.value)}
      />

      {/* Table */}
      <table>
        <thead>
          <tr>
            <th onClick={() => handleSort("id")}>
              ID {sorters.find((s) => s.field === "id")?.order === "asc" ? "↑" : "↓"}
            </th>
            <th onClick={() => handleSort("name")}>
              Name {sorters.find((s) => s.field === "name")?.order === "asc" ? "↑" : "↓"}
            </th>
            <th onClick={() => handleSort("price")}>
              Price {sorters.find((s) => s.field === "price")?.order === "asc" ? "↑" : "↓"}
            </th>
          </tr>
        </thead>
        <tbody>
          {data?.data.map((product) => (
            <tr key={product.id}>
              <td>{product.id}</td>
              <td>{product.name}</td>
              <td>${product.price}</td>
            </tr>
          ))}
        </tbody>
      </table>

      {/* Pagination */}
      <div>
        <button
          onClick={() => setCurrent(current - 1)}
          disabled={current === 1}
        >
          Previous
        </button>
        <span>Page {current}</span>
        <button onClick={() => setCurrent(current + 1)}>
          Next
        </button>

        <select
          value={pageSize}
          onChange={(e) => setPageSize(Number(e.target.value))}
        >
          <option value={10}>10 per page</option>
          <option value={20}>20 per page</option>
          <option value={50}>50 per page</option>
        </select>
      </div>
    </div>
  );
};
```

**Giải thích:**
- `tableQuery`: Chứa data và loading state
- `current`, `setCurrent`: Current page và setter
- `pageSize`, `setPageSize`: Page size và setter
- `sorters`, `setSorters`: Sorting state và setter
- `filters`, `setFilters`: Filters state và setter

### useInfiniteList Hook

```typescript
// Example 14: Infinite scroll with useInfiniteList
import { useInfiniteList } from "@refinedev/core";
import { useEffect, useRef } from "react";

export const InfiniteProductList = () => {
  const {
    data,
    isLoading,
    hasNextPage,
    fetchNextPage,
    isFetchingNextPage,
  } = useInfiniteList<IProduct>({
    resource: "products",
    pagination: {
      pageSize: 20,
    },
  });

  const observerTarget = useRef(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting && hasNextPage && !isFetchingNextPage) {
          fetchNextPage();
        }
      },
      { threshold: 1 }
    );

    if (observerTarget.current) {
      observer.observe(observerTarget.current);
    }

    return () => observer.disconnect();
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);

  if (isLoading) return <div>Loading...</div>;

  const allProducts = data?.pages.flatMap((page) => page.data) ?? [];

  return (
    <div>
      <div>
        {allProducts.map((product) => (
          <div key={product.id}>
            <h3>{product.name}</h3>
            <p>${product.price}</p>
          </div>
        ))}
      </div>

      {/* Intersection observer target */}
      <div ref={observerTarget} style={{ height: "20px" }}>
        {isFetchingNextPage && <div>Loading more...</div>}
      </div>
    </div>
  );
};
```

---

## 8. Authentication

### Auth Provider

```typescript
// Example 15: Creating auth provider
import { AuthProvider } from "@refinedev/core";
import axios from "axios";

export const authProvider: AuthProvider = {
  // Login
  login: async ({ email, password }) => {
    try {
      const { data } = await axios.post("/api/login", { email, password });

      localStorage.setItem("token", data.token);
      localStorage.setItem("user", JSON.stringify(data.user));

      return {
        success: true,
        redirectTo: "/",
      };
    } catch (error) {
      return {
        success: false,
        error: {
          name: "LoginError",
          message: "Invalid email or password",
        },
      };
    }
  },

  // Logout
  logout: async () => {
    localStorage.removeItem("token");
    localStorage.removeItem("user");

    return {
      success: true,
      redirectTo: "/login",
    };
  },

  // Check authentication
  check: async () => {
    const token = localStorage.getItem("token");

    if (token) {
      return {
        authenticated: true,
      };
    }

    return {
      authenticated: false,
      redirectTo: "/login",
      logout: true,
    };
  },

  // Get user identity
  getIdentity: async () => {
    const user = localStorage.getItem("user");

    if (user) {
      return JSON.parse(user);
    }

    return null;
  },

  // Handle errors
  onError: async (error) => {
    if (error.status === 401 || error.status === 403) {
      return {
        logout: true,
        redirectTo: "/login",
        error,
      };
    }

    return { error };
  },
};

// Usage in App
import { Refine, Authenticated } from "@refinedev/core";

function App() {
  return (
    <Refine
      authProvider={authProvider}
      dataProvider={dataProvider(API_URL)}
      // ...
    >
      <Routes>
        <Route
          element={
            <Authenticated fallback={<Navigate to="/login" />}>
              <Layout>
                <Outlet />
              </Layout>
            </Authenticated>
          }
        >
          {/* Protected routes */}
        </Route>

        <Route path="/login" element={<LoginPage />} />
      </Routes>
    </Refine>
  );
}
```

### useLogin Hook

```typescript
// Example 16: Login page with useLogin
import { useLogin } from "@refinedev/core";
import { useState } from "react";

export const LoginPage = () => {
  const { mutate: login, isLoading } = useLogin();
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();

    login({ email, password });
  };

  return (
    <form onSubmit={handleSubmit}>
      <h1>Login</h1>

      <input
        type="email"
        placeholder="Email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        required
      />

      <input
        type="password"
        placeholder="Password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        required
      />

      <button type="submit" disabled={isLoading}>
        {isLoading ? "Logging in..." : "Login"}
      </button>
    </form>
  );
};
```

### useGetIdentity Hook

```typescript
// Example 17: Display user info with useGetIdentity
import { useGetIdentity } from "@refinedev/core";

interface IUser {
  id: string;
  name: string;
  email: string;
  avatar?: string;
}

export const UserProfile = () => {
  const { data: user, isLoading } = useGetIdentity<IUser>();

  if (isLoading) return <div>Loading...</div>;

  return (
    <div>
      {user?.avatar && <img src={user.avatar} alt={user.name} />}
      <h2>{user?.name}</h2>
      <p>{user?.email}</p>
    </div>
  );
};
```

---

## 9. Access Control

### Access Control Provider

```typescript
// Example 18: Creating access control provider
import { AccessControlProvider } from "@refinedev/core";

export const accessControlProvider: AccessControlProvider = {
  can: async ({ resource, action, params }) => {
    const user = JSON.parse(localStorage.getItem("user") || "{}");
    const userRole = user.role; // "admin", "editor", "viewer"

    // Admin has full access
    if (userRole === "admin") {
      return { can: true };
    }

    // Editor can create, edit, but not delete
    if (userRole === "editor") {
      if (action === "delete") {
        return {
          can: false,
          reason: "Only admins can delete",
        };
      }
      return { can: true };
    }

    // Viewer can only list and show
    if (userRole === "viewer") {
      if (action === "list" || action === "show") {
        return { can: true };
      }
      return {
        can: false,
        reason: "You don't have permission",
      };
    }

    return { can: false };
  },
};

// Usage in App
<Refine
  authProvider={authProvider}
  accessControlProvider={accessControlProvider}
  dataProvider={dataProvider(API_URL)}
  // ...
/>
```

### useCan Hook

```typescript
// Example 19: Conditional rendering with useCan
import { useCan } from "@refinedev/core";

export const ProductActions = ({ id }: { id: string }) => {
  const { data: canEdit } = useCan({
    resource: "products",
    action: "edit",
    params: { id },
  });

  const { data: canDelete } = useCan({
    resource: "products",
    action: "delete",
    params: { id },
  });

  return (
    <div>
      {canEdit?.can && (
        <button>Edit</button>
      )}

      {canDelete?.can && (
        <button>Delete</button>
      )}

      {!canDelete?.can && canDelete?.reason && (
        <p>{canDelete.reason}</p>
      )}
    </div>
  );
};
```

---

## 10. Multiple Data Providers

```typescript
// Example 20: Using multiple data providers
import { Refine } from "@refinedev/core";
import dataProvider from "@refinedev/simple-rest";

const API_URL_1 = "https://api1.example.com";
const API_URL_2 = "https://api2.example.com";

function App() {
  return (
    <Refine
      dataProvider={{
        default: dataProvider(API_URL_1),
        api2: dataProvider(API_URL_2),
      }}
      resources={[
        {
          name: "products",
          // Uses default data provider
        },
        {
          name: "users",
          meta: {
            dataProviderName: "api2", // Uses api2 data provider
          },
        },
      ]}
    >
      {/* Routes */}
    </Refine>
  );
}

// Using specific data provider in hooks
import { useList } from "@refinedev/core";

export const UserList = () => {
  const { data } = useList({
    resource: "users",
    meta: {
      dataProviderName: "api2",
    },
  });

  // ...
};
```

---

## 11. UI Framework Integration

### Ant Design Integration

```typescript
// Example 21: Refine with Ant Design
import { Refine } from "@refinedev/core";
import { ThemedLayoutV2, RefineThemes } from "@refinedev/antd";
import { ConfigProvider } from "antd";
import dataProvider from "@refinedev/simple-rest";

import "@refinedev/antd/dist/reset.css";

function App() {
  return (
    <ConfigProvider theme={RefineThemes.Blue}>
      <Refine
        dataProvider={dataProvider(API_URL)}
        resources={[
          {
            name: "products",
            list: "/products",
            create: "/products/create",
            edit: "/products/edit/:id",
            show: "/products/show/:id",
          },
        ]}
      >
        <Routes>
          <Route
            element={
              <ThemedLayoutV2>
                <Outlet />
              </ThemedLayoutV2>
            }
          >
            <Route path="/products">
              <Route index element={<ProductList />} />
              <Route path="create" element={<ProductCreate />} />
              <Route path="edit/:id" element={<ProductEdit />} />
              <Route path="show/:id" element={<ProductShow />} />
            </Route>
          </Route>
        </Routes>
      </Refine>
    </ConfigProvider>
  );
}

// Using Ant Design components with Refine hooks
import { List, Table, Space, EditButton, ShowButton, DeleteButton } from "@refinedev/antd";
import { useTable } from "@refinedev/antd";

export const ProductList = () => {
  const { tableProps } = useTable<IProduct>();

  return (
    <List>
      <Table {...tableProps} rowKey="id">
        <Table.Column dataIndex="id" title="ID" />
        <Table.Column dataIndex="name" title="Name" />
        <Table.Column dataIndex="price" title="Price" />
        <Table.Column
          title="Actions"
          render={(_, record) => (
            <Space>
              <EditButton hideText size="small" recordItemId={record.id} />
              <ShowButton hideText size="small" recordItemId={record.id} />
              <DeleteButton hideText size="small" recordItemId={record.id} />
            </Space>
          )}
        />
      </Table>
    </List>
  );
};
```

---

## 12. Forms và Validation

```typescript
// Example 22: Form with validation using react-hook-form
import { useForm } from "@refinedev/react-hook-form";
import { z } from "zod";
import { zodResolver } from "@hookform/resolvers/zod";

const productSchema = z.object({
  name: z.string().min(3, "Name must be at least 3 characters"),
  price: z.number().positive("Price must be positive"),
  category: z.string().min(1, "Category is required"),
  description: z.string().optional(),
});

type ProductFormValues = z.infer<typeof productSchema>;

export const ProductCreate = () => {
  const {
    refineCore: { onFinish, formLoading },
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<ProductFormValues>({
    resolver: zodResolver(productSchema),
  });

  return (
    <form onSubmit={handleSubmit(onFinish)}>
      <div>
        <label>Name:</label>
        <input {...register("name")} />
        {errors.name && <span>{errors.name.message}</span>}
      </div>

      <div>
        <label>Price:</label>
        <input type="number" {...register("price", { valueAsNumber: true })} />
        {errors.price && <span>{errors.price.message}</span>}
      </div>

      <div>
        <label>Category:</label>
        <select {...register("category")}>
          <option value="">Select category</option>
          <option value="electronics">Electronics</option>
          <option value="clothing">Clothing</option>
          <option value="books">Books</option>
        </select>
        {errors.category && <span>{errors.category.message}</span>}
      </div>

      <div>
        <label>Description:</label>
        <textarea {...register("description")} />
      </div>

      <button type="submit" disabled={formLoading}>
        {formLoading ? "Creating..." : "Create Product"}
      </button>
    </form>
  );
};
```

---

## Best Practices - Junior & Middle Levels

### 1. Resource Naming

```typescript
// ✅ GOOD: Consistent naming
resources={[
  { name: "products", list: "/products" },
  { name: "categories", list: "/categories" },
]}

// ❌ BAD: Inconsistent naming
resources={[
  { name: "product", list: "/products" },
  { name: "category", list: "/category" },
]}
```

### 2. Error Handling

```typescript
// ✅ GOOD: Handle errors properly
const { data, isLoading, isError, error } = useList();

if (isLoading) return <div>Loading...</div>;
if (isError) return <div>Error: {error?.message}</div>;

// ❌ BAD: No error handling
const { data } = useList();
return <div>{data?.data.map(...)}</div>;
```

### 3. Type Safety

```typescript
// ✅ GOOD: Use TypeScript interfaces
interface IProduct {
  id: string;
  name: string;
  price: number;
}

const { data } = useList<IProduct>();

// ❌ BAD: No types
const { data } = useList();
```

---

## Common Pitfalls

### 1. Forgetting Resource Name

```typescript
// ❌ BAD: Missing resource
const { data } = useList();

// ✅ GOOD: Specify resource
const { data } = useList({ resource: "products" });
```

### 2. Not Handling Loading States

```typescript
// ❌ BAD: No loading state
const { data } = useList();
return <div>{data?.data.map(...)}</div>;

// ✅ GOOD: Handle loading
const { data, isLoading } = useList();
if (isLoading) return <div>Loading...</div>;
return <div>{data?.data.map(...)}</div>;
```

### 3. Incorrect Data Provider Setup

```typescript
// ❌ BAD: Wrong API URL format
dataProvider("api.example.com")

// ✅ GOOD: Complete URL
dataProvider("https://api.example.com")
```

---

## Tài Liệu Tham Khảo

### Official Documentation
- [Refine Docs](https://refine.dev/docs/)
- [Refine GitHub](https://github.com/refinedev/refine)
- [API Reference](https://refine.dev/docs/api-reference/core/)

### Next Level
- Xem `Advanced-Refine-Patterns.md` cho Senior level patterns
- Xem `Principal-Refine-Patterns.md` cho Enterprise patterns

---

**Tài liệu này bao gồm Junior và Middle levels. Để học các patterns nâng cao, hãy tiếp tục với Advanced-Refine-Patterns.md.**
