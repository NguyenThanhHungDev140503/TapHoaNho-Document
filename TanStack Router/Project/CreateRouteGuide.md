## Tổng quan

Dưới đây là tài liệu hướng dẫn từng bước để:

- **Tạo một module route mới** (tương tự `staff.routes.ts` hoặc các module quản lý khác).
- **(Tuỳ chọn) Tạo layout route riêng** (tương tự `auth.layout.tsx`, `admin.layout.tsx`, `staff.layout.tsx`).
- **Đăng ký module mới vào `routeTree.ts`** để router nhận diện và hoạt động.

Mình sẽ dùng ví dụ giả định là tạo module **`sales`** (bán hàng) để minh hoạ. Bạn có thể đổi tên theo nhu cầu thực tế.

---

## 1. Hiểu cấu trúc routing hiện tại

### 1.1. Các khối chính

- **Root route**: `rootRoute` trong `__root.ts` là gốc của toàn bộ cây route.
- **Layout routes**:
  - `authLayoutRoute` (`/auth`) – không sidebar, dùng `AuthLayout`.
  - `mainLayoutRoute` – layout chính có sidebar.
  - `adminLayoutRoute` – layout con dưới `mainLayoutRoute` cho phần admin (`/admin`).
  - `staffLayoutRoute` – layout con dưới `mainLayoutRoute` cho phần staff (`/staff`).
- **Module routes**:
  - Mỗi module cung cấp một object kiểu `ModuleRoutes` trong thư mục `modules/…`, ví dụ:
    - `staffRoutes` (`modules/staff.routes.ts`)
    - `productsRoutes`, `usersRoutes`, … (`modules/management/...`)

- **`routeTree.ts`**:
  - Import tất cả module + layout.
  - Dùng hàm đệ quy `buildRoutesFromConfig` để biến config (`HierarchicalModuleRouteConfig`) thành các route thực tế của TanStack Router.
  - Gắn các route con vào layout tương ứng (`authLayoutRoute`, `adminLayoutRoute`, `staffLayoutRoute`) và cuối cùng vào `rootRoute`.

---

## 2. Tạo file module route mới

Giả sử bạn muốn tạo module **`sales`** dành cho staff (hoặc cho admin – logic tương tự, chỉ khác chỗ đăng ký layout).

### 2.1. Vị trí file

Tuỳ theo loại module:

- **Module dành riêng cho staff** (tương tự `staff.routes.ts`):  
  Bạn có thể đặt ngay trong `modules` (giống `staff.routes.ts`), ví dụ:
  - `frontend/src/app/routes/modules/sales.routes.ts`

- **Module quản trị (admin)**:  
  Đặt trong `modules/management` (giống `products.routes.ts`, `orders.routes.ts`, …), ví dụ:
  - `frontend/src/app/routes/modules/management/sales.routes.ts`

Dưới đây là ví dụ cho **module staff `sales.routes.ts`**.

### 2.2. Nội dung cơ bản của module route

```typescript
// frontend/src/app/routes/modules/sales.routes.ts
import type { ModuleRoutes } from '../type/types';
import { PendingComponent } from '../../../components/feedback/PendingComponent';
import { ErrorComponent } from '../../../components/feedback/ErrorComponent';
import { createRoleGuard } from '../utils/routeGuards';

// Ví dụ các page component
import { SalesDashboardPage } from '../../../features/sales/pages/SalesDashboardPage';
import { SalesOrderDetailPage } from '../../../features/sales/pages/SalesOrderDetailPage';

// Cấu hình module routes với cấu trúc phân cấp
export const salesRoutes: ModuleRoutes<any> = {
  moduleName: 'sales',
  basePath: '/staff', // hoặc '/admin' hoặc '/sales' tuỳ chiến lược URL
  routes: [
    {
      path: '/staff', // Route cha để group các route con cho staff
      children: [
        {
          path: 'sales-dashboard',
          component: SalesDashboardPage,
          pendingComponent: PendingComponent,
          errorComponent: ErrorComponent,
          beforeLoad: createRoleGuard(['staff']), // Chỉ Staff hoặc thêm role khác
          meta: {
            title: 'Bảng điều khiển bán hàng',
            description: 'Trang tổng quan bán hàng cho nhân viên',
            requiresAuth: true,
          },
        },
        {
          path: 'sales-order/:orderId',
          component: SalesOrderDetailPage,
          pendingComponent: PendingComponent,
          errorComponent: ErrorComponent,
          beforeLoad: createRoleGuard(['staff']),
          meta: {
            title: 'Chi tiết đơn hàng bán',
            description: 'Xem chi tiết đơn hàng bán',
            requiresAuth: true,
          },
        },
      ],
    },
  ],
};
```

