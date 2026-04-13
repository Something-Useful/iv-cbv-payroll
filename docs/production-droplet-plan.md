# Production deployment on DigitalOcean droplet

Status: planning
Last updated: 2026-04-13

## Current state

The app runs on a DO droplet via Docker with sandbox Pinwheel/Argyle tokens.
PostgreSQL runs in Docker on the same droplet.

## What needs to happen

### 1. HTTPS + domain

**Required.** Production Pinwheel/Argyle webhooks require HTTPS. Rails enforces
`force_ssl = true` in production.

- Point a domain (or subdomain) at the droplet IP
- Set up a reverse proxy (nginx or Caddy) with Let's Encrypt SSL
- Caddy is simpler (auto-renews certs); nginx is more battle-tested
- Proxy `https://yourdomain.com` -> `localhost:3000`
- Set `DOMAIN_NAME=yourdomain.com` in env

The health check endpoint (`/health`) is excluded from SSL redirect for
load balancer use.

### 2. Production API tokens

Swap sandbox tokens for production ones:

**Pinwheel:**
- `PINWHEEL_API_TOKEN_PRODUCTION` (or per-agency: `LA_LDH_PINWHEEL_ENVIRONMENT=production`)

**Argyle:**
- `ARGYLE_API_TOKEN_PRODUCTION_ID`
- `ARGYLE_API_TOKEN_PRODUCTION_SECRET`
- `ARGYLE_PRODUCTION_WEBHOOK_SECRET`

**NSC (if using education verification):**
- `NSC_API_URL`, `NSC_TOKEN_URL`, `NSC_CLIENT_ID`, `NSC_CLIENT_SECRET`, `NSC_ACCOUNT_ID`

### 3. Register webhook URLs with providers

In the Pinwheel and Argyle dashboards, register your production webhook endpoints:
- Pinwheel: `https://yourdomain.com/webhooks/pinwheel/events`
- Argyle: `https://yourdomain.com/webhooks/argyle/events`

These receive account sync events. Without them, the income verification flow
won't complete (the app waits for webhook callbacks to know when data is ready).

### 4. Rails production secrets

```bash
# Generate with: bin/rails secret
SECRET_KEY_BASE=<64-char-hex-string>

# Active Record encryption (encrypts sensitive fields at rest)
# Generate each with: bin/rails db:encryption:init
ACTIVE_RECORD_ENCRYPTION_PRIMARY_KEY=<value>
ACTIVE_RECORD_ENCRYPTION_DETERMINISTIC_KEY=<value>
ACTIVE_RECORD_ENCRYPTION_KEY_DERIVATION_SALT=<value>
```

These must remain stable. Changing encryption keys invalidates existing
encrypted data in the database.

### 5. Background worker process

The app uses SolidQueue (database-backed job queue) for async work like
transmitting reports and syncing education data. You need a separate process:

```bash
docker run -d --env-file .env.production iv-cbv-payroll:latest bin/rails solid_queue:start
```

This can run on the same droplet. SolidQueue uses PostgreSQL (no Redis needed
for jobs).

The job dashboard is at `/jobs`, protected by HTTP basic auth:
- `MISSION_CONTROL_USER`
- `MISSION_CONTROL_PASSWORD`

### 6. PostgreSQL persistence + backups

If PostgreSQL runs in Docker, ensure the data volume persists across container
restarts. Set up periodic backups:

```bash
# Example: daily pg_dump to a file
docker exec postgres pg_dump -U app app > backup_$(date +%Y%m%d).sql
```

Consider switching to DigitalOcean Managed Databases ($15/mo) for automatic
backups, failover, and not worrying about volume management.

### 7. Email (AWS SES) — if using invitations

The app sends invitation emails via AWS SES v2. Required if caseworkers
will create invitations that email links to applicants.

- `AWS_REGION`
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- Verify the sending domain in the SES console
- Update the default sender (currently a placeholder in Devise config)

**Not needed if** you're only using generic links or the demo launcher.

### 8. File storage (AWS S3) — if using ActiveStorage

Production uses S3 for file storage (uploaded documents, generated reports).

