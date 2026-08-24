# `flyoverhead.docker.prometheus`

`Prometheus` docker service deployment.

## Requirements

Alerting rules are deployed from this role's `files` directory: every `*.yml`
file there is copied to `<prometheus_docker_config.path>/alerts/` and picked up
by the `rule_files: [alerts/*.yml]` entry in the rendered `prometheus.yml`. The
role ships `node-exporter.yml` and `embedded-exporter.yml`; add your own
alongside them.

## Role variables

| Variable | Description | Example |
| :--- | :--- | :--- |
| `prometheus_docker_config` | Docker configuration | Definition example in [defaults/main.yml](defaults/main.yml) |
| `prometheus_service_config` | Service configuration, retention and the image uid | Definition example in [defaults/main.yml](defaults/main.yml) |
| `prometheus_command` | Service start options | Definition example in [defaults/main.yml](defaults/main.yml) |
| `prometheus_global_config` | Global configuration parameters | `Example below` |
| `prometheus_scrape_jobs` | Scrape jobs configuration parameters | `Example below` |
| `prometheus_alert_jobs` | Alerting jobs configuration parameters | `Example below` |

The TSDB is persisted to `<prometheus_docker_config.path>/data`, owned by
`prometheus_service_config.uid` (65534, the `nobody` user the `prom/prometheus`
image runs as). Without that volume the whole metrics history is lost on every
image update.

`prometheus_command` replaces the image `CMD` outright, so `--storage.tsdb.path`
has to be repeated there. Drop it and Prometheus falls back to a relative
`data/` inside its working directory, outside the mounted volume.

## Facts set by this role

| Fact | Description |
| :--- | :--- |
| `prometheus_container_exists` | Whether the container is already present |
| `prometheus_container_update` | Whether the running container's image differs from the configured tag |

## Dependencies

| Name | Description |
| :--- | :--- |
| `flyoverhead.docker.docker` | [README.md](../docker/README.md) |

Optional, and only for the default `prometheus_scrape_jobs` /
`prometheus_alert_jobs`, which reference the ports these roles define:

| Name | Description |
| :--- | :--- |
| `flyoverhead.docker.node_exporter` | [README.md](../node_exporter/README.md) |
| `flyoverhead.docker.alertmanager` | [README.md](../alertmanager/README.md) |

## Example playbook

```yaml
- hosts: docker
  gather_facts: true
  roles:
    - role: flyoverhead.docker.docker
    - role: flyoverhead.docker.node_exporter
    - role: flyoverhead.docker.alertmanager
    - role: flyoverhead.docker.prometheus
```

## Global configuration example

```yaml
prometheus_global_config:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
```

## Scrape configuration example

```yaml
prometheus_scrape_jobs:
  - job_name: node-exporter
    scrape_interval: 1m
    static_configs:
      - targets:
          - 192.168.1.1:9100
        labels:
          instance: node-exporter
  - job_name: prometheus
    scrape_interval: 1m
    static_configs:
      - targets:
          - 192.168.1.1:9090
        labels:
          instance: prometheus
```

## Alerting configuration example

```yaml
prometheus_alert_jobs:
  - static_configs:
      - targets:
          - 192.168.1.1:9093
```

## Check mode

`--check --diff` reports drift in the rendered configuration and the compose
file against a host where the stack is already deployed. The restart and
recreate handlers report what they would do without touching the container.

Detection reads the running container through
`community.docker.docker_container_info`, so a check run needs a reachable
docker daemon: against a host without the engine it fails in `detect.yml`. Run
the `docker` role for real once first.

## Tags

| Tag | Purpose |
| :--- | :--- |
| `prometheus.detect` | Container state detection only |
| `prometheus.install` | First deployment |
| `prometheus.update` | Image tag change: pull and recreate |
| `prometheus.config` | `prometheus.yml`, alerting rules and the data volume |

## License

GPL-3.0-only

## Author Information

fLy0v3rH34d
