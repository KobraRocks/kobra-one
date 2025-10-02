# KobraOne — Thruster‑first Dev Bootstrap

This sets up a **pure Thruster + Puma** developer environment under `/kobra-one/`, creates a sample tenant, writes `.env.development` for each Rails app, and a `Procfile.dev` that launches both apps with **Thruster** (HTTP‑only in dev by default). It also writes optional dev initializers to enable `X‑Sendfile` and subdomain hosts.

> Thruster is an HTTP/2 proxy that sits in front of Puma. It adds HTTP caching for public assets and **X‑Sendfile** support for efficient static file delivery. In dev, we run it HTTP‑only (no TLS) and keep things simple.

---

## 1) Script: `bootstrap_dev_thruster.sh`

> Save as `bootstrap_dev_thruster.sh`, `chmod +x bootstrap_dev_thruster.sh`, then run it. Optionally pass a custom base path.

```bash
#!/usr/bin/env bash
set -euo pipefail

# ============================================
# KobraOne Dev Bootstrap — Thruster edition
# --------------------------------------------
# Usage:
#   ./bootstrap_dev_thruster.sh               # uses /kobra-one as base
#   ./bootstrap_dev_thruster.sh ~/kobra-one   # custom base dir
#
# Creates:
#   BASE/{Kengine,Kockpit,shared,http/<tenant>/{public,protected}}
#   BASE/Kengine/.env.development
#   BASE/Kockpit/.env.development
#   BASE/Procfile.dev
#   Optional: dev initializers enabling X-Sendfile + hosts
#
# Notes:
# - This does NOT create Rails apps. Place your apps in:
#     BASE/Kengine
#     BASE/Kockpit
#   (or run `rails new` in each folder).
# - Idempotent: safe to re-run.
# - Runs Thruster in HTTP-only mode for dev (no root, no TLS).
# ============================================

BASE="${1:-/kobra-one}"

# ---- Constants ----
SYSTEM_TENANT_HOST="kobra.localhost"
KOCKPIT_HOST="kockpit.kobra.localhost"
SAMPLE_TENANT="alice.kobra.localhost"

TENANT_ROOT="$BASE/http"
KENGINE_DIR="$BASE/Kengine"
KOCKPIT_DIR="$BASE/Kockpit"
SHARED_DIR="$BASE/shared"

# Thruster (front) and Puma (target) ports
KENGINE_THRUSTER_HTTP=3100
KENGINE_THRUSTER_TARGET=3200
KOCKPIT_THRUSTER_HTTP=3101
KOCKPIT_THRUSTER_TARGET=3201

# ---- Helpers ----
ensure_dir() { mkdir -p "$1"; }
write_file_if_absent() { [[ -f "$1" ]] || { mkdir -p "$(dirname "$1")"; printf "%s" "$2" > "$1"; }; }
append_line_once() { local file="$1" line="$2"; grep -qF -- "$line" "$file" 2>/dev/null || echo "$line" >> "$file"; }

# ---- Create base layout ----
echo "==> Base: $BASE"
ensure_dir "$BASE" "$TENANT_ROOT" "$KENGINE_DIR" "$KOCKPIT_DIR" "$SHARED_DIR"

# ---- /etc/hosts suggestion (no sudo required) ----
HOSTS_LINE="127.0.0.1 $SYSTEM_TENANT_HOST $KOCKPIT_HOST $SAMPLE_TENANT"
echo "==> Add this to /etc/hosts if not present:"
echo "    $HOSTS_LINE"

# ---- Sample tenant data ----
for T in "$SAMPLE_TENANT"; do
  echo "==> Tenant dirs: $T"
  ensure_dir "$TENANT_ROOT/$T/public"
  ensure_dir "$TENANT_ROOT/$T/protected"
  write_file_if_absent "$TENANT_ROOT/$T/public/index.html" "<!doctype html><html><head><meta charset=\"utf-8\"><title>$T</title></head><body style=\"font-family:sans-serif\"><h1>It works 🎉</h1><p>Tenant: <b>$T</b></p><p>Public root: <code>$TENANT_ROOT/$T/public</code></p></body></html>"
  write_file_if_absent "$TENANT_ROOT/$T/protected/secret.txt" "Top secret for $T"
  : > "$TENANT_ROOT/$T/database.sqlite"
done

# ---- .env files for both apps ----
echo "==> Writing .env.development"
write_file_if_absent "$KENGINE_DIR/.env.development" \
"RAILS_ENV=development
TENANT_ROOT=$TENANT_ROOT
SYSTEM_TENANT_HOST=$SYSTEM_TENANT_HOST
THRUSTER_HTTP_PORT=$KENGINE_THRUSTER_HTTP
THRUSTER_TARGET_PORT=$KENGINE_THRUSTER_TARGET
"

write_file_if_absent "$KOCKPIT_DIR/.env.development" \
"RAILS_ENV=development
THRUSTER_HTTP_PORT=$KOCKPIT_THRUSTER_HTTP
THRUSTER_TARGET_PORT=$KOCKPIT_THRUSTER_TARGET
"

# ---- Procfile.dev (thruster runs Puma for you) ----
echo "==> Writing Procfile.dev"
write_file_if_absent "$BASE/Procfile.dev" \
"web-kengine: bash -lc 'cd \"$KENGINE_DIR\" && export $(grep -v \"^#\" .env.development | xargs) && thrust bin/rails server'
web-kockpit: bash -lc 'cd \"$KOCKPIT_DIR\" && export $(grep -v \"^#\" .env.development | xargs) && thrust bin/rails server'
"

# ---- Optional: write dev initializers if apps exist ----
if [[ -d "$KENGINE_DIR/config" ]]; then
  echo "==> Adding optional dev initializer to Kengine"
  write_file_if_absent "$KENGINE_DIR/config/initializers/dev_kobraone.rb" \
"if Rails.env.development?
  # Allow subdomains like alice.kobra.localhost
  Rails.application.config.hosts += ['kockpit.kobra.localhost', /.*\\.kobra\\.localhost$/]

  # Let Thruster serve files via X-Sendfile when you call send_file(..., x_sendfile: true)
  Rails.application.config.action_dispatch.x_sendfile_header = 'X-Sendfile'

  # Serve public files in dev (Thruster adds basic caching in front)
  Rails.application.config.public_file_server.enabled = true
end
"
fi

if [[ -d "$KOCKPIT_DIR/config" ]]; then
  echo "==> Adding optional dev initializer to Kockpit"
  write_file_if_absent "$KOCKPIT_DIR/config/initializers/dev_kobraone.rb" \
"if Rails.env.development?
  Rails.application.config.hosts += ['kockpit.kobra.localhost']
end
"
fi

# ---- README.dev ----
write_file_if_absent "$BASE/README.dev.md" \
"# KobraOne Dev (Thruster)

## Run
1) In each app folder, ensure Rails app exists and run: \`bundle install\`.
2) Start both apps using a Procfile runner (foreman/overmind):

   \`cd $BASE && foreman start -f Procfile.dev\`

Visit:
- http://$SAMPLE_TENANT:$KENGINE_THRUSTER_HTTP/
- http://$SYSTEM_TENANT_HOST:$KENGINE_THRUSTER_HTTP/
- http://$KOCKPIT_HOST:$KOCKPIT_THRUSTER_HTTP/

## Protected files (dev)
- After authenticating, return `send_file(path, x_sendfile: true)` from a controller. Thruster will stream the file efficiently.
- Place protected files under: \`$TENANT_ROOT/<host>/protected\`.

## Notes on HTTPS in dev
- Thruster provisions TLS via Let's Encrypt in prod, but for dev we run HTTP-only.
- If you must test HTTPS-only features (e.g., Service Workers outside pure `localhost`), you can:
  - Temporarily run Puma with mkcert TLS *without* Thruster; or
  - Use a local TLS terminator (e.g., Caddy) in front of Thruster with mkcert.

"

echo "==> Done. Next: put Rails apps into $KENGINE_DIR and $KOCKPIT_DIR, then run Foreman."
```

