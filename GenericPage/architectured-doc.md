# Software Architecture Document (arc42) — GenericPage Pattern

## 1. Introduction and Goals
- **Purpose**: Chuẩn hóa cách xây CRUD pages bằng cấu hình (GenericPage + GenericTable + GenericForm) để giảm lặp code, nhất quán UI/UX.
- **Goals**: DRY, type-safe (TS generics), dễ mở rộng (custom actions/filters), dễ bảo trì (config tách biệt), hỗ trợ pagination/sort/filter và CRUD.
- **Stakeholders**: FE devs, Tech Leads; kỳ vọng giảm thời gian tạo page, dễ chỉnh sửa UI/logic.
- **Success criteria**: Tạo page CRUD mới chủ yếu bằng config; giảm ≥40-50% code lặp cho list/form; hỗ trợ custom actions/filters mà không fork core.

## 2. Architecture Constraints
- Tech: React + Ant Design; TanStack Router/Query; TypeScript strict.
- Org: Tuân thủ design system Antd; route loader với search params; camelCase trên FE.
- Time/Cost: Ưu tiên ship nhanh, hạn chế trừu tượng hóa quá mức; giữ backward compatibility với pages hiện tại.

## 3. System Scope and Context
- **Scope**: FE layer cho CRUD pages (list + create/update/delete).
- **In-scope**: GenericPage container, GenericTable, GenericForm, GenericFilterBar (tùy chọn), customActions, hooks useApi*.
- **Out-of-scope**: Backend APIs; thống kê phức tạp domain-specific (viết custom).
- **External deps**: Antd (Table, Form, message, Popconfirm), TanStack Router/Query, Axios qua BaseApiService.

## 4. Solution Strategy
- Pattern **Configuration-Driven** + **Composable**: Table/Form định nghĩa qua config; actions/filters tùy biến; có slot/prop để cắm UI custom.
- Dùng hooks useApi* (create/update/delete/list/paginated) + BaseApiService để chuẩn hóa gọi API và cache invalidation.
- Keep it simple: slots optional; cho phép wrapper render header/statistics ngoài GenericPage nếu không muốn slot.

## 5. Building Block View
- **GenericPage**: Container orchestrates Table + Form modal/drawer, CRUD mutations, invalidation.
- **GenericTable**: Hiển thị data, pagination/sort, rowSelection, customActions (table/row), optional Popconfirm delete; sử dụng usePaginationWithRouter.
- **GenericForm**: Render field theo type (text/number/select/date/password/textarea/…); customRender override.
- **GenericFilterBar (optional)**: Partial generic filter builder (search/select/dateRange/numberRange/checkbox), emit onFilterChange.
- **Config**: `GenericPageConfig` (entity metadata, apiService, table config, form create/update, features, customActions, optional slots).
- **API layer**: BaseApiService + userApiService/productApiService...; hooks useApiCreate/Update/Delete/Detail/List/Paginated.

### 5.1 File Structure (đề xuất)
```
src/
└── components/GenericCRUD/
    ├── GenericPage/
    │   ├── GenericPage.tsx
    │   ├── GenericPageConfig.ts
    │   └── index.ts
    ├── GenericTable/
    │   ├── GenericTable.tsx
    │   ├── GenericTableConfig.ts
    │   ├── components/
    │   │   ├── TableToolbar.tsx
    │   │   └── TableActions.tsx
    │   └── index.ts
    ├── GenericForm/
    │   ├── GenericForm.tsx
    │   ├── GenericFormConfig.ts
    │   ├── renderers/
    │   │   ├── TextField.tsx
    │   │   ├── NumberField.tsx
    │   │   ├── SelectField.tsx
    │   │   └── index.ts
    │   └── index.ts
    └── GenericFilterBar/ (optional)
        ├── GenericFilterBar.tsx
        └── types.ts
└── features/
    └── <entity>/
        ├── config/
        │   └── <entity>PageConfig.ts
        └── pages/
            └── <Entity>ListPage.tsx
```


## 6. Runtime View
- **Read**: URL search params → routeApi.useLoaderData/usePaginationWithRouter → apiService.getPaginated → render Table.
- **Write (Create/Update)**: Form submit → useApiCreate/Update → onSuccess: message/Modal.success + invalidate (query + router).
- **Delete**: customActions row/table + Popconfirm → useApiDelete → onSuccess message + invalidate.
- **Filters**: GenericFilterBar/onFilterChange → navigate/update search params → refetch loader/query.
- **Wrapper UI**: Header/Statistics có thể render ngoài GenericPage hoặc qua slot.

## 7. Deployment View
- FE SPA/Vite build; chạy trên browser; phụ thuộc backend API (HTTPS). Không có thay đổi hạ tầng đặc thù.

## 8. Cross-cutting Concepts
- **State**: Server state via TanStack Query + Router loader; search params trong URL.
- **Feedback**: Antd `message` hoặc `Modal.success/error` cho kết quả CRUD; Popconfirm cho xác nhận xóa.
- **Error handling**: unwrapResponse ném lỗi; mutation onError hiển thị message.
- **Extensibility**: customActions, customRender (Form), optional slots (header/statistics/filters/toolbar/table/footer) hoặc wrapper ngoài.
- **Validation**: Antd Form rules; custom validator; password optional trong update.
- **Styling/UX**: Antd; nhất quán spacing/card/table/form; pagination/sort mặc định.

## 9. Architecture Decisions (ADR-lite)
- **Config-driven GenericPage**: Chọn để giảm lặp CRUD, giữ type-safe.
- **Partial generic filters**: builder tùy chọn, tránh over-abstract; filters phức tạp có thể viết custom.
- **Slots optional**: Cho phép cắm Header/Statistics/Filters; nhưng không bắt buộc, có thể render ngoài wrapper.
- **Use Antd message thay Notification Modal**: đơn giản, không cần modal riêng; Popconfirm cho xác nhận xóa đơn giản.

## 10. Quality Requirements
- **Maintainability**: Config rõ ràng, generic hooks; slot/prop tối giản.
- **Reusability**: Table/Form/CRUD reuse đa entity; filters/actions reuse một phần.
- **Performance**: Pagination server-side; cache query; invalidate chọn lọc.
- **Usability**: Phản hồi tức thì (message), confirm xóa an toàn (Popconfirm).

## 11. Risks and Technical Debts
- Over-abstraction nếu lạm dụng slots/builder → theo dõi và giữ optional.
- Phức tạp filters đặc thù → cho phép viết custom ngoài GenericFilterBar.
- Đồng bộ URL params và table state → cần test navigate/back.
- Migration từ pages cũ: cần pilot 1-2 page trước khi rollout.

## 12. Glossary
- **GenericPage**: Container CRUD cấu hình.
- **GenericTable**: Bảng dữ liệu + pagination/sort/actions.
- **GenericForm**: Form dynamic từ config field type.
- **GenericFilterBar**: Builder filter tùy chọn (partial generic).
- **customActions**: Hành động tùy biến (row/table/both).
- **slots**: Điểm cắm UI tùy biến (header/statistics/filters/toolbar/table/footer).

