# Phase: Role-Based Access Control (RBAC) Implementation

## Mục tiêu
Thiết kế và triển khai hệ thống phân quyền dựa trên vai trò (Role-Based Access Control) cho ứng dụng Store Management System.

## Yêu cầu phân quyền

### Admin (Quản trị viên)
- ✅ Truy cập được các trang chung cho User (Profile, Login, etc.)
- ✅ Truy cập được **TẤT CẢ** các trang Management:
  - Users
  - Products
  - Categories
  - Suppliers
  - Customers
  - Orders
  - Inventory
  - Promotions
  - Reports

### Staff (Nhân viên)
- ✅ Truy cập được các trang chung cho User (Profile, Login, etc.)
- ✅ Truy cập được trang Staff Order (`/staff/order`)
- ✅ Truy cập được các trang Management sau:
  - Products (`/admin/products`)
  - Inventory (`/admin/inventory`)
  - Suppliers (`/admin/suppliers`)

## Cấu trúc tài liệu

```
phase-role-based-access-control/
├── README.md (file này)
├── architecture.md - Kiến trúc và thiết kế route guards
├── implementation-plan.md - Kế hoạch triển khai chi tiết
├── file-changes.md - Danh sách các file cần sửa/thêm/xóa
└── testing-plan.md - Kế hoạch kiểm thử
```

## Công nghệ sử dụng
- **TanStack Router**: Route guards sử dụng `beforeLoad` hook
- **Zustand**: Auth store đã có sẵn với role checking
- **TypeScript**: Type-safe route guards

## Trạng thái
- [ ] Đang chờ duyệt
- [ ] Đã duyệt - Sẵn sàng triển khai
- [ ] Đang triển khai
- [ ] Hoàn thành

