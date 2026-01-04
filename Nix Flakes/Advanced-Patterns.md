# Advanced Patterns cho Nix Flakes

## Mục đích

Tài liệu này cung cấp các patterns nâng cao cho Nix Flakes, dành cho lập trình viên Senior trở lên. Các patterns này giúp:
- Tối ưu hóa cấu trúc flake
- Tái sử dụng code giữa các flakes
- Quản lý dependencies phức tạp
- Tích hợp Nix vào CI/CD pipeline

---

## 1. Flake Outputs Nâng cao

### 1.1. Multiple DevShells với Cấu hình Khác nhau

Có thể định nghĩa nhiều devShells với cấu hình khác nhau:

```nix
outputs = { self, nixpkgs, flake-utils }:
{
  devShells.default = pkgs.mkShell {
    buildInputs = [ pkgs.nodejs_20 pkgs.yarn-berry pkgs.dotnet-sdk_9 ];
    shellHook = ''
      echo "Full development environment"
    '';
  };

  devShells.frontend-only = pkgs.mkShell {
    buildInputs = [ pkgs.nodejs_20 pkgs.yarn-berry ];
    shellHook = ''
      echo "Frontend development environment"
    '';
  };

  devShells.backend-only = pkgs.mkShell {
    buildInputs = [ pkgs.dotnet-sdk_9 pkgs.postgresql_16 ];
    shellHook = ''
      echo "Backend development environment"
    '';
  };
}
```

**Sử dụng:**
```bash
# Load shell mặc định
nix develop

# Load shell cụ thể
nix develop .#frontend-only
nix develop .#backend-only
```

### 1.2. Packages Output với Custom Derivations

Tạo packages tùy chỉnh với custom derivations:

```nix
outputs = { self, nixpkgs, ... }:
let
  pkgs = nixpkgs.legacyPackages.${system};
in
{
  packages.my-custom-tool = pkgs.stdenv.mkDerivation {
    name = "my-custom-tool";
    buildInputs = [ pkgs.nodejs_20 pkgs.yarn-berry ];
    src = ./.;
    buildPhase = ''
      cd frontend
      yarn install
      yarn build
    '';
    installPhase = ''
      mkdir -p $out/bin
      cp -r dist $out/
    '';
  };

  packages.backend-builder = pkgs.stdenv.mkDerivation {
    name = "backend-builder";
    buildInputs = [ pkgs.dotnet-sdk_9 ];
    src = ./backend;
    buildPhase = ''
      dotnet restore
      dotnet build -c Release
    '';
    installPhase = ''
      mkdir -p $out/bin
      cp -r bin/Release/net9.0/* $out/bin/
    '';
  };
}
```

**Sử dụng:**
```bash
# Cài đặt package
nix profile install .#my-custom-tool

# Hoặc chạy trực tiếp
nix run .#my-custom-tool

# Build backend
nix build .#backend-builder
```

### 1.3. Apps Output cho Ứng dụng Độc lập

Tạo outputs cho ứng dụng độc lập:

```nix
outputs = { self, nixpkgs, ... }:
{
  apps.my-app = {
    type = "app";
    program = pkgs.writeShellScriptBin {
      name = "my-app";
      runtimeInputs = [ pkgs.nodejs_20 ];
      text = ''
        #!/bin/sh
        cd frontend
        yarn dev
      '';
    };
  };

  apps.backend-server = {
    type = "app";
    program = pkgs.writeShellScriptBin {
      name = "backend-server";
      runtimeInputs = [ pkgs.dotnet-sdk_9 pkgs.postgresql_16 ];
      text = ''
        #!/bin/sh
        cd backend
        dotnet run
      '';
    };
  };
}
```

**Sử dụng:**
```bash
# Chạy app
nix run .#my-app
nix run .#backend-server
```

### 1.4. Hydra Outputs - Tối ưu Build

Hydra cho phép build nhiều configurations cùng lúc:

```nix
outputs = { self, nixpkgs, ... }:
{
  hydraJobs = {
    frontend-build = {
      inputs = {
        nodejs = pkgs.nodejs_20;
        yarn = pkgs.yarn-berry;
      };
      shell = ''
        cd frontend
        yarn install
        yarn build
      '';
    };

    backend-build = {
      inputs = {
        dotnet = pkgs.dotnet-sdk_9;
      };
      shell = ''
        cd backend
        dotnet restore
        dotnet build
      '';
    };
  };
}
```

