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

### `prometheus_retention_time`

Type: `str`. Required: `false`.

Maximum sample age, for example 30d or 1w12h. An empty string omits the time
flag.

Default:

```yaml
prometheus_retention_time: ''
```

### `prometheus_retention_size`

Type: `str`. Required: `false`.

Maximum local block size, for example 50GB. An empty string omits the size flag.
A positive size alone disables time-based retention.

Default:

```yaml
prometheus_retention_size: ''
```

### `prometheus_web_config`

Type: `dict`. Required: `false`.

Native Prometheus web configuration, including tls_server_config and
basic_auth_users. An empty mapping disables the web config file.

Default:

```yaml
prometheus_web_config: {}
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
- `/etc/prometheus/web-config.ansible.yml`
- `/etc/default/prometheus`
- `/etc/sysconfig/prometheus`
- `/etc/systemd/system/prometheus.service`
- `/etc/systemd/system/prometheus.service.d/10-ansible.conf`
- `/usr/local/bin/prometheus`
- `/usr/local/bin/promtool`
- `/usr/local/libexec/prometheus/`
- `/var/lib/prometheus/metrics2/`
- `/var/lib/prometheus/metrics/`

## Check Mode

The install task supports check mode on a fresh host. For a full role check,
provision the selected release first so the matching promtool is available to
validate configuration.

## Service Behavior

Package installations use the distribution systemd unit and service account.
Binary installations verify the release archive against its SHA-256 manifest and
supply their own unit and service account. Both modes validate the same
configuration with the selected promtool, use the distribution's environment
file and data path, and enable the service at boot. Both units run with a
read-only filesystem and write TSDB data only in the active data directory.
Changes to configuration, retention or installation mode restart the server.

## Security Notes

- Only root and the prometheus service group can read the configuration. Use
  Ansible Vault for scrape credentials.
- The HTTP API has no authentication by default. Keep the loopback listener or
  restrict access before exposing it.
- Supply bcrypt password hashes in `prometheus_web_config.basic_auth_users`. Use
  absolute paths for TLS files readable by the service account. The role
  validates the web configuration with `promtool check web-config` before
  installation. An empty mapping restores the plain HTTP listener.

## Operational Notes

- With `tasks_from: install`, package installations include the vendor's default
  files. Run the full role to apply the scrape configuration and activate the
  selected service.
- openSUSE uses `/etc/sysconfig/prometheus` and `/var/lib/prometheus/metrics`.
  AlmaLinux, Debian, Fedora and Ubuntu use `/etc/default/prometheus` and
  `/var/lib/prometheus/metrics2`.
- Package mode installs the available distribution version and leaves updates to
  the package manager. To keep an upstream installation pinned, set
  `prometheus_install: binary` and specify its numeric `prometheus_version`.
- Binary mode stores `prometheus-<version>` and `promtool-<version>` directly in
  `/usr/local/libexec/prometheus/`. The install task points the commands in
  `/usr/local/bin/` at the selected release. The full role also updates the
  version used by the systemd unit and restarts the service when that version
  changes. When switching to package mode, the full role removes the local links
  so the distribution commands in `/usr/bin/` take effect.
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
- With both retention variables empty, Prometheus defaults to 15 days. A
  positive size alone applies no age limit. When both are set, blocks expire at
  the first limit reached. Versions supporting native retention settings give
  them precedence over the CLI flags.

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

### Install Prometheus without configuring or starting it

Provision commands and the service account while retaining service state.

```yaml
- name: Provision Prometheus
  hosts: monitoring
  tasks:
    - name: Install the distribution package
      ansible.builtin.include_role:
        name: jomrr.prometheus
        tasks_from: install
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

### Serve HTTPS with basic authentication and size-based retention

Provision the TLS files and Vault credentials before applying this example.

```yaml
- name: Secure the Prometheus listener
  hosts: monitoring
  roles:
    - role: jomrr.prometheus
      prometheus_retention_size: 50GB
      prometheus_web_config:
        tls_server_config:
          cert_file: /etc/prometheus/tls/server.crt
          key_file: /etc/prometheus/tls/server.key
        basic_auth_users:
          monitor: "{{ vault_monitor_bcrypt }}"
      prometheus_config:
        scrape_configs:
          - job_name: prometheus
            scheme: https
            tls_config:
              ca_file: /etc/prometheus/tls/ca.crt
              server_name: prometheus.example.org
            basic_auth:
              username: monitor
              password: "{{ vault_monitor_password }}"
            static_configs:
              - targets: ['127.0.0.1:9090']
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
- [Prometheus storage retention](https://prometheus.io/docs/prometheus/latest/storage/)
- [Prometheus HTTPS and authentication](https://prometheus.io/docs/prometheus/latest/configuration/https/)

## Author

[Jonas Mauer](https://github.com/jomrr)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for the full license text.

Copyright (c) 2026 Jonas Mauer.
