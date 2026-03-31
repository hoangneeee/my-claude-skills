---
name: mbfs-aiteam-fe
description: Frontend development guide for MBFS AI Team projects using React 19, Shadcn UI (new-york), Tailwind CSS, Zustand, React Hook Form, TanStack Query, and Axios. Use when creating components, pages, API services, nav items, or managing permissions/auth guards.
allowed-tools: Read, Glob, Grep, Write, Edit, Bash
---

# MBFS AI Team Frontend Development Guide

## Stack

- **React 19** + **TypeScript** (strict)
- **Shadcn UI** (new-york style) — components in `src/ui/`
- **Tailwind CSS 4** — CSS variables for theming
- **Zustand** — state management with localStorage persistence
- **React Hook Form** — form handling
- **TanStack Query v5** — server state / data fetching
- **Axios** — HTTP client via `src/api/apiClient.ts`
- **React Router v7** — routing
- **i18next** — translations (use i18n keys, not raw strings)
- **pnpm** — package manager

---

## 1. Component Pattern

Every component is a functional component with TypeScript. Shadcn imports from `@/ui/*`, icons use local SVGs.

```tsx
// src/components/<name>/<name>.tsx
import { Button } from "@/ui/button";
import { Card, CardContent, CardHeader, CardTitle } from "@/ui/card";
import { Icon } from "@/components/icon";

export type CameraCardProps = {
  cameraId: string;
  name: string;
  onSelect: (id: string) => void;
};

export default function CameraCard({ cameraId, name, onSelect }: CameraCardProps) {
  return (
    <Card>
      <CardHeader>
        <CardTitle className="flex items-center gap-2">
          <Icon icon="local:ic-camera" size="20" />
          {name}
        </CardTitle>
      </CardHeader>
      <CardContent>
        <Button onClick={() => onSelect(cameraId)}>Select</Button>
      </CardContent>
    </Card>
  );
}
```

**Rules:**

- Always use `@/ui/*` for shadcn, never `@/components/ui/*`
- Prefer `local:ic-<name>` icons over library icons
- Use Tailwind classes for styling — no inline styles
- Type all props explicitly

---

## 2. Form Pattern (React Hook Form + Shadcn)

```tsx
import { useForm } from "react-hook-form";
import { Button } from "@/ui/button";
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from "@/ui/form";
import { Input } from "@/ui/input";

type FormValues = {
  name: string;
  email: string;
};

export default function ExampleForm({ onSubmit }: { onSubmit: (v: FormValues) => void }) {
  const form = useForm<FormValues>({
    defaultValues: { name: "", email: "" },
  });

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        <FormField
          control={form.control}
          name="name"
          rules={{ required: "Name is required" }}
          render={({ field }) => (
            <FormItem>
              <FormLabel>Name</FormLabel>
              <FormControl>
                <Input placeholder="Enter name" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <Button type="submit">Submit</Button>
      </form>
    </Form>
  );
}
```

---

## 3. Page / Feature Structure

When creating a new feature, follow this folder structure:

```
src/pages/<feature>/
├── index.tsx          # Main list/dashboard page
├── detail.tsx         # Detail view (optional)
├── filter.tsx         # Filter bar component (optional)
└── <feature>-modal.tsx  # Create/Edit modal
```

**Page with data fetching:**

```tsx
// src/pages/camera/index.tsx
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import cameraService from "@/api/services/cameraService";

export default function CameraPage() {
  const queryClient = useQueryClient();

  const { data, isLoading } = useQuery({
    queryKey: ["cameras"],
    queryFn: () => cameraService.listCameras(),
  });

  const { mutate: createCamera } = useMutation({
    mutationFn: cameraService.createCamera,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["cameras"] });
    },
  });

  if (isLoading) return <LoadingScreen />;

  return (
    <div>
      {data?.list.map((cam) => <CameraCard key={cam.id} {...cam} />)}
    </div>
  );
}
```

---

