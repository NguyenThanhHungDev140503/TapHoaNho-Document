# Hướng dẫn Nix Flakes và Devenv cho TapHoaNho

## Mục lục

1. [Giới thiệu](#giới-thiệu)
2. [Nix là gì?](#nix-là-gì)
3. [Nix Flakes là gì?](#nix-flakes-là-gì)
4. [Devenv là gì?](#devenv-là-gì)
5. [Cài đặt Nix](#cài-đặt-nix)
   - [Linux](#linux)
   - [macOS](#macos)
   - [Windows WSL](#windows-wsl)
6. [Cấu trúc files](#cấu-trúc-files)
   - [flake.nix](#flakenix)
   - [devenv.nix](#devenvnix)
   - [devenv.yaml](#devenvyaml)
   - [.env.secrets](#envsecrets)
   - [.envrc](#envrc)
7. [Quản lý Secrets](#quản-lý-secrets)
8. [Sử dụng Profiles](#sử-dụng-profiles)
9. [Commands hữu ích](#commands-hữu-ích)
10. [Workflow phát triển](#workflow-phát-triển)
11. [Troubleshooting](#troubleshooting)

---

## Giới thiệu

Tài liệu này hướng dẫn developer mới trong dự án TapHoaNho cách sử dụng Nix Flakes và Devenv để quản lý môi trường phát triển một cách nhất quán và hiệu quả.

**Mục tiêu:**
- Đảm bảo mọi developer có cùng môi trường phát triển
- Tự động hóa việc cài đặt và quản lý dependencies
- Cô lập môi trường để tránh xung đột giữa các dự án
- Quản lý secrets một cách an toàn

---

## Nix là gì?

Nix là một package manager và build tool cho Linux và các hệ thống khác. Nó cung cấp:

- **Reproducible builds**: Cùng source code luôn build ra cùng kết quả
- **Declarative package management**: Định nghĩa dependencies trong file cấu hình
- **Isolated environments**: Mỗi dự án có môi trường riêng biệt lập
- **Rollback capability**: Có thể rollback về phiên bản cũ bất cứ lúc nào

**Tại sao dùng Nix?**
- Không còn "works on my machine" - môi trường nhất quán cho mọi người
- Dependencies được quản lý tự động
- Dễ dàng chuyển đổi giữa các máy tính khác nhau
- Hỗ trợ đa ngôn ngữ và framework

---

## Nix Flakes là gì?

Nix Flakes là một tính năng mới của Nix giúp:

- **Reproducible configuration**: Mọi cấu hình được xác định rõ ràng trong file `flake.nix`
- **Composable**: Có thể chia sẻ và tái sử dụng flakes giữa các dự án
- **Version locking**: Lock file đảm bảo mọi người dùng cùng phiên bản packages
- **Git-friendly**: Flakes hoạt động tốt với Git và có thể commit vào repository

**Cấu trúc cơ bản:**
```
flake.nix          - Định nghĩa inputs và outputs
flake.lock         - Lock file (tự động sinh ra)
.devenv/           - Thư mục chứa các devenv shells (nếu dùng devenv)
```

---

## Devenv là gì?

Devenv là một công cụ xây dựng trên Nix Flakes giúp quản lý môi trường phát triển một cách dễ dàng hơn. Nó cung cấp:

- **Đơn giản hóa cấu hình**: Dễ dàng định nghĩa packages, scripts, và processes
- **Profile-based environments**: Có nhiều môi trường khác nhau (dev, test, prod)
- **Integration với Nix Flakes**: Tận dụng sức mạnh của Nix với cú pháp đơn giản
- **Shell hooks**: Tự động chạy script khi vào/ra môi trường

**Ưu điểm so với flake thuần túy:**
- Cú pháp đơn giản và dễ đọc hơn
- Có sẵn các tính năng phổ biến (dotenv, scripts, processes)
- Dễ dàng mở rộng với profiles

---

## Cài đặt Nix

### Linux

**Bước 1: Cài đặt Nix**

Sử dụng script cài đặt chính thức:

```bash
# Tải và chạy script cài đặt
curl -L https://nixos.org/nix/install | sh

# Script sẽ:
# - Cài đặt Nix vào /nix
# - Cấu hình PATH
# - Tạo user profile
```

**Bước 2: Kích hoạt Nix Flakes**

```bash
# Tạo thư mục cấu hình Nix (nếu chưa có)
mkdir -p ~/.config/nix

# Kích hoạt experimental features
echo "experimental-features = nix-command flakes" >> ~/.config/nix/nix.conf

# Reload shell hoặc mở terminal mới
source ~/.bashrc  # hoặc ~/.zshrc
```

**Bước 3: Xác nhận cài đặt**

```bash
# Kiểm tra Nix đã được cài đặt
nix --version

# Kiểm tra flakes đã được kích hoạt
nix flake --help
```

### macOS

**Bước 1: Cài đặt Nix**

```bash
# Tải và chạy script cài đặt
curl -L https://nixos.org/nix/install | sh

# Script sẽ cài đặt Nix vào /nix
```

**Bước 2: Kích hoạt Nix Flakes**

```bash
# Tạo thư mục cấu hình Nix
mkdir -p ~/.config/nix

# Kích hoạt experimental features
echo "experimental-features = nix-command flakes" >> ~/.config/nix/nix.conf

# Reload shell
source ~/.zshrc  # hoặc ~/.bashrc
```

**Bước 3: Xác nhận cài đặt**

```bash
nix --version
nix flake --help
```

### Windows WSL

**Bước 1: Cài đặt WSL2 (nếu chưa có)**

```bash
# Trong PowerShell (Run as Administrator)
wsl --install
```

**Bước 2: Cài đặt Nix trong WSL**

```bash
# Trong WSL terminal
curl -L https://nixos.org/nix/install | sh
```

**Bước 3: Kích hoạt Nix Flakes**

```bash
mkdir -p ~/.config/nix
echo "experimental-features = nix-command flakes" >> ~/.config/nix/nix.conf
source ~/.bashrc
```

**Bước 4: Xác nhận cài đặt**

```bash
nix --version
nix flake --help
```

---

## Cấu trúc Files

### flake.nix

File [`flake.nix`](../flake.nix:1) là file chính định nghĩa Nix Flake cho dự án.

**Cấu trúc cơ bản:**

```nix
{
  description = "TapHoaNho - Retail Store Management System";

  inputs = {
    nixpkgs.url = "github:cachix/devenv-nixpkgs/rolling";
    devenv.url = "github:cachix/devenv";
    flake-parts.url = "github:hercules-ci/flake-parts";
  };

  outputs = inputs@{ flake-parts, nixpkgs, ... }:
    flake-parts.lib.mkFlake { inherit inputs; } {
      imports = [ inputs.devenv.flakeModule ];
      
      systems = nixpkgs.lib.systems.flakeExposed;

      perSystem = { config, self', inputs', pkgs, system, ... }: {
        packages.default = pkgs.hello;
        formatter = pkgs.nixfmt-rfc-style;
        devenv.shells.default = import ./devenv.nix { inherit pkgs; };
      };
    };
}
```

**Giải thích:**

- **description**: Mô tả ngắn về dự án
- **inputs**: Định nghĩa các dependencies từ các nguồn khác (nixpkgs, devenv, flake-parts)
- **outputs**: Định nghĩa những gì flake này cung cấp (shells, packages, formatter)
- **devenv.shells.default**: Import shell từ file [`devenv.nix`](../devenv.nix:1)

### devenv.nix

File [`devenv.nix`](../devenv.nix:1) định nghĩa môi trường phát triển cụ thể cho dự án.

**Các phần chính:**

#### 1. Dotenv - Load Secrets

```nix
dotenv.enable = true;
dotenv.filename = ".env.secrets";
```

Điều này cho phép Devenv tự động load biến môi trường từ file [`.env.secrets`](../.env.secrets.example:1). File này chứa các secrets như database connection strings, API keys, v.v.

#### 2. Packages - Dependencies

```nix
packages = with pkgs; [
  # Core utilities
  git
  curl
  jq
  coreutils
  
  # Node.js & Yarn
  nodejs_20
  yarn-berry
  nodePackages.typescript
  nodePackages.eslint
  
  # .NET SDK 9
  dotnet-sdk_9
  
  # PostgreSQL client
  postgresql_16
  
  # C# tooling
  omnisharp-roslyn
];
```

**Giải thích:**
- **git, curl, jq, coreutils**: Các công cụ cơ bản cho development
- **nodejs_20**: Node.js version 20 (LTS)
- **yarn-berry**: Yarn 4.x (package manager mới hơn npm)
- **nodePackages.typescript**: TypeScript compiler
- **nodePackages.eslint**: ESLint cho linting JavaScript/TypeScript
- **dotnet-sdk_9**: .NET 9.0 SDK cho backend
- **postgresql_16**: PostgreSQL client tools
- **omnisharp-roslyn**: C# compiler tools

#### 3. Languages - Language Support

```nix
languages.javascript = {
  enable = true;
  package = pkgs.nodejs_20;
};

languages.typescript.enable = true;

languages.dotnet = {
  enable = true;
  package = pkgs.dotnet-sdk_9;
};
```

Điều này kích hoạt hỗ trợ cho JavaScript, TypeScript, và .NET trong shell.

#### 4. Processes - Background Services

```nix
processes = {
  frontend.exec = "${pkgs.bash}/bin/bash -c 'cd frontend && yarn dev'";
  backend.exec = "${pkgs.bash}/bin/bash -c 'cd RetailStoreManagement/src/WebApi && dotnet watch run --launch-profile http'";
};
```

**Processes** cho phép chạy các services trong background với lệnh `devenv up`.

#### 5. Scripts - Custom Commands

```nix
scripts = {
  setup.exec = ''
    echo "📦 Installing frontend dependencies..."
    (cd frontend && yarn install)
    echo ""
    echo "📦 Restoring .NET packages..."
    (cd RetailStoreManagement && dotnet restore)
    echo ""
    echo "✅ Setup complete!"
  '';
  
  build-all.exec = ''
    echo "🔨 Building frontend and backend..."
    (cd frontend && yarn build) &
    (cd RetailStoreManagement && dotnet build) &
    wait
    echo "✅ Build complete!"
  '';
  
  db-check.exec = ''
    echo "🔍 Checking Neon database connectivity..."
    if pg_isready -h ep-lucky-queen-a1u66w5c-pooler.ap-southeast-1.aws.neon.tech -d store_management -U neondb_owner -q 2>/dev/null; then
      echo "✅ Database connection: SUCCESS"
    else
      echo "⚠️  Database connection: FAILED (check network/VPN)"
    fi
  '';
  
  lint-frontend.exec = ''
    cd frontend && yarn lint
  '';
};
```

**Scripts** có thể chạy với lệnh `devenv <script-name>`.

#### 6. Git Hooks - Pre-commit Checks

```nix
git-hooks.hooks = {
  pre-commit = {
    enable = true;
    name = "Lint check";
    entry = "${pkgs.bash}/bin/bash -c 'export PATH=${pkgs.nodejs_20}/bin:${pkgs.yarn-berry}/bin:$PATH && cd frontend && yarn lint'";
    pass_filenames = false;
  };
};
```

Git hooks tự động chạy trước khi commit để đảm bảo code quality.

#### 7. Environment Variables

```nix
env = {
  COREPACK_ENABLE_STRICT = "0";
};
```

Biến môi trường không phải secrets được định nghĩa ở đây.

#### 8. Profiles - Environment-specific Configurations

```nix
profiles = {
  # Development profile (default)
  dev.module = {
    env = {
      NODE_ENV = "development";
      ASPNETCORE_ENVIRONMENT = "Development";
      VITE_API_URL = "http://localhost:5175";
    };
  };

  # Testing profile
  test.module = { pkgs, ... }: {
    packages = [ 
      pkgs.playwright-driver.browsers
    ];
    env = {
      NODE_ENV = "test";
      ASPNETCORE_ENVIRONMENT = "Testing";
      VITE_API_URL = "http://localhost:5175";
    };
    scripts.test-frontend.exec = ''
      echo "🧪 Running frontend tests..."
      cd frontend && yarn test
    '';
    scripts.test-backend.exec = ''
      echo "🧪 Running backend tests..."
      cd RetailStoreManagement && dotnet test
    '';
    scripts.test-all.exec = ''
      echo "🧪 Running all tests..."
      (cd frontend && yarn test) &
      (cd RetailStoreManagement && dotnet test) &
      wait
      echo "✅ All tests complete!"
    '';
  };

  # Production profile
  prod.module = {
    env = {
      NODE_ENV = "production";
      ASPNETCORE_ENVIRONMENT = "Production";
    };
    scripts.build-prod.exec = ''
      echo "🚀 Building for production..."
      (cd frontend && yarn build) &
      (cd RetailStoreManagement && dotnet publish -c Release) &
      wait
      echo "✅ Production build complete!"
    '';
  };

  # Fullstack profile (combines dev + test)
  fullstack.extends = [ "dev" "test" ];
};
```

**Profiles** cho phép có nhiều môi trường khác nhau.

### devenv.yaml

File [`devenv.yaml`](../devenv.yaml:1) định nghĩa inputs cho Devenv.

```yaml
inputs:
  nixpkgs:
    url: github:cachix/devenv-nixpkgs/rolling
```

File này thường ngắn và chỉ định nghĩa các inputs.

### .env.secrets

File [`.env.secrets`](../.env.secrets.example:1) chứa các biến môi trường nhạy cảm (secrets).

**Ví dụ từ [`.env.secrets.example`](../.env.secrets.example:1):**

```bash
# ============================================
# TapHoaNho - Environment Secrets Template
# ============================================
# Copy file này thành .env.secrets và điền giá trị:
#   cp .env.secrets.example .env.secrets
# ============================================

# .NET Backend Environment Variables
# ============================================
ASPNETCORE_ENVIRONMENT=Development

# Logging
Logging__LogLevel__Default=Information
Logging__LogLevel__Microsoft.AspNetCore=Warning
Logging__LogLevel__Microsoft.EntityFrameworkCore=Information

# Database (Neon PostgreSQL)
ConnectionStrings__DefaultConnection=

# JWT Settings
JwtSettings__SecretKey=
JwtSettings__Issuer=store-management
JwtSettings__Audience=store-management
JwtSettings__ExpiryMinutes=60
JwtSettings__RefreshExpiryDays=7

# ImageKit
ImageKit__PublicKey=
ImageKit__PrivateKey=
ImageKit__UrlEndpoint=

# ============================================
# React/Vite Frontend Environment Variables
# ============================================
VITE_API_BASE_URL=http://localhost:5175
VITE_APP_NAME=Tap Hoa Nho
VITE_APP_VERSION=1.0.0
VITE_IMAGEKIT_PUBLIC_KEY=
VITE_IMAGEKIT_URL_ENDPOINT=
```

**Quan trọng:**
- File `.env.secrets` đã được thêm vào [`.gitignore`](../.gitignore:1) - KHÔNG commit secrets!
- Chỉ commit file `.env.secrets.example` làm template
- Mỗi developer cần copy file này và điền giá trị riêng

### .envrc

File [`.envrc`](../.envrc:1) là file cấu hình cho Direnv (công cụ tự động load môi trường).

```bash
# Direnv configuration để tự động load nix shell với devenv
# Cài đặt direnv: nix profile install nixpkgs#direnv
# Sau đó chạy: direnv allow

# Thêm các đường dẫn nix phổ biến vào PATH trước
[ -d /nix/var/nix/profiles/default/bin ] && export PATH="/nix/var/nix/profiles/default/bin:$PATH"
[ -d ~/.nix-profile/bin ] && export PATH="$HOME/.nix-profile/bin:$PATH"

# Source nix profile từ các vị trí phổ biến
if [ -f /etc/profile.d/nix.sh ]; then
  source /etc/profile.d/nix.sh
elif [ -f ~/.nix-profile/etc/profile.d/nix.sh ]; then
  source ~/.nix-profile/etc/profile.d/nix.sh
elif [ -f /nix/var/nix/profiles/default/etc/profile.d/nix.sh ]; then
  source /nix/var/nix/profiles/default/etc/profile.d/nix.sh
fi

# Load flake development shell với --no-pure-eval (required for devenv)
use flake . --no-pure-eval
```

**Công dụng:**
- Tự động load môi trường Nix khi `cd` vào thư mục dự án
- Đảm bảo Nix có sẵn trong PATH trước khi load flake
- Tự động unload khi rời khỏi thư mục

---

## Quản lý Secrets

### Setup Secrets

**Bước 1: Copy template**

```bash
# Vào thư mục dự án
cd shiny-carnival

# Copy template thành file secrets
cp .env.secrets.example .env.secrets
```

**Bước 2: Điền giá trị**

Mở file `.env.secrets` và điền các giá trị:

```bash
# Database connection string
ConnectionStrings__DefaultConnection="Host=ep-lucky-queen-a1u66w5c-pooler.ap-southeast-1.aws.neon.tech;Database=store_management;Username=neondb_owner;Password=your-password;SSL Mode=Require;"

# JWT secret key
JwtSettings__SecretKey="your-super-secret-key-at-least-32-characters-long"

# ImageKit credentials
ImageKit__PublicKey="your-public-key"
ImageKit__PrivateKey="your-private-key"
ImageKit__UrlEndpoint="https://ik.imagekit.io/your-endpoint"
```

**Bước 3: Load secrets**

Khi bạn vào môi trường Devenv, secrets sẽ tự động được load:

```bash
# Vào môi trường
devenv shell

# Kiểm tra secrets đã được load
echo $ConnectionStrings__DefaultConnection
```

### Best Practices cho Secrets

1. **KHÔNG bao giờ commit `.env.secrets`**
   - File này đã được thêm vào `.gitignore`
   - Chỉ commit `.env.secrets.example`

2. **Sử dụng environment variables thay vì hardcode**
   - Đừng hardcode secrets trong code
   - Luôn dùng biến môi trường

3. **Rotate secrets định kỳ**
   - Đổi API keys, passwords định kỳ
   - Cập nhật `.env.secrets` tương ứng

4. **Sử dụng secrets khác nhau cho mỗi môi trường**
   - Dev, test, prod nên có secrets riêng
   - Có thể dùng profiles để quản lý

---

## Sử dụng Profiles

### Profile mặc định (dev)

Profile mặc định là `dev`, không cần chỉ định:

```bash
# Vào môi trường dev (mặc định)
devenv shell

# Hoặc dùng direnv (tự động)
cd shiny-carnival
# Môi trường dev tự động được load
```

### Profile test

```bash
# Vào môi trường test
devenv shell --profile test

# Chạy tests
devenv test-frontend
devenv test-backend
devenv test-all
```

Profile test có thêm:
- Playwright cho E2E testing
- Các biến môi trường `NODE_ENV=test`, `ASPNETCORE_ENVIRONMENT=Testing`

### Profile prod

```bash
# Vào môi trường prod
devenv shell --profile prod

# Build cho production
devenv build-prod
```

Profile prod có:
- Biến môi trường `NODE_ENV=production`, `ASPNETCORE_ENVIRONMENT=Production`
- Script build-prod để build cho production

### Profile fullstack

```bash
# Kết hợp dev + test
devenv shell --profile fullstack
```

### Tạo profile mới

Bạn có thể thêm profile mới vào file [`devenv.nix`](../devenv.nix:167):

```nix
profiles = {
  # Profile staging
  staging.module = {
    env = {
      NODE_ENV = "staging";
      ASPNETCORE_ENVIRONMENT = "Staging";
      VITE_API_URL = "https://staging-api.taphoanho.com";
    };
    packages = [
      pkgs.playwright-driver.browsers
    ];
    scripts.deploy-staging.exec = ''
      echo "🚀 Deploying to staging..."
      # Your deployment commands here
    '';
  };
};
```

---

## Commands hữu ích

### Vào môi trường

**Cách 1: Sử dụng devenv shell**

```bash
# Vào shell (profile mặc định: dev)
devenv shell

# Vào shell với profile cụ thể
devenv shell --profile test
devenv shell --profile prod
```

**Cách 2: Sử dụng Direnv (tự động)**

```bash
# Cài đặt direnv (nếu chưa có)
nix profile install nixpkgs#direnv

# Cấu hình shell (chỉ cần làm 1 lần)
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc
source ~/.bashrc

# Cho phép direnv trong thư mục dự án
cd shiny-carnival
direnv allow

# Từ nay, mỗi khi cd vào thư mục, môi trường tự động được load
cd ..
cd shiny-carnival  # Môi trường tự động load
```

### Chạy services

```bash
# Vào môi trường trước
devenv shell

# Chạy frontend và backend
devenv up

# Chạy riêng lẻ
devenv processes.frontend
devenv processes.backend
```

### Scripts

```bash
# Setup dependencies
devenv setup

# Build all
devenv build-all

# Build frontend
devenv build-frontend

# Build backend
devenv build-backend

# Check database connection
devenv db-check

# Lint frontend
devenv lint-frontend
```

### Tests

```bash
# Chạy tất cả tests
devenv test-all

# Chạy frontend tests
devenv test-frontend

# Chạy backend tests
devenv test-backend
```

### Nix commands

```bash
# Update dependencies (cập nhật flake.lock)
nix flake update

# Check flake
nix flake check

# Format Nix files
nix fmt

# Show packages
nix flake show
```

---

## Workflow phát triển

### Bắt đầu dự án

**Bước 1: Clone repository**

```bash
git clone <repository-url>
cd TapHoaNho/shiny-carnival
```

**Bước 2: Setup secrets**

```bash
# Copy template
cp .env.secrets.example .env.secrets

# Điền các giá trị secrets
nano .env.secrets  # hoặc dùng editor khác
```

**Bước 3: Cài đặt direnv (khuyến nghị)**

```bash
# Cài đặt direnv
nix profile install nixpkgs#direnv

# Cấu hình shell
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc
source ~/.bashrc

# Cho phép direnv
direnv allow
```

**Bước 4: Setup dependencies**

```bash
# Môi trường tự động được load (nếu dùng direnv)
# Hoặc vào thủ công
devenv shell

# Install dependencies
devenv setup
```

### Workflow hàng ngày

**1. Bắt đầu làm việc**

```bash
# Vào thư mục dự án
cd shiny-carnival

# Môi trường tự động load (nếu dùng direnv)
# Hoặc vào thủ công
devenv shell
```

**2. Chạy services**

```bash
# Chạy frontend và backend
devenv up

# Mở terminal khác để xem logs
# Frontend: http://localhost:5173
# Backend: http://localhost:5175
```

**3. Thực hiện thay đổi**

- Thay đổi code
- Lưu file
- Services tự động reload (hot reload)

**4. Kiểm tra**

```bash
# Lint frontend
devenv lint-frontend

# Chạy tests
devenv test-all
```

**5. Commit code**

```bash
# Git hooks tự động chạy lint trước khi commit
git add .
git commit -m "feat: add new feature"
```

### Workflow cho PR

**1. Tạo branch mới**

```bash
git checkout -b feature/new-feature
```

**2. Thực hiện thay đổi**

```bash
# Vào môi trường
devenv shell

# Chạy services
devenv up

# Thực hiện thay đổi code
```

**3. Test**

```bash
# Chạy tests
devenv test-all

# Lint
devenv lint-frontend
```

**4. Commit và push**

```bash
git add .
git commit -m "feat: add new feature"
git push origin feature/new-feature
```

**5. Tạo PR**

- Tạo Pull Request trên GitHub/GitLab
- CI/CD sẽ tự động chạy tests

### Workflow cho release

**1. Switch sang profile prod**

```bash
devenv shell --profile prod
```

**2. Build cho production**

```bash
devenv build-prod
```

**3. Deploy**

```bash
# Deploy frontend và backend
# (tùy theo deployment pipeline của dự án)
```

---

## Troubleshooting

### Lỗi: "command not found: nix"

**Nguyên nhân:** Nix chưa được cài đặt hoặc chưa có trong PATH.

**Giải pháp:**

```bash
# Kiểm tra Nix đã được cài đặt
which nix

# Nếu không tìm thấy, cài đặt lại
curl -L https://nixos.org/nix/install | sh

# Reload shell
source ~/.bashrc
```

### Lỗi: "command not found: dotnet"

**Nguyên nhân:** Chưa vào môi trường Devenv.

**Giải pháp:**

```bash
# Vào môi trường
devenv shell

# Hoặc dùng direnv
cd shiny-carnival
# Môi trường tự động load
```

### Lỗi: "node_modules not found"

**Nguyên nhân:** Chưa install dependencies.

**Giải pháp:**

```bash
# Vào môi trường
devenv shell

# Install dependencies
cd frontend && yarn install
```

Hoặc dùng script:

```bash
devenv setup
```

### Lỗi: "PostgreSQL connection failed"

**Nguyên nhân:** Database chưa chạy hoặc connection string sai.

**Giải pháp:**

```bash
# Kiểm tra connection string
cat .env.secrets | grep ConnectionStrings__DefaultConnection

# Kiểm tra database connectivity
devenv db-check

# Nếu dùng Neon, kiểm tra VPN/network
# Neon có thể bị chặn ở một số mạng
```

### Lỗi: "direnv: error .envrc is blocked"

**Nguyên nhân:** File `.envrc` đã thay đổi và cần được phê duyệt lại.

**Giải pháp:**

```bash
# Xem thay đổi
direnv diff

# Phê duyệt thay đổi
direnv allow
```

### Lỗi: "direnv: not found"

**Nguyên nhân:** Direnv chưa được cài đặt.

**Giải pháp:**

```bash
# Cài đặt direnv
nix profile install nixpkgs#direnv

# Cấu hình shell
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc
source ~/.bashrc
```

### Lỗi: "Corepack is about to download yarn-1.22.22.tgz"

**Nguyên nhân:** Corepack đang cố download yarn 1.x thay vì dùng yarn 4.x từ Nix.

**Giải pháp:**

```bash
# Đảm bảo đã vào môi trường
devenv shell

# Kiểm tra yarn version
yarn --version
# Phải là 4.x (ví dụ: 4.10.3)

# Nếu vẫn sai, kiểm tra PATH
which yarn
# Phải là /nix/store/.../yarn-berry-4.12.0/bin/yarn
```

### Lỗi: Yarn vẫn load từ NVM thay vì Nix

**Nguyên nhân:** PATH từ NVM đang được ưu tiên hơn PATH từ Nix.

**Giải pháp:**

```bash
# Kiểm tra yarn đến từ đâu
which yarn

# Nếu thấy ~/.nvm/..., tạm thời disable NVM
unset NVM_DIR

# Reload môi trường
devenv shell

# Kiểm tra lại
which yarn
# Phải là /nix/store/...
```

### Lỗi: "nix: command not found" khi direnv load

**Nguyên nhân:** File `.envrc` chưa được cấu hình đúng để load Nix trước.

**Giải pháp:**

```bash
# Kiểm tra file .envrc
cat .envrc

# Phải có các dòng thêm PATH và source nix profile
# Xem phần ".envrc" ở trên

# Nếu thiếu, cập nhật lại file .envrc
# Sau đó chạy
direnv allow
```

### Lỗi: ".envrc is blocked" sau khi chỉnh sửa

**Nguyên nhân:** Direnv block file sau khi chỉnh sửa để đảm bảo an toàn.

**Giải pháp:**

```bash
# Xem nội dung thay đổi
direnv diff

# Phê duyệt thay đổi
direnv allow
```

### Lỗi: Tests failed

**Nguyên nhân:** Code có bug hoặc tests sai.

**Giải pháp:**

```bash
# Chạy tests với verbose output
devenv test-frontend --verbose
devenv test-backend --verbose

# Xem logs để hiểu tại sao tests failed

# Sửa code hoặc tests
```

### Lỗi: Build failed

**Nguyên nhân:** Code có lỗi compile hoặc dependencies thiếu.

**Giải pháp:**

```bash
# Xem error message
devenv build-frontend
devenv build-backend

# Nếu thiếu dependencies
devenv setup

# Nếu có lỗi code
# Sửa lỗi compile
```

### Lỗi: Flake lock file desynchronized

**Nguyên nhân:** `flake.lock` không đồng bộ với `flake.nix`.

**Giải pháp:**

```bash
# Update lock file
nix flake update
```

### Lỗi: Cannot access secrets

**Nguyên nhân:** File `.env.secrets` không tồn tại hoặc sai format.

**Giải pháp:**

```bash
# Kiểm tra file tồn tại
ls -la .env.secrets

# Nếu không có, copy template
cp .env.secrets.example .env.secrets

# Điền lại các giá trị
nano .env.secrets

# Reload môi trường
devenv shell
```

### Lỗi: Git hooks không chạy

**Nguyên nhân:** Git hooks chưa được cài đặt.

**Giải pháp:**

```bash
# Cài đặt git hooks
devenv git install

# Kiểm tra hooks đã được cài đặt
ls -la .git/hooks/pre-commit
```

### Lỗi: Profile không hoạt động

**Nguyên nhân:** Profile không được định nghĩa đúng trong [`devenv.nix`](../devenv.nix:167).

**Giải pháp:**

```bash
# Kiểm tra profile tồn tại
devenv shell --profile test

# Nếu lỗi, kiểm tra cấu hình profile trong devenv.nix
# Xem phần "Profiles" ở trên
```

---

## Tài liệu tham khảo

- [Nix Manual](https://nixos.org/manual/nix/stable/)
- [Nix Flakes](https://nixos.wiki/wiki/Flakes)
- [Devenv Documentation](https://devenv.sh/)
- [Direnv Documentation](https://direnv.net/)
- [.NET 9.0 Documentation](https://learn.microsoft.com/dotnet/)
- [Node.js 20 Documentation](https://nodejs.org/docs/latest-v20.x/)
- [Yarn Documentation](https://yarnpkg.com/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

---

## Hỗ trợ

Nếu bạn gặp vấn đề không được giải quyết trong phần Troubleshooting:

1. Kiểm tra logs chi tiết để hiểu lỗi
2. Tìm kiếm trong tài liệu tham khảo
3. Hỏi team hoặc mở issue trên repository

**Happy coding! 🚀**
