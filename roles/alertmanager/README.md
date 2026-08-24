# `flyoverhead.docker.alertmanager`

`Alertmanager` docker service deployment.

## Requirements

Message templates are deployed from this role's `files` directory: every
`*.tmpl` file there is copied to `<alertmanager_docker_config.path>/templates/`
and picked up by the `templates:` entry in the rendered `alertmanager.yml`. The
role ships `telegram.tmpl` and `slack.tmpl`; add your own alongside them.

## Role variables

| Variable | Description | Example |
| :--- | :--- | :--- |
| `alertmanager_docker_config` | Docker configuration | Definition example in [defaults/main.yml](defaults/main.yml) |
| `alertmanager_service_config` | Service configuration and the image uid | Definition example in [defaults/main.yml](defaults/main.yml) |
| `alertmanager_command` | Service start options | Definition example in [defaults/main.yml](defaults/main.yml) |
| `alertmanager_global_config` | Global configuration parameters | `Example below` |
| `alertmanager_route_config` | Route configuration parameters | `Example below` |
| `alertmanager_receivers_config` | Receivers configuration parameters | `Example below` |

Silences and the notification log are persisted to
`<alertmanager_docker_config.path>/data`, owned by
`alertmanager_service_config.uid` (65534, the `nobody` user the
`prom/alertmanager` image runs as). Without that volume every image update
re-notifies alerts that were already silenced.

`alertmanager_command` replaces the image `CMD` outright, so `--storage.path`
has to be repeated there.

## Security note

`alertmanager.yml` holds receiver credentials (bot tokens, webhook URLs) and is
rendered world-readable, because the `prom/alertmanager` image runs as `nobody`
and reads it through the bind mount. Keep the values themselves in
`ansible-vault`; they are still cleartext on the host once rendered.

## Facts set by this role

| Fact | Description |
| :--- | :--- |
| `alertmanager_container_exists` | Whether the container is already present |
| `alertmanager_container_update` | Whether the running container's image differs from the configured tag |

## Dependencies

| Name | Description |
| :--- | :--- |
| `flyoverhead.docker.docker` | [README.md](../docker/README.md) |

## Example playbook

```yaml
- hosts: docker
  gather_facts: true
  roles:
    - role: flyoverhead.docker.docker
    - role: flyoverhead.docker.alertmanager
```

## Global configuration example

```yaml
alertmanager_global_config:
  resolve_timeout: 5m
```

## Route configuration example

```yaml
alertmanager_route_config:
  group_wait: 10s
  group_interval: 10m
  repeat_interval: 60m
  group_by:
    - alertname
  receiver: telegram
```

## Receivers configuration example

```yaml
alertmanager_receivers_config:
  - name: telegram
    telegram_configs:
      - send_resolved: true
        api_url: https://api.telegram.org
        bot_token: <telegram_bot_token>
        chat_id: <telegram_chat_id>
        message: '{% raw %}{{ template "telegram.custom.message" . }}{% endraw %}'
        parse_mode: Markdown
```

`chat_id` is coerced to an integer by the template: Alertmanager wants an int64
there, and a vault-sourced value would otherwise be rendered as a quoted string
and rejected.

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
| `alertmanager.detect` | Container state detection only |
| `alertmanager.install` | First deployment |
| `alertmanager.update` | Image tag change: pull and recreate |
| `alertmanager.config` | `alertmanager.yml`, message templates and the data volume |

## License

GPL-3.0-only

## Author Information

fLy0v3rH34d
