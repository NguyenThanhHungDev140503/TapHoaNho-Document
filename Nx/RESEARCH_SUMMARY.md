# Nx Research Summary

## 1. Mục Tiêu Research

**Tại sao nghiên cứu thư viện này?**
- Nx là công cụ quản lý monorepo mạnh mẽ, hỗ trợ đa ngôn ngữ (React, .NET, Angular, Node.js)
- Cần hiểu cách tích hợp Nx vào dự án hiện tại có cả React (frontend) và .NET (backend)
- Mong muốn tối ưu hóa CI/CD với caching, distributed task execution
- Cần kiến trúc monorepo scale tốt cho nhiều team và nhiều dự án

**Dùng trong project nào, mục đích gì?**
- Project: TapHoaNho (Retail Store Management)
- Mục đích: 
  - Quản lý monorepo chứa nhiều applications (React frontend, .NET WebAPI, mobile app)
  - Tối ưu hóa build time với caching
  - Tự động hóa CI/CD với distributed tasks
  - Chuẩn hóa code generation và scaffolding

---

## 2. Nguồn Thông Tin Đã Sử Dụng

### Official Documentation
| Tiêu đề | URL | Ngày truy cập | Ghi chú |
|---------|-----|---------------|---------|
| Nx React Monorepo Tutorial | https://nx.dev/docs/getting-started/tutorials/react-monorepo-tutorial | 03/01/2026 | Hướng dẫn setup React monorepo với workspace structure apps/, libs/ |
| Nx Angular Monorepo Tutorial | https://nx.dev/docs/getting-started/tutorials/angular-monorepo-tutorial | 03/01/2026 | Giống React, hướng dẫn setup Angular workspace |
| TypeScript Monorepo Maintenance | https://nx.dev/docs/features/maintain-typescript-monorepos | 03/01/2026 | TypeScript Project References với workspaces từ npm/yarn/pnpm |
| Folder Structure Decisions | https://nx.dev/docs/concepts/decisions/folder-structure | 03/01/2026 | Gợi ý cách tổ chức projects theo scope/application |
| Adding Nx to Monorepo | https://nx.dev/docs/guides/adopting-nx/adding-to-monorepo | 03/01/2026 | Hướng dẫn thêm Nx vào existing npm/yarn/pnpm workspace |
| Nx .NET Introduction | https://nx.dev/docs/technologies/dotnet/introduction | 03/01/2026 | Plugin @nx/dotnet cho quản lý .NET projects |
| Nx .NET Generators | https://nx.dev/docs/technologies/dotnet/generators | 03/01/2026 | Các generators cho .NET (init, lib, app, test) |
| Nx React Generators | https://nx.dev/docs/technologies/react/generators | 03/01/2026 | Generators cho React (app, lib, host/remote cho Module Federation) |
| Nx Plugin Registry | https://nx.dev/docs/plugin-registry | 03/01/2026 | Danh sách plugins chính thức cho Angular, React, Vue, Next.js, Express, Playwright |
| Nx Cloud Setup | https://nx.dev/docs/getting-started/tutorials/react-monorepo-tutorial (Nx Cloud setup section) | 03/01/2026 | Remote caching và self-healing trong CI |
| Vite Configuration | https://nx.dev/docs/technologies/build-tools/vite/guides/configure-vite | 03/01/2026 | Config Vite trong monorepo với cacheDir tùy chỉnh |

### Blog Posts & Articles
| Tiêu đề | URL | Ngày truy cập | Ghi chú |
|---------|-----|---------------|---------|
| Building a Scalable Data-Layered React Architecture with Nx | https://medium.com/@boxofkarthi/building-a-scalable-data-layered-react-architecture-with-nx-c0b606651e30 | 03/01/2026 | Multi-layer architecture với Nx: Data Access, Business Logic, UI layers |
| Monorepo Architecture on LinkedIn | https://www.linkedin.com/pulse/full-stack-mobile-development-explained-how-one-team-builds-qr12f | 03/01/2026 | Benefits của monorepo: shared components, design tokens, utilities |

### Open Source Projects (Code Examples)
| Repo | URL | Ngày truy cập | Ghi chú |
|------|-----|---------------|---------|
| MyMoney (Nx + .NET + React/Angular) | https://github.com/RelativeForce/MyMoney | 03/01/2026 | ASP.NET Core 6.0 backend + Nx monorepo frontend (React & Angular) |
| Monorepo Showcase (Nx + Flutter + .NET + React) | https://github.com/Pablo-gitub/monorepo-showcase | 03/01/2026 | Polyglot monorepo với Flutter, .NET, React, clean architecture |
| Nx React Monorepo Netflix Deploy | https://github.com/coder-do/nx-react-monorepo-netflix-deploy | 03/01/2026 | React app đơn giản với deploy lên Netlify |
| Nx Console GitHub Issues | https://github.com/naver/fe-news/blob/0664ce8de464318997b8020b50df15c9cce1bec7/issues/2022-02.md | 03/01/2026 | Nx Console UI enhancements và features |

