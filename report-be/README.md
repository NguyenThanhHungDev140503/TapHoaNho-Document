# Tài Liệu Backend Reports - Retail Store Management

## Mục Lục

- [Tổng Quan](#tổng-quan)
- [Cấu Trúc Tài Liệu](#cấu-trúc-tài-liệu)
- [Quick Start](#quick-start)
- [Các Endpoint Reports](#các-endpoint-reports)

## Tổng Quan

Thư mục này chứa tài liệu hướng dẫn implementation chi tiết cho các chức năng báo cáo (Reports) trong hệ thống Retail Store Management.

### Các Chức Năng Báo Cáo

1. **Báo Cáo Khuyến Mãi (Promotion Reports)**
   - Báo cáo hiệu quả từng mã giảm giá
   - Báo cáo đơn hàng có sử dụng khuyến mãi
   - Tính toán tỉ lệ chuyển đổi

2. **Dự Báo Tồn Kho (Inventory Forecast)**
   - Dự báo tồn kho dựa trên số lượng bán trung bình
   - Đề xuất nhập hàng
   - Cảnh báo tồn kho thấp/nguy hiểm

## Cấu Trúc Tài Liệu

```
report-be/
├── README.md (file này)
└── PROMOTION_REPORTS_IMPLEMENTATION.md
    ├── Tổng Quan Kiến Trúc Backend
    ├── Phân Tích Kiến Trúc Hiện Tại
    ├── Implementation Báo Cáo Khuyến Mãi
    ├── Implementation Dự Báo Tồn Kho
    ├── Code Examples Chi Tiết
    └── Testing và Validation
```

## Quick Start

### 1. Đọc Tài Liệu Implementation

Bắt đầu với file [PROMOTION_REPORTS_IMPLEMENTATION.md](./PROMOTION_REPORTS_IMPLEMENTATION.md) để hiểu:
- Kiến trúc backend hiện tại
- Cách implement các endpoint reports
- Code examples chi tiết

### 2. Implementation Steps

1. **Tạo DTOs**: Tạo các file DTO trong `Models/Report/`
2. **Cập nhật Interface**: Thêm methods mới vào `IReportService`
3. **Implement Service**: Implement logic trong `ReportService`
4. **Cập nhật Controller**: Thêm endpoints vào `ReportsController`
5. **Testing**: Viết unit tests và integration tests

### 3. API Endpoints

Sau khi implement, các endpoint sau sẽ có sẵn:

```
GET /api/admin/reports/promotion
GET /api/admin/reports/inventory-forecast
```

## Các Endpoint Reports

### 1. Promotion Report

**Endpoint:** `GET /api/admin/reports/promotion`

**Query Parameters:**
- `startDate` (optional): Ngày bắt đầu
- `endDate` (optional): Ngày kết thúc
- `promoId` (optional): Filter theo mã khuyến mãi cụ thể
- `includeOrderDetails` (optional, default: false): Có bao gồm chi tiết đơn hàng không

**Response:**
- `promotionEffectiveness`: Danh sách hiệu quả từng mã khuyến mãi
- `ordersWithPromotion`: Danh sách đơn hàng có sử dụng khuyến mãi (nếu `includeOrderDetails=true`)
- `summary`: Tổng hợp thống kê

### 2. Inventory Forecast

**Endpoint:** `GET /api/admin/reports/inventory-forecast`

**Query Parameters:**
- `productId` (optional): Filter theo sản phẩm cụ thể
- `categoryId` (optional): Filter theo danh mục
- `lookbackMonths` (optional, default: 3): Số tháng để tính trung bình
- `leadTimeDays` (optional, default: 7): Thời gian giao hàng dự kiến (ngày)
- `safetyStockMultiplier` (optional, default: 1.5): Hệ số an toàn

**Response:**
- `forecasts`: Danh sách dự báo cho từng sản phẩm
- `summary`: Tổng hợp thống kê

## Tài Liệu Tham Khảo

- [ERD Analysis](../../shiny-carnival/docs/PhanTichERD_StoreManagement.md)
- [SRS Document](../../shiny-carnival/docs/SRS_StoreManagement.md)
- [Database Schema](../../shiny-carnival/RetailStoreManagement/docs/db/schema.md)

## Liên Hệ

Nếu có câu hỏi hoặc cần hỗ trợ, vui lòng tham khảo tài liệu implementation chi tiết hoặc liên hệ team phát triển.