## 4. API Service Pattern

```tsx
// src/api/services/cameraService.ts
import apiClient from "@/api/apiClient";
import type { Camera, CreateCameraRequest, PaginationParams } from "@/types";

enum CameraApi {
  List   = "/camera-manager/api/camera",
  Create = "/camera-manager/api/camera/create",
  Update = "/camera-manager/api/camera/update",
  Delete = "/camera-manager/api/camera/delete",
}

const listCameras = (params?: PaginationParams) =>
  apiClient.get<{ list: Camera[]; total: number }>({ url: CameraApi.List, params });

const createCamera = (data: CreateCameraRequest) =>
  apiClient.post<Camera>({ url: CameraApi.Create, data });

const updateCamera = (data: Partial<Camera> & { id: string }) =>
  apiClient.put<Camera>({ url: CameraApi.Update, data });

const deleteCamera = (id: string) =>
  apiClient.delete({ url: CameraApi.Delete, params: { id } });

export default { listCameras, createCamera, updateCamera, deleteCamera };
```

**Rules:**

- Use enum for all API URLs
- Pass params as `{ url, params }`, body as `{ url, data }`
- Export a default object with all methods

---

## 5. Nav Data Frontend

Nav items are defined in `src/layouts/dashboard/nav/nav-data/nav-data-frontend.tsx`.

**NavItemDataProps structure:**

```ts
type NavItemDataProps = {
  title: string;           // i18n key, e.g. "sys.nav.camera"
  path: string;            // route path, e.g. "/management/camera"
  icon?: ReactNode;        // use <Icon icon="local:ic-xxx" size="24" />
  auth?: string[];         // permission keys — user needs ANY one to see this item
  caption?: string;        // i18n key for tooltip
  children?: NavItemDataProps[];  // nested items (unlimited depth)
  hidden?: boolean;
  disabled?: boolean;
};
```

**Adding a new menu item:**

```tsx
// src/layouts/dashboard/nav/nav-data/nav-data-frontend.tsx
import { PERMISSIONS } from "@/constant/permisson";
import { Icon } from "@/components/icon";

export const frontendNavData: NavProps["data"] = [
  {
    name: "sys.nav.management",     // Group label (i18n key)
    items: [
      {
        title: "sys.nav.camera",
        path: "/management/camera",
        icon: <Icon icon="local:ic-camera" size="24" />,
        auth: [PERMISSIONS.CAMERA_READ],   // hide if user lacks this permission
        children: [
          {
            title: "sys.nav.camera_list",
            path: "/management/camera/list",
            // no icon needed for children
          },
          {
            title: "sys.nav.camera_config",
            path: "/management/camera/config",
            auth: [PERMISSIONS.CAMERA_MANAGE],   // stricter permission for sub-item
          },
        ],
      },
    ],
  },
];
```

**How it works:**

- `useFilteredNavData()` in `src/layouts/dashboard/nav/nav-data/index.ts` auto-filters items based on user permissions
- If a parent has children and ALL children are filtered out, the parent is also hidden
- Route mode: `GLOBAL_CONFIG.routerMode === "backend"` uses API-driven menu instead

**Adding translation key:**

```json
// src/locales/vi.json (and en.json)
{
  "sys": {
    "nav": {
      "camera": "Quản lý Camera",
      "camera_list": "Danh sách camera",
      "camera_config": "Cấu hình camera"
    }
  }
}
```

---

## 6. AuthGuard — Bảo vệ Route và Component

### 6a. Route-level protection

Routes are automatically protected in `src/layouts/dashboard/main.tsx`. It reads the `auth` field from the matching nav item and wraps the page:

```tsx
// Automatic — no action needed if nav item has auth set correctly
<AuthGuard checkAny={currentNavAuth} fallback={<Page403 />}>
  <Outlet />
</AuthGuard>
```

If a nav item has `auth: [PERMISSIONS.CAMERA_READ]`, any user accessing that route without the permission sees `Page403`.

