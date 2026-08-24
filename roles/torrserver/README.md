# `flyoverhead.docker.torrserver`

`TorrServer` docker service deployment.

## Image

This role uses `ghcr.io/yourok/torrserver`, published from the upstream
TorrServer repository. It replaces the third-party `ksey/torrserver:latest` the
role previously used, which was last built in 2022 and only ever published a
moving `latest` tag -- so `detect.yml` could never see an update.

The upstream image reads its paths from `TS_CONF_PATH` (`/opt/ts/config`) and
`TS_TORR_DIR` (`/opt/ts/torrents`), where the previous image used `/TS/db`.

`torrserver_service_config.data_dir` defaults to `db`, which is the same host
directory the previous image used, so an existing `torrserver.db` survives the
migration untouched.

## Role variables

| Variable | Description | Example |
| :--- | :--- | :--- |
| `torrserver_docker_config` | Docker configuration | Definition example in [defaults/main.yml](defaults/main.yml) |
| `torrserver_service_config` | Service port and host data directories | Definition example in [defaults/main.yml](defaults/main.yml) |

## Facts set by this role

| Fact | Description |
| :--- | :--- |
| `torrserver_container_exists` | Whether the container is already present |
| `torrserver_container_update` | Whether the running container's image differs from the configured tag |

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
    - role: flyoverhead.docker.torrserver
```

## Service configuration example

```yaml
torrserver_service_config:
  listening_port: 8070
  data_dir: db
  torrents_dir: torrents
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
| `torrserver.detect` | Container state detection only |
| `torrserver.install` | First deployment |
| `torrserver.update` | Image tag change: pull and recreate |

## License

GPL-3.0-only

## Author Information

fLy0v3rH34d
