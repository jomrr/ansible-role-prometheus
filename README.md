# Ansible Role: prometheus

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-prometheus)
![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-prometheus)
![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-prometheus)
[![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-prometheus/dev.yml?branch=dev&label=dev)](https://github.com/jomrr/ansible-role-prometheus/actions/workflows/dev.yml?query=branch%3Adev)
[![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-prometheus/main.yml?branch=main&label=main)](https://github.com/jomrr/ansible-role-prometheus/actions/workflows/main.yml?query=branch%3Amain)

Install the upstream Prometheus server and manage its scrape configuration and
systemd service.

## Purpose

Run Prometheus as a standalone time-series database. By default it listens on
127.0.0.1:9090 and scrapes its own metrics every 15 seconds. Supply a complete
`prometheus_config` to monitor other targets.

## Requirements

- A systemd host on x86_64 or aarch64 with HTTPS access to Prometheus release
  assets on GitHub.

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

### `prometheus_version`

Type: `str`. Required: `false`.

Upstream release version in major.minor.patch form.

Default:

```yaml
prometheus_version: 3.13.5
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
- `/etc/systemd/system/prometheus.service`
- `/usr/local/libexec/prometheus/`
- `/var/lib/prometheus/`

## Check Mode

Run check mode after provisioning the selected release. Initial installation and
version upgrades need the matching promtool binary.

## Service Behavior

The role verifies the archive against the release SHA-256 manifest and checks
configuration changes with promtool before installing them. It restarts
Prometheus on changes, preserves the time series in /var/lib/prometheus and
enables startup at boot. Reapplying the same settings leaves a running service
undisturbed.

## Security Notes

- Only root and the prometheus service group can read the configuration. Use
  Ansible Vault for scrape credentials.
- The HTTP API has no authentication by default. Keep the loopback listener or
  restrict access before exposing it.

## Operational Notes

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
- [Prometheus configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)

## Author

[Jonas Mauer](https://github.com/jomrr)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for the full license text.

Copyright (c) 2026 Jonas Mauer.
