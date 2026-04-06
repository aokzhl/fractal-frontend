# Layer Structure Reference

Detailed folder structures, code examples, and naming conventions for each
Fractal FSD layer. Use this reference when creating, reviewing, or
reorganizing project structure.

---

## App Layer

Application shell: providers, routing, layouts, global styles, entry point.
Organized by segments or functional groups — no slices.

```text
app/
  layouts/         ← Layout wrappers (sidebar + outlet, auth layout, etc.)
  providers/       ← Global providers (state, theme, etc.)
  styles/          ← Global CSS, reset, theme variables
  router.tsx       ← Route configuration (or per framework convention)
  app.tsx          ← Root component
```

Routing is framework-specific. It can live in `app/` directly, or follow
framework conventions:

- **Vite + any router**: `app/router.tsx`
- **TanStack Router (file-based)**: `routes/` at src root (framework-managed)
- **Next.js App Router**: `app/` at project root (framework-managed)

When the framework manages route files, they are **thin** — only routing
config, params, guards. They import and render page components.

```typescript
// Thin route file example (framework-managed)
// routes/$workspaceSlug/projects/index.tsx
export const Route = createFileRoute('/$ws/projects/')({
  component: ProjectListPage,
});

// The actual page component lives in pages/
// pages/projects/project-list-page.tsx
export function ProjectListPage() { /* all UI logic here */ }
```

**Layouts** live in `app/layouts/`. They define the page shell (sidebar +
content area, centered auth card, etc.) and are consumed by routing.

```typescript
// app/layouts/app-layout.tsx
export function AppLayout({ children }) {
  return (
    <div className="flex h-screen">
      <Sidebar />
      <main className="flex-1">{children}</main>
    </div>
  );
}
```

**Belongs in app:** Global providers, routing, layouts, global styles,
error boundaries, analytics initialization.

**Does not belong:** Feature-specific code, business logic, page-level UI.

---

## Pages Layer

Screens bound to routes. **Page-first approach** — pages own substantial
logic. In early project stages, most code lives here.

A page slice is **a group of related pages by domain**, not a single
screen. One slice can contain multiple page components:

```text
pages/
  auth/                       ← One slice, multiple pages
    ui/
      LoginPage.tsx
      SignupPage.tsx
    model/
      auth.ts
    index.ts
  issues/                     ← One slice, multiple pages
    ui/
      IssueListPage.tsx
      IssueBoardPage.tsx
      IssueRow.tsx
      IssueCard.tsx
    model/
      issues.ts
    index.ts
  profile/
    ui/
      ProfilePage.tsx
      ProfileForm.tsx
    model/
      profile.ts
    index.ts
```

**Belongs in pages:** Page-specific UI, forms, validation, data fetching,
state management, business logic. Even code that looks reusable stays here
if it is simpler to keep local.

**Does not belong:** Code genuinely reused in 2+ page slices (extract
only when the team agrees).

### Page Layout Patterns

A page composes widgets, features, and entities from lower layers, plus
its own local UI:

```typescript
// pages/product-detail/ui/ProductDetailPage.tsx
import { Header } from '@/widgets/header';
import { AddToCart } from '@/features/add-to-cart';
import { ProductCard } from '@/entities/product';

export function ProductDetailPage({ productId }) {
  const product = useProductDetail(productId); // local hook

  return (
    <>
      <Header />
      <ProductCard data={product} />
      <AddToCart productId={productId} />
      <RelatedProducts products={product.related} /> {/* local */}
    </>
  );
}
```

### Page-first workflow

```text
Step 1:  pages/CheckoutPage.tsx (200 lines, all logic inside)
            ↓ page grows, logic needs reuse
Step 2:  features/checkout/ + pages/CheckoutPage.tsx (thin — composes)
```

---

## Widgets Layer

Composite UI blocks reused across **multiple pages**. Add only when the
same composition appears in 2+ pages.

```text
widgets/
  header/
    ui/
      Header.tsx
      Navigation.tsx
      UserMenu.tsx
    model/
      header.ts
    index.ts
  sidebar/
    ui/
      Sidebar.tsx
    model/
      sidebar.ts
    index.ts
```

A widget composes features and entities:

