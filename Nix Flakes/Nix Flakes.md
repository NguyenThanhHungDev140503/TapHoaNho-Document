# Nix Flakes - Hướng dẫn Toàn diện

## Mục đích

Tài liệu này cung cấp hướng dẫn toàn diện về Nix Flakes, từ cơ bản đến nâng cao, được thiết kế cho các cấp độ kỹ năng khác nhau:
- **Junior**: Cơ bản về Nix và Nix Flakes, cài đặt, sử dụng cơ bản
- **Middle**: Cấu hình flake.nix, tích hợp .NET/Node.js, direnv
- **Senior**: Advanced configuration, troubleshooting, best practices

---

## Phần 1: Cơ bản về Nix và Nix Flakes (Junior)

### 1.1. Nix là gì?

Nix là một công cụ quản lý package và build system được thiết kế để:
- Tạo môi trường phát triển tái tạo được (reproducible)
- Quản lý dependencies một cách deklarative
- Hỗ trợ rollback nhanh chóng
- Chia sẻ môi trường giữa các thành viên trong team

### 1.2. Nix Flakes là gì?

Nix Flakes là một tính năng mới của Nix giúp:
- Quản lý dependencies giữa các dự án
- Định nghĩa rõ ràng input và output của một dự án
- Tái sử dụng cấu hình dễ dàng hơn
- Hỗ trợ multiple outputs (packages, apps, devShells)
- Cải thiện tính modular và tái sử dụng

### 1.3. Tại sao sử dụng Nix Flakes?

**Lợi ích chính:**
1. **Reproducibility**: Môi trường phát triển nhất quán trên mọi máy
2. **Isolation**: Mỗi dự án có môi trường riêng, tránh xung đột dependencies
3. **Pinning**: Pin phiên bản cụ thể của packages, đảm bảo tính ổn định
4. **Rollback**: Dễ dàng rollback về phiên bản trước nếu có vấn đề
5. **CI/CD Friendly**: Tích hợp tốt với CI/CD pipeline

### 1.4. Các khái niệm cơ bản

#### Flake
Một Flake là một dự án Nix có file `flake.nix` ở thư mục gốc. Flake định nghĩa:
- **Inputs**: Các dependencies từ các nguồn khác (nixpkgs, GitHub, local flakes)
- **Outputs**: Các kết quả có thể được build từ flake (packages, apps, devShells)
- **Description**: Mô tả về flake

#### Flake Registry
Flake Registry là nơi lưu trữ các flakes có thể tái sử dụng:
- URL: https://flake.nixos.org
- Cho phép tìm kiếm và sử dụng flakes từ cộng đồng

#### Flake Input
Inputs là cách flake nhận dependencies:
- **nixpkgs**: Repository chính của Nix packages
- **GitHub URLs**: Flake từ GitHub repository
- **Local paths**: Flake từ thư mục local
- **Follows**: Theo dõi một flake khác (để nhận updates)

#### Flake Output
Outputs là những gì flake có thể cung cấp:
- **packages**: Các Nix packages có thể cài đặt
- **apps**: Các ứng dụng có thể chạy
- **devShells**: Môi trường phát triển có thể load

---

## Phần 2: Cài đặt Nix và kích hoạt Flakes (Junior)

### 2.1. Cài đặt Nix

#### Trên Linux/macOS
```bash
# Sử dụng script cài đặt đơn giản
sh <(curl -L https://nixos.org/nix/install | head -n 1)>

# Hoặc cài đặt qua package manager (nếu có)
# Ubuntu/Debian
sudo apt install nix

# macOS
brew install nix
```

#### Sau khi cài đặt
```bash
# Reload shell hoặc mở terminal mới
source ~/.nix-profile/etc/profile.d/nix.sh

# Kiểm tra cài đặt
nix --version
```

### 2.2. Kích hoạt Nix Flakes

Flakes vẫn là tính năng experimental, cần kích hoạt thủ công:

