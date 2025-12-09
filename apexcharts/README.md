# ApexCharts + React – Bản đồ tài liệu

## Giới thiệu nhanh
- ApexCharts là thư viện chart JS hiện đại với nhiều loại biểu đồ (line, area, bar, pie, candlestick...) và API cấu hình đơn giản.
- Dùng wrapper `react-apexcharts` để nhúng vào React, hỗ trợ dữ liệu động và responsive tốt.

## Cấu trúc tài liệu
- `apexcharts.md`: Hướng dẫn đầy đủ (Junior/Middle/Senior) về cài đặt, API cơ bản, use case phổ biến, best practices.
- `Advanced-Patterns.md`: Cấu hình nâng cao, tối ưu hiệu năng, custom hook, testing nâng cao.
- `Principal-Level-Patterns.md`: Pattern ở quy mô enterprise (SSR/SSG, data loading chiến lược, resiliency, monitoring, migration).
- `RESEARCH_SUMMARY.md`: Mục tiêu research, danh sách nguồn (URL, tiêu đề, ngày truy cập), key findings, đề xuất kiến trúc & use case.

## Cách sử dụng tài liệu
- Junior: Đọc `apexcharts.md` phần Giới thiệu & Junior Level, làm theo Quick Start.
- Middle: Đọc tiếp phần Middle (pagination, live update, event handling).
- Senior: Đọc phần Senior + `Advanced-Patterns.md` (performance, custom hooks, testing).
- Principal/Staff: Đọc `Principal-Level-Patterns.md` (SSR/SSG, observability, resiliency, migration).

## Key Concepts
- `ReactApexChart` component nhận `options` (config) và `series` (data); thay đổi state ⇒ chart rerender.
- `type` quyết định hành vi (line/area/bar/pie/rangeBar/candlestick/heatmap/treemap...).
- Responsive & theming qua `responsive`, `theme`, `fill`, `stroke`, `plotOptions`.

## Quick Start (React)
1) Cài đặt:
   - `npm install apexcharts react-apexcharts`
2) Dùng tối thiểu:
   - Import `ReactApexChart`, truyền `options`, `series`, `type`, `height/width`.
   - Giữ `options` ổn định (memo) để tránh re-render không cần thiết.

## External Resources
- Official docs: https://apexcharts.com/docs/react-charts/
- GitHub samples (React): https://github.com/apexcharts/apexcharts.js/tree/main/samples/react
- Blog so sánh chart libs (React): https://blog.logrocket.com/best-react-chart-libraries-2025/

