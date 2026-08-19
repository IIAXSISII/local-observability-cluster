# vmagent Service Design

**Date:** 2026-06-04
**Status:** Approved

## Summary

Add a `vmagent` service to the local observability cluster's `docker-compose.yml`. vmagent is VictoriaMetrics' lightweight metrics scraping agent. It will scrape its own metrics and VictoriaMetrics' metrics, then remote-write the collected data to the single-node VictoriaMetrics instance.

## Components

### New file: `vmagent-config.yml`

A Prometheus-compatible scrape config with two jobs:

- **`vmagent`** — scrapes `vmagent:8429/metrics` (vmagent self-metrics)
- **`victoriametrics`** — scrapes `victoriametrics:8428/metrics`

Scrape interval: `15s` (consistent with the Grafana `timeInterval` datasource setting).

### Modified file: `docker-compose.yml`

New `vmagent` service:

- **Image:** `victoriametrics/vmagent:latest`
- **Platform:** `linux/arm64` (consistent with all other services)
- **Port:** `8429` exposed locally (vmagent UI + self-metrics endpoint)
- **Volume:** `./vmagent-config.yml` mounted into the container
- **Command flags:**
  - `-promscrape.config=/etc/vmagent/config.yml`
  - `-remoteWrite.url=http://victoriametrics:8428/api/v1/write`
- **depends_on:** `victoriametrics`

## Data Flow

```
vmagent:8429/metrics  ──┐
                        ├─► vmagent scrapes (every 15s) ──► remote write ──► victoriametrics:8428
victoriametrics:8428/metrics ──┘
```

## Out of Scope

- Scraping additional targets (langfuse, clickhouse, redis, etc.) — can be added to `vmagent-config.yml` later
- Cluster-mode VictoriaMetrics — current setup is intentionally single-node
- TLS or auth on scrape targets — all internal Docker network, no auth needed
