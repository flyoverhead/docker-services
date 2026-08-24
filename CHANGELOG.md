# Changelog

All notable changes to `flyoverhead.docker`.

## 2.0.1

### Fixed

- **Check mode**: `ansible-playbook --check --diff` now completes against a host
  this collection has already converged. Every role performed state discovery
  with modules ansible-core does not execute in a check run and then
  dereferenced the registered result unconditionally, so the play aborted on an
  undefined attribute. Nothing in the collection had ever set `check_mode`.
- **xray**: a check run failed in `detect.yml` even against a fully converged
  host. `detect | derive public key from private key` runs a one-shot container,
  and `community.docker.docker_container` does not run it in a check run, so
  `xray_public_key` came out empty, `xray_server_keys_exists` turned false, and
  `config.yml` walked into `server.yml` to reissue the REALITY keypair -- where
  it failed on `xray_keys.container.Output`. That is the exact reissue the tag
  assert in `detect.yml` exists to prevent. The derivation now carries
  `check_mode: false`: `xray x25519 -i` only computes, and the `-runtime`
  container is one-shot and cleaned up.
- **pihole**: the three `SELECT COUNT(*)` list probes in `config.yml` are
  `ansible.builtin.command`, which skips itself in a check run unless given
  `creates`/`removes`, leaving `changed_when` no `stdout` to compare against --
  so a check run called the lists converged whatever the `*.list` files held.
  They now carry `check_mode: false`. The DELETE/INSERT handlers still skip, so
  `gravity.db` is never written in a check run.

### Changed

- **singbox**, **xray**: secret generation is now explicitly skipped in a check
  run (`when: not ansible_check_mode`) and the consuming `set_fact` substitutes
  `CHECK-MODE-PLACEHOLDER-*` values. A check run can therefore preview a newly
  added client or a first deployment instead of failing on the missing container
  output, and it never mints a real UUID, short id or keypair.
- Every role README gained a `## Check mode` section stating what a check run
  covers and what it cannot -- notably that the container roles need a reachable
  docker daemon for detection, so check mode does not work against a host that
  does not have the engine yet.

## 2.0.0

Synchronization release: every role brought onto one structure, one set of
conventions and current upstream images.

### Breaking changes

- **pihole**: migrated to Pi-hole v6 (`2024.07.0` -> `2026.07.2`). Every v5
  environment variable was renamed to the `FTLCONF_*` scheme, so
  `pihole_service_config` changed with it: `dnsmasq_listening_address` became
  `dns_listening_mode` (`LOCAL`/`SINGLE`/`BIND`/`ALL`/`NONE`), `web_address` was
  removed, and `etc_dnsmasq_d` was added for the v5 migration start. See the
  role README for the full mapping. Back up the data directory first.
- **torrserver**: switched from `ksey/torrserver:latest` (last built 2022,
  moving tag) to upstream `ghcr.io/yourok/torrserver:MatriX.143`. Container
  paths changed from `/TS/db` to `/opt/ts/config`; the host directory is
  unchanged, so `torrserver.db` survives.
- **grafana**: switched from `grafana/grafana-oss` to `grafana/grafana`. The
  `grafana-oss` mirror stopped receiving tags after `13.0.2`. Provisioned
  dashboards moved from `/var/lib/grafana/dashboards` to
  `/etc/grafana/dashboards` so they no longer nest inside the data volume.
- **singbox**: `singbox_server_config.tag` added and rendered onto the inbound;
  credential readback now locates the inbound by that tag. `route.rules` must be
  an explicit `[]`. The `box.log` bind mount was replaced by a `data` volume
  mounted at the container working directory.
- **collection**: `ansible.posix` and `community.general` are now declared
  dependencies. They were always required by the roles but never declared, so a
  `ansible-galaxy collection install` of this collection alone left the
  `sysctl` and `pam_limits` tasks unable to run.

### Fixed

- Image updates never took effect. Every role's `update.yml` notified a handler
  using `state: restarted`, which maps to `docker compose restart` and keeps the
  existing container and image. Added a `recreate container` handler using
  `docker compose up` with `pull: always` and `recreate: always`.
- `start container` handlers ran without `become` in eight roles while the
  matching `restart` handler had it, so a first deployment depended on the
  connection user's docker group membership having already taken effect.
- `docker_container_info` in every `detect.yml` ran without `become`, for the
  same reason.
- **dnscrypt**, **pihole**: `when` conditions tested an undefined `services`
  variable instead of `ansible_facts.services`, so the avahi, dnsmasq and
  systemd-resolved handling silently never ran.
- **singbox**: a client carrying its own `uuid`/`short_id` inherited the
  credentials generated for the previous client in the loop, handing two clients
  the same UUID. Generated facts are now reset per iteration.
- **singbox**: `client.yml` built `{'name': item}` from a mapping, so the client
  name became a dict — breaking the rendered inbound and the client profile
  filenames.