---

## 2) Minimal controller example for protected files (Kengine)

Add a route and controller to test `X‑Sendfile` with Thruster:

```ruby
# config/routes.rb
get "/protected-demo", to: "protected_demo#show"
```

```ruby
# app/controllers/protected_demo_controller.rb
class ProtectedDemoController < ApplicationController
  before_action :authenticate_user! # or stub in dev

  def show
    tenant_root = ENV.fetch("TENANT_ROOT")
    path = File.join(tenant_root, request.host, "protected/secret.txt")
    raise ActionController::RoutingError, "Not Found" unless File.file?(path)

    # Tell Thruster to take over the file send (fast, low Ruby CPU)
    send_file path, x_sendfile: true, disposition: :inline, type: "text/plain"
  end
end
```

---

## 3) Tips

* **No root needed**: We bind Thruster to high HTTP ports (3100/3101).
* **Multi‑tenant hosts**: The initializer allows `*.kobra.localhost`. Add the entries to `/etc/hosts`.
* **Parity with prod**: In production you’ll run Thruster with `TLS_DOMAIN` to get automatic Let’s Encrypt certificates and still enjoy asset caching + X‑Sendfile.

---

### Quick commands

```bash
cd /kobra-one && foreman start -f Procfile.dev
# Kengine served at:  http://alice.kobra.localhost:3100/
# Kockpit at:         http://kockpit.kobra.localhost:3101/
```
