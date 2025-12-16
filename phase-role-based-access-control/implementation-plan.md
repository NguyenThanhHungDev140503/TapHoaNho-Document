# Implementation Plan: Role-Based Access Control

## Tổng quan

Kế hoạch triển khai được chia thành 4 giai đoạn chính:

1. **Phase 1**: Tạo Route Guard Utilities
2. **Phase 2**: Cập nhật Route Definitions
3. **Phase 3**: Cập nhật Sidebar Menu
4. **Phase 4**: Testing & Validation

## Phase 1: Tạo Route Guard Utilities

### 1.1. Tạo file `routeGuards.ts`

**File mới**: `src/app/routes/utils/routeGuards.ts`

**Nội dung**:
- Function `createRoleGuard(allowedRoles: AllowedRoles[])`
- Type definitions cho roles
- Helper functions cho role checking

**Dependencies**:
- `useAuthStore` từ `features/auth/store/authStore`
- `redirect` từ `@tanstack/react-router`
- `LoaderContext` từ `type/types`

### 1.2. Tạo Unauthorized Page

**File mới**: `src/features/auth/pages/UnauthorizedPage.tsx`

**Nội dung**:
- Component hiển thị thông báo "Bạn không có quyền truy cập"
- Button quay lại trang chủ hoặc trang trước

### 1.3. Thêm route cho Unauthorized Page

**File sửa**: `src/app/routes/modules/auth.routes.ts`

**Thay đổi**:
- Thêm route `/auth/unauthorized`

## Phase 2: Cập nhật Route Definitions

### 2.1. Cập nhật Type Definitions

**File sửa**: `src/app/routes/type/types.ts`

**Thay đổi**:
- Thêm `allowedRoles?: AllowedRoles[]` vào `ManagementRouteDefinition`
- Thêm `allowedRoles?: AllowedRoles[]` vào `CrudModuleDefinition`

### 2.2. Cập nhật routeHelpers

**File sửa**: `src/app/routes/utils/routeHelpers.ts`

**Thay đổi**:
- Import `createRoleGuard` từ `routeGuards.ts`
- Cập nhật `generateManagementRouteConfigs` để tự động thêm `beforeLoad` guard
- Cập nhật `generateCRUDRoutesConfigs` để tự động thêm `beforeLoad` guard

### 2.3. Cập nhật các Route Definitions

**Files sửa**:
1. `src/app/routes/modules/management/definition/users.definition.ts`
   - `allowedRoles: ['admin']`

2. `src/app/routes/modules/management/definition/products.definition.ts`
   - `allowedRoles: ['admin', 'staff']`

3. `src/app/routes/modules/management/definition/categories.definition.ts`
   - `allowedRoles: ['admin']`

4. `src/app/routes/modules/management/definition/suppliers.definition.ts`
   - `allowedRoles: ['admin', 'staff']`

5. `src/app/routes/modules/management/definition/customers.definition.ts`
   - `allowedRoles: ['admin']` (hoặc có thể là `['admin', 'staff']` nếu cần)

6. `src/app/routes/modules/management/definition/orders.definition.ts`
   - `allowedRoles: ['admin']` (hoặc có thể là `['admin', 'staff']` nếu cần)

7. `src/app/routes/modules/management/definition/inventory.definition.ts`
   - `allowedRoles: ['admin', 'staff']`

8. `src/app/routes/modules/management/definition/promotions.definition.ts`
   - `allowedRoles: ['admin']`

9. `src/app/routes/modules/management/definition/reports.definition.ts`
   - `allowedRoles: ['admin']`

## Phase 3: Cập nhật Sidebar Menu

### 3.1. Tạo Menu Configuration

**File mới**: `src/components/layout/menuConfig.ts`

**Nội dung**:
- Export `menuItemsConfig` với role requirements cho mỗi menu item
- Helper function `getMenuRoles(key: string)`

### 3.2. Cập nhật Sidebar Component

**File sửa**: `src/components/layout/Sidebar.tsx`

**Thay đổi**:
- Import `useAuthStore` và `getMenuRoles`
- Filter menu items dựa trên user role
- Sử dụng `useMemo` để optimize re-renders

**Logic**:
```typescript
const filteredItems = useMemo(() => {
  const { isAdmin, isStaff } = useAuthStore.getState();
  
  return items.filter(item => {
    const requiredRoles = getMenuRoles(item.key);
    return requiredRoles.some(role => {
      if (role === 'admin') return isAdmin();
      if (role === 'staff') return isStaff();
      return false;
    });
  });
}, [isAdmin, isStaff]);
```

## Phase 4: Testing & Validation

### 4.1. Manual Testing Checklist

#### Admin User Tests
- [ ] Đăng nhập với tài khoản Admin
- [ ] Kiểm tra Sidebar hiển thị đầy đủ menu items
- [ ] Truy cập tất cả Management pages thành công
- [ ] Truy cập `/staff/order` - nên redirect hoặc hiển thị lỗi

#### Staff User Tests
- [ ] Đăng nhập với tài khoản Staff
- [ ] Kiểm tra Sidebar chỉ hiển thị:
  - Profile
  - Order (Staff)
  - Products
  - Inventory
  - Suppliers
- [ ] Truy cập `/admin/products` thành công
- [ ] Truy cập `/admin/inventory` thành công
- [ ] Truy cập `/admin/suppliers` thành công
- [ ] Truy cập `/admin/users` - nên redirect đến `/unauthorized`
- [ ] Truy cập `/admin/categories` - nên redirect đến `/unauthorized`
- [ ] Truy cập `/admin/promotions` - nên redirect đến `/unauthorized`
- [ ] Truy cập `/admin/reports` - nên redirect đến `/unauthorized`

#### Unauthenticated User Tests
- [ ] Truy cập bất kỳ protected route nào
- [ ] Nên redirect đến `/auth/login`

### 4.2. Edge Cases

- [ ] User đổi role trong session (nếu có)
- [ ] User logout trong khi đang ở protected route
- [ ] Direct URL access (không qua navigation)
- [ ] Browser back/forward buttons

## Timeline Estimate

- **Phase 1**: 1-2 giờ
- **Phase 2**: 2-3 giờ
- **Phase 3**: 1-2 giờ
- **Phase 4**: 1-2 giờ

**Tổng cộng**: 5-9 giờ

## Risks & Mitigations

### Risk 1: Route guards không hoạt động đúng
**Mitigation**: Test kỹ từng route, đảm bảo `beforeLoad` được gọi đúng cách

### Risk 2: Sidebar menu không update khi role thay đổi
**Mitigation**: Sử dụng `useMemo` với dependencies đúng, đảm bảo re-render khi role thay đổi

### Risk 3: Type safety bị mất
**Mitigation**: Đảm bảo tất cả types được định nghĩa đúng, sử dụng TypeScript strict mode

## Dependencies

- TanStack Router (đã có)
- Zustand (đã có)
- TypeScript (đã có)
- Ant Design (đã có)

## Notes

- Tạm thời cho phép Staff dùng chung route với Admin (`/admin/*`) cho các trang được phép
- Có thể refactor sau để tách route riêng cho Staff nếu cần
- Unauthorized page có thể được cải thiện sau với design tốt hơn

