# TanStack Router Query 

## Tổng Quan

**TanStack Router Query** (search params) là lớp quản lý URL search parameters của TanStack Router, giúp bạn:

- Quản lý state UI (filters, sort, pagination, search…) thông qua URL.
- Đảm bảo type-safe với TypeScript + validation schema (Zod, Valibot, ArkType).
- Đồng bộ hai chiều **URL ↔ UI state** một cách reactive.
- Tối ưu UX với bookmarkable/shareable URLs và browser history tự nhiên.

Thư mục này là **kho tài liệu chuyên sâu** cho phần search params, được tổ chức giống cấu trúc TanStack Query: tách rõ file chính, advanced patterns, principal patterns và nhật ký research.

---

## Cấu Trúc Tài Liệu

### 1. File chính (Junior/Middle/Senior)

- `TanStack Router Query.md`  
  Hướng dẫn toàn diện theo level:
  - Junior: khái niệm cơ bản, useSearch, Link, useNavigate, hello world.
  - Middle: pagination, filter, sort, custom hooks domain, debounce cơ bản.
  - Senior (mở đầu): selective subscription, stripSearchParams, kết hợp với TanStack Query.

### 2. Advanced Patterns (Senior)

- `Advanced-Patterns.md`  
  Tập trung vào:
  - Workflow patterns nâng cao (filter + sort + paginate, wizard form, collaborative view…).
  - Search history, undo/redo, diff/merge.
  - Performance optimization (select, memo, debounce navigate, lazy load…).

### 3. Principal-Level Patterns (Principal/Staff)

- `Principal-Level-Patterns.md`  
  Dành cho Principal/Staff:
  - Enterprise-scale architecture cho URL-first state.
  - Centralized schema & search-key factory.
  - Tích hợp với API client, monitoring, logging.
  - Migration strategies (local state / Redux → URL-first với TanStack Router Query).

### 4. Research Summary

- `RESEARCH_SUMMARY.md`  
  Nhật ký research:
  - Mục tiêu, phạm vi sử dụng.
  - Danh sách nguồn (docs, blog, repo, issue/PR) với URL đầy đủ và ngày truy cập.
  - Key findings, ưu/nhược điểm, so sánh với giải pháp khác.
  - Roadmap và use case đã xác định.

---

## Cách Sử Dụng Tài Liệu (Theo Level)

### Cho Junior Developer

- Đọc:
  - `TanStack Router Query.md` – phần **Giới thiệu** + **Junior Level – Cơ Bản**.
- Tập trung:
  - Hiểu query/search params là gì.
  - Đọc/ghi search params với `useSearch`, `Link`, `useNavigate`.
  - Nắm được pitfall cơ bản (mất params khi không dùng functional update).

### Cho Middle Developer

- Đọc:
  - `TanStack Router Query.md` – phần **Middle Level – Trung Cấp**.
  
    Tập trung:
  
  - Tạo custom hook domain (`useProductSearch`, `useTableSearch`…).
  - Kết hợp pagination + filter + sort vào URL.
  - Debounce search input để tránh spam URL.

### Cho Senior Developer

- Đọc:
  - `TanStack Router Query.md` – phần **Senior Level – Nâng Cao (mở đầu)**.
  - `Advanced-Patterns.md`.
  - Các phần nâng cao trong `WORKFLOW_PATTERNS.md`.
- Tập trung:
  - Query-key/search-key factory.
  - Search history, undo/redo, diff/merge.
  - Performance optimization (select, memo, debounce, lazy loading).
  - Integration với TanStack Query.

### Cho Principal/Staff Engineer

- Đọc:
  - `Principal-Level-Patterns.md`.
  - `ARCHITECTURE.md`.
  - `RESEARCH_SUMMARY.md` (phần kiến trúc và roadmap).
- Tập trung:
  - Thiết kế kiến trúc URL-first cho hệ thống lớn (multi-module, multi-team).
  - Chuẩn hoá schema, search key, utilities dùng chung.
  - Chiến lược migration và monitoring/observability.

---

## Key Concepts (Tóm tắt nhanh)

```typescript
// Schema cho search params
const searchSchema = z.object({
  page: z.number().int().positive().default(1),
  limit: z.number().int().min(1).max(100).default(10),
  category: z.string().optional(),
  sort: z.enum(['name', 'price', 'date']).default('name'),
})

// Route với validateSearch
export const Route = createFileRoute('/products')({
  validateSearch: zodValidator(searchSchema),
})

// Đọc search params (typed)
function ProductsPage() {
  const search = Route.useSearch()
  // search.page: number, search.category?: string, ...
}
```

- **URL-first state**: search params là nguồn sự thật cho UI state.
- **Validation pipeline**: raw URL → parse → validate (schema) → typed object.
- **Functional update**: luôn cập nhật search bằng `(prev) => ({ ...prev, ... })`.
- **Selective subscription**: chỉ subscribe field cần thiết để tối ưu re-render.

---

## External Resources

- TanStack Router Official Docs: https://tanstack.com/router
- Search Params Guide: https://tanstack.com/router/latest/docs/framework/react/guide/search-params
- useSearch Hook API: https://tanstack.com/router/latest/docs/framework/react/api/router/useSearchHook
- Validation Guide: https://tanstack.com/router/latest/docs/framework/react/how-to/validate-search-params

---

**Cập nhật lần cuối:** 2025-12-03  
**Phiên bản:** 2.0 (chuẩn hoá theo cấu trúc TanStack Query)

