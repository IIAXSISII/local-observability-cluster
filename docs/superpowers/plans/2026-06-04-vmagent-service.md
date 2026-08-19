# vmagent Service Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `vmagent` service to the local observability cluster that scrapes its own metrics and VictoriaMetrics' metrics, then remote-writes to the single-node VictoriaMetrics instance.

**Architecture:** A new `vmagent-config.yml` file provides a Prometheus-compatible scrape config with two jobs (`vmagent` self-metrics and `victoriametrics`). The `vmagent` container mounts this config and is started with `-remoteWrite.url` pointing at the existing `victoriametrics` service. A named volume backs vmagent's WAL so in-flight data survives restarts.

**Tech Stack:** Docker Compose, VictoriaMetrics vmagent (`victoriametrics/vmagent:latest`), Prometheus scrape config format.

---

## File Map

| Action | Path | Responsibility |
|--------|------|----------------|
| Create | `vmagent-config.yml` | Prometheus-style scrape config: two jobs, 15s interval |
| Modify | `docker-compose.yml` | Add `vmagent` service + `vmagent-data` named volume |

---

### Task 1: Create `vmagent-config.yml`

**Files:**
- Create: `vmagent-config.yml`

- [ ] **Step 1: Create the scrape config file**

Create `/Users/sgautam/git/iiaxsisii/observability-cluster/vmagent-config.yml` with this exact content:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: vmagent
    static_configs:
      - targets: ['vmagent:8429']

  - job_name: victoriametrics
    static_configs:
      - targets: ['victoriametrics:8428']
```

- [ ] **Step 2: Verify file is valid YAML**

Run:
```bash
docker run --rm -v $(pwd)/vmagent-config.yml:/cfg.yml mikefarah/yq e '.' /cfg.yml
```

Expected: the file content is printed without errors. If `yq` is not available locally, skip this step — vmagent itself will error on startup if the YAML is invalid (caught in Task 2).

---

### Task 2: Add `vmagent` service to `docker-compose.yml`

**Files:**
- Modify: `docker-compose.yml`

- [ ] **Step 1: Add the `vmagent` service block**

In `docker-compose.yml`, add the following service entry under the `# ── Metrics` comment block, after the `victoriametrics` service and before `grafana`:

```yaml
  vmagent:
    image: victoriametrics/vmagent:latest
    platform: linux/arm64
    ports:
      - "8429:8429"
    volumes:
      - ./vmagent-config.yml:/etc/vmagent/config.yml
      - vmagent-data:/vmagentdata
    command:
      - "-promscrape.config=/etc/vmagent/config.yml"
      - "-remoteWrite.url=http://victoriametrics:8428/api/v1/write"
    depends_on:
      - victoriametrics
```

- [ ] **Step 2: Add `vmagent-data` to the top-level `volumes` block**

In the `volumes:` section at the bottom of `docker-compose.yml`, add:

```yaml
  vmagent-data:
```

The final `volumes:` block should look like:

```yaml
volumes:
  langfuse-db-data:
  clickhouse-data:
  clickhouse-logs:
  redis-data:
  minio-data:
  victoriametrics-data:
  victoriatraces-data:
  grafana-data:
  vmagent-data:
```

- [ ] **Step 3: Validate the compose file**

Run:
```bash
docker compose config --quiet
```

Expected: exits with code 0 and no output (valid). If there's a parse error, the output will describe the problem.

---

### Task 3: Start vmagent and verify it is scraping

- [ ] **Step 1: Start only the vmagent service (and its dependency)**

Run:
```bash
docker compose up -d victoriametrics vmagent
```

Expected output (abbreviated):
```
 ✔ Container observability-cluster-victoriametrics-1  Running
 ✔ Container observability-cluster-vmagent-1          Started
```

- [ ] **Step 2: Check vmagent is healthy**

Run:
```bash
docker compose logs vmagent --tail=20
```

Expected: logs contain lines like:
```
started scraping target ...
```
and no `ERROR` lines about config or remote write connectivity.

- [ ] **Step 3: Verify vmagent's own metrics endpoint**

Run:
```bash
curl -s http://localhost:8429/metrics | grep '^vm_app_version'
```

Expected: one line like:
```
vm_app_version{version="...",short_version="..."} 1
```

- [ ] **Step 4: Verify metrics arrived in VictoriaMetrics**

Run:
```bash
curl -s 'http://localhost:9091/api/v1/query?query=vm_rows_added_to_storage_total{job="vmagent"}' | python3 -m json.tool
```

Expected: a JSON response with `"status":"success"` and at least one result entry. This confirms vmagent scraped itself and wrote to VictoriaMetrics.

Run the same check for the `victoriametrics` job:
```bash
curl -s 'http://localhost:9091/api/v1/query?query=vm_rows_added_to_storage_total{job="victoriametrics"}' | python3 -m json.tool
```

Expected: same — `"status":"success"` with results.

- [ ] **Step 5: Confirm scrape targets are healthy in vmagent UI**

Open `http://localhost:8429/targets` in a browser.

Expected: two targets listed (`vmagent` and `victoriametrics`), both with state `up`.
