# Generic Page Pattern - Giải pháp tránh Code Lặp lại cho CRUD Pages

Dựa trên phân tích file cấu trúc user page và yêu cầu của bạn, tôi đề xuất **Configuration-Driven Pattern** kết hợp **Compound Components** để tạo generic page.

## 📋 Mục lục

1. [Phân tích Problem](#1-phân-tích-problem)
2. [Solution Overview](#2-solution-overview)
3. [Generic Components Architecture](#3-generic-components-architecture)
   - [Slot Pattern (Customizable Areas)](#33-slot-pattern-customizable-areas)
4. [Implementation Guide](#4-implementation-guide)
   - [GenericFilterBar (Optional, Partial Generic Filter)](#44-genericfilterbar-optional-partial-generic-filter)
5. [Usage Examples](#5-usage-examples)
   - [Slot-based Usage](#55-slot-based-usage)
6. [Advanced Patterns](#6-advanced-patterns)

---

## 1. Phân tích Problem

### 1.1. Điểm chung giữa các pages (User, Product, Supplier)

✅ **Giống nhau:**
- Table với pagination
- Search/Filter functionality
- Sort columns
- CRUD actions (Create, Edit, Delete)
- Form với validation
- Loading/Error states
- Modal/Drawer cho Create/Edit

❌ **Khác nhau:**
- Entity types (User, Product, Supplier)
- Column definitions (tên, số lượng, render logic)
- Form fields (types, validation rules)
- API endpoints
- Business logic riêng

### 1.2. Code Duplication Issues

```typescript
// ❌ Code lặp lại - Mỗi entity phải viết lại
// UserListPage.tsx
const columns = [
  { title: 'Username', dataIndex: 'username' },
  { title: 'Full Name', dataIndex: 'fullName' },
  // ...
];

// ProductListPage.tsx  
const columns = [
  { title: 'Product Name', dataIndex: 'productName' },
  { title: 'Price', dataIndex: 'price' },
  // ...
];

// SupplierListPage.tsx
const columns = [
  { title: 'Supplier Name', dataIndex: 'name' },
  { title: 'Phone', dataIndex: 'phone' },
  // ...
];
```

---

## 2. Solution Overview

### 2.1. Configuration-Driven Pattern

**Ý tưởng**: Tạo **config object** cho mỗi entity, **GenericPage** nhận config và render.

```typescript
// Định nghĩa config cho User
const userPageConfig = {
  entity: 'users',
  apiService: userApiService,
  columns: [...],
  formFields: [...],
  // ...
};

// Sử dụng GenericPage
<GenericPage config={userPageConfig} />
```

### 2.2. Compound Components Pattern

**Ý tưởng**: Tạo các sub-components có thể compose lại.

```typescript
<GenericPage.Table config={tableConfig} />
<GenericPage.Form config={formConfig} />
<GenericPage.Actions config={actionsConfig} />
```

### 2.3. Recommended Architecture

```
                    ┌─────────────────┐
                    │   PageConfig    │
                    │  (per entity)   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  GenericPage    │
                    │   (Container)   │
                    └────────┬────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
    ┌─────▼─────┐     ┌─────▼─────┐     ┌─────▼─────┐
    │  Generic  │     │  Generic  │     │  Generic  │
    │   Table   │     │   Form    │     │  Actions  │
    └───────────┘     └───────────┘     └───────────┘
```

---

## 3. Generic Components Architecture

### 3.1. File Structure

```typescript
src/
├── components/
│   └── GenericCRUD/
│       ├── GenericPage/
│       │   ├── GenericPage.tsx              # Main page component
│       │   ├── GenericPageConfig.ts         # Config types
│       │   └── index.ts
│       ├── GenericTable/
│       │   ├── GenericTable.tsx             # Table component
│       │   ├── GenericTableConfig.ts        # Table config types
│       │   ├── components/
│       │   │   ├── TableToolbar.tsx         # Search, filters, actions
│       │   │   └── TableActions.tsx         # Row actions (edit, delete)
│       │   └── index.ts
│       ├── GenericForm/
│       │   ├── GenericForm.tsx              # Form component
│       │   ├── GenericFormConfig.ts         # Form config types
│       │   ├── renderers/
│       │   │   ├── TextField.tsx            # Text input renderer
│       │   │   ├── NumberField.tsx          # Number input renderer
│       │   │   ├── SelectField.tsx          # Select renderer
│       │   │   └── index.ts
│       │   └── index.ts
│       └── index.ts
│
└── features/
    ├── users/
    │   ├── config/
    │   │   └── userPageConfig.ts            # User-specific config
    │   └── pages/
    │       └── UserListPage.tsx              # Use GenericPage with config
    ├── products/
    │   ├── config/
    │   │   └── productPageConfig.ts
    │   └── pages/
    │       └── ProductListPage.tsx
    └── suppliers/
        ├── config/
        │   └── supplierPageConfig.ts
        └── pages/
            └── SupplierListPage.tsx
```

### 3.2. Config Type Definitions

```typescript
// src/components/GenericCRUD/GenericPage/GenericPageConfig.ts

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
}

// ==================== Table Config ====================

export interface TableConfig<TData> {
  // Column definitions (tương tự Ant Design Table columns)
  columns: GenericColumnType<TData>[];

  // Row key (default: 'id')
  rowKey?: keyof TData | ((record: TData) => string | number);

  // Sorting
  defaultSortBy?: string;
  defaultSortDesc?: boolean;

  // Pagination
  defaultPageSize?: number;
  pageSizeOptions?: number[];

  // Row selection (for bulk actions)
  rowSelection?: {
    enabled: boolean;
    onChange?: (selectedRowKeys: React.Key[], selectedRows: TData[]) => void;
  };

  // Custom row render
  onRow?: (record: TData) => React.HTMLAttributes<HTMLTableRowElement>;
}

export interface GenericColumnType<TData> extends Omit<ColumnType<TData>, 'render'> {
  // Standard Ant Design column props
  title: string;
  dataIndex?: keyof TData;
  key?: string;
  width?: number | string;
  fixed?: 'left' | 'right';
  sorter?: boolean;

  // Custom render function
  render?: (value: any, record: TData, index: number) => React.ReactNode;

  // Quick formatters (để tránh phải viết render function)
  format?: {
    type: 'date' | 'datetime' | 'currency' | 'number' | 'boolean' | 'custom';
    options?: any;
  };
}

// ==================== Form Config ====================

export interface FormConfig<TFormData> {
  // Form fields
  fields: FormFieldConfig<TFormData>[];

  // Form layout
  layout?: 'horizontal' | 'vertical' | 'inline';
  labelCol?: { span: number };
  wrapperCol?: { span: number };

  // Initial values
  initialValues?: Partial<TFormData>;

  // Validation
  validateTrigger?: 'onChange' | 'onBlur' | 'onSubmit';

  // Custom form rendering
  customRender?: (form: FormInstance, fields: React.ReactNode) => React.ReactNode;
}

export interface FormFieldConfig<TFormData> {
  // Field metadata
  name: keyof TFormData;
  label: string;
  required?: boolean;

  // Field type (determines which renderer to use)
  type: 'text' | 'number' | 'select' | 'date' | 'datetime' | 'textarea' | 'password' | 'email' | 'phone' | 'custom';

  // Validation rules
  rules?: Rule[];

  // Field-specific config (based on type)
  config?: FieldTypeConfig;

  // Conditional rendering
  visible?: boolean | ((formData: Partial<TFormData>) => boolean);
  disabled?: boolean | ((formData: Partial<TFormData>) => boolean);

  // Custom renderer (override default)
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
  // Async options loading
  fetchOptions?: () => Promise<SelectOption[]>;
  // Dependent selects
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
  
  // Action scope
  scope: 'table' | 'row' | 'both';

  // Visibility
  visible?: boolean | ((record?: TData) => boolean);
  disabled?: boolean | ((record?: TData) => boolean);

  // Action handler
  onClick: (record?: TData, selectedRows?: TData[]) => void | Promise<void>;

  // Confirmation
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

### 3.3. Slot Pattern (Customizable Areas)

- Mục tiêu: giữ core GenericPage (Table/Form/CRUD) nhưng mở điểm cắm để thêm phần đặc thù từng entity.
- Đề xuất các slot:
  - `header`: render header tuỳ biến (hero, breadcrumb, CTA).
  - `statistics`: KPI/metrics đặc thù domain.
  - `toolbar`: chèn thêm UI không phải action (badge/toggle/filter phụ) trước bảng.
  - `filters`: cắm filter tùy biến hoặc GenericFilterBar.
  - `table`: override bảng mặc định (render bảng custom hoặc wrap GenericTable mặc định).
  - `footer`: ghi chú, secondary actions.
- Cách dùng (ý tưởng): truyền `slots` trong config hoặc prop GenericPage; nếu không truyền sẽ dùng mặc định (header cơ bản, bảng mặc định, không statistics/filters/footer).

---

## 4. Implementation Guide

### 4.1. GenericTable Component

```typescript
// src/components/GenericCRUD/GenericTable/GenericTable.tsx

import { Table, Button, Space, Input, Popconfirm, message } from 'antd';
import { PlusOutlined, EditOutlined, DeleteOutlined, SearchOutlined, ReloadOutlined } from '@ant-design/icons';
import { useState } from 'react';
import { usePaginationWithRouter } from '@/hooks/usePaginationWithRouter';
import { useApiDelete } from '@/hooks/useApi';
import type { GenericPageConfig } from '../GenericPage/GenericPageConfig';
import type { TableConfig, GenericColumnType, CustomAction } from '../GenericPage/GenericPageConfig';

interface GenericTableProps<TData, TCreate, TUpdate> {
  config: GenericPageConfig<TData, TCreate, TUpdate>;
  routeApi: any;
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
    ...tableConfig.columns.map(col => ({
      ...col,
      sorter: col.sorter,
      render: col.render || ((value: any) => {
        // Auto-format dựa trên type
        if (col.format) {
          switch (col.format.type) {
            case 'date':
              return value ? new Date(value).toLocaleDateString() : '-';
            case 'datetime':
              return value ? new Date(value).toLocaleString() : '-';
            case 'currency':
              return value != null ? `${value.toLocaleString()} đ` : '-';
            case 'number':
              return value != null ? value.toLocaleString() : '-';
            case 'boolean':
              return value ? 'Yes' : 'No';
            default:
              return value ?? '-';
          }
        }
        return value ?? '-';
      }),
    })),
    // Actions column
    {
      title: 'Actions',
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
              Edit
            </Button>
          )}
          {enableDelete && (
            <Popconfirm
              title={`Delete ${entity.displayName}?`}
              description="This action cannot be undone."
              onConfirm={() => deleteItem.mutate(record.id!)}
              okText="Delete"
              cancelText="Cancel"
              okButtonProps={{ danger: true }}
            >
              <Button
                type="link"
                danger
                icon={<DeleteOutlined />}
                loading={deleteItem.isPending}
              >
                Delete
              </Button>
            </Popconfirm>
          )}
          {/* Custom row actions */}
          {config.customActions
            ?.filter(action => action.scope === 'row' || action.scope === 'both')
            .filter(action => 
              typeof action.visible === 'function' 
                ? action.visible(record) 
                : action.visible !== false
            )
            .map(action => (
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
          placeholder={`Search ${entity.displayNamePlural.toLowerCase()}...`}
          allowClear
          enterButton={<SearchOutlined />}
          onSearch={handleSearch}
          defaultValue={params.search}
          style={{ width: 300 }}
        />
        <Button icon={<ReloadOutlined />} onClick={() => refetch()}>
          Refresh
        </Button>
      </Space>
      
      <Space>
        {/* Table-level custom actions */}
        {config.customActions
          ?.filter(action => action.scope === 'table' || action.scope === 'both')
          .map(action => (
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
            Create {entity.displayName}
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
            : (record) => record[tableConfig.rowKey as keyof TData] as string | number || record.id as string | number
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
            `${range[0]}-${range[1]} of ${total} ${entity.displayNamePlural.toLowerCase()}`,
          pageSizeOptions: tableConfig.pageSizeOptions || ['10', '20', '50', '100'],
        }}
        scroll={{ x: 'max-content' }}
      />
    </div>
  );
}
```

### 4.2. GenericForm Component

```typescript
// src/components/GenericCRUD/GenericForm/GenericForm.tsx

import { Form, Input, InputNumber, Select, DatePicker, Button, Space } from 'antd';
import type { FormInstance } from 'antd/es/form';
import type { FormConfig, FormFieldConfig, FieldTypeConfig } from '../GenericPage/GenericPageConfig';

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
        return (
          <InputNumber
            placeholder={fieldConfig.config?.placeholder}
            min={fieldConfig.config?.min}
            max={fieldConfig.config?.max}
            step={fieldConfig.config?.step}
            precision={fieldConfig.config?.precision}
            formatter={fieldConfig.config?.formatter}
            parser={fieldConfig.config?.parser}
            style={{ width: '100%' }}
          />
        );

      case 'select':
        return (
          <Select
            placeholder={fieldConfig.config?.placeholder}
            options={fieldConfig.config?.options}
            mode={fieldConfig.config?.mode}
            showSearch={fieldConfig.config?.showSearch}
            allowClear={fieldConfig.config?.allowClear}
          />
        );

      case 'date':
        return (
          <DatePicker
            placeholder={fieldConfig.config?.placeholder}
            format={fieldConfig.config?.format}
            showTime={fieldConfig.config?.showTime}
            disabledDate={fieldConfig.config?.disabledDate}
            style={{ width: '100%' }}
          />
        );

      case 'textarea':
        return (
          <Input.TextArea
            placeholder={fieldConfig.config?.placeholder}
            rows={fieldConfig.config?.rows}
            maxLength={fieldConfig.config?.maxLength}
            showCount={fieldConfig.config?.showCount}
          />
        );

      default:
        return <Input placeholder={fieldConfig.config?.placeholder} />;
    }
  };

  const fields = config.fields.map(fieldConfig => {
    // Check visibility
    const visible =
      typeof fieldConfig.visible === 'function'
        ? fieldConfig.visible(form.getFieldsValue())
        : fieldConfig.visible !== false;

    if (!visible) return null;

    const disabled =
      typeof fieldConfig.disabled === 'function'
        ? fieldConfig.disabled(form.getFieldsValue())
        : fieldConfig.disabled;

    return (
      <Form.Item
        key={String(fieldConfig.name)}
        name={fieldConfig.name as string}
        label={fieldConfig.label}
        rules={fieldConfig.rules}
      >
        {React.cloneElement(renderField(fieldConfig) as React.ReactElement, { disabled })}
      </Form.Item>
    );
  });

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

### 4.3. GenericPage Component (Main Container)

```typescript
// src/components/GenericCRUD/GenericPage/GenericPage.tsx

import { useState } from 'react';
import { Card, Modal, Drawer } from 'antd';
import { GenericTable } from '../GenericTable/GenericTable';
import { GenericForm } from '../GenericForm/GenericForm';
import { useApiCreate, useApiUpdate, useApiDetail } from '@/hooks/useApi';
import type { GenericPageConfig } from './GenericPageConfig';

interface GenericPageProps<TData, TCreate, TUpdate> {
  config: GenericPageConfig<TData, TCreate, TUpdate>;
  routeApi: any;
  formMode?: 'modal' | 'drawer';
}

export function GenericPage<TData extends { id?: number | string }, TCreate, TUpdate>({
  config,
  routeApi,
  formMode = 'modal',
}: GenericPageProps<TData, TCreate, TUpdate>) {
  const { entity, apiService, form: formConfig, features = {} } = config;
  
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
        message.success(`${entity.displayName} created successfully!`);
        setCreateModalVisible(false);
        config.onAfterCreate?.(data);
      },
      onError: (error) => {
        message.error(`Failed to create ${entity.displayName.toLowerCase()}: ${error.message}`);
      },
    },
  });

  const updateItem = useApiUpdate<TData, TUpdate>({
    apiService,
    entity: entity.name,
    options: {
      onSuccess: (data) => {
        message.success(`${entity.displayName} updated successfully!`);
        setEditModalVisible(false);
        setEditingId(null);
        config.onAfterUpdate?.(data);
      },
      onError: (error) => {
        message.error(`Failed to update ${entity.displayName.toLowerCase()}: ${error.message}`);
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
    <div>
      <Card title={entity.displayNamePlural}>
        <GenericTable
          config={config}
          routeApi={routeApi}
          onCreateClick={() => setCreateModalVisible(true)}
          onEditClick={handleEditClick}
        />
      </Card>

      {/* Create Form */}
      {createModalVisible && (
        <FormContainer
          {...formContainerProps}
          title={`Create ${entity.displayName}`}
        >
          <GenericForm
            config={formConfig.create}
            onSubmit={handleCreate}
            onCancel={() => setCreateModalVisible(false)}
            submitButtonText="Create"
            loading={createItem.isPending}
          />
        </FormContainer>
      )}

      {/* Edit Form */}
      {editModalVisible && (
        <FormContainer
          {...formContainerProps}
          title={`Edit ${entity.displayName}`}
        >
          {isLoadingEdit ? (
            <div>Loading...</div>
          ) : (
            <GenericForm
              config={formConfig.update}
              onSubmit={handleUpdate}
              onCancel={() => {
                setEditModalVisible(false);
                setEditingId(null);
              }}
              initialValues={editingItem}
              submitButtonText="Update"
              loading={updateItem.isPending}
            />
          )}
        </FormContainer>
      )}
    </div>
  );
}
```

---

### 4.4. GenericFilterBar (Optional, Partial Generic Filter)

- Mục tiêu: tái sử dụng các filter type phổ biến (search, select, dateRange, numberRange, checkbox) và cắm qua slot `filters`.
- Khi dùng: kết hợp với `usePaginationWithRouter` hoặc handler riêng để đẩy filter state vào URL/search params.

Ví dụ ngắn:

```typescript
import { GenericFilterBar } from '@/components/GenericCRUD/GenericFilterBar/GenericFilterBar';

<GenericFilterBar
  config={{
    filters: [
      { type: 'search', name: 'search', placeholder: 'Search by name', width: 280 },
      {
        type: 'select',
        name: 'categoryId',
        label: 'Category',
        fetchOptions: async () => {
          const cats = await categoryApi.getAll();
          return cats.map(c => ({ label: c.categoryName, value: c.id }));
        },
        showSearch: true,
        allowClear: true,
      },
      { type: 'numberRange', name: 'price', label: 'Price', fields: ['minPrice', 'maxPrice'], min: 0 },
    ],
    layout: 'inline',
    showApplyButton: true,
    showClearButton: true,
    onFilterChange: handleFilterChange, // đẩy vào navigate/search params
  }}
/>
```

Cắm vào GenericPage qua slot `filters`:

```tsx
<GenericPage
  config={productPageConfig}
  routeApi={routeApi}
  slots={{
    filters: (
      <GenericFilterBar config={filterConfig} />
    ),
  }}
/>
```

---

## 5. Usage Examples

### 5.1. User Page Config

```typescript
// src/features/users/config/userPageConfig.ts

import { userApiService } from '../api/UserApiService';
import type { GenericPageConfig } from '@/components/GenericCRUD/GenericPage/GenericPageConfig';
import type { UserEntity } from '../types/entity';
import type { CreateUserRequest, UpdateUserRequest } from '../types/api';

export const userPageConfig: GenericPageConfig<UserEntity, CreateUserRequest, UpdateUserRequest> = {
  // Entity metadata
  entity: {
    name: 'users',
    displayName: 'User',
    displayNamePlural: 'Users',
  },

  // API Service
  apiService: userApiService,

  // Table configuration
  table: {
    columns: [
      {
        title: 'ID',
        dataIndex: 'id',
        key: 'id',
        width: 80,
        sorter: true,
      },
      {
        title: 'Username',
        dataIndex: 'username',
        key: 'username',
        sorter: true,
      },
      {
        title: 'Full Name',
        dataIndex: 'fullName',
        key: 'fullName',
        sorter: true,
      },
      {
        title: 'Role',
        dataIndex: 'role',
        key: 'role',
        sorter: true,
        render: (role: number) => (role === 0 ? 'Admin' : 'Staff'),
      },
      {
        title: 'Created At',
        dataIndex: 'createdAt',
        key: 'createdAt',
        sorter: true,
        format: {
          type: 'datetime',
        },
      },
    ],
    defaultSortBy: 'id',
    defaultSortDesc: true,
    defaultPageSize: 20,
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
            { required: true, message: 'Please enter username' },
            { min: 1, max: 50, message: 'Username must be 1-50 characters' },
          ],
          config: {
            placeholder: 'Enter username',
            maxLength: 50,
          },
        },
        {
          name: 'password',
          label: 'Password',
          type: 'password',
          required: true,
          rules: [
            { required: true, message: 'Please enter password' },
            { min: 6, message: 'Password must be at least 6 characters' },
          ],
          config: {
            placeholder: 'Enter password',
          },
        },
        {
          name: 'fullName',
          label: 'Full Name',
          type: 'text',
          required: true,
          rules: [
            { required: true, message: 'Please enter full name' },
            { min: 1, max: 100, message: 'Full name must be 1-100 characters' },
          ],
          config: {
            placeholder: 'Enter full name',
            maxLength: 100,
          },
        },
        {
          name: 'role',
          label: 'Role',
          type: 'select',
          required: true,
          rules: [{ required: true, message: 'Please select role' }],
          config: {
            placeholder: 'Select role',
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
          rules: [
            { required: true, message: 'Please enter username' },
            { min: 1, max: 50, message: 'Username must be 1-50 characters' },
          ],
          config: {
            placeholder: 'Enter username',
            maxLength: 50,
          },
        },
        {
          name: 'password',
          label: 'Password',
          type: 'password',
          required: false,
          rules: [
            { min: 6, message: 'Password must be at least 6 characters' },
          ],
          config: {
            placeholder: 'Leave blank to keep current password',
          },
        },
        {
          name: 'fullName',
          label: 'Full Name',
          type: 'text',
          required: true,
          rules: [
            { required: true, message: 'Please enter full name' },
            { min: 1, max: 100, message: 'Full name must be 1-100 characters' },
          ],
          config: {
            placeholder: 'Enter full name',
            maxLength: 100,
          },
        },
        {
          name: 'role',
          label: 'Role',
          type: 'select',
          required: true,
          rules: [{ required: true, message: 'Please select role' }],
          config: {
            placeholder: 'Select role',
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
};
```

### 5.2. User List Page Usage

```typescript
// src/features/users/pages/UserListPage.tsx

import { getRouteApi } from '@tanstack/react-router';
import { GenericPage } from '@/components/GenericCRUD/GenericPage/GenericPage';
import { userPageConfig } from '../config/userPageConfig';

const routeApi = getRouteApi('/admin/users');

export const UserListPage = () => {
  return <GenericPage config={userPageConfig} routeApi={routeApi} formMode="modal" />;
};
```

### 5.3. Product Page Config

```typescript
// src/features/products/config/productPageConfig.ts

import { productApiService } from '../api/ProductApiService';
import { categoryApiService } from '@/features/categories/api/CategoryApiService';
import { supplierApiService } from '@/features/suppliers/api/SupplierApiService';
import type { GenericPageConfig } from '@/components/GenericCRUD/GenericPage/GenericPageConfig';
import type { ProductEntity } from '../types/entity';
import type { CreateProductRequest, UpdateProductRequest } from '../types/api';

export const productPageConfig: GenericPageConfig<ProductEntity, CreateProductRequest, UpdateProductRequest> = {
  entity: {
    name: 'products',
    displayName: 'Product',
    displayNamePlural: 'Products',
  },

  apiService: productApiService,

  table: {
    columns: [
      {
        title: 'ID',
        dataIndex: 'id',
        width: 80,
        sorter: true,
      },
      {
        title: 'Product Name',
        dataIndex: 'productName',
        sorter: true,
      },
      {
        title: 'Barcode',
        dataIndex: 'barcode',
        sorter: true,
      },
      {
        title: 'Price',
        dataIndex: 'price',
        sorter: true,
        format: {
          type: 'currency',
        },
      },
      {
        title: 'Unit',
        dataIndex: 'unit',
      },
      {
        title: 'Category',
        dataIndex: 'categoryName',
        sorter: true,
      },
      {
        title: 'Supplier',
        dataIndex: 'supplierName',
        sorter: true,
      },
      {
        title: 'Inventory',
        dataIndex: 'inventoryQuantity',
        sorter: true,
        format: {
          type: 'number',
        },
      },
    ],
    defaultSortBy: 'id',
    defaultPageSize: 20,
  },

  form: {
    create: {
      fields: [
        {
          name: 'productName',
          label: 'Product Name',
          type: 'text',
          required: true,
          rules: [
            { required: true, message: 'Please enter product name' },
            { min: 1, max: 100, message: 'Product name must be 1-100 characters' },
          ],
          config: {
            placeholder: 'Enter product name',
            maxLength: 100,
          },
        },
        {
          name: 'barcode',
          label: 'Barcode',
          type: 'text',
          required: true,
          rules: [
            { required: true, message: 'Please enter barcode' },
            { min: 1, max: 50, message: 'Barcode must be 1-50 characters' },
          ],
          config: {
            placeholder: 'Enter barcode',
            maxLength: 50,
          },
        },
        {
          name: 'price',
          label: 'Price',
          type: 'number',
          required: true,
          rules: [
            { required: true, message: 'Please enter price' },
            { type: 'number', min: 0.01, message: 'Price must be greater than 0' },
          ],
          config: {
            placeholder: 'Enter price',
            min: 0.01,
            precision: 2,
            formatter: (value) => `${value}`.replace(/\B(?=(\d{3})+(?!\d))/g, ','),
          },
        },
        {
          name: 'unit',
          label: 'Unit',
          type: 'text',
          required: true,
          rules: [
            { required: true, message: 'Please enter unit' },
            { min: 1, max: 20, message: 'Unit must be 1-20 characters' },
          ],
          config: {
            placeholder: 'Enter unit (e.g., can, bottle, pack)',
            maxLength: 20,
          },
        },
        {
          name: 'categoryId',
          label: 'Category',
          type: 'select',
          required: true,
          rules: [{ required: true, message: 'Please select category' }],
          config: {
            placeholder: 'Select category',
            showSearch: true,
            allowClear: true,
            fetchOptions: async () => {
              // Fetch categories from API
              const categories = await categoryApiService.getAll();
              return categories.map(cat => ({
                label: cat.categoryName,
                value: cat.id!,
              }));
            },
          },
        },
        {
          name: 'supplierId',
          label: 'Supplier',
          type: 'select',
          required: true,
          rules: [{ required: true, message: 'Please select supplier' }],
          config: {
            placeholder: 'Select supplier',
            showSearch: true,
            allowClear: true,
            fetchOptions: async () => {
              const suppliers = await supplierApiService.getAll();
              return suppliers.map(sup => ({
                label: sup.name,
                value: sup.id!,
              }));
            },
          },
        },
      ],
      layout: 'vertical',
    },
    update: {
      fields: [
        // Same as create fields...
      ],
      layout: 'vertical',
    },
  },

  features: {
    enableCreate: true,
    enableEdit: true,
    enableDelete: true,
    enableBulkDelete: false,
    enableExport: true,
  },

  // Custom actions
  customActions: [
    {
      key: 'export',
      label: 'Export',
      icon: <ExportOutlined />,
      type: 'default',
      scope: 'table',
      onClick: async (_, selectedRows) => {
        // Export logic
        message.info('Exporting products...');
      },
    },
  ],
};
```

### 5.4. Product List Page Usage

```typescript
// src/features/products/pages/ProductListPage.tsx

import { getRouteApi } from '@tanstack/react-router';
import { GenericPage } from '@/components/GenericCRUD/GenericPage/GenericPage';
import { productPageConfig } from '../config/productPageConfig';

const routeApi = getRouteApi('/admin/products');

export const ProductListPage = () => {
  return <GenericPage config={productPageConfig} routeApi={routeApi} formMode="modal" />;
};
```

### 5.5. Slot-based Usage

```typescript
// src/features/products/pages/ProductListPageWithSlots.tsx
import { GenericPage } from '@/components/GenericCRUD/GenericPage/GenericPage';
import { GenericFilterBar } from '@/components/GenericCRUD/GenericFilterBar/GenericFilterBar';
import { ProductStatistics } from '../components/ProductStatistics';
import { productPageConfig } from '../config/productPageConfig';

export const ProductListPage = () => {
  return (
    <GenericPage
      config={productPageConfig}
      routeApi={routeApi}
      slots={{
        header: <ProductHeader />,               // Custom header
        statistics: <ProductStatistics />,       // KPI cards
        filters: (
          <GenericFilterBar
            config={{
              filters: [
                { type: 'search', name: 'search', placeholder: 'Search products' },
                { type: 'select', name: 'categoryId', label: 'Category', fetchOptions: fetchCategories },
              ],
              showApplyButton: true,
              showClearButton: true,
              onFilterChange: handleFilterChange, // đẩy vào navigate/search params
            }}
          />
        ),
      }}
    />
  );
};
```

---

## 6. Advanced Patterns

### 6.1. Custom Column Renderers

```typescript
// Trong config
columns: [
  {
    title: 'Status',
    dataIndex: 'status',
    render: (status: string) => {
      const color = status === 'active' ? 'green' : 'red';
      return <Tag color={color}>{status.toUpperCase()}</Tag>;
    },
  },
]
```

### 6.2. Dependent Fields (Cascading Selects)

```typescript
{
  name: 'cityId',
  label: 'City',
  type: 'select',
  config: {
    placeholder: 'Select city',
    dependsOn: 'countryId',
    getOptionsWhen: async (countryId) => {
      const cities = await fetchCitiesByCountry(countryId);
      return cities.map(city => ({ label: city.name, value: city.id }));
    },
  },
}
```

### 6.3. Custom Validators

```typescript
{
  name: 'email',
  label: 'Email',
  type: 'email',
  rules: [
    { required: true, message: 'Please enter email' },
    { type: 'email', message: 'Invalid email format' },
    {
      validator: async (_, value) => {
        const exists = await checkEmailExists(value);
        if (exists) {
          throw new Error('Email already exists');
        }
      },
    },
  ],
}
```

### 6.4. Conditional Fields

```typescript
{
  name: 'discountType',
  label: 'Discount Type',
  type: 'select',
  config: {
    options: [
      { label: 'Percentage', value: 'percent' },
      { label: 'Fixed Amount', value: 'fixed' },
    ],
  },
},
{
  name: 'discountValue',
  label: 'Discount Value',
  type: 'number',
  visible: (formData) => !!formData.discountType,
  rules: [
    {
      validator: (_, value) => {
        const type = form.getFieldValue('discountType');
        if (type === 'percent' && value > 100) {
          return Promise.reject('Percentage cannot exceed 100');
        }
        return Promise.resolve();
      },
    },
  ],
}
```

### 6.5. Bulk Actions

```typescript
customActions: [
  {
    key: 'bulkDelete',
    label: 'Delete Selected',
    type: 'default',
    danger: true,
    scope: 'table',
    visible: (_, selectedRows) => selectedRows && selectedRows.length > 0,
    confirm: {
      title: 'Delete Selected Items?',
      content: 'This action cannot be undone.',
    },
    onClick: async (_, selectedRows) => {
      await Promise.all(
        selectedRows!.map(row => productApiService.delete(row.id!))
      );
      message.success('Items deleted successfully');
    },
  },
]
```

### 6.6. Import/Export Actions

```typescript
customActions: [
  {
    key: 'import',
    label: 'Import',
    icon: <ImportOutlined />,
    scope: 'table',
    onClick: () => {
      // Open import modal
    },
  },
  {
    key: 'export',
    label: 'Export',
    icon: <ExportOutlined />,
    scope: 'table',
    onClick: async (_, selectedRows) => {
      const data = selectedRows && selectedRows.length > 0
        ? selectedRows
        : await productApiService.getAll();
      exportToExcel(data, 'products.xlsx');
    },
  },
]
```

---

## 7. Tổng kết

### Quy tắc Full / Partial / Custom

- **Full Generic**: Table, Form, CRUD actions (config-driven).
- **Partial Generic**: Filters (GenericFilterBar + slot `filters`), Bulk actions, Export/Import (customActions/strategy).
- **Custom qua Slots**: Header, Statistics, toolbar extras, complex filters — cắm vào `slots` (header/statistics/toolbar/filters/table/footer).

### ✅ Ưu điểm của Configuration-Driven Pattern:

1. **DRY (Don't Repeat Yourself)**
   - Viết code một lần, dùng cho mọi entity
   - Table, Form, Actions logic được tái sử dụng

2. **Type-Safe**
   - TypeScript generics đảm bảo type safety
   - Autocomplete tốt với config

3. **Maintainable**
   - Logic tập trung trong GenericPage
   - Config riêng biệt, dễ đọc, dễ test

4. **Extensible**
   - Custom renderers
   - Custom actions
   - Custom validators
   - Hooks để customize behavior

5. **Consistent UI/UX**
   - Tất cả pages có giao diện và behavior nhất quán
   - Dễ dàng update global behavior

### 📊 So sánh trước và sau:

**❌ Trước (Code lặp lại):**
```typescript
// UserListPage.tsx - 300 dòng
// ProductListPage.tsx - 300 dòng
// SupplierListPage.tsx - 300 dòng
// → Total: 900 dòng code lặp lại
```

**✅ Sau (Configuration-Driven):**
```typescript
// GenericPage - 200 dòng (viết 1 lần)
// userPageConfig.ts - 80 dòng
// productPageConfig.ts - 100 dòng
// supplierPageConfig.ts - 80 dòng
// → Total: 460 dòng (giảm 49%)
```

### 🎯 Khi nào nên dùng pattern này?

✅ **Phù hợp khi:**
- Có nhiều CRUD pages tương tự nhau
- Muốn UI/UX nhất quán
- Cần dễ maintain và extend
- Team có nhiều developers làm việc song song

❌ **Không phù hợp khi:**
- Page có logic phức tạp, đặc biệt
- UI khác biệt nhiều giữa các pages
- Cần performance optimization cao
- Team nhỏ, ít pages

### 🚀 Migration Path:

1. **Phase 1**: Tạo GenericPage components
2. **Phase 2**: Migrate 1 entity pilot (User)
3. **Phase 3**: Migrate các entities còn lại
4. **Phase 4**: Add advanced features (import/export, bulk actions)

Pattern này giúp bạn tránh được code duplication, dễ maintain, và scale tốt khi có nhiều CRUD pages tương tự nhau! 🎉