```bash
# Tạo thư mục config
mkdir -p ~/.config/nix

# Kích hoạt flakes
echo "experimental-features = nix-command flakes" >> ~/.config/nix/nix.conf
```

### 2.3. Kiểm tra Flakes đã kích hoạt

```bash
# Kiểm tra trong config
cat ~/.config/nix/nix.conf

# Hoặc kiểm tra bằng lệnh
nix flake --help
# Nếu thấy lệnh flake thì đã kích hoạt thành công
```

### 2.4. Nix Channels

Nix có nhiều channels khác nhau:

| Channel | Mô tả | Khi sử dụng |
|---------|---------|--------------|
| stable | Phiên bản ổn định, được test kỹ càng | Production |
| unstable | Phiên bản mới nhất, có thể không ổn định | Development, cần phiên bản mới |
| nixos-23.11 | Phiên bản ổn định 23.11 | Production |
| nixos-unstable | Phiên bản unstable của NixOS 23.11 | Development |

**Trong dự án TapHoaNho:**
- Sử dụng `nixos-unstable` để có truy cập đến .NET 9.0

---

## Phần 3: File flake.nix cơ bản (Junior)

### 3.1. Cấu trúc cơ bản

File `flake.nix` cơ bản:

```nix
{
  description = "Mô tả về flake của bạn";

  inputs = {
    # nixpkgs - repository chính của Nix packages
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";

    # flake-utils - thư viện tiện ích
    flake-utils.url = "github:numtide/flake-utils";
  };

  outputs = { self, nixpkgs, flake-utils }:
    let
      # Định nghĩa packages
      pkgs = nixpkgs.legacyPackages.${system};
    in
    {
      # Development shell
      devShells.default = pkgs.mkShell {
        buildInputs = [ pkgs.hello ];
      };
    };
}
```

### 3.2. Giải thích từng phần

#### Description
- Mô tả ngắn gọn về flake
- Hiển thị khi chạy `nix flake show`

#### Inputs
- **nixpkgs**: Repository chính chứa tất cả Nix packages
- **flake-utils**: Thư viện tiện ích giúp viết flake dễ dàng hơn
- Có thể thêm nhiều inputs từ GitHub, local paths, hoặc URLs khác

#### Outputs
- **devShells**: Môi trường phát triển có thể load bằng `nix develop`
- **packages**: Packages có thể cài đặt bằng `nix profile install`
- **apps**: Ứng dụng có thể chạy bằng `nix run`

### 3.3. Flake Utils

flake-utils là thư viện tiện ích giúp:
- Hỗ trợ multiple systems (x86_64, aarch64, v.v.)
- Giảm bớt code lặp lại (boilerplate)
- Cung cấp các helper functions phổ biến

**Sử dụng flake-utils:**
```nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    flake-utils.url = "github:numtide/flake-utils";
  };

  outputs = { self, nixpkgs, flake-utils }:
    flake-utils.lib.eachDefaultSystem (system:
      let
        pkgs = nixpkgs.legacyPackages.${system};
      in
      {
        devShells.default = pkgs.mkShell {
          buildInputs = [ pkgs.hello ];
        };
      }
    );
}
```

---

## Phần 4: Sử dụng nix develop (Junior)

### 4.1. Vào môi trường phát triển

```bash
# Cách 1: Sử dụng nix develop (thủ công)
nix develop

# Cách 2: Sử dụng nix shell (tương tự)
nix shell
```

### 4.2. Điều gì xảy ra khi chạy nix develop?

1. **Nix đọc file flake.nix**: Phân tích inputs và outputs
2. **Tạo môi trường**: Build hoặc download tất cả dependencies
3. **Load shell**: Đưa bạn vào shell với tất cả tools đã cài đặt
4. **Set environment variables**: Cấu hình các biến môi trường cần thiết

### 4.3. Thoát khỏi môi trường

```bash
# Nhấn Ctrl+D hoặc gõ exit
exit

# Môi trường sẽ tự động được unload
```

### 4.4. Kiểm tra môi trường

