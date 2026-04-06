# Fractal Nesting

How to scale a slice's internal structure when it outgrows a flat layout.
Applies to any slice-based layer — features, entities, widgets.

---

## When to Use

When a slice has **2+ distinct sub-blocks** that need shared code between
them. **Do not create in advance** — add when the slice outgrows a flat
structure.

Features are the most common candidate. Entities may need it when a
domain concept has distinct sub-types with shared logic (e.g., voucher
with gift/discount variants). Widgets rarely need it.

---

## Structure: common/ + modules/

```text
features/checkout/
  common/          ← Shared code for modules/
    domain/
      order-draft.types.ts
    model/
      order-draft.store.ts
  modules/         ← Sub-slices (fractal nesting)
    delivery-form/
      model/
        delivery.ts
      ui/
        DeliveryForm.tsx
      index.ts
    payment-form/
      model/
        payment.ts
      ui/
        PaymentForm.tsx
      index.ts
  model/           ← Slice's own logic, composes sub-slices
    checkout.facade.ts
  ui/              ← Slice's own UI, composes sub-slices
    CheckoutLayout.tsx
  index.ts
```

Entity example:

```text
entities/voucher/
  common/
    domain/
      voucher.types.ts        ← Shared types for all voucher kinds
    model/
      voucher.store.ts        ← Shared state
  modules/
    gift-voucher/
      domain/
        gift-voucher.types.ts
      model/
        gift-voucher.ts
      index.ts
    discount-voucher/
      model/
        discount-voucher.ts
      index.ts
  index.ts
```

---

## common/ vs model/

| | `common/` | `model/` |
|---|---|---|
| **Who uses it** | `modules/`, `model/`, `ui/` | Only the parent slice itself |
| **Direction** | Shared down to sub-slices | Composes up from sub-slices |
| **Contains** | Types, stores, UI shared across modules | Slice-level facade, orchestration |

`common/` is the shared infrastructure of the slice — a contract between
modules. If something is needed by multiple `modules/`, put it in
`common/`.

Can contain any segments: `domain/`, `model/`, `ui/`.

---

## Composition

The slice's `model/` imports from `modules/` to orchestrate their models
into a unified facade:

```typescript
// features/checkout/model/checkout.facade.ts
import { deliveryFormModel } from "../modules/delivery-form";
import { paymentFormModel } from "../modules/payment-form";
import { orderDraftStore } from "../common/model/order-draft.store";

export function submitCheckout() {
  const delivery = deliveryFormModel.getValues();
  const payment = paymentFormModel.getValues();
  const draft = orderDraftStore.getDraft();
  return api.createOrder({ ...draft, delivery, payment });
}
```

Sub-slices don't know about each other or about parent `model/`. Only
the parent `model/` knows about all of them and wires them together.

---

## Import Rules

```text
ui/        →  model/, domain/, common/               ✅
model/     →  domain/, common/, modules/             ✅  (composes)
domain/    →  nothing inside the slice                ✅
modules/*  →  ../common/                              ✅
modules/*  →  lower layers (entities/, shared/)       ✅
modules/*  →  ../modules/*                            ❌  siblings
modules/*  →  ../model/, ../ui/                       ❌  parent
common/    →  lower layers (entities/, shared/)        ✅
common/    →  model/, modules/, ui/                    ❌
```

**Key restrictions:**

1. `domain/` imports nothing inside the slice — it's the pure layer.
2. `common/` does not know about anything inside the slice — only lower
   layers.
3. `modules/` cannot import sibling modules or parent's `model/`/`ui/`.
4. Only `model/` and `ui/` at the slice root can compose modules.

---

## Recursive Nesting

Sub-slices follow the same structure recursively. If a module itself grows
complex, it can have its own `common/` + `modules/`:

```text
features/checkout/
  modules/
    delivery-form/
      common/          ← if it has its own modules/
      modules/
      model/
      ui/
      index.ts
```

In practice, two levels of nesting are usually sufficient. If you need
three, consider whether the slice should be split.
