---
name: fractal-frontend
description: >
  Fractal frontend architecture skill. Use when organizing project structure
  with layers, deciding where code belongs, defining public APIs and import
  boundaries, working with fractal nesting, or deciding whether logic should
  remain local or be extracted.
---

# Fractal Frontend

> Fractal architecture for frontend projects. Inspired by Feature-Sliced
> Design and FEOD. Strictness can be adjusted based on project scale and
> team context.

---

## 1. Core Philosophy & Layer Overview

Core principle: **"Start simple, extract when needed."**

Place code in `pages/` first. Duplication across pages is acceptable and does
not automatically require extraction. Extract only when the team agrees.

**Not all layers are required.** Most projects start with `app/`, `pages/`,
and `shared/`. Add other layers only when they provide clear value.

Fractal FSD uses 6 layers with strict top-down import direction:

```text
app/       → Application shell: providers, router, layouts, CSS theme
pages/     → Screens bound to routes. Page-first approach — start fat
widgets/   → Compositions of features reused across multiple pages
features/  → Business features — isolated modules
entities/  → Reusable business modules — domain data, stores, logic, primitive UI
shared/    → Infrastructure, UI kit, utilities — zero business logic
```

### Import Rules

```text
app/       →  pages, widgets, features, entities, shared
pages/     →  widgets, features, entities, shared
widgets/   →  features, entities, shared
features/  →  entities, shared
entities/  →  entities, shared
shared/    →  shared
```

Import only through `index.ts`. Never reach into module internals.

**Exception:** `shared/` has no barrel `index.ts` — import files directly
for tree-shaking.

**Cross-import restrictions:**

- Features do NOT import other features
- Widgets do NOT import other widgets
- Entities CAN import other entities (prefer `import type`, avoid cycles)

### Routing

Routing lives in `app/` or follows framework conventions (Next.js `app/`,
TanStack Router `routes/`, etc.). Route files are **thin** — they import
and render page components, nothing more.

---

## 2. Decision Framework

When writing new code, follow this tree:

**Step 1 — Where is this code used?**

- Used in only one page → keep it in that page.
- Used in 2+ pages but duplication is manageable → keeping separate copies
  in each page is also valid.
- An entity or feature used in only one page → keep it in that page.

**Step 2 — Is it a UI primitive, utility, or infrastructure with zero
business logic?**

- UI components → `shared/ui/`
- Utility functions → `shared/lib/`
- HTTP client, API helpers → `shared/api/`
- i18n, analytics → `shared/i18n/`, `shared/analytics/`

**Step 3 — Is it a complete user action reused in 2+ places, and does the
team agree to extract it?**

- Yes → `features/`
- Uncertain or single use → keep in the page.

**Step 4 — Is it a reusable business module (domain model, session,
workspace context) used in 2+ places?**

- Yes → `entities/`
- Uncertain or single use → keep in the page.

**Step 5 — Is it a composition of features reused across multiple pages?**

- Yes → `widgets/`
- One page only → keep in the page.

**Step 6 — Is it app-wide configuration?**

- Global providers, router, layouts → `app/`

**Golden Rule: When in doubt, keep it in `pages/`. Extract only when the
team agrees.**

---

## 3. Quick Placement Table

| Scenario              | Single use                                  | Multi-use (with team agreement)       |
| --------------------- | ------------------------------------------- | ------------------------------------- |
| User profile form     | `pages/profile/ui/ProfileForm.tsx`          | `features/profile-form/`             |
| Product card          | `pages/products/ui/ProductCard.tsx`         | `entities/product/ui/ProductCard.tsx` |
| Data fetching         | `pages/product-detail/model/product.ts`     | `entities/product/model/`             |
| Auth tokens/session   | `entities/session/` (always)                | `entities/session/` (always)         |
| Auth login form       | `pages/login/ui/LoginForm.tsx`              | `features/auth/`                     |
| HTTP client           | `shared/api/` (always)                      | `shared/api/` (always)               |
| Generic Card layout   | —                                           | `shared/ui/card.tsx`                 |
| Date formatting util  | —                                           | `shared/lib/format-date.ts`          |
| Sidebar navigation    | `app/layouts/sidebar.tsx`                   | `widgets/sidebar/`                   |

