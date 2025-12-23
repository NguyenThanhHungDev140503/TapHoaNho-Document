# Tài Liệu Routing Architecture

Thư mục này chứa tài liệu chi tiết về kiến trúc routing của dự án.

## Các File Trong Thư Mục

### 📄 RoutingArchitecture.md
Tài liệu chính mô tả chi tiết về:
- Kiến trúc tổng thể của hệ thống routing
- Các khái niệm cốt lõi (ModuleRoutes, HierarchicalModuleRouteConfig, Layout Routes, Guards)
- Luồng xử lý routing (khởi tạo, navigation, route tree construction)
- Call graph và control flow diagrams (Mermaid)
- Hướng dẫn tạo module mới
- Best practices

**Đối tượng**: Tất cả developers trong team, đặc biệt là developers mới vào team.

### 📊 routing-call-graph.png
Hình ảnh call graph chi tiết thể hiện luồng gọi hàm trong quá trình xây dựng route tree.

**Được tạo từ**: `routing-call-graph.dot`

### 🔧 routing-call-graph.dot
File Graphviz DOT chứa định nghĩa call graph. Có thể chỉnh sửa và tạo lại hình ảnh bằng lệnh:

```bash
dot -Tpng routing-call-graph.dot -o routing-call-graph.png
```

**Yêu cầu**: Cài đặt Graphviz (`sudo apt install graphviz` hoặc tương đương)

## Cách Sử Dụng

### Cho Developers Mới

1. **Bắt đầu với**: `RoutingArchitecture.md`
   - Đọc phần "Tổng Quan" để hiểu tổng thể
   - Đọc phần "Các Khái Niệm Cốt Lõi" để nắm vững các khái niệm
   - Xem các diagram trong phần "Call Graph & Control Flow"
   - Tham khảo "Hướng Dẫn Tạo Module Mới" khi cần thêm route

2. **Tham khảo call graph**: Xem `routing-call-graph.png` để hiểu luồng gọi hàm

### Khi Cần Cập Nhật Tài Liệu

1. **Cập nhật call graph**:
   - Chỉnh sửa `routing-call-graph.dot`
   - Chạy lệnh: `dot -Tpng routing-call-graph.dot -o routing-call-graph.png`
   - Commit cả file `.dot` và `.png`

2. **Cập nhật tài liệu chính**:
   - Chỉnh sửa `RoutingArchitecture.md`
   - Đảm bảo các diagram Mermaid vẫn render đúng
   - Cập nhật phần "Tóm Tắt" nếu có thay đổi lớn

## Liên Kết Liên Quan

- [CreateRouteGuide.md](./CreateRouteGuide.md) - Hướng dẫn tạo route mới
- [FSD with Code-Based.md](./FSD%20with%20Code-Based.md) - So sánh với mô hình FSD

## Ghi Chú

- Tài liệu này được cập nhật khi có thay đổi lớn về kiến trúc routing
- Các diagram Mermaid có thể được render trực tiếp trên GitHub/GitLab
- File `.dot` được giữ lại để dễ dàng chỉnh sửa và tạo lại hình ảnh







