# Devenv - Hướng dẫn Toàn diện

## Mục đích

Tài liệu này cung cấp hướng dẫn về **devenv.sh** - công cụ tạo môi trường phát triển nhanh, declarative, và reproducible sử dụng Nix.

---

## 1. Devenv là gì?

**Devenv** là một lớp abstraction trên Nix, giúp:
- ⚡ **Nhanh**: Kích hoạt môi trường dưới 100ms nhờ caching
- 📝 **Declarative**: Định nghĩa môi trường bằng cú pháp đơn giản
- 🔄 **Reproducible**: Cùng cấu hình = cùng môi trường trên mọi máy
- 🧩 **Composable**: Kết hợp nhiều cấu hình lại với nhau

### So sánh với Nix Flakes thuần

| Tiêu chí | Nix Flakes | Devenv |
|----------|------------|--------|
| Độ phức tạp | Cao | Thấp |
| Learning curve | Dốc | Dễ hơn |
| Services (DB, Redis) | Tự cấu hình | Có sẵn |
| Process management | Không có | Có sẵn |
| File cấu hình | `flake.nix` | `devenv.nix` + `devenv.yaml` |

---

## 2. Cài đặt Devenv

```bash
# Cài đặt devenv
nix profile install nixpkgs#devenv

# Kiểm tra
devenv --version
```

---

## 3. Khởi tạo Dự án

```bash
# Khởi tạo devenv trong thư mục dự án
devenv init
```

Lệnh này tạo ra các file:
- `devenv.nix` - Cấu hình môi trường
- `devenv.yaml` - Cấu hình inputs/dependencies
- `.envrc` - Tích hợp direnv
- `.gitignore` - Bỏ qua các file không cần thiết

---

## 4. Cấu trúc file `devenv.nix`

### 4.1. Cấu trúc cơ bản

```nix
{ pkgs, lib, config, inputs, ... }:
{
  # Các cấu hình ở đây
}
```

**Tham số:**
| Tham số | Mô tả |
|---------|-------|
| `pkgs` | Nixpkgs packages (ví dụ: `pkgs.git`) |
| `lib` | Nix library functions |
| `config` | Cấu hình devenv hiện tại |
| `inputs` | Dependencies từ `devenv.yaml` |

### 4.2. Các Option Chính

| Option | Mô tả |
|--------|-------|
| `packages` | Packages cài đặt |
| `env` | Biến môi trường |
| `enterShell` | Script chạy khi vào shell |
| `scripts` | Custom scripts |
| `languages` | Cấu hình ngôn ngữ |
| `services` | Cấu hình services |
| `processes` | Định nghĩa processes |
| `tasks` | Định nghĩa tasks |
| `git-hooks` | Git hooks tự động |

---

## 5. Chi tiết từng Option

### 5.1. `packages` - Cài đặt packages

```nix
{
  packages = [ 
    pkgs.git 
    pkgs.curl 
    pkgs.jq
    pkgs.nodejs_20 
    pkgs.dotnet-sdk_9 
  ];
}
```

### 5.2. `env` - Biến môi trường

```nix
{
  env.GREET = "Hello";
  env.DATABASE_URL = "postgresql://localhost:5432/mydb";
  env.NODE_ENV = "development";
}
```

### 5.3. `enterShell` - Script khi vào shell

```nix
{
  enterShell = ''
    echo "🚀 Development environment ready!"
    git --version
    node --version
  '';
}
```

### 5.4. `scripts` - Custom scripts

```nix
{
  # Script đơn giản
  scripts.build.exec = "yarn build";
  
  # Script với packages riêng
  scripts.analyze = {
    exec = ''
      curl "https://api.example.com" | jq '.data'
    '';
    packages = [ pkgs.curl pkgs.jq ];
    description = "Analyze API response";
  };
}
```

**Sử dụng:**
```bash
devenv shell
build        # Chạy script build
analyze      # Chạy script analyze
```

### 5.5. `languages` - Cấu hình ngôn ngữ

```nix
{
  # JavaScript/Node.js
  languages.javascript = {
    enable = true;
    package = pkgs.nodejs_20;
  };
  
  # .NET
  languages.dotnet = {
    enable = true;
    package = pkgs.dotnet-sdk_9;
  };
  
  # Python với venv
  languages.python = {
    enable = true;
    version = "3.11";
    venv.enable = true;
    venv.requirements = ''
      flask
      requests
    '';
  };
  
  # Rust
  languages.rust = {
    enable = true;
    channel = "stable";
  };
}
```

### 5.6. `services` - Cấu hình services

```nix
{
  # PostgreSQL
  services.postgres = {
    enable = true;
    package = pkgs.postgresql_16;
    initialDatabases = [{ name = "mydb"; }];
    extensions = extensions: [ 
      extensions.postgis 
    ];
    initialScript = "CREATE EXTENSION IF NOT EXISTS postgis;";
  };
  
  # Redis
  services.redis.enable = true;
  
  # MySQL
  services.mysql = {
    enable = true;
    initialDatabases = [{ name = "mydb"; }];
  };
}
```

**Services có sẵn:**
- PostgreSQL, MySQL, MariaDB
- Redis, Memcached
- RabbitMQ, Kafka
- Elasticsearch, OpenSearch
- MinIO (S3-compatible storage)
- Caddy, Nginx
- Và nhiều hơn nữa...

### 5.7. `processes` - Định nghĩa processes

```nix
{
  processes = {
    frontend.exec = "cd frontend && yarn dev";
    backend.exec = "cd backend && dotnet run";
    worker.exec = "cd worker && python main.py";
  };
}
```

**Sử dụng:**
```bash
devenv up          # Chạy tất cả processes
devenv up frontend # Chạy process cụ thể
```

### 5.8. `tasks` - Định nghĩa tasks

