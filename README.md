# Fractal Frontend

A skill for fractal frontend architecture. Provides a structured approach to organizing frontend projects with layered architecture, strict import rules, and fractal nesting.

Inspired by [Feature-Sliced Design](https://feature-sliced.design/), [FEOD](https://habr.com/ru/companies/sportmaster_lab/articles/972410/), and Evolution Design (ED).

## Install

```bash
npx skills add aokzhl/fractal-frontend
```

## Architecture

6 layers with strict top-down import direction:

```
app/       → Application shell: providers, router, layouts, CSS theme
pages/     → Screens bound to routes. Compose + page-level orchestration
widgets/   → Compositions of features reused across multiple pages
features/  → Business features — isolated modules (not micro use-cases)
entities/  → Reusable business modules — domain data, services, domain UI
shared/    → Infrastructure, UI kit, utilities — zero business logic
```

Minimal project: `app/` + `pages/` + `features/` + `shared/`. Add `entities/` and `widgets/` on signal.

## Key Principles

- **Feature-first by default** — new business code lives in `features/`, not in `pages/`, `entities/`, or `shared/`
- **Feature = cohesive product block, not a use-case** — `features/auth/` (login, signup, recovery), not `features/login/` + `features/signup/`
- **Entities as business modules** — domain data, services, and domain UI; can contain logic and presentation
- **Fractal nesting** — `common/` + `modules/` pattern for any slice-based layer
- **Modules grow organically** — no fixed stages; restructure when complexity demands it
- **Strict import rules** — features don't import features, entities can cross-import with `import type`

## Usage

The skill activates when you work on project structure, code placement, import boundaries, or fractal nesting. Invoke directly with `/fractal-frontend`.

## Reference Files

Loaded on demand when specific situations apply:

- `layer-structure.md` — folder structures and naming conventions per layer
- `feature-anatomy.md` — internal structure of features
- `fractal-nesting.md` — common/ + modules/ pattern for any layer
- `cross-feature-communication.md` — cross-import resolution strategies
- `practical-examples.md` — auth, CRUD, complex features, page evolution patterns

## License

MIT
