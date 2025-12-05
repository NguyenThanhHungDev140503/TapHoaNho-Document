# TanStack Query - Research Summary

## Tổng Quan

Tài liệu này tổng hợp các nguồn tham khảo và thông tin mới nhất về TanStack Query v5 được sử dụng để cập nhật tài liệu.

## Thông Tin Version

- **Package Name**: `@tanstack/react-query`
- **Version Hiện Tại**: v5.90.11 (theo npm registry, tháng 1/2025)
- **DevTools Package**: `@tanstack/react-query-devtools` v5.91.1
- **Framework Support**: React, Vue, Solid, Svelte, Angular

## Xác Nhận Package Name

✅ **Package name `@tanstack/react-query` là chính xác và mới nhất**

- TanStack Query được rebrand từ React Query vào năm 2022
- Package name chính thức: `@tanstack/react-query`
- Tên cũ `react-query` đã không còn được maintain
- Tất cả documentation và examples đều sử dụng `@tanstack/react-query`

**Nguồn tham khảo:**
- [TanStack Query Official Docs](https://tanstack.com/query/latest)
- [npm registry - @tanstack/react-query](https://www.npmjs.com/package/@tanstack/react-query)
- [GitHub Repository](https://github.com/TanStack/query)

## Thay Đổi Quan Trọng trong v5

### 1. `cacheTime` → `gcTime`

**Thay đổi:**
- `cacheTime` (v4) → `gcTime` (v5)
- `gc` = "Garbage Collection" - thuật ngữ phổ biến trong computer science
- Mục đích: Làm rõ rằng đây là thời gian garbage collection, không phải thời gian cache

**Nguồn:**
- [Migration Guide v5](https://tanstack.com/query/latest/docs/react/guides/migrating-to-v5)
- [Important Defaults](https://tanstack.com/query/latest/docs/react/guides/important-defaults)

### 2. `isLoading` vs `isPending` trong v5

**Thay đổi:**
- `status: 'loading'` → `status: 'pending'`
- `isLoading` (v4) → `isPending` (v5) cho initial state
- `isInitialLoading` (v4) → `isLoading` (v5) = `isPending && isFetching`
- `isInitialLoading` bị deprecated và sẽ bị xóa trong version tiếp theo

**Giải thích:**
- `isPending`: true khi query chưa có data (initial state hoặc disabled query)
- `isLoading`: true khi `isPending && isFetching` (đang fetch lần đầu)
- `isFetching`: true khi đang fetch (bao gồm cả background refetch)

**Nguồn:**
- [Migration Guide v5 - Status Changes](https://tanstack.com/query/latest/docs/react/guides/migrating-to-v5#status-loading-has-been-changed-to-status-pending)
- [Background Fetching Indicators](https://tanstack.com/query/latest/docs/react/guides/background-fetching-indicators)

### 3. Object Signature Bắt Buộc

**Thay đổi:**
- v4: Hỗ trợ cả positional arguments và object signature
- v5: Chỉ hỗ trợ object signature

```typescript
// v4 (deprecated trong v5)
useQuery(key, fn, options)
useMutation(fn, options)

// v5 (required)
useQuery({ queryKey, queryFn, ...options })
useMutation({ mutationFn, ...options })
```

**Nguồn:**
- [Migration Guide v5 - Object Signature](https://tanstack.com/query/latest/docs/react/guides/migrating-to-v5)

### 4. `context` → `queryClient`

**Thay đổi:**
- v4: Sử dụng `context` prop để pass custom query client
- v5: Pass `queryClient` instance trực tiếp như tham số thứ 2

```typescript
// v4
useQuery({ queryKey, queryFn, context: customContext })

// v5
useQuery({ queryKey, queryFn }, queryClient)
```

**Nguồn:**
- [Migration Guide v5 - Context Changes](https://tanstack.com/query/latest/docs/react/guides/migrating-to-v5)

## Nguồn Tham Khảo Chính

### Official Documentation

1. **TanStack Query Official Docs**
   - URL: https://tanstack.com/query/latest
   - Mô tả: Tài liệu chính thức đầy đủ nhất
   - Cập nhật: Liên tục

2. **Installation Guide**
   - URL: https://tanstack.com/query/latest/docs/framework/react/installation
   - Nội dung: Hướng dẫn cài đặt cho React
   - Package: `@tanstack/react-query`

3. **Migration Guide v5**
   - URL: https://tanstack.com/query/latest/docs/react/guides/migrating-to-v5
   - Nội dung: Hướng dẫn migrate từ v4 sang v5
   - Breaking changes chi tiết

4. **Important Defaults**
   - URL: https://tanstack.com/query/latest/docs/react/guides/important-defaults
   - Nội dung: Giải thích về staleTime, gcTime, và các defaults quan trọng

5. **API Reference**
   - URL: https://tanstack.com/query/latest/docs/react/reference/useQuery
   - Nội dung: API reference đầy đủ cho tất cả hooks

### Community Resources

1. **GitHub Repository**
   - URL: https://github.com/TanStack/query
   - Mô tả: Source code, issues, discussions
   - Stars: 40k+ (tháng 1/2025)

2. **npm Registry**
   - Package: https://www.npmjs.com/package/@tanstack/react-query
   - Version: v5.90.11 (latest)
   - Weekly Downloads: Rất cao (top package)

3. **Discord Community**
   - URL: https://discord.com/invite/WrRKjPJ
   - Mô tả: Community support và discussions

### Blog Posts & Articles

1. **Practical React Query** (TkDodo)
   - URL: https://tkdodo.eu/blog/practical-react-query
   - Tác giả: Dominik Dorfmeister (maintainer)
   - Nội dung: Best practices và patterns

2. **Effective React Query Keys**
   - URL: https://tkdodo.eu/blog/effective-react-query-keys
   - Nội dung: Best practices cho query keys

3. **React Query and TypeScript**
   - URL: https://tkdodo.eu/blog/react-query-and-type-script
   - Nội dung: TypeScript patterns và tips

### Research Tools Sử Dụng

1. **Context7**
   - Library ID: `/tanstack/query`
   - Mô tả: Documentation từ official source
   - Code snippets: 864+
   - Benchmark Score: 89

2. **Tavily Search**
   - Query: "TanStack Query v5 latest version package name @tanstack/react-query 2024 2025"
   - Kết quả: Xác nhận package name và version mới nhất

3. **npm Search**
   - Package: `@tanstack/react-query`
   - Version: v5.90.11
   - DevTools: v5.91.1

## Xác Nhận Tính Chính Xác

### ✅ Package Name
- **Đúng**: `@tanstack/react-query`
- **Sai**: `react-query` (deprecated)
- **Xác nhận**: Tất cả official docs và npm registry đều dùng `@tanstack/react-query`

### ✅ Version
- **Hiện tại**: v5.90.11 (tháng 1/2025)
- **Tài liệu cập nhật**: v5.90+ (latest)
- **Breaking changes**: Đã được document đầy đủ

### ✅ API Changes
- **Object signature**: Đã được cập nhật trong tất cả examples
- **isPending/isLoading**: Đã được giải thích rõ ràng
- **gcTime**: Đã thay thế cacheTime trong tất cả examples

## Kết Luận

Tài liệu TanStack Query đã được cập nhật với:
1. ✅ Package name chính xác: `@tanstack/react-query`
2. ✅ Version mới nhất: v5.90+
3. ✅ Breaking changes từ v4 → v5 đã được document
4. ✅ Tất cả examples sử dụng object signature (v5 requirement)
5. ✅ Giải thích rõ ràng về `isPending` vs `isLoading`
6. ✅ `gcTime` thay vì `cacheTime` trong tất cả examples

**Ngày cập nhật**: 2025-01-27
**Người cập nhật**: AI Assistant
**Nguồn**: Official TanStack Query Documentation, npm registry, Context7, Tavily Search

---

## Changelog

### 2025-01-27
- Cập nhật version từ v5 → v5.90+
- Thêm section về thay đổi v5 quan trọng
- Cập nhật giải thích về isPending vs isLoading
- Xác nhận package name `@tanstack/react-query` là chính xác
- Thêm links đến Important Defaults và Installation Guide
- Tạo RESEARCH_SUMMARY.md với nguồn tham khảo đầy đủ