---

## 4. Architectural Rules (MUST)

These rules are the foundation. Violations weaken the architecture. If you
must break a rule, document the reason and get team consensus.

### 4-1. Import only from lower layers

`app → pages → widgets → features → entities → shared`. Upward
imports and cross-imports between slices on the same layer are forbidden
(except entities → entities).

### 4-2. Public API — every module exports through index.ts

External consumers import only from a module's `index.ts`. Direct imports
of internal files are forbidden.

```typescript
// ✅ Correct
import { LoginForm } from "@/features/auth";

// ❌ Violation — bypasses public API
import { LoginForm } from "@/features/auth/ui/LoginForm";
```

**Exception:** `shared/` has no barrel index — import files directly:

```typescript
// ✅ Correct
import { cn } from "@/shared/lib/cn";
import { Button } from "@/shared/ui/button";

// ❌ Violation — barrel import from shared
import { cn, Button } from "@/shared";
```

### 4-3. No cross-imports between features or widgets

Features never import other features. Widgets never import other widgets.
If they need to interact, the layer above (page or widget) composes them.

### 4-4. Domain-based file naming

Name files after the business domain, not their technical role:

```text
// ❌ Technical-role naming
model/types.ts          ← Which types? User? Order? Mixed?
model/utils.ts

// ✅ Domain-based naming
model/user.ts           ← User types + related logic
model/order.ts          ← Order types + related logic
model/profile.ts
```

### 4-5. No business logic in shared/

`shared/` contains only infrastructure, UI kit, and utilities — zero
business logic. Business calculations, domain rules, and workflows
belong in `entities/` or higher.

### 4-6. Segment dependency is unidirectional

```text
domain → model → ui
```

Never import backwards (e.g., `domain/` importing from `model/`).

---

## 5. Recommendations (SHOULD)

### 5-1. Pages First — place code where it is used

Place code in `pages/` first. Extract to lower layers only when truly needed.

**What stays in pages:**

- Large UI blocks used only in one page
- Page-specific forms, validation, data fetching, state management
- Page-specific business logic and API integrations
- Code that looks reusable but is simpler to keep local

**Evolution pattern:** Start with everything in `pages/profile/`. When
another page needs the same user data and the team agrees, extract the
shared model to `entities/user/`. Keep page-specific UI in the page.

### 5-2. Be conservative with entities

Entities are reusable business modules — broader than strict FSD "domain
models". They can contain domain data, business logic, state, and
primitive UI. Session management, workspace context, and similar stateful
business modules also belong here.

The entities layer is highly accessible — almost every other layer can
import from it, so changes propagate widely.

1. **Start without entities.** `app/` + `pages/` + `shared/` is valid.
2. **Do not split slices prematurely.** Keep code in pages.
3. **Place it in entities only when 2+ consumers are confirmed.**

**Exception:** Core domain entities that are clearly central to the whole
app (Issue in a tracker, Product in e-commerce, Session in an auth system)
can go to `entities/` from the start. This is domain awareness, not
premature abstraction.

### 5-3. Start with minimal layers

```text
// ✅ Valid minimal project
src/
  app/         ← Providers, routing, layouts
  pages/       ← All page-level code
  shared/      ← Infrastructure, UI kit, utils

// Add layers only when the team decides they are needed:
// + entities/  ← When business modules are reused in 2+ places
// + features/  ← When user interactions are reused in 2+ places
// + widgets/   ← When compositions are reused across pages
```

### 5-4. entities/ vs shared/ — know the boundary

Ask: "Does this have business context or just plumbing?"

- Business context (session, workspace, domain data) → `entities/`
- Pure plumbing (HTTP client, i18n, UI kit, utils) → `shared/`

