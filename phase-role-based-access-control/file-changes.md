# File Changes: Role-Based Access Control

## Files mới cần tạo

### 1. Route Guard Utilities
- **Path**: `src/app/routes/utils/routeGuards.ts`
- **Purpose**: Utility functions cho route guards
- **Dependencies**: `useAuthStore`, `@tanstack/react-router`, `type/types`

### 2. Unauthorized Page
- **Path**: `src/features/auth/pages/UnauthorizedPage.tsx`
- **Purpose**: Hiển thị thông báo khi user không có quyền truy cập
- **Dependencies**: Ant Design components

### 3. Menu Configuration
- **Path**: `src/components/layout/menuConfig.ts`
- **Purpose**: Configuration cho menu items với role requirements
- **Dependencies**: None

## Files cần sửa đổi

### 1. Type Definitions
- **Path**: `src/app/routes/type/types.ts`
- **Changes**:
  - Thêm `allowedRoles?: AllowedRoles[]` vào `ManagementRouteDefinition`
  - Thêm `allowedRoles?: AllowedRoles[]` vào `CrudModuleDefinition`
  - Import type `AllowedRoles` từ `routeGuards.ts`

### 2. Route Helpers
- **Path**: `src/app/routes/utils/routeHelpers.ts`
- **Changes**:
  - Import `createRoleGuard` từ `routeGuards.ts`
  - Cập nhật `generateManagementRouteConfigs` để thêm `beforeLoad` guard
  - Cập nhật `generateCRUDRoutesConfigs` để thêm `beforeLoad` guard

### 3. Auth Routes
- **Path**: `src/app/routes/modules/auth.routes.ts`
- **Changes**:
  - Thêm route cho `/auth/unauthorized` page

### 4. Route Definitions (9 files)

#### 4.1. Users Definition
- **Path**: `src/app/routes/modules/management/definition/users.definition.ts`
- **Changes**: Thêm `allowedRoles: ['admin']`

#### 4.2. Products Definition
- **Path**: `src/app/routes/modules/management/definition/products.definition.ts`
- **Changes**: Thêm `allowedRoles: ['admin', 'staff']`

#### 4.3. Categories Definition
- **Path**: `src/app/routes/modules/management/definition/categories.definition.ts`
- **Changes**: Thêm `allowedRoles: ['admin']`

#### 4.4. Suppliers Definition
- **Path**: `src/app/routes/modules/management/definition/suppliers.definition.ts`
- **Changes**: Thêm `allowedRoles: ['admin', 'staff']`

#### 4.5. Customers Definition
- **Path**: `src/app/routes/modules/management/definition/customers.definition.ts`
- **Changes**: Thêm `allowedRoles: ['admin']` (hoặc `['admin', 'staff']` nếu cần)

#### 4.6. Orders Definition
- **Path**: `src/app/routes/modules/management/definition/orders.definition.ts`
- **Changes**: Thêm `allowedRoles: ['admin']` (hoặc `['admin', 'staff']` nếu cần)

#### 4.7. Inventory Definition
- **Path**: `src/app/routes/modules/management/definition/inventory.definition.ts`
- **Changes**: Thêm `allowedRoles: ['admin', 'staff']`

#### 4.8. Promotions Definition
- **Path**: `src/app/routes/modules/management/definition/promotions.definition.ts`
- **Changes**: Thêm `allowedRoles: ['admin']`

#### 4.9. Reports Definition
- **Path**: `src/app/routes/modules/management/definition/reports.definition.ts`
- **Changes**: Thêm `allowedRoles: ['admin']`

### 5. Sidebar Component
- **Path**: `src/components/layout/Sidebar.tsx`
- **Changes**:
  - Import `useAuthStore`, `getMenuRoles` từ `menuConfig.ts`
  - Thêm logic filter menu items dựa trên role
  - Sử dụng `useMemo` để optimize

### 6. Admin Layout (Optional - có thể cần điều chỉnh)
- **Path**: `src/app/routes/layout/admin.layout.tsx`
- **Changes**: 
  - Có thể cần điều chỉnh logic `beforeLoad` để không conflict với route-level guards
  - Hoặc giữ nguyên nếu chỉ check authentication

### 7. Staff Layout (Optional - có thể cần điều chỉnh)
- **Path**: `src/app/routes/layout/staff.layout.tsx`
- **Changes**: 
  - Có thể cần điều chỉnh logic `beforeLoad` để không conflict với route-level guards
  - Hoặc giữ nguyên nếu chỉ check authentication

## Files không cần sửa đổi

- `src/features/auth/store/authStore.ts` - Đã có sẵn role checking
- Các API services - Không ảnh hưởng
- Các feature hooks - Không ảnh hưởng
- Các page components - Không ảnh hưởng

## Summary

- **Files mới**: 3 files
- **Files sửa đổi**: 13 files
- **Total changes**: 16 files

## Migration Notes

1. Tất cả changes đều backward compatible
2. Default behavior: Nếu không có `allowedRoles`, mặc định chỉ Admin truy cập được
3. Không cần migration script cho database
4. Không cần thay đổi API endpoints

