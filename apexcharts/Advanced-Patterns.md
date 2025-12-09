# ApexCharts + React – Advanced Patterns (Senior)

## Cấu hình nâng cao
- Global config: tạo factory `createBaseOptions()` và merge per-chart; tránh lặp lại theme, grid, tooltip.
- Query key factory: nếu dùng React Query/Loader, tạo hàm `chartKeys(domain, params)` để quản lý cache/prefetch.
- Custom client: gói `ReactApexChart` thành component `ChartBase` nhận `options`, `series`, `type`, áp dụng theme responsive mặc định.

## Performance optimization
- Giảm render:
  - Memo `options` và bọc `ReactApexChart` bằng `React.memo`.
  - Tắt `dataLabels`/`markers` khi >2k điểm; dùng tooltip.
  - Với realtime: tắt animation (`animations.enabled=false`) hoặc giảm `speed`.
- Downsample/aggregate:
  - Thực hiện ở API; hoặc client-side lược bớt điểm (e.g. chỉ giữ mỗi N điểm).
  - Với heatmap/treemap lớn, gom nhóm data ở server.
- Select optimization:
  - Với mảng object lớn, tạo `select`/mapper trước khi set state để tránh compute trong render.

## Custom hooks architecture
- `useApexChartOptions(baseOptions, deps)`: trả về options đã memo.
- `useLineChart(data, categories)`: chuẩn hoá series, trả options/series sẵn dùng.
- `useBrushChart(mainId, brushId, series)`: trả options cho cặp chart có `brush.target`.

## Prefetching & initial data
- Khi biết route trước, prefetch data và set `options.series` với `initialData` để tránh flash rỗng.
- Với SSR/SSG (Next/Remix), serialize data dạng { x, y } và bật `xaxis.type='datetime'`.

## Testing nâng cao
- Unit hook: kiểm tra hook tạo options/series (snapshot object).
- Integration: dùng MSW mock API, render component, assert số điểm (DOM) và legend/tooltip toggle.
- Interaction: giả lập click `dataPointSelection` và kiểm tra callback được gọi với đúng payload.

## Error & resiliency
- Bọc chart bằng boundary và fallback UI.
- Kiểm tra NaN/null trước khi set state; strip giá trị bất hợp lệ để tránh chart lỗi silent.

