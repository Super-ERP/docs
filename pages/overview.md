---
title: "CRM v2"
---

Quandatics' multitenant CRM for the full lead-to-cash lifecycle, plus the vendor
infrastructure that distributes and licenses it to client deployments.

| Go to | Purpose |
| --- | --- |
| [Production CRM](https://app.quandatics.com) | Live application |
| [Latest staging deployment](https://github.com/Super-ERP/crm-v2/actions/workflows/deploy-staging.yml) | Preview URL in the latest run summary |
| [Module directory](/product/module-directory) | Every capability and its code map |
| [External developer guide](/external-developers/overview) | API integration and public contribution path |
| [Add a module](/extensibility/adding-a-module) | Placement and integration checklist |
| [Contributing](/contributing) | Local setup and review rules |
| [Operations](/operations) | Deploy flow, operator workspace, and recovery |
| [Release log](https://github.com/Super-ERP/crm-v2/blob/main/docs/operations/release-log.md) | Signed immutable release record |

## What this is

The repository builds two distinct surfaces:

1. **The CRM application** (`apps/web`) — the multitenant Next.js product: routes,
   server actions, business services, schema, RLS, and the plugin system.
2. **The vendor distribution stack** — a Cloudflare Worker
   ([`apps/control-plane`](/operations)) for client/deployment metadata, commercial
   controls, deployment identity, heartbeats, and immutable signed entitlement
   leases, plus a client-hosted agent (`apps/deployment-agent`) that registers,
   heartbeats, and applies signed entitlements. They share the
   `packages/control-protocol` signing/command contracts.

## Module map

| Domain | Capabilities |
| --- | --- |
| CRM | Leads, Accounts, Contacts |
| Sales | Opportunities, Funnel, Approvals, Products, Quotations, Payment Milestones |
| Delivery | Projects, Sales Orders |
| Finance | O2C, P2P, Intercompany, Forecast |
| Platform | Dashboard, Team & RBAC, Settings, Audit, Documentation, Tenancy & Auth |

## Repository map

```text
crm-v2/
├── apps/
│   ├── web/                 Next.js application, routes, services, and database
│   ├── control-plane/       Vendor-operated Cloudflare Worker (operator console,
│   │                        entitlements, heartbeats, deployment identity)
│   └── deployment-agent/    Client-hosted agent (registration, heartbeat,
│                            entitlement apply, remote commands)
├── packages/
│   └── control-protocol/    Signed envelope + command contract, shared by
│                            control-plane and deployment-agent
├── docs/
│   └── operations/          Release log (machine-appended; runbooks live in this site)
├── ops/                     Operational scripts and operator notes
├── deploy/client/           Pull-only, Cosign-verified production bundle
├── .github/                 CI, production, and staging workflows
├── AGENTS.md                Rules for AI coding agents
└── README.md                Repository landing page
```

Local-dev and CI commands run from the repo **root** — the root `package.json`
scripts delegate to the workspace packages, so you don't need to `cd apps/web`.

## Local development

```bash
cp .env.example .env            # sensible localhost defaults; Microsoft optional for email login
docker compose -f docker-compose.dev.yaml up -d   # local Postgres 17 on :5432 (matches .env.example)
pnpm install
pnpm run db:generate             # (already generated; re-run after schema changes)
pnpm run db:setup                # apply migrations + RLS + views, then seed
pnpm run dev                     # http://localhost:3000
```

> `docker-compose.dev.yaml` runs **only** Postgres for local dev; the app itself runs on the host via `pnpm run dev`. If you already have a Postgres 17 elsewhere, point `DATABASE_URL` (the RLS-enforced `crm_app` role) and `DATABASE_ADMIN_URL` (the superuser, for migrations + seed) at it instead. `crm_app` is created by `db:setup`.

The seed creates a **Demo Entity** and a demo Owner login (printed at the end, default `admin@demo.local` / `Password123!`).

To start with a populated demo (extra logins + sample customers/funnels/quotations) instead of an empty entity, use the seeded setup — same as `db:setup`, plus `db/seed-sample.ts` layered on top:

```bash
pnpm run db:setup-seeded         # migrations + RLS + views + base seed + sample data
```

This adds four more logins under the Demo Entity (all password `Password123!`) and a set of sample customers, contacts, funnels, quotations, a won project, and leads to play with:

| Email | Role | Tier |
| --- | --- | --- |
| `admin@demo.local` | Owner (superadmin) | 100 |
| `manager@demo.local` | Manager | 60 |
| `sales1@demo.local` | Rep | 20 |
| `sales2@demo.local` | Rep | 20 |
| `viewer@demo.local` | Viewer | 10 |

The sample seed is idempotent and **dev-only** — don't run it on an internet-exposed deployment (it mints well-known default credentials). It is intentionally not part of the production Docker `migrate` step.

## Signed client release images

Client releases come from annotated strict SemVer tags such as `v1.2.3`.
`.github/workflows/release-images.yml` builds Linux AMD64 images for
the web runtime, migrator, encrypted-backup runtime, and deployment agent on
GitHub-hosted runners. It pushes each build by immutable digest first, blocks on
High or Critical Trivy findings, creates an SPDX JSON SBOM and maximum-mode
BuildKit provenance, then signs and verifies the digest with GitHub OIDC and
Cosign. Only verified digests receive the version and source-commit tags.

The workflow publishes a `release-manifest-<tag>` artifact containing each
GHCR repository and digest, source commit, workflow signing identity, and build
time. All client image values must come from that manifest and retain the
`ghcr.io/...@sha256:...` form. Tags are discovery labels, never deployment
coordinates. The source-free bundle under `deploy/client/` verifies the exact
workflow identity before pulling any image.

## Production

Production uses the pull-only, Cosign-verified bundle in `deploy/client/`.
Follow the [client deployment bundle runbook](/operations#new-customer-or-installation);
do not build the source Compose stack on a client production host.

## Operator workspace

Vendor operators onboard and maintain client deployments in the protected
control-plane UI. Create the client, current contract, and deployment; open the
deployment workspace; issue its one-time install token; then register, configure,
review, sign, and verify its heartbeat. Use the same workspace to issue a new
immutable signed version after a contract, configuration, or approved-release
change. See [operator onboarding, signing, and recovery](/operations#operator-workspace-client-onboarding-and-signing).

This is an operator workflow, not a customer or integration-partner interface.
No documentation update records a live deployment or signed release; only
completed `release-images` runs append to the [release log](https://github.com/Super-ERP/crm-v2/blob/main/docs/operations/release-log.md).