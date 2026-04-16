# Grafana Public Dashboard Lockdown

## Goal

Stop exposing the full Grafana UI to the internet after the dashboard is configured, while still allowing public access to a Grafana public dashboard.

This approach:

- blocks normal Grafana UI access from the public internet
- keeps the Grafana container private
- exposes only the URL paths needed for the public dashboard
- uses a reverse proxy as the public entry point

## Scope

This runbook assumes:

- Grafana is running in Docker Compose
- the dashboard is already configured
- a Grafana public dashboard URL already exists
- the VPS runs Ubuntu

## High-Level Plan

1. Stop publishing Grafana directly on port `3000`.
2. Confirm Grafana is no longer reachable from the public internet.
3. Install a reverse proxy.
4. Expose only the public dashboard paths through the reverse proxy.
5. Open only `80` and `443` in UFW.
6. Keep Grafana private behind Docker networking.

## Why UFW Alone Is Not Enough

Docker port publishing can make a container reachable even when UFW rules do not look like they explicitly allow that port.

Because of that, the correct fix is:

- remove Docker host port publishing for Grafana
- put a reverse proxy in front
- limit the proxy to only the required public dashboard routes

## Step 1: Remove Direct Grafana Port Publishing

Edit `docker-compose.yml` and find the `grafana` service.

Remove this section:

```yaml
ports:
  - "3000:3000"
```

The `grafana` service should look like this afterwards:

```yaml
grafana:
  build:
    context: .
    dockerfile: ./docker/grafana/Dockerfile.grafana
  restart: unless-stopped
  networks:
    - mesh-bridge
  volumes:
    - grafana_data:/var/lib/grafana
    - ./docker/grafana/provisioning:/etc/grafana/provisioning
  depends_on:
    - timescaledb
```

Do not add another public port mapping for Grafana after this.

## Step 2: Restart The Stack

From the compose directory:

```bash
docker compose down
docker compose up -d
```

Check status:

```bash
docker compose ps
```

Expected result:

- `grafana` is running
- `grafana` no longer shows `0.0.0.0:3000->3000/tcp`

## Step 3: Confirm Grafana Is No Longer Public

On the VPS:

```bash
ss -tulpn | grep 3000
```

Expected:

- no host listener for Grafana on `3000`

From another machine, opening `http://YOUR_VPS_IP:3000` should fail.

## Step 4: Choose A Public Hostname

Create a DNS record such as:

- `clima.example.com -> YOUR_VPS_IP`

Using a real hostname is strongly recommended so the reverse proxy can provide HTTPS.

## Step 5: Install Caddy

Install Caddy on Ubuntu:

```bash
apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt update
apt install -y caddy
```

## Step 6: Configure Caddy To Allow Only Public Dashboard Paths

Edit the Caddy config:

```bash
nano /etc/caddy/Caddyfile
```

Example configuration:

```caddy
clima.example.com {
    @allowed path /public-dashboards/* /api/public/* /public/build/* /public/plugins/* /public/img/* /public/fonts/* /favicon.ico

    handle @allowed {
        reverse_proxy 127.0.0.1:3000
    }

    handle {
        respond "Forbidden" 403
    }
}
```

This allows:

- the public dashboard route
- the public API routes needed by the dashboard
- static assets such as JavaScript, plugins, fonts, and images

This blocks:

- `/login`
- `/explore`
- `/dashboards`
- other non-public Grafana routes

## Step 7: Make Grafana Reachable To The Reverse Proxy

Because Grafana is no longer published on a host port, Caddy needs a way to reach it.

There are two practical options.

### Option A: Rebind Grafana To Localhost Only

Publish Grafana only to loopback:

```yaml
ports:
  - "127.0.0.1:3000:3000"
```

This is the simplest option if Caddy runs on the VPS host.

Result:

- the public internet cannot hit `:3000`
- Caddy can still reverse proxy to `127.0.0.1:3000`

If you choose this option, use this under the `grafana` service instead of no `ports` block:

```yaml
ports:
  - "127.0.0.1:3000:3000"
```

This is usually the easiest and safest approach.

### Option B: Put Caddy In Docker Too

Run Caddy in the same Docker network and reverse proxy to `grafana:3000`.

This works well too, but it is more moving parts than needed for a small VPS.

For this runbook, prefer Option A.

## Step 8: Restart After Choosing The Localhost Binding

If using Option A, update `docker-compose.yml`, then restart:

```bash
docker compose down
docker compose up -d
docker compose ps
```

Expected result:

- `grafana` shows `127.0.0.1:3000->3000/tcp`
- Grafana is not reachable from outside on `YOUR_VPS_IP:3000`
- Caddy can still proxy to it

## Step 9: Validate Caddy Configuration

Check the config:

```bash
caddy validate --config /etc/caddy/Caddyfile
```

Reload:

```bash
systemctl reload caddy
systemctl status caddy
```

## Step 10: Open Only The Required Firewall Ports

Set UFW for the final public shape:

```bash
ufw allow OpenSSH
ufw allow 80/tcp
ufw allow 443/tcp
ufw delete allow 3000/tcp || true
ufw status
```

Do not allow `5432/tcp`.

If you no longer need public Mosquitto access, review whether `1883/tcp` should remain open as well.

## Step 11: Test The Public Dashboard

Open:

```text
https://clima.example.com/public-dashboards/YOUR_PUBLIC_DASHBOARD_ID
```

Verify:

- the public dashboard loads
- charts render
- static assets load successfully

Then test blocked paths:

```text
https://clima.example.com/login
https://clima.example.com/explore
https://clima.example.com/d/YOUR_DASHBOARD_UID/YOUR_DASHBOARD_NAME
```

Expected result:

- these should return `403`

## Step 12: If Assets Fail To Load

If the public dashboard page opens but looks broken, check browser developer tools and add any missing paths to the Caddy allowlist.

Typical paths already covered:

- `/public-dashboards/*`
- `/api/public/*`
- `/public/build/*`
- `/public/plugins/*`
- `/public/img/*`
- `/public/fonts/*`
- `/favicon.ico`

If your Grafana version needs something else, add only that path, not a broad catch-all proxy rule.

## Final Recommended State

- live Grafana admin UI is not public
- public dashboard path is reachable
- PostgreSQL is not public
- Grafana remains manageable from the VPS host
- HTTPS is terminated by Caddy
