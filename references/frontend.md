# Frontend Standards

## Canonical Stack

| Area | Standard | Do not use |
|------|----------|------------|
| UI | **shadcn/ui** (Radix primitives, `src/components/ui/`) | MUI, Ant Design, Chakra |
| Styling | **Tailwind CSS v4** (`@import "tailwindcss"`, `@theme`) | CSS Modules-only, styled-components |
| Forms | **react-hook-form + Zod** via shadcn `<Form>` / `<FormField>` (`zodResolver`) | Yup, Formik, uncontrolled ad-hoc validation |
| HTTP | **Axios** singleton with typed API modules | Raw `fetch` in components |
| Client state | **Redux Toolkit** slices for session, UI, cross-feature | Prop drilling, multiple stores |
| Server state | **TBD** — RTK Query recommended; TanStack Query acceptable | Ad-hoc `useEffect` + Axios per screen |
| Auth tokens | Access token **in memory**; refresh via **httpOnly cookie** | localStorage/sessionStorage for tokens |
| Routing | **react-router-dom v6+** with `React.lazy` + `Suspense` | Loading entire bundle |
| Tables | **@tanstack/react-table** + shadcn Table primitives | Hand-rolled `<table>` sorting/filter |
| Dates | **date-fns** with `en-IN` locale | `moment`, manual locale |
| Icons | **lucide-react** | |
| Toasts | **sonner** | |
| Testing | **Vitest + React Testing Library** | Jest for new code |
| E2E | **Playwright** for critical flows | Manual-only QA |
| Path aliases | `@/` → `src/` (Vite + tsconfig) | Deep `../../../` imports |
| Env vars | `VITE_*` only; no secrets in FE bundle | |
| Error UX | Route `ErrorBoundary` + Axios mapper → sonner | Per-page alert divs |
| Accessibility | **WCAG 2.1 AA** — see [accessibility.md](accessibility.md) | Icon-only buttons without `aria-label`; `div` click handlers |
| Responsive | **Mobile-first**; layouts work on phone, tablet, and desktop | Desktop-only fixed widths, horizontal scroll on mobile |
| Theme modes | **Light + dark** supported; **light default**; user toggle persisted | Dark-only UI, no theme tokens |
| UX quality | Clean, professional, easy navigation; clear hierarchy and feedback | Cluttered layouts, cryptic labels, inconsistent patterns |
| Package manager | **pnpm** workspaces | Mixing npm/pnpm |
| Theming | **Axis burgundy `#971237` accent** → `--primary`; tokens in `globals.css` only | Hardcoded hex in components |

---

## Project Structure

```
thrivehr-ui/
├── src/
│   ├── app/
│   │   ├── App.tsx
│   │   ├── ErrorBoundary.tsx
│   │   └── providers.tsx              # Redux + Router + ThemeProvider (light default)
│   ├── components/
│   │   ├── layout/                    # AppShell, Sidebar, MobileNav, PageHeader
│   │   ├── theme/
│   │   │   └── theme-toggle.tsx       # Light / dark switch in header
│   │   └── ui/                        # shadcn/ui (CLI-added, customized in-repo)
│   ├── api/
│   │   ├── client.ts                  # Axios singleton (see assets/axios-client.template.ts)
│   │   ├── error-handler.ts           # Envelope error → sonner mapping
│   │   └── types/
│   │       └── envelope.ts            # Response envelope types
│   ├── features/
│   │   └── {domain}/
│   │       ├── components/
│   │       ├── hooks/
│   │       ├── api.ts                 # Domain-specific API calls
│   │       └── types.ts
│   ├── store/
│   │   ├── index.ts                   # configureStore
│   │   └── slices/
│   │       └── {domain}.slice.ts
│   ├── routes/
│   │   ├── index.tsx                  # Lazy routes + Suspense
│   │   └── protected-route.tsx
│   ├── lib/
│   │   └── utils.ts                   # cn() helper, shared utilities
│   └── styles/
│       └── globals.css                # Tailwind v4 @theme + shadcn tokens
├── e2e/                               # Playwright tests
├── public/
├── .env.example
├── index.html
├── vite.config.ts
├── tsconfig.json
├── tailwind.config.ts
└── package.json
```

---

## shadcn/ui Conventions

- Add components via CLI: `npx shadcn@latest add button` → `src/components/ui/button.tsx`
- Components are **owned by the project** — customize freely, do not treat as external npm package
- Use `cn()` (`clsx` + `tailwind-merge`) for all class composition:

