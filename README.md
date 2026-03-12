# Nokia Monitoring Stack — gnmic + Prometheus

Collects gNMI streaming telemetry from Nokia SR OS devices and exposes metrics to your existing Grafana instance.

## Prerequisites

- Nokia SR OS with gNMI enabled (port 57400)
- Docker + Docker Compose
- Grafana already running in Docker

## Quick Start

### 1. Configure your Nokia devices

Edit `gnmic/gnmic.yaml`:
- Set `username` / `password` (global or per-device)
- Replace the placeholder target(s) with your actual Nokia router IPs/hostnames
- Each target needs port 57400 appended: `router.yourdomain.com:57400`

### 2. Enable gNMI on Nokia SR OS

On each router (requires admin access):
```
configure system grpc
    allow-unsupported-transceiver
    no shutdown
exit
configure system security
    profile "grpc-user"
        netconf base-op-authorization rpc
    exit
exit
```
Verify: `show system grpc`

### 3. Start the stack

```bash
cd ~/clawd/nokia-monitoring
docker compose up -d
docker compose logs -f   # watch for connection errors
```

### 4. Connect Grafana to Prometheus

**Option A — Host IP (simplest, works always):**
In Grafana → Connections → Data Sources → Add Prometheus:
- URL: `http://<your-host-ip>:9090`

**Option B — Container network (cleaner, no host IP):**
Find your Grafana container's network:
```bash
docker inspect <grafana-container-name> | grep -A5 Networks
```
Add that network name to `docker-compose.yml` (see comments at bottom of file),
then use `http://prometheus-nokia:9090` as the Grafana datasource URL.

### 5. Verify metrics are flowing

```bash
# Check gnmic is collecting
curl http://localhost:9804/metrics | grep nokia

# Check Prometheus has targets up
open http://localhost:9090/targets
```

## File Structure

```
nokia-monitoring/
├── docker-compose.yml
├── gnmic/
│   └── gnmic.yaml          # Targets, subscriptions, output config
├── prometheus/
│   └── prometheus.yml      # Scrape config
└── README.md
```

## Useful Metric Names in Grafana

After data flows, metrics will appear with the `nokia_` prefix:

| What | Metric (approximate) |
|------|----------------------|
| Port oper state | `nokia_state_port_oper_state` |
| Port in octets | `nokia_state_port_statistics_in_octets` |
| Port out octets | `nokia_state_port_statistics_out_octets` |
| Interface oper state | `nokia_state_router_interface_oper_state` |
| CPU usage | `nokia_state_system_cpu_summary_usage` |

Use Prometheus' Explore tab in Grafana to browse exact metric names — they vary slightly based on encoding.

## Troubleshooting

**gnmic can't connect to router:**
- Confirm `show system grpc` shows gNMI running
- Check firewall — port 57400 TCP must be open from the Docker host to the router
- Try: `docker exec gnmic gnmic -a <router-ip>:57400 -u admin -p pass --skip-verify capabilities`

**No metrics in Prometheus:**
- Check `http://localhost:9090/targets` — gnmic target should be UP
- Check gnmic logs: `docker compose logs gnmic`

**Grafana shows "no data":**
- Confirm Prometheus datasource URL is correct
- Check time range — data only appears after gnmic starts collecting
- Browse raw metrics at `http://localhost:9804/metrics`
