# Jira Software (Data Center) Ansible Automation

Ansible automation to install, configure, validate, and uninstall Atlassian
Jira Software / Data Center on RHEL 9 hosts, with optional cluster (Data
Center) support and PostgreSQL as the backing database.

## Contents

- [Overview](#overview)
- [Directory Structure](#directory-structure)
- [Requirements](#requirements)
- [Preparing the Installer](#preparing-the-installer)
- [Inventory](#inventory)
- [Configuration Variables](#configuration-variables)
- [Usage](#usage)
  - [1. Install Jira](#1-install-jira)
  - [2. Reconfigure Jira](#2-reconfigure-jira)
  - [3. Validate the Installation](#3-validate-the-installation)
  - [4. Uninstall Jira](#4-uninstall-jira)
- [Role Task Flow](#role-task-flow)
- [Tags Reference](#tags-reference)
- [Cluster (Data Center) Mode](#cluster-data-center-mode)
- [Known Issues / Things to Fix Before Production Use](#known-issues--things-to-fix-before-production-use)
- [Troubleshooting](#troubleshooting)

## Overview

A single `jira` role drives the entire lifecycle (precheck → prerequisites →
install → configure → systemd → validate), imported with tags from
`roles/jira/tasks/main.yml`. Two playbooks wrap the role:

- `playbooks/install_jira.yml` — runs the full role (all tags) to install and
  bring Jira up.
- `playbooks/uninstall_jira.yml` — imports only the `uninstall.yml` task file
  to tear Jira down.

There is no separate `configure`/`validate`-only playbook shipped yet; those
stages are reached via `--tags` against `install_jira.yml` (see
[Usage](#usage)).

## Directory Structure

```
jira-dc-ansible
├── ansible.cfg
├── inventory
│   ├── group_vars
│   │   └── jira.yml
│   └── hosts.yml
├── playbooks
│   ├── install_jira.yml
│   └── uninstall_jira.yml
└── roles
    └── jira
        ├── defaults
        │   └── main.yml
        ├── files
        │   └── atlassian-jira-software-11.3.8.tar.gz
        ├── handlers
        │   └── main.yml
        ├── meta
        │   └── main.yml
        ├── tasks
        │   ├── configure.yml
        │   ├── install.yml
        │   ├── main.yml
        │   ├── precheck.yml
        │   ├── prerequisites.yml
        │   ├── systemd.yml
        │   ├── uninstall.yml
        │   └── validate.yml
        └── templates
            ├── cluster.properties.j2
            ├── dbconfig.xml.j2
            ├── jira-application.properties.j2
            ├── jira.service.j2
            ├── server.xml.j2
            └── setenv.sh.j2
```

## Configuration Variables

Variables referenced by the role (define/override in
`inventory/group_vars/jira.yml` and `roles/jira/defaults/main.yml`):

**Package & paths**

| Variable             | Used for                                             |
|-----------------------|-------------------------------------------------------|
| `jira_version`        | Displayed during precheck; should match the archive   |
| `jira_archive`        | Installer filename under `files/`                     |
| `jira_extract_dir`    | Directory name produced by extracting the archive     |
| `jira_install_dir`    | Final installation path (e.g. `/app/jira`)             |
| `jira_home_dir`       | Jira home directory                                    |
| `jira_shared_dir`     | Shared home (cluster mode only)                        |

**User / ownership**

| Variable               | Used for                                    |
|--------------------------|----------------------------------------------|
| `jira_user` / `jira_group` | OS account Jira runs as, and file ownership in most tasks |
| `jira_user_home`         | Home directory for the `jira_user` account   |
| `jira_owner` / `jira_owner_group` | Ownership used specifically for `jira_home_dir` / `jira_shared_dir` in `prerequisites.yml` — see [Known Issues](#known-issues--things-to-fix-before-production-use) |
| `jira_directory_mode`    | Mode for `jira_home_dir`                     |
| `jira_shared_dir_mode`   | Mode for `jira_shared_dir`                   |

**JVM / Tomcat**

| Variable              | Used for                              |
|-------------------------|-----------------------------------------|
| `jira_jvm_min_heap`     | `JVM_MINIMUM_MEMORY` in `setenv.sh`     |
| `jira_jvm_max_heap`     | `JVM_MAXIMUM_MEMORY` in `setenv.sh`     |
| `jira_jvm_code_cache`   | `-XX:ReservedCodeCacheSize` in `setenv.sh` |
| `jira_http_port`        | Tomcat connector port in `server.xml`   |
| `jira_context_path`     | Context path used by the HTTP validation check |

**Database**

| Variable            | Used for                                  |
|-----------------------|---------------------------------------------|
| `jira_db_host`        | PostgreSQL host (precheck + validate connectivity) |
| `jira_db_port`        | PostgreSQL port                             |
| `jira_db_name`        | Database name (shown in validation summary) |
| `jira_db_jdbc_jar`    | Expected JDBC driver filename under `{{ jira_install_dir }}/lib` |

**Deployment / clustering**

| Variable                | Used for                                     |
|----------------------------|-------------------------------------------------|
| `jira_deployment_mode`     | `"standalone"` or `"cluster"` — gates shared-home, `cluster.properties`, and node-id checks |
| `jira_node_id`             | Written to / verified in `cluster.properties` when clustered |

**Service**

| Variable                  | Used for                                    |
|------------------------------|------------------------------------------------|
| `jira_service_name`         | systemd unit name                              |
| `jira_service_enabled`      | Whether the unit is enabled at boot            |
| `jira_service_started`      | Whether the play starts the service            |
| `jira_start_script` / `jira_stop_script` | Paths validated in `systemd.yml`  |
| `jira_pid_file`             | PID file waited on after start, validated later |
| `jira_start_timeout`        | Seconds to wait for the PID file to appear     |

## Usage

Run from the project root so `ansible.cfg` and relative paths (`files/`,
`inventory/`) resolve correctly.

### 1. Install Jira

```bash
ansible-playbook playbooks/install_jira.yml
```

This runs, in order: `precheck` → `prerequisites` → `install` → `configure`
→ `systemd` → `validate` (see [Role Task Flow](#role-task-flow)). The
playbook's `pre_tasks` also independently detect `JAVA_HOME` and fail early
if Java isn't already present — in practice the role's own
`prerequisites.yml` installs `java-21-openjdk`, so run the full playbook
rather than skipping straight to later tags on a fresh host.

To install a single node in cluster mode, set `jira_deployment_mode: cluster`
and a unique `jira_node_id` for that host (e.g. in `host_vars/`), then run
the same playbook against each node.

### 2. Reconfigure Jira

To re-apply JVM heap, Tomcat port, `dbconfig.xml`, `cluster.properties`, or
`jira-application.properties` changes without repeating install/prerequisite
steps:

```bash
ansible-playbook playbooks/install_jira.yml --tags configure
```

Configuration changes trigger the `Restart Jira` handler automatically.

### 3. Validate the Installation

```bash
ansible-playbook playbooks/install_jira.yml --tags validate
```

Checks install/home/shared-home directories, `dbconfig.xml`, JDBC driver
presence, systemd service state, the Java process, the PID file, HTTP
reachability (`200`/`302`), PostgreSQL connectivity, file ownership, the logs
directory, and (in cluster mode) the node ID in `cluster.properties`. See
[Known Issues](#known-issues--things-to-fix-before-production-use) for one
variable this stage needs defined.

### 4. Uninstall Jira

```bash
ansible-playbook playbooks/uninstall_jira.yml
```

Stops and disables the systemd service, removes the unit file, and removes
(per the hardcoded vars in the playbook) the install directory, Jira home,
shared home (cluster mode), OS user, and OS group. The installer archive
under `files/` is preserved. Review
[Known Issues](#known-issues--things-to-fix-before-production-use) — this
playbook currently targets `hosts: all`, not `hosts: jira`.

## Role Task Flow

`roles/jira/tasks/main.yml` imports, in order:

| Tags                    | File                | Purpose |
|--------------------------|---------------------|---------|
| `always`, `precheck`     | `precheck.yml`      | OS check (RHEL 9), archive presence, user/group/dir existence report, Java check, PostgreSQL reachability, disk space (>=10 GB on `/app`) |
| `prerequisites`          | `prerequisites.yml` | Installs Java 21, rsync/unzip/tar/wget/fontconfig, creates `jira` user/group, `/app`, install dir, home dir, shared dir (cluster) |
| `install`                | `install.yml`       | Copies/extracts the installer, cleans up incomplete installs, sets ownership, validates start/stop scripts exist |
| `configure`               | `configure.yml`     | Creates home/shared-home dirs, edits `setenv.sh` (heap, code cache), deploys `jira-application.properties`, `dbconfig.xml`, `cluster.properties` (cluster mode), sets Tomcat HTTP port in `server.xml`, validates the deployed files |
| `systemd`                | `systemd.yml`       | Deploys the systemd unit, enables/starts the service, waits for the PID file, checks active/enabled state |
| `validate`                | `validate.yml`      | Full post-install validation (see above) |
| *(imported directly, not tagged in `main.yml`)* | `uninstall.yml` | Full teardown, called via `import_role: tasks_from: uninstall` in `uninstall_jira.yml` |

## Tags Reference

```bash
# Only OS/prereq checks
ansible-playbook playbooks/install_jira.yml --tags precheck

# Only install packages/user/dirs
ansible-playbook playbooks/install_jira.yml --tags prerequisites

# Only extract/install the app
ansible-playbook playbooks/install_jira.yml --tags install

# Only push config + restart
ansible-playbook playbooks/install_jira.yml --tags configure

# Only (re)deploy the systemd unit / start service
ansible-playbook playbooks/install_jira.yml --tags systemd

# Only run validation
ansible-playbook playbooks/install_jira.yml --tags validate
```

## Cluster (Data Center) Mode

Set `jira_deployment_mode: cluster` to enable:

- Creation of `jira_shared_dir` (`prerequisites.yml`, `configure.yml`).
- Deployment of `cluster.properties` from `cluster.properties.j2`, keyed by
  `jira_node_id`.
- Shared-home and cluster-properties checks in `validate.yml`.
- Shared-home removal in `uninstall.yml` when `jira_remove_shared_home: true`.

Each cluster node needs its own unique `jira_node_id`; set this per-host
(e.g. in `inventory/host_vars/<node>.yml`) rather than in the shared
`group_vars/jira.yml`.

## Known Issues / Things to Fix Before Production Use

These are things spotted in the current task files worth resolving before
relying on this automation in production:

- **`validate.yml` references an undefined variable.** The final `assert`
  checks `jira_service_file.stat.exists`, but no task in this role registers
  `jira_service_file` (the systemd unit's `stat` result isn't captured
  anywhere). Either add a `stat` task for
  `/etc/systemd/system/{{ jira_service_name }}.service` registered as
  `jira_service_file` in `validate.yml`, or remove that line from the
  `assert`.
- **`uninstall_jira.yml` targets `hosts: all`** instead of `hosts: jira`.
  Running it as-is will attempt the Jira uninstall role against every host
  in inventory, not just Jira nodes. Change it to `hosts: jira` (or a more
  specific group/limit) before running against a real inventory.
- **Ownership variable mismatch between `prerequisites.yml` and
  `configure.yml`.** `prerequisites.yml` creates `jira_home_dir` and
  `jira_shared_dir` owned by `jira_owner`/`jira_owner_group`, while
  `configure.yml` creates/expects them owned by `jira_user`/`jira_group`, and
  `validate.yml`'s ownership check also uses `jira_user`/`jira_group`. Make
  sure `jira_owner`/`jira_owner_group` are set identically to
  `jira_user`/`jira_group` in `group_vars/jira.yml`, or standardize on one
  pair of variables.
- **No dedicated `configure_jira.yml` / `validate_jira.yml` playbooks** exist
  yet (unlike the PostgreSQL automation, which has one playbook per stage).
  Today you reach those stages via `--tags` on `install_jira.yml`. Add thin
  wrapper playbooks if you want parity/consistency with the PostgreSQL repo.
- **`install_jira.yml`'s `pre_tasks`** will fail the whole play if Java isn't
  already installed, even though `prerequisites.yml` (further into the same
  role run) would have installed it. Since the `precheck` role tag also runs
  first, this is currently redundant/conflicting — decide whether Java
  installation should be a hard pre-condition or something the role handles.

## Troubleshooting

- **"Jira archive not found"**: confirm the file exists at
  `jira-dc-ansible/files/{{ jira_archive }}` and that `jira_archive` in
  `group_vars/jira.yml` matches the actual filename exactly.
- **"Only RHEL 9 is supported"**: `precheck.yml` asserts
  `ansible_distribution == "RedHat"` and major version `9`; this playbook
  will not run on Rocky/Alma/CentOS or other RHEL versions without editing
  `precheck.yml`.
- **PostgreSQL not reachable**: `precheck.yml` and `validate.yml` both probe
  `jira_db_host:jira_db_port` with `wait_for`. Confirm the database is up,
  the port is open, and firewall rules allow the Jira host to reach it.
- **HTTP validation fails (`Jira HTTP endpoint is unavailable`)**: Jira can
  take several minutes to fully start after `systemd.yml` starts the
  service; re-run with `--tags validate` after confirming
  `systemctl status {{ jira_service_name }}` shows `active` and the logs
  under `{{ jira_install_dir }}/logs` show a completed startup.
- **Verbose output**: add `-vvv` to any `ansible-playbook` command for
  detailed task-level debugging.