```typescript
import { cn } from '@/lib/utils';
<div className={cn('base-class', conditional && 'conditional-class')} />
```

- **Loading states**: use shadcn `Skeleton` components
- **Empty states**: dedicated component pattern with icon + message + optional action
- **Toasts**: use `sonner` — `toast.success()`, `toast.error()` — never `alert()` or `window.confirm()`

---

## Theming (Tailwind v4 + shadcn)

### Axis Bank brand palette

**Accent color is Axis burgundy** — use for primary actions, links, focus rings, key highlights, and `--primary` in shadcn. Confirm exact hex with the official *Axis Bank Core Brand Guidelines* PDF if design provides an updated value.

| Token | Hex | Usage |
|-------|-----|--------|
| **burgundy (accent)** | `#971237` | Primary buttons, links, active nav, brand accents, `bg-primary` |
| burgundy dark | `#7A0E2C` | Hover / pressed states on primary controls |
| burgundy light | `#F5E6EB` | Subtle tinted backgrounds, selected rows, badges |
| white | `#FFFFFF` | Page backgrounds, cards on tinted layouts |
| charcoal | `#1A1A1A` | Primary body text |
| slate | `#4A4A4A` | Secondary text, captions |
| border | `#E5E5E5` | Dividers, input borders |
| success | `#0D7A3E` | Success states (not brand, but accessible green) |
| warning | `#B45309` | Warnings |
| destructive | `#B91C1C` | Errors, destructive actions |

Common alternate from public brand references: maroon `#800000` / `#891B3F`. **Default in this project: `#971237` (burgundy accent)** unless design overrides.

### `globals.css` — Tailwind v4 `@theme` + shadcn variables

Single source of truth: `src/styles/globals.css` (or `src/styles/theme.css` imported from globals). **Never hardcode hex in components** — use `bg-primary`, `text-primary`, `border-primary`, or semantic tokens.

```css
@import "tailwindcss";

/* Tailwind v4 theme tokens */
@theme {
  --color-axis-burgundy: #971237;
  --color-axis-burgundy-dark: #7a0e2c;
  --color-axis-burgundy-light: #f5e6eb;
  --color-axis-charcoal: #1a1a1a;
  --color-axis-slate: #4a4a4a;
  --color-axis-border: #e5e5e5;

  --color-primary: var(--color-axis-burgundy);
  --color-primary-foreground: #ffffff;
  --radius-default: 0.5rem;
}

/* shadcn/ui (HSL) — map accent burgundy to --primary */
:root {
  --background: 0 0% 100%;
  --foreground: 0 0% 10%;

  --primary: 345 78% 33%;           /* ~#971237 burgundy */
  --primary-foreground: 0 0% 100%;

  --secondary: 345 45% 95%;         /* burgundy-light tint */
  --secondary-foreground: 345 78% 25%;

  --muted: 0 0% 96%;
  --muted-foreground: 0 0% 29%;

  --accent: 345 45% 95%;
  --accent-foreground: 345 78% 33%;

  --destructive: 0 72% 41%;
  --destructive-foreground: 0 0% 100%;

  --border: 0 0% 90%;
  --input: 0 0% 90%;
  --ring: 345 78% 33%;              /* focus ring = burgundy */

  --radius: 0.5rem;
}

.dark {
  --background: 0 0% 7%;
  --foreground: 0 0% 95%;

  --primary: 345 70% 45%;             /* burgundy accent — slightly lighter on dark */
  --primary-foreground: 0 0% 100%;

  --secondary: 345 20% 18%;
  --secondary-foreground: 0 0% 95%;

  --muted: 0 0% 15%;
  --muted-foreground: 0 0% 65%;

  --accent: 345 20% 18%;
  --accent-foreground: 345 70% 55%;

  --destructive: 0 62% 50%;
  --destructive-foreground: 0 0% 100%;

  --border: 0 0% 20%;
  --input: 0 0% 20%;
  --ring: 345 70% 45%;
}
```

### Light / dark mode (required)

- **Both themes required** on every new screen and shared component.
- **Default: light** — app loads in light mode unless the user previously chose dark.
- Use **`next-themes`** (works with Vite + React) with `attribute="class"` on `<html>`:

```typescript
// src/app/providers.tsx
import { ThemeProvider } from 'next-themes';

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <ThemeProvider attribute="class" defaultTheme="light" enableSystem={false} storageKey="thrivehr-theme">
      {children}
    </ThemeProvider>
  );
}
```

- Add shadcn **theme toggle** in app header (sun/moon); persist via `next-themes` + `localStorage`.
- Test all new UI in **both** modes before merge — burgundy accent and contrast must pass WCAG AA in each.
- Charts, images, and third-party embeds must remain readable in dark mode.