```bash
# Kiểm tra package đã cài
which <package-name>

# Kiểm tra biến môi trường
echo $VARIABLE_NAME

# Liệt kê tất cả packages trong môi trường
nix-store --query --references
```

---

## Phần 5: Cấu hình flake.nix cho dự án thực tế (Middle)

### 5.1. Cấu trúc flake.nix trong dự án TapHoaNho

File: [`shiny-carnival/flake.nix`](../shiny-carnival/flake.nix:1-88)

```nix
{
  description = "Môi trường phát triển cho Retail Store Management System";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";  # Dùng unstable để có .NET 9.0
    flake-utils.url = "github:numtide/flake-utils";
  };

  outputs = { self, nixpkgs, flake-utils }:
    flake-utils.lib.eachDefaultSystem (system:
      let
        pkgs = import nixpkgs {
          inherit system;
        };

        # Node.js version - sử dụng LTS 20
        nodejs = pkgs.nodejs_20;

        # .NET 9.0 SDK
        dotnet-sdk = pkgs.dotnet-sdk_9;

        # PostgreSQL 16
        postgresql = pkgs.postgresql_16;

        # Yarn - sử dụng yarn-berry (yarn 4.x)
        yarn = pkgs.yarn-berry;

        # Các công cụ phát triển
        devTools = with pkgs; [
          # Công cụ cơ bản (tr, grep, sed, head, v.v.)
          coreutils
          git
          curl
          jq
          # Công cụ để quản lý database
          postgresql
          # Công cụ để build .NET
          dotnet-sdk
          # Node.js và package manager
          nodejs
          yarn
          # TypeScript compiler (có thể cần global)
          nodePackages.typescript
          # ESLint (có thể cần global)
          nodePackages.eslint
        ];
      in
      {
        devShells.default = pkgs.mkShell {
          buildInputs = devTools;

          shellHook = ''
            # Đặt PATH từ Nix TRƯỚC mọi thứ khác để đảm bảo ưu tiên tuyệt đối
            export PATH="${pkgs.lib.makeBinPath devTools}:$PATH"

            # Disable Corepack để tránh xung đột với yarn từ nix
            export COREPACK_ENABLE_STRICT=0
            # Xóa corepack khỏi hash table nếu có
            hash -d corepack 2>/dev/null || true

            echo "🚀 Retail Store Management System - Development Environment"
            echo "=================================================="
            echo ""
            echo "📦 Công cụ đã cài đặt:"
            echo "  • Node.js: $(node --version 2>/dev/null || echo 'N/A')"
            echo "  • Yarn: $(yarn --version 2>/dev/null || echo 'N/A')"
            echo "  • .NET SDK: $(dotnet --version 2>/dev/null || echo 'N/A')"
            echo "  • PostgreSQL: $(psql --version 2>/dev/null | head -n1 || echo '16.x (installed)')"
            echo ""
            echo "📁 Cấu trúc dự án:"
            echo "  • Frontend: ./frontend"
            echo "  • Backend: ./RetailStoreManagement"
            echo ""
            echo "🔧 Lệnh hữu ích:"
            echo "  • Frontend dev: cd frontend && yarn dev"
            echo "  • Backend dev: cd RetailStoreManagement && dotnet run"
            echo "  • Restore packages: cd frontend && yarn install"
            echo "  • Restore .NET: cd RetailStoreManagement && dotnet restore"
            echo ""
          '';

          # Biến môi trường
          DOTNET_ROOT = "${dotnet-sdk}";
          # PATH được set trong shellHook để đảm bảo ưu tiên
        };
      }
    );
}
```

### 5.2. Giải thích cấu hình

#### Dòng 5: nixos-unstable
- Sử dụng unstable channel để có truy cập đến .NET 9.0
- .NET 9.0 chỉ có sẵn trong unstable channel

#### Dòng 20: dotnet-sdk_9
- Cài đặt .NET SDK phiên bản 9.0
- Cung cấp các công cụ dotnet CLI (build, run, test, v.v.)

