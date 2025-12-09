# ApexCharts + React – Hướng dẫn toàn diện

## Mục lục
- Giới thiệu tổng quan
- Junior Level – Cơ bản
- Middle Level – Trung cấp
- Senior Level – Nâng cao (mở đầu)
- Tài liệu tham khảo

## Giới thiệu tổng quan
- ApexCharts là thư viện chart dựa trên JS với API cấu hình, hỗ trợ nhiều loại biểu đồ và animation.
- Lý do dùng: cú pháp cấu hình đơn giản, nhiều chart type, responsive tốt, hỗ trợ wrapper React (`react-apexcharts`).
- Khái niệm cốt lõi:
  - `ReactApexChart`: component React để render chart.
  - `options`: cấu hình hiển thị (chart, axes, plotOptions, theme, responsive...).
  - `series`: dữ liệu; cấu trúc thay đổi theo `type` (ví dụ line/area dùng array số, candlestick/rangeBar dùng mảng 4 giá trị).

## Junior Level – Cơ bản
### Cài đặt & cấu hình
- Cài: `npm install apexcharts react-apexcharts`
- Import: `import ReactApexChart from 'react-apexcharts'`
- Tối thiểu:
  - Giữ `options` trong `useMemo` hoặc biến hằng để tránh re-render.
  - `series` đặt trong state để dễ cập nhật.

### Hello World (Line)
```tsx
import { useMemo, useState } from 'react';
import ReactApexChart from 'react-apexcharts';

export default function SimpleLine() {
  const options = useMemo(() => ({
    chart: { id: 'basic-line', toolbar: { show: false } },
    xaxis: { categories: ['Jan', 'Feb', 'Mar', 'Apr'] },
    stroke: { curve: 'smooth' }
  }), []);

  const [series] = useState([{ name: 'Revenue', data: [30, 40, 25, 50] }]);

  return <ReactApexChart options={options} series={series} type="line" height={320} />;
}
```

### 3–5 use case phổ biến
- Dashboard KPI (line/area/bar).
- Phân bố & thành phần (pie/donut/treemap).
- So sánh range/biến động (rangeBar, boxPlot).
- Chuỗi thời gian (datetime line/area, candlestick).
- Heatmap cho ma trận giá trị.

### Pitfalls (❌/✅)
- ❌ Truyền object `options` inline → re-render liên tục.  
  ✅ Dùng `useMemo` hoặc hằng.
- ❌ Không set `height/width` → layout khó đoán.  
  ✅ Chỉ định `height` rõ ràng.
- ❌ Dữ liệu datetime không chuyển `Date`/timestamp → trục sai định dạng.  
  ✅ Dùng timestamp/`new Date(...)` và `xaxis.type = 'datetime'`.
- ❌ State `series` mutate trực tiếp.  
  ✅ Tạo mảng mới khi cập nhật.

### Best practices cơ bản (5–10)
- Giữ `options` ổn định (memo), tách cấu hình chung ra ngoài component.
- Đặt `chart.id` cố định để hỗ trợ brush/selection giữa nhiều chart.
- Bật `toolbar` hoặc tắt tuỳ UI; ẩn nếu dùng trong card nhỏ.
- Dùng `responsive` để giảm width/legend ở breakpoint nhỏ.
- Với dark mode, dùng `theme: { mode: 'dark' }`.
- Tránh dữ liệu quá dài trong 1 series; cân nhắc downsample/throttle ở UI.
- Kiểm soát `animations` (tắt hoặc giảm) khi dữ liệu cập nhật liên tục.

## Middle Level – Trung cấp
### Pagination / Large data
- Trục datetime: chỉ render window cần thiết, phân trang/virtualize ở API, truyền slice vào chart.
- Dùng `dataLabels.enabled = false` khi dữ liệu dày; dùng tooltip thay vì label.

### Live update / streaming
- Cập nhật `series` qua state bất biến; cân nhắc debounce/throttle nếu source đẩy quá nhanh.
- Tắt animation hoặc giảm `animations.speed` để tránh lag.

### Event handling & interaction
- Lắng nghe sự kiện qua `chart.events` (ví dụ `dataPointSelection`, `click`) để sync với UI khác.
- Dùng `brush/selection` giữa 2 chart (ví dụ mini-map + main chart) bằng `chart.id` & `brush.target`.

### Error handling
- Bao quanh component bằng boundary đơn giản; khi dữ liệu rỗng, hiển thị state trống thay vì chart.
- Validate shape `series` trước khi render (đặc biệt với range/candlestick cần 4 giá trị).

## Senior Level – Nâng cao (mở đầu)
- Tối ưu performance:
  - Memo `options`; limit re-render bằng `React.memo`.
  - Tắt/giảm `dataLabels`, `markers` khi series lớn.
  - Giảm số điểm hiển thị (downsample) ở client, hoặc server-side aggregation.
- Tích hợp state management:
  - Đặt chart config/data trong custom hook (per domain) để tái sử dụng.
  - Chuẩn hoá schema series theo domain (e.g. `[{ x: timestamp, y: value }]`).
- Monitoring/logging:
  - Wrap chart render với logging khi nhận dữ liệu bất thường (NaN, null).

## Tài liệu tham khảo
- Official React docs: https://apexcharts.com/docs/react-charts/
- React samples: https://github.com/apexcharts/apexcharts.js/tree/main/samples/react
- Blog tổng quan chart libs (2025): https://blog.logrocket.com/best-react-chart-libraries-2025/

