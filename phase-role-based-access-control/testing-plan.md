# Testing Plan: Role-Based Access Control

## Test Strategy

Testing được chia thành 3 loại:
1. **Unit Tests**: Test route guards và utility functions
2. **Integration Tests**: Test route navigation và guards
3. **Manual Tests**: Test UI và user experience

## 1. Unit Tests

### 1.1. Route Guards Tests

**File**: `src/app/routes/utils/__tests__/routeGuards.test.ts`

**Test Cases**:
```typescript
describe('createRoleGuard', () => {
  it('should allow admin access for admin-only routes', () => {
    // Mock admin user
    // Call guard
    // Assert no error thrown
  });

  it('should allow staff access for staff-allowed routes', () => {
    // Mock staff user
    // Call guard
    // Assert no error thrown
  });

  it('should deny staff access for admin-only routes', () => {
    // Mock staff user
    // Call guard
    // Assert redirect thrown
  });

  it('should redirect unauthenticated users to login', () => {
    // Mock unauthenticated user
    // Call guard
    // Assert redirect to /auth/login
  });
});
```

### 1.2. Menu Config Tests

**File**: `src/components/layout/__tests__/menuConfig.test.ts`

**Test Cases**:
```typescript
describe('getMenuRoles', () => {
  it('should return correct roles for each menu item', () => {
    // Test each menu item
  });
});
```

## 2. Integration Tests

### 2.1. Route Navigation Tests

**File**: `src/app/routes/__tests__/routeGuards.integration.test.tsx`

**Test Cases**:
```typescript
describe('Route Guards Integration', () => {
  it('should allow admin to access all admin routes', async () => {
    // Login as admin
    // Navigate to each admin route
    // Assert successful navigation
  });

  it('should allow staff to access allowed routes only', async () => {
    // Login as staff
    // Navigate to allowed routes (products, inventory, suppliers)
    // Assert successful navigation
    // Navigate to denied routes (users, categories, etc.)
    // Assert redirect to /unauthorized
  });

  it('should redirect unauthenticated users', async () => {
    // Logout
    // Navigate to protected route
    // Assert redirect to /auth/login
  });
});
```

## 3. Manual Testing Checklist

### 3.1. Admin User Tests

#### Authentication
- [ ] Đăng nhập với tài khoản Admin thành công
- [ ] User info hiển thị đúng trong Sidebar
- [ ] Logout hoạt động đúng

#### Sidebar Menu
- [ ] Tất cả menu items hiển thị:
  - [ ] Profile
  - [ ] Order (nếu có)
  - [ ] Users
  - [ ] Products
  - [ ] Suppliers
  - [ ] Categories
  - [ ] Customers
  - [ ] Orders
  - [ ] Inventory
  - [ ] Promotions
  - [ ] Reports

#### Route Access
- [ ] Truy cập `/admin/users` thành công
- [ ] Truy cập `/admin/products` thành công
- [ ] Truy cập `/admin/categories` thành công
- [ ] Truy cập `/admin/suppliers` thành công
- [ ] Truy cập `/admin/customers` thành công
- [ ] Truy cập `/admin/orders` thành công
- [ ] Truy cập `/admin/inventory` thành công
- [ ] Truy cập `/admin/promotions` thành công
- [ ] Truy cập `/admin/reports` thành công
- [ ] Truy cập `/auth/profile` thành công

### 3.2. Staff User Tests

#### Authentication
- [ ] Đăng nhập với tài khoản Staff thành công
- [ ] User info hiển thị đúng trong Sidebar
- [ ] Logout hoạt động đúng

#### Sidebar Menu
- [ ] Chỉ hiển thị các menu items sau:
  - [ ] Profile
  - [ ] Order (Staff)
  - [ ] Products
  - [ ] Inventory
  - [ ] Suppliers
- [ ] KHÔNG hiển thị:
  - [ ] Users
  - [ ] Categories
  - [ ] Customers
  - [ ] Orders
  - [ ] Promotions
  - [ ] Reports

#### Route Access - Allowed Routes
- [ ] Truy cập `/staff/order` thành công
- [ ] Truy cập `/admin/products` thành công
- [ ] Truy cập `/admin/inventory` thành công
- [ ] Truy cập `/admin/suppliers` thành công
- [ ] Truy cập `/auth/profile` thành công

#### Route Access - Denied Routes
- [ ] Truy cập `/admin/users` → Redirect đến `/unauthorized`
- [ ] Truy cập `/admin/categories` → Redirect đến `/unauthorized`
- [ ] Truy cập `/admin/customers` → Redirect đến `/unauthorized`
- [ ] Truy cập `/admin/orders` → Redirect đến `/unauthorized`
- [ ] Truy cập `/admin/promotions` → Redirect đến `/unauthorized`
- [ ] Truy cập `/admin/reports` → Redirect đến `/unauthorized`

### 3.3. Unauthenticated User Tests

#### Route Access
- [ ] Truy cập `/admin/users` → Redirect đến `/auth/login`
- [ ] Truy cập `/admin/products` → Redirect đến `/auth/login`
- [ ] Truy cập `/staff/order` → Redirect đến `/auth/login`
- [ ] Truy cập `/auth/profile` → Redirect đến `/auth/login`
- [ ] Truy cập `/auth/login` → Hiển thị login page

### 3.4. Edge Cases

#### Direct URL Access
- [ ] Nhập trực tiếp URL `/admin/users` khi chưa login → Redirect đến login
- [ ] Nhập trực tiếp URL `/admin/users` khi là Staff → Redirect đến unauthorized
- [ ] Nhập trực tiếp URL `/admin/products` khi là Staff → Truy cập thành công

#### Browser Navigation
- [ ] Sử dụng browser back button sau khi bị redirect
- [ ] Sử dụng browser forward button
- [ ] Refresh page khi đang ở protected route

#### Session Management
- [ ] Logout khi đang ở protected route → Redirect đến login
- [ ] Token expire khi đang ở protected route → Redirect đến login
- [ ] Role thay đổi trong session (nếu có) → Menu và routes update

## 4. Test Data

### Admin User
```
Username: admin
Password: [test password]
Role: 0 (Admin)
```

### Staff User
```
Username: staff
Password: [test password]
Role: 1 (Staff)
```

## 5. Test Environment

- **Browser**: Chrome, Firefox, Safari
- **Screen Sizes**: Desktop (1920x1080), Tablet (768x1024), Mobile (375x667)
- **Network**: Normal, Slow 3G, Offline

## 6. Bug Reporting Template

```
**Title**: [Brief description]

**Role**: Admin / Staff / Unauthenticated

**Steps to Reproduce**:
1. ...
2. ...

**Expected Behavior**:
...

**Actual Behavior**:
...

**Screenshots**:
[If applicable]

**Browser/OS**:
...
```

## 7. Acceptance Criteria

✅ **All tests pass**:
- Unit tests: 100% pass
- Integration tests: 100% pass
- Manual tests: All checkboxes checked

✅ **No console errors**:
- No TypeScript errors
- No runtime errors
- No warnings

✅ **Performance**:
- Route guards execute < 50ms
- Menu filtering < 10ms
- No unnecessary re-renders

✅ **User Experience**:
- Clear error messages
- Smooth redirects
- No content flashing

