# Quandatics CRM — Docs site

A [Zudoku](https://zudoku.dev) static docs site for the Quandatics CRM
developer/operator documentation. Built with npm (isolated from the app's pnpm
workspace).

## Run locally

```bash
npm install
npm run dev        # local preview with hot reload
npm run build      # static build → dist/
npm run check:docs # verify every route/plugin has a registered product page
```

## Repository layout

This repo is self-contained — all content is authored here directly.

- `pages/` — the published site (Zudoku navigates `pages/**/*.{md,mdx}`).
  Product capability pages live under `pages/product/<domain>/`; developer,
  codebase, API, and operations pages are authored here as well.
- `archive/` — internal working documents that are **not** published: design
  specs (`archive/specs/`), implementation plans (`archive/plans/`), and
  operational runbooks (`archive/operations/`). They are kept in the repo for
  history and reference, not rendered into the site.
- `catalog/modules.json` — the coverage registry validated by `check:docs`.
- `apis/crm-api.yaml` — the OpenAPI spec behind the API playground.

## Adding a product capability

1. Add one canonical page under
   `pages/product/<domain>/<capability>.mdx`.
2. Register its route, schema, permissions, plugin flag, and page in
   `catalog/modules.json`.
3. Add the page to `zudoku.config.tsx`.
4. Run `npm run check:docs`.

The build fails if a plugin flag is undocumented, a catalog page or route is
missing, or a catalog page is absent from navigation.

## External developer access

The public **External developers** section is for partners integrating through
stable contracts instead of CRM internals. It documents API usage, sandbox expectations, key handling,
support flow, and the current read-only boundary. Do not publish server access,
database credentials, restricted contributor instructions, or production secrets.

Repository visibility never grants production, database, secret, or server
access.

## Deploy to Vercel

The `docs-quality` GitHub Actions workflow is the only production deployment
mechanism. Do not run manual production deployments for routine releases.

1. Create the Vercel project and set **Root Directory** = the repository root (`.`) with
   framework preset **Zudoku**.
2. Add its token as the `VERCEL_TOKEN` GitHub Actions secret.
3. Keep the Vercel organization and project IDs in `docs-quality.yml` aligned
   with that project.

Pull requests must pass the `docs-quality` verification job before merge. After
the change reaches `main`, that same workflow verifies the merge commit and
creates exactly one production deployment from the checked-out source.
If GitHub does not emit the expected push run, manually dispatch `docs-quality`;
the workflow still verifies `main` before deploying.
