# Luồng Validate Promotion khi tạo đơn hàng với mã giảm giá

## Tổng quan

Tài liệu này mô tả chi tiết luồng validation promotion khi tạo một đơn hàng mới với mã giảm giá trong hệ thống RetailStoreManagement.

## Luồng tổng thể

```mermaid
flowchart TD
    A[Start: CreateOrderAsync với PromoCode] --> B{Có PromoCode?}
    B -->|Không| Z[Skip promotion validation]
    B -->|Có| C[Tìm promotion theo PromoCode]
    C --> D{Promotion tồn tại?}
    D -->|Không| E[Return Error: Promotion code not found 404]
    D -->|Có| F[ValidatePromotionBasic]
    
    F --> G{Status = Active?}
    G -->|Không| H[Return Error: Promotion is not active]
    G -->|Có| I{Date trong khoảng StartDate - EndDate?}
    I -->|Không| J[Return Error: Promotion is not valid for current date]
    I -->|Có| K{UsageLimit > 0?}
    K -->|Không| L[Skip usage limit check]
    K -->|Có| M{UsedCount < UsageLimit?}
    M -->|Không| N[Return Error: Promotion usage limit exceeded]
    M -->|Có| L
    
    L --> O[Basic Validation PASSED]
    O --> P[Tính TotalAmount từ OrderItems]
    P --> Q[ValidatePromotion với TotalAmount]
    
    Q --> R[ValidatePromotionBasic lại]
    R --> S{Basic checks PASS?}
    S -->|Không| T[Return Error từ basic validation]
    S -->|Có| U{TotalAmount >= MinOrderAmount?}
    U -->|Không| V[Return Error: Order amount must be at least $X]
    U -->|Có| W[CalculateDiscount]
    
    W --> X{DiscountType?}
    X -->|Percent| Y[discountAmount = totalAmount * discountValue / 100]
    X -->|Fixed| AA[discountAmount = discountValue]
    
    Y --> AB[Create Order với discountAmount]
    AA --> AB
    Z --> P2[Tính TotalAmount từ OrderItems]
    P2 --> AB
    
    AB --> AC[Save Order to Database]
    AC --> AD[Create OrderItems]
    AD --> AE{Promotion applied?}
    AE -->|Có| AF[Increment promotion.UsedCount++]
    AE -->|Không| AG[Skip]
    AF --> AH[Save Changes]
    AG --> AH
    AH --> AI[Return OrderDetailsDto]
    
    style F fill:#e1f5fe
    style Q fill:#fff3e0
    style W fill:#e8f5e8
    style E fill:#ffebee
    style H fill:#ffebee
    style J fill:#ffebee
    style N fill:#ffebee
    style V fill:#ffebee
    style T fill:#ffebee
```

## Chi tiết các bước validation

### Bước 1: ValidatePromotionBasic (Không cần TotalAmount)

```mermaid
flowchart TD
    A[ValidatePromotionBasic] --> B{Status == Active?}
    B -->|No| C[Return: Promotion is not active]
    B -->|Yes| D{Current Date >= StartDate?}
    D -->|No| E[Return: Promotion is not valid for current date]
    D -->|Yes| F{Current Date <= EndDate?}
    F -->|No| E
    F -->|Yes| G{UsageLimit > 0?}
    G -->|No| H[Skip usage check - Return Valid]
    G -->|Yes| I{UsedCount < UsageLimit?}
    I -->|No| J[Return: Promotion usage limit exceeded]
    I -->|Yes| H
    
    style C fill:#ffebee
    style E fill:#ffebee
    style J fill:#ffebee
    style H fill:#e8f5e8
```

### Bước 2: ValidatePromotion (Cần TotalAmount)

```mermaid
flowchart TD
    A[ValidatePromotion với TotalAmount] --> B[Gọi ValidatePromotionBasic]
    B --> C{Basic Validation PASS?}
    C -->|No| D[Return error từ basic validation]
    C -->|Yes| E{TotalAmount >= MinOrderAmount?}
    E -->|No| F[Return: Order amount must be at least $X]
    E -->|Yes| G[Return: Valid]
    
    style D fill:#ffebee
    style F fill:#ffebee
    style G fill:#e8f5e8
```