### Usage rules

- **Primary CTA / links**: `variant="default"` (shadcn Button) → burgundy via `--primary`
- **Do not** use burgundy for large body text blocks (readability)
- **Contrast**: burgundy on white must meet **WCAG 2.1 AA** (4.5:1 for normal text); use white text on burgundy buttons
- **Charts / data viz**: burgundy as series accent; pair with neutral grays, not competing reds
- **Logos / marketing assets**: follow brand PDF; UI tokens above are for product chrome only

---

## Responsive design (mobile + desktop)

Every screen and shared component must work from **~320px (mobile)** through **desktop (1280px+)**. Design **mobile-first**, then enhance with `md:` / `lg:` breakpoints.

### Layout rules

| Pattern | Mobile | Tablet / desktop |
|---------|--------|------------------|
| App shell | Collapsible **sheet/drawer** nav or bottom nav; compact header | Persistent **sidebar** + top bar |
| Page padding | `px-4 py-4` | `px-6 lg:px-8`, `max-w-7xl mx-auto` for content |
| Grids | Single column (`grid-cols-1`) | `md:grid-cols-2`, `lg:grid-cols-3` as needed |
| Tables | Card list or **horizontal scroll** wrapper with hint; avoid tiny unreadable columns | Full table with TanStack Table |
| Forms | Full-width inputs; sticky primary action footer optional | Two-column forms only when labels stay aligned |
| Modals / dialogs | Full-screen or near full-screen on small viewports | Centered dialog `max-w-lg` / `max-w-2xl` |
| Touch targets | Min **44×44px** tap areas for buttons and links | Same (accessibility) |

### Tailwind breakpoints (default)

Use Tailwind defaults unless project config changes: `sm` 640px, `md` 768px, `lg` 1024px, `xl` 1280px.

```tsx
<div className="flex flex-col gap-4 md:flex-row md:items-center md:justify-between">
  <h1 className="text-xl font-semibold md:text-2xl">Job listings</h1>
  <Button className="w-full md:w-auto">Create job</Button>
</div>
```

### Verification

- Resize browser or use DevTools device toolbar for mobile / tablet / desktop.
- Playwright: include at least one **mobile viewport** smoke test for critical flows.
- No clipped content, no unusable horizontal scroll except intentional table wrappers.

---

## UI/UX standards (clean, professional, usable)

All ThriveHR UI should feel **banking-grade**: calm, clear, and easy to scan — not flashy or cluttered.

### Navigation & information architecture

- **Obvious primary action** per screen (one burgundy CTA where applicable).
- **Consistent shell**: logo area, nav, user menu, breadcrumbs on deep pages (`md+`).
- **Page titles** + short description where context helps (especially admin flows).
- **Back / cancel** paths always visible on multi-step flows (apply, upload, onboarding).
- Group related actions; avoid more than one primary button per section.

### Clarity & feedback

- **Labels** on every input; placeholder is not a label.
- **Empty states**: explain what is missing and what to do next (with action).
- **Loading**: Skeleton or spinner — never a blank screen during fetch.
- **Errors**: Human-readable copy via sonner + inline field errors on forms.
- **Success**: Brief confirmation toast after create/update/delete.

### Visual design

- Generous **whitespace**; avoid dense walls of fields or table columns.
- **Typography hierarchy**: page title → section heading → body; limit font sizes (e.g. `text-sm` / `text-base` / `text-lg` / `text-2xl`).
- **Alignment**: left-align text; right-align numbers in tables.
- **Icons** (lucide): reinforce meaning, don’t replace labels on primary actions.
- Stay **professional**: no playful animations, meme copy, or neon colors outside the brand palette.

### Usability checklist (per feature PR)

- [ ] Works on mobile and desktop
- [ ] Works in light and dark mode
- [ ] Keyboard navigable; focus visible (burgundy ring)
- [ ] Screen reader labels on icon-only controls
- [ ] Primary flow completable without hunting for controls

Full WCAG checklist, tooling, and patterns: **[accessibility.md](accessibility.md)**.

---

## Redux Toolkit

- One store at `src/store/index.ts` using `configureStore`
- Feature slices at `src/store/slices/{domain}.slice.ts`
- Use `createSlice` with `extraReducers` for async operations
- Avoid prop drilling — connect via `useSelector` / `useDispatch` (typed hooks)

