# `flyoverhead.docker.docker`

`Docker` engine installation and configuration.

This role installs Docker CE from the official Docker repository and provides
the shared variables (`docker_path`, `docker_network_mode`,
`docker_restart_policy`, `docker_timezone`) that every service role in this
collection consumes. Run it before any other role of the collection.

## Requirements

- Debian 12 "Bookworm" (tested) or Debian 13 "Trixie"

- `ansible.posix` and `community.docker` collections

## Role variables

| Variable | Description | Example |
| :--- | :--- | :--- |
| `docker_daemon_config` | Contents of `/etc/docker/daemon.json` | Definition example in [defaults/main.yml](defaults/main.yml) |
| `docker_network_mode` | Docker containers network mode | `host` |
| `docker_path` | Root path for services files on host machine | `/opt/docker` |
| `docker_repo` | Docker repository, packages and requirements | Definition example in [defaults/main.yml](defaults/main.yml) |
| `docker_restart_policy` | Docker containers restart policy | `always` |
| `docker_timezone` | Default timezone for docker services | `Europe/Moscow` |
| `docker_users` | List of users to be added to the `docker` group | `['{{ ansible_user }}']` |

`docker_repo.requirements` includes `python3-requests`, which the
`community.docker` modules used by the service roles of this collection need on
the managed host. Keep it when overriding the list.

## Facts set by this role

| Fact | Description |
| :--- | :--- |
| `docker_installed` | Whether the `docker-ce` package is present |
| `docker_installed_version` | Installed `docker-ce` package version (only when installed) |

## Dependencies

None

## Example playbook

```yaml
- hosts: docker
  gather_facts: true
  roles:
    - role: flyoverhead.docker.docker
```

## Daemon configuration example

```yaml
docker_daemon_config:
  log-driver: json-file
  log-opts:
    max-file: '3'
    max-size: 10m
  default-address-pools:
    - base: 172.30.0.0/16
      size: 24
```

## Check mode

`--check --diff` reports drift in `daemon.json` and in the docker group
membership against a host that already has the engine installed.

It does not work against a host that does not: `install.yml` only simulates the
apt install, so the `docker` group is never created, and `config | add users to
docker group` then fails on a group that does not exist. Run the role for real
once before using check mode on a new host.

## Tags

| Tag | Purpose |
| :--- | :--- |
| `docker.detect` | Detect the installed engine only |
| `docker.install` | Repository setup and package installation |
| `docker.config` | `daemon.json` and `docker` group membership |

## License

GPL-3.0-only

## Author Information

fLy0v3rH34d