### Bước 3: CalculateDiscount

```mermaid
flowchart TD
    A[CalculateDiscount] --> B{DiscountType?}
    B -->|Percent| C[discountAmount = totalAmount * discountValue / 100]
    B -->|Fixed| D[discountAmount = discountValue]
    C --> E[Return discountAmount]
    D --> E
    
    style E fill:#e8f5e8
```

## Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant OrderService
    participant PromotionRepo
    participant ValidatePromotionBasic
    participant ValidatePromotion
    participant CalculateDiscount
    participant OrderRepo
    participant PromotionEntity

    Client->>OrderService: CreateOrderAsync(request với PromoCode)
    
    Note over OrderService: 1. Validate Customer & Products
    
    alt Có PromoCode
        OrderService->>PromotionRepo: GetQueryable().FirstOrDefaultAsync(PromoCode)
        PromotionRepo-->>OrderService: promotion
        
        alt Promotion không tồn tại
            OrderService-->>Client: Error 404: Promotion code not foundOreview
        else Promotion tồn tại
            OrderService->>ValidatePromotionBasic: Validate(promotion)
            ValidatePromotionBasic->>ValidatePromotionBasic: Check Status == Active
            ValidatePromotionBasic->>ValidatePromotionBasic: Check Date range
            ValidatePromotionBasic->>ValidatePromotionBasic: Check UsageLimit
            
            alt Basic Validation FAIL
                ValidatePromotionBasic-->>OrderService: (false, errorMessage)
                OrderService-->>Client: Error 400: errorMessage
            else Basic Validation PASS
                ValidatePromotionBasic-->>OrderService: (true, "Valid")
            end
        end
    end
    
    Note over OrderService: 2. Tính TotalAmount từ OrderItems
    
    loop For each OrderItem
        OrderService->>OrderService: Calculate subtotal = Price * Quantity
        OrderService->>OrderService: totalAmount += subtotal
    end
    
    alt Có Promotion
        OrderService->>ValidatePromotion: Validate(promotion, totalAmount)
        ValidatePromotion->>ValidatePromotionBasic: ValidateBasic(promotion)
        ValidatePromotionBasic-->>ValidatePromotion: (true/false, message)
        
        alt Full Validation FAIL
            ValidatePromotion-->>OrderService: (false, errorMessage)
            OrderService-->>Client: Error 400: errorMessage
        else Full Validation PASS
            ValidatePromotion->>ValidatePromotion: Check TotalAmount >= MinOrderAmount
            ValidatePromotion-->>OrderService: (true, "Valid")
            
            OrderService->>CalculateDiscount: Calculate(promotion, totalAmount)
            CalculateDiscount-->>OrderService: discountAmount
            
            OrderService->>OrderRepo: AddAsync(order)
            OrderRepo-->>OrderService: order saved
            
            OrderService->>OrderRepo: AddAsync(orderItems)
            
            OrderService->>PromotionEntity: promotion.UsedCount++
            OrderService->>PromotionRepo: UpdateAsync(promotion)
            
            OrderService-->>Client: OrderDetailsDto (Success)
        end
    else Không có Promotion
        OrderService->>OrderRepo: AddAsync(order)
        OrderService-->>Client: OrderDetailsDto (Success)
    end