```typescript
// widgets/product-catalog/ui/ProductCatalog.tsx
import { ProductList, FilterPanel } from "@/features/catalog";
import { AddToCartButton } from "@/features/cart";
import { ProductCard } from "@/entities/product";

export function ProductCatalog() {
  return (
    <>
      <FilterPanel />
      <ProductList
        renderProduct={(p) => (
          <ProductCard
            product={p}
            actions={<AddToCartButton product={p} />}
          />
        )}
      />
    </>
  );
}
```

**Belongs in widgets:** Navigation bars, sidebars, dashboards — complex
blocks combining data from multiple entities/features.

**Does not belong:** Simple UI primitives (→ `shared/ui/`), single-use
page sections (→ keep in page).

---

## Features Layer

Independent, isolated business features. **Create only when used in 2+
places** (or when a page outgrows flat structure).

```text
features/
  auth/
    ui/
      LoginForm.tsx
      RegisterForm.tsx
    model/
      auth.ts               ← State + API calls
    index.ts
  cart/
    domain/
      cart.types.ts
    model/
      cart.ts
    ui/
      CartButton.tsx
    index.ts
```

Features have fractal nesting. For detailed internal structure, read
`references/feature-anatomy.md`.

**Belongs in features:** Isolated units of business functionality — each
feature encapsulates its own bounded context with state, logic, and UI.

**Does not belong:** Domain data models (→ `entities/`), infrastructure
(→ `shared/`), UI primitives (→ `shared/`).

---

## Entities Layer

Reusable business modules. **Create only when used in 2+ places.**
Starting without this layer is valid.

Entities are broader than strict FSD "domain models" — they can contain
domain data, business logic, state, and primitive UI. Stateful business
modules like session management and workspace context also belong here.

```text
// Minimal entity — model only
entities/user/
  model/
    user.ts
  index.ts

// Entity with all segments
entities/product/
  domain/
    product.types.ts
    product.mapper.ts
  model/
    product.store.ts        ← State + API calls
  ui/
    ProductCard.tsx          ← Primitive UI only
  index.ts

// Stateful business module
entities/session/
  domain/
    session.types.ts        ← Session, TokenPair types
  model/
    session.store.ts        ← Current session state
    token.ts                ← getToken, setToken, clearToken
  index.ts
```

### Entity UI rules

Entity `ui/` contains only **primitive components** — badge, avatar, card,
status indicator. Complex UI (lists with filters, dialogs, forms) belongs
in `features/`.

Entity components receive data **through props** — they don't pull
dependencies on other entities. Wiring happens in upper layers.

```typescript
// ✅ entities/project/ui/ProjectCard.tsx — pure, props only
function ProjectCard({ name, openIssuesCount }: Props) { /* ... */ }

// ❌ entities/project/ui/ProjectCard.tsx — reaches into another entity
import { openIssuesCount } from "@/entities/issue";
```

### When entity, when feature

| Situation | Where |
|---|---|
| Data used by **multiple features** | `entities/` |
| Data used by **one feature only** | `features/*/model/` |
| Data starts in one feature, another needs it | Move to `entities/` |
| Core domain entity (central to the whole app) | `entities/` from start |

---

## Shared Layer

Infrastructure, UI kit, and utilities with **zero business logic**. No
slices — organized by segments only.

**No barrel `index.ts`** — import files directly for tree-shaking.

```text
shared/
  ui/                ← Button, Input, Modal, Card, Badge, Avatar
  lib/               ← cn(), formatDate(), debounce()
  api/               ← HTTP client, API helpers, route constants
  i18n/              ← Localization setup
  assets/            ← Images, fonts, icons
```

```typescript
// ✅ Direct file imports
import { Button } from "@/shared/ui/button";
import { cn } from "@/shared/lib/cn";

// ❌ No barrel imports
import { Button, cn } from "@/shared";
```

**Belongs in shared:** UI primitives, utility functions, HTTP client,
i18n configuration, analytics setup, assets.

**Does not belong:** Business logic, session management
(→ `entities/session/`), domain types (→ `entities/`).

---

## Path Aliases

Configure `@/` alias for clean imports:

```typescript
// ✅ Alias imports
import { Button } from "@/shared/ui/button";
import { useUser } from "@/entities/user";

// ❌ Relative imports across layers
import { Button } from "../../../shared/ui/button";
```

Configure in both bundler config and `tsconfig.json` for IDE support.
