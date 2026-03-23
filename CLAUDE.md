# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a **documentation repository** for **Front-API / LaravelCiaf** — a B2B SaaS platform for financial reconciliation automation in the Argentine retail/commerce sector. The actual application code lives elsewhere (Hostinger VPS); this repo contains PRDs, design specs, deployment guides, and UX/UI documentation.

## Application Stack (documented, not in this repo)

- **PHP 8.2 + Laravel 12**
- **TALL Stack:** Tailwind CSS 3, Alpine.js 3, Livewire 3.7
- **Database:** MySQL 8 (Hostinger VPS)
- **Auth/RBAC:** Laravel Breeze + Spatie
- **External:** Python server for reconciliation processing

## Local Development Commands (for the application)

```bash
php artisan serve          # Start local server
php artisan test           # Run test suite
npm run dev                # Compile frontend assets
```

## Production Deployment (Hostinger VPS)

Full runbook is in `docs/contexto_proyecto/DEPLOY_HOSTINGER.md`. Key phases:

```bash
php artisan down
git pull origin main
composer install --no-dev --optimize-autoloader
npm ci && npm run build
php artisan migrate --force
php artisan config:cache && php artisan route:cache && php artisan view:cache
php artisan queue:restart
php artisan up
```

## Documentation Structure

All docs are in `docs/contexto_proyecto/`:

| File | Content |
|------|---------|
| `PRD_MAESTRO.md` | Master PRD: modules M-1 to M-12, epics E0-E5, user stories with acceptance criteria |
| `usuarios.md` | RBAC: 4 roles (Super Admin, Manager, Programador, Operador) with permission matrix |
| `use_cases.md` | 11 use cases (CU-1 to CU-11) covering operator/programmer workflows |
| `DEPLOY_HOSTINGER.md` | Production deployment runbook with rollback procedures |
| `plan-conciliacion-database.md` | DB schema for reconciliation results (8 tables, upsert strategy) |
| `guia_base_ux_ui.md` | Design system: 50+ views, design tokens, known UI inconsistencies |
| `ux_ui_design_epica_o_us_0_1.md` | Template for documenting UX/UI specs per user story |
| `ux_ui_design_us_0_2_*.md` | Design spec: Operator-to-Team assignment (Livewire `TeamOperatorAssignment`) |
| `ux_ui_design_us_0_3_*.md` | Design spec: Client-to-Team assignment |

`index.html` is a web dashboard providing an overview of the project status.

## Architecture Overview

```
Modules (M-1 to M-12):
  M-1:  Auth & RBAC
  M-2:  Client Management
  M-3:  API Integration Catalog (banking adapters)
  M-4:  ETL Business Rules (Pyodide/Monaco editor)
  M-5:  Workflow Core (types, batches, executions)
  M-6:  Conciliation Workflow (daily, main use case)
  M-7:  Cash Audit Workflow (monthly)
  M-8:  PDF Report Generation
  M-9:  Programmer Dashboard & Metrics
  M-10: Admin Panel
  M-11: Teams & Adapters [active development — E0/E1]
  M-12: Lead Enrichment [not documented]

Epic order: E0 → E1 → E2 & E3 (parallel) → E4 → E5
```

**Core workflow:**
1. Operator creates workflow request (Excel upload or adapter config)
2. Programmer reviews, dry-runs, activates workflow
3. System auto-generates daily tasks for operators
4. Operators upload files via API adapter, RPA, or manual Excel
5. Python server processes and returns reconciliation results
6. Programmer analyzes results, generates PDF reports

## User Roles

- **Super Admin:** Full system control, user & team management
- **Manager:** Client & adapter config (no user management)
- **Programador (Analyst):** Workflow design, execution, analysis, PDF reports
- **Operador:** Task execution, adapter setup, workflow requests (scoped to team)

## Writing New Design Specs

Use `ux_ui_design_epica_o_us_0_1.md` as the template when documenting UX/UI specifications for new user stories.