**Sử dụng:**
```bash
# Build tất cả jobs
nix build .#hydraJobs

# Build job cụ thể
nix build .#hydraJobs.frontend-build
nix build .#hydraJobs.backend-build
```

---

## 2. Flake Inputs và Dependencies Nâng cao

### 2.1. Overlays cho Nixpkgs

Sử dụng overlays để tùy chỉnh nixpkgs:

```nix
outputs = { self, nixpkgs, ... }:
{
  packages = {
    default = pkgs.mkShell {
      buildInputs = [ pkgs.hello ];
    };
  };

  # Overlay tùy chỉnh
  overlays = {
    # Thêm package mới
    my-packages = final: prev: {
      my-custom-package = pkgs.callPackage ./my-package.nix;
    };
  };
}
```

### 2.2. Custom Flake Inputs

Tạo custom flake inputs:

```nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    flake-utils.url = "github:numtide/flake-utils";

    # Custom input từ GitHub
    my-dependency.url = "github:username/repo";

    # Custom input từ local path
    local-dependency.url = "../other-flake";

    # Custom input với follow
    nixpkgs.follows = "/nixpkgs";
  };
}
```

### 2.3. Input theo dõi (Follows)

Theo dõi một input để nhận updates:

```nix
inputs = {
  nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
  nixpkgs.follows = "/nixpkgs";
}
```

Khi `/nixpkgs` được update, flake của bạn cũng sẽ nhận updates.

### 2.4. Input với điều kiện (If)

Sử dụng điều kiện để chỉ load input khi cần:

```nix
inputs = {
  nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
  my-dependency.url = "github:username/repo";
  my-dependency.flake = false;  # Chỉ load khi cần
};
```

```nix
outputs = { self, nixpkgs, ... }:
let
  pkgs = import nixpkgs {
    inherit system;
  };
in
{
  packages.default = pkgs.mkShell {
    buildInputs = [ pkgs.hello ] ++ (pkgs.lib.optionals self.my-dependency.flake [ self.my-dependency ]);
  };
}
```

---

## 3. Nix Modules và Overlays

### 3.1. Tạo Nix Module

Nix module là file có thể tái sử dụng giữa các flakes:

```nix
# File: my-module/flake.nix
{
  description = "My Nix module";

  outputs = { self, nixpkgs }:
  let
    pkgs = nixpkgs.legacyPackages.${system};
  in
  {
    packages.my-function = pkgs.writeShellScriptBin {
      name = "my-function";
      text = ''
        #!/bin/sh
        echo "Hello from my-module!"
      '';
    };
  };
}
```

### 3.2. Sử dụng Module từ Flake khác

```nix
# File: project/flake.nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    my-module.url = "path:../my-module";
  };

  outputs = { self, nixpkgs, my-module }:
  let
    pkgs = nixpkgs.legacyPackages.${system};
  in
  {
    devShells.default = pkgs.mkShell {
      buildInputs = [ my-module.packages.my-function ];
    };
  };
}
```

### 3.3. Overlays để Tùy chỉnh Nixpkgs

```nix
outputs = { self, nixpkgs, ... }:
{
  packages = {
    default = pkgs.mkShell {
      buildInputs = [ pkgs.hello ];
    };
  };

  overlays = {
    # Override package
    my-hello = final: prev: {
      hello = prev.hello.overrideAttrs (old: {
        postInstall = ''
          echo "Custom post-install"
        '';
      });
    };
  };
}
```

---

## 4. Multi-System Support

### 4.1. Hỗ trợ Multiple Systems với Flake Utils

```nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    flake-utils.url = "github:numtide/flake-utils";
  };

  outputs = { self, nixpkgs, flake-utils }:
    flake-utils.lib.eachDefaultSystem (system:
      let
        pkgs = import nixpkgs {
          inherit system;
        };
      in
      {
        devShells.default = pkgs.mkShell {
          buildInputs = [ pkgs.hello ];
        };
      }
    );
}
```

### 4.2. System-Specific Configuration

```nix
outputs = { self, nixpkgs, ... }:
let
  pkgs = nixpkgs.legacyPackages.${system};
in
{
  devShells.default = pkgs.mkShell {
    buildInputs = [ pkgs.hello ] ++ pkgs.lib.optionals (system == "x86_64-linux") [
      pkgs.gcc
    ];
  };
}
```

---

