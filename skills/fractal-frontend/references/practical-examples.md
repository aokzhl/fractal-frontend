# Practical Examples

Concrete patterns for common scenarios within Fractal FSD. Examples are
framework and library agnostic — focus on structure and placement, not
specific tools.

---

## Authentication Pattern

Auth spans three layers. The key question: what goes where?

### Tokens and session → entities/

Session is a reusable business module — it carries user context used
across the app:

```text
entities/
  session/
    domain/
      session.types.ts        ← Session, TokenPair types
    model/
      session.store.ts        ← Current session state
      token.ts                ← getToken, setToken, clearToken
    index.ts
```

```typescript
// entities/session/domain/session.types.ts
export interface Session {
  userId: string;
  email: string;
  role: "admin" | "member";
}

export interface TokenPair {
  accessToken: string;
  refreshToken: string;
}
```

### Auth UI → pages (single use) or features (multi-use)

```text
// If login form is only used on the login page:
pages/login/
  ui/
    LoginPage.tsx
    LoginForm.tsx
  model/
    login.ts              ← Form state, validation, API call
  index.ts

// If login form is reused (modal login + dedicated page):
features/auth/
  ui/
    LoginForm.tsx
    RegisterForm.tsx
  model/
    auth.ts               ← State + API calls
  index.ts
```

### User profile data → entities (when reused)

```text
// Do NOT create entities/user just for auth.
// Create it only when user profile is needed in 2+ places
// (e.g., avatars in comments, names in issue assignees).

entities/user/
  domain/
    user.types.ts         ← Profile: displayName, avatar, role
    user.mapper.ts        ← DTO → domain
  model/
    user.store.ts         ← State + API calls
  ui/
    UserAvatar.tsx         ← Primitive UI only
  index.ts
```

### Summary

| What | Where | Why |
|---|---|---|
| Tokens, refresh, session state | `entities/session/` | Business module |
| Login/register forms | `pages/` or `features/auth/` | UI + interaction |
| User profile data | `entities/user/` | Domain model |

---

## CRUD Entity

A typical entity with all segments:

```text
entities/product/
  domain/
    product.types.ts      ← Domain type + constants
    product.mapper.ts     ← DTO → Domain mapping
  model/
    product.store.ts      ← State + API calls
  ui/
    ProductCard.tsx        ← Primitive display component
    ProductBadge.tsx
  index.ts
```

### domain/ — pure types and mappers

```typescript
// entities/product/domain/product.types.ts
export interface Product {
  id: string;
  name: string;
  price: number;
  formattedPrice: string;
  isOnSale: boolean;
}

// entities/product/domain/product.mapper.ts
import type { ProductDTO } from "@/shared/api/product";
import type { Product } from "./product.types";

export const toProduct = (dto: ProductDTO): Product => ({
  id: dto.id,
  name: dto.name,
  price: dto.price,
  formattedPrice: `$${dto.price.toFixed(2)}`,
  isOnSale: dto.price < 10,
});
```

### model/ — state + API calls

```typescript
// entities/product/model/product.store.ts
import { httpClient } from "@/shared/api/client";
import { toProduct } from "../domain/product.mapper";

// API calls + state in one place
export const productApi = {
  getById: async (id: string) => {
    const dto = await httpClient.get(`/products/${id}`);
    return toProduct(dto);
  },
  getAll: async () => {
    const dtos = await httpClient.get("/products");
    return dtos.map(toProduct);
  },
};

// Store that holds product data, exposes getById, list, etc.
// Specific implementation depends on your state manager
```

### ui/ — primitive components only

```typescript
// entities/product/ui/ProductCard.tsx
// Receives data through props — no dependencies on other entities

interface ProductCardProps {
  name: string;
  formattedPrice: string;
  isOnSale: boolean;
}

export function ProductCard({ name, formattedPrice, isOnSale }: ProductCardProps) {
  return (
    <div>
      <h3>{name}</h3>
      <span>{formattedPrice}</span>
      {isOnSale && <span>Sale!</span>}
    </div>
  );
}
```

### index.ts — public API

```typescript
// entities/product/index.ts
export type { Product } from "./domain/product.types";
export { toProduct } from "./domain/product.mapper";
export { productApi, productStore } from "./model/product.store";
export { ProductCard } from "./ui/ProductCard";
export { ProductBadge } from "./ui/ProductBadge";
```

---

## Complex Feature with Sub-Modules

A checkout feature with delivery and payment sub-blocks:

