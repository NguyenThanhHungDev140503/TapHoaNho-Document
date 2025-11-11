# Advanced Refine Patterns - Senior Level

## Mục Lục

- [1. Custom Data Providers](#1-custom-data-providers)
- [2. Advanced Authentication](#2-advanced-authentication)
- [3. Real-time Updates](#3-real-time-updates)
- [4. Audit Logs](#4-audit-logs)
- [5. Advanced Access Control](#5-advanced-access-control)
- [6. Optimistic Updates](#6-optimistic-updates)
- [7. Custom Hooks](#7-custom-hooks)
- [8. Performance Optimization](#8-performance-optimization)

---

## 1. Custom Data Providers

### GraphQL Data Provider

```typescript
// Example 1: Custom GraphQL data provider
import { DataProvider } from "@refinedev/core";
import { GraphQLClient } from "graphql-request";
import * as gql from "gql-query-builder";

export const graphqlDataProvider = (client: GraphQLClient): DataProvider => ({
  getList: async ({ resource, pagination, sorters, filters, meta }) => {
    const { current = 1, pageSize = 10 } = pagination ?? {};
    
    const queryVariables: any = {
      offset: (current - 1) * pageSize,
      limit: pageSize,
    };
    
    // Build sorting
    if (sorters && sorters.length > 0) {
      queryVariables.orderBy = sorters.map((sorter) => ({
        [sorter.field]: sorter.order.toUpperCase(),
      }));
    }
    
    // Build filters
    if (filters && filters.length > 0) {
      queryVariables.where = filters.reduce((acc, filter) => {
        if (filter.operator === "eq") {
          acc[filter.field] = { _eq: filter.value };
        } else if (filter.operator === "contains") {
          acc[filter.field] = { _ilike: `%${filter.value}%` };
        } else if (filter.operator === "in") {
          acc[filter.field] = { _in: filter.value };
        }
        return acc;
      }, {} as any);
    }
    
    const { query, variables } = gql.query({
      operation: resource,
      variables: queryVariables,
      fields: meta?.fields ?? ["id", "name"],
    });
    
    const response = await client.request(query, variables);
    
    return {
      data: response[resource],
      total: response[`${resource}_aggregate`]?.aggregate?.count ?? 0,
    };
  },
  
  getOne: async ({ resource, id, meta }) => {
    const { query, variables } = gql.query({
      operation: resource,
      variables: {
        where: { id: { _eq: id } },
      },
      fields: meta?.fields ?? ["id", "name"],
    });
    
    const response = await client.request(query, variables);
    
    return {
      data: response[resource][0],
    };
  },
  
  create: async ({ resource, variables, meta }) => {
    const { query, variables: gqlVariables } = gql.mutation({
      operation: `insert_${resource}_one`,
      variables: {
        object: {
          value: variables,
          type: "object",
          required: true,
        },
      },
      fields: meta?.fields ?? ["id"],
    });
    
    const response = await client.request(query, gqlVariables);
    
    return {
      data: response[`insert_${resource}_one`],
    };
  },
  
  update: async ({ resource, id, variables, meta }) => {
    const { query, variables: gqlVariables } = gql.mutation({
      operation: `update_${resource}_by_pk`,
      variables: {
        pk_columns: {
          value: { id },
          type: "object",
          required: true,
        },
        _set: {
          value: variables,
          type: "object",
          required: true,
        },
      },
      fields: meta?.fields ?? ["id"],
    });
    
    const response = await client.request(query, gqlVariables);
    
    return {
      data: response[`update_${resource}_by_pk`],
    };
  },
  
  deleteOne: async ({ resource, id }) => {
    const { query, variables } = gql.mutation({
      operation: `delete_${resource}_by_pk`,
      variables: {
        id: {
          value: id,
          type: "uuid",
          required: true,
        },
      },
      fields: ["id"],
    });
    
    const response = await client.request(query, variables);
    
    return {
      data: response[`delete_${resource}_by_pk`],
    };
  },
  
  getApiUrl: () => client.url,
  
  custom: async ({ url, method, headers, payload }) => {
    const response = await client.request(payload as string);
    return { data: response };
  },
});

// Usage
import { GraphQLClient } from "graphql-request";

const client = new GraphQLClient("https://api.example.com/graphql", {
  headers: {
    Authorization: `Bearer ${localStorage.getItem("token")}`,
  },
});

<Refine
  dataProvider={graphqlDataProvider(client)}
  // ...
/>
```

### Multi-tenant Data Provider

```typescript
// Example 2: Multi-tenant data provider wrapper
import { DataProvider } from "@refinedev/core";

export const multiTenantDataProvider = (
  baseDataProvider: DataProvider,
  getTenantId: () => string | null
): DataProvider => {
  return {
    ...baseDataProvider,
    
    getList: async (params) => {
      const tenantId = getTenantId();
      
      if (!tenantId) {
        throw new Error("Tenant ID is required");
      }
      
      // Add tenant filter to all requests
      const filters = [
        ...(params.filters || []),
        {
          field: "tenantId",
          operator: "eq" as const,
          value: tenantId,
        },
      ];
      
      return baseDataProvider.getList({
        ...params,
        filters,
      });
    },
    
    create: async (params) => {
      const tenantId = getTenantId();
      
      if (!tenantId) {
        throw new Error("Tenant ID is required");
      }
      
      // Add tenant ID to all create operations
      return baseDataProvider.create({
        ...params,
        variables: {
          ...params.variables,
          tenantId,
        },
      });
    },
    
    // Similar for update, delete, etc.
  };
};

// Usage
const getTenantId = () => {
  const user = JSON.parse(localStorage.getItem("user") || "{}");
  return user.tenantId;
};

<Refine
  dataProvider={multiTenantDataProvider(
    dataProvider(API_URL),
    getTenantId
  )}
  // ...
/>
```

---

## 2. Advanced Authentication

### JWT Token Refresh

```typescript
// Example 3: Auth provider with token refresh
import { AuthProvider } from "@refinedev/core";
import axios from "axios";

let refreshTokenPromise: Promise<string> | null = null;

const refreshAccessToken = async (): Promise<string> => {
  if (refreshTokenPromise) {
    return refreshTokenPromise;
  }
  
  refreshTokenPromise = (async () => {
    try {
      const refreshToken = localStorage.getItem("refreshToken");
      
      const { data } = await axios.post("/api/refresh", {
        refreshToken,
      });
      
      localStorage.setItem("token", data.accessToken);
      localStorage.setItem("refreshToken", data.refreshToken);
      
      return data.accessToken;
    } finally {
      refreshTokenPromise = null;
    }
  })();
  
  return refreshTokenPromise;
};

// Setup axios interceptor
axios.interceptors.request.use(
  async (config) => {
    const token = localStorage.getItem("token");

    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }

    return config;
  },
  (error) => Promise.reject(error)
);

axios.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // If 401 and not already retried
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        const newToken = await refreshAccessToken();
        originalRequest.headers.Authorization = `Bearer ${newToken}`;

        return axios(originalRequest);
      } catch (refreshError) {
        // Refresh failed, logout user
        localStorage.removeItem("token");
        localStorage.removeItem("refreshToken");
        window.location.href = "/login";

        return Promise.reject(refreshError);
      }
    }

    return Promise.reject(error);
  }
);

export const authProvider: AuthProvider = {
  login: async ({ email, password }) => {
    const { data } = await axios.post("/api/login", { email, password });

    localStorage.setItem("token", data.accessToken);
    localStorage.setItem("refreshToken", data.refreshToken);
    localStorage.setItem("user", JSON.stringify(data.user));

    return {
      success: true,
      redirectTo: "/",
    };
  },

  logout: async () => {
    const refreshToken = localStorage.getItem("refreshToken");

    try {
      await axios.post("/api/logout", { refreshToken });
    } catch (error) {
      console.error("Logout error:", error);
    }

    localStorage.removeItem("token");
    localStorage.removeItem("refreshToken");
    localStorage.removeItem("user");

    return {
      success: true,
      redirectTo: "/login",
    };
  },

  check: async () => {
    const token = localStorage.getItem("token");

    if (!token) {
      return {
        authenticated: false,
        redirectTo: "/login",
        logout: true,
      };
    }

    try {
      // Verify token is still valid
      await axios.get("/api/verify");

      return {
        authenticated: true,
      };
    } catch (error) {
      return {
        authenticated: false,
        redirectTo: "/login",
        logout: true,
      };
    }
  },

  getIdentity: async () => {
    const user = localStorage.getItem("user");
    return user ? JSON.parse(user) : null;
  },

  onError: async (error) => {
    if (error.status === 401 || error.status === 403) {
      return {
        logout: true,
        redirectTo: "/login",
        error,
      };
    }

    return { error };
  },
};
```

### OAuth Authentication

```typescript
// Example 4: OAuth authentication with Google
import { AuthProvider } from "@refinedev/core";
import { GoogleAuthProvider, signInWithPopup, getAuth } from "firebase/auth";

const firebaseAuth = getAuth();
const googleProvider = new GoogleAuthProvider();

export const oauthAuthProvider: AuthProvider = {
  login: async ({ providerName }) => {
    if (providerName === "google") {
      try {
        const result = await signInWithPopup(firebaseAuth, googleProvider);
        const user = result.user;
        const token = await user.getIdToken();

        // Send token to your backend
        const { data } = await axios.post("/api/auth/google", { token });

        localStorage.setItem("token", data.accessToken);
        localStorage.setItem("user", JSON.stringify(data.user));

        return {
          success: true,
          redirectTo: "/",
        };
      } catch (error) {
        return {
          success: false,
          error: {
            name: "LoginError",
            message: "Google login failed",
          },
        };
      }
    }

    return {
      success: false,
      error: {
        name: "LoginError",
        message: "Unknown provider",
      },
    };
  },

  logout: async () => {
    await firebaseAuth.signOut();
    localStorage.removeItem("token");
    localStorage.removeItem("user");

    return {
      success: true,
      redirectTo: "/login",
    };
  },

  check: async () => {
    const token = localStorage.getItem("token");

    if (token) {
      return { authenticated: true };
    }

    return {
      authenticated: false,
      redirectTo: "/login",
      logout: true,
    };
  },

  getIdentity: async () => {
    const user = localStorage.getItem("user");
    return user ? JSON.parse(user) : null;
  },

  onError: async (error) => {
    if (error.status === 401) {
      return {
        logout: true,
        redirectTo: "/login",
        error,
      };
    }

    return { error };
  },
};

// Login page with OAuth
import { useLogin } from "@refinedev/core";

export const LoginPage = () => {
  const { mutate: login } = useLogin();

  return (
    <div>
      <h1>Login</h1>

      <button onClick={() => login({ providerName: "google" })}>
        Sign in with Google
      </button>
    </div>
  );
};
```

---

## 3. Real-time Updates

### Live Provider with Supabase

```typescript
// Example 5: Real-time updates with Supabase
import { LiveProvider } from "@refinedev/core";
import { RealtimeChannel } from "@supabase/supabase-js";
import { supabaseClient } from "./supabaseClient";

export const liveProvider = (client: typeof supabaseClient): LiveProvider => {
  const channels: Record<string, RealtimeChannel> = {};

  return {
    subscribe: ({ channel, types, params, callback }) => {
      const channelName = `${channel}:${params?.ids?.join(",")}`;

      if (!channels[channelName]) {
        channels[channelName] = client.channel(channelName);
      }

      const realtimeChannel = channels[channelName];

      types.forEach((type) => {
        switch (type) {
          case "created":
            realtimeChannel.on(
              "postgres_changes",
              {
                event: "INSERT",
                schema: "public",
                table: channel,
              },
              (payload) => {
                callback({
                  channel,
                  type: "created",
                  payload: {
                    ids: [payload.new.id],
                  },
                  date: new Date(),
                });
              }
            );
            break;

          case "updated":
            realtimeChannel.on(
              "postgres_changes",
              {
                event: "UPDATE",
                schema: "public",
                table: channel,
              },
              (payload) => {
                callback({
                  channel,
                  type: "updated",
                  payload: {
                    ids: [payload.new.id],
                  },
                  date: new Date(),
                });
              }
            );
            break;

          case "deleted":
            realtimeChannel.on(
              "postgres_changes",
              {
                event: "DELETE",
                schema: "public",
                table: channel,
              },
              (payload) => {
                callback({
                  channel,
                  type: "deleted",
                  payload: {
                    ids: [payload.old.id],
                  },
                  date: new Date(),
                });
              }
            );
            break;
        }
      });

      realtimeChannel.subscribe();

      return realtimeChannel;
    },

    unsubscribe: (channel) => {
      if (channel) {
        supabaseClient.removeChannel(channel);
      }
    },

    publish: async ({ channel, type, payload, date }) => {
      // Supabase handles publishing automatically
      return;
    },
  };
};

// Usage in App
import { Refine } from "@refinedev/core";
import { dataProvider, liveProvider } from "@refinedev/supabase";
import { supabaseClient } from "./supabaseClient";

<Refine
  dataProvider={dataProvider(supabaseClient)}
  liveProvider={liveProvider(supabaseClient)}
  options={{
    liveMode: "auto", // or "manual" or "off"
  }}
  // ...
/>

// Using live updates in components
import { useList } from "@refinedev/core";

export const ProductList = () => {
  const { data } = useList({
    resource: "products",
    liveMode: "auto", // Enable live updates for this component
  });

  // Data will automatically update when changes occur
  return (
    <ul>
      {data?.data.map((product) => (
        <li key={product.id}>{product.name}</li>
      ))}
    </ul>
  );
};
```

### Custom WebSocket Live Provider

```typescript
// Example 6: Custom WebSocket live provider
import { LiveProvider, LiveEvent } from "@refinedev/core";

export const websocketLiveProvider = (url: string): LiveProvider => {
  let ws: WebSocket | null = null;
  const subscribers = new Map<string, Set<(event: LiveEvent) => void>>();

  const connect = () => {
    ws = new WebSocket(url);

    ws.onopen = () => {
      console.log("WebSocket connected");
    };

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      const { channel, type, payload } = data;

      const channelSubscribers = subscribers.get(channel);

      if (channelSubscribers) {
        channelSubscribers.forEach((callback) => {
          callback({
            channel,
            type,
            payload,
            date: new Date(),
          });
        });
      }
    };

    ws.onerror = (error) => {
      console.error("WebSocket error:", error);
    };

    ws.onclose = () => {
      console.log("WebSocket disconnected");
      // Reconnect after 3 seconds
      setTimeout(connect, 3000);
    };
  };

  connect();

  return {
    subscribe: ({ channel, types, callback }) => {
      if (!subscribers.has(channel)) {
        subscribers.set(channel, new Set());

        // Send subscribe message to server
        ws?.send(JSON.stringify({
          action: "subscribe",
          channel,
          types,
        }));
      }

      subscribers.get(channel)?.add(callback);

      return {
        unsubscribe: () => {
          const channelSubscribers = subscribers.get(channel);
          channelSubscribers?.delete(callback);

          if (channelSubscribers?.size === 0) {
            subscribers.delete(channel);

            // Send unsubscribe message to server
            ws?.send(JSON.stringify({
              action: "unsubscribe",
              channel,
            }));
          }
        },
      };
    },

    unsubscribe: (subscription) => {
      subscription?.unsubscribe();
    },

    publish: async ({ channel, type, payload }) => {
      ws?.send(JSON.stringify({
        action: "publish",
        channel,
        type,
        payload,
      }));
    },
  };
};
```

---

## 4. Audit Logs

```typescript
// Example 7: Audit log provider
import { AuditLogProvider } from "@refinedev/core";
import axios from "axios";

export const auditLogProvider: AuditLogProvider = {
  create: async (params) => {
    const { resource, action, data, previousData, meta, author } = params;

    await axios.post("/api/audit-logs", {
      resource,
      action,
      data,
      previousData,
      meta,
      author,
      timestamp: new Date().toISOString(),
    });
  },

  get: async ({ resource, action, meta, author }) => {
    const { data } = await axios.get("/api/audit-logs", {
      params: {
        resource,
        action,
        meta,
        author,
      },
    });

    return data;
  },

  update: async ({ id, name }) => {
    const { data } = await axios.patch(`/api/audit-logs/${id}`, {
      name,
    });

    return data;
  },
};

// Usage in App
<Refine
  auditLogProvider={auditLogProvider}
  // ...
/>

// Audit logs are automatically created for CRUD operations
// You can also manually create audit logs
import { useLog } from "@refinedev/core";

export const CustomAction = () => {
  const { log } = useLog();

  const handleCustomAction = async () => {
    await someCustomOperation();

    // Log the custom action
    log({
      resource: "products",
      action: "custom-action",
      data: { customField: "value" },
      meta: { description: "Custom operation performed" },
    });
  };

  return <button onClick={handleCustomAction}>Custom Action</button>;
};

// View audit logs
import { useLogList } from "@refinedev/core";

export const AuditLogList = () => {
  const { data, isLoading } = useLogList({
    resource: "products",
  });

  if (isLoading) return <div>Loading...</div>;

  return (
    <table>
      <thead>
        <tr>
          <th>Action</th>
          <th>Author</th>
          <th>Timestamp</th>
          <th>Data</th>
        </tr>
      </thead>
      <tbody>
        {data?.data.map((log) => (
          <tr key={log.id}>
            <td>{log.action}</td>
            <td>{log.author?.name}</td>
            <td>{new Date(log.timestamp).toLocaleString()}</td>
            <td>{JSON.stringify(log.data)}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
};
```

---

## 5. Advanced Access Control

### Field-Level Access Control

```typescript
// Example 8: Field-level permissions
import { AccessControlProvider } from "@refinedev/core";

interface FieldPermissions {
  [resource: string]: {
    [field: string]: {
      [role: string]: {
        read: boolean;
        write: boolean;
      };
    };
  };
}

const fieldPermissions: FieldPermissions = {
  products: {
    price: {
      admin: { read: true, write: true },
      editor: { read: true, write: true },
      viewer: { read: true, write: false },
    },
    cost: {
      admin: { read: true, write: true },
      editor: { read: false, write: false },
      viewer: { read: false, write: false },
    },
  },
};

export const accessControlProvider: AccessControlProvider = {
  can: async ({ resource, action, params }) => {
    const user = JSON.parse(localStorage.getItem("user") || "{}");
    const userRole = user.role;

    // Resource-level permissions
    if (action === "delete" && userRole !== "admin") {
      return {
        can: false,
        reason: "Only admins can delete",
      };
    }

    // Field-level permissions
    if (params?.field) {
      const fieldPerm = fieldPermissions[resource]?.[params.field]?.[userRole];

      if (!fieldPerm) {
        return { can: false, reason: "No permission for this field" };
      }

      if (action === "field" && params.fieldAction === "read") {
        return {
          can: fieldPerm.read,
          reason: !fieldPerm.read ? "Cannot read this field" : undefined,
        };
      }

      if (action === "field" && params.fieldAction === "write") {
        return {
          can: fieldPerm.write,
          reason: !fieldPerm.write ? "Cannot edit this field" : undefined,
        };
      }
    }

    return { can: true };
  },
};

// Usage in components
import { useCan } from "@refinedev/core";

export const ProductForm = ({ product }: { product: IProduct }) => {
  const { data: canReadPrice } = useCan({
    resource: "products",
    action: "field",
    params: { field: "price", fieldAction: "read" },
  });

  const { data: canEditPrice } = useCan({
    resource: "products",
    action: "field",
    params: { field: "price", fieldAction: "write" },
  });

  const { data: canReadCost } = useCan({
    resource: "products",
    action: "field",
    params: { field: "cost", fieldAction: "read" },
  });

  return (
    <form>
      <div>
        <label>Name:</label>
        <input defaultValue={product.name} />
      </div>

      {canReadPrice?.can && (
        <div>
          <label>Price:</label>
          <input
            defaultValue={product.price}
            disabled={!canEditPrice?.can}
          />
        </div>
      )}

      {canReadCost?.can && (
        <div>
          <label>Cost:</label>
          <input defaultValue={product.cost} disabled />
        </div>
      )}
    </form>
  );
};
```

### Row-Level Security

```typescript
// Example 9: Row-level access control
export const rowLevelAccessControl: AccessControlProvider = {
  can: async ({ resource, action, params }) => {
    const user = JSON.parse(localStorage.getItem("user") || "{}");

    // Check if user owns the resource
    if (action === "edit" || action === "delete") {
      const recordOwnerId = params?.resource?.userId;

      if (recordOwnerId && recordOwnerId !== user.id && user.role !== "admin") {
        return {
          can: false,
          reason: "You can only modify your own records",
        };
      }
    }

    // Department-based access
    if (resource === "projects") {
      const recordDepartment = params?.resource?.department;

      if (recordDepartment && recordDepartment !== user.department && user.role !== "admin") {
        return {
          can: false,
          reason: "You can only access projects in your department",
        };
      }
    }

    return { can: true };
  },
};
```

---

## 6. Optimistic Updates

```typescript
// Example 10: Optimistic updates with rollback
import { useUpdate, useInvalidate } from "@refinedev/core";
import { useQueryClient } from "@tanstack/react-query";

export const ProductToggleActive = ({ product }: { product: IProduct }) => {
  const queryClient = useQueryClient();
  const invalidate = useInvalidate();
  const { mutate } = useUpdate();

  const handleToggle = () => {
    const queryKey = ["products", "list"];

    // Get current data
    const previousData = queryClient.getQueryData(queryKey);

    // Optimistically update UI
    queryClient.setQueryData(queryKey, (old: any) => {
      if (!old) return old;

      return {
        ...old,
        data: old.data.map((p: IProduct) =>
          p.id === product.id
            ? { ...p, isActive: !p.isActive }
            : p
        ),
      };
    });

    // Perform actual update
    mutate(
      {
        resource: "products",
        id: product.id,
        values: {
          isActive: !product.isActive,
        },
      },
      {
        onError: (error) => {
          // Rollback on error
          queryClient.setQueryData(queryKey, previousData);

          console.error("Update failed:", error);
          alert("Failed to update product");
        },
        onSuccess: () => {
          // Invalidate to ensure data is fresh
          invalidate({
            resource: "products",
            invalidates: ["list"],
          });
        },
      }
    );
  };

  return (
    <button onClick={handleToggle}>
      {product.isActive ? "Deactivate" : "Activate"}
    </button>
  );
};
```

---

## 7. Custom Hooks

### useResource Hook

```typescript
// Example 11: Custom hook for resource operations
import { useList, useCreate, useUpdate, useDelete } from "@refinedev/core";
import { useState } from "react";

export function useResource<T>(resource: string) {
  const [filters, setFilters] = useState<any[]>([]);
  const [sorters, setSorters] = useState<any[]>([]);

  const { data, isLoading, refetch } = useList<T>({
    resource,
    filters,
    sorters,
  });

  const { mutate: create, isLoading: isCreating } = useCreate();
  const { mutate: update, isLoading: isUpdating } = useUpdate();
  const { mutate: remove, isLoading: isDeleting } = useDelete();

  const handleCreate = (values: Partial<T>) => {
    create(
      {
        resource,
        values,
      },
      {
        onSuccess: () => refetch(),
      }
    );
  };

  const handleUpdate = (id: string, values: Partial<T>) => {
    update(
      {
        resource,
        id,
        values,
      },
      {
        onSuccess: () => refetch(),
      }
    );
  };

  const handleDelete = (id: string) => {
    remove(
      {
        resource,
        id,
      },
      {
        onSuccess: () => refetch(),
      }
    );
  };

  return {
    data: data?.data ?? [],
    total: data?.total ?? 0,
    isLoading,
    isCreating,
    isUpdating,
    isDeleting,
    filters,
    setFilters,
    sorters,
    setSorters,
    create: handleCreate,
    update: handleUpdate,
    delete: handleDelete,
    refetch,
  };
}

// Usage
export const ProductManager = () => {
  const {
    data: products,
    isLoading,
    create,
    update,
    delete: deleteProduct,
    setFilters,
  } = useResource<IProduct>("products");

  if (isLoading) return <div>Loading...</div>;

  return (
    <div>
      <button onClick={() => create({ name: "New Product", price: 0 })}>
        Add Product
      </button>

      <input
        placeholder="Search..."
        onChange={(e) =>
          setFilters([
            {
              field: "name",
              operator: "contains",
              value: e.target.value,
            },
          ])
        }
      />

      <ul>
        {products.map((product) => (
          <li key={product.id}>
            {product.name}
            <button onClick={() => update(product.id, { price: 99 })}>
              Update
            </button>
            <button onClick={() => deleteProduct(product.id)}>
              Delete
            </button>
          </li>
        ))}
      </ul>
    </div>
  );
};
```

### useModal Hook

```typescript
// Example 12: Custom modal hook for CRUD operations
import { useState } from "react";
import { useCreate, useUpdate } from "@refinedev/core";

export function useModalForm<T>(resource: string) {
  const [isOpen, setIsOpen] = useState(false);
  const [mode, setMode] = useState<"create" | "edit">("create");
  const [editingId, setEditingId] = useState<string | null>(null);
  const [initialValues, setInitialValues] = useState<Partial<T>>({});

  const { mutate: create, isLoading: isCreating } = useCreate();
  const { mutate: update, isLoading: isUpdating } = useUpdate();

  const openCreate = () => {
    setMode("create");
    setInitialValues({});
    setIsOpen(true);
  };

  const openEdit = (id: string, values: Partial<T>) => {
    setMode("edit");
    setEditingId(id);
    setInitialValues(values);
    setIsOpen(true);
  };

  const close = () => {
    setIsOpen(false);
    setEditingId(null);
    setInitialValues({});
  };

  const submit = (values: Partial<T>) => {
    if (mode === "create") {
      create(
        {
          resource,
          values,
        },
        {
          onSuccess: () => close(),
        }
      );
    } else if (mode === "edit" && editingId) {
      update(
        {
          resource,
          id: editingId,
          values,
        },
        {
          onSuccess: () => close(),
        }
      );
    }
  };

  return {
    isOpen,
    mode,
    initialValues,
    isLoading: isCreating || isUpdating,
    openCreate,
    openEdit,
    close,
    submit,
  };
}

// Usage
export const ProductList = () => {
  const modal = useModalForm<IProduct>("products");

  return (
    <div>
      <button onClick={modal.openCreate}>Create Product</button>

      {/* Product list */}
      <ul>
        {products.map((product) => (
          <li key={product.id}>
            {product.name}
            <button onClick={() => modal.openEdit(product.id, product)}>
              Edit
            </button>
          </li>
        ))}
      </ul>

      {/* Modal */}
      {modal.isOpen && (
        <Modal onClose={modal.close}>
          <h2>{modal.mode === "create" ? "Create" : "Edit"} Product</h2>
          <ProductForm
            initialValues={modal.initialValues}
            onSubmit={modal.submit}
            isLoading={modal.isLoading}
          />
        </Modal>
      )}
    </div>
  );
};
```

---

## 8. Performance Optimization

### Query Caching Strategy

```typescript
// Example 13: Custom caching configuration
import { QueryClient } from "@tanstack/react-query";

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Cache data for 5 minutes
      staleTime: 5 * 60 * 1000,

      // Keep unused data in cache for 10 minutes
      gcTime: 10 * 60 * 1000,

      // Retry failed requests 3 times
      retry: 3,

      // Refetch on window focus
      refetchOnWindowFocus: true,

      // Refetch on reconnect
      refetchOnReconnect: true,
    },
    mutations: {
      // Retry failed mutations once
      retry: 1,
    },
  },
});

// Usage in App
import { QueryClientProvider } from "@tanstack/react-query";

<QueryClientProvider client={queryClient}>
  <Refine
    // ...
  />
</QueryClientProvider>
```

### Prefetching Data

```typescript
// Example 14: Prefetch related data
import { useOne, useList } from "@refinedev/core";
import { useQueryClient } from "@tanstack/react-query";
import { useEffect } from "react";

export const ProductDetail = ({ id }: { id: string }) => {
  const queryClient = useQueryClient();

  const { data: product } = useOne<IProduct>({
    resource: "products",
    id,
  });

  // Prefetch related category
  useEffect(() => {
    if (product?.data.categoryId) {
      queryClient.prefetchQuery({
        queryKey: ["categories", "detail", product.data.categoryId],
        queryFn: () =>
          fetch(`/api/categories/${product.data.categoryId}`).then((res) =>
            res.json()
          ),
      });
    }
  }, [product?.data.categoryId, queryClient]);

  return <div>{/* Product details */}</div>;
};
```

### Pagination Optimization

```typescript
// Example 15: Optimized pagination with prefetching
import { useTable } from "@refinedev/core";
import { useQueryClient } from "@tanstack/react-query";
import { useEffect } from "react";

export const ProductTable = () => {
  const queryClient = useQueryClient();

  const {
    tableQuery: { data },
    current,
    setCurrent,
    pageSize,
  } = useTable<IProduct>({
    resource: "products",
    pagination: {
      current: 1,
      pageSize: 20,
    },
  });

  // Prefetch next page
  useEffect(() => {
    const nextPage = current + 1;

    queryClient.prefetchQuery({
      queryKey: ["products", "list", { page: nextPage, pageSize }],
      queryFn: () =>
        fetch(
          `/api/products?_start=${(nextPage - 1) * pageSize}&_end=${nextPage * pageSize}`
        ).then((res) => res.json()),
    });
  }, [current, pageSize, queryClient]);

  return (
    <div>
      {/* Table */}
      <table>
        {/* ... */}
      </table>

      {/* Pagination */}
      <button onClick={() => setCurrent(current - 1)} disabled={current === 1}>
        Previous
      </button>
      <button onClick={() => setCurrent(current + 1)}>Next</button>
    </div>
  );
};
```

---

## Best Practices - Senior Level

### 1. Data Provider Organization

```typescript
// ✅ GOOD: Modular data provider
// dataProviders/rest.ts
export const restDataProvider = createDataProvider(API_URL);

// dataProviders/graphql.ts
export const graphqlDataProvider = createGraphQLDataProvider(GRAPHQL_URL);

// dataProviders/index.ts
export const dataProviders = {
  default: restDataProvider,
  graphql: graphqlDataProvider,
};

// ❌ BAD: Everything in one file
const dataProvider = { /* 1000+ lines */ };
```

### 2. Error Handling

```typescript
// ✅ GOOD: Comprehensive error handling
const { mutate, error } = useCreate();

mutate(
  { resource: "products", values },
  {
    onError: (error) => {
      if (error.statusCode === 422) {
        // Validation error
        showValidationErrors(error.errors);
      } else if (error.statusCode === 403) {
        // Permission error
        showPermissionError();
      } else {
        // Generic error
        showGenericError(error.message);
      }
    },
  }
);

// ❌ BAD: No error handling
mutate({ resource: "products", values });
```

### 3. Type Safety

```typescript
// ✅ GOOD: Strict typing
interface IProduct {
  id: string;
  name: string;
  price: number;
  categoryId: string;
}

const { data } = useList<IProduct>({ resource: "products" });

// ❌ BAD: No types
const { data } = useList({ resource: "products" });
```

---

## Tài Liệu Tham Khảo

### Official Documentation
- [Refine Advanced Guides](https://refine.dev/docs/guides-concepts/)
- [Data Provider Interface](https://refine.dev/docs/data/data-provider/)
- [Auth Provider](https://refine.dev/docs/authentication/auth-provider/)

### Next Level
- Xem `Principal-Refine-Patterns.md` cho Enterprise patterns

---

**Tài liệu này bao gồm Senior level patterns. Để học các patterns enterprise, hãy tiếp tục với Principal-Refine-Patterns.md.**