#### Dòng 17: nodejs_20
- Cài đặt Node.js LTS phiên bản 20
- Runtime cho frontend React

#### Dòng 26: yarn-berry
- Cài đặt Yarn 4.x (yarn-berry)
- Package manager cho Node.js

#### Dòng 54: PATH từ Nix ưu tiên
- Đảm bảo tools từ Nix được ưu tiên trong PATH
- Tránh xung đột với tools từ hệ thống hoặc NVM

#### Dòng 57-59: Disable Corepack
- Corepack có thể can thiệp và download yarn 1.x
- Disable để đảm bảo yarn 4.x từ Nix được sử dụng

#### Dòng 83: DOTNET_ROOT
- Đặt biến môi trường cho .NET
- Giúp .NET CLI tìm đúng SDK

---

## Phần 6: Tích hợp .NET Core với Nix (Middle)

### 6.1. Các phiên bản .NET có sẵn trong Nixpkgs

| Phiên bản | Package Nixpkgs | Channel cần thiết |
|-----------|-------------------|------------------|
| .NET 8.0 | dotnet-sdk_8 | stable, nixos-23.11 |
| .NET 9.0 | dotnet-sdk_9 | unstable, nixos-unstable |

### 6.2. Quản lý NuGet Dependencies

#### Cách hoạt động của NuGet trong môi trường Nix

1. **Packages được lưu trữ trong thư mục người dùng**:
   - `/home/nguyen-thanh-hung/.nuget/packages/`
   - Nix không quản lý trực tiếp NuGet packages

2. **NuGet Config được tự động tạo**:
   - File config tại `/home/nguyen-thanh-hung/.nuget/NuGet/NuGet.Config`
   - Cấu hình cache và sources

3. **NuGet Packages trong .csproj**:
   - Các packages được khai báo sử dụng `PackageReference`
   - Tự động tải về khi chạy `dotnet restore`

#### Ví dụ PackageReference

File: [`shiny-carnival/RetailStoreManagement/src/Infrastructure/Infrastructure.csproj`](../shiny-carnival/RetailStoreManagement/src/Infrastructure/Infrastructure.csproj:15-25)

```xml
<ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore" Version="9.0.0" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="9.0.0">
        <PrivateAssets>all</PrivateAssets>
        <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
    <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="9.0.2" />
    <PackageReference Include="EFCore.NamingConventions" Version="9.0.0" />
    <PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="9.0.0" />
    <PackageReference Include="BCrypt.Net-Next" Version="4.0.3" />
</ItemGroup>
```

### 6.3. Chạy lệnh dotnet trong môi trường Nix

#### Restore packages
```bash
# Restore packages (tự động khi vào shell)
dotnet restore
```

#### Build
```bash
# Build
dotnet build
```

#### Run development server
```bash
# Chạy development server
dotnet run
```

#### Chạy Entity Framework migrations
```bash
# Chạy migrations
dotnet ef database update
```

### 6.4. Xử lý global.json

File `global.json` được sử dụng để pin phiên bản .NET:

```json
{
  "sdk": {
    "version": "9.0.100"
  }
}
```

**Lưu ý:**
- Nếu `global.json` tồn tại, .NET CLI sẽ sử dụng phiên bản được chỉ định
- Đảm bảo phiên bản trong `global.json` khớp với phiên bản trong Nix (`pkgs.dotnet-sdk_9`)

---

## Phần 7: Tích hợp Node.js/React với Nix (Middle)

### 7.1. Các phiên bản Node.js có sẵn trong Nixpkgs

| Phiên bản | Package Nixpkgs | Mô tả |
|-----------|-------------------|---------|
| Node.js 18 | nodejs_18 | Previous LTS |
| Node.js 20 | nodejs_20 | Current LTS |
| Node.js 21 | nodejs_21 | Current |

### 7.2. Package Managers cho Node.js

#### Yarn
- **yarn-berry**: Yarn 4.x (được sử dụng trong dự án)
- **yarn**: Yarn 1.x (phiên bản cũ, được cài đặt qua corepack)

