# `flyoverhead.docker`

[![Version](https://img.shields.io/badge/version-2.0.1-blue)](galaxy.yml)
[![ansible-core](https://img.shields.io/badge/ansible--core-%E2%89%A52.16-black?logo=ansible&logoColor=white)](https://docs.ansible.com/ansible-core/devel/index.html)
[![License](https://img.shields.io/badge/license-GPL--3.0--only-green)](https://www.gnu.org/licenses/gpl-3.0)
[![Platform](https://img.shields.io/badge/platform-Debian%2012%20%7C%2013-A81D33?logo=debian&logoColor=white)](#-supported-os)
[![Roles](https://img.shields.io/badge/roles-10-orange)](#-roles)

Docker services deployment and configuration.

## 🚀 Quick Start

### Requirements

- `ansible-core >=2.16`

- Collections: `ansible.posix`, `community.docker >=3.4.3`, `community.general`

- `jmespath` on the controller (the VPN roles use the `json_query` filter)

- Task `>=3.20`, Vagrant and a VMware Fusion provider, for the test harness only

The `docker` role installs `python3-requests` on every managed host, which the
`community.docker` modules need there.

### Installation

Installing the collection dependencies:

```bash
ansible-galaxy collection install -r requirements.yml
pip install -r requirements.txt
```

Installing the collection itself:

```bash
ansible-galaxy collection install git+https://github.com/flyoverhead/docker.git
```

### Roles usage

Full documentation and usage examples of role `<role>` can be found in
`roles/<role>/README.md`.

Run `flyoverhead.docker.docker` first, on every host. Every other role reads
`docker_path`, `docker_network_mode`, `docker_restart_policy` and
`docker_timezone` from its defaults, and needs the engine and
`python3-requests` in place.

### Example Playbook

```yaml
---
- hosts: all
  become: true
  gather_facts: true
  roles:
    - flyoverhead.docker.docker
    - flyoverhead.docker.node_exporter
    - flyoverhead.docker.prometheus
    - flyoverhead.docker.alertmanager
    - flyoverhead.docker.grafana
    - flyoverhead.docker.dnscrypt
    - flyoverhead.docker.pihole
    - flyoverhead.docker.torrserver
    - flyoverhead.docker.xray
```

## 🖥 Supported OS

| OS | Status |
| :--- | :--- |
| Debian 12 "Bookworm" (AArch64, x86_64) | Tested |
| Debian 13 "Trixie" | Supported, untested |

Debian 11 "Bullseye" is no longer listed: nothing in the collection is
bullseye-specific, but it is out of standard support and is not tested.

## 📦 Roles

<details>
<summary><b>All 10 roles</b> — image and pinned tag for each</summary>

| Name | Description | Image | Version |
| :--- | :--- | :--- | :--- |
| [alertmanager](roles/alertmanager/README.md) | `Alertmanager` alert routing and notification | `prom/alertmanager` | `v0.34.0` |
| [dnscrypt](roles/dnscrypt/README.md) | `DNSCrypt Proxy` encrypted upstream resolver | `klutchell/dnscrypt-proxy` | `2.1.18` |
| [docker](roles/docker/README.md) | `Docker` engine, shared paths and daemon settings | n/a (apt) | repository latest |
| [grafana](roles/grafana/README.md) | `Grafana` dashboards and provisioning | `grafana/grafana` | `13.1.4` |
| [node_exporter](roles/node_exporter/README.md) | `Node exporter` host metrics | `prom/node-exporter` | `v1.12.1` |
| [pihole](roles/pihole/README.md) | `Pi-hole` DNS ad blocking | `pihole/pihole` | `2026.07.2` |
| [prometheus](roles/prometheus/README.md) | `Prometheus` metrics collection and alerting rules | `prom/prometheus` | `v3.14.0` |
| [singbox](roles/singbox/README.md) | `Sing-box` VLESS + REALITY VPN | `ghcr.io/sagernet/sing-box` | `v1.13.19` |
| [torrserver](roles/torrserver/README.md) | `TorrServer` torrent streaming | `ghcr.io/yourok/torrserver` | `MatriX.143` |
| [xray](roles/xray/README.md) | `Xray` VLESS + REALITY VPN | `teddysun/xray` | `26.7.28` |

</details>

Every image tag is pinned in the role's `defaults/main.yml`. Nothing tracks a
moving tag: the update detection in each role compares the running container's
image against the pinned tag, which a moving tag would defeat.

## 🏗 Role Structure

<details>
<summary><b>The five files every service role has</b> — and why an image change recreates rather than restarts</summary>

Every service role follows the same shape:

| File | Purpose |
| :--- | :--- |
| `tasks/detect.yml` | Sets `<role>_container_exists` and `<role>_container_update` |
| `tasks/install.yml` | Runs when the container does not exist |
| `tasks/update.yml` | Runs when the pinned tag differs from the running image |
| `tasks/config.yml` | Always runs; renders configuration and data volumes |
| `handlers/main.yml` | `start` / `restart` / `recreate container` |

`restart container` maps to `docker compose restart`, which reloads
bind-mounted configuration. An image change goes through `recreate container`
(`docker compose up` with `pull: always`, `recreate: always`), because
`restart` would keep running the old image.

</details>

## 🏷 Tags

Every role exposes `<role>.detect`, `<role>.install`, `<role>.update` and,
where it has one, `<role>.config`. The VPN roles use a different split —
`<role>.docker`, `<role>.server`, `<role>.clients`, `<role>.tuning` — described
in their own READMEs.

```bash
# render every configuration file without touching images
ansible-playbook -i inventory.yml playbook.yml --tags prometheus.config,grafana.config

# pull and recreate anything whose pinned tag moved
ansible-playbook -i inventory.yml playbook.yml --tags pihole.update
```

## 🧪 Testing

The `tests` directory holds a Vagrant-based integration harness: one Debian 12
box running the whole stack, with the image tags pinned in
`tests/group_vars/all.yml`.

Launch it from the root directory of the collection:

```bash
task deploy      # vagrant up + ansible-playbook
task provision   # re-run the playbook against a running box
task destroy     # tear the box down
```

After a successful deployment the services are reachable from the host machine
on the forwarded ports declared in the `Vagrantfile`:

| Service | Port |
| :--- | :--- |
| grafana | 3000 |
| dnscrypt-proxy | 5353 |
| torrserver | 8070 |
| pihole web | 8080 |
| sing-box | 8443 |
| prometheus | 9090 |
| alertmanager | 9093 |
| node-exporter | 9100 |

Linting matches CI, via the pre-commit hooks in `.pre-commit-config.yaml`:

```bash
ansible-lint
yamllint .
```

## ⚠️ Gotchas

- **`singbox` and `xray` both default to listening on `:443`**, so pick one per
  host or move one of them to another port.

## 📄 License

- GPL-3.0-only

## 👤 Author Information

fLy0v3rH34d