- `BUCKET_NAME` — S3 bucket name
- Same AWS credentials as SES

**Not needed immediately** if the features you're piloting don't involve
file uploads.

### 9. New tenant configuration

For each pilot state or Propel, add an entry to `config/client-agency-config.yml`.
Use `sandbox` as a template. Key decisions per tenant:

- `id` — short slug (e.g., `az_des`, `propel`)
- `transmission_method` — how reports reach the agency: `json`, `sftp`, or `shared_email`
- `staff_portal_enabled` — whether caseworkers log in to create invitations
- `sso` — Azure AD config (if staff portal is enabled)
- `activity_types` — which flows are available (education, employment, etc.)
- `applicant_attributes` — which fields the invitation form requires
- Logo SVG in `app/assets/images/`
- Agency-specific locale strings in `config/locales/en.yml`

Generic links work without SSO: `/activities/links/<agency_id>` and
`/cbv/links/<agency_id>`.

### 10. Host authorization

Rails rejects requests from unknown hosts in production. The app auto-allows
`DOMAIN_NAME` and all agency domains from the config. Make sure each agency's
`agency_domain` env var is set, or requests to those domains will 403.

---

## Minimal viable production checklist

For an initial pilot with generic links (no caseworker portal, no email
invitations):

- [ ] Domain + HTTPS (nginx/Caddy + Let's Encrypt)
- [ ] `DOMAIN_NAME` env var
- [ ] `SECRET_KEY_BASE` generated and set
- [ ] Active Record encryption keys generated and set
- [ ] `RAILS_ENV=production`
- [ ] Production Pinwheel/Argyle tokens
- [ ] Webhook URLs registered with providers
- [ ] SolidQueue worker process running
- [ ] At least one tenant in `client-agency-config.yml` with `pilot_ended: false`
- [ ] PostgreSQL data volume persisted
- [ ] Database migrated (`bin/rails db:migrate`)

## Full production checklist

Everything above, plus:

- [ ] AWS SES configured for invitation emails
- [ ] AWS S3 bucket for file storage
- [ ] SSO configured for caseworker portal (per agency)
- [ ] Transmission method configured per agency (SFTP/JSON API)
- [ ] Database backups (managed DB or cron pg_dump)
- [ ] Monitoring (New Relic: `NEWRELIC_KEY`)
- [ ] Log aggregation
- [ ] Uptime monitoring on `/health`

## Environment variable reference

Full list of env vars needed for production:

```bash
# Core
RAILS_ENV=production
SECRET_KEY_BASE=
DOMAIN_NAME=
RAILS_LOG_TO_STDOUT=true
RAILS_SERVE_STATIC_FILES=true

# Database
DB_HOST=
DB_NAME=app
DB_USERNAME=
DB_PASSWORD=

# Encryption
ACTIVE_RECORD_ENCRYPTION_PRIMARY_KEY=
ACTIVE_RECORD_ENCRYPTION_DETERMINISTIC_KEY=
ACTIVE_RECORD_ENCRYPTION_KEY_DERIVATION_SALT=

# AWS (for SES + S3)
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
BUCKET_NAME=

# Pinwheel (per-environment)
PINWHEEL_API_TOKEN_PRODUCTION=
# Or per-agency: LA_LDH_PINWHEEL_ENVIRONMENT=production

# Argyle (per-environment)
ARGYLE_API_TOKEN_PRODUCTION_ID=
ARGYLE_API_TOKEN_PRODUCTION_SECRET=
ARGYLE_PRODUCTION_WEBHOOK_SECRET=

# Job dashboard
MISSION_CONTROL_USER=
MISSION_CONTROL_PASSWORD=

# Per-agency domains
SANDBOX_DOMAIN_NAME=
# LA_LDH_DOMAIN_NAME=
# NH_DHHS_DOMAIN_NAME=

# Per-agency SSO (if staff portal enabled)
# AZURE_SANDBOX_CLIENT_ID=
# AZURE_SANDBOX_CLIENT_SECRET=
# AZURE_SANDBOX_TENANT_ID=

# Analytics/monitoring (optional)
# NEWRELIC_KEY=
# MIXPANEL_TOKEN=
```
