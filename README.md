# Jira Software Data Center Ansible Automation

Production-oriented Ansible automation for installing, configuring, validating, and uninstalling Atlassian Jira Software Data Center on RHEL 9 with PostgreSQL.

The role supports **two installation methods**:

- **Archive installation** using the Atlassian `.tar.gz` distribution.
- **Linux binary installer** using the Atlassian `-x64.bin` installer in unattended mode.

Both methods converge on the same Ansible-managed configuration, systemd service, validation, and uninstall lifecycle.

> Current tested Jira version: **11.3.8**

## Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Directory Structure](#directory-structure)
- [Requirements](#requirements)
- [Installation Methods](#installation-methods)
  - [Archive Method](#archive-method)
  - [Linux Binary Installer Method](#linux-binary-installer-method)
- [Preparing the Installers](#preparing-the-installers)
- [Inventory](#inventory)
- [Configuration Variables](#configuration-variables)
- [Usage](#usage)
  - [Syntax Check](#syntax-check)
  - [Precheck](#precheck)
  - [Install Jira](#install-jira)
  - [Reconfigure Jira](#reconfigure-jira)
  - [Validate Jira](#validate-jira)
  - [Uninstall Jira](#uninstall-jira)
- [Role Task Flow](#role-task-flow)
- [Binary Installer Response File](#binary-installer-response-file)
- [Database Configuration Behavior](#database-configuration-behavior)
- [Systemd Ownership Model](#systemd-ownership-model)
- [Idempotency](#idempotency)
- [Uninstall Behavior](#uninstall-behavior)
- [Cluster Data Center Mode](#cluster-data-center-mode)
- [Tags Reference](#tags-reference)
- [Validation](#validation)
- [Troubleshooting](#troubleshooting)
- [Security Recommendations](#security-recommendations)

## Overview

The `jira` role manages the complete Jira Software Data Center lifecycle:

```text
precheck
   |
   v
prerequisites
   |
   v
installation method dispatcher
   |
   +-- archive   -> install_archive.yml
   |
   +-- installer -> install_installer.yml
   |
   v
configure
   |
   v
systemd
   |
   v
validate
```

The installation method is selected with:

```yaml
jira_install_method: "archive"
```

or:

```yaml
jira_install_method: "installer"
```

The archive and binary installer paths have both been tested successfully with Jira Software 11.3.8.

The binary installer implementation additionally supports:

- unattended Install4j execution;
- a generated response file;
- custom installation and Jira Home directories;
- custom HTTP and control ports;
- prevention of Atlassian-managed service creation;
- prevention of automatic Jira startup;
- normalization of the installer-generated runtime user;
- handoff to the same Ansible-managed systemd service used by the archive installation.

## Key Features

- RHEL 9 prechecks.
- Java 21 support.
- PostgreSQL connectivity validation.
- Installer SHA256 verification.
- Dual installation methods: `.tar.gz` and `.bin`.
- Custom installation directory under `/app`.
- Jira Home management.
- Jira Data Center shared-home support.
- Cluster node configuration.
- JVM heap and code-cache configuration.
- Tomcat HTTP connector configuration.
- Ansible-managed systemd service.
- HTTP and process validation.
- Safe handling of Jira-secured `dbconfig.xml`.
- Idempotent repeated execution.
- Common uninstall workflow for archive and binary installations.
- Database preserved by default during uninstall.

## Directory Structure

```text
jira-dc-ansible/
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
        │   ├── atlassian-jira-software-11.3.8-x64.bin
        │   └── atlassian-jira-software-11.3.8.tar.gz
        ├── handlers
        │   └── main.yml
        ├── meta
        │   └── main.yml
        ├── tasks
        │   ├── configure.yml
        │   ├── install.yml
        │   ├── install_archive.yml
        │   ├── install_installer.yml
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
            ├── response.varfile.j2
            ├── server.xml.j2
            └── setenv.sh.j2
```

## Requirements

### Control node

- Ansible 2.15 or later recommended.
- Python 3.
- Jira installation media stored under `roles/jira/files/`.

### Managed node

- RHEL 9.
- At least 10 GB free space on `/app`.
- Network access to the PostgreSQL server.
- Privilege escalation/root access for package, user, directory, and systemd management.

### Database

The PostgreSQL database and database account must already exist and be reachable.

Example configuration:

```yaml
jira_db_host: "localhost"
jira_db_port: 15432
jira_db_name: "jira"
jira_db_username: "jira"
jira_db_password: "jira"
```

For production, store the database password in **Ansible Vault** rather than plaintext inventory.

## Installation Methods

### Archive Method

Set:

```yaml
jira_install_method: "archive"
```

The role uses:

```yaml
jira_archive: "atlassian-jira-software-{{ jira_version }}.tar.gz"
jira_archive_sha256: "<SHA256>"
jira_extract_dir: "atlassian-jira-software-{{ jira_version }}-standalone"
```

The archive workflow is implemented in:

```text
roles/jira/tasks/install_archive.yml
```

The role:

1. verifies the archive and checksum;
2. copies it to the target staging directory;
3. removes incomplete installation content if necessary;
4. extracts the archive;
5. renames the extracted directory to `jira_install_dir`;
6. normalizes ownership;
7. validates the Jira start and stop scripts;
8. removes the temporary staged archive.

### Linux Binary Installer Method

Set:

```yaml
jira_install_method: "installer"
```

The role uses:

```yaml
jira_installer: "atlassian-jira-software-{{ jira_version }}-x64.bin"
jira_installer_sha256: "<SHA256>"
```

For Jira 11.3.8 the tested SHA256 is:

```text
369137f5b170684e010526b8da43bfbbb71aaf3c1a54184059e5f61b2d07860b
```

The binary workflow is implemented in:

```text
roles/jira/tasks/install_installer.yml
```

The installer is executed unattended using Install4j:

```text
-q
-varfile <response-file>
-dir <installation-directory>
```

The response file explicitly prevents the Atlassian installer from creating its own service or starting Jira before Ansible configuration is complete.

## Preparing the Installers

Place the installation media under:

```text
roles/jira/files/
```

For Jira 11.3.8:

```text
roles/jira/files/atlassian-jira-software-11.3.8.tar.gz
roles/jira/files/atlassian-jira-software-11.3.8-x64.bin
```

Verify checksums before configuring the inventory:

```bash
sha256sum roles/jira/files/atlassian-jira-software-11.3.8.tar.gz
sha256sum roles/jira/files/atlassian-jira-software-11.3.8-x64.bin
```

The precheck validates only the media required by the selected `jira_install_method`.

For example, when:

```yaml
jira_install_method: "installer"
```

the archive checks are skipped and the binary installer checksum is validated.

## Inventory

Example `inventory/hosts.yml`:

```yaml
all:
  children:
    jira:
      hosts:
        localhost:
          ansible_connection: local
```

For remote nodes:

```yaml
all:
  children:
    jira:
      hosts:
        jira01.example.com:
          ansible_host: 10.1.20.20
          ansible_user: ansible
```

For multiple Data Center nodes, define host-specific node IDs.

## Configuration Variables

### Product and installation

| Variable | Purpose |
|---|---|
| `jira_version` | Jira Software version |
| `jira_install_method` | `archive` or `installer` |
| `jira_archive` | `.tar.gz` filename |
| `jira_archive_sha256` | SHA256 for archive |
| `jira_installer` | Linux `.bin` filename |
| `jira_installer_sha256` | SHA256 for binary installer |
| `jira_extract_dir` | Directory produced by archive extraction |
| `jira_base_dir` | Base installation filesystem |
| `jira_install_dir` | Final Jira installation directory |
| `jira_home_dir` | Jira local home |
| `jira_shared_dir` | Jira shared home |
| `jira_temp_dir` | Installer staging directory |

Example:

```yaml
jira_version: "11.3.8"
jira_install_method: "installer"

jira_archive: "atlassian-jira-software-{{ jira_version }}.tar.gz"
jira_archive_sha256: "ec7cac96e9df2e732af5ab3ad00bdaed82aeb8d153de87fd049f23fb775bfba5"

jira_installer: "atlassian-jira-software-{{ jira_version }}-x64.bin"
jira_installer_sha256: "369137f5b170684e010526b8da43bfbbb71aaf3c1a54184059e5f61b2d07860b"

jira_base_dir: "/app"
jira_install_dir: "{{ jira_base_dir }}/jira-{{ jira_version }}"
jira_home_dir: "{{ jira_base_dir }}/jira-data"
jira_shared_dir: "/shared/jira"
jira_temp_dir: "/var/tmp"
```

### Runtime user

```yaml
jira_user: "jira"
jira_group: "jira"
jira_user_home: "/home/jira"
```

The binary installer can generate `bin/user.sh` with another runtime account such as `jira1`. The Ansible installer workflow normalizes this file back to the configured `jira_user`.

### JVM

```yaml
java_home: "/usr/lib/jvm/java-21-openjdk-21.0.12.0.8-1.2.el9.x86_64"

jira_jvm_min_heap: "2g"
jira_jvm_max_heap: "4g"
jira_jvm_code_cache: "512m"
jira_gc: "G1GC"
```

### Network

```yaml
jira_http_port: 8008
jira_https_port: 8443
jira_control_port: 8005
jira_context_path: ""
```

`jira_control_port` is also supplied to the Linux binary installer response file.

### Database

```yaml
jira_database_type: "postgres72"

jira_db_host: "localhost"
jira_db_port: 15432
jira_db_name: "jira"
jira_db_username: "jira"
jira_db_password: "jira"

jira_db_driver: "org.postgresql.Driver"
jira_db_jdbc_version: "42.7.11"
jira_db_jdbc_jar: "postgresql-{{ jira_db_jdbc_version }}.jar"
```

### Data Center

```yaml
jira_deployment_mode: "cluster"
jira_cluster_name: "jira-dc"
jira_node_id: "jira-node01"

jira_ehcache_listener_port: 40001
jira_ehcache_object_port: 40011
```

### Service

```yaml
jira_service_name: "jira"

jira_start_script: "{{ jira_install_dir }}/bin/start-jira.sh"
jira_stop_script: "{{ jira_install_dir }}/bin/stop-jira.sh"

jira_pid_file: "{{ jira_install_dir }}/work/catalina.pid"

jira_service_enabled: true
jira_service_started: true

jira_start_timeout: 300
jira_stop_timeout: 300
```

### Uninstall

```yaml
jira_remove_install_dir: true
jira_remove_home: true
jira_remove_shared_home: true
jira_remove_user: true
jira_remove_group: true

jira_remove_database: false
```

`jira_remove_database: false` documents the intended safety behavior: the current uninstall workflow does **not** modify the PostgreSQL database.

## Usage

Run commands from the repository root.

### Syntax Check

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jira.yml \
  --syntax-check
```

For uninstall:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/uninstall_jira.yml \
  --syntax-check
```

### Precheck

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jira.yml \
  --tags precheck
```

Precheck validates:

- RHEL 9;
- selected installation method;
- selected installer existence;
- selected installer SHA256;
- Jira user/group state;
- installation and home directories;
- Java;
- PostgreSQL reachability;
- minimum free disk space.

### Install Jira

Select the desired method in:

```text
inventory/group_vars/jira.yml
```

For archive:

```yaml
jira_install_method: "archive"
```

For binary installer:

```yaml
jira_install_method: "installer"
```

Then run:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jira.yml
```

### Reconfigure Jira

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jira.yml \
  --tags configure
```

Configuration tasks manage Jira application properties, JVM settings, cluster configuration, Tomcat settings, and Jira database bootstrap behavior.

### Validate Jira

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jira.yml \
  --tags validate
```

### Uninstall Jira

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/uninstall_jira.yml
```

The uninstall is common to both archive and binary installations.

## Role Task Flow

`roles/jira/tasks/main.yml` imports:

| Tag | Task file | Purpose |
|---|---|---|
| `always`, `precheck` | `precheck.yml` | Platform, installer, Java, PostgreSQL and disk checks |
| `prerequisites` | `prerequisites.yml` | Packages, Java, user/group and directories |
| `install` | `install.yml` | Select installation method |
| `configure` | `configure.yml` | JVM, DB bootstrap, cluster and Tomcat configuration |
| `systemd` | `systemd.yml` | Ansible-managed Jira systemd service |
| `validate` | `validate.yml` | Full post-install validation |

`install.yml` dispatches to:

```text
jira_install_method=archive
        |
        +--> install_archive.yml

jira_install_method=installer
        |
        +--> install_installer.yml
```

`uninstall.yml` is called directly by `playbooks/uninstall_jira.yml`.

## Binary Installer Response File

The template is:

```text
roles/jira/templates/response.varfile.j2
```

It was derived from the response file generated by a successful Jira Software 11.3.8 Install4j installation.

Representative content:

```text
# install4j response file for Jira Software {{ jira_version }}

app.install.service$Boolean=false
app.jiraHome={{ jira_home_dir }}
existingInstallationDir=/opt/Jira Software
httpPort$Long={{ jira_http_port }}
launch.application$Boolean=false
portChoice=custom
rmiPort$Long={{ jira_control_port }}
sys.adminRights$Boolean=true
sys.adminRightsUiRootUnix$Boolean=false
sys.confirmedUpdateInstallationString=false
sys.installationDir={{ jira_install_dir }}
sys.languageId=en
```

Two settings are particularly important:

```text
app.install.service$Boolean=false
launch.application$Boolean=false
```

Ansible deliberately prevents the binary installer from creating or starting its own service.

The desired lifecycle is:

```text
Atlassian .bin
     |
     v
Install Jira files only
     |
     v
Normalize installer runtime user
     |
     v
configure.yml
     |
     v
jira.service.j2
     |
     v
systemd.yml
     |
     v
Start Jira
     |
     v
validate.yml
```

### Installer runtime user

The Jira `.bin` installer may generate:

```text
{{ jira_install_dir }}/bin/user.sh
```

with an installer-selected account such as:

```bash
JIRA_USER="jira1"
```

This conflicts with the Ansible-managed service when it runs as:

```ini
User=jira
Group=jira
```

`install_installer.yml` therefore normalizes `JIRA_USER` to:

```bash
JIRA_USER="jira"
```

or, more generally, the value of:

```yaml
jira_user
```

This is required for the binary installer and is not needed for archive extraction.

## Database Configuration Behavior

`dbconfig.xml` requires special handling because Jira modifies it after startup.

The initial Ansible template contains the database credentials and connection settings. After Jira starts, Jira may rewrite the file. For the tested Jira 11.3.8 deployment, Jira:

- added the XML declaration;
- reformatted the XML;
- replaced the plaintext password with `{ATL_SECURED}`;
- added `pool-max-idle`;
- added `tcpKeepAlive=true`.

For example, the live file contains:

```xml
<password>{ATL_SECURED}</password>
```

instead of the original plaintext password.

Therefore `configure.yml` treats `dbconfig.xml` as a **bootstrap configuration**:

1. check whether `{{ jira_home_dir }}/dbconfig.xml` exists;
2. create it from `dbconfig.xml.j2` only when absent;
3. on subsequent runs, preserve Jira's runtime-managed content;
4. continue enforcing file ownership and mode.

This prevents Ansible from replacing `{ATL_SECURED}` with a plaintext password and prevents an unnecessary Jira restart on every playbook run.

A deliberate database migration or connection change should be treated as a separate operational procedure rather than silently overwriting an existing runtime `dbconfig.xml`.

## Systemd Ownership Model

Regardless of installation method, Jira is managed by Ansible systemd configuration.

The Atlassian binary installer is configured with:

```text
Install as service: No
Start Jira: No
```

Ansible then deploys:

```text
roles/jira/templates/jira.service.j2
```

through `systemd.yml`.

This provides one consistent service lifecycle for both:

```text
.tar.gz -> Ansible systemd
.bin    -> Ansible systemd
```

rather than mixing Atlassian installer service management with Ansible service management.

## Idempotency

The installer method has been tested through repeated execution.

After a successful binary installation, a second full playbook run completed with:

```text
ok=109
changed=0
unreachable=0
failed=0
skipped=14
```

This verifies that the role:

- detects an existing Jira installation;
- skips the binary installer;
- does not recreate the response file unnecessarily;
- preserves the normalized Jira runtime user;
- preserves Jira's secured `dbconfig.xml`;
- does not restart Jira when configuration is unchanged;
- validates the running installation successfully.

A clean second run with `changed=0` is the expected steady-state behavior.

## Uninstall Behavior

The same Ansible uninstall workflow supports both installation methods.

The role intentionally does **not** invoke the Install4j uninstaller for `.bin` deployments. Ansible owns the lifecycle and removes the installation directory directly.

For a binary installation this also removes:

```text
{{ jira_install_dir }}/.install4j/
```

because it is contained within the Jira installation directory.

The uninstall workflow:

1. detects the Jira systemd unit;
2. stops Jira;
3. disables Jira;
4. removes the Ansible-managed service file;
5. reloads systemd;
6. verifies no Jira Java process remains;
7. removes the installation directory when enabled;
8. removes Jira Home when enabled;
9. removes shared home when enabled;
10. removes the Jira OS user/group when enabled;
11. removes temporary archive/binary/response files;
12. validates the resulting state;
13. preserves the PostgreSQL database.

Example:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/uninstall_jira.yml
```

A tested full uninstall resulted in:

```text
jira.service       -> removed
/app/jira-11.3.8   -> removed
/app/jira-data     -> removed
/shared/jira       -> removed
jira OS user       -> removed
jira OS group      -> removed
Jira process       -> none
PostgreSQL jira DB -> preserved
```

The database is intentionally **not modified**.

Be careful with:

```yaml
jira_remove_home: true
jira_remove_shared_home: true
```

These settings delete Jira application data from the filesystem. Review them before running an uninstall in production.

## Cluster Data Center Mode

Set:

```yaml
jira_deployment_mode: "cluster"
```

Cluster mode enables:

- Jira shared-home creation;
- `cluster.properties`;
- node ID configuration;
- shared-home validation;
- shared-home cleanup during uninstall when enabled.

Example:

```yaml
jira_cluster_name: "jira-dc"
jira_node_id: "jira-node01"
jira_shared_dir: "/shared/jira"
```

Each Jira node must have a unique `jira_node_id`.

For multiple nodes, define node-specific values in `host_vars` or directly on inventory hosts rather than sharing one node ID across the group.

## Tags Reference

```bash
# Precheck
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jira.yml \
  --tags precheck

# Prerequisites
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jira.yml \
  --tags prerequisites

# Installation
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jira.yml \
  --tags install

# Configuration
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jira.yml \
  --tags configure

# systemd
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jira.yml \
  --tags systemd

# Validation
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jira.yml \
  --tags validate
```

For a new host, prefer the complete playbook rather than running individual later-stage tags.

## Validation

The role validates, among other things:

- installation directory;
- `start-jira.sh`;
- `stop-jira.sh`;
- PostgreSQL JDBC driver;
- Jira Home;
- `dbconfig.xml`;
- `cluster.properties` in cluster mode;
- shared home;
- systemd service file;
- service active state;
- Java process;
- PID file;
- HTTP port;
- Jira HTTP endpoint;
- PostgreSQL reachability;
- Java version;
- installation ownership;
- log directory;
- Jira cluster node ID.

A successful validation summary resembles:

```text
========================================
Jira validation completed
Install Directory : /app/jira-11.3.8
Home Directory    : /app/jira-data
HTTP Port         : 8008
Database          : jira
Service           : jira
Deployment Mode   : cluster
========================================
```

## Troubleshooting

### Binary installer checksum failure

Verify:

```bash
sha256sum roles/jira/files/atlassian-jira-software-11.3.8-x64.bin
```

and compare it with:

```yaml
jira_installer_sha256
```

### Archive checksum failure

Verify:

```bash
sha256sum roles/jira/files/atlassian-jira-software-11.3.8.tar.gz
```

and compare it with:

```yaml
jira_archive_sha256
```

### Binary installer creates `jira1`

If Jira fails with a message similar to:

```text
Jira has been installed to run as jira1
```

check:

```bash
cat /app/jira-11.3.8/bin/user.sh
```

The binary installer workflow should normalize:

```bash
JIRA_USER="jira"
```

to match `jira_user`.

Do not solve this by changing the systemd service to `jira1`; Ansible should remain the authority for the Jira runtime account.

### Jira service does not start

Check:

```bash
systemctl status jira.service --no-pager -l
journalctl -u jira.service -n 100 --no-pager
```

Then:

```bash
tail -100 /app/jira-11.3.8/logs/catalina.out
```

and, when present:

```bash
tail -100 /app/jira-data/log/atlassian-jira.log
```

### HTTP validation takes a long time

Jira startup may require several minutes. The role waits for the configured HTTP port using `jira_start_timeout`.

Check in another terminal:

```bash
systemctl status jira --no-pager -l
ss -lntp | grep ':8008'
ps -ef | grep '[j]ava'
```

### `dbconfig.xml` changes on every run

Do not continuously overwrite Jira's live `dbconfig.xml`.

Jira 11.3.8 secures and rewrites this file after startup. The current role treats it as bootstrap-only and preserves the runtime version on subsequent runs.

### PostgreSQL unreachable

Verify:

```bash
nc -vz <jira_db_host> <jira_db_port>
```

or:

```bash
psql -h <jira_db_host> -p <jira_db_port> -U jira -d jira
```

Also check firewall and PostgreSQL listener/`pg_hba.conf` configuration.

### Installer method selection

Confirm:

```bash
grep -E \
  'jira_install_method|jira_archive:|jira_installer:' \
  inventory/group_vars/jira.yml
```

Expected examples:

```text
jira_install_method: "installer"
jira_archive: "atlassian-jira-software-{{ jira_version }}.tar.gz"
jira_installer: "atlassian-jira-software-{{ jira_version }}-x64.bin"
```

### Verbose Ansible troubleshooting

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jira.yml \
  -vvv
```

## Security Recommendations

- Store `jira_db_password` in Ansible Vault.
- Do not commit production database passwords.
- Verify Atlassian installer SHA256 values before deployment.
- Restrict access to `inventory/group_vars/` if it contains secrets.
- Back up Jira Home, shared home, and PostgreSQL before destructive uninstall or upgrade operations.
- Review `jira_remove_home` and `jira_remove_shared_home` before running uninstall.
- Keep `jira_remove_database: false` unless database deletion is implemented as an explicit, separately controlled operation.
- Use unique `jira_node_id` values for every Data Center node.
- Validate firewall access to PostgreSQL and Jira ports.
- Test installer/version changes in a non-production environment before rollout.

## Tested Lifecycle

The Jira 11.3.8 binary-installer implementation has been exercised through:

```text
Clean host
   |
   v
Precheck
   |
   v
Unattended .bin installation
   |
   v
Ansible configuration
   |
   v
Ansible-managed systemd
   |
   v
Jira startup
   |
   v
Full validation
   |
   v
Second full run
   |
   +--> changed=0 / failed=0
   |
   v
Ansible uninstall
   |
   +--> service removed
   +--> installation removed
   +--> home removed
   +--> shared home removed
   +--> user/group removed
   +--> process absent
   +--> PostgreSQL database preserved
```

The archive installation remains available by changing only:

```yaml
jira_install_method: "archive"
```

This allows one Jira role to maintain both supported deployment approaches while preserving a common configuration, validation, systemd, and uninstall model.
