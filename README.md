# Container Resource Monitor

A lightweight, self-hosted monitoring stack that gives you real-time visibility into the CPU and memory usage of every container running on your Docker host — built with cAdvisor, Prometheus, and Grafana.

I put this together mainly to get hands-on with the observability side of DevOps rather than just the deployment side. It's easy to `docker run` something and call it done; it's a different skill entirely to actually *know* what your containers are doing once they're up.

![Dashboard Screenshot](./dashboard-stress-test.png)

## Why this exists

Most "Docker + Grafana" tutorials stop at importing a community dashboard and calling it a project. I wanted something I could actually explain line by line in an interview — where I understood every query, built every panel manually, and hit (and fixed) a real production-style bug along the way. That last part turned out to be the most valuable part of building this.

## Architecture

```
┌───────────┐      scrapes      ┌────────────┐      queries      ┌─────────┐
│ cAdvisor  │ ───────────────►  │ Prometheus │ ───────────────►  │ Grafana │
└───────────┘                   └────────────┘                   └─────────┘
      │
      ▼
Docker Engine
(reads container
 stats via /var/run/docker.sock
 and cgroup filesystem)
```

- **cAdvisor** runs as a container itself, reads live resource stats for every other container on the host directly from the kernel's cgroup interface, and exposes them as Prometheus-formatted metrics on port 8080.
- **Prometheus** scrapes cAdvisor every 10 seconds and stores the time-series data.
- **Grafana** queries Prometheus and renders it as dashboards.

All three services run via a single `docker-compose.yml`, so the whole stack comes up with one command.

## What it monitors

- **CPU usage per container** — computed with `rate()` over the raw CPU counter, so it reflects actual usage over time rather than a cumulative total
- **Memory usage per container** — live RSS memory in human-readable units (MiB/GiB)

Both panels are labeled by container name, not container ID, so the dashboard is actually readable at a glance.

## Getting it running

You'll need Docker and Docker Compose installed. This was built and tested on WSL2 Ubuntu, but it'll run anywhere Docker does.

```bash
git clone https://github.com/<your-username>/container-monitor.git
cd container-monitor
docker compose up -d
```

Give it about 15–20 seconds to start scraping, then:

| Service    | URL                     | Notes                          |
|------------|-------------------------|---------------------------------|
| Grafana    | http://localhost:3000   | login: `admin` / `admin`       |
| Prometheus | http://localhost:9090   | check `/targets` to confirm scraping |
| cAdvisor   | http://localhost:8080   | raw per-container metrics UI   |

### Connecting Grafana to Prometheus

1. In Grafana, go to **Connections → Data sources → Add data source → Prometheus**
2. Set the URL to `http://prometheus:9090` (container-to-container networking, not `localhost`)
3. Save & test — you should get a green success message

### Building the dashboard

If you want to recreate the panels yourself rather than import a JSON file:

**CPU panel**
```
rate(container_cpu_usage_seconds_total{name!=""}[1m])
```

**Memory panel**
```
container_memory_usage_bytes{name!=""}
```

Set the legend format to `{{name}}` on both so you get clean container names instead of raw label strings.

## A bug I ran into (and actually had to debug properly)

This is the part of the project I'm most glad happened, honestly.

When I first brought the stack up, Prometheus was scraping cAdvisor successfully and cAdvisor itself looked healthy — but every single metric only ever showed data for host-level systemd cgroups (`system.slice`, `init.scope`, `docker.service`). Nothing for my actual containers. No `name` label, no `image` label, nothing.

Digging into the cAdvisor container logs turned up the real error:

```
Failed to create existing container: ... failed to identify the read-write
layer ID for container "...". open /rootfs/var/lib/docker/image/overlayfs/
layerdb/mounts/<id>/mount-id: no such file or directory
```

Turned out my Docker installation had the **containerd image store** (snapshotter) feature enabled by default — a newer storage backend that recent Docker versions ship with. cAdvisor v0.49.x's Docker integration was written against the older classic `overlayfs2` layer store, so it couldn't map any container ID back to a real container. It could see that containers existed at the cgroup level, but couldn't resolve what they actually were.

The fix was two-part:

1. Disable the containerd snapshotter in the Docker daemon config (`/etc/docker/daemon.json`):
   ```json
   {
     "features": {
       "containerd-snapshotter": false
     }
   }
   ```
2. Add the `-docker_only=true` flag to cAdvisor's startup command, and mount `/sys/fs/cgroup` explicitly so it could walk the cgroup v2 hierarchy correctly.

After a daemon restart and rebuilding the stack, cAdvisor immediately started reporting proper `name` and `image` labels for every container.

I'm including this because I think it's a more honest signal of understanding than a clean dashboard screenshot alone — anyone can import a Grafana JSON. Fewer people have actually had to read cAdvisor source-adjacent error messages and figure out why a metrics pipeline is silently returning the wrong data.

## Proving it works: a live stress test

To confirm the pipeline was actually reactive and not just displaying static noise, I ran a controlled CPU load test against the host and watched it show up in near real-time on the dashboard:

```bash
docker run --rm -it polinux/stress stress --cpu 2 --timeout 60s
```

The spike visible in the screenshot above is that exact 60-second run — visible in Grafana roughly 10–15 seconds after it started, which lines up with the Prometheus scrape interval.

## What I'd add next

- **Log aggregation** with Loki, correlated against the same time axis as the metrics
- **Alertmanager rules** for things like sustained high CPU or OOM-killed containers, routed to Slack
- **A pre-built dashboard JSON** in this repo so others can import it in one click instead of rebuilding panels by hand

## Stack

- Docker Compose
- [cAdvisor](https://github.com/google/cadvisor) `v0.49.1`
- [Prometheus](https://prometheus.io/) `v2.53.0`
- [Grafana](https://grafana.com/) `11.1.0`

## License

MIT — use it, fork it, break it, learn from it.