---

## 3. Phát Hiện Chính (Key Findings)

### Ưu điểm của Nx
1. **Smart Caching**: Local và remote caching giúp giảm build time đáng kể (90%+ cache hit rate cho unchanged code)
2. **Distributed Task Execution**: Parallel chạy tasks trên nhiều agents trong CI
3. **Project Graph Awareness**: Tự động phát hiện dependencies giữa projects, chạy tasks theo thứ tự tối ưu
4. **Code Generation**: Generators mạnh mẽ với các plugins (@nx/react, @nx/dotnet, @nx/angular, etc.)
5. **Multi-language Support**: Hỗ trợ React, Angular, Vue, Node.js, .NET, Go, Rust, Python, v.v.
6. **Workspaces Integration**: Hoạt động tốt với npm, yarn, pnpm workspaces
7. **Nx Cloud**: Remote caching, distributed execution, self-healing CI pipelines
8. **Modular Architecture**: Encourage chia nhỏ projects thành libs reusable

### Hạn chế của Nx
1. **Learning Curve**: Cần thời gian để hiểu concepts (projects, libs, apps, generators, executors, plugins)
2. **Complexity cho small teams**: Có thể overkill cho projects nhỏ hoặc solo dev
3. **Migration Cost**: Migrate existing monorepo sang Nx cần effort
4. **Configuration Complexity**: nx.json, project.json có thể phức tạp với nhiều custom configs
5. **Plugin Dependencies**: Một số plugins có thể không cập nhật thường xuyên hoặc có bugs
6. **CI Integration**: Cần setup carefully cho CI/CD pipelines để tận dụng caching

### So sánh với các tools khác

| Tính năng | Nx | Turborepo | Lerna | pnpm workspaces |
|-----------|-----|-----------|-------|------------------|
| Caching | ✅ (local + remote via Nx Cloud) | ✅ (local + remote) | ❌ | ❌ |
| Distributed Execution | ✅ (via Nx Cloud) | ✅ | ❌ | ❌ |
| Code Generation | ✅ (generators/plugins) | ❌ | ⚠️ (limited) | ❌ |
| Multi-language | ✅ (React, .NET, Go, Rust, etc.) | ⚠️ (JS/TS focus) | ❌ (JS/TS only) | ❌ (JS/TS only) |
| Project Graph | ✅ (auto-detect) | ✅ | ⚠️ (manual) | ❌ |
| CLI | ✅ (nx run, nx g, etc.) | ✅ (turbo) | ⚠️ (lerna run) | ⚠️ (pnpm exec) |
| Learning Curve | ⚠️ (medium) | ⚠️ (easy-medium) | ⚠️ (easy) | ✅ (easy) |

**Kết luận**: Nx phù hợp nhất cho **enterprise monorepos** với:
- Nhiều languages (JS/TS + .NET + Go, v.v.)
- Cần caching và distributed execution
- Muốn tận dụng code generation
- Teams lớn, nhiều developers

---

## 4. Kiến Trúc/Cách Dùng Đề Xuất

### Kiến trúc monorepo với Nx cho TapHoaNho

```
TapHoaNho/
├── apps/                          # Applications (deployable units)
│   ├── frontend/                   # React app (shiny-carnival/frontend/)
│   ├── mobile/                     # React Native app (tương lai)
│   └── webapi/                    # .NET WebAPI (CORE-MOBILE-APP/WebApi/)
├── libs/                          # Shared libraries
│   ├── ui/                        # Shared UI components
│   ├── shared-types/                # TypeScript/DTO types
│   ├── api-client/                 # API client layer
│   └── utils/                     # Utilities and helpers
├── backend/                       # .NET backend (CORE-MOBILE-APP/)
│   ├── Application/
│   ├── Domain/
│   ├── Infrastructure/
│   └── Shared/
├── tools/                         # Build tools and scripts
│   ├── eslint-rules/
│   └── generators/
├── nx.json                        # Nx workspace configuration
├── package.json                    # Root package.json (workspaces)
├── tsconfig.base.json              # Base TypeScript config
└── tsconfig.json                  # Root TypeScript config
```

### Pattern sử dụng Nx trong hệ thống

**1. Workspace Setup**
- Sử dụng `create-nx-workspace@latest --preset=react-monorepo` cho initial setup
- Hoặc thêm Nx vào existing monorepo với `nx add @nx/react @nx/dotnet`
- Configure workspaces trong package.json: `"workspaces": ["apps/*", "libs/*", "backend/*"]`