- **singbox**: `singbox_inactive_clients` matched a list of mappings against
  client names with `rejectattr('name', 'in', ...)`, so every active client was
  treated as inactive. Configured clients are now normalized first.
- **singbox**: `singbox_all_clients` was built as `a + b | sort`, which sorts
  only `b` because Jinja binds a filter tighter than `+`. A reissued client was
  appended rather than substituted, doubling the inbound and leaving one task
  reporting changed forever.
- **singbox**: the one-shot `generate reality-keypair` container was named after
  the service, colliding with the running container.
- **singbox**: credential readback searched the deployed `tls` block for
  `vless_address`, so changing that address silently reissued the keypair and
  every client credential. It now locates the inbound by tag and fails loudly
  when it cannot, and refuses to continue when `config.json` holds a private key
  with no recoverable public key.
- **singbox**, **xray**: `server.j2` and `client.j2` emitted a trailing comma
  whenever the last optional section was empty. `log` is now rendered last and
  unconditionally.
- **grafana**: `datasources.j2` provisioned `secureJsonData` with literal
  `"..."` placeholders for the TLS cert, key and CA.
- **grafana**: two tasks shared the name
  `config | grafana datasources and dashboards folders`.
- **grafana**: dropped `grafana-simple-json-datasource` from the default plugin
  list; it is deprecated and no longer in the catalog.
- **prometheus**: `embedded-exporter.yml` had a `prometheus_target_interval_length_seconds`
  metric misspelled as `rometheus_...`, so the "target scraping slow" alert
  could never fire, plus a stray quote in a rule description.
- **pihole**: `changed_when: pihole_data_dir_result.uid != 999` compared the
  data directory's owner against a uid the container never uses, reporting
  changed on every run. Two tasks also shared the name `install | run handlers`.
- **xray**: `meta/main.yml` described the role as "Sing-Box VPN docker service
  deployment".
- **docker**: `meta/main.yml` declared `license: Docker`; **grafana** declared
  the invalid `license: AGPL-3.0-only, Apache-2.0`.

### Added

- Persistent data volumes where state was previously lost on every image
  update: `prometheus` (TSDB), `alertmanager` (silences and notification log)
  and `grafana` (`grafana.db`). Each is owned by the uid its image runs as,
  exposed as `<role>_service_config.uid`.
- `--storage.tsdb.path` and `--storage.path` in `prometheus_command` and
  `alertmanager_command`: the compose `command` replaces the image `CMD`
  outright, so without them both services wrote to a relative path outside the
  mounted volume.
- `prometheus_service_config.retention_time`, defaulting to `30d`.
- Consistent tags on every role: `<role>.detect`, `<role>.install`,
  `<role>.update`, `<role>.config`. Each include carries `apply.tags` mirroring
  its own tags, without which a tagged run expands the include and then filters
  out everything inside it.
- `singbox_client_inbounds`, mirroring `xray_client_inbounds`.
- `docker_daemon_config`: `/etc/docker/daemon.json` is now templated from a
  mapping instead of a static file.
- `docker_installed_version` fact.
- `python3-requests` in `docker_repo.requirements`.
- `{{ ansible_managed | comment }}` headers on every rendered file, `owner`,
  `group` and explicit `mode` on every file and template task, and `backup: true`
  on configuration files.
- `build_ignore` in `galaxy.yml`, so the test harness and editor droppings stay
  out of the built artifact.

### Changed

- Image tags updated: alertmanager `v0.27.0` -> `v0.34.0`, node_exporter
  `v1.8.2` -> `v1.12.1`, prometheus `v3.0.0` -> `v3.14.0`, grafana `10.4.12` ->
  `13.1.4`, dnscrypt `v2.1.5` -> `2.1.18`, sing-box `v1.12.9` -> `v1.13.19`,
  xray `26.3.23` -> `26.7.28`. Grafana dashboard 1860 pinned to revision 45.
- `config.json` in both VPN roles is now mode `0600`, and client profiles on the
  controller `0600` in a `0700` directory. Both images run as root, so the
  tighter mode costs nothing.
- `min_ansible_version` raised from `2.13` to `2.16` in every role, matching
  `meta/runtime.yml`. Platforms are now `bookworm` and `trixie`.
- All roles declare `license: GPL-3.0-only` — the licence of the role, not of
  the upstream project it deploys.
- `singbox` and `xray` inherit `docker_restart_policy` and `docker_timezone`
  like the other roles instead of hardcoding them. `network_mode` stays `host`
  deliberately, with the reason recorded in the defaults.
- Test harness: `tests/group_vars/xray.yml` still referenced
  `xray_service_config`, `xray_geofiles` and `protocol`-keyed clients, none of
  which the role has used for several releases. Rewritten against the current
  variables, `tests/group_vars/singbox.yml` added, and `tests/playbook.yml` now
  applies the `docker` role it always depended on.
- Every role README documents its tags, the facts it sets and the host-level
  changes it makes.

## 1.1.0

- Refactor collection

## 1.0.0

- Add collection
