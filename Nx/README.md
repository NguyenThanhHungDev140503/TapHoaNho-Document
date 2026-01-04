# Nx Documentation

**Nx** là một build system và monorepo toolkit mạnh mẽ giúp developers quản lý, build, và scale large applications với features như task caching, code generation, và CI optimization.

---

## Cấu Trúc Tài Liệu

Bộ tài liệu Nx được tổ chức thành các file sau:

### 1. [Nx.md](./Nx.md)
**File chính** - Hướng dẫn toàn diện cho Junior, Middle, và Senior developers.
- **Junior Level**: Cài đặt, cấu hình cơ bản, commands cơ bản
- **Middle Level**: Features nâng cao, patterns, optimization
- **Senior Level**: Strategies nâng cao, integration với other tools

### 2. [Advanced-Patterns.md](./Advanced-Patterns.md)
**Patterns nâng cao** cho Senior developers.
- Custom configuration, caching strategies
- Performance optimization, testing nâng cao
- Custom plugins và generators

### 3. [Principal-Level-Patterns.md](./Principal-Level-Patterns.md)
**Enterprise/Principal patterns** cho Principal/Staff engineers.
- System design, enterprise architecture
- Migration strategies, production monitoring
- Advanced error recovery, scaling

### 4. [RESEARCH_SUMMARY.md](./RESEARCH_SUMMARY.md)
**Nhật ký research** và tóm tắt quá trình nghiên cứu.
- Nguồn thông tin (official docs, blogs, open-source)
- Key findings, so sánh với các tools khác
- Architecture recommendations, use cases

---

## Cách Sử Dụng Tài Liệu