```typescript
// src/store/hooks.ts
import { useDispatch, useSelector } from 'react-redux';
import type { RootState, AppDispatch } from './index';

export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
export const useAppSelector = useSelector.withTypes<RootState>();
```

---

## React Hook Form + Zod

All forms use `react-hook-form` with `@hookform/resolvers/zod` and shadcn `Form` components. Use **Zod** on the frontend (same library as Fastify backend) — prefer shared schemas in a `packages/shared` or `src/schemas/` module when they mirror API request bodies.

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { Form, FormField, FormItem, FormLabel, FormControl, FormMessage } from '@/components/ui/form';

const schema = z.object({
  email: z.string().email(),
  name: z.string().min(2),
});

type FormValues = z.infer<typeof schema>;

function MyForm() {
  const form = useForm<FormValues>({
    resolver: zodResolver(schema),
    defaultValues: { email: '', name: '' },
  });

  const onSubmit = form.handleSubmit((data) => { /* call API */ });

  return (
    <Form {...form}>
      <form onSubmit={onSubmit}>
        <FormField control={form.control} name="email" render={({ field }) => (
          <FormItem>
            <FormLabel>Email</FormLabel>
            <FormControl><Input {...field} /></FormControl>
            <FormMessage />
          </FormItem>
        )} />
      </form>
    </Form>
  );
}
```

---

## Axios Layer

Use a single Axios instance configured in `src/api/client.ts`. See [assets/axios-client-template.md](../assets/axios-client-template.md) for the full template.

Key requirements:
- Base URL from `VITE_API_BASE_URL`
- `withCredentials: true` for httpOnly cookie refresh flow
- Request interceptor: attach access token + `X-Correlation-Id`
- Response interceptor: unwrap envelope, handle 401 refresh, map errors to sonner
- Timeout: configurable (default 30s)

Pagination types in `src/api/types/envelope.ts`:

```typescript
export interface PaginationMeta {
  currentPage: number;
  totalPages: number;
  totalItems: number;
  hasNext: boolean;
  hasPrevious: boolean;
  pageSize: number;
}
```

Per-domain API modules:

```typescript
// src/features/jobs/api.ts
import { apiClient } from '@/api/client';
import type { Job } from './types';

export const jobsApi = {
  list: (params: ListParams) => apiClient.get<Job[]>('/job/api/v1/listings', { params }),
  getById: (id: string) => apiClient.get<Job>(`/job/api/v1/listings/${id}`),
  create: (data: CreateJobInput) => apiClient.post<Job>('/job/api/v1/listings', data),
};
```

---

## Routing (Lazy Loading)

```typescript
// src/routes/index.tsx
import { lazy, Suspense } from 'react';
import { createBrowserRouter } from 'react-router-dom';

const Dashboard = lazy(() => import('@/features/dashboard/pages/DashboardPage'));
const Jobs = lazy(() => import('@/features/jobs/pages/JobsPage'));

export const router = createBrowserRouter([
  {
    path: '/',
    element: <ProtectedRoute />,
    errorElement: <ErrorBoundary />,
    children: [
      { index: true, element: <Suspense fallback={<Skeleton />}><Dashboard /></Suspense> },
      { path: 'jobs', element: <Suspense fallback={<Skeleton />}><Jobs /></Suspense> },
    ],
  },
]);
```

---

## TanStack Table

All dashboard/data tables use `@tanstack/react-table` with shadcn Table primitives.

- Column definitions in feature folder
- Sorting, filtering, pagination aligned with backend `meta.pagination` (`currentPage`, `pageSize`, `totalItems`, `totalPages`, `hasNext`, `hasPrevious`); request query params remain `page` and `limit`
- Server-side pagination by default for large datasets

---

## File Upload (Presigned URL Flow)

1. Request presigned URL from backend: `POST /{service}/api/v1/uploads/presign`
2. Upload file directly to S3 via `PUT` with progress tracking (`onUploadProgress`)
3. Poll virus scan status: `GET /{service}/api/v1/uploads/{id}/status`
4. Retry with exponential backoff on transient failures
5. Show progress bar + scan status in UI

---

## Error Handling

- Route-level `ErrorBoundary` catches React render errors
- Axios response interceptor maps envelope errors to sonner toasts
- Standard empty-state component for no-data scenarios
- Never show raw error objects to users — map all codes from `core.md` error table

---

## Deferred Decisions

| Decision | Defer until |
|----------|-------------|
| i18n (`react-i18next`) | Product confirms multi-language |
| Storybook | Design system stabilizes |
| Motion library | Specific marketing animations scoped |
| Micro-frontends | Multiple independent deployable UIs needed |
