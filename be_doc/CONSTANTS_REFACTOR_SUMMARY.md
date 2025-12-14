# Constants Refactoring Summary

## ✅ Đã tạo Constants Classes

### 1. InventoryConstants.cs
- `LOW_STOCK_THRESHOLD = 10` - Đã được sử dụng trong InventoryService và ProductService

### 2. ValidationConstants.cs
Các constants cho validation constraints (cần cập nhật models để sử dụng):

**String Length Constraints:**
- `MAX_LENGTH_NAME = 100` - Dùng cho: ProductName, CategoryName, Name (Customer/Supplier), FullName
- `MAX_LENGTH_CODE = 50` - Dùng cho: Barcode, PromoCode, Username
- `MAX_LENGTH_PHONE = 20` - Dùng cho: Phone number
- `MAX_LENGTH_UNIT = 20` - Dùng cho: Unit (pcs, kg, etc.)
- `MAX_LENGTH_EMAIL = 100` - Dùng cho: Email address
- `MAX_LENGTH_ADDRESS = 255` - Dùng cho: Address
- `MAX_LENGTH_DESCRIPTION = 255` - Dùng cho: Description

**Minimum Length Constraints:**
- `MIN_LENGTH_PASSWORD = 6` - Dùng cho: Password minimum length

**Numeric Constraints:**
- `MIN_PRICE = 0.01m` - Dùng cho: Minimum price/discount value
- `MIN_QUANTITY = 1` - Dùng cho: Minimum quantity for order items
- `MIN_USAGE_LIMIT = 1` - Dùng cho: Minimum usage limit for promotions

**Role Constraints:**
- `ROLE_ADMIN = 0`
- `ROLE_STAFF = 1`

### 3. ReportConstants.cs
- `DEFAULT_TOP_ITEMS_PAGE_SIZE = 10` - ✅ Đã cập nhật trong ReportService
- `DEFAULT_TOP_ITEMS_PAGE = 1` - ✅ Đã cập nhật trong ReportService
- `MAX_LOOKBACK_MONTHS = 12` - ✅ Đã cập nhật trong InventoryForecastRequest
- `MIN_LOOKBACK_MONTHS = 1` - ✅ Đã cập nhật trong InventoryForecastRequest
- `MAX_LEAD_TIME_DAYS = 90` - ✅ Đã cập nhật trong InventoryForecastRequest
- `MIN_LEAD_TIME_DAYS = 1` - ✅ Đã cập nhật trong InventoryForecastRequest
- `MAX_SAFETY_STOCK_MULTIPLIER = 3.0` - ✅ Đã cập nhật trong InventoryForecastRequest
- `MIN_SAFETY_STOCK_MULTIPLIER = 1.0` - ✅ Đã cập nhật trong InventoryForecastRequest

## 📋 Cần Refactor (Chưa cập nhật)

### Models cần cập nhật để sử dụng ValidationConstants:

1. **Product Models:**
   - `CreateProductRequest.cs` - MaxLength(100), MaxLength(50), MaxLength(20), Range(0.01, ...)
   - `UpdateProductRequest.cs` - MaxLength(100), MaxLength(50), MaxLength(20), Range(0.01, ...)

2. **Customer Models:**
   - `CreateCustomerRequest.cs` - MaxLength(100), MaxLength(20), MaxLength(100), MaxLength(255)
   - `UpdateCustomerRequest.cs` - MaxLength(100), MaxLength(20), MaxLength(100), MaxLength(255)

3. **Supplier Models:**
   - `CreateSupplierRequest.cs` - MaxLength(100), MaxLength(20), MaxLength(100), MaxLength(255)
   - `UpdateSupplierRequest.cs` - MaxLength(100), MaxLength(20), MaxLength(100), MaxLength(255)

4. **Category Models:**
   - `CreateCategoryRequest.cs` - MaxLength(100)
   - `UpdateCategoryRequest.cs` - MaxLength(100)

5. **User Models:**
   - `CreateUserRequest.cs` - MaxLength(50), MinLength(6), MaxLength(100), Range(0, 1)
   - `UpdateUserRequest.cs` - MaxLength(50), MaxLength(100), Range(0, 1)

6. **Order Models:**
   - `CreateOrderRequest.cs` - MaxLength(50)
   - `AddOrderItemRequest.cs` - Range(1, int.MaxValue)

7. **Promotion Models:**
   - `CreatePromotionRequest.cs` - MaxLength(50), MaxLength(255), Range(0.01, ...), Range(1, ...)
   - `UpdatePromotionRequest.cs` - MaxLength(50), MaxLength(255), Range(0.01, ...), Range(1, ...)

## 🔍 Các Patterns Khác Cần Xem Xét

### String Literals (Có thể cần constants):
- RegularExpression patterns: `"^(percent|fixed)$"`, `"^(active|inactive)$"` - Có thể dùng enum thay vì string
- Status strings: "pending", "paid", "canceled", "active", "inactive" - Đã có enums nhưng có thể cần constants cho string representation

### Hardcoded Values Khác:
- `int.MaxValue` trong Range attributes - Có thể tạo constant nếu muốn giới hạn
- Default values trong models (vd: `= 1`, `= 0`, `= "active"`) - Có thể tạo constants

## 📝 Ghi Chú

- Các constants đã được tạo trong `Common/` folder
- Sử dụng `using static RetailStoreManagement.Common.ValidationConstants;` để truy cập trực tiếp
- Các models hiện tại vẫn hoạt động với hardcoded values, nhưng nên refactor để tuân thủ DRY
- Việc refactor tất cả models có thể được thực hiện từng bước, không cần làm hết một lúc

