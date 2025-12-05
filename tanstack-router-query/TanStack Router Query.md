# TanStack Router Query - Hướng Dẫn Toàn Diện

## Mục Lục

1. Giới Thiệu Tổng Quan
2. Junior Level – Cơ Bản
3. Middle Level – Trung Cấp
4. Senior Level – Nâng Cao (phần mở đầu)
5. Tài Liệu Tham Khảo

---

## Giới Thiệu Tổng Quan

### TanStack Router Query là gì?

**TanStack Router Query** (search params) là lớp quản lý state dựa trên URL của TanStack Router. Nó tập trung vào:

- Quản lý search params (query string) một cách **type-safe**.
- **Validate** và **chuẩn hoá** dữ liệu URL với schema (Zod, Valibot, ArkType).
- Đồng bộ hai chiều giữa **URL ↔ UI state**.
- Cho phép **bookmark/share** state thông qua URL, hỗ trợ browser back/forward tự nhiên.

### Khi nào nên dùng TanStack Router Query?

- Khi bạn có:
  - Bộ lọc (filters), sort, pagination.
  - Search nâng cao, nhiều tham số.
  - Multi-step form cần **khôi phục từ URL**.
  - Dashboard / báo cáo cần **URL shareable**.
- Và bạn muốn:
  - Đồng bộ UI state với URL.
  - Không tự tay parse/serialize `window.location.search`.
  - Được bảo vệ bởi type-safety và validation rõ ràng.

Các khái niệm cốt lõi:

- **Search Params / Query**: phần sau dấu `?` trong URL.
- **Validation Schema**: Zod/Valibot/ArkType mô tả cấu trúc và kiểu của search.
- **parseSearch / stringifySearch**: chuyển đổi giữa URL string và object.
- **useSearch / useNavigate**: API React để đọc/ghi search params.

> Ghi chú: Phần lý thuyết chi tiết hơn (URL state, validation pipeline, serialization…) đã có trong `THEORY_GUIDE.md`. File này đóng vai trò “tổng quan + lộ trình theo level”.

---

## Junior Level – Cơ Bản

### 1. Hello World: Search Params cơ bản

Ví dụ tối giản: route `/products` với `page`, `limit`, `category`, `sort`, `ascending`:

```typescript
import { z } from 'zod'
import { createFileRoute } from '@tanstack/react-router'
import { zodValidator } from '@tanstack/zod-adapter'

const searchSchema = z.object({
  page: z.number().int().positive().default(1),
  limit: z.number().int().positive().default(10),
  category: z.string().optional(),
  sort: z.enum(['name', 'price', 'date']).default('name'),
  ascending: z.boolean().default(true),
})

export const Route = createFileRoute('/products')({
  validateSearch: zodValidator(searchSchema),
})

function ProductsPage() {
  const { page, limit, category, sort, ascending } = Route.useSearch()

  return (
    <div>
      <h1>Products - Page {page}</h1>
      <p>Category: {category || 'All'}</p>
      <p>
        Sort: {sort} ({ascending ? 'ASC' : 'DESC'})
      </p>
    </div>
  )
}
```

**Điểm chính cho Junior:**

- Thấy rõ: `validateSearch` bảo vệ URL → `Route.useSearch()` luôn trả về object **đã validate**.
- Toàn bộ state UI (page, limit, category, sort, ascending) đến từ URL, không cần `useState`.

### 2. Điều khiển search params từ UI

```typescript
import { Link, useNavigate } from '@tanstack/react-router'

function ProductsNavigation() {
  const navigate = useNavigate({ from: '/products' })

  return (
    <div>
      {/* Link với search cố định */}
      <Link to="/products" search={{ page: 2, category: 'electronics' }}>
        Page 2 - Electronics
      </Link>

      {/* Navigate dạng function để giữ lại params cũ */}
      <button
        onClick={() =>
          navigate({
            search: (prev) => ({ ...prev, page: prev.page + 1 }),
          })
        }
      >
        Next Page
      </button>
    </div>
  )
}
```

**Pitfall điển hình (Junior nên nhớ):**

- ❌ Ghi đè toàn bộ search:

```typescript
navigate({ search: { page: 2 } }) // mất category, sort, limit...
```

- ✅ Dùng functional update:

```typescript
navigate({ search: (prev) => ({ ...prev, page: 2 }) })
```

---

## Middle Level – Trung Cấp

### 1. Pagination + Filter + Sort trong một hook

Ý tưởng: gom toàn bộ logic search vào 1 custom hook domain-specific, để component UI chỉ gọi hàm:

```typescript
// useProductSearch.ts
import { useNavigate } from '@tanstack/react-router'
import { Route } from './__generated__/productsRoute' // ví dụ
import type { ProductsSearch } from './schemas'

export function useProductSearch() {
  const search = Route.useSearch()
  const navigate = useNavigate({ from: Route.fullPath })

  const updateSearch = (updates: Partial<ProductsSearch>) => {
    navigate({
      search: (prev) => ({ ...prev, ...updates }),
    })
  }

  return {
    search,
    goToPage: (page: number) => updateSearch({ page }),
    setCategory: (category?: string) => updateSearch({ category, page: 1 }),
    setSort: (sortBy: string, order: 'asc' | 'desc') =>
      updateSearch({ sortBy, order }),
    clearFilters: () =>
      navigate({
        search: {
          page: 1,
          limit: 10,
          sortBy: 'createdAt',
          order: 'desc',
        },
      }),
  }
}
```

Component chỉ tập trung render:

```typescript
function ProductsPage() {
  const { search, goToPage, setCategory, setSort, clearFilters } =
    useProductSearch()

  // fetchProductsFromSearch là hàm domain của bạn
  const { products, total } = fetchProductsFromSearch(search)

  // render filters + list + pagination...
}
```

**Middle-level takeaway:**

- Đưa toàn bộ **search navigation logic** vào custom hook.
- Giữ component UI **thuần trình bày**.

### 2. Debounce search input (tránh spam URL)

```typescript
import { useEffect, useState } from 'react'
import { useProductSearch } from './useProductSearch'

export function SearchInput() {
  const { search, updateSearch } = useProductSearch()
  const [input, setInput] = useState(search.search ?? '')

  useEffect(() => {
    const timer = setTimeout(() => {
      updateSearch({ search: input || undefined, page: 1 })
    }, 300)

    return () => clearTimeout(timer)
  }, [input])

  return (
    <input
      value={input}
      onChange={(e) => setInput(e.target.value)}
      placeholder="Search products..."
    />
  )
}
```

---

## Senior Level – Nâng Cao (phần mở đầu)

Các phần Senior chi tiết hơn (middleware, search history, undo/redo, diff/merge, perf tuning) được tập trung trong `Advanced-Patterns.md`. Dưới đây chỉ là “cái nhìn tổng quan”.

### 1. Selective subscription để tối ưu re-render

```typescript
// Re-render khi BẤT KỲ search param nào đổi
const search = Route.useSearch()

// Re-render CHỈ khi page đổi
const page = Route.useSearch({ select: (s) => s.page })
```

Ý tưởng:

- Component nào chỉ quan tâm `page` / `sort` thì chỉ subscribe vào đúng trường đó.
- Giảm số lần re-render khi có nhiều param thay đổi cùng lúc.

### 2. stripSearchParams để dọn URL

```typescript
import { stripSearchParams } from '@tanstack/react-router'

export const Route = createFileRoute('/products')({
  validateSearch: zodValidator(searchSchema),
  search: {
    middlewares: [
      stripSearchParams({
        page: 1,
        limit: 10,
        sortBy: 'createdAt',
        order: 'desc',
      }),
    ],
  },
})
```

- Các giá trị mặc định không cần thiết sẽ bị loại khỏi URL → URL ngắn, dễ đọc.
- Object đầy đủ vẫn có đủ field nhờ schema default.

### 3. Kết hợp với server-state (TanStack Query)

```typescript
import { useQuery } from '@tanstack/react-query'

function ProductsWithQuery() {
  const search = Route.useSearch()

  const { data, isLoading } = useQuery({
    queryKey: ['products', search],
    queryFn: () => fetchProducts(search),
  })

  // UI...
}
```

- **Router search** làm nguồn sự thật cho UI state (filters, pagination).
- **TanStack Query** quản lý server-state và caching, keyed theo `search`.

---

## Tài Liệu Tham Khảo

- Lý thuyết chi tiết: xem `THEORY_GUIDE.md`.
- Kiến trúc, DTO, use case thực tế: xem `ARCHITECTURE.md`.
- Implementation step-by-step: xem `IMPLEMENT_GUIDE.md`.
- Workflow patterns & best practices: xem `WORKFLOW_PATTERNS.md`.
- Research & nguồn tham khảo: xem `RESEARCH_SUMMARY.md`.

External:

- TanStack Router Docs: https://tanstack.com/router
- Search Params Guide: https://tanstack.com/router/latest/docs/framework/react/guide/search-params
- useSearch Hook API: https://tanstack.com/router/latest/docs/framework/react/api/router/useSearchHook
- Validation Guide: https://tanstack.com/router/latest/docs/framework/react/how-to/validate-search-params

---

**Phiên bản:** draft – sẽ tiếp tục hoàn thiện dần theo cùng phong cách với `TanStack Query.md`.


