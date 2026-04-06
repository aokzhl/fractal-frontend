# Cross-Feature Communication

How features interact without direct imports. Features never import other
features — this is a core rule of Fractal FSD. This reference covers
strategies for resolving situations where features need to share data
or trigger each other's actions.

---

## The Problem

Features are isolated modules. When two features need to interact, a
direct import would create coupling and violate the architecture:

```typescript
// ❌ features/cart imports features/promo — forbidden
import { applyPromoCode } from "@/features/promo";
```

---

## Strategy 1: Shared Entity

When two features need the same **data**, extract it to `entities/`.

```text
// Before: two features duplicate order types
features/order-create/
  model/order.ts        ← Order types (duplicated)
features/order-history/
  model/order.ts        ← Order types (duplicated)

// After: shared domain in entities
entities/order/
  domain/
    order.types.ts      ← Shared types
  model/
    order.store.ts      ← Shared state
  index.ts

features/order-create/  ← Imports from entities/order
features/order-history/  ← Imports from entities/order
```

**When to use:** Two features operate on the same business data. The
shared part is domain logic (types, validation, business calculations).

**Key:** Extract only the genuinely shared domain logic. Feature-specific
UI, state, and API calls stay in the feature.

---

## Strategy 2: Compose in Page or Widget

Use the layer above (page or widget) to wire features together through
**props, render props, callbacks, or dependency injection**. The features
never reference each other.

### Props and Callbacks

```typescript
// pages/CartPage.tsx
import { Cart } from "@/features/cart";
import { PromoCodeInput } from "@/features/promo";

export function CartPage() {
  const { applyPromo } = usePromoActions();

  return (
    <>
      <Cart onApplyPromo={applyPromo} />
      <PromoCodeInput />
    </>
  );
}
```

### Render Props / Slots

```typescript
// widgets/product-catalog/ui/ProductCatalog.tsx
import { ProductList } from "@/features/catalog";
import { AddToCartButton } from "@/features/cart";

export function ProductCatalog() {
  return (
    <ProductList
      renderActions={(product) => <AddToCartButton product={product} />}
    />
  );
}
```

### Dependency Injection

```typescript
// features/notifications/model/notifications.ts
interface NotificationDeps {
  getUserName: (userId: string) => string;
}

export const createNotificationService = (deps: NotificationDeps) => ({
  format: (n) => `${deps.getUserName(n.userId)}: ${n.message}`,
});

// pages/dashboard/model/setup.ts — wire dependencies
import { createNotificationService } from "@/features/notifications";
import { getUserName } from "@/entities/user";

export const notificationService = createNotificationService({ getUserName });
```

**When to use:** Features are genuinely independent concepts and the
connection between them is a composition concern.

---

## Strategy 3: Merge into One Feature with Modules

When two "features" are so tightly coupled that separating them creates
more ceremony than value, merge them into one feature with `modules/`:

```text
// Before: two features that always change together
features/issue-list/
  model/issues.ts
  ui/IssueTable.tsx
features/issue-filters/
  model/filters.ts
  ui/FilterBar.tsx

// After: one feature with modules
features/issues/
  common/
    domain/
      issue-filter.types.ts
  modules/
    issue-list/
      model/issues.ts
      ui/IssueTable.tsx
      index.ts
    issue-filters/
      model/filters.ts
      ui/FilterBar.tsx
      index.ts
  model/
    issues.facade.ts       ← Composes both modules
  ui/
    IssuesView.tsx         ← Composes both UIs
  index.ts
```

**When to use:**

- The two "features" always change together
- They share most dependencies
- Separating them leads to excessive prop drilling or event passing
- They represent sub-blocks of one user-facing capability

**When NOT to use:**

- The features are genuinely independent (use Strategy 2)
- They have different lifecycles or ownership

---

## Decision Flowchart

```text
Two features need to interact
  │
  ├─ Do they share business data (types, domain logic)?
  │   └─ YES → Strategy 1: Extract shared data to entities/
  │
  ├─ Is the connection a UI composition concern?
  │   └─ YES → Strategy 2: Compose in page/widget via props
  │
  ├─ Do they always change together / represent one capability?
  │   └─ YES → Strategy 3: Merge into one feature with modules/
  │
  └─ None of the above?
      └─ Reconsider your feature boundaries. The need for interaction
         often signals that the features are not properly decomposed.
```

---

## Entity Cross-Imports

Unlike features, **entities CAN import other entities**. This is allowed
because entities represent the business domain, and domain concepts
naturally reference each other (an Order references a User, a Comment
references an Issue).

**Rules:**

1. **Prefer `import type`** to minimize coupling:
   ```typescript
   // entities/order/domain/order.types.ts
   import type { User } from "@/entities/user";

   export interface Order {
     id: string;
     customer: User;
   }
   ```

2. **Avoid cycles.** If entity A imports entity B and entity B imports
   entity A, consider merging them or extracting the shared part.

3. **Entity UI must not import other entities.** Wiring entity UI
   components happens in upper layers (features, widgets, pages) via props.

4. **Aggregates own related data.** An `order` entity that imports
   `product` and `user` types is fine — it's an aggregate.
