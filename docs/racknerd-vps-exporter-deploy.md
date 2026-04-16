# RackNerd VPS Deployment Runbook

## Scope

This runbook installs the Meshtastic metrics exporter stack on a small VPS and prepares it for production use with:

- Docker Engine
- Docker Compose plugin
- Meshtastic metrics exporter
- TimescaleDB
- Grafana

It assumes:

- the VPS runs Debian or Ubuntu
- Mosquitto is already running on the VPS
- the exporter will connect to that existing Mosquitto broker
- telemetry is collected from public `LongFast`
- storage will later be filtered by Meshtastic source node ID
- Grafana stays private and is used mainly to build dashboards and create snapshots

## High-Level Plan

1. Prepare the VPS.
2. Install Docker and Compose.
3. Copy or clone the exporter repo onto the VPS.
4. Configure `.env` for the VPS broker and topic.
5. Start the stack.
6. Verify Grafana, TimescaleDB, and exporter health.
7. Keep Grafana private and use snapshots for no-login sharing when needed.
8. Import or restore local data if desired.
9. Keep the Docker volumes for persistence.

## Step 1: Log In And Inspect The VPS

```bash
ssh root@YOUR_VPS_IP
```

Check the system:

```bash
uname -a
cat /etc/os-release
free -h
df -h
```

Minimum practical checks:

- at least 2 GB RAM is preferable
- enough disk space for TimescaleDB growth
- Mosquitto already running or ready to be installed separately

For a `717 MiB` VPS:

- the stack may still run, but memory is tight
- keep the MQTT topic narrow
- avoid extra containers
- do not expect much headroom for growth

## Step 2: Update The VPS

```bash
apt update
apt upgrade -y
apt install -y ca-certificates curl gnupg lsb-release ufw
```

If `apt` is not available, stop and adjust for the actual distro.

## Step 3: Install Docker Engine And Compose

Install Docker from Docker's official repository:

```bash
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
```

For Ubuntu:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | tee /etc/apt/sources.list.d/docker.list > /dev/null
```

For Debian, replace `ubuntu` with `debian` in the URL above.

Install packages:

```bash
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Verify:

```bash
docker --version
docker compose version
systemctl enable docker
systemctl start docker
```

## Step 4: Prepare The Deployment Directory

Choose a fixed deployment path:

```bash
mkdir -p /opt/meshtastic
cd /opt/meshtastic
```

Then either:

### Option A: Clone the exporter repo directly

```bash
git clone https://github.com/tcivie/meshtastic-metrics-exporter.git
cd meshtastic-metrics-exporter
```

### Option B: Copy your local tested clone

From your local machine:

```bash
rsync -avz /home/fcz/dev/mesh/meshtastic-metrics-exporter/ root@YOUR_VPS_IP:/opt/meshtastic/meshtastic-metrics-exporter/
```

Then on the VPS:

```bash
cd /opt/meshtastic/meshtastic-metrics-exporter
```

## Step 5: Configure The Exporter `.env`

Edit the file:

```bash
nano .env
```

Recommended VPS-oriented baseline:

```dotenv
DATABASE_URL=postgres://postgres:postgres@timescaledb:5432/meshtastic

MQTT_HOST=host.docker.internal
MQTT_PORT=1883
MQTT_USERNAME=YOUR_MQTT_USERNAME
MQTT_PASSWORD=YOUR_MQTT_PASSWORD
MQTT_KEEPALIVE=60
MQTT_TOPIC='msh/YOUR_REGION/2/e/LongFast/#'
MQTT_IS_TLS=false

MQTT_PROTOCOL=MQTTv311
MQTT_CALLBACK_API_VERSION=VERSION2

MESH_HIDE_SOURCE_DATA=false
MESH_HIDE_DESTINATION_DATA=false
MQTT_SERVER_KEY=1PG7OiApB1nwvP+rz05pAQ==

EXPORTER_MESSAGE_TYPES_TO_FILTER=TEXT_MESSAGE_APP

REPORT_NODE_CONFIGURATIONS=true
ENABLE_STREAM_HANDLER=true
LOG_LEVEL=INFO
LOG_FILE_MAX_SIZE=10MB
LOG_FILE_BACKUP_COUNT=5
```

Important:

- if `MQTT_PASSWORD` contains `$`, write each `$` as `$$`
- keep the topic as narrow as possible
- `host.docker.internal` works here because the compose file adds `host-gateway`

## Step 6: Review Ports And Exposure

The current compose file exposes:

- Grafana on `3000`
- TimescaleDB on `5432`

