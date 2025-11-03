# Artefactual Migrate Ansible Role

This project provides an Ansible role that installs the Artefactual `migrate`
CLI together with its bundled Temporal CLI, configures application settings, and
manages the accompanying `systemd` units on Rocky Linux 9 and Ubuntu 20.04+.
It is intended for servers that already run the Archivematica Storage Service;
other environments are out of scope.

## Role layout

The Ansible role lives at the repository root with the usual subdirectories
(`defaults/`, `files/`, `handlers/`, `meta/`, `tasks/`, `templates/`, `vars/`).
Defaults cover downloading release artifacts, installing both CLI binaries,
templating `config.json`, and provisioning the worker and Temporal services.
Temporal is run in bundled local-development mode using the packaged binary
(`temporal server start-dev --db-filename ./temporal.db`). Storage Service
management commands default to host mode, expecting direct access to `manage.py`.
The role provisions a dedicated `temporal` system user for the Temporal service,
while the migrate worker runs as the existing `archivematica` user with its
configuration stored at `~/.config/migrate/config.json`. Two services are
managed:

- `temporal.service` keeps the bundled Temporal development server running with
  its database stored under `/var/lib/migrate`.
- `migrate.service` runs `migrate worker` so workflows are processed in the
  Temporal queue as soon as jobs are submitted.

## Quick start

```yaml
- hosts: migrate_hosts
  become: true
  roles:
    - role: migrate
      vars:
        migrate_storage_service_api_username: "api_user"
        migrate_storage_service_api_key: "secret"
        migrate_storage_service_source_location_id: "source-location-uuid"
        migrate_storage_service_move_target_location_id: "target-location-uuid"
        migrate_storage_service_replication_targets:
          - id: "replica-location-1"
            name: "Replica Location 1"
        migrate_storage_service_host_manage_path: "/opt/storage_service/manage.py"
        migrate_temporal_grpc_port: 7233
        migrate_temporal_web_port: 8233
```

The role defaults to downloading release `v0.1.0`.  Override `migrate_version`
when new versions are published.  Verify that the Storage Service settings and
workflow toggles match your environment. Helper variables prefixed with
`migrate_storage_service_` and `migrate_temporal_` let you adjust Storage
Service credentials, host command paths, location identifiers, and Temporal
ports without redefining the full `migrate_config` structure.

> Note: SELinux port labeling relies on the `community.general` collection. Add
> it to your playbook requirements if you plan to run on SELinux-enabled hosts.

## Variables

- `migrate_version`: Git tag to download (`v0.1.0` by default).
- `migrate_package_url`: Optional full URL to a `.deb` or `.rpm` artifact; when
  set it overrides the tag-derived URL.
- `migrate_arch_map`: Maps detected CPU architectures to release suffixes.
- `migrate_release_base_url`: Override to use an internal artifact mirror.
- `migrate_config_dir` / `migrate_data_dir`: File system layout for config and
  runtime data (config defaults to `~archivematica/.config/migrate`).
- `migrate_temporal_extra_args`: Extra flags appended to the Temporal `start-dev`
  command.
- `migrate_temporal_grpc_port` / `migrate_temporal_web_port`: Ports exposed by
  the Temporal dev server (used for config, systemd, and SELinux labeling).
- `migrate_service_environment`: Extra environment key/value pairs for the
  worker service.
- `migrate_service_home`: Base directory used for the migrate worker user
  (defaults to `/var/lib/archivematica`).
- `migrate_storage_service_api_url` / `migrate_storage_service_api_username` /
  `migrate_storage_service_api_key`: Credentials injected into the Storage
  Service API block inside `config.json`.
- `migrate_storage_service_host_python_path` / `migrate_storage_service_host_manage_path` /
  `migrate_storage_service_host_environment`: How Storage Service management
  commands are executed on the host.
- `migrate_storage_service_source_location_id` /
  `migrate_storage_service_move_target_location_id` /
  `migrate_storage_service_replication_targets`: Storage Service location
  identifiers embedded into `config.json`.
- `migrate_selinux_required_packages` / `migrate_selinux_port_labels`: Control
  which packages are installed and which ports receive SELinux labels when
  SELinux is enabled (defaults label the Temporal gRPC/Web ports defined above
  as `http_port_t`).
- `temporal_runtime_user` / `temporal_runtime_group`: Account under which the
  Temporal dev server runs (`temporal` by default).
- `migrate_service_user` / `migrate_service_group`: Existing account used for
  the migrate worker (`archivematica` by default).
- `migrate_service_home`: Home directory for the migrate worker
  (`/var/lib/archivematica` by default).

See `defaults/main.yml` for the full list.

## Installed Files

By default the role lays down the following on the target host:

- `/usr/bin/migrate` and `/usr/bin/temporal` from the downloaded package.
- `/etc/systemd/system/migrate.service` (worker service unit).
- `/etc/systemd/system/temporal.service` (Temporal dev server unit).
- `/var/lib/archivematica/.config/migrate/config.json` (rendered migrate configuration).
- `/var/lib/archivematica/.config/migrate/{{ migrate_sqlite_filename }}` (SQLite state database).
- `/var/lib/migrate` owned by the Temporal service account for runtime data.

## Default Ports

- Temporal gRPC: `{{ migrate_temporal_grpc_port }}` (defaults to `7233/tcp`).
- Temporal Web UI: `{{ migrate_temporal_web_port }}` (defaults to `8233/tcp`).
- The migrate worker consumes Temporal only and exposes no additional listening port.

## Testing

You can exercise the role locally with `ansible-playbook` using the supplied
example or integrate it into your existing playbooks. Molecule scenarios can be
added later as part of CI once the target infrastructure is known.