#### NPM
- **nodejs**: NPM được cài đặt cùng với Node.js
- Có thể sử dụng thay cho Yarn

#### pnpm
- **nodePackages.pnpm**: pnpm từ Nixpkgs
- Package manager nhanh hơn NPM

### 7.3. Chạy lệnh Node.js trong môi trường Nix

#### Cài đặt dependencies
```bash
cd frontend

# Cài đặt dependencies
yarn install

# Hoặc với npm
npm install
```

#### Chạy development server
```bash
cd frontend

# Chạy development server
yarn dev

# Hoặc với npm
npm run dev
```

#### Build production
```bash
cd frontend

# Build
yarn build

# Hoặc với npm
npm run build
```

### 7.4. Xử lý Corepack và NVM

#### Vấn đề Corepack
- Corepack có thể can thiệp và download yarn 1.x
- Xung đột với yarn 4.x từ Nix

#### Giải pháp trong flake.nix
```nix
shellHook = ''
  # Disable Corepack để tránh xung đột với yarn từ nix
  export COREPACK_ENABLE_STRICT=0
  # Xóa corepack khỏi hash table nếu có
  hash -d corepack 2>/dev/null || true
'';
```

#### Vấn đề NVM
- NVM có thể ưu tiên yarn/node từ NVM
- Gây ra xung đột với tools từ Nix

#### Giải pháp
1. **Đảm bảo PATH từ Nix được ưu tiên**:
   ```nix
   export PATH="${pkgs.lib.makeBinPath devTools}:$PATH"
   ```

2. **Tạm thời disable NVM trong Nix shell**:
   ```bash
   unset NVM_DIR
   ```

---

## Phần 8: Sử dụng Direnv (Middle)

### 8.1. Direnv là gì?

Direnv là công cụ giúp:
- Tự động load/unload môi trường khi vào/ra thư mục
- Dựa trên file `.envrc` trong mỗi thư mục dự án
- Giảm bớt việc phải chạy `nix develop` thủ công

### 8.2. Cài đặt Direnv

```bash
# Cài đặt direnv qua Nix
nix profile install nixpkgs#direnv

# Hoặc nếu bạn dùng NixOS hoặc có nix-env:
nix-env -iA nixos.direnv
```

### 8.3. Cấu hình shell hook

#### Bash
```bash
# Thêm vào ~/.bashrc
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc

# Reload shell config
source ~/.bashrc
```

#### Zsh
```bash
# Thêm vào ~/.zshrc
echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc

# Reload shell config
source ~/.zshrc
```

#### Fish
```bash
# Thêm vào ~/.config/fish/config.fish
echo 'direnv hook fish | source' >> ~/.config/fish/config.fish
```

### 8.4. File .envrc trong dự án TapHoaNho

File: [`shiny-carnival/.envrc`](../shiny-carnival/.envrc:1-23)

```bash
# Đảm bảo nix command có sẵn trước khi load flake
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

# Load flake development shell
use flake
```

### 8.5. Giải thích từng phần

#### Dòng 8-10: Thêm PATH của Nix
- Đảm bảo `nix` command có sẵn trước khi direnv cố load flake
- Giải quyết vấn đề "chicken and egg"

#### Dòng 12-19: Source Nix profile
- Load đầy đủ môi trường Nix
- Kiểm tra các vị trí phổ biến của Nix profile

#### Dòng 22: use flake
- Lệnh direnv để load development shell từ `flake.nix`
- Tự động chạy khi `cd` vào thư mục dự án

### 8.6. Cho phép direnv

```bash
# Vào thư mục dự án
cd /home/nguyen-thanh-hung/Documents/TapHoaNho/shiny-carnival

# Cho phép direnv (chỉ cần chạy 1 lần)
direnv allow
```

### 8.7. Kiểm tra direnv

```bash
# Kiểm tra trạng thái
direnv status

# Xem các biến môi trường được export
direnv export bash | grep PATH
```

---

## Phần 9: Advanced Configuration trong flake.nix (Senior)

