# TanStack Router Query – Principal-Level Patterns

## Mục Lục

1. Mục Tiêu Ở Cấp Principal/Staff
2. Enterprise Architecture Cho URL-First State
3. Centralized Schema & Search Key Factory
4. Integration Với API Client, Monitoring, Logging
5. Migration Strategies
6. Liên Kết Tới Tài Liệu Chi Tiết

> File này mang tính **định hướng kiến trúc & pattern cấp hệ thống**. Chi tiết code có thể được giữ trong `ARCHITECTURE.md`, `IMPLEMENT_GUIDE.md`, `WORKFLOW_PATTERNS.md`.

---

## 1. Mục Tiêu Ở Cấp Principal/Staff

Khi sử dụng TanStack Router Query ở quy mô lớn, mục tiêu không chỉ là “dùng được”, mà là:

- **Chuẩn hoá** cách quản lý URL state cho toàn hệ thống.
- Đảm bảo **type-safety + validation** đồng nhất giữa các module/feature team.
- Tối ưu hoá **observability** (monitoring, logging, metrics) quanh việc thay đổi search params.
- Định nghĩa các **pattern & guideline** để team junior/middle/senior implement nhất quán.

---

## 2. Enterprise Architecture Cho URL-First State

Khung kiến trúc gợi ý (chi tiết hơn xem `ARCHITECTURE.md` + `RESEARCH_SUMMARY.md`):

- `router/` – khai báo router + routes.
- `schemas/` – search schemas (Zod/Valibot/ArkType).
- `hooks/` – domain hooks (`useProductSearch`, `useTableSearch`, `useWizardSearch`…).
- `components/` – filters, pagination, search bar, wizard steps.
- `utils/` – serialization, helpers cho search.
- `types/` – type chung cho search DTO.

Mục tiêu:

- **Không để logic search phân tán** vào từng component.
- Mọi màn hình có search phức tạp đều dùng chung pattern:
  - Schema → Route → Domain Hook → UI.

---

## 3. Centralized Schema & Search Key Factory

### 3.1 Centralized Search Schema

- Mỗi domain (products, orders, reports…) có 1 schema rõ ràng.
- Mọi chỗ dùng search của domain đó đều import cùng một schema/type.

### 3.2 Search Key / Query Key Factory

Áp dụng pattern Query Key Factory (tương tự TanStack Query) cho search/query:

```typescript
export const productSearchKeys = {
  all: ['products'] as const,
  list: (search: ProductSearch) => [...productSearchKeys.all, 'list', search] as const,
  detail: (id: string) => [...productSearchKeys.all, 'detail', id] as const,
}
```

Lợi ích:

- Đồng nhất cách key hoá dữ liệu server-state theo search.
- Dễ logging/monitoring (chỉ cần log theo prefix `['products', ...]`).

---

## 4. Integration Với API Client, Monitoring, Logging

### 4.1 API Client + Search

- URL search params là “contract” giữa client & backend.
- Ở cấp Principal, cần:
  - Chuẩn hoá cách map search params → backend query.
  - Giảm duplication mapping ở từng màn hình.

### 4.2 Monitoring & Logging

- Log các sự kiện quan trọng liên quan đến search:
  - Thay đổi search (trước/sau).
  - Search bất thường (URL quá dài, param invalid…).
- Gợi ý:
  - Hook vào router events / subscribe search changes (tham khảo pattern trong `ARCHITECTURE.md`).
  - Gửi dữ liệu sang hệ thống như Sentry, OpenTelemetry, custom dashboard.

---

## 5. Migration Strategies

Chiến lược điển hình:

- **Từ local state (useState) → URL-first**:
  - Bước 1: mapping state hiện tại → search schema tương đương.
  - Bước 2: wrap logic cũ vào domain hook sử dụng `useSearch`.
  - Bước 3: thay dần `useState` bằng search params.

- **Từ Redux/Zustand → URL-first cho phần UI state**:
  - Giữ lại store cho app state (auth, global settings).
  - Dời filter/pagination/search sang URL.
  - Đảm bảo behaviors cũ (back/forward, share link) không bị phá vỡ.

Chi tiết từng case có thể được mô tả trong `RESEARCH_SUMMARY.md` (Use Cases & Roadmap) và mở rộng dần khi bạn triển khai thực tế.

---

## 6. Liên Kết Tới Tài Liệu Chi Tiết

- **Kiến trúc & ví dụ cụ thể**: xem `ARCHITECTURE.md`.
- **Patterns & workflow thực tế**: xem `WORKFLOW_PATTERNS.md`.
- **Implementation step-by-step (cho team)**: xem `IMPLEMENT_GUIDE.md`.
- **Nền tảng lý thuyết**: xem `THEORY_GUIDE.md`.
- **Nguồn research & quyết định kiến trúc**: xem `RESEARCH_SUMMARY.md`.

---

**Status:** Skeleton định hướng cho Principal/Staff – sẽ được bổ sung dần theo kinh nghiệm thực tế và các quyết định kiến trúc trong dự án.


