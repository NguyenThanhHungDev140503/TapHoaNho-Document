# ADVANCED FEATURES IMPLEMENTATION PLAN

> **Tài liệu này**: Planning chi tiết cho 6 tính năng nâng cao của Universal CRUD Hooks System
> 
> **Liên kết**: Phần mở rộng của `customHookCallAPIPlan.md` và `VALIDATION_REPORT.md`
> 
> **Status**: Planning Phase - Chưa implement
> 
> **Priority**: OPTIONAL - Implement sau khi hoàn thành MVP

---

## MỤC LỤC

1. [Performance Optimization Guide](#1-performance-optimization-guide)
2. [Error Boundary Integration](#2-error-boundary-integration)
3. [Optimistic Updates Details](#3-optimistic-updates-details)
4. [Prefetching Strategy](#4-prefetching-strategy)
5. [Testing Guidelines](#5-testing-guidelines)
6. [Debouncing Search](#6-debouncing-search)

---

## TỔNG QUAN

### Mục tiêu:
Cải thiện performance, UX, và maintainability của Universal CRUD Hooks System thông qua 6 tính năng nâng cao.

### Thứ tự ưu tiên implement:
1. **HIGH**: Performance Optimization (staleTime configuration)
2. **HIGH**: Debouncing Search (improve UX)
3. **MEDIUM**: Error Boundary Integration (better error handling)
4. **MEDIUM**: Prefetching Strategy (improve perceived performance)
5. **LOW**: Optimistic Updates (advanced UX)
6. **LOW**: Testing Guidelines (quality assurance)

### Thời gian ước tính:
- **Performance Optimization**: 2-3 giờ
- **Debouncing Search**: 1-2 giờ
- **Error Boundary**: 2-3 giờ
- **Prefetching**: 3-4 giờ
- **Optimistic Updates**: 4-5 giờ
- **Testing Guidelines**: 5-6 giờ

**Tổng**: 17-23 giờ

---

## 1. PERFORMANCE OPTIMIZATION GUIDE

### 1.1. Mục tiêu

Tối ưu hóa performance bằng cách cấu hình `staleTime` phù hợp cho từng loại data, giảm số lượng API calls không cần thiết.

### 1.2. Vấn đề hiện tại

**Current State**:
- Plan hiện tại có mention `staleTime` nhưng chưa có best practices
- Developers có thể không biết nên set `staleTime` bao nhiêu cho từng entity
- Default `staleTime: 0` → Data luôn stale → Refetch quá nhiều

**Impact**:
- ❌ Unnecessary API calls
- ❌ Increased server load
- ❌ Slower perceived performance
- ❌ Higher bandwidth usage

### 1.3. Giải pháp

#### 1.3.1. StaleTime Configuration Constants

**File**: `src/config/queryConfig.ts` (NEW)

```typescript
/**
 * StaleTime Configuration cho các loại data khác nhau
 * 
 * Nguyên tắc:
 * - STATIC: Data hiếm khi thay đổi (categories, settings)
 * - FREQUENT: Data thay đổi thường xuyên (products, inventory)
 * - MODERATE: Data thay đổi vừa phải (orders, customers)
 * - RARE: Data thay đổi ít (reports, analytics)
 */
export const STALE_TIME = {
  /**
   * STATIC: Infinity
   * Use for: Categories, Settings, Static configurations
   * Rationale: Data này hiếm khi thay đổi, chỉ refetch khi invalidate manually
   */
  STATIC: Infinity,

  /**
   * FREQUENT: 1 minute
   * Use for: Products, Inventory, Real-time data
   * Rationale: Data thay đổi thường xuyên, cần fresh data
   */
  FREQUENT: 1000 * 60, // 1 minute

  /**
   * MODERATE: 5 minutes
   * Use for: Orders, Customers, Suppliers
   * Rationale: Data thay đổi vừa phải, balance giữa freshness và performance
   */
  MODERATE: 1000 * 60 * 5, // 5 minutes

  /**
   * RARE: 30 minutes
   * Use for: Reports, Analytics, Historical data
   * Rationale: Data ít thay đổi, có thể cache lâu
   */
  RARE: 1000 * 60 * 30, // 30 minutes

  /**
   * NEVER: 0
   * Use for: One-time queries, Critical data
   * Rationale: Luôn fetch fresh data
   */
  NEVER: 0,
} as const;

/**
 * GcTime Configuration
 * Thời gian giữ cache sau khi không sử dụng
 */
export const GC_TIME = {
  /**
   * SHORT: 5 minutes
   * Use for: Temporary data, Search results
   */
  SHORT: 1000 * 60 * 5,

  /**
   * MEDIUM: 30 minutes (default)
   * Use for: Most entities
   */
  MEDIUM: 1000 * 60 * 30,

  /**
   * LONG: 1 hour
   * Use for: Static data, Rarely changing data
   */
  LONG: 1000 * 60 * 60,

  /**
   * PERMANENT: Infinity
   * Use for: App-wide static data
   */
  PERMANENT: Infinity,
} as const;

/**
 * Entity-specific configuration
 */
export const ENTITY_QUERY_CONFIG = {
  // Static data
  categories: {
    staleTime: STALE_TIME.STATIC,
    gcTime: GC_TIME.LONG,
  },
  
  // Frequent updates
  products: {
    staleTime: STALE_TIME.FREQUENT,
    gcTime: GC_TIME.MEDIUM,
  },
  inventory: {
    staleTime: STALE_TIME.FREQUENT,
    gcTime: GC_TIME.MEDIUM,
  },
  
  // Moderate updates
  orders: {
    staleTime: STALE_TIME.MODERATE,
    gcTime: GC_TIME.MEDIUM,
  },
  customers: {
    staleTime: STALE_TIME.MODERATE,
    gcTime: GC_TIME.MEDIUM,
  },
  suppliers: {
    staleTime: STALE_TIME.MODERATE,
    gcTime: GC_TIME.MEDIUM,
  },
  
  // Rare updates
  reports: {
    staleTime: STALE_TIME.RARE,
    gcTime: GC_TIME.LONG,
  },
  
  // User-specific
  users: {
    staleTime: STALE_TIME.MODERATE,
    gcTime: GC_TIME.MEDIUM,
  },
  promotions: {
    staleTime: STALE_TIME.MODERATE,
    gcTime: GC_TIME.MEDIUM,
  },
} as const;
```


