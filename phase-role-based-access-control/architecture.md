# Architecture: Role-Based Access Control

## Tổng quan kiến trúc

Hệ thống RBAC được xây dựng trên 3 lớp:

1. **Route Guard Layer**: Kiểm tra quyền truy cập ở cấp route
2. **UI Layer**: Ẩn/hiện menu items trong Sidebar dựa trên role
3. **Auth Store Layer**: Quản lý authentication state và role checking (đã có sẵn)

## 1. Route Guard Layer

### 1.1. Route Guard Utility

Tạo utility function để kiểm tra quyền truy cập:

```typescript
// src/app/routes/utils/routeGuards.ts
export type AllowedRoles = 'admin' | 'staff' | 'both';

export function createRoleGuard(allowedRoles: AllowedRoles[]) {
  return (ctx: LoaderContext) => {
    const { isAuthenticated, isAdmin, isStaff } = useAuthStore.getState();
    
    if (!isAuthenticated) {
      throw redirect({ to: '/auth/login' });
    }
    
    const hasAccess = allowedRoles.some(role => {
      if (role === 'admin') return isAdmin();
      if (role === 'staff') return isStaff();
      if (role === 'both') return isAdmin() || isStaff();
      return false;
    });
    
    if (!hasAccess) {
      throw redirect({ to: '/unauthorized' });
    }
  };
}
```

### 1.2. Route Configuration với Guards

Mỗi route definition sẽ có thêm `allowedRoles`:

```typescript
// Example: products.definition.ts
export const productAdminDefinition: ManagementRouteDefinition = {
  // ... existing config
  allowedRoles: ['admin', 'staff'], // Admin và Staff đều truy cập được
};
```

### 1.3. Integration với routeHelpers

Cập nhật `generateManagementRouteConfigs` để tự động thêm `beforeLoad` guard:

```typescript
export function generateManagementRouteConfigs(definition, customConfig) {
  const guard = definition.allowedRoles 
    ? createRoleGuard(definition.allowedRoles)
    : createRoleGuard(['admin']); // Default: chỉ Admin
  
  return [{
    ...config,
    beforeLoad: guard,
  }];
}
```

## 2. UI Layer - Sidebar Menu Filtering

### 2.1. Menu Items Configuration

Tạo configuration object cho menu items với role requirements:

```typescript
// src/components/layout/menuConfig.ts
export const menuItemsConfig = [
  {
    key: 'profile',
    roles: ['admin', 'staff'],
    // ... menu item config
  },
  {
    key: 'order',
    roles: ['staff'],
    // ... menu item config
  },
  {
    key: 'users',
    roles: ['admin'],
    // ... menu item config
  },
  // ...
];
```

### 2.2. Sidebar Component Update

Filter menu items dựa trên user role:

```typescript
const Sidebar = () => {
  const { isAdmin, isStaff } = useAuthStore();
  
  const filteredItems = useMemo(() => {
    return items.filter(item => {
      const requiredRoles = getMenuRoles(item.key);
      return requiredRoles.some(role => {
        if (role === 'admin') return isAdmin();
        if (role === 'staff') return isStaff();
        return false;
      });
    });
  }, [isAdmin, isStaff]);
  
  return <Menu items={filteredItems} />;
};
```

## 3. Route Structure

### 3.1. Admin Routes (chỉ Admin)
- `/admin/users`
- `/admin/categories`
- `/admin/customers`
- `/admin/orders`
- `/admin/promotions`
- `/admin/reports`

### 3.2. Shared Routes (Admin + Staff)
- `/admin/products`
- `/admin/inventory`
- `/admin/suppliers`

### 3.3. Staff Routes (chỉ Staff)
- `/staff/order`

### 3.4. Common Routes (tất cả authenticated users)
- `/auth/profile`

## 4. Flow Diagram

```
User Request → Route Guard (beforeLoad)
                ↓
         Check Authentication
                ↓
         Check Role Permission
                ↓
    ┌───────────┴───────────┐
    │                       │
  Allowed              Denied
    │                       │
    ↓                       ↓
Load Route          Redirect to
                      /unauthorized
```

## 5. Error Handling

### 5.1. Unauthorized Access
- Redirect đến `/unauthorized` page
- Hiển thị thông báo phù hợp

### 5.2. Unauthenticated Access
- Redirect đến `/auth/login`
- Lưu redirect URL để quay lại sau khi login

## 6. Type Safety

Tất cả route guards và menu filtering đều type-safe với TypeScript:

```typescript
type Role = 'admin' | 'staff';
type AllowedRoles = Role | 'both';

interface RouteGuardConfig {
  allowedRoles: AllowedRoles[];
}
```

