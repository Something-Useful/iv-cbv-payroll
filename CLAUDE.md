# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Current Engagement

This fork is being adapted for CCX (https://www.communitycx.co/). The original codebase was
built by Nava/CMS. CCX is the platform operator — application-level branding (logos, name,
chrome) should be CCX, not Emmy/Nava. Individual state agencies and potentially Propel are
the tenants, configured via `client-agency-config.yml`.

## Repository Structure

Monorepo with three top-level directories:
- `app/` — Ruby on Rails application (the main codebase)
- `infra/` — Terraform infrastructure
- `docs/` — Documentation

**All development commands run from the `app/` directory.** Run Rails commands locally (not Docker) unless explicitly told otherwise.

## The Application: Eligibility Made Easy (Emmy)

Emmy is a multi-tenant consent-based verification (CBV) tool that lets benefit applicants verify income and community engagement via payroll providers and educational records. It's piloted with government agencies (LA Dept of Health, NH DHHS, plus a sandbox).

### Two main user flows

1. **CBV Flow** (`/cbv/*`) — Applicants verify income by connecting to payroll providers (Pinwheel, Argyle). Caseworkers create invitations; applicants link accounts, review data, and submit reports transmitted to agencies.
2. **Activities Flow** (`/activities/*`) — Applicants document community engagement (employment, education, volunteering, job training) with monthly hour tracking.

### Key architectural concepts

- **Client agencies** configured in `config/client-agency-config.yml` — each has its own provider environments, transmission methods (JSON API, SFTP, email), and locale overrides.
- **Aggregators** (Pinwheel, Argyle) — external payroll data providers accessed via webhook-driven sync. NSC (National Student Clearinghouse) for education data.
- **Report transmission** — `Transmitter` service with per-agency strategies (`transmitters/` directory): SFTP with optional GPG encryption, JSON API with HMAC, or email.
- **Flow navigators** (`CbvFlowNavigator`, `ActivityFlowNavigator`) — service objects that manage step progression through each flow.
- **ViewComponent** for reusable UI components (`app/components/`), with Lookbook gallery at `/lookbook` in dev.
- **SolidQueue** for background jobs; dashboard at `/jobs`.

## Common Commands (from `app/`)

```bash
# Development
bin/dev                     # Start all services (Rails, esbuild, CSS, ngrok, worker)
bin/rails server            # Rails server only
bin/rails console           # Rails console

# Testing
bin/rspec                            # All tests
bin/rspec spec/path/to_spec.rb       # Single file
bin/rspec spec/path/to_spec.rb:42    # Single example by line
E2E_RUN_TESTS=1 bin/rspec spec/e2e/  # E2E tests (replay mode)
E2E_RECORD_MODE=1 bin/rspec spec/e2e/your_spec.rb  # E2E record mode
npm test                             # JavaScript tests (Vitest)

# Linting
make lint                   # RuboCop with auto-fix
./bin/rubocop               # RuboCop (no auto-fix: ./bin/rubocop -l)
npm run format              # Prettier
pre-commit                  # All linters (rubocop, erblint, i18n-tasks, prettier, etc.)

# Database
bin/rails db:migrate
bin/rails db:seed
RAILS_ENV=test bin/rails db:test:prepare

# i18n
i18n-tasks missing          # Show missing translation keys
i18n-tasks normalize        # Sort locale files
i18n-tasks unused           # Show unused keys
```

## Coding Conventions

- **Ruby:** RuboCop with rails-omakase base. 2-space indentation.
- **JS/TS:** Prettier — no semicolons, double quotes, 100-char width, 2-space indent.
- **Naming:** snake_case (Ruby), camelCase (JS), kebab-case (Stimulus controllers).
- **Controllers:** Use `before_action` callbacks for guard redirects, not inline redirects.
- **ERB/HTML:** Each HTML tag on its own line (opening, content, closing). Do not use margin/padding utility classes unless explicitly requested. Do not use `usa-prose` classes unless required by design.
- **CSS:** Uses USWDS (US Web Design System).

## Testing Conventions

- RSpec for Ruby, Vitest for JS. Test files: `_spec.rb` / `.test.ts`.
- **Prefer controller tests** over request specs.
- E2E tests in `spec/e2e/` use Capybara/Selenium with a three-layer recording system (modal callbacks, VCR HTTP requests, ngrok webhooks). Use `verify_page` after each navigation in E2E specs.
- Use `Timecop` for time-freezing in controller specs (via `around` blocks).
- Factories in `spec/factories/` (FactoryBot). Use `let` for object setup, `before` for shared context.

## i18n

- English (`en.yml`) and Spanish (`es.yml`). When adding strings, add to `en.yml` only — do not touch `es.yml` unless asked.
- Prefer relative i18n keys in views (e.g., `t(".title")`).
- Agency-specific copy uses `client_agency_translation(".key")` helper, with agency IDs as sub-keys in locale files.

## Commit Messages

- If following a ticket (e.g., FFS-1234), prefix: `FFS-1234: Add timeout page title`.
- Call out migrations, feature flags, env var changes, and operational impacts.

## Environment

- Secrets go in `.env.local` (never committed). Test overrides in `.env.test.local`.
- Inline `<script>` tags require nonce: `<%= javascript_tag nonce: true %>` (CSP enforced).