### 9.1. Multiple DevShells

Có thể định nghĩa nhiều devShells khác nhau:

```nix
outputs = { self, nixpkgs, ... }:
{
  devShells.default = pkgs.mkShell { ... };
  devShells.withPostgres = pkgs.mkShell {
    buildInputs = devTools ++ [ pkgs.postgresql ];
  };
  devShells.minimal = pkgs.mkShell {
    buildInputs = [ pkgs.git ];
  };
}
```

**Sử dụng:**
```bash
# Load shell mặc định
nix develop

# Load shell cụ thể
nix develop .#withPostgres
```

### 9.2. Custom Packages

Có thể tạo custom packages trong flake:

```nix
outputs = { self, nixpkgs, ... }:
{
  packages.my-custom-app = pkgs.stdenv.mkDerivation {
    name = "my-custom-app";
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
}
```

### 9.3. Shell Hooks Nâng cao

#### Pre-shell hook
```nix
shellHook = ''
  # Chạy trước khi vào shell
  echo "Đang chuẩn bị môi trường..."
'';
```

#### Post-shell hook
```nix
shellHook = ''
  # Chạy sau khi vào shell
  echo "Môi trường đã sẵn sàng!"
  # Chạy các lệnh setup
'';
```

### 9.4. Environment Variables

Có thể thêm nhiều environment variables:

```nix
devShells.default = pkgs.mkShell {
  shellHook = ''
    # Database
    export DATABASE_URL="postgresql://localhost:5432/mydb"

    # API Keys
    export API_KEY="your-api-key"

    # .NET
    export DOTNET_ROOT="${dotnet-sdk}"
    export DOTNET_CLI_HOME="${HOME}/.dotnet"

    # Node.js
    export NODE_ENV="development"
  '';
}
```

**Lưu ý bảo mật:**
- Không đặt secrets trực tiếp trong `flake.nix` (file được commit vào git)
- Sử dụng file `.env` riêng (đã được thêm vào `.gitignore`)

---

## Phần 10: Flake Outputs - Packages, Apps, DevShells (Senior)

### 10.1. Packages Output

Packages output cho phép cài đặt packages từ flake:

```nix
outputs = { self, nixpkgs, ... }:
{
  packages.my-tool = pkgs.writeShellScriptBin {
    name = "my-tool";
    text = ''
      #!/bin/sh
      echo "Hello from my-tool!"
    '';
  };
}
```

**Sử dụng:**
```bash
# Cài đặt package
nix profile install .#my-tool

# Hoặc chạy trực tiếp
nix run .#my-tool
```

### 10.2. Apps Output

Apps output cho phép chạy ứng dụng:

```nix
outputs = { self, nixpkgs, ... }:
{
  apps.my-app = {
    type = "app";
    program = pkgs.hello;
  };
}
```

**Sử dụng:**
```bash
# Chạy app
nix run .#my-app
```

### 10.3. DevShells Output

DevShells output cho phép load môi trường phát triển:

```nix
outputs = { self, nixpkgs, ... }:
{
  devShells.default = pkgs.mkShell {
    buildInputs = [ pkgs.nodejs_20 pkgs.yarn-berry ];
  };
}
```

**Sử dụng:**
```bash
# Load dev shell
nix develop
```

### 10.4. So sánh giữa các outputs

| Output | Mô tả | Sử dụng |
|--------|---------|----------|
| packages | Packages có thể cài đặt | `nix profile install`, `nix run` |
| apps | Ứng dụng có thể chạy | `nix run` |
| devShells | Môi trường phát triển | `nix develop` |

---

## Phần 11: Flake Inputs và Dependencies (Senior)

### 11.1. Input Types

#### nixpkgs
```nix
inputs = {
  nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
}
```

#### GitHub URL
```nix
inputs = {
  my-flake.url = "github:username/repo";
}
```

#### Local path
```nix
inputs = {
  local-flake.url = "../other-flake";
}
```

#### Follows
```nix
inputs = {
  nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
  nixpkgs.follows = "/nixpkgs";  # Theo dõi nixpkgs
}
```

