# Plan Implementation Generic Page Pattern

## Mục Lục

1. [Phân Tích Hiện Trạng](#1-phân-tích-hiện-trạng)
2. [Kiến Trúc GenericPage](#2-kiến-trúc-genericpage)
3. [Implementation Steps](#3-implementation-steps)
4. [Code Implementation Chi Tiết](#4-code-implementation-chi-tiết)
5. [Migration Path](#5-migration-path)
6. [Cấu Trúc Dự Án Sau Khi Triển Khai](#6-cấu-trúc-dự-án-sau-khi-triển-khai)
7. [Testing Strategy](#7-testing-strategy)
8. [Risks và Mitigation](#8-risks-và-mitigation)

---

## 1. Phân Tích Hiện Trạng

### 1.1. Codebase Hiện Tại

**Cấu trúc UserManagementPage hiện tại:**
```
features/users/
├── pages/
│   └── UserManagementPage.tsx      # Container component
├── hooks/
│   ├── useUserManagementPage.ts    # Business logic (262 lines)
│   └── useUsers.ts                 # API hooks wrapper
├── components/
│   ├── UserHeader.tsx              # Header với title + button
│   ├── UserStatistics.tsx         # Statistics cards
│   ├── UserSearchFilter.tsx       # Search, filter, sort
│   ├── UserTable.tsx              # Table component
│   ├── UserModals.tsx             # All modals
│   └── UserForm.tsx               # Form component
├── api/
│   └── UserApiService.ts          # Extends BaseApiService
└── types/
    ├── entity.ts
    └── api.ts
```

**Điểm mạnh:**
- ✅ Đã có BaseApiService với CRUD methods chuẩn
- ✅ Đã có useApi hooks (useApiCreate, useApiUpdate, useApiDelete, useApiPaginated)
- ✅ Đã có usePaginationWithRouter hook cho URL sync
- ✅ TanStack Router với routeApi và loader pattern
- ✅ TypeScript với type safety tốt

**Vấn đề cần giải quyết:**
- ❌ Code lặp lại giữa các entity pages (User, Product, Supplier, etc.)
- ❌ Mỗi entity phải viết lại Table, Form, Modals, SearchFilter
- ❌ Logic CRUD giống nhau nhưng phải copy-paste
- ❌ Khó maintain khi cần thay đổi UI/UX chung

### 1.2. Mapping với Tài Liệu GenericPage

**Tài liệu GenericPage đề xuất:**
- Configuration-Driven Pattern
- GenericPage container component
- GenericTable component
- GenericForm component
- GenericFilterBar (optional)
- Slot pattern cho customization

**Phù hợp với codebase:**
- ✅ BaseApiService đã có sẵn → phù hợp với `apiService` trong config
- ✅ usePaginationWithRouter đã có → phù hợp với GenericTable
- ✅ useApi hooks đã có → phù hợp với GenericPage mutations
- ✅ TanStack Router pattern → phù hợp với routeApi prop

---

## 2. Kiến Trúc GenericPage

### 2.1. Component Hierarchy

```
GenericPage (Container)
├── Slots (Optional)
│   ├── header: ReactNode
│   ├── statistics: ReactNode
│   ├── filters: ReactNode
│   ├── toolbar: ReactNode
│   └── footer: ReactNode
├── GenericTable
│   ├── TableToolbar (Search, Refresh, Custom Actions)
│   ├── Ant Design Table
│   └── Row Actions (Edit, Delete, Custom)
└── GenericForm (Modal/Drawer)
    ├── Form Fields (từ config)
    └── Submit/Cancel buttons
```

### 2.2. Config-Driven Architecture

```typescript
// Config object định nghĩa toàn bộ behavior
const userPageConfig: GenericPageConfig<UserEntity, CreateUserRequest, UpdateUserRequest> = {
  entity: { name: 'users', displayName: 'User', displayNamePlural: 'Users' },
  apiService: userApiService,
  table: { columns: [...], ... },
  form: { create: {...}, update: {...} },
  features: { enableCreate: true, ... },
  customActions: [...],
}

// Sử dụng
<GenericPage config={userPageConfig} routeApi={routeApi} />
```

### 2.3. Integration với Codebase Hiện Tại

**Tận dụng:**
- `BaseApiService<TData, TCreate, TUpdate>` → `config.apiService`
- `usePaginationWithRouter` → trong GenericTable
- `useApiCreate/Update/Delete` → trong GenericPage
- `TanStack Router routeApi` → prop của GenericPage
- `Ant Design` components → UI layer
- **Param mapping**: toàn bộ query params gửi lên backend phải dùng PascalCase (`Page`, `PageSize`, `StartDate`, `EndDate`, ...), kể cả các endpoint reports như `top-products` và `top-customers`; FE giữ camelCase nội bộ và convert khi call API.

**Mở rộng:**
- GenericTable component mới
- GenericForm component mới
- GenericPageConfig types
- Slot pattern cho customization

---

## 3. Implementation Steps

### Phase 1: Core Infrastructure (Tuần 1)

**Step 1.1: Tạo Config Types**
- Tạo `GenericPageConfig.ts` với đầy đủ type definitions
- Đảm bảo type safety với TypeScript generics

**Step 1.2: Tạo GenericForm Component**
- Implement GenericForm với field renderers
- Hỗ trợ các field types: text, number, select, date, textarea, password, email
- Custom renderer support

**Step 1.3: Tạo GenericTable Component**
- Implement GenericTable với Ant Design Table
- Tích hợp usePaginationWithRouter
- Row actions (Edit, Delete, Custom)
- Table toolbar (Search, Refresh, Create button)

**Step 1.4: Tạo GenericPage Container**
- Orchestrate Table + Form
- Manage modal/drawer state
- Handle CRUD mutations
- Slot pattern implementation

### Phase 2: User Page Migration (Tuần 2)

**Step 2.1: Tạo UserPageConfig**
- Convert UserTable columns → config
- Convert UserForm fields → config
- Define custom actions nếu có

**Step 2.2: Migrate UserManagementPage**
- Thay thế bằng GenericPage
- Test toàn bộ CRUD operations
- Verify URL sync, pagination, sort

**Step 2.3: Refactor Components (Optional)**
- Giữ UserHeader, UserStatistics như slots
- Hoặc migrate sang GenericPage slots

### Phase 3: Extend to Other Entities (Tuần 3-4)

**Step 3.1: Product Page**
- Tạo ProductPageConfig
- Migrate ProductManagementPage

**Step 3.2: Supplier Page**
- Tạo SupplierPageConfig
- Migrate SupplierManagementPage

**Step 3.3: Other Entities**
- Categories, Customers, Orders, etc.

### Phase 4: Advanced Features (Tuần 5+)

**Step 4.1: GenericFilterBar (Optional)**
- Implement partial generic filter builder
- Support: search, select, dateRange, numberRange

**Step 4.2: Bulk Actions**
- Row selection
- Bulk delete
- Bulk export

**Step 4.3: Import/Export**
- Excel import/export
- Custom actions

---

## 4. Code Implementation Chi Tiết

### 4.1. GenericPageConfig Types

**File:** `src/components/GenericCRUD/GenericPage/GenericPageConfig.ts`

```typescript
import type { BaseApiService } from '@/lib/api/base/BaseApiService';
import type { ColumnType } from 'antd/es/table';
import type { FormInstance, Rule } from 'antd/es/form';

// ==================== Core Config Type ====================

export interface GenericPageConfig<TData, TCreate, TUpdate> {
  // REQUIRED - Entity metadata
  entity: {
    name: string;                      // 'users', 'products', 'suppliers'
    displayName: string;               // 'User', 'Product', 'Supplier'
    displayNamePlural: string;         // 'Users', 'Products', 'Suppliers'
  };

  // REQUIRED - API Service
  apiService: BaseApiService<TData, TCreate, TUpdate>;

  // REQUIRED - Table configuration
  table: TableConfig<TData>;

  // REQUIRED - Form configuration
  form: {
    create: FormConfig<TCreate>;
    update: FormConfig<TUpdate>;
  };

  // OPTIONAL - Feature flags
  features?: {
    enableCreate?: boolean;            // Default: true
    enableEdit?: boolean;              // Default: true
    enableDelete?: boolean;            // Default: true
    enableBulkDelete?: boolean;        // Default: false
    enableExport?: boolean;            // Default: false
    enableImport?: boolean;            // Default: false
  };

  // OPTIONAL - Customization
  customActions?: CustomAction<TData>[];
  customFilters?: FilterConfig[];
  onBeforeCreate?: (data: TCreate) => Promise<TCreate | void>;
  onAfterCreate?: (result: TData) => void;
  onBeforeUpdate?: (id: number | string, data: TUpdate) => Promise<TUpdate | void>;
  onAfterUpdate?: (result: TData) => void;
  onBeforeDelete?: (id: number | string) => Promise<boolean>;
  onAfterDelete?: () => void;

  // OPTIONAL - Slots
  slots?: {
    header?: React.ReactNode;
    statistics?: React.ReactNode;
    filters?: React.ReactNode;
    toolbar?: React.ReactNode;
    footer?: React.ReactNode;
  };
}

// ==================== Table Config ====================

export interface TableConfig<TData> {
  columns: GenericColumnType<TData>[];
  rowKey?: keyof TData | ((record: TData) => string | number);
  defaultSortBy?: string;
  defaultSortDesc?: boolean;
  defaultPageSize?: number;
  pageSizeOptions?: number[];
  rowSelection?: {
    enabled: boolean;
    onChange?: (selectedRowKeys: React.Key[], selectedRows: TData[]) => void;
  };
  onRow?: (record: TData) => React.HTMLAttributes<HTMLTableRowElement>;
}

export interface GenericColumnType<TData> extends Omit<ColumnType<TData>, 'render'> {
  title: string;
  dataIndex?: keyof TData;
  key?: string;
  width?: number | string;
  fixed?: 'left' | 'right';
  sorter?: boolean;
  render?: (value: any, record: TData, index: number) => React.ReactNode;
  format?: {
    type: 'date' | 'datetime' | 'currency' | 'number' | 'boolean' | 'custom';
    options?: any;
  };
}

// ==================== Form Config ====================

export interface FormConfig<TFormData> {
  fields: FormFieldConfig<TFormData>[];
  layout?: 'horizontal' | 'vertical' | 'inline';
  labelCol?: { span: number };
  wrapperCol?: { span: number };
  initialValues?: Partial<TFormData>;
  validateTrigger?: 'onChange' | 'onBlur' | 'onSubmit';
  customRender?: (form: FormInstance, fields: React.ReactNode) => React.ReactNode;
}

export interface FormFieldConfig<TFormData> {
  name: keyof TFormData;
  label: string;
  required?: boolean;
  type: 'text' | 'number' | 'select' | 'date' | 'datetime' | 'textarea' | 'password' | 'email' | 'phone' | 'custom';
  rules?: Rule[];
  config?: FieldTypeConfig;
  visible?: boolean | ((formData: Partial<TFormData>) => boolean);
  disabled?: boolean | ((formData: Partial<TFormData>) => boolean);
  render?: (fieldConfig: FormFieldConfig<TFormData>, form: FormInstance) => React.ReactNode;
}

export type FieldTypeConfig =
  | TextFieldConfig
  | NumberFieldConfig
  | SelectFieldConfig
  | DateFieldConfig
  | TextAreaFieldConfig;

export interface TextFieldConfig {
  placeholder?: string;
  maxLength?: number;
  prefix?: React.ReactNode;
  suffix?: React.ReactNode;
}

export interface NumberFieldConfig {
  placeholder?: string;
  min?: number;
  max?: number;
  step?: number;
  precision?: number;
  formatter?: (value: number | undefined) => string;
  parser?: (value: string | undefined) => number;
}

export interface SelectFieldConfig {
  placeholder?: string;
  options: SelectOption[];
  mode?: 'multiple' | 'tags';
  showSearch?: boolean;
  allowClear?: boolean;
  fetchOptions?: () => Promise<SelectOption[]>;
  dependsOn?: string;
  getOptionsWhen?: (dependentValue: any) => Promise<SelectOption[]>;
}

export interface SelectOption {
  label: string;
  value: string | number;
  disabled?: boolean;
}

export interface DateFieldConfig {
  placeholder?: string;
  format?: string;
  showTime?: boolean;
  disabledDate?: (current: moment.Moment) => boolean;
}

export interface TextAreaFieldConfig {
  placeholder?: string;
  rows?: number;
  maxLength?: number;
  showCount?: boolean;
}

// ==================== Custom Actions ====================

export interface CustomAction<TData> {
  key: string;
  label: string;
  icon?: React.ReactNode;
  type?: 'primary' | 'default' | 'dashed' | 'link' | 'text';
  danger?: boolean;
  scope: 'table' | 'row' | 'both';
  visible?: boolean | ((record?: TData) => boolean);
  disabled?: boolean | ((record?: TData) => boolean);
  onClick: (record?: TData, selectedRows?: TData[]) => void | Promise<void>;
  confirm?: {
    title: string;
    content?: string;
    okText?: string;
    cancelText?: string;
  };
}

// ==================== Filter Config ====================

export interface FilterConfig {
  name: string;
  label: string;
  type: 'text' | 'select' | 'dateRange' | 'numberRange';
  config?: any;
}
```

### 4.2. GenericForm Component

**File:** `src/components/GenericCRUD/GenericForm/GenericForm.tsx`

```typescript
import { Form, Input, InputNumber, Select, DatePicker, Button, Space } from 'antd';
import type { FormInstance } from 'antd/es/form';
import type { FormConfig, FormFieldConfig, FieldTypeConfig } from '../GenericPage/GenericPageConfig';
import { useState, useEffect } from 'react';

interface GenericFormProps<TFormData> {
  config: FormConfig<TFormData>;
  onSubmit: (values: TFormData) => void | Promise<void>;
  onCancel: () => void;
  initialValues?: Partial<TFormData>;
  submitButtonText?: string;
  loading?: boolean;
}

export function GenericForm<TFormData>({
  config,
  onSubmit,
  onCancel,
  initialValues,
  submitButtonText = 'Submit',
  loading = false,
}: GenericFormProps<TFormData>) {
  const [form] = Form.useForm();

  // Sync initialValues với form
  useEffect(() => {
    if (initialValues) {
      form.setFieldsValue(initialValues);
    } else if (config.initialValues) {
      form.setFieldsValue(config.initialValues);
    }
  }, [initialValues, config.initialValues, form]);

  const renderField = (fieldConfig: FormFieldConfig<TFormData>) => {
    // Custom render override
    if (fieldConfig.render) {
      return fieldConfig.render(fieldConfig, form);
    }

    // Default renderers based on type
    switch (fieldConfig.type) {
      case 'text':
      case 'email':
      case 'phone':
        return (
          <Input
            placeholder={fieldConfig.config?.placeholder}
            maxLength={fieldConfig.config?.maxLength}
            prefix={fieldConfig.config?.prefix}
            suffix={fieldConfig.config?.suffix}
            type={fieldConfig.type === 'email' ? 'email' : fieldConfig.type === 'phone' ? 'tel' : 'text'}
          />
        );

      case 'password':
        return (
          <Input.Password
            placeholder={fieldConfig.config?.placeholder}
            maxLength={fieldConfig.config?.maxLength}
          />
        );

      case 'number':
        const numberConfig = fieldConfig.config as NumberFieldConfig | undefined;
        return (
          <InputNumber
            placeholder={numberConfig?.placeholder}
            min={numberConfig?.min}
            max={numberConfig?.max}
            step={numberConfig?.step}
            precision={numberConfig?.precision}
            formatter={numberConfig?.formatter}
            parser={numberConfig?.parser}
            style={{ width: '100%' }}
          />
        );

      case 'select':
        const selectConfig = fieldConfig.config as SelectFieldConfig | undefined;
        const [options, setOptions] = useState<SelectOption[]>(selectConfig?.options || []);
        const [loadingOptions, setLoadingOptions] = useState(false);

        // Fetch options nếu có fetchOptions
        useEffect(() => {
          if (selectConfig?.fetchOptions) {
            setLoadingOptions(true);
            selectConfig.fetchOptions().then((opts) => {
              setOptions(opts);
              setLoadingOptions(false);
            });
          }
        }, []);

        // Handle dependent selects
        const dependsOnValue = selectConfig?.dependsOn
          ? form.getFieldValue(selectConfig.dependsOn)
          : undefined;

        useEffect(() => {
          if (selectConfig?.dependsOn && selectConfig?.getOptionsWhen && dependsOnValue) {
            setLoadingOptions(true);
            selectConfig.getOptionsWhen(dependsOnValue).then((opts) => {
              setOptions(opts);
              setLoadingOptions(false);
            });
          }
        }, [dependsOnValue]);

        return (
          <Select
            placeholder={selectConfig?.placeholder}
            options={options}
            mode={selectConfig?.mode}
            showSearch={selectConfig?.showSearch}
            allowClear={selectConfig?.allowClear}
            loading={loadingOptions}
          />
        );

      case 'date':
      case 'datetime':
        const dateConfig = fieldConfig.config as DateFieldConfig | undefined;
        return (
          <DatePicker
            placeholder={dateConfig?.placeholder}
            format={dateConfig?.format}
            showTime={fieldConfig.type === 'datetime' || dateConfig?.showTime}
            disabledDate={dateConfig?.disabledDate}
            style={{ width: '100%' }}
          />
        );

      case 'textarea':
        const textareaConfig = fieldConfig.config as TextAreaFieldConfig | undefined;
        return (
          <Input.TextArea
            placeholder={textareaConfig?.placeholder}
            rows={textareaConfig?.rows}
            maxLength={textareaConfig?.maxLength}
            showCount={textareaConfig?.showCount}
          />
        );

      default:
        return <Input placeholder={fieldConfig.config?.placeholder} />;
    }
  };

  const fields = config.fields.map((fieldConfig) => {
    // Check visibility
    const formValues = form.getFieldsValue();
    const visible =
      typeof fieldConfig.visible === 'function'
        ? fieldConfig.visible(formValues)
        : fieldConfig.visible !== false;

    if (!visible) return null;

    const disabled =
      typeof fieldConfig.disabled === 'function'
        ? fieldConfig.disabled(formValues)
        : fieldConfig.disabled;

    const fieldElement = renderField(fieldConfig);
    if (!fieldElement) return null;

    return (
      <Form.Item
        key={String(fieldConfig.name)}
        name={fieldConfig.name as string}
        label={fieldConfig.label}
        rules={fieldConfig.rules}
        required={fieldConfig.required}
      >
        {React.cloneElement(fieldElement as React.ReactElement, { disabled })}
      </Form.Item>
    );
  }).filter(Boolean);

  const formContent = config.customRender ? config.customRender(form, fields) : fields;

  return (
    <Form
      form={form}
      layout={config.layout || 'vertical'}
      labelCol={config.labelCol}
      wrapperCol={config.wrapperCol}
      initialValues={initialValues || config.initialValues}
      onFinish={onSubmit}
      validateTrigger={config.validateTrigger || 'onSubmit'}
    >
      {formContent}

      <Form.Item>
        <Space>
          <Button type="primary" htmlType="submit" loading={loading}>
            {submitButtonText}
          </Button>
          <Button onClick={onCancel}>Cancel</Button>
        </Space>
      </Form.Item>
    </Form>
  );
}
```

### 4.3. GenericTable Component

**File:** `src/components/GenericCRUD/GenericTable/GenericTable.tsx`

```typescript
import { Table, Button, Space, Input, Popconfirm, message } from 'antd';
import { PlusOutlined, EditOutlined, DeleteOutlined, SearchOutlined, ReloadOutlined } from '@ant-design/icons';
import { useState } from 'react';
import { usePaginationWithRouter } from '@/hooks/usePaginationWithRouter';
import { useApiDelete } from '@/hooks/useApi';
import type { GenericPageConfig } from '../GenericPage/GenericPageConfig';
import type { TableConfig, GenericColumnType, CustomAction } from '../GenericPage/GenericPageConfig';

interface GenericTableProps<TData, TCreate, TUpdate> {
  config: GenericPageConfig<TData, TCreate, TUpdate>;
  routeApi: {
    useSearch: () => Record<string, unknown>;
  };
  onCreateClick?: () => void;
  onEditClick?: (record: TData) => void;
}

export function GenericTable<TData extends { id?: number | string }, TCreate, TUpdate>({
  config,
  routeApi,
  onCreateClick,
  onEditClick,
}: GenericTableProps<TData, TCreate, TUpdate>) {
  const { entity, apiService, table: tableConfig, features = {} } = config;
  
  // Default features
  const {
    enableCreate = true,
    enableEdit = true,
    enableDelete = true,
    enableBulkDelete = false,
  } = features;

  // Pagination với URL sync
  const {
    items,
    isLoading,
    params,
    totalCount,
    handlePageChange,
    handleSearch,
    handleSort,
    refetch,
  } = usePaginationWithRouter<TData>({
    apiService,
    entity: entity.name,
    routeApi,
  });

  // Delete mutation
  const deleteItem = useApiDelete({
    apiService,
    entity: entity.name,
    options: {
      onSuccess: () => {
        message.success(`${entity.displayName} deleted successfully!`);
        config.onAfterDelete?.();
      },
      onError: (error) => {
        message.error(`Failed to delete ${entity.displayName.toLowerCase()}: ${error.message}`);
      },
    },
  });

  // Selected rows (for bulk actions)
  const [selectedRowKeys, setSelectedRowKeys] = useState<React.Key[]>([]);
  const [selectedRows, setSelectedRows] = useState<TData[]>([]);

  // Build columns với Actions column
  const columns: GenericColumnType<TData>[] = [
    ...tableConfig.columns.map((col) => ({
      ...col,
      sorter: col.sorter,
      render: col.render || ((value: any) => {
        // Auto-format dựa trên type
        if (col.format) {
          switch (col.format.type) {
            case 'date':
              return value ? new Date(value).toLocaleDateString('vi-VN') : '-';
            case 'datetime':
              return value ? new Date(value).toLocaleString('vi-VN') : '-';
            case 'currency':
              return value != null ? `${value.toLocaleString('vi-VN')} đ` : '-';
            case 'number':
              return value != null ? value.toLocaleString('vi-VN') : '-';
            case 'boolean':
              return value ? 'Có' : 'Không';
            default:
              return value ?? '-';
          }
        }
        return value ?? '-';
      }),
    })),
    // Actions column
    {
      title: 'Thao tác',
      key: 'actions',
      fixed: 'right' as const,
      width: 150,
      render: (_: any, record: TData) => (
        <Space>
          {enableEdit && (
            <Button
              type="link"
              icon={<EditOutlined />}
              onClick={() => onEditClick?.(record)}
            >
              Sửa
            </Button>
          )}
          {enableDelete && (
            <Popconfirm
              title={`Xóa ${entity.displayName}?`}
              description="Hành động này không thể hoàn tác."
              onConfirm={async () => {
                const shouldDelete = config.onBeforeDelete
                  ? await config.onBeforeDelete(record.id!)
                  : true;
                if (shouldDelete) {
                  deleteItem.mutate(record.id!);
                }
              }}
              okText="Xóa"
              cancelText="Hủy"
              okButtonProps={{ danger: true }}
            >
              <Button
                type="link"
                danger
                icon={<DeleteOutlined />}
                loading={deleteItem.isPending}
              >
                Xóa
              </Button>
            </Popconfirm>
          )}
          {/* Custom row actions */}
          {config.customActions
            ?.filter((action) => action.scope === 'row' || action.scope === 'both')
            .filter((action) => 
              typeof action.visible === 'function' 
                ? action.visible(record) 
                : action.visible !== false
            )
            .map((action) => (
              <Button
                key={action.key}
                type={action.type || 'link'}
                danger={action.danger}
                icon={action.icon}
                disabled={
                  typeof action.disabled === 'function'
                    ? action.disabled(record)
                    : action.disabled
                }
                onClick={() => {
                  if (action.confirm) {
                    Popconfirm.confirm({
                      title: action.confirm.title,
                      content: action.confirm.content,
                      onOk: () => action.onClick(record),
                      okText: action.confirm.okText,
                      cancelText: action.confirm.cancelText,
                    });
                  } else {
                    action.onClick(record);
                  }
                }}
              >
                {action.label}
              </Button>
            ))}
        </Space>
      ),
    },
  ];

  // Toolbar
  const toolbar = (
    <Space style={{ marginBottom: 16, width: '100%', justifyContent: 'space-between' }}>
      <Space>
        <Input.Search
          placeholder={`Tìm kiếm ${entity.displayNamePlural.toLowerCase()}...`}
          allowClear
          enterButton={<SearchOutlined />}
          onSearch={handleSearch}
          defaultValue={params.search}
          style={{ width: 300 }}
        />
        <Button icon={<ReloadOutlined />} onClick={() => refetch()}>
          Làm mới
        </Button>
      </Space>
      
      <Space>
        {/* Table-level custom actions */}
        {config.customActions
          ?.filter((action) => action.scope === 'table' || action.scope === 'both')
          .map((action) => (
            <Button
              key={action.key}
              type={action.type}
              danger={action.danger}
              icon={action.icon}
              onClick={() => action.onClick(undefined, selectedRows)}
              disabled={
                typeof action.disabled === 'function'
                  ? action.disabled()
                  : action.disabled
              }
            >
              {action.label}
            </Button>
          ))}
        
        {enableCreate && (
          <Button
            type="primary"
            icon={<PlusOutlined />}
            onClick={onCreateClick}
          >
            Tạo {entity.displayName}
          </Button>
        )}
      </Space>
    </Space>
  );

  // Row selection config
  const rowSelection = enableBulkDelete || tableConfig.rowSelection?.enabled
    ? {
        selectedRowKeys,
        onChange: (keys: React.Key[], rows: TData[]) => {
          setSelectedRowKeys(keys);
          setSelectedRows(rows);
          tableConfig.rowSelection?.onChange?.(keys, rows);
        },
      }
    : undefined;

  return (
    <div>
      {toolbar}
      
      <Table<TData>
        columns={columns}
        dataSource={items}
        rowKey={
          typeof tableConfig.rowKey === 'function'
            ? tableConfig.rowKey
            : (record) => (record[tableConfig.rowKey as keyof TData] as string | number) || (record.id as string | number)
        }
        loading={isLoading}
        rowSelection={rowSelection}
        onRow={tableConfig.onRow}
        onChange={(pagination, _, sorter) => {
          if (pagination.current && pagination.pageSize) {
            handlePageChange(pagination.current, pagination.pageSize);
          }
          if (!Array.isArray(sorter) && sorter.field) {
            handleSort(String(sorter.field), sorter.order === 'descend');
          }
        }}
        pagination={{
          current: params.page,
          pageSize: params.pageSize,
          total: totalCount,
          showSizeChanger: true,
          showQuickJumper: true,
          showTotal: (total, range) =>
            `${range[0]}-${range[1]} của ${total} ${entity.displayNamePlural.toLowerCase()}`,
          pageSizeOptions: tableConfig.pageSizeOptions || ['10', '20', '50', '100'],
        }}
        scroll={{ x: 'max-content' }}
      />
    </div>
  );
}
```

### 4.4. GenericPage Container Component

**File:** `src/components/GenericCRUD/GenericPage/GenericPage.tsx`

```typescript
import { useState } from 'react';
import { Card, Modal, Drawer, message } from 'antd';
import { GenericTable } from '../GenericTable/GenericTable';
import { GenericForm } from '../GenericForm/GenericForm';
import { useApiCreate, useApiUpdate, useApiDetail } from '@/hooks/useApi';
import { useRouter } from '@tanstack/react-router';
import type { GenericPageConfig } from './GenericPageConfig';

interface GenericPageProps<TData, TCreate, TUpdate> {
  config: GenericPageConfig<TData, TCreate, TUpdate>;
  routeApi: {
    useSearch: () => Record<string, unknown>;
  };
  formMode?: 'modal' | 'drawer';
}

export function GenericPage<TData extends { id?: number | string }, TCreate, TUpdate>({
  config,
  routeApi,
  formMode = 'modal',
}: GenericPageProps<TData, TCreate, TUpdate>) {
  const { entity, apiService, form: formConfig, features = {} } = config;
  const router = useRouter();
  
  // Form state
  const [createModalVisible, setCreateModalVisible] = useState(false);
  const [editModalVisible, setEditModalVisible] = useState(false);
  const [editingId, setEditingId] = useState<number | string | null>(null);

  // API hooks
  const createItem = useApiCreate<TData, TCreate>({
    apiService,
    entity: entity.name,
    options: {
      onSuccess: (data) => {
        message.success(`${entity.displayName} đã được tạo thành công!`);
        setCreateModalVisible(false);
        config.onAfterCreate?.(data);
        router.invalidate();
      },
      onError: (error) => {
        message.error(`Không thể tạo ${entity.displayName.toLowerCase()}: ${error.message}`);
      },
    },
  });

  const updateItem = useApiUpdate<TData, TUpdate>({
    apiService,
    entity: entity.name,
    options: {
      onSuccess: (data) => {
        message.success(`${entity.displayName} đã được cập nhật thành công!`);
        setEditModalVisible(false);
        setEditingId(null);
        config.onAfterUpdate?.(data);
        router.invalidate();
      },
      onError: (error) => {
        message.error(`Không thể cập nhật ${entity.displayName.toLowerCase()}: ${error.message}`);
      },
    },
  });

  const { data: editingItem, isLoading: isLoadingEdit } = useApiDetail<TData>({
    apiService,
    entity: entity.name,
    id: editingId || undefined,
    options: {
      enabled: !!editingId,
    },
  });

  // Handlers
  const handleCreate = async (values: TCreate) => {
    const processedData = config.onBeforeCreate ? await config.onBeforeCreate(values) : values;
    if (processedData) {
      createItem.mutate(processedData as TCreate);
    }
  };

  const handleUpdate = async (values: TUpdate) => {
    if (!editingId) return;
    const processedData = config.onBeforeUpdate
      ? await config.onBeforeUpdate(editingId, values)
      : values;
    if (processedData) {
      updateItem.mutate({ id: editingId, data: processedData as TUpdate });
    }
  };

  const handleEditClick = (record: TData) => {
    setEditingId(record.id!);
    setEditModalVisible(true);
  };

  // Form container (Modal or Drawer)
  const FormContainer = formMode === 'modal' ? Modal : Drawer;
  const formContainerProps = formMode === 'modal'
    ? {
        open: createModalVisible || editModalVisible,
        onCancel: () => {
          setCreateModalVisible(false);
          setEditModalVisible(false);
          setEditingId(null);
        },
        footer: null,
        width: 600,
      }
    : {
        open: createModalVisible || editModalVisible,
        onClose: () => {
          setCreateModalVisible(false);
          setEditModalVisible(false);
          setEditingId(null);
        },
        width: 600,
      };

  return (
    <div style={{ padding: '24px', background: '#f5f5f5', minHeight: '100vh' }}>
      {/* Header Slot */}
      {config.slots?.header}

      {/* Statistics Slot */}
      {config.slots?.statistics}

      {/* Filters Slot */}
      {config.slots?.filters}

      {/* Main Content */}
      <Card
        title={entity.displayNamePlural}
        style={{
          borderRadius: '12px',
          boxShadow: '0 4px 12px rgba(0, 0, 0, 0.1)',
          border: 'none',
        }}
      >
        {/* Toolbar Slot */}
        {config.slots?.toolbar}

        <GenericTable
          config={config}
          routeApi={routeApi}
          onCreateClick={() => setCreateModalVisible(true)}
          onEditClick={handleEditClick}
        />
      </Card>

      {/* Footer Slot */}
      {config.slots?.footer}

      {/* Create Form */}
      {createModalVisible && (
        <FormContainer
          {...formContainerProps}
          title={`Tạo ${entity.displayName}`}
        >
          <GenericForm
            config={formConfig.create}
            onSubmit={handleCreate}
            onCancel={() => setCreateModalVisible(false)}
            submitButtonText="Tạo"
            loading={createItem.isPending}
          />
        </FormContainer>
      )}

      {/* Edit Form */}
      {editModalVisible && (
        <FormContainer
          {...formContainerProps}
          title={`Chỉnh sửa ${entity.displayName}`}
        >
          {isLoadingEdit ? (
            <div>Đang tải...</div>
          ) : (
            <GenericForm
              config={formConfig.update}
              onSubmit={handleUpdate}
              onCancel={() => {
                setEditModalVisible(false);
                setEditingId(null);
              }}
              initialValues={editingItem}
              submitButtonText="Cập nhật"
              loading={updateItem.isPending}
            />
          )}
        </FormContainer>
      )}
    </div>
  );
}
```

### 4.5. UserPageConfig Example

**File:** `src/features/users/config/userPageConfig.ts`

```typescript
import { userApiService } from '../api/UserApiService';
import type { GenericPageConfig } from '@/components/GenericCRUD/GenericPage/GenericPageConfig';
import type { UserEntity } from '../types/entity';
import type { CreateUserRequest, UpdateUserRequest } from '../types/api';
import { UserHeader } from '../components/UserHeader';
import { UserStatistics } from '../components/UserStatistics';

export const userPageConfig: GenericPageConfig<UserEntity, CreateUserRequest, UpdateUserRequest> = {
  // Entity metadata
  entity: {
    name: 'users',
    displayName: 'Người dùng',
    displayNamePlural: 'Người dùng',
  },

  // API Service
  apiService: userApiService,

  // Table configuration
  table: {
    columns: [
      {
        title: 'Người dùng',
        key: 'user',
        render: (_, record: UserEntity) => (
          <Space>
            <Avatar
              size="large"
              icon={<UserOutlined />}
              style={{
                backgroundColor: record.role === 0 ? '#faad14' : '#52c41a',
              }}
            />
            <div>
              <div style={{ fontWeight: 500 }}>{record.fullName}</div>
              <Text type="secondary">@{record.username}</Text>
            </div>
          </Space>
        ),
      },
      {
        title: 'Vai trò',
        dataIndex: 'role',
        key: 'role',
        sorter: true,
        render: (role: number) => (
          <Tag color={role === 0 ? 'gold' : 'green'}>
            {role === 0 ? 'Admin' : 'Staff'}
          </Tag>
        ),
      },
      {
        title: 'Ngày tạo',
        dataIndex: 'createdAt',
        key: 'createdAt',
        sorter: true,
        format: {
          type: 'datetime',
        },
      },
    ],
    defaultSortBy: 'createdAt',
    defaultSortDesc: true,
    defaultPageSize: 10,
    pageSizeOptions: [10, 20, 50, 100],
  },

  // Form configuration
  form: {
    create: {
      fields: [
        {
          name: 'username',
          label: 'Username',
          type: 'text',
          required: true,
          rules: [
            { required: true, message: 'Vui lòng nhập username!' },
            { min: 3, message: 'Username phải có ít nhất 3 ký tự!' },
          ],
          config: {
            placeholder: 'Nhập username',
            maxLength: 50,
          },
        },
        {
          name: 'password',
          label: 'Password',
          type: 'password',
          required: true,
          rules: [
            { required: true, message: 'Vui lòng nhập password!' },
            { min: 6, message: 'Password phải có ít nhất 6 ký tự!' },
          ],
          config: {
            placeholder: 'Nhập mật khẩu',
          },
        },
        {
          name: 'fullName',
          label: 'Họ và tên',
          type: 'text',
          required: true,
          rules: [{ required: true, message: 'Vui lòng nhập họ tên!' }],
          config: {
            placeholder: 'Nhập họ và tên',
            maxLength: 100,
          },
        },
        {
          name: 'role',
          label: 'Vai trò',
          type: 'select',
          required: true,
          rules: [{ required: true, message: 'Vui lòng chọn vai trò!' }],
          config: {
            placeholder: 'Chọn vai trò',
            options: [
              { label: 'Admin', value: 0 },
              { label: 'Staff', value: 1 },
            ],
          },
        },
      ],
      layout: 'vertical',
    },
    update: {
      fields: [
        {
          name: 'username',
          label: 'Username',
          type: 'text',
          required: true,
          disabled: true, // Username không thể đổi
          rules: [
            { required: true, message: 'Vui lòng nhập username!' },
            { min: 3, message: 'Username phải có ít nhất 3 ký tự!' },
          ],
          config: {
            placeholder: 'Nhập username',
            maxLength: 50,
          },
        },
        {
          name: 'password',
          label: 'Password',
          type: 'password',
          required: false,
          rules: [
            {
              validator(_, value) {
                if (!value || value.length === 0) {
                  return Promise.resolve();
                }
                if (value.length < 6) {
                  return Promise.reject(new Error('Password phải có ít nhất 6 ký tự!'));
                }
                return Promise.resolve();
              },
            },
          ],
          config: {
            placeholder: 'Để trống nếu không muốn đổi mật khẩu',
          },
        },
        {
          name: 'fullName',
          label: 'Họ và tên',
          type: 'text',
          required: true,
          rules: [{ required: true, message: 'Vui lòng nhập họ tên!' }],
          config: {
            placeholder: 'Nhập họ và tên',
            maxLength: 100,
          },
        },
        {
          name: 'role',
          label: 'Vai trò',
          type: 'select',
          required: true,
          rules: [{ required: true, message: 'Vui lòng chọn vai trò!' }],
          config: {
            placeholder: 'Chọn vai trò',
            options: [
              { label: 'Admin', value: 0 },
              { label: 'Staff', value: 1 },
            ],
          },
        },
      ],
      layout: 'vertical',
    },
  },

  // Features
  features: {
    enableCreate: true,
    enableEdit: true,
    enableDelete: true,
    enableBulkDelete: false,
  },

  // Slots - Tận dụng components hiện có
  slots: {
    header: <UserHeader onAddUser={() => {}} />, // onAddUser sẽ được override bởi GenericPage
    statistics: <UserStatistics totalUsers={0} adminCount={0} staffCount={0} />, // Sẽ cần tính toán từ data
  },
};
```

### 4.6. UserManagementPage Migration

**File:** `src/features/users/pages/UserManagementPage.tsx` (Sau migration)

```typescript
import { getRouteApi } from '@tanstack/react-router';
import { ENDPOINTS } from '@/app/routes/type/endpoint';
import { GenericPage } from '@/components/GenericCRUD/GenericPage/GenericPage';
import { userPageConfig } from '../config/userPageConfig';

const routeApi = getRouteApi(ENDPOINTS.ADMIN.USERS);

export function UserManagementPage() {
  return <GenericPage config={userPageConfig} routeApi={routeApi} formMode="modal" />;
}
```

**So sánh:**
- **Trước:** 107 lines (UserManagementPage) + 262 lines (useUserManagementPage) = 369 lines
- **Sau:** ~10 lines (UserManagementPage) + ~200 lines (userPageConfig) = 210 lines
- **Giảm:** ~43% code, logic tập trung, dễ maintain

---

## 5. Migration Path

### 5.1. Phase 1: Core Infrastructure (Tuần 1)

**Day 1-2: Config Types**
- [ ] Tạo `GenericPageConfig.ts` với đầy đủ types
- [ ] Test type safety với TypeScript

**Day 3-4: GenericForm**
- [ ] Implement GenericForm component
- [ ] Test với các field types
- [ ] Test conditional fields, dependent selects

**Day 5: GenericTable**
- [ ] Implement GenericTable component
- [ ] Tích hợp usePaginationWithRouter
- [ ] Test pagination, sort, search

### 5.2. Phase 2: User Page Migration (Tuần 2)

**Day 1-2: GenericPage Container**
- [ ] Implement GenericPage container
- [ ] Test CRUD operations
- [ ] Test modal/drawer modes

**Day 3-4: UserPageConfig**
- [ ] Convert UserTable columns → config
- [ ] Convert UserForm fields → config
- [ ] Test với UserManagementPage

**Day 5: Testing & Refinement**
- [ ] Test toàn bộ User CRUD flow
- [ ] Fix bugs nếu có
- [ ] Code review

### 5.3. Phase 3: Extend to Other Entities (Tuần 3-4)

**Week 3:**
- [ ] ProductPageConfig + migration
- [ ] SupplierPageConfig + migration

**Week 4:**
- [ ] CategoryPageConfig + migration
- [ ] CustomerPageConfig + migration
- [ ] Other entities

### 5.4. Phase 4: Advanced Features (Tuần 5+)

**Optional:**
- [ ] GenericFilterBar component
- [ ] Bulk actions
- [ ] Import/Export features

---

## 6. Cấu Trúc Dự Án Sau Khi Triển Khai

### 6.1. Directory Structure

```
src/
├── components/
│   └── GenericCRUD/
│       ├── GenericPage/
│       │   ├── GenericPage.tsx              # Main container
│       │   ├── GenericPageConfig.ts          # Config types
│       │   └── index.ts
│       ├── GenericTable/
│       │   ├── GenericTable.tsx              # Table component
│       │   ├── components/
│       │   │   └── TableToolbar.tsx         # Optional: Extract toolbar
│       │   └── index.ts
│       ├── GenericForm/
│       │   ├── GenericForm.tsx               # Form component
│       │   ├── renderers/
│       │   │   ├── TextField.tsx             # Optional: Extract renderers
│       │   │   ├── NumberField.tsx
│       │   │   ├── SelectField.tsx
│       │   │   └── index.ts
│       │   └── index.ts
│       ├── GenericFilterBar/                 # Optional
│       │   ├── GenericFilterBar.tsx
│       │   └── types.ts
│       └── index.ts
│
├── features/
│   ├── users/
│   │   ├── config/
│   │   │   └── userPageConfig.ts            # User-specific config
│   │   ├── pages/
│   │   │   └── UserManagementPage.tsx       # ~10 lines, dùng GenericPage
│   │   ├── components/                       # Optional: Custom components
│   │   │   ├── UserHeader.tsx               # Dùng như slot
│   │   │   └── UserStatistics.tsx           # Dùng như slot
│   │   ├── api/
│   │   │   └── UserApiService.ts
│   │   ├── hooks/
│   │   │   └── useUsers.ts
│   │   └── types/
│   │       ├── entity.ts
│   │       └── api.ts
│   │
│   ├── products/
│   │   ├── config/
│   │   │   └── productPageConfig.ts
│   │   ├── pages/
│   │   │   └── ProductManagementPage.tsx
│   │   └── ...
│   │
│   └── suppliers/
│       ├── config/
│       │   └── supplierPageConfig.ts
│       ├── pages/
│       │   └── SupplierManagementPage.tsx
│       └── ...
│
├── hooks/
│   ├── useApi.ts                            # Existing - không đổi
│   ├── usePaginationWithRouter.ts           # Existing - không đổi
│   └── ...
│
└── lib/
    └── api/
        └── base/
            └── BaseApiService.ts            # Existing - không đổi
```

### 6.2. File Count Comparison

**Trước:**
- Mỗi entity: ~8-10 files (Page, Hook, Table, Form, Modals, Header, Statistics, SearchFilter)
- 10 entities = ~80-100 files

**Sau:**
- Generic components: ~5-7 files (GenericPage, GenericTable, GenericForm, Config types)
- Mỗi entity: ~3-4 files (Page, Config, optional custom components)
- 10 entities = ~35-47 files

**Giảm:** ~50% số lượng files

### 6.3. Code Reuse

**Generic Components:**
- GenericPage: ~200 lines (viết 1 lần)
- GenericTable: ~300 lines (viết 1 lần)
- GenericForm: ~250 lines (viết 1 lần)
- Config types: ~400 lines (viết 1 lần)
- **Total:** ~1150 lines (viết 1 lần, dùng cho tất cả entities)

**Per Entity:**
- PageConfig: ~150-200 lines (config-driven)
- Page component: ~10 lines
- **Total per entity:** ~160-210 lines

**So với trước:**
- Trước: ~300-400 lines per entity × 10 = 3000-4000 lines
- Sau: 1150 lines (generic) + 160-210 lines × 10 = 2750-3250 lines
- **Giảm:** ~10-20% tổng số lines, nhưng **tăng đáng kể maintainability**

---

## 7. Testing Strategy

### 7.1. Unit Tests

**GenericForm:**
- [ ] Test field rendering cho mỗi type
- [ ] Test validation rules
- [ ] Test conditional fields
- [ ] Test dependent selects

**GenericTable:**
- [ ] Test column rendering
- [ ] Test pagination
- [ ] Test sort
- [ ] Test search
- [ ] Test row actions

**GenericPage:**
- [ ] Test create flow
- [ ] Test update flow
- [ ] Test delete flow
- [ ] Test modal/drawer modes

### 7.2. Integration Tests

**User Page:**
- [ ] Test full CRUD flow
- [ ] Test URL sync
- [ ] Test error handling
- [ ] Test slots rendering

### 7.3. E2E Tests (Optional)

- [ ] Test User CRUD flow end-to-end
- [ ] Test Product CRUD flow end-to-end
- [ ] Test navigation, pagination, filters

---

## 8. Risks và Mitigation

### 8.1. Over-Abstraction

**Risk:** GenericPage quá phức tạp, khó customize

**Mitigation:**
- Giữ slots pattern cho customization
- Cho phép custom renderers
- Không force tất cả pages dùng GenericPage

### 8.2. Type Safety

**Risk:** TypeScript generics có thể phức tạp

**Mitigation:**
- Viết type definitions rõ ràng
- Cung cấp examples
- Code review kỹ

### 8.3. Migration Complexity

**Risk:** Migration từ code cũ mất thời gian

**Mitigation:**
- Migration từng entity một
- Giữ code cũ song song trong thời gian transition
- Test kỹ trước khi deploy

### 8.4. Performance

**Risk:** Generic components có thể chậm hơn custom code

**Mitigation:**
- Profile và optimize nếu cần
- Sử dụng React.memo cho components
- Lazy load nếu cần

### 8.5. Learning Curve

**Risk:** Team cần thời gian học pattern mới

**Mitigation:**
- Tài liệu chi tiết
- Code examples
- Pair programming
- Training session

---

## 9. Success Metrics

### 9.1. Code Reduction

- **Target:** Giảm ≥40% code lặp lại
- **Measurement:** So sánh lines of code trước/sau

### 9.2. Development Speed

- **Target:** Tạo page CRUD mới trong <2 giờ (thay vì 1 ngày)
- **Measurement:** Track time để tạo page mới

### 9.3. Maintainability

- **Target:** Bug fix/feature update áp dụng cho tất cả pages
- **Measurement:** Track số lần cần update nhiều files

### 9.4. Consistency

- **Target:** Tất cả pages có UI/UX nhất quán
- **Measurement:** Design review, user feedback

---

## 10. Next Steps

1. **Review Plan:** Team review và approve plan
2. **Setup Branch:** Tạo feature branch `feature/generic-page`
3. **Phase 1:** Implement core infrastructure
4. **Phase 2:** Migrate User page (pilot)
5. **Phase 3:** Extend to other entities
6. **Phase 4:** Advanced features (optional)

---

## 11. References

- [GenericPage.md](./GenericPage.md) - Tài liệu chi tiết về pattern
- [architectured-doc.md](./architectured-doc.md) - Architecture document
- [PAGE_STRUCTURE_GUIDE.md](../Structured/PAGE_STRUCTURE_GUIDE.md) - Current page structure

---

**Tác giả:** AI Assistant  
**Ngày tạo:** 2024  
**Version:** 1.0

