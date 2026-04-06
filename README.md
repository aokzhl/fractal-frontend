# Fractal Frontend

A Claude Code skill for fractal frontend architecture. Provides a structured approach to organizing frontend projects with layered architecture, strict import rules, and fractal nesting.

Inspired by [Feature-Sliced Design](https://feature-sliced.design/) and [FEOD](https://habr.com/ru/companies/sportmaster_lab/articles/972410/).

## Install

Copy the skill into your project:

```bash
git clone https://github.com/aokzhl/fractal-frontend /tmp/fractal-frontend
cp -r /tmp/fractal-frontend/skills/fractal-frontend .claude/skills/fractal-frontend
```

Or as a git submodule:

```bash
git submodule add https://github.com/aokzhl/fractal-frontend .claude/skills/fractal-frontend
```

## Architecture

6 layers with strict top-down import direction:

```
app/       → Application shell: providers, router, layouts, CSS theme
pages/     → Screens bound to routes. Page-first approach — start fat
widgets/   → Compositions of features reused across multiple pages
features/  → Business features — isolated modules
entities/  → Reusable business modules — domain data, stores, logic, primitive UI
shared/    → Infrastructure, UI kit, utilities — zero business logic
```

## Key Principles

- **Pages first** — start with everything in pages, extract only when the team agrees
- **Entities as business modules** — broader than FSD "domain models", can contain logic and presentation
- **Fractal nesting** — common/ + modules/ pattern for any slice-based layer
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
