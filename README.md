# KobraOne Architecture

**KobraOne** is a multi-tenant SaaS platform built on **Rails**.  
Each **tenant = hostname** and owns:
- Its own **SQLite database**  
- Its own **static file folders** (`public/`, `protected/`)

All tenants share the same Rails runtime (**Kengine**) for application features.  
A separate Rails app (**Kockpit**) manages tenant lifecycle.  
The **master domain** (`kobra.rocks`) acts as sales & onboarding front.

---

## Components

### 1. Kengine (Data Plane)
- Rails app serving **all tenant sites** (`*.kobra.rocks` and custom domains).
- Switches database connection **per request**, keyed by `Host` header.
- Serves tenant-specific static files:
  - `/srv/http/{tenant}/public` → directly cached/compressed by proxy
  - `/srv/http/{tenant}/protected` → served via Rails authz + `X-Sendfile`
- Exposes reusable customer modules (checkout, billing portal, CRM-lite).
- Special-cased: `kobra.rocks` is the **system tenant**.  
  - It can access onboarding endpoints (e.g., `POST /offer/landing_page`).  
  - Those endpoints internally call **Kockpit** to create new tenants.

### 2. Kockpit (Control Plane)
- Rails app at `kockpit.kobra.rocks`.
- Owns the **global admin DB** (tenants table, lifecycle state).
- Provides APIs to:
  - Provision tenants (`POST /api/tenants`)  
  - Migrate/seeds tenant DBs  
  - Enable/disable tenants  
  - Export/backup tenants
- Implements provisioning jobs:
  1. Create `/srv/http/{host}/{public,protected}`
  2. Create `/srv/http/{host}/database.sqlite`
  3. Run tenant migrations/seeds
  4. Update status → `active`

### 3. Front Proxy (Thruster)
- Runs in front of Puma servers (one per app).
- Provides:
  - **TLS termination** (auto Let’s Encrypt per host)
  - **HTTP/2**
  - **Static file caching & compression**
  - **X-Sendfile support** for protected assets
- Routes traffic by `Host`:
  - `kobra.rocks` → static sales site (or Kengine “system tenant”)
  - `*.kobra.rocks` → Kengine app
  - `kockpit.kobra.rocks` → Kockpit app

---

## Domain Layout

- `kobra.rocks` → Sales / onboarding (system tenant in Kengine)  
- `kockpit.kobra.rocks` → Tenant management (Kockpit)  
- `*.kobra.rocks` → Tenants (Kengine)  
- `customdomains.com` → Supported by provisioning; DNS must point to server

---

## Request Lifecycle

1. Customer visits `kobra.rocks` and purchases a plan (`POST /offer/landing_page`).  
2. Kengine (system tenant) forwards the request internally to **Kockpit**.  
3. Kockpit provisions the tenant (filesystem + SQLite + migrations).  
4. Tenant is marked `active`.  
5. Customer can access `https://{tenant}.kobra.rocks`, served by Kengine with the correct DB.

---

## File System Layout

```

/srv/http/{tenant}/
public/               # Public static files
protected/            # Auth-gated static files
database.sqlite       # Tenant-specific SQLite DB
/srv/kobraone/
Kengine/              # Rails app (tenants data plane)
Kockpit/              # Rails app (tenant control plane)
shared/               # Shared env files, scripts, etc.

```

---

## Operational Model

- **DB switching:** Middleware in Kengine switches AR connection per request host.
- **Tenant provisioning:** Asynchronous job in Kockpit (Solid Queue).
- **Backups:** Nightly rsync of `/srv/http/*/database.sqlite` and static folders.
- **TLS:** Handled automatically by Thruster; one cert per host.
- **Security:**
  - Only `kobra.rocks` may hit provisioning endpoints.
  - Host header validation ensures requests only served for known tenants.
  - Protected static files require authz and are streamed by Thruster via `X-Sendfile`.

---

## Deployment (Arch Linux)

- **System packages:** `ruby`, `nodejs`, `yarn`, `sqlite`, `libvips`, `openssl` …
- **User:** `deploy` under `/srv/`
- **Systemd services:**
  - `puma-kengine.service` / `thruster-kengine.service`
  - `puma-kockpit.service` / `thruster-kockpit.service`
- **Release flow:**
  1. Sync code to `/srv/kobraone/{Kengine,Kockpit}`
  2. `bundle install --deployment`
  3. Run migrations on Kockpit global DB
  4. Restart Puma services (Thruster stays running)

---

## Advantages

- **Simple tenant isolation**: one SQLite file per tenant
- **Low ops overhead**: no DB server to manage
- **Clean separation**:
  - Kengine = customer features
  - Kockpit = control plane
- **Scalable growth path**:
  - Small tenants live in this shared instance
  - Mature tenants can be migrated to dedicated servers easily

---

## Next Steps

- [ ] Implement system-only guard for provisioning endpoints in Kengine  
- [ ] Finalize Kengine → Kockpit API client with API keys  
- [ ] Add tenant migration rake task in Kockpit  
- [ ] Configure Thruster units for apex, wildcard, and kockpit domains  
- [ ] Document onboarding flow for support team
