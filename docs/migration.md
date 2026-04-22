# Meshtastic Metrics VPS Migration v2

## Goals

* Clean migration from old VPS → new VPS
* Full TimescaleDB restore (no schema drift)
* Keep MQTT on old VPS (temporary)
* Minimal OS + minimal disk growth
* Docker configured to avoid disk explosion
* Grafana fully restored (dashboards, datasources)
* Future-proof system (Reticulum, etc.)

---

## 0. Variables

* OLD_VPS_IP = your current VPS
* NEW_VPS_IP = your new VPS
* APP_DIR = /opt/meshtastic/meshtastic-metrics-exporter

---

## 1. Base Setup

```bash
apt update
apt upgrade -y

apt install -y \
  curl ca-certificates gnupg \
  ufw caddy rsync nano htop jq
```

---

## 2. Remove Snap Bloat

```bash
apt purge -y snapd || true
rm -rf /var/lib/snapd /snap /var/snap || true
apt autoremove -y
apt clean
```

---

## 3. Install Docker

```bash
install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

echo \
"deb [arch=$(dpkg --print-architecture) \
signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
> /etc/apt/sources.list.d/docker.list

apt update

apt install -y \
  docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

systemctl enable docker
systemctl start docker
```

If the new VPS runs Debian, replace `ubuntu` with `debian` in the Docker repository URL above before running `apt update`.

---

## 4. Docker Disk Protection

```bash
cat >/etc/docker/daemon.json <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "5m",
    "max-file": "2"
  }
}
EOF

systemctl restart docker
```

---

## 5. Limit System Logs

```bash
mkdir -p /etc/systemd/journald.conf.d

cat >/etc/systemd/journald.conf.d/limits.conf <<'EOF'
[Journal]
SystemMaxUse=100M
SystemKeepFree=500M
MaxRetentionSec=7day
EOF

systemctl restart systemd-journald
```

---

## 6. Firewall

```bash
ufw default deny incoming
ufw default allow outgoing

ufw allow OpenSSH
ufw allow 80/tcp
ufw allow 443/tcp

ufw enable
```

Do not rely on UFW alone for Docker-published ports. If `docker-compose.yml` still publishes `3000:3000` or `5432:5432`, those services may remain reachable even if UFW does not obviously allow them.

---

## 7. Copy Application

```bash
mkdir -p /opt/meshtastic

rsync -avz root@OLD_VPS_IP:/opt/meshtastic/meshtastic-metrics-exporter /opt/meshtastic/
cd /opt/meshtastic/meshtastic-metrics-exporter
```

---

## 8. Disable init.sql (CRITICAL)

Temporarily remove any:

```
/docker-entrypoint-initdb.d/
```

from docker-compose.

---

## 9. Dump Database (OLD VPS)

This is the authoritative cutover dump. Do it only after the new VPS is ready for restore.

Expect a brief ingestion pause during the final dump and restore window. If true zero-loss ingestion matters, this procedure is not enough by itself; you would need broker-side retention or a replication-based approach.

```bash
cd /opt/meshtastic/meshtastic-metrics-exporter

docker compose stop exporter

docker exec -i meshtastic-metrics-exporter-timescaledb-1 \
pg_dump -U postgres -d meshtastic \
> /root/meshtastic_dump.sql
```

---

## 10. Transfer Dump

```bash
scp root@OLD_VPS_IP:/root/meshtastic_dump.sql /root/
```

---

## 11. Clean Start

```bash
docker compose down
docker volume rm meshtastic-metrics-exporter_timescaledb_data || true
```

---

## 12. Start DB Only

```bash
docker compose up -d timescaledb
docker logs -f meshtastic-metrics-exporter-timescaledb-1
```

Wait for:

```
database system is ready
```

---

## 13. Enable Restore Mode

```bash
docker exec -i meshtastic-metrics-exporter-timescaledb-1 \
psql -U postgres -d meshtastic \
-c "SELECT timescaledb_pre_restore();"
```

---

## 14. Restore Full DB

```bash
cat /root/meshtastic_dump.sql | \
docker exec -i meshtastic-metrics-exporter-timescaledb-1 \
psql -v ON_ERROR_STOP=1 -U postgres -d meshtastic
```

---

## 15. Disable Restore Mode

```bash
docker exec -i meshtastic-metrics-exporter-timescaledb-1 \
psql -U postgres -d meshtastic \
-c "SELECT timescaledb_post_restore();"
```

---

## 16. Verify Data

```bash
docker exec -it meshtastic-metrics-exporter-timescaledb-1 \
psql -U postgres -d meshtastic \
-c "SELECT MIN(time), MAX(time), COUNT(*) FROM environment_metrics;"
```

---

## 17. Restore Grafana

OLD VPS:

```bash
cd /opt/meshtastic/meshtastic-metrics-exporter
docker compose stop grafana

docker run --rm \
-v meshtastic-metrics-exporter_grafana_data:/volume \
-v /root:/backup \
alpine tar czf /backup/grafana.tar.gz -C /volume .
```

NEW VPS:

```bash
scp root@OLD_VPS_IP:/root/grafana.tar.gz /root/

mkdir /root/grafana_restore
tar xzf /root/grafana.tar.gz -C /root/grafana_restore

cd /opt/meshtastic/meshtastic-metrics-exporter
docker compose stop grafana || true

docker run --rm \
-v meshtastic-metrics-exporter_grafana_data:/volume \
-v /root/grafana_restore:/restore \
alpine sh -c "cp -a /restore/. /volume/"
```

---

## 18. Runtime Config And Port Exposure

Update `.env` for the temporary MQTT dependency on the old VPS:

```
MQTT_HOST=OLD_VPS_IP
```

Then edit `docker-compose.yml` before the first full startup:

- change Grafana from `3000:3000` to `127.0.0.1:3000:3000`
- remove `5432:5432` unless you explicitly need remote PostgreSQL access during migration

Recommended target shape:

```yaml
grafana:
  ports:
    - "127.0.0.1:3000:3000"

timescaledb:
  # no public ports block unless temporarily required
```

This keeps Grafana reachable for local Caddy proxying and keeps TimescaleDB off the public internet.

---

## 19. Start Everything

```bash
cd /opt/meshtastic/meshtastic-metrics-exporter
docker compose up -d
```

---

## 20. Copy Caddy Config

```bash
scp root@OLD_VPS_IP:/etc/caddy/Caddyfile /etc/caddy/Caddyfile

caddy validate --config /etc/caddy/Caddyfile
systemctl restart caddy
```

---

## 21. Test

* Grafana loads
* dashboards present
* data visible
* exporter running
* new telemetry arrives after cutover
* `docker compose ps` does not show `0.0.0.0:3000->3000/tcp`
* `docker compose ps` does not show `0.0.0.0:5432->5432/tcp`

---

## 22. DNS Cutover

Point:

```
clima.falafel.com.br → NEW_VPS_IP
```

---

## 23. Keep Old VPS

Wait 2–3 days before deleting.

---

## 24. Weekly Cleanup

```bash
cat >/etc/cron.weekly/docker-clean <<'EOF'
#!/bin/sh
docker system prune -f
journalctl --vacuum-size=100M
EOF

chmod +x /etc/cron.weekly/docker-clean
```

---

## Final Notes

* Always restore DB before the new exporter starts consuming live messages
* The cutover point is when the old exporter is stopped for the authoritative dump
* Disable `init.sql` during full restore
* Stop Grafana before taking a volume-level backup
* Docker logs must be capped
* Do not trust UFW alone when Docker is publishing ports
* MQTT dependency remains external for now
* System is ready for future expansion