```

## Các điều kiện validation

### ValidatePromotionBasic
1. **Status Check**: `promotion.Status == PromotionStatus.Active`
2. **Date Range Check**: 
   - `DateTime.UtcNow >= promotion.StartDate.ToDateTime(TimeOnly.MinValue)`
   - `DateTime.UtcNow <= promotion.EndDate.ToDateTime(TimeOnly.MaxValue)`
3. **Usage Limit Check**: 
   - Nếu `UsageLimit > 0`: `UsedCount < UsageLimit`
   - Nếu `UsageLimit == 0`: Không giới hạn, skip check

### ValidatePromotion (Full)
1. Tất cả điều kiện của ValidatePromotionBasic
2. **MinOrderAmount Check**: `totalAmount >= promotion.MinOrderAmount`

## Các trường hợp lỗi

| Lỗi | Điều kiện | HTTP Status | Message |
|-----|-----------|-------------|---------|
| Promotion code not found | PromoCode không tồn tại trong DB | 404 | "Promotion code not found" |
| Promotion is not active | Status != Active | 400 | "Promotion is not active" |
| Promotion is not valid for current date | Date ngoài khoảng StartDate - EndDate | 400 | "Promotion is not valid for current date" |
| Promotion usage limit exceeded | UsedCount >= UsageLimit | 400 | "Promotion usage limit exceeded" |
| Order amount too low | TotalAmount < MinOrderAmount | 400 | "Order amount must be at least {MinOrderAmount:C}" |

## Ví dụ thực tế

### Ví dụ 1: Promotion hợp lệ
```
Input:
- PromoCode: "SUMMER2024"
- OrderItems: [{ProductId: 1, Quantity: 2}, {ProductId: 2, Quantity: 1}]
- Product Prices: [100000, 50000]
- TotalAmount: 250000
- Promotion: {Status: Active, MinOrderAmount: 100000, UsageLimit: 100, UsedCount: 50}

Flow:
1. ✅ ValidatePromotionBasic: PASS (Active, Date OK, Usage OK)
2. ✅ Tính TotalAmount: 250000
3. ✅ ValidatePromotion: PASS (250000 >= 100000)
4. ✅ CalculateDiscount: discountAmount = ...
5. ✅ Create Order với discount
```

### Ví dụ 2: Promotion không hợp lệ - MinOrderAmount quá cao
```
Input:
- PromoCode: "VIP2024"
- OrderItems: [{ProductId: 1, Quantity: 1}]
- Product Price: 100000
- TotalAmount: 100000
- Promotion: {Status: Active, MinOrderAmount: 1990000, ...}

Flow:
1. ✅ ValidatePromotionBasic: PASS
2. ✅ Tính TotalAmount: 100000
3. ❌ ValidatePromotion: FAIL (100000 < 1990000)
4. ❌ Return Error: "Order amount must be at least $1,990,000.00"
```

## Code Reference

### ValidatePromotionBasic
```csharp
private (bool isValid, string message) ValidatePromotionBasic(PromotionEntity promotion)
{
    if (promotion.Status != PromotionStatus.Active)
        return (false, "Promotion is not active");

    var now = DateTime.UtcNow;
    var startDate = promotion.StartDate.ToDateTime(TimeOnly.MinValue);
    var endDate = promotion.EndDate.ToDateTime(TimeOnly.MaxValue);

    if (now < startDate || now > endDate)
        return (false, "Promotion is not valid for current date");

    if (promotion.UsageLimit > 0 && promotion.UsedCount >= promotion.UsageLimit)
        return (false, "Promotion usage limit exceeded");

    return (true, "Valid");
}
```

### ValidatePromotion
```csharp
private (bool isValid, string message) ValidatePromotion(PromotionEntity promotion, decimal orderTotal)
{
    // Validate basic conditions first
    var basicValidation = ValidatePromotionBasic(promotion);
    if (!basicValidation.isValid)
        return basicValidation;

    // Validate minOrderAmount
    if (orderTotal < promotion.MinOrderAmount)
        return (false, $"Order amount must be at least {promotion.MinOrderAmount:C}");

    return (true, "Valid");
}
```

## Tối ưu hóa

1. **Validation 2 bước**: 
   - Bước 1 (Basic): Validate sớm để tránh tính toán không cần thiết
   - Bước 2 (Full): Validate sau khi có TotalAmount để check MinOrderAmount

2. **Early Return**: Trả về lỗi ngay khi phát hiện điều kiện không hợp lệ

3. **Reuse Logic**: ValidatePromotion gọi lại ValidatePromotionBasic để tránh duplicate code

## Notes

- **Currency Format**: `{MinOrderAmount:C}` format theo culture hiện tại (VND hoặc USD)
- **Date Comparison**: Sử dụng `DateTime.UtcNow` để so sánh với `DateOnly.ToDateTime()`
- **Usage Limit**: Nếu `UsageLimit == 0` nghĩa là không giới hạn, skip check
- **Discount Calculation**: Chỉ tính discount sau khi validation PASS hoàn toàn