```nix
{
  tasks = {
    "myproj:setup".exec = "yarn install && dotnet restore";
    "myproj:build".exec = "yarn build && dotnet build";
    
    # Task chạy trước khi vào shell
    "devenv:enterShell".after = [ "myproj:setup" ];
  };
}
```

### 5.9. `git-hooks` - Git hooks tự động

```nix
{
  git-hooks.hooks = {
    # Hooks có sẵn
    prettier.enable = true;
    eslint.enable = true;
    shellcheck.enable = true;
    
    # Custom hook
    custom-lint = {
      enable = true;
      name = "Custom Lint";
      entry = "yarn lint";
    };
  };
}
```

---

## 6. File `devenv.yaml`

### 6.1. Cấu trúc cơ bản

```yaml
inputs:
  nixpkgs:
    url: github:NixOS/nixpkgs/nixos-unstable
```

### 6.2. Thêm inputs từ GitHub

```yaml
inputs:
  nixpkgs:
    url: github:NixOS/nixpkgs/nixos-unstable
  
  my-package:
    url: github:username/repo
```

### 6.3. Imports từ các modules khác

```yaml
inputs:
  nixpkgs:
    url: github:NixOS/nixpkgs/nixos-unstable
  
  shared-devenv:
    url: github:myorg/shared-devenv

imports:
  - ./frontend
  - ./backend
  - shared-devenv/common
```

---

## 7. Profiles - Nhiều môi trường

```nix
{
  profiles = {
    backend.module = {
      services.postgres.enable = true;
      services.redis.enable = true;
      env.ENVIRONMENT = "backend";
    };
    
    frontend.module = {
      languages.javascript.enable = true;
      processes.dev.exec = "yarn dev";
      env.ENVIRONMENT = "frontend";
    };
    
    testing.module = {
      packages = [ pkgs.playwright pkgs.cypress ];
      env.NODE_ENV = "test";
    };
  };
}
```

**Sử dụng:**
```bash
devenv shell --profile backend
devenv shell --profile frontend
devenv shell --profile testing
```

---

## 8. Các lệnh Devenv

| Lệnh | Mô tả |
|------|-------|
| `devenv init` | Khởi tạo dự án mới |
| `devenv shell` | Vào môi trường dev |
| `devenv up` | Chạy tất cả processes |
| `devenv up <name>` | Chạy process cụ thể |
| `devenv test` | Chạy tests |
| `devenv update` | Cập nhật dependencies |
| `devenv search <name>` | Tìm packages |
| `devenv info` | Hiển thị thông tin |
| `devenv gc` | Garbage collection |
| `devenv container build` | Build container |
| `devenv container run` | Chạy container |

---

## 9. Container từ Devenv

### 9.1. Build container

```bash
# Build container chứa shell
devenv container build shell

# Build container chứa processes
devenv container build processes
```

### 9.2. Chạy container

```bash
# Chạy shell trong container
devenv container run shell

# Chạy processes trong container
devenv container run processes
```

### 9.3. Custom container

```nix
{
  containers.myapp = {
    name = "myapp";
    registry = "ghcr.io/myorg";
    copyToRoot = [ ./dist ];
    startupCommand = "node server.js";
  };
}
```

---

## 10. Ví dụ cho TapHoaNho

### 10.1. File `devenv.nix`

```nix
{ pkgs, ... }: {
  # Packages cơ bản
  packages = [ 
    pkgs.git 
    pkgs.curl 
    pkgs.jq 
  ];
  
  # Node.js cho frontend
  languages.javascript = {
    enable = true;
    package = pkgs.nodejs_20;
    yarn.enable = true;
  };
  
  # .NET cho backend
  languages.dotnet = {
    enable = true;
    package = pkgs.dotnet-sdk_9;
  };
  
  # PostgreSQL
  services.postgres = {
    enable = true;
    package = pkgs.postgresql_16;
    initialDatabases = [{ name = "taphoanhodb"; }];
  };
  
  # Processes
  processes = {
    frontend.exec = "cd frontend && yarn dev";
    backend.exec = "cd RetailStoreManagement && dotnet run";
  };
  
  # Scripts
  scripts = {
    build-all.exec = ''
      cd frontend && yarn build
      cd ../RetailStoreManagement && dotnet build
    '';
    test-all.exec = ''
      cd frontend && yarn test
      cd ../RetailStoreManagement && dotnet test
    '';
  };
  
  # Git hooks
  git-hooks.hooks = {
    prettier.enable = true;
    eslint.enable = true;
  };
  
  # Khi vào shell
  enterShell = ''
    echo "🚀 TapHoaNho Development Environment"
    echo "=================================="
    echo "📦 Tools: Node.js $(node --version), .NET $(dotnet --version)"
    echo "🗄️  Database: PostgreSQL ready on $PGHOST:$PGPORT"
    echo ""
    echo "🔧 Commands:"
    echo "  devenv up        - Start all services"
    echo "  build-all        - Build frontend & backend"
    echo "  test-all         - Run all tests"
  '';
}
```

### 10.2. File `devenv.yaml`

```yaml
inputs:
  nixpkgs:
    url: github:NixOS/nixpkgs/nixos-unstable
```

### 10.3. Sử dụng

```bash
# Vào môi trường
devenv shell

# Chạy tất cả services và processes
devenv up

# Build
build-all

# Test
test-all
```

---

## 11. Tham khảo

- [Devenv Documentation](https://devenv.sh/)
- [Devenv Options Reference](https://devenv.sh/reference/options/)
- [Devenv Services](https://devenv.sh/services/)
- [Devenv Languages](https://devenv.sh/languages/)

---

**Ngày tạo:** 2026-01-03  
**Dự án:** TapHoaNho - Retail Store Management System