### 11.2. Pinning Inputs

#### Pin đến commit cụ thể
```nix
inputs = {
  nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable?ref=nixos-23.11";
}
```

#### Pin đến tag cụ thể
```nix
inputs = {
  nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable?ref=23.11";
}
```

#### Pin đến commit hash
```nix
inputs = {
  nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable?ref=a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6";
}
```

### 11.3. Updating Inputs

```bash
# Update tất cả inputs
nix flake update

# Update input cụ thể
nix flake update nixpkgs

# Lock file
nix flake lock
```

### 11.4. Flake Lock

File `flake.lock` được tạo tự động để:
- Pin phiên bản cụ thể của tất cả inputs
- Đảm bảo tính reproducibility
- Nên được commit vào git

---

## Phần 12: Troubleshooting Nâng cao (Senior)

### 12.1. Lỗi: "command not found: dotnet"

**Nguyên nhân:**
- Chưa vào môi trường Nix
- Direnv chưa được allow

**Giải pháp:**
```bash
# Đảm bảo bạn đã vào nix shell
nix develop

# Hoặc nếu dùng direnv
direnv allow
```

### 12.2. Lỗi: "Corepack is about to download yarn-1.22.22.tgz"

**Nguyên nhân:**
- Corepack đang can thiệp và download yarn 1.x
- Yarn từ NVM đang được ưu tiên

**Giải pháp:**
1. **Kiểm tra yarn đang dùng từ đâu**:
   ```bash
   which yarn
   # Phải là /nix/store/.../yarn-berry-4.12.0/bin/yarn
   ```

2. **Kiểm tra yarn version**:
   ```bash
   yarn --version
   # Phải là 4.x (ví dụ: 4.10.3)
   ```

3. **Disable corepack tạm thời**:
   ```bash
   export COREPACK_ENABLE_STRICT=0
   ```

4. **Nếu dùng nvm, tạm thời disable nvm**:
   ```bash
   unset NVM_DIR
   ```

### 12.3. Lỗi: Direnv không tự động load

**Nguyên nhân:**
- Hook chưa được thêm vào shell config
- Chưa reload shell config
- Direnv chưa được allow

**Giải pháp:**
1. **Kiểm tra hook đã được thêm**:
   ```bash
   grep "direnv hook" ~/.bashrc  # hoặc ~/.zshrc
   ```

2. **Reload shell config**:
   ```bash
   source ~/.bashrc  # hoặc source ~/.zshrc
   ```

3. **Kiểm tra direnv đã được allow**:
   ```bash
   direnv status
   ```

4. **Mở terminal mới** nếu vẫn không hoạt động

### 12.4. Lỗi: "nix: command not found" khi direnv load

**Nguyên nhân:**
- Direnv cần `nix` command nhưng `nix` chưa có trong PATH
- File `.envrc` chưa được cấu hình đúng

**Giải pháp:**
1. **Kiểm tra file `.envrc` có đúng không**:
   ```bash
   cat .envrc
   # Phải có các dòng thêm PATH và source nix profile
   ```

2. **Đảm bảo nix đã được cài đặt**:
   ```bash
   which nix
   ```

3. **Nếu vẫn lỗi, thử chạy direnv allow lại**:
   ```bash
   direnv allow
   ```

### 12.5. Lỗi: ".envrc is blocked" sau khi chỉnh sửa

**Nguyên nhân:**
- Sau khi chỉnh sửa file `.envrc`, direnv sẽ block file

**Giải pháp:**
```bash
# Xem nội dung thay đổi
direnv diff

# Phê duyệt thay đổi
direnv allow
```

### 12.6. Lỗi: Build fails với .NET

**Nguyên nhân:**
- Phiên bản .NET không khớp giữa global.json và Nix
- Missing dependencies

**Giải pháp:**
1. **Kiểm tra global.json**:
   ```bash
   cat global.json
   # Đảm bảo phiên bản khớp với pkgs.dotnet-sdk_9
   ```

