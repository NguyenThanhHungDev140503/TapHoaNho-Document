# TanStack Router Query – Advanced Patterns (Senior)

## Mục Lục

1. Tổng Quan
2. Workflow Patterns Nâng Cao
3. Search History, Undo/Redo, Presets
4. Performance Optimization
5. Testing Nâng Cao

> Ghi chú: Nhiều đoạn code trong file này được tách/gom lại từ `WORKFLOW_PATTERNS.md` và `IMPLEMENT_GUIDE.md` để dễ tra cứu theo góc nhìn “pattern”.

---

## 1. Tổng Quan

Advanced patterns tập trung vào:

- Xây dựng **API hooks** reuse được (filter + sort + paginate, wizard form…).
- Tối ưu UX với URL shareable, bookmarkable, undo/redo.
- Hạn chế re-render không cần thiết và tối ưu hoá việc cập nhật URL.

Nếu bạn chưa quen:

- Hãy đọc `TanStack Router Query.md` trước (Junior/Middle/Senior).
- Sau đó quay lại file này để áp dụng vào các use case thực tế.

---

## 2. Workflow Patterns Nâng Cao

### 2.1 Filter + Sort + Paginate (Table Pattern)

Pattern chuẩn cho e-commerce, admin dashboard, data table:

```typescript
// Schema (ví dụ)
const tableSearchSchema = z.object({
  page: z.number().default(1),
  limit: z.number().default(20),
  sortBy: z.string().default('createdAt'),
  order: z.enum(['asc', 'desc']).default('desc'),
  filters: z.record(z.string()).optional(),
  search: z.string().optional(),
})

// Hook domain
export function useTableSearch() {
  const search = Route.useSearch()
  const navigate = useNavigate()

  return {
    search,
    applyFilter: (key: string, value: string) => {
      navigate({
        search: (prev) => ({
          ...prev,
          filters: { ...prev.filters, [key]: value },
          page: 1, // luôn reset page khi filter đổi
        }),
      })
    },
    setSort: (sortBy: string) => {
      navigate({
        search: (prev) => ({
          ...prev,
          sortBy,
          order:
            prev.sortBy === sortBy
              ? prev.order === 'asc'
                ? 'desc'
                : 'asc'
              : 'desc',
        }),
      })
    },
    goToPage: (page: number) => {
      navigate({ search: (prev) => ({ ...prev, page }) })
    },
  }
}
```

Điểm nhấn:

- **Functional update** luôn được dùng.
- **Reset page** về 1 khi filter thay đổi.
- Toàn bộ behavior table (filter, sort, paginate) gom lại trong một hook.

---

## 3. Search History, Undo/Redo, Presets

### 3.1 Search History

Lưu lại lịch sử search để người dùng có thể quay lại state trước đó:

```typescript
export function useSearchHistory() {
  const search = Route.useSearch()
  const navigate = useNavigate()
  const [history, setHistory] = useState<SearchParams[]>([])

  useEffect(() => {
    setHistory((prev) => {
      const last = prev[prev.length - 1]
      const isDuplicate = last && JSON.stringify(last) === JSON.stringify(search)
      return isDuplicate ? prev : [...prev, search]
    })
  }, [search])

  return {
    history,
    goToHistoryItem: (index: number) => {
      navigate({ search: history[index] })
    },
  }
}
```

### 3.2 Undo / Redo với Search

```typescript
export function useSearchUndoRedo() {
  const search = Route.useSearch()
  const navigate = useNavigate()
  const [past, setPast] = useState<SearchParams[]>([])
  const [future, setFuture] = useState<SearchParams[]>([])

  const updateSearch = (next: SearchParams) => {
    setPast((prev) => [...prev, search])
    setFuture([])
    navigate({ search: next })
  }

  const undo = () => {
    if (past.length === 0) return
    const newPast = past.slice(0, -1)
    const current = past[past.length - 1]
    setFuture((prev) => [search, ...prev])
    setPast(newPast)
    navigate({ search: current })
  }

  const redo = () => {
    if (future.length === 0) return
    const current = future[0]
    const newFuture = future.slice(1)
    setPast((prev) => [...prev, search])
    setFuture(newFuture)
    navigate({ search: current })
  }

  return { updateSearch, undo, redo, canUndo: past.length > 0, canRedo: future.length > 0 }
}
```

### 3.3 Search Presets (Saved Views)

```typescript
export function useSearchPresets() {
  const navigate = useNavigate()

  const presets = {
    mostPopular: { sort: 'popularity', order: 'desc', page: 1 },
    newest: { sort: 'createdAt', order: 'desc', page: 1 },
    cheapest: { sort: 'price', order: 'asc', page: 1 },
  }

  return {
    applyPreset: (preset: keyof typeof presets) => {
      navigate({ search: presets[preset] })
    },
  }
}
```

---

## 4. Performance Optimization

### 4.1 Selective Subscription + Memoized Selector

```typescript
const selectPage = useCallback((search: SearchParams) => search.page, [])
const page = Route.useSearch({ select: selectPage })
```

- Selector được memo hoá để tránh tạo function mới mỗi lần render.
- Chỉ re-render component khi `page` thực sự thay đổi.

### 4.2 Debounce Navigation

```typescript
const navigate = useNavigate()

const debouncedNavigate = useMemo(
  () =>
    debounce((nextSearch: SearchParams) => {
      navigate({ search: nextSearch })
    }, 300),
  [navigate]
)
```

- Hữu ích khi bạn có UI thay đổi search liên tục (slider, range-input…).

### 4.3 Search Diff & Merge Utilities

```typescript
export function useSearchDiff() {
  const search = Route.useSearch()

  const diff = (other: SearchParams) => {
    const changes: Record<string, [unknown, unknown]> = {}
    for (const key in search) {
      if (search[key] !== other[key]) {
        changes[key] = [search[key], other[key]]
      }
    }
    return changes
  }

  const merge = (...searches: SearchParams[]) => {
    return searches.reduce((acc, curr) => ({ ...acc, ...curr }), search)
  }

  return { diff, merge }
}
```

---

## 5. Testing Nâng Cao

Ví dụ test cho search params (extract từ `WORKFLOW_PATTERNS.md`):

```typescript
describe('Product Search', () => {
  it('should update page in URL', () => {
    render(<ProductsPage />)
    const nextButton = screen.getByText('Next')
    fireEvent.click(nextButton)
    expect(window.location.search).toContain('page=2')
  })

  it('should validate search params', () => {
    const invalid = { page: 'invalid' }
    expect(() => searchSchema.parse(invalid)).toThrow()
  })

  it('should preserve search on navigation', () => {
    render(<ProductCard />)
    const link = screen.getByRole('link')
    expect(link).toHaveAttribute('href', expect.stringContaining('?page=1'))
  })
})
```

- Kiểm tra behavior:
  - URL đúng với ý nghĩa (page, filter…).
  - Validation schema hoạt động đúng (throw khi sai).
  - Search được preserve khi điều hướng.

---

**Status:** Draft – đã gom nhóm các pattern chính, có thể tiếp tục mở rộng thêm từ `WORKFLOW_PATTERNS.md` khi bạn refine theo từng dự án cụ thể.