This deployment should not treat Grafana as a public application.

Recommended exposure model:

- expose SSH
- expose Grafana only if you need direct private access to manage dashboards
- do not expose PostgreSQL publicly
- use Grafana snapshots when you want a no-login shared view

Before starting production use, decide whether you really want those ports exposed.

For a safer first pass:

- allow SSH
- allow Grafana only if you need remote web access
- keep PostgreSQL closed to the public internet unless there is a specific reason

Example firewall setup:

```bash
ufw allow OpenSSH
ufw allow 3000/tcp
ufw enable
ufw status
```

If you do not need direct database access from outside the VPS, remove or restrict port `5432`.

Strong recommendation:

- edit `docker-compose.yml` and remove the `5432:5432` port mapping before long-term deployment

## Step 7: Start The Stack

From the exporter directory:

```bash
docker compose up -d
```

Check status:

```bash
docker compose ps
```

Expected services:

- `timescaledb`
- `exporter`
- `grafana`

## Step 8: Verify Logs

Check the logs:

```bash
docker compose logs --tail=100 timescaledb
docker compose logs --tail=100 grafana
docker compose logs --tail=100 exporter
```

What to look for:

- TimescaleDB initialized and healthy
- Grafana provisioning finished without fatal errors
- exporter connected to MQTT and started processing messages

## Step 9: Open Grafana

Browse to:

```text
http://YOUR_VPS_IP:3000
```

Default first login is normally:

- username: `admin`
- password: `admin`

If Grafana forces a password change, save the new password securely.

Then verify:

- datasource `timescaledb` exists
- dashboards are visible under `Dashboards -> Browse`
- your custom dashboard works as expected

## Step 10: Snapshot Sharing Model

The intended sharing model for this VPS is:

- keep Grafana private
- build and update the dashboard inside Grafana
- publish snapshots when you want a read-only view without login

Why this fits:

- snapshots do not require exposing live Grafana access
- snapshots are good enough for telemetry that updates every 30 minutes
- this avoids building a separate public site just to show one chart

Practical workflow:

1. Open the dashboard in Grafana.
2. Use `Share`.
3. Choose `Snapshot`.
4. Publish the snapshot.
5. Share the snapshot link.

Note:

- snapshots are not live
- create a new snapshot whenever you want an updated public view

## Step 11: Confirm Data Arrival

In Grafana Explore, use queries like:

```sql
select * from node_details order by updated_at desc limit 20;
```

```sql
select time, source_id, destination_id, portnum, packet_id
from mesh_packet_metrics
order by time desc
limit 50;
```

```sql
select *
from environment_metrics
order by time desc
limit 20;
```

This confirms:

- node discovery
- packet ingestion
- telemetry storage

## Step 12: Persist Data Across Restarts

The compose file already uses named Docker volumes:

- `grafana_data`
- `timescaledb_data`

These survive:

```bash
docker compose down
docker compose up -d
```

They do not survive:

```bash
docker compose down -v
```

Do not use `-v` unless you intentionally want to wipe Grafana and TimescaleDB state.

## Step 13: Migrate Your Local Database Later

When you are ready to move from local to VPS:

Create a dump on the local machine:

```bash
cd /home/fcz/dev/mesh/meshtastic-metrics-exporter
docker compose exec -T timescaledb pg_dump -U postgres -d meshtastic -Fc > meshtastic.dump
```

Copy the dump to the VPS:

```bash
scp meshtastic.dump root@YOUR_VPS_IP:/opt/meshtastic/meshtastic-metrics-exporter/
```

Restore it on the VPS:

```bash
cd /opt/meshtastic/meshtastic-metrics-exporter
docker compose exec -T timescaledb dropdb -U postgres --if-exists meshtastic
docker compose exec -T timescaledb createdb -U postgres meshtastic
cat meshtastic.dump | docker compose exec -T timescaledb pg_restore -U postgres -d meshtastic --clean --if-exists
```

Then verify the historical rows are present.

## Step 14: Suggested Cutover Sequence

1. Keep local collection running.
2. Bring the VPS stack up and verify it works.
3. Stop the local exporter only.
4. Take a final local DB dump.
5. Restore the dump to the VPS.
6. Start or restart the VPS exporter.
7. Confirm new data is arriving on the VPS.
8. Retire the local stack after validation.

## Step 15: Next Improvement

The current exporter stores all matching traffic on the configured topic.

Planned improvement for this setup:

- add an allowlist based on `mesh_packet.from`
- keep only the weather node IDs in TimescaleDB

That is the right next change before relying on the VPS instance long term.