## 5. Custom Nix Packages

### 5.1. Tạo Package Tùy chỉnh

```nix
# File: packages/my-custom-package.nix
{ pkgs, stdenv, lib, fetchurl }:

stdenv.mkDerivation {
  name = "my-custom-package";
  version = "1.0.0";

  src = fetchurl {
    url = "https://example.com/my-package-1.0.0.tar.gz";
    sha256 = "sha256-...";
  };

  buildInputs = [ pkgs.nodejs ];

  buildPhase = ''
    echo "Building my-custom-package..."
  '';

  installPhase = ''
    mkdir -p $out/bin
    cp my-script $out/bin/
    chmod +x $out/bin/my-script
  '';
}
```

### 5.2. Sử dụng Custom Package trong Flake

```nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
  };

  outputs = { self, nixpkgs }:
  let
    pkgs = nixpkgs.legacyPackages.${system};
  in
  {
    packages.my-custom = pkgs.callPackage ./packages/my-custom-package.nix;
  };
}
```

---

## 6. Shell Hooks Nâng cao

### 6.1. Pre-Hook

Chạy trước khi vào shell:

```nix
devShells.default = pkgs.mkShell {
  shellHook = ''
    # Kiểm tra dependencies
    echo "Checking dependencies..."

    # Chạy các lệnh setup
    if [ -f .pre-nix-shell ]; then
      ./.pre-nix-shell
    fi
  '';
}
```

### 6.2. Post-Hook

Chạy sau khi vào shell:

```nix
devShells.default = pkgs.mkShell {
  shellHook = ''
    echo "Development environment ready!"

    # Chạy các lệnh cleanup
    if [ -f .post-nix-shell ]; then
      ./.post-nix-shell
    fi
  '';
}
```

### 6.3. Interactive Hooks

Sử dụng interactive hooks để hỏi người dùng:

```nix
devShells.default = pkgs.mkShell {
  shellHook = ''
    # Kiểm tra nếu cần setup database
    if [ ! -f .env ]; then
      echo "Database URL not set. Would you like to set it now? (y/n)"
      read -r answer
      if [ "$answer" = "y" ]; then
        echo "Enter database URL:"
        read -r db_url
        echo "DATABASE_URL=$db_url" > .env
      fi
    fi
  '';
}
```

---

## 7. Environment Variables Management

### 7.1. Environment Variables từ File

```nix
devShells.default = pkgs.mkShell {
  shellHook = ''
    # Load environment variables từ file
    if [ -f .env ]; then
      set -a
      . .env
    fi
  '';
}
```

### 7.2. Environment Variables với Default Values

```nix
devShells.default = pkgs.mkShell {
  shellHook = ''
    # Set default values
    export DATABASE_URL="''${DATABASE_URL:-postgresql://localhost:5432/mydb}"

    # Override từ file .env nếu có
    if [ -f .env ]; then
      set -a
      . .env
    fi
  '';
}
```

### 7.3. Environment Variables cho CI/CD

```nix
devShells.ci = pkgs.mkShell {
  shellHook = ''
    # CI-specific variables
    export CI=true
    export NODE_ENV=production

    # Load secrets từ environment
    export DATABASE_URL="''${DATABASE_URL}"
  '';
}
```

---

## 8. Flake Composition Patterns

### 8.1. Flake Composition

Kết hợp nhiều flakes thành một flake lớn:

```nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    flake-utils.url = "github:numtide/flake-utils";

    # Các flakes con
    frontend.url = "github:username/frontend-flake";
    backend.url = "github:username/backend-flake";
  };

  outputs = { self, nixpkgs, flake-utils, frontend, backend }:
  let
    pkgs = nixpkgs.legacyPackages.${system};
  in
  {
    devShells.default = pkgs.mkShell {
      buildInputs = [
        frontend.packages.default
        backend.packages.default
      ];
    };
  };
}
```

### 8.2. Flake Inheritance

```nix
# Base flake
{
  description = "Base flake";

  outputs = { self, nixpkgs }:
  let
    pkgs = nixpkgs.legacyPackages.${system};
  in
  {
    packages.base-tools = [ pkgs.git pkgs.curl ];
  };
}
```

```nix
# Project flake kế thừa từ base flake
{
  description = "Project flake";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    base.url = "github:username/base-flake";
  };

  outputs = { self, nixpkgs, base }:
  let
    pkgs = nixpkgs.legacyPackages.${system};
  in
  {
    devShells.default = pkgs.mkShell {
      buildInputs = base.packages.base-tools ++ [ pkgs.nodejs_20 ];
    };
  };
}
```