### 6b. Component/UI-level protection

Use `<AuthGuard>` to show/hide UI blocks:

```tsx
import AuthGuard from "@/components/auth/auth-guard";
import { PERMISSIONS } from "@/constant/permisson";

export default function CameraPage() {
  return (
    <div>
      <CameraList />

      {/* Only show create button if user has CAMERA_MANAGE permission */}
      <AuthGuard checkAny={[PERMISSIONS.CAMERA_MANAGE]}>
        <Button>Add Camera</Button>
      </AuthGuard>
    </div>
  );
}
```

### 6c. Inline permission check with hook

```tsx
import { useAuthCheck } from "@/components/auth/use-auth";
import { PERMISSIONS } from "@/constant/permisson";

export default function CameraTable() {
  const { check, checkAny } = useAuthCheck();  // default: permission-based

  const canManage = check(PERMISSIONS.CAMERA_MANAGE);
  const canReadAny = checkAny([PERMISSIONS.CAMERA_READ, PERMISSIONS.CAMERA_MANAGE]);

  return (
    <table>
      {rows.map((row) => (
        <tr key={row.id}>
          <td>{row.name}</td>
          {canManage && (
            <td>
              <Button variant="ghost" size="sm">Edit</Button>
              <Button variant="destructive" size="sm">Delete</Button>
            </td>
          )}
        </tr>
      ))}
    </table>
  );
}
```

**`useAuthCheck` API:**

| Method | Description |
|--------|-------------|
| `check(permission)` | User has this exact permission |
| `checkAny(permissions[])` | User has at least ONE permission |
| `checkAll(permissions[])` | User has ALL permissions |

Pass `"role"` to check by role instead: `useAuthCheck("role")`.

### 6d. Adding a new permission constant

```ts
// src/constant/permisson.ts
export const PERMISSIONS = {
  // ... existing
  CAMERA_EXPORT: "camera.export",   // add new permission here
} as const;
```

---

## 7. Zustand Store Pattern

```tsx
// src/store/cameraStore.ts
import { create } from "zustand";
import { persist } from "zustand/middleware";

type CameraStore = {
  selectedCameraId: string | null;
  actions: {
    setSelectedCamera: (id: string | null) => void;
  };
};

const useCameraStore = create<CameraStore>()(
  persist(
    (set) => ({
      selectedCameraId: null,
      actions: {
        setSelectedCamera: (id) => set({ selectedCameraId: id }),
      },
    }),
    {
      name: "camera-store",
      partialize: (state) => ({ selectedCameraId: state.selectedCameraId }),
    },
  ),
);

// Export selector hooks (don't export the store directly for use in components)
export const useSelectedCameraId = () => useCameraStore((s) => s.selectedCameraId);
export const useCameraActions = () => useCameraStore((s) => s.actions);
```

---

## 8. Adding a New Route

```tsx
// src/routes/sections/dashboard.tsx
const CameraPage = lazy(() => import("@/pages/camera"));

export const dashboardRoutes: RouteObject[] = [
  {
    element: (
      <LoginAuthGuard>
        <DashboardLayout />
      </LoginAuthGuard>
    ),
    children: [
      // ... existing routes
      {
        path: "management/camera",
        element: (
          <Suspense fallback={<LoadingScreen />}>
            <CameraPage />
          </Suspense>
        ),
      },
    ],
  },
];
```

---

## 9. Conventions Checklist

- [ ] Use `@/ui/*` for all shadcn imports
- [ ] Use `local:ic-<name>` for icons (not lucide/iconify library icons)
- [ ] Use i18n keys for all user-visible strings (never hardcode text)
- [ ] Enum for API URLs in service files
- [ ] `auth` field on nav items maps to `PERMISSIONS.*` constants
- [ ] Run `pnpm run lint` with Biome before committing
- [ ] Commit messages follow conventional commits (`feat:`, `fix:`, `refactor:`, etc.)
