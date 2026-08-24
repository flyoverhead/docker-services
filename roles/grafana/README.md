# `flyoverhead.docker.grafana`

`Grafana` docker service deployment.

## Role variables

| Variable | Description | Example |
| :--- | :--- | :--- |
| `grafana_docker_config` | Docker configuration | Definition example in [defaults/main.yml](defaults/main.yml) |
| `grafana_service_config` | Service configuration, admin credentials and the image uid | Definition example in [defaults/main.yml](defaults/main.yml) |
| `grafana_plugins` | Plugins to be installed | Definition example in [defaults/main.yml](defaults/main.yml) |
| `grafana_dashboards` | Dashboards to be downloaded from grafana.com | `Example below` |
| `grafana_source` | Provisioned metrics datasource | Definition example in [defaults/main.yml](defaults/main.yml) |

The role tracks `grafana/grafana`, the OSS distribution. The `grafana/grafana-oss`
mirror this role used previously stopped receiving tags after `13.0.2`.

`grafana.db` is persisted to `<grafana_docker_config.path>/data`, owned by
`grafana_service_config.uid` (472, the `grafana` user the image runs as).
Without that volume every image update loses users, API keys, alert rules and
any dashboard created through the UI.

Provisioned dashboards are mounted read-only at `/etc/grafana/dashboards` rather
than under `/var/lib/grafana`, so the read-only dashboard mount does not nest
inside the writable data volume.

`grafana_service_config.admin_password` defaults to a placeholder. Override it,
and keep the value in `ansible-vault`.

## Facts set by this role

| Fact | Description |
| :--- | :--- |
| `grafana_container_exists` | Whether the container is already present |
| `grafana_container_update` | Whether the running container's image differs from the configured tag |

## Dependencies

| Name | Description |
| :--- | :--- |
| `flyoverhead.docker.docker` | [README.md](../docker/README.md) |

Optional, and only for the default `grafana_source`, which reads the port from
this role:

| Name | Description |
| :--- | :--- |
| `flyoverhead.docker.prometheus` | [README.md](../prometheus/README.md) |

## Example playbook

```yaml
- hosts: docker
  gather_facts: true
  roles:
    - role: flyoverhead.docker.docker
    - role: flyoverhead.docker.prometheus
    - role: flyoverhead.docker.grafana
      vars:
        grafana_service_config:
          listening_port: 3000
          admin_username: grafana
          admin_password: '{{ vault_grafana_admin_password }}'
          uid: 472
```

## Dashboards list example

```yaml
grafana_dashboards:
  - dashboard_id: 1860 # Node Exporter Full
    revision_id: 45
    name: node_exporter
```

`dashboard_id` and `revision_id` come from the dashboard's page on
[grafana.com/grafana/dashboards](https://grafana.com/grafana/dashboards/). Both
are pinned on purpose: a moving revision would change the deployed JSON without
any change in this repository.

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
| `grafana.detect` | Container state detection only |
| `grafana.install` | First deployment |
| `grafana.update` | Image tag change: pull and recreate |
| `grafana.config` | Provisioning, dashboards and the data volume |

## License

GPL-3.0-only

## Author Information

fLy0v3rH34d