### Cho Junior Developers
1. **Bắt đầu với** [Nx.md](./Nx.md#junior-level---cơ-bản)
   - Cài đặt Nx workspace
   - Hiểu basic concepts (apps, libs, projects, targets)
   - Học các commands cơ bản: `nx run`, `nx g`, `nx build`

2. **Thực hành với**:
   - Tạo React app trong monorepo
   - Tạo shared library
   - Chạy tests và builds

3. **Sau đó đọc**:
   - Các best practices trong [Nx.md](./Nx.md#junior-level---cơ-bản)
   - Common pitfalls và cách tránh

### Cho Middle Developers
1. **Nắm vững Junior content** trước
2. **Đọc tiếp trong** [Nx.md](./Nx.md#middle-level---trung-cấp):
   - Task orchestration (affected, parallel execution)
   - Caching strategies
   - Dependency management
   - Integration với .NET backend

3. **Deep dive vào**:
   - React patterns (Module Federation, shared components)
   - .NET integration với @nx/dotnet plugin
   - Testing strategies

4. **Mở rộng với** [Advanced-Patterns.md](./Advanced-Patterns.md):
   - Performance optimization
   - Custom hooks và middleware

### Cho Senior Developers
1. **Đọc hết** [Nx.md](./Nx.md) để hiểu đầy đủ
2. **Tập trung vào** [Advanced-Patterns.md](./Advanced-Patterns.md):
   - Custom plugins và generators
   - Advanced caching strategies
   - CI/CD optimization
   - Testing orchestration

3. **Tham khảo**:
   - Real-world examples từ open-source projects
   - Integration patterns với third-party tools

4. **Xem xét** migration và scaling:
   - Migrate existing monorepos
   - Optimize cho large teams

### Cho Principal/Staff Engineers
1. **Đọc hết tất cả** files trên để hiểu đầy đủ
2. **Tập trung vào** [Principal-Level-Patterns.md](./Principal-Level-Patterns.md):
   - Enterprise architecture với Nx
   - System design considerations
   - Migration strategies (monorepo to Nx)
   - Production monitoring và metrics
   - Team scaling patterns

3. **Quyết định**:
   - Khi nào dùng Nx vs Turborepo
   - Architecture patterns cho organization
   - CI/CD pipeline optimization
   - Training và onboarding strategies

---

## Key Concepts

### 1. Workspace
Workspace là root của Nx monorepo, chứa tất cả projects. Mỗi workspace có:
- **nx.json**: Configuration file
- **apps/**: Deployable applications
- **libs/**: Shared libraries
- **project.json**: Configuration cho mỗi project

### 2. Project Graph
Nx tự động xây dựng dependency graph giữa projects. Nó hiểu:
- Libraries phụ thuộc vào nhau như thế nào
- Applications phụ thuộc vào libraries nào
- Cách tối ưu task execution order

### 3. Targets và Executors
- **Targets**: Named operations (build, test, lint, serve)
- **Executors**: Code thực thi targets (webpack, vitest, dotnet build)

**Ví dụ**:
```json
// project.json
{
  "targets": {
    "build": {
      "executor": "@nx/vite:build",
      "options": {
        "outputPath": "dist/apps/my-app"
      }
    }
  }
}
```

### 4. Caching
Nx caches kết quả của tasks để tránh chạy lại:
- **Local caching**: Mặc định trên máy local
- **Remote caching**: Via Nx Cloud (chia sẻ giữa team members)

**Ví dụ**:
```bash
# Lần đầu chạy: 45 giây
nx build my-app

# Lần sau (không đổi code): <1 giây (from cache)
nx build my-app
```

### 5. Affected Projects
Nx chỉ chạy tasks trên projects bị ảnh hưởng bởi code changes:
```bash
# Chỉ test projects bị ảnh hưởng
nx affected --target=test

# Build và test changed projects
nx affected --target=build --target=test --parallel
```

---

## External Resources

### Official Documentation
- **Nx Official Docs**: https://nx.dev - Documentation chính thức và comprehensive
- **Nx GitHub**: https://github.com/nrwl/nx - Source code và issues
- **Nx Cloud**: https://nx.app - Remote caching và distributed execution

### Community Resources
- **Nx Console**: https://marketplace.visualstudio.com/items?itemName=nrwl.angular-console - VS Code extension cho Nx
- **Nx Discord**: https://discord.gg/nx - Community chat
- **Nx Twitter**: https://twitter.com/nxdevdev - Updates và news

### Examples and Tutorials
- **Nx Examples**: https://github.com/nrwl/nx-examples - Code examples cho various use cases
- **Nx Recipes**: https://nx.dev/recipes - Best practices và patterns
- **Blog Posts**: https://nx.dev/blog

---

## Quick Start

### Cài đặt Workspace Mới

```bash
# Tạo React monorepo mới
npx create-nx-workspace@latest --preset=react-monorepo

# Hoặc tạo workspace với nhiều preset
npx create-nx-workspace@latest --preset=apps
```

### Các Commands Cơ Bản

```bash
# Chạy task trên project
nx run my-app:build

# Tạo library mới
nx g @nx/react:lib my-lib

# Build affected projects
nx affected --target=build

# Test affected projects
nx affected --target=test --parallel

# Xem project graph
nx graph
```

### Workspace Structure

```
my-workspace/
├── apps/              # Deployable applications
│   ├── my-app/
│   └── another-app/
├── libs/              # Shared libraries
│   ├── ui/
│   └── utils/
├── nx.json           # Nx configuration
├── package.json       # Root package.json
└── tsconfig.base.json  # Base TypeScript config
```

---

## Khi Nên Sử Dụng Nx?

Nx phù hợp nhất cho:

✅ **Monorepos với nhiều projects** (frontend, backend, shared libs)
✅ **Teams lớn** cần collaboration và consistency
✅ **Multi-language stacks** (React + .NET + Go, v.v.)
✅ **Cần caching và distributed execution** để tối ưu CI/CD
✅ **Muốn code generation** để tăng productivity
✅ **Complex dependency graphs** cần smart orchestration

Nx có thể overkill cho:

❌ **Single project** hoặc small team
❌ **Simple apps** không cần advanced features
❌ **Teams** không muốn học curve mới

---

## Next Steps

1. **Đọc [Nx.md](./Nx.md)** để bắt đầu với basics
2. **Tham khảo [RESEARCH_SUMMARY.md](./RESEARCH_SUMMARY.md)** để hiểu research process
3. **Xem xét các examples** trong open-source projects
4. **Thử nghiệm với** small project trước khi áp dụng vào production
5. **Setup Nx Console** (VS Code extension) để tận dụng UI features