**Điểm cần chú ý:**

- **`ModuleRoutes`**:
  - `moduleName`: tên định danh module (dùng để debug, logging…).
  - `basePath`: base URL (không bắt buộc trùng path layout, nhưng nên nhất quán).
  - `routes`: mảng các `HierarchicalModuleRouteConfig`.

- **`routes` cấp cao nhất** thường có:
  - Một phần tử có `path` là route cha (`'/staff'`, `'/admin'`, …).
  - `children`: danh sách route con, mỗi route con là 1 đường dẫn tương đối (ví dụ: `'order'`, `'qr-scanner'`…).

- **Guard quyền**:
  - Dùng `createRoleGuard(['staff'])`, `['admin']`, hoặc `['admin', 'staff']` tuỳ module.

- **Meta**: dùng cho SEO, breadcrumb, title, v.v.

---

## 3. (Tuỳ chọn) Tạo layout route mới

Bạn chỉ cần **layout mới** nếu muốn:

- Một **nhánh layout riêng** có cấu trúc khác (khung khác, sidebar khác).
- Hoặc cơ chế guard ở layout-level (như `adminLayoutRoute`, `staffLayoutRoute`).

### 3.1. Ví dụ layout mới cho `sales`

```typescript
// frontend/src/app/routes/layout/sales.layout.tsx
import { createRoute, redirect, Outlet, type AnyRoute } from '@tanstack/react-router';
import { useAuthStore } from '../../../features/auth/store/authStore';

export function createSalesLayoutRoute(parentRoute: AnyRoute) {
  return createRoute({
    getParentRoute: () => parentRoute,
    path: 'sales',            // URL sẽ là '/sales' dưới parent
    component: () => <Outlet />,
    beforeLoad: async () => {
      const { isAuthenticated, user } = useAuthStore.getState();

      if (!isAuthenticated) {
        throw redirect({
          to: '/auth/login' as any,
        });
      }

      // Ví dụ: chỉ Staff được phép vào layout này
      if (!user || user.role !== 1) {
        throw redirect({
          to: '/auth/unauthorized' as any,
        });
      }
    },
  });
}
```

**Ghi chú:**

- `parentRoute` sẽ là `mainLayoutRoute` (giống `adminLayoutRoute`, `staffLayoutRoute`).
- `path: 'sales'` → URL gốc của layout là `/sales`.
- `beforeLoad` chứa guard cho toàn bộ nhánh.

Nếu bạn **không cần layout mới**, bạn có thể:

- Gắn trực tiếp routes vào layout sẵn có (`staffLayoutRoute` hoặc `adminLayoutRoute`), như phần sau trong `routeTree.ts`.

---

## 4. Đăng ký module vào `routeTree.ts`

Đây là bước quan trọng để router thực sự biết và sử dụng module mới.

### 4.1. Import module mới (và layout nếu có)

Mở `frontend/src/app/routes/routeTree.ts`:

1. **Import module routes mới**:

```typescript
import { salesRoutes } from './modules/sales.routes';
```

hoặc nếu bạn đặt trong `management`:

```typescript
import { salesRoutes } from './modules/management/sales.routes';
```

2. **Nếu có layout mới**, import thêm:

```typescript
import { createSalesLayoutRoute } from './layout/sales.layout';
```

### 4.2. Thêm vào mảng module tương ứng

#### Trường hợp 1: Module đi dưới layout `staff` (giống `staffRoutes`)

- Có 2 cách:

**Cách A – Đưa vào `staffModuleRoutes` (dùng chung `staffLayoutRoute`)**

Trong `routeTree.ts` bạn có:

```typescript
// Staff routes (có sidebar)
const staffModuleRoutes: ModuleRoutes<any>[] = [
  staffRoutes
];
```

Sửa thành:

```typescript
const staffModuleRoutes: ModuleRoutes<any>[] = [
  staffRoutes,
  salesRoutes, // thêm module mới
];
```

Lúc này:

- `staffRoutesConfig` và `staffRoutesBuilt` sẽ tự động bao gồm cả routes từ `salesRoutes`.
- Tất cả đều nằm dưới `staffLayoutRoute` (URL: `/staff/...`).

