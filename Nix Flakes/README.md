# Nix Flakes - Tài liệu Tham Khảo

## Tổng quan

Nix Flakes là một tính năng của Nix giúp quản lý và tái sử dụng các cấu hình môi trường phát triển một cách hiệu quả. Tài liệu này cung cấp hướng dẫn toàn diện về cách sử dụng Nix Flakes trong dự án TapHoaNho (Fullstack .NET Core + React).

## Mục đích của bộ tài liệu

Bộ tài liệu này được thiết kế để:
- Giúp lập trình viên hiểu và sử dụng Nix Flakes hiệu quả
- Cung cấp hướng dẫn cho các cấp độ kỹ năng khác nhau (Junior, Middle, Senior, Principal)
- Tích hợp kinh nghiệm thực tế từ dự án TapHoaNho
- Giải quyết các vấn đề thường gặp khi làm việc với Nix Flakes

## Bản đồ tài liệu

```
Nix Flakes/
├── README.md                    # File này - Tổng quan & bản đồ tài liệu
├── Nix Flakes.md               # File chính - Hướng dẫn cho Junior/Middle/Senior
├── Advanced-Patterns.md        # Patterns nâng cao - Dành cho Senior
├── Principal-Level-Patterns.md  # Patterns cấp Principal - Dành cho Principal
└── RESEARCH_SUMMARY.md         # Tóm tắt quá trình research
```

### Hướng dẫn theo cấp độ kỹ năng

#### Cấp Junior
Bắt đầu với file **[Nix Flakes.md](Nix Flakes.md)**:
- Phần 1: Cơ bản về Nix và Nix Flakes
- Phần 2: Cài đặt Nix và kích hoạt Flakes
- Phần 3: File flake.nix cơ bản
- Phần 4: Sử dụng nix develop

#### Cấp Middle
Tiếp tục với file **[Nix Flakes.md](Nix Flakes.md)**:
- Phần 5: Cấu hình flake.nix cho dự án thực tế
- Phần 6: Tích hợp .NET Core với Nix
- Phần 7: Tích hợp Node.js/React với Nix
- Phần 8: Sử dụng Direnv

#### Cấp Senior
Đọc file **[Nix Flakes.md](Nix Flakes.md)** và **[Advanced-Patterns.md](Advanced-Patterns.md)**:
- Phần 9: Advanced configuration trong flake.nix
- Phần 10: Flake outputs (packages, apps, devShells)
- Phần 11: Flake inputs và dependencies
- Phần 12: Troubleshooting nâng cao
- Phần 13: Best practices cho Nix Flakes

#### Cấp Principal
Đọc file **[Principal-Level-Patterns.md](Principal-Level-Patterns.md)**:
- Flake composition và reusability
- Nixpkgs contribution
- Binary cache optimization
- CI/CD integration với Nix
- Enterprise Nix patterns

## Cách sử dụng bộ tài liệu

### Đối với người mới bắt đầu
1. Đọc file **[Nix Flakes.md](Nix Flakes.md)** từ đầu đến cuối
2. Thực hiện các ví dụ trong tài liệu trên môi trường thực tế
3. Tham khảo **[RESEARCH_SUMMARY.md](RESEARCH_SUMMARY.md)** để hiểu quá trình research

### Đối với lập trình viên có kinh nghiệm
1. Đọc **[Advanced-Patterns.md](Advanced-Patterns.md)** để học các patterns nâng cao
2. Áp dụng các patterns vào dự án của bạn
3. Tham khảo **[Principal-Level-Patterns.md](Principal-Level-Patterns.md)** cho các kiến trúc lớn

### Đối với Principal/Architect
1. Đọc **[Principal-Level-Patterns.md](Principal-Level-Patterns.md)** để hiểu các patterns cấp cao
2. Áp dụng các patterns để thiết kế hệ thống Nix cho tổ chức
3. Tích hợp Nix vào CI/CD pipeline

## Cấu trúc dự án TapHoaNho

Dự án TapHoaNho sử dụng Nix Flakes với cấu trúc sau:

```
shiny-carnival/
├── flake.nix                    # Cấu hình Nix Flakes chính
├── .envrc                       # Cấu hình Direnv
├── frontend/                     # React/Node.js frontend
├── RetailStoreManagement/       # .NET Core backend
└── docs/                         # Tài liệu
    └── NIX_DOTNET_INTEGRATION.md
```

## Tham khảo nhanh

### Lệnh cơ bản
```bash
# Vào môi trường Nix
nix develop

# Build flake
nix build

# Run package từ flake
nix run <package-name>
```

### File quan trọng
- [`flake.nix`](../shiny-carnival/flake.nix) - Cấu hình chính
- [`.envrc`](../shiny-carnival/.envrc) - Direnv config

### Links nhanh
- [Nix Flakes Documentation](https://nixos.wiki/wiki/Flakes)
- [.NET 9.0 Documentation](https://learn.microsoft.com/dotnet/)
- [Node.js 20 Documentation](https://nodejs.org/docs/latest-v20.x)

## Nguồn tham khảo

- [NixOS Wiki](https://nixos.wiki/)
- [Nix Flakes Examples](https://github.com/NixOS/nixpkgs/tree/master/flake-module-registry)
- [Nix Pills](https://nixos.org/guides/nix-pills/)

## Phiên bản

- Phiên bản tài liệu: 1.0.0
- Ngày tạo: 2026-01-03
- Nix version: 2.18+
- Nixpkgs channel: unstable