```text
features/checkout/
  common/
    domain/
      order-draft.types.ts      ← Shared between delivery + payment
    model/
      order-draft.store.ts      ← Shared state
  modules/
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
  model/
    checkout.facade.ts          ← Orchestrates modules
  ui/
    CheckoutLayout.tsx          ← Composes module UIs
  index.ts
```

### Orchestration in model/

```typescript
// features/checkout/model/checkout.facade.ts
import { deliveryModel } from "../modules/delivery-form";
import { paymentModel } from "../modules/payment-form";
import { orderDraftStore } from "../common/model/order-draft.store";

export const checkoutFacade = {
  submit: async () => {
    const delivery = deliveryModel.getValues();
    const payment = paymentModel.getValues();
    const draft = orderDraftStore.getDraft();
    return api.createOrder({ ...draft, delivery, payment });
  },
  isValid: () => deliveryModel.isValid() && paymentModel.isValid(),
};
```

### Composition in ui/

```typescript
// features/checkout/ui/CheckoutLayout.tsx
import { DeliveryForm } from "../modules/delivery-form";
import { PaymentForm } from "../modules/payment-form";
import { checkoutFacade } from "../model/checkout.facade";

export function CheckoutLayout() {
  return (
    <form onSubmit={checkoutFacade.submit}>
      <DeliveryForm />
      <PaymentForm />
      <button disabled={!checkoutFacade.isValid()}>Place Order</button>
    </form>
  );
}
```

---

## Page Evolution

How a page grows from fat to thin:

### Step 1: Fat page (all logic inside)

```text
pages/products/
  ui/
    ProductsPage.tsx       ← 300 lines: list, filters, cards
    ProductCard.tsx
    FilterBar.tsx
  model/
    products.ts            ← State + API + filtering
  index.ts
```

### Step 2: Extract when it hurts

Another page needs the same product list with filters. Team agrees to
extract:

```text
// Extracted feature
features/product-catalog/
  model/
    catalog.ts
  ui/
    ProductList.tsx
    FilterBar.tsx
  index.ts

// Extracted entity (used by catalog + other features)
entities/product/
  domain/
    product.types.ts
  ui/
    ProductCard.tsx
  index.ts

// Page becomes thin — just composes
pages/products/
  ui/
    ProductsPage.tsx       ← 30 lines: imports and renders
  index.ts
```

---

## Widget Composition

A header widget that composes features from different domains:

```text
widgets/header/
  ui/
    Header.tsx
    Navigation.tsx
    UserMenu.tsx
  model/
    header.ts
  index.ts
```

```typescript
// widgets/header/ui/Header.tsx
import { WorkspaceSwitcher } from "@/features/workspace";
import { UserAvatar } from "@/entities/user";
import { Navigation } from "./Navigation";

export function Header() {
  return (
    <header>
      <WorkspaceSwitcher />
      <Navigation />
      <UserAvatar />
    </header>
  );
}
```

**This widget exists because** the same header appears on every
authenticated page. If it were only on one page, it would stay in that
page's local UI.

---

## Type Placement Guide

| Type scope | Location |
|---|---|
| API response/request shapes | `shared/api/` or contracts package |
| Domain model for a reused entity | `entities/*/domain/` |
| Types used only in one page | `pages/*/model/` or `pages/*/domain/` |
| Types used only in one feature | `features/*/model/` or `features/*/domain/` |
| Generic utility types (`Nullable<T>`) | `shared/lib/types.ts` |
| Infrastructure types (HTTP config) | `shared/api/` |

**Rule:** Raw API shapes (DTOs) stay close to transport. Domain models
with business logic go in entities. If you only need the raw shape and
have no business logic, a shared types file is sufficient.

---

## entities/ vs shared/ — When to Use Each

| Question | Answer | Layer |
|---|---|---|
| Does it have **zero** logic and **zero** state? | UI primitive or utility | `shared/` |
| Is it **infrastructure** with no business context? | HTTP client, i18n, analytics | `shared/` |
| Is it a **reusable business module**? | Domain model, session, workspace | `entities/` |
| Is it a **user-facing capability**? | Feature with UI + state | `features/` |

```text
shared/ui/button.tsx                ← Zero logic, zero state
shared/lib/cn.ts                    ← Pure utility
shared/api/client.ts                ← Infrastructure (HTTP client)
entities/session/model/token.ts     ← Business module (session state)
entities/user/domain/user.ts        ← Business module (User profile)
features/auth/model/auth.ts         ← User capability (login flow)
```
