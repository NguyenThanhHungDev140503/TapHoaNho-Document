# TanStack Query Documentation

Tài liệu toàn diện về TanStack Query (React Query) từ cơ bản đến nâng cao, được viết bằng tiếng Việt với TypeScript examples.

## 📚 Cấu Trúc Tài Liệu

### 1. **TanStack Query.md** (File Chính)
Tài liệu chính bao gồm:
- **Junior Level**: Cơ bản về TanStack Query
  - Installation và setup
  - useQuery và useMutation basics
  - Query keys và caching
  - Error handling cơ bản
  - Common pitfalls

- **Middle Level**: Trung cấp
  - Query invalidation strategies
  - Optimistic updates
  - Pagination và infinite queries
  - Parallel và dependent queries
  - Advanced error handling

- **Senior Level**: Nâng cao (phần đầu)
  - Advanced caching strategies
  - Integration với Zustand
  - Production monitoring

### 2. **Advanced-Patterns.md**
Patterns nâng cao cho Senior developers:
- Custom Query Client configuration
- Query Key Factory pattern
- Custom hooks architecture
- Performance optimization techniques
  - Select optimization
  - Structural sharing
  - Prefetching strategies
  - Initial data patterns
- Testing strategies
  - Unit testing hooks
  - Integration testing với MSW
  - Component testing

### 3. **Principal-Level-Patterns.md**
Enterprise patterns cho Principal/Staff engineers:
- System design ở quy mô lớn
- Centralized API client với interceptors
- Advanced prefetching strategies
- SSR/SSG integration (Next.js)
- Streaming queries
- Migration strategies (Redux → TanStack Query)
- Production monitoring và metrics
- Advanced error recovery
- Best practices summary

## 🎯 Cách Sử Dụng Tài Liệu

### Cho Junior Developers
1. Đọc phần **Junior Level** trong `TanStack Query.md`
2. Thực hành với các examples cơ bản
3. Làm quen với useQuery và useMutation
4. Hiểu về query keys và caching

### Cho Middle Developers
1. Review Junior Level nếu cần
2. Đọc phần **Middle Level** trong `TanStack Query.md`
3. Học về optimistic updates và pagination
4. Thực hành với infinite queries

### Cho Senior Developers
1. Đọc phần **Senior Level** trong `TanStack Query.md`
2. Đọc toàn bộ `Advanced-Patterns.md`
3. Implement custom hooks architecture
4. Setup testing strategies
5. Optimize performance

### Cho Principal/Staff Engineers
1. Review tất cả levels trước
2. Đọc `Principal-Level-Patterns.md`
3. Design enterprise-scale architecture
4. Setup monitoring và alerting
5. Plan migration strategies

## 🔑 Key Concepts

### Query Keys
```typescript
// Hierarchical structure
['posts'] // All posts
['posts', 'list'] // Posts list
['posts', 'list', { status: 'published' }] // Filtered posts
['posts', 'detail', 1] // Specific post
```

### Caching Strategy
```typescript
staleTime: 1000 * 60 * 5  // Data fresh for 5 minutes
gcTime: 1000 * 60 * 10     // Keep in cache for 10 minutes
```

### Optimistic Updates
```typescript
onMutate: async (newData) => {
  await queryClient.cancelQueries({ queryKey })
  const previous = queryClient.getQueryData(queryKey)
  queryClient.setQueryData(queryKey, newData)
  return { previous }
}
```

## 📖 Code Examples

Tất cả examples trong tài liệu:
- ✅ Sử dụng TypeScript
- ✅ Include detailed explanations
- ✅ Show best practices
- ✅ Highlight common mistakes
- ✅ Provide real-world use cases

## 🔗 External Resources

### Official Documentation
- [TanStack Query Docs](https://tanstack.com/query/latest)
- [API Reference](https://tanstack.com/query/latest/docs/react/reference/useQuery)
- [Migration Guide](https://tanstack.com/query/latest/docs/react/guides/migrating-to-v5)

### Community
- [GitHub Repository](https://github.com/TanStack/query)
- [Discord Community](https://discord.com/invite/WrRKjPJ)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/react-query)

### Blog Posts
- [Practical React Query](https://tkdodo.eu/blog/practical-react-query)
- [Effective React Query Keys](https://tkdodo.eu/blog/effective-react-query-keys)
- [React Query and TypeScript](https://tkdodo.eu/blog/react-query-and-type-script)

## 🚀 Quick Start

```bash
# Install TanStack Query
npm install @tanstack/react-query

# Install DevTools (optional)
npm install @tanstack/react-query-devtools
```

```typescript
// Setup QueryClient
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'

const queryClient = new QueryClient()

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <YourApp />
    </QueryClientProvider>
  )
}
```

## 📝 Notes

- **Version**: TanStack Query v5
- **Language**: Tiếng Việt
- **Code**: TypeScript
- **Last Updated**: 2025-11-10

## 🤝 Contributing

Nếu bạn tìm thấy lỗi hoặc muốn đóng góp:
1. Tạo issue để discuss
2. Submit pull request với improvements
3. Follow coding standards trong examples

## 📄 License

Tài liệu này được tạo cho mục đích học tập và tham khảo.

---

**Happy Learning! 🎉**

