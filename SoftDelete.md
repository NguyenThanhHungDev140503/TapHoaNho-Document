# Giải thích Code: ConfigureSoftDelete

Đây là một phương thức cấu hình **Soft Delete** (xóa mềm) cho Entity Framework Core. Thay vì xóa vĩnh viễn dữ liệu khỏi database, nó đánh dấu bản ghi là "đã xóa" bằng cách set giá trị `DeletedAt`.

## Phân tích chi tiết từng phần:

### 1. Xác định kiểu Entity cơ sở
```csharp
var baseEntityType = typeof(BaseEntity<int>);
```
- Lấy kiểu dữ liệu của `BaseEntity<int>` - đây là class cơ sở cho các entity trong hệ thống
- `BaseEntity<int>` có thuộc tính `DeletedAt` kiểu `DateTime?`

### 2. Lọc các Entity kế thừa từ BaseEntity
```csharp
foreach (var entityType in modelBuilder.Model.GetEntityTypes()
             .Where(e => baseEntityType.IsAssignableFrom(e.ClrType)))
```
- `modelBuilder.Model.GetEntityTypes()`: Lấy tất cả các entity types đã được đăng ký
- `baseEntityType.IsAssignableFrom(e.ClrType)`: Chỉ lấy các entity kế thừa từ `BaseEntity<int>`
- Ví dụ: Product, Category, Customer... nếu chúng kế thừa từ BaseEntity

### 3. Xây dựng Expression Tree động
```csharp
var parameter = Expression.Parameter(entityType.ClrType, "e");
var property = Expression.Property(parameter, "DeletedAt");
var nullValue = Expression.Constant(null, typeof(DateTime?));
var equals = Expression.Equal(property, nullValue);
var lambda = Expression.Lambda(equals, parameter);
```

Đoạn này tạo ra một **Lambda Expression** động tương đương với:
```csharp
e => e.DeletedAt == null
```

Chi tiết:
- `parameter`: Tham số `e` (đại diện cho entity)
- `property`: Thuộc tính `e.DeletedAt`
- `nullValue`: Giá trị `null`
- `equals`: Biểu thức so sánh `e.DeletedAt == null`
- `lambda`: Lambda expression hoàn chỉnh

### 4. Áp dụng Query Filter
```csharp
modelBuilder.Entity(entityType.ClrType)
    .HasQueryFilter(lambda);
```
- Áp dụng **Global Query Filter** cho entity
- Mọi truy vấn SELECT sẽ tự động thêm điều kiện `WHERE DeletedAt IS NULL`
- Chỉ lấy các bản ghi chưa bị xóa mềm

## Kết quả thực tế:

Khi bạn query:
```csharp
var products = await context.Products.ToListAsync();
```

EF Core tự động chuyển thành:
```sql
SELECT * FROM Products WHERE DeletedAt IS NULL
```

## Lợi ích:

✅ **Tự động**: Không cần nhớ thêm điều kiện `DeletedAt == null` mỗi lần query  
✅ **An toàn**: Tránh hiển thị dữ liệu đã xóa  
✅ **Linh hoạt**: Có thể khôi phục dữ liệu khi cần  
✅ **Audit**: Giữ lại lịch sử xóa cho mục đích kiểm tra  

## Bypass Query Filter (nếu cần):

Nếu muốn lấy cả dữ liệu đã xóa:
```csharp
var allProducts = await context.Products
    .IgnoreQueryFilters()
    .ToListAsync();
```