| Code                          | shared/ | entities/ | features/ |
| ----------------------------- | ------- | --------- | --------- |
| HTTP client, interceptors     | ✅      |           |           |
| i18n configuration            | ✅      |           |           |
| Session, token refresh        |         | ✅        |           |
| Current workspace context     |         | ✅        |           |
| Type `Product`, `toProduct()` |         | ✅        |           |
| `productApi.getById()`        |         | ✅        |           |
| Product list with filters     |         |           | ✅        |
| Create product dialog         |         |           | ✅        |

---

## 6. Anti-patterns (AVOID)

- **Do not create entities prematurely.** Data used in one place belongs
  in that place.
- **Do not put business logic in shared/.** Infrastructure and utilities only.
- **Do not extract single-use code.** A feature used by one page stays
  in that page.
- **Do not use technical-role file names.** Use domain-based names.
- **Do not create common/ + modules/ in advance.** Add when the feature
  outgrows a flat structure (2+ distinct sub-blocks).
- **Do not create god features.** Features with broad responsibilities
  should be split (e.g., `user-management/` → `auth/`, `profile-edit/`).
- **Be cautious adding UI to entities.** Entity UI should only be
  imported from higher layers — never from other entities.
- **Do not create empty layer directories** "just in case."

---

## 7. Segments

Each module (in `entities/`, `features/`, and feature's `common/`)
is organized into 3 segments with unidirectional dependency:

```text
domain → model → ui
```

| Segment   | Purpose                                    | Depends on   |
| --------- | ------------------------------------------ | ------------ |
| `domain/` | Types, mappers, pure operations            | nothing      |
| `model/`  | State, stores, actions, computed, API calls | `domain/`   |
| `ui/`     | Components                                 | all above    |

**Segments are a recommendation, not a rigid rule.** `domain/model/ui` is
the default set, but you can adapt it to your needs: remove segments that
add no value, or introduce custom ones (e.g., `service/` for API calls,
`lib/` for internal helpers) when the module demands it. The only hard
rule is **unidirectional dependency** — lower segments must not import
from higher ones.

Not all segments are required. Start with what you need.

If a segment has only one concern, the filename may match the module name
(e.g., `features/auth/model/auth.ts`).

---

## 8. Quick Reference

- **Import direction**: `app → pages → widgets → features → entities → shared`
- **Minimal project**: `app/` + `pages/` + `shared/`
- **Create entities when**: 2+ consumers share the same business module, and the team agrees
- **Create features when**: 2+ consumers share the same user interaction, and the team agrees
- **Segments**: `domain → model → ui` (unidirectional)
- **Public API**: Import through `index.ts` (except `shared/` — direct file imports)
- **No cross-imports**: features ✕ features, widgets ✕ widgets
- **Entities cross-import**: allowed, prefer `import type`
- **File naming**: Domain-based (`user.ts`, `order.ts`), never technical-role (`types.ts`)
- **Routing**: In `app/` or per framework convention
- **Layouts**: In `app/`
- **entities/ vs shared/**: entities = reusable business modules, shared = infrastructure + UI kit + utils

---

## 9. Conditional References

Read reference files **only** when the specific situation applies.
Do **not** preload all references.

- **When creating, reviewing, or reorganizing folder structure** for
  layers and modules:
  → Read `references/layer-structure.md`

- **When creating or restructuring a feature**, working with segments,
  or the feature checklist:
  → Read `references/feature-anatomy.md`

- **When a slice outgrows a flat structure** and needs common/ + modules/
  (fractal nesting) — in features, entities, or widgets:
  → Read `references/fractal-nesting.md`

- **When resolving interaction between features**, deciding how features
  communicate, or dealing with entity cross-imports:
  → Read `references/cross-feature-communication.md`

- **When implementing concrete patterns** for authentication, CRUD
  entities, complex features, page evolution, or widget composition:
  → Read `references/practical-examples.md`
  Note: If you already loaded `layer-structure.md` in this conversation,
  address structure first, then load patterns in a follow-up step.
