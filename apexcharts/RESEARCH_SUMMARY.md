# Research Summary – ApexCharts + React

## 1. Mục tiêu
- Hiểu cách dùng ApexCharts trong React (wrapper `react-apexcharts`), xây dựng bộ tài liệu đa cấp (Junior → Principal).
- Xác định best practices, tối ưu hiệu năng, pattern enterprise (SSR/SSG, observability, resiliency).

## 2. Nguồn thông tin đã sử dụng
- Official docs – React guide, truy cập 2025-12-09: https://apexcharts.com/docs/react-charts/
- GitHub samples (React), truy cập 2025-12-09: https://github.com/apexcharts/apexcharts.js/tree/main/samples/react
- LogRocket – Best React chart libraries 2025, truy cập 2025-12-09: https://blog.logrocket.com/best-react-chart-libraries-2025/
- Genspark – ApexCharts resources overview, truy cập 2025-12-09: https://www.genspark.ai/spark/apexcharts-resources-and-documentation/d99e865f-cb19-48a7-9500-5f5c7ed627b1
- Kite Metric – React dashboard chart libs comparison, truy cập 2025-12-09: https://kitemetric.com/blogs/choosing-the-right-charting-library-for-your-react-dashboard

## 3. Phát hiện chính
- Ưu điểm: API cấu hình, nhiều chart type, animation/responsive tốt, wrapper React đơn giản.
- Hạn chế: Dễ re-render nếu không memo `options`; data lớn cần downsample/tắt animation; SSR cần client-only render.
- So sánh: So với Chart.js/Recharts, ApexCharts mạnh ở cấu hình nhanh và nhiều loại biểu đồ; visx/D3 linh hoạt hơn nhưng khó hơn.

## 4. Kiến trúc / cách dùng đề xuất
- Dùng `ChartBase` bọc `ReactApexChart`, áp dụng theme mặc định, nhận `options/series` đã memo.
- Adapter data chuẩn `{ x, y }` (datetime dùng timestamp), lock `chart.id` cho brush/selection.
- Downsample hoặc phân trang dữ liệu lớn; tắt `dataLabels/markers` khi >2k điểm.
- Client-only render cho Next/Remix; prefetch data và truyền vào chart để tránh flash rỗng.

## 5. Use cases đã xác định
- Dashboard KPI (line/area/bar) – xem `apexcharts.md` (Junior/Middle).
- Thành phần/đóng góp (pie/donut/treemap) – xem `apexcharts.md`.
- Biến động/range (rangeBar/boxPlot/candlestick) – xem `apexcharts.md`.
- Live update/realtime (debounce/throttle, tắt animation) – xem `Advanced-Patterns.md`.
- Enterprise integration (SSR/SSG, observability, migration) – xem `Principal-Level-Patterns.md`.