2. **Xóa global.json nếu không cần**:
   ```bash
   rm global.json
   ```

3. **Restore lại**:
   ```bash
   dotnet restore
   ```

### 12.7. Lỗi: Node modules không tìm thấy

**Nguyên nhân:**
- Chưa chạy `yarn install` hoặc `npm install`
- node_modules bị xóa

**Giải pháp:**
```bash
cd frontend
yarn install
```

### 12.8. Debugging Nix

#### Kiểm tra store
```bash
# Liệt kê tất cả packages trong store
nix-store --query --references

# Kiểm tra package cụ thể
nix-store -q <package-name>
```

#### Kiểm tra derivation
```bash
# Xem log của build
nix log

# Xem derivation đang chạy
nix show-derivation
```

#### Garbage collection
```bash
# Xóa các packages không được sử dụng
nix-collect-garbage -d

# Xóa các old generations
nix-collect-garbage --delete-old
```

---

## Phần 13: Best Practices cho Nix Flakes (Senior)

### 13.1. File Organization

```
project/
├── flake.nix              # Cấu hình chính
├── flake.lock             # Lock file (commit vào git)
├── .envrc                 # Direnv config (commit vào git)
├── .env                   # Secrets (KHÔNG commit vào git)
├── frontend/
│   ├── package.json
│   └── node_modules/    # Thêm vào .gitignore
└── backend/
    ├── *.csproj
    └── bin/                # Thêm vào .gitignore
```

### 13.2. Version Control

**Nên commit vào git:**
- `flake.nix` - Cấu hình chính
- `flake.lock` - Lock file
- `.envrc` - Direnv config

**KHÔNG commit vào git:**
- `.env` - Chứa secrets
- `node_modules/` - Dependencies
- `bin/`, `obj/` - Build artifacts

### 13.3. Reproducibility

**Đảm bảo tính reproducibility:**
1. Commit `flake.lock` vào git
2. Sử dụng cùng channel trên mọi máy
3. Pin inputs đến version cụ thể
4. Không sử dụng packages từ hệ thống

### 13.4. Performance

**Tối ưu hóa hiệu suất:**
1. Sử dụng binary cache
2. Tận dụng Nix store cache
3. Tránh build lại packages đã có
4. Sử dụng `nix build` thay vì `nix develop` khi có thể

### 13.5. Security

**Các thực hành bảo mật:**
1. Không đặt secrets trong `flake.nix`
2. Sử dụng file `.env` riêng cho secrets
3. Thêm `.env` vào `.gitignore`
4. Review dependencies thường xuyên
5. Cập nhật packages thường xuyên

### 13.6. Team Collaboration

**Làm việc với team:**
1. Sử dụng cùng flake structure
2. Commit `flake.lock` vào git
3. Document các custom packages
4. Chia sẻ flake patterns tái sử dụng được

### 13.7. CI/CD Integration

**Tích hợp với CI/CD:**
1. Sử dụng `nix build` trong CI
2. Cache Nix store
3. Sử dụng `nix flake check` để validate
4. Test trên multiple systems

---

## Tham khảo

- [Nix Flakes Documentation](https://nixos.wiki/wiki/Flakes)
- [Nixpkgs](https://search.nixos.org/packages)
- [Flake Utils](https://github.com/numtide/flake-utils)
- [Nix Pills](https://nixos.org/guides/nix-pills/)

## File liên quan trong dự án

- [`shiny-carnival/flake.nix`](../shiny-carnival/flake.nix:1-88) - Cấu hình chính
- [`shiny-carnival/.envrc`](../shiny-carnival/.envrc:1-23) - Direnv config
- [`shiny-carnival/NIX_SETUP.md`](../shiny-carnival/NIX_SETUP.md:1-427) - Hướng dẫn setup
- [`shiny-carnival/docs/NIX_DOTNET_INTEGRATION.md`](../shiny-carnival/docs/NIX_DOTNET_INTEGRATION.md:1-774) - Báo cáo tích hợp .NET
