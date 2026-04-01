# Weather Telemetry Plan

## Goal

Collect telemetry from Meshtastic weather nodes, store the values over time, and view them later in charts on a dashboard.

## Current Direction

- Keep using the public `LongFast` channel on the nodes.
- Keep using the existing Mosquitto broker on the VPS.
- Run `tcivie/meshtastic-metrics-exporter` with TimescaleDB and Grafana on the VPS.
- Filter packets by originating Meshtastic node ID before storing them.
- Store data only for the weather nodes that belong to this setup.

## Important Detail

- Do not filter by MQTT topic suffix alone.
- In Meshtastic MQTT, the topic suffix identifies the MQTT gateway node.
- The real sender to filter on is the packet source node ID from `mesh_packet.from`.

## Planned Stack

- Mosquitto
- Meshtastic metrics exporter
- TimescaleDB
- Grafana

## Steps

1. Confirm both weather nodes are publishing telemetry on `LongFast`.
2. Record the Meshtastic node IDs for the weather nodes.
3. Deploy the exporter stack on the VPS with Docker Compose.
4. Point the exporter to the existing Mosquitto broker.
5. Subscribe only to the relevant Meshtastic channel topic for the region.
6. Add filtering so packets are stored only when `mesh_packet.from` matches one of the allowed node IDs.
7. Verify that telemetry packets from the two weather nodes are reaching TimescaleDB.
8. Open Grafana and confirm that node-specific charts show data for temperature and other environmental values.
9. Increase retention settings if the default retention window is too short.
10. Back up the database volume if historical weather data becomes important.

## Suggested Configuration Direction

- MQTT topic: subscribe to the `LongFast` topic for the correct region, not to a broader wildcard than needed.
- Allowed nodes: keep an allowlist of the weather node IDs.
- Retention: extend beyond the default if the goal is seasonal or long-term comparison.
- Dashboards: use node-level filtering so each weather node can be viewed separately or compared together.

## First Validation Checklist

- Both nodes appear in the exporter database.
- Only the expected node IDs are stored as telemetry sources.
- Temperature, humidity, pressure, and any other enabled sensor values are present.
- New points continue to arrive at the expected telemetry interval.
- Grafana can plot one node and both nodes together.

## Later Improvements

- Move telemetry to a private channel if public `LongFast` noise becomes a problem.
- Add alerts for stale telemetry or sensor values outside expected ranges.
- Add a dedicated dashboard for weather history and node health.