---

## 9. Testing và Validation

### 9.1. Flake Check

Kiểm tra tính hợp lệ của flake:

```bash
nix flake check
```

### 9.2. Flake Show

Hiển thị thông tin về flake:

```bash
nix flake show
```

### 9.3. Flake Metadata

Thêm metadata cho flake:

```nix
{
  description = "Môi trường phát triển cho Retail Store Management System";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    flake-utils.url = "github:numtide/flake-utils";
  };

  outputs = { self, nixpkgs, flake-utils }:
    let
      pkgs = nixpkgs.legacyPackages.${system};
    in
    {
        # Metadata
        devShells.default = pkgs.mkShell {
          meta = {
            description = "Development shell cho dự án TapHoaNho";
            maintainers = [ "username" ];
            license = "MIT";
          };
          buildInputs = [ pkgs.hello ];
        };
    };
}
```

---

## 10. Performance Optimization

### 10.1. Binary Cache

Sử dụng binary cache để tăng tốc độ build:

```bash
# Thêm binary cache
nix-config --substituters https://cache.nixos.org https://cache.nixos.org

# Hoặc sử dụng cachix
cachix use https://taphoanho-nix.cachix.org
```

### 10.2. Store Optimization

```bash
# Xóa các packages không sử dụng
nix-collect-garbage -d

# Xóa các old generations
nix-collect-garbage --delete-old

# Tối ưu hóa store
nix-store --optimise
```

### 10.3. Build Caching

```nix
# Sử dụng build cache
nix build --keep-going

# Xóa build cache
nix-store --delete-old
```

---

## 11. Troubleshooting Nâng cao

### 11.1. Debug Flake Evaluation

```bash
# Xem cách Nix đánh giá flake
nix eval --impure --json .#devShells.default

# Xem derivation chi tiết
nix show-derivation
```

### 11.2. Store Queries Nâng cao

```bash
# Tìm packages không được sử dụng
nix-store --query --requisites --referrers

# Xem dependencies của một package
nix-store --query --tree <package-name>

# Tìm các packages có cùng tên
nix-store --query --references <package-name>
```

### 11.3. Flake Lock Issues

```bash
# Xóa lock file và tạo lại
rm flake.lock
nix flake update

# Pin đến commit cụ thể
nix flake lock --update-input nixpkgs --ref nixos-23.11
```

### 11.4. Build Failures

```bash
# Xem log build
nix log -f

# Xem derivation đang chạy
nix show-derivation

# Build với verbose output
nix build -v
```

---

## 12. Best Practices cho Senior

### 12.1. Flake Structure

- Sử dụng `flake-utils.lib.eachDefaultSystem` cho multi-system support
- Tách logic thành các functions tái sử dụng
- Sử dụng overlays để tùy chỉnh nixpkgs
- Document rõ ràng các custom packages

### 12.2. Dependency Management

- Pin inputs đến version cụ thể cho production
- Sử dụng `flake.lock` cho reproducibility
- Theo dõi inputs với `follows`
- Review dependencies thường xuyên

### 12.3. Performance

- Sử dụng binary cache (cachix)
- Tối ưu hóa Nix store thường xuyên
- Sử dụng `nix build` thay vì `nix develop` khi có thể
- Tránh build lại packages đã có

### 12.4. Security

- Không đặt secrets trong `flake.nix`
- Sử dụng file `.env` riêng cho secrets
- Review dependencies thường xuyên
- Cập nhật packages thường xuyên

### 12.5. Documentation

- Document rõ ràng mục đích của flake
- Thêm ví dụ cho từng output
- Giải thích các quyết định thiết kế
- Cập nhật tài liệu khi thay đổi

---

## Tham khảo

- [Nix Flakes Documentation](https://nixos.wiki/wiki/Flakes)
- [Flake Utils](https://github.com/numtide/flake-utils)
- [Nix Pills](https://nixos.org/guides/nix-pills/)
- [Cachix](https://www.cachix.org/)

## File liên quan trong dự án

- [`shiny-carnival/flake.nix`](../shiny-carnival/flake.nix:1-88) - Cấu hình chính
- [`shiny-carnival/.envrc`](../shiny-carnival/.envrc:1-23) - Direnv config
