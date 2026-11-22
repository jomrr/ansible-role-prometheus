# Ansible Role: prometheus

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-prometheus)
![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-prometheus)
![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-prometheus)
[![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-prometheus/dev.yml?branch=dev&label=dev)](https://github.com/jomrr/ansible-role-prometheus/actions/workflows/dev.yml?query=branch%3Adev)
[![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-prometheus/main.yml?branch=main&label=main)](https://github.com/jomrr/ansible-role-prometheus/actions/workflows/main.yml?query=branch%3Amain)

Install Prometheus from distribution packages or an upstream binary release and
configure the server.

## Purpose

Run Prometheus as a standalone time-series database. By default it listens on
127.0.0.1:9090 and scrapes its own metrics every 15 seconds. Supply a complete
`prometheus_config` to monitor other targets.

## Requirements

- Package installs on AlmaLinux use EPEL. The role installs epel-release from
  AlmaLinux Extras.
- Binary installs require x86_64 or aarch64 and HTTPS access to upstream release
  assets on GitHub.
- Binary hosts following a release channel also need HTTPS access to
  prometheus.io.

## Dependencies

```yaml
collections:
  - name: community.general
    version: '>=12.0.0'
  - name: ansible.posix
    version: '>=2.0.0'
  - name: containers.podman
    version: '>=1.7.0'
```

## Role Variables

### `prometheus_install`

Type: `str`. Required: `false`.

Install the distribution package or an upstream binary release.

Default:

```yaml
prometheus_install: package
```

### `prometheus_version`

Type: `str`. Required: `false`.

Binary release to install. Use a major.minor.patch version or follow the stable,
latest or lts field in the Prometheus download catalog.

Default:

```yaml
prometheus_version: stable
```

### `prometheus_listen_address`

Type: `str`. Required: `false`.

HTTP listen endpoint in host:port or [IPv6]:port form.

Default:

```yaml
prometheus_listen_address: 127.0.0.1:9090
```

### `prometheus_config`

Type: `dict`. Required: `false`.

Complete native Prometheus configuration, including scrape jobs and optional
alerting or remote storage settings.

Default:

```yaml
prometheus_config:
  global:
    scrape_interval: 15s
  scrape_configs:
    - job_name: prometheus
      static_configs:
        - targets:
            - 127.0.0.1:9090
```

## Managed Files

- `/etc/prometheus/prometheus.yml`
- `/etc/default/prometheus`
- `/etc/sysconfig/prometheus`
- `/etc/systemd/system/prometheus.service`
- `/usr/local/bin/prometheus`
- `/usr/local/bin/promtool`
- `/usr/local/libexec/prometheus/`
- `/var/lib/prometheus/metrics2/`
- `/var/lib/prometheus/metrics/`

## Check Mode

Run check mode after provisioning the selected release. Initial installation and
version upgrades need the matching promtool binary.

## Service Behavior

Package installations use the distribution systemd unit and service account.
Binary installations verify the release archive against its SHA-256 manifest and
supply their own unit and service account. Both modes validate the same
configuration with the selected promtool, use the distribution's environment
file and data path, and enable the service at boot. Changes to configuration or
installation mode restart the server.

## Security Notes

- Only root and the prometheus service group can read the configuration. Use
  Ansible Vault for scrape credentials.
- The HTTP API has no authentication by default. Keep the loopback listener or
  restrict access before exposing it.

## Operational Notes

- openSUSE uses `/etc/sysconfig/prometheus` and `/var/lib/prometheus/metrics`.
  AlmaLinux, Debian, Fedora and Ubuntu use `/etc/default/prometheus` and
  `/var/lib/prometheus/metrics2`.
- Package mode installs the available distribution version and leaves updates to
  the package manager. To keep an upstream installation pinned, set
  `prometheus_install: binary` and specify its numeric `prometheus_version`.
- Binary mode stores `prometheus-<version>` and `promtool-<version>` directly in
  `/usr/local/libexec/prometheus/`. The commands in `/usr/local/bin/` point to
  the selected release. A version change switches the links and restarts the
  service. Switching to package mode removes these links so the distribution
  commands in `/usr/bin/` take effect.
- Binary installations follow `stable` by default. With `stable`, `latest` or
  `lts`, each run reads the official download catalog. `stable` selects the
  highest stable version, `latest` the release marked latest by Prometheus, and
  `lts` the highest version marked for long-term support. An LTS series change
  takes effect on the next run. If the catalog lookup fails or the channel has
  no release, the running service keeps its version.
- Both modes reuse a database created by earlier role versions directly in
  /var/lib/prometheus. Before selecting an older server version, check
  configuration and storage compatibility and back up the database.
- Use absolute paths for external files such as rule files, client certificates
  and bearer tokens. Provision them with read access for the service account.
- Changing the HTTP listener also requires updating any self-scrape target that
  uses the old address.

## Supported Platforms

| OS Family | Distribution | Version | Container Image |
| --------- | ------------ | ------- | --------------- |
| RedHat | AlmaLinux | latest | [jomrr/molecule-almalinux:latest](https://hub.docker.com/r/jomrr/molecule-almalinux) |
| Debian | Debian | latest | [jomrr/molecule-debian:latest](https://hub.docker.com/r/jomrr/molecule-debian) |
| RedHat | Fedora | latest | [jomrr/molecule-fedora:latest](https://hub.docker.com/r/jomrr/molecule-fedora) |
| Suse | OpenSuse Leap | latest | [jomrr/molecule-opensuse-leap:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-leap) |
| Suse | OpenSuse Tumbleweed | latest | [jomrr/molecule-opensuse-tumbleweed:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-tumbleweed) |
| Debian | Ubuntu | latest | [jomrr/molecule-ubuntu:latest](https://hub.docker.com/r/jomrr/molecule-ubuntu) |

## Example Playbook

### Monitor the Prometheus server

Query the `up` metric at `http://127.0.0.1:9090` after the first scrape.

```yaml
- name: Start Prometheus
  hosts: monitoring
  gather_facts: true
  roles:
    - role: jomrr.prometheus
```

### Pin an upstream binary release

Install a specific upstream release independently of the distribution package version.

```yaml
- name: Run a pinned Prometheus server
  hosts: monitoring
  gather_facts: true
  roles:
    - role: jomrr.prometheus
      prometheus_install: binary
      prometheus_version: '3.13.5'
```

### Follow the upstream LTS channel

Install the newest binary release marked for long-term support.

```yaml
- name: Run the current Prometheus LTS release
  hosts: monitoring
  roles:
    - role: jomrr.prometheus
      prometheus_install: binary
      prometheus_version: lts
```

### Scrape an application

Collect metrics from an application every 30 seconds.

```yaml
- name: Monitor an application
  hosts: monitoring
  gather_facts: true
  roles:
    - role: jomrr.prometheus
      prometheus_config:
        global:
          scrape_interval: 30s
        scrape_configs:
          - job_name: application
            static_configs:
              - targets: ['app.example.org:8080']
```

## References

- [Prometheus releases and checksums](https://prometheus.io/download/)
- [Prometheus release catalog](https://prometheus.io/download.json)
- [Prometheus configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)

## Author

[Jonas Mauer](https://github.com/jomrr)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for the full license text.

Copyright (c) 2026 Jonas Mauer.