**2. Project Organization**
- **apps/**: Deployable applications (React frontend, .NET WebAPI)
- **libs/**: Shared libraries (UI components, types, utilities)
- **backend/**: .NET projects theo Clean Architecture
- Mỗi project có `project.json` với targets (build, test, lint, serve)

**3. Caching Strategy**
- Enable local caching trong `nx.json`
- Sử dụng Nx Cloud cho remote caching và distributed execution
- Configure cache inputs để tránh over-caching

**4. CI/CD Integration**
- Sử dụng `nx affected` để chỉ build/test changed projects
- Parallel execution trong CI với `--parallel` flag
- Distributed tasks với Nx Cloud agents

**5. Code Generation**
- Tạo custom generators cho:
  - New feature scaffolding
  - CRUD operations (React components + .NET endpoints)
  - Shared types/interfaces
- Sử dụng `@nx/react` và `@nx/dotnet` generators cho standard operations

### Patterns nên tránh

1. **❌ Quá nhiều nested folders**: Giữ cấu trúc phẳng (apps/, libs/, backend/) để dễ quản lý
2. **❌ Hardcoding paths**: Sử dụng Nx's dependency injection để truy cập các libs
3. **❌ Disabling caching**: Chỉ disable khi cần thiết (tests, deployments)
4. **❌ Manually managing dependencies**: Để Nx auto-detect project graph
5. **❌ Over-engineering generators**: Giữ generators đơn giản, dễ maintain

---

## 5. Use Cases Đã Xác Định

### 1. React Frontend Development
**Location**: [Nx.md](#junior-level---cơ-bản)
- Setup React app trong Nx monorepo
- Tạo shared UI components library
- Code generation cho React components, hooks
- Build và test với Vite/Webpack
- Linting với ESLint

### 2. .NET Backend Development
**Location**: [Nx.md](#junior-level---cơ-bản)
- Setup .NET projects với @nx/dotnet plugin
- Tạo class libraries và WebAPI projects
- Build và test .NET solutions
- Integration với React frontend (shared types)

### 3. Cross-platform Mobile (React Native)
**Location**: [Nx.md](#middle-level---trung-cấp)
- Tạo React Native app trong monorepo
- Chia sẻ business logic với React web
- Tạo native modules khi cần

### 4. API Client Layer Architecture
**Location**: [Advanced-Patterns.md](#custom-api-client-architecture)
- Tạo centralized API client library
- Shared types/interfaces với .NET backend
- Error handling và retry logic
- Request/response interceptors

### 5. Clean Architecture với .NET
**Location**: [Principal-Level-Patterns.md](#clean-architecture-with-nx)
- Domain, Application, Infrastructure, WebApi layers
- Dependency injection với Nx
- Shared types giữa React và .NET

### 6. CI/CD Optimization
**Location**: [Principal-Level-Patterns.md](#caching-strategies-for-ci-cd)
- Nx Cloud integration
- Distributed task execution
- Affected projects optimization
- Self-healing CI pipelines

### 7. Module Federation (Micro-frontends)
**Location**: [Advanced-Patterns.md](#module-federation-with-react)
- Host và remote applications
- Shared dependencies
- Lazy loading và code splitting

### 8. Testing Strategy
**Location**: [Advanced-Patterns.md](#testing-in-nx-workspace)
- Unit tests (Vitest, xUnit)
- Integration tests (Supertest, ASP.NET Core testing)
- E2E tests (Playwright, Cypress)
- Test orchestration với Nx

---

## 6. Các Công cụ và Plugins Nx Quan Trọng

### Core Plugins
- **@nx/react**: React applications và libraries
- **@nx/angular**: Angular applications và libraries
- **@nx/dotnet**: .NET projects
- **@nx/node**: Node.js applications
- **@nx/nextjs**: Next.js framework
- **@nx/nuxt**: Nuxt.js framework
- **@nx/vite**: Vite bundler
- **@nx/webpack**: Webpack bundler
- **@nx/eslint**: ESLint configuration
- **@nx/jest**: Jest testing
- **@nx/vitest**: Vitest testing
- **@nx/playwright**: Playwright E2E testing

### Community Plugins
- **@nxext/nx-extensions**: Capacitor, Ionic, Stencil, Svelte, SolidJS, Preact
- **nx-biome**: Biome toolchain integration
- **@lucasvieirasilva/nx-plugins**: Python, data migrations
- **nxrocks**: Spring Boot, Flutter, Quarkus, Micronaut, Ktor

---

## 7. Kết Luận Research

Nx là lựa chọn phù hợp cho **TapHoaNho monorepo** vì:
1. Hỗ trợ cả **React (frontend)** và **.NET (backend)**
2. Caching và distributed execution giúp giảm build time
3. Code generation giúp tăng productivity
4. Project graph tự động giúp quản lý dependencies
5. Scalable cho future growth (mobile apps, microservices)

**Next Steps**:
- Migrate existing projects sang Nx workspace
- Setup Nx Cloud cho remote caching
- Tạo custom generators cho team patterns
- Configure CI/CD với Nx optimization
- Training team về Nx concepts và best practices
