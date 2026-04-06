# Feature Anatomy

Internal structure of features — the main differentiator of Fractal FSD.

---

## What is a Feature

A feature is not just a component and not just a folder. It is a
mini-application inside your project — an isolated unit with its own
Bounded Context that encapsulates business logic, state, and UI for a
specific piece of functionality.

A feature can be as simple as a single component or as complex as a
standalone application with its own sub-features. Internal structure
scales fractally as the feature grows — see `references/fractal-nesting.md`.

---

## Segments

Each feature is organized into 3 segments with unidirectional dependency:

```text
domain → model → ui
```

| Segment   | Purpose                                     | Depends on  |
| --------- | ------------------------------------------- | ----------- |
| `domain/` | Types, mappers, pure operations             | nothing     |
| `model/`  | State, stores, actions, computed, API calls | `domain/`   |
| `ui/`     | Components                                  | all above   |

**Segments are a recommendation, not a rigid rule.** `domain/model/ui` is
the default set, but you can remove segments or add custom ones
(e.g., `service/` for API calls, `lib/` for helpers) when needed.
The only hard rule is **unidirectional dependency**.

Start with what you need — typically `model/` + `ui/` is enough for a
simple feature.

```text
features/favorites/
  model/
    favorites.ts          ← State + API calls + business logic
  ui/
    ToggleFavorite.tsx
  index.ts
```

---

## Simple Feature

No sub-features needed. Flat structure:

```text
features/favorites/
  model/
    favorites.ts
  ui/
    ToggleFavorite.tsx
  index.ts
```

Or with domain types extracted:

```text
features/cart/
  domain/
    cart.types.ts
  model/
    cart.ts               ← State + API calls
  ui/
    CartButton.tsx
    CartDrawer.tsx
  index.ts
```

---

## Complex Feature (Fractal Nesting)

When a feature has 2+ distinct sub-blocks of functionality that need
shared code between them, introduce `common/` and `modules/`.

For the full pattern, import rules, and examples, see
`references/fractal-nesting.md`.

---

## Checklist: Creating a New Feature

1. **Determine if it's really a feature.** Is it an isolated unit of
   business functionality with its own bounded context?
   → feature. Reusable business module? → `entities/`.

2. **Create the directory:**
   ```text
   src/features/<name>/
   ```

3. **Start with only the segments you need.** Typically `model/` + `ui/`:
   ```text
   features/<name>/
     model/
       <name>.ts
     ui/
       <Component>.tsx
     index.ts
   ```

4. **Create `index.ts`** that re-exports the public API:
   ```typescript
   export { MyComponent } from "./ui/MyComponent";
   export { myModel } from "./model/my-model";
   ```

5. **Verify imports.** The feature should only import from:
   `entities/`, `shared/`.

6. **Do NOT add `common/` or `modules/` upfront.** Add when 2+ distinct
   sub-blocks emerge and need shared code.

7. **Add `domain/` when** types or mappers appear that need to be
   separated from `model/` for clarity or reuse.