**Cách B – Tạo layout riêng cho sales (nếu bạn đã tạo `sales.layout.tsx`)**

- Ở phần tạo `staffLayoutRoute`, `adminLayoutRoute`, bạn thêm:

```typescript
// Tạo salesLayoutRoute với mainLayoutRoute làm parent
const salesLayoutRoute = createSalesLayoutRoute(mainLayoutRoute);
```

- Sau đó build routes cho module này:

```typescript
const salesRoutesConfig = salesModuleRoutes.flatMap(m => m.routes as HierarchicalModuleRouteConfig[]);
const salesRoutesBuilt = salesRoutesConfig.flatMap(config => {
  if (config.children && config.children.length > 0) {
    return buildRoutesFromConfig(config.children, salesLayoutRoute);
  }
  return buildRoutesFromConfig([config], salesLayoutRoute);
});
salesLayoutRoute.addChildren(salesRoutesBuilt);
```

- Và thêm `salesLayoutRoute` vào `mainLayoutRoute`:

```typescript
mainLayoutRoute.addChildren([adminLayoutRoute, staffLayoutRoute, salesLayoutRoute]);
```

> Cách này phù hợp nếu bạn muốn `/sales/...` là một nhánh layout riêng (không nằm dưới `/staff`).

#### Trường hợp 2: Module đi dưới layout `admin` (giống `productsRoutes`)

- Chỉ cần thêm vào `adminModuleRoutes`:

```typescript
const adminModuleRoutes: ModuleRoutes<any>[] = [
  productsRoutes,
  usersRoutes,
  categoriesRoutes,
  customersRoutes,
  ordersRoutes,
  inventoryRoutes,
  promotionsRoutes,
  reportsRoutes,
  suppliersRoutes,
  salesRoutes, // thêm module mới
];
```

- Không cần thêm logic build riêng, vì đoạn:

```typescript
const adminRoutes = buildRoutesFromConfig(
  adminModuleRoutes.flatMap(m => m.routes as HierarchicalModuleRouteConfig[]),
  adminLayoutRoute,
);
adminLayoutRoute.addChildren(adminRoutes);
```

sẽ tự động xử lý.

### 4.3. Đảm bảo cây route cuối cùng có layout/module mới

Ở cuối file, bạn đã có:

```typescript
const routeTree = rootRoute.addChildren([
  authLayoutRoute,
  mainLayoutRoute,
]);
```

Nếu bạn giữ layout `sales` là con của `mainLayoutRoute` (Cách B), hoặc module `sales` là một phần của `staffModuleRoutes` / `adminModuleRoutes` (Cách A), **không cần sửa phần này** – vì `mainLayoutRoute` đã chứa tất cả layout con (`admin`, `staff`, `sales` nếu có).

---

## 5. Checklist nhanh khi thêm một module route mới

- **[ ]** Tạo file module mới (`*.routes.ts`) với:
  - `moduleName`, `basePath`, `routes` theo chuẩn `ModuleRoutes`.
  - Danh sách `children` với `path`, `component`, `pendingComponent`, `errorComponent`, `beforeLoad`, `meta`.
- **[ ]** (Tuỳ chọn) Tạo layout mới (`*.layout.tsx`) nếu cần nhánh riêng:
  - `createXxxLayoutRoute(parentRoute: AnyRoute)`.
  - `path` phù hợp (`'sales'`, `'reports'`, …).
  - `beforeLoad` để check login/role nếu cần.
- **[ ]** Import module + layout (nếu có) vào `routeTree.ts`.
- **[ ]** Thêm module vào mảng `adminModuleRoutes` hoặc `staffModuleRoutes` (hoặc build riêng với layout mới).
- **[ ]** Đảm bảo gọi `addChildren` đúng parent (`adminLayoutRoute`, `staffLayoutRoute`, `salesLayoutRoute`, …).
- **[ ]** Kiểm tra path:
  - Layout `path`: ví dụ `'staff'` → `/staff`.
  - Module config cha `path`: `'/staff'`.
  - Con `path`: `'order'` → `/staff/order`.

---

Nếu bạn muốn, ở bước tiếp theo mình có thể:

- Đề xuất **cụ thể** cấu trúc cho một module thực tế trong dự án (ví dụ: `returns.routes.ts`, `warehouse.routes.ts`, …).
- Hoặc cùng bạn áp dụng quy trình này để tạo **một module mới thật sự trong code** (mình sẽ trình bày plan sửa code chi tiết và đợi bạn OK trước khi sửa).