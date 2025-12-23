Việc kết hợp **TanStack Router (File-based routing)** với **Feature-Sliced Design (FSD)** là một bài toán thú vị vì hai mô hình này có tư duy hơi trái ngược nhau về cấu trúc thư mục:

* **TanStack Router (File-based):** Yêu cầu một thư mục `routes` cụ thể, nơi cấu trúc file quy định URL.
* **FSD:** Yêu cầu chia theo lớp (`pages`, `features`, `entities`...), nơi logic được đóng gói theo nghiệp vụ chứ không phải theo URL.

Để giải quyết vấn đề này mà không phá vỡ nguyên tắc FSD, giải pháp tốt nhất là sử dụng mô hình **"Proxy Route" (Route ủy quyền)**. Thư mục `routes` của TanStack Router sẽ chỉ đóng vai trò là lớp vỏ (shell) định nghĩa URL và Loader, sau đó import giao diện thực tế từ lớp `pages` của FSD.

Dưới đây là hướng dẫn cấu hình chi tiết:

---

### 1. Cấu trúc thư mục đề xuất

Trong FSD, chúng ta sẽ giữ nguyên các lớp tiêu chuẩn. Chúng ta sẽ đặt thư mục `routes` (nơi TanStack Router scan) nằm ngang hàng với `app`, `pages`, `widgets`... hoặc coi nó là một phần cấu hình hạ tầng.

```text
src/
├── app/                  # Providers, Global styles, entry point
├── entities/             # Business entities (User, Product...)
├── features/             # User interactions (Auth, Filter...)
├── pages/                # FSD Pages (UI composition)
│   ├── home/             # Slice cho trang Home
│   │   └── ui/
│   │       └── Page.tsx  # Component hiển thị chính
│   └── product-detail/
│       └── ui/
│           └── Page.tsx
├── shared/               # UI Kit, libs, api client
└── routes/               # TANSTACK ROUTER (File-based routing folder)
    ├── __root.tsx
    ├── index.tsx         # Route "/" -> trỏ về pages/home
    └── products/
        └── $id.tsx       # Route "/products/:id" -> trỏ về pages/product-detail

```

### 2. Cấu hình `tsr.config.json`

Bạn cần chỉ định cho TanStack Router CLI biết thư mục `routes` nằm ở đâu.

```json
{
  "routesDirectory": "./src/routes",
  "generatedRouteTree": "./src/routeTree.gen.ts",
  "quoteStyle": "single"
}

```

### 3. Thực hiện mô hình "Proxy Route"

Đây là bước quan trọng nhất. File trong `src/routes` **không nên chứa UI logic**. Nó chỉ nên làm 3 việc:

1. Định nghĩa params/search params (Validate).
2. Định nghĩa Loader (Data fetching requirements).
3. Lazy import component từ `src/pages`.

#### Ví dụ 1: Trang chủ (`src/routes/index.tsx`)

**Bước 3.1: Tạo Component trong FSD Page Layer**
File: `src/pages/home/ui/Page.tsx`

```tsx
import { ProductList } from '@/widgets/product-list';

export const HomePage = () => {
  return (
    <main>
      <h1>Welcome Home</h1>
      <ProductList />
    </main>
  );
};

```

**Bước 3.2: Tạo Route File**
File: `src/routes/index.tsx`

```tsx
import { createFileRoute } from '@tanstack/react-router';
// Import component từ FSD Layer
import { HomePage } from '@/pages/home/ui/Page'; 

export const Route = createFileRoute('/')({
  component: HomePage, // Gán trực tiếp component FSD vào đây
});

```

> **Tối ưu hóa (Code Splitting):** Để đảm bảo hiệu năng tốt hơn, bạn nên dùng tính năng `.lazy.tsx` của TanStack Router để tách code (chunking).

**Cách tối ưu với `.lazy.tsx`:**

1. File `src/routes/index.tsx` (Chỉ chứa Loader và Meta):
```tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/')({
  // Loader logic nếu cần (sẽ bàn ở mục 4)
})

```


2. File `src/routes/index.lazy.tsx` (Chứa Component UI):
```tsx
import { createLazyFileRoute } from '@tanstack/react-router'
import { HomePage } from '@/pages/home/ui/Page' // Import từ FSD

export const Route = createLazyFileRoute('/')({
  component: HomePage,
})

```



### 4. Xử lý Data Loading (Loader) trong FSD

Trong Render-as-you-fetch, việc fetch data xảy ra ở Router level. Tuy nhiên, logic gọi API (`api request`) nên nằm trong `shared/api` hoặc `entities`.

Ví dụ với trang chi tiết sản phẩm:

File: `src/routes/products/$id.tsx`

```tsx
import { createFileRoute } from '@tanstack/react-router';
import { productApi } from '@/entities/product'; // Logic fetch nằm ở Entity
import { queryOptions } from '@tanstack/react-query';

// Định nghĩa Query Options để dùng chung (cho cả Router loader và useQuery trong component)
const productDetailQuery = (id: string) => queryOptions({
  queryKey: ['products', id],
  queryFn: () => productApi.getProductById(id),
});

export const Route = createFileRoute('/products/$id')({
  // 1. Loader thực hiện fetch data trước khi render
  loader: ({ context: { queryClient }, params }) => 
    queryClient.ensureQueryData(productDetailQuery(params.id)),
    
  // 2. Component sẽ được lazy load từ file .lazy.tsx hoặc import trực tiếp nếu không tách file
});

```

File: `src/routes/products/$id.lazy.tsx`

```tsx
import { createLazyFileRoute } from '@tanstack/react-router';
import { ProductDetailPage } from '@/pages/product-detail/ui/Page';

export const Route = createLazyFileRoute('/products/$id')({
  component: ProductDetailPage,
});

```

### 5. Kết nối `Router` vào `App` (FSD Layer)

Cuối cùng, bạn khởi tạo Router ở lớp `app`.

File: `src/app/providers/RouterProvider.tsx`

```tsx
import { createRouter, RouterProvider as TanStackRouterProvider } from '@tanstack/react-router';
import { routeTree } from '../../routeTree.gen'; // File được sinh tự động
import { QueryClient } from '@tanstack/react-query';

const queryClient = new QueryClient();

// Tạo router instance
const router = createRouter({
  routeTree,
  context: {
    queryClient, // Inject queryClient vào context để loader sử dụng
  },
  defaultPreload: 'intent',
});

export const AppRouterProvider = () => {
  return <TanStackRouterProvider router={router} />;
};

```

### Tổng kết

Cách tiếp cận này mang lại lợi ích kép:

1. **Tuân thủ FSD:** Logic nghiệp vụ, UI, và API vẫn nằm gọn trong `src/pages`, `src/features`, `src/entities`. Thư mục `src/routes` chỉ chứa các file cấu hình mỏng (thin wrappers).
2. **Tận dụng TanStack Router:** Bạn vẫn được hưởng lợi từ Type-safe routing, Automatic route generation, và cơ chế Loader mạnh mẽ.

**Lưu ý:** Đừng bao giờ viết JSX phức tạp (như chia layout, logic state) ngay trong file `src/routes/*.tsx`. Hãy luôn coi file trong `routes` chỉ là nơi **"đấu nối dây điện"** (wiring) giữa URL và các component FSD.