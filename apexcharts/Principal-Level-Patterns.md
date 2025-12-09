# ApexCharts + React – Principal/Staff Patterns

## Kiến trúc & system design
- Centralized chart layer: tạo module `charts/` chứa base theme, palette, formatter (number/date), helper tạo axes; UI chỉ truyền data domain.
- Multi-tenant/theming: cấu hình theme theo tenant (light/dark/palette) và truyền xuống `ChartBase`.
- Versioned config: lock version `apexcharts`/`react-apexcharts` và có regression checklist khi upgrade.

## SSR/SSG & data loading
- Next/Remix: render chart client-side (`dynamic({ ssr:false })`) để tránh window undefined; prefetch data tại server và truyền vào component.
- Streaming/partial data: render skeleton hoặc chart rỗng, cập nhật series khi data đến; tắt animation cho incremental update.
- Prefetch chiến lược: load aggregated data cho overview; drill-down mới gọi chi tiết.

## Observability & monitoring
- Instrument render: log lỗi khi options/series invalid, đếm số điểm > ngưỡng.
- Theo dõi thời gian render chart và số lần re-render để phát hiện bottleneck (React Profiler/Sentry perf).
- Tracing luồng data (API → adapter → series) để dễ debug sai lệch.

## Resiliency & error recovery
- Fallback chart: khi thiếu dữ liệu, render empty state thay vì lỗi.
- Debounce/throttle cập nhật series với nguồn realtime để tránh nghẽn.
- Hard limit kích thước payload; cắt bớt và hiển thị cảnh báo nếu vượt ngưỡng.

## Migration & interoperability
- Lộ trình chuyển từ chart cũ: tạo adapter map schema cũ sang `{ x, y }`/range format; đồng thời duy trì snapshot test.
- Tách lớp formatter (number/date/percent) dùng chung giữa thư viện chart để giảm lock-in.

## Production checklist
- Kiểm tra dark mode, RTL, responsive (mobile/desktop).
- Kiểm soát font & đo chiều cao container cố định để tránh layout shift.
- Đảm bảo a11y: màu có tương phản đủ, legend/tooltip dễ đọc; cung cấp bảng dữ liệu fallback nếu cần.

