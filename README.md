# f5-bd-ansible-f5-upgrades

Automated upgrade orchestration for F5 BIG-IP devices (standalone and HA pairs) and F5OS platforms (rSeries / Velos), with pre/post snapshot collection, JSON diff reporting, and bastion-based artifact storage.

---

## Status

| Platform | Status |
|---|---|
| BIG-IP HA Pair | ✅ Testing ready |
| BIG-IP Standalone | 🚧 Testing In Progress |
| F5OS rSeries | 🚧 Work in progress |
| F5OS Velos | 🚧 Work in progress |

---

## Repository Structure

```
f5-bd-ansible-f5-upgrades/
├── ansible.cfg
├── collections/
│   └── requirements.yml
├── inventory/
│   └── hosts.ini                        # Reference only - inventory managed in AAP
├── group_vars/
│   ├── bigip_ha.yml                     # Connection + defaults for bigip_ha group
│   ├── bigip.yml                        # Connection + defaults for bigip_standalone group
│   └── f5os.yml                        # Connection + defaults for F5OS groups
├── playbooks/
│   ├── BIG-IP/
│   │   ├── Upgrades/
│   │   │   ├── ha/
│   │   │   │   ├── upgrade.yaml         # HA pair upgrade (9 plays)
│   │   │   │   └── extra_vars/
│   │   │   │       └── upgrade_vars.yml
│   │   │   └── standalone/
│   │   │       ├── upgrade.yaml         # Standalone upgrade (5 plays)
│   │   │       └── extra_vars/
│   │   │           └── upgrade_vars.yml
│   │   └── Tests/
│   │       └── test_software_install.yaml  # Socket isolation test playbook
│   └── F5OS/
│       └── Upgrades/
│           ├── rSeries/
│           │   └── upgrade.yaml         # 🚧 Work in progress
│           └── Velos/
│               └── upgrade.yaml         # 🚧 Work in progress
└── roles/
    ├── bastion_image_push/              # Stage ISO from bastion CIFS to BIG-IP
    ├── bastion_storage/                 # Store artifacts from EE to bastion CIFS
    ├── bigip_upgrade/                   # Install and activate image via module
    ├── bigip_wait/                      # Layered post-reboot readiness checks
    ├── snapshot_pre/                    # Baseline state capture (all partitions)
    ├── snapshot_post/                   # Post-upgrade capture + diff + regression check
    ├── snapshot_report/                 # AAP stdout diff summary
    ├── f5os_upgrade_rseries/            # rSeries image import + install
    └── f5os_wait_rseries/               # rSeries post-reboot readiness checks
```

---

## Requirements

### Ansible
- Ansible Core >= 2.15
- Ansible Automation Platform 2.6 (AAP)

### Collections
```bash
ansible-galaxy collection install -r collections/requirements.yml
```

| Collection | Purpose |
|---|---|
| `f5networks.f5_bigip` | BIG-IP iControl REST modules (httpapi) |
| `f5networks.f5os` | F5OS REST modules |
| `ansible.netcommon` | Network connection plugins |

### Bastion Requirements
- `sshpass` installed on the bastion host
  ```bash
  sudo yum install -y sshpass   # RHEL/Rocky
  sudo apt install -y sshpass   # Ubuntu
  ```

### BIG-IP Compatibility
- BIG-IP 13.1+ (uses `/mgmt/shared/echo` for REST readiness)
- Tested on 16.1.x → 17.5.x upgrades

---

## Network Topology

```
AAP EE  <── HTTPS/SSH ──>  BIG-IP mgmt
   │
   └── SSH ──> bastion
                  │
                  └── CIFS mount  /mnt/software/F5/BIGIP/   (ISO images)
                                  /mnt/software/F5-Backups/  (UCS + snapshots)
```

- **Image transfer**: bastion reads ISO from CIFS, uploads to BIG-IP via `sshpass + SCP`
- **Backup storage**: UCS and snapshot JSONs flow `BIG-IP → EE → bastion CIFS`
- The bastion has no direct HTTPS path to BIG-IP

---

## License Management

The `bigip_license` role checks `serviceCheckDate` against today's date and reactivates if needed. It runs before image staging on both HA nodes and before upgrade on standalone.

### How it works

1. Reads `serviceCheckDate` and `registrationKey` from `/mgmt/tm/sys/license` via REST
2. Compares `serviceCheckDate` against today — if valid, skips reactivation entirely
3. If reactivation needed: `bigip_license` module on the EE contacts `activate.f5.com` directly — BIG-IP does **not** need internet access
4. After reactivation the device goes `Offline` briefly while services restart
5. Polls `sync-status` until `Changes Pending` or `In Sync` (positive poll — only exits on known-good states)
6. If `Changes Pending` → triggers config sync, polls until `In Sync`
7. Confirms new `serviceCheckDate >= today` before proceeding

### Modes

| Mode | Behaviour |
|---|---|
| `automatic` | EE proxies reactivation to `activate.f5.com`. BIG-IP needs no internet. Standard reg key licenses only. **Default.** |
| `check_only` | Validates `serviceCheckDate` only. Hard fails if expired with clear message. |
| `skip` | Skips license check for all hosts. |
| `bigip_license_skip: true` | Skips license role for a specific host (set in inventory host vars). |

### BIG-IQ Licensed Devices

> **Important**: If your BIG-IP was licensed through BIG-IQ (pool license or BIG-IQ issued registration key), set `bigip_license_skip: true` and reactivate manually through BIG-IQ **before** running the upgrade playbook. Do not use `automatic` mode — `SOAPLicenseClient` points to `activate.f5.com` not your BIG-IQ instance and may break BIG-IQ license tracking.
>
> Set `bigip_license_skip: true` in AAP inventory host vars (not extra_vars) so it applies per-device.

### HA Behaviour

| Play | Host | Mode | Notes |
|---|---|---|---|
| Play 1a | Standby | From extra_vars (`automatic` default) | Safe — standby going Offline briefly is acceptable |
| Play 1b | Active | Always `check_only` (hardcoded) | Never auto-reactivates active — risks failover |
| Play 6 | Old-active (now standby) | From extra_vars, `skip_sync=true` (hardcoded) | Nodes on different versions — sync skipped, `Changes Pending` clears after both nodes upgraded |

### Key Extra Variables

```yaml
bigip_license_mode: automatic        # automatic | check_only | skip
bigip_license_skip: false            # set true per-host in inventory for BIG-IQ licensed devices
bigip_license_server: activate.f5.com  # must be reachable from AAP EE
bigip_license_skip_sync: false       # managed automatically by Play 6 - do not override
```


### Upgrade Flow

```
Play 0   Pre-flight         Validate vars, detect live HA role (active/standby),
                            check versions, detect split-brain resume condition
Play 1a  License (standby)  Check/reactivate license on standby via EE proxy
Play 1b  License (active)   Check license on active (check_only - no reactivation)
Play 2   Image staging      bastion SCP → BIG-IP /shared/images/ (both nodes)
Play 3   Pre-snapshot       Collect VS, pools, routes, ARP, sync state (both nodes)
Play 4   Upgrade standby    Install + activate → URI volume poll → reboot → bigip_wait
Play 5   Verify standby     Post-snapshot + diff report on standby
Play 6   License (old-active) Reactivate old-active (now standby), skip_sync=true
Play 7   Failover           Force active → standby on original active node
Play 8   Upgrade active     Install + activate → URI volume poll → reboot → bigip_wait
Play 9   Failback           Optional: restore original active/standby roles
Play 10  Final snapshot     Post-snapshot + diff report (both nodes)
Play 11  Store artifacts    Copy JSONs + UCS from EE to bastion CIFS
```

### Split-Brain Detection

If a previous upgrade run was interrupted after one node was upgraded but before the second node, the playbook detects this automatically:
- Node already on target version → `bigip_already_upgraded: true` → skipped
- Peer node disconnected due to version mismatch → allowed to proceed
- Genuinely out of sync (not version mismatch) → aborted with clear message

### AAP Job Template Setup

| Field | Value |
|---|---|
| Playbook | `playbooks/BIG-IP/Upgrades/ha/upgrade.yaml` |
| Inventory | `bigip_ha` group |
| Credential | AAP Machine Credential (BIG-IP admin) + bastion Machine Credential |
| Extra Variables | Paste from `extra_vars/upgrade_vars.yml` |

### Key Extra Variables

```yaml
bigip_target_version: "17.5.1"
bigip_image_filename: "BIGIP-17.5.1.5-0.0.6.iso"
bastion_image_path: "/mnt/software/F5/BIGIP/17.x/17.5.1"
bastion_backup_root: "/mnt/software/F5-Backups/bigip-backups"
bigip_install_volume: "HD1.2"
ansible_command_timeout: 600
```

> **WARNING**: Do NOT include `ansible_connection`, `ansible_network_os`, or `ansible_httpapi_*` in AAP Extra Variables — they override inventory group vars and break bastion delegation.

---

## BIG-IP Standalone Upgrade

### Upgrade Flow

```
Play 0  Pre-flight        Validate vars, check version, set timestamps
Play 1  Image staging     bastion SCP → BIG-IP /shared/images/
Play 2  Pre-snapshot      Collect VS, pools, routes, ARP, sync state
Play 3  Upgrade           Install + activate → URI volume poll → reboot → bigip_wait
                          NOTE: Device is OFFLINE during reboot - schedule maintenance window
Play 4  Post-snapshot     Post-upgrade state capture + diff report
Play 5  Store artifacts   Copy JSONs + UCS from EE to bastion CIFS
```

### AAP Job Template Setup

| Field | Value |
|---|---|
| Playbook | `playbooks/BIG-IP/Upgrades/standalone/upgrade.yaml` |
| Inventory | `bigip_standalone` group |
| Credential | AAP Machine Credential (BIG-IP admin) + bastion Machine Credential |
| Extra Variables | Paste from `extra_vars/upgrade_vars.yml` |

---

## Image Install Flow (Both HA and Standalone)

The `bigip_upgrade` role uses the following pattern due to a known socket issue with the `f5networks.f5_bigip` collection:

```
1. bigip_software_install state=activated
   └── Triggers install. httpapi socket dies here (known module bug - ticket raised).

2. Pause 120s
   └── Allows BIG-IP to pass the 0% startup phase before polling begins.

3. URI poll /mgmt/tm/sys/software/volume/HD1.2 every 30s (up to 30 min)
   └── Exit when:
       a. status == 'complete'   → install done, pause 6 min for reboot to start
       b. 'restarting' in status → device already rebooting
       c. non-200 response       → device unreachable, mid-reboot

4. bigip_wait role
   └── Layered readiness checks (see below)
```

> **Known Issue**: `bigip_software_install` with `volume_uri` polling fails after `state=activated` because the httpapi persistent socket is destroyed during the install process. URI polling is used as a workaround. Ticket raised against `f5networks.f5_bigip`.

---

## Post-Reboot Readiness (bigip_wait)

```
Check  SSH already up?     → If yes, skip port down/up layers (device came back
                              during install pause) and go straight to REST checks

Layer 1  SSH port DOWN      → Confirms reboot started (ignored if already up)
Layer 2  SSH port UP        → Kernel + sshd running (skipped if already up)
Layer 3  HTTPS port UP      → Web stack listening
Layer 4  /mgmt/shared/echo  → Returns {"stage":"STARTED"} when REST initialised
Layer 5  /mgmt/tm/sys/version → mcpd config plane ready
Layer 6a /mgmt/tm/ltm/virtual → LTM config loaded
Layer 6b /mgmt/tm/cm/sync-status → HA awareness confirmed
         └── If sync == 'offline': retry every 20s for up to 5 minutes
```

---

## Snapshot & Diff

Every upgrade run produces three JSON files per device stored on the bastion:

```
/mnt/software/F5-Backups/bigip-backups/<hostname>/<date>/
  <hostname>_<timestamp>_pre.json
  <hostname>_<timestamp>_post.json
  <hostname>_<timestamp>_diff.json
  pre_upgrade_<hostname>_<timestamp>.ucs
```

### What is captured

| Data | Method | Notes |
|---|---|---|
| Virtual server status | `bigip_device_info` | All partitions via partition loop |
| Pool member health | `bigip_device_info` | All partitions via partition loop |
| Sync status | `bigip_device_info` | |
| System version | `bigip_device_info` | |
| Route table | `bigip_command` tmsh | stdout lines |
| ARP table | `bigip_command` tmsh | stdout lines |
| HA failover state | `uri` | No module subset available |
| UCS backup | `bigip_ucs_fetch` | Saved pre-upgrade only |

### Multi-Partition Collection

Partitions are collected in two passes to capture VSes in all non-Common partitions (e.g. `T1-JuiceShop/WebApp-VIP`):
1. `gather_subset: partitions` — get all partition names
2. Loop `bigip_device_info` with `gather_subset: virtual-servers, ltm-pools` per partition

### Regression Detection

The diff report fails the play if any virtual server that was `available` pre-upgrade is not `available` post-upgrade. Sync state changes are **not** treated as regressions — version mismatch between HA nodes always causes `Disconnected` sync state during the upgrade window.

---

## Variables Reference

### BIG-IP Common

| Variable | Default | Description |
|---|---|---|
| `bigip_target_version` | required | Version string to verify post-upgrade |
| `bigip_image_filename` | required | ISO filename on bastion CIFS |
| `bastion_image_path` | required | Full path to ISO on bastion |
| `bastion_backup_root` | required | Root path for backups on bastion |
| `bigip_install_volume` | `HD1.2` | Boot volume to install to |
| `bigip_reboot_after_install` | `true` | Activate after install |
| `bigip_upgrade_dry_run` | `false` | Skip install and reboot |
| `bigip_https_port` | `443` | BIG-IP management HTTPS port |
| `bigip_ssh_port` | `22` | BIG-IP SSH port |
| `bigip_reboot_initial_delay` | `60` | Seconds before bigip_wait starts probing |
| `bigip_wait_retry_delay` | `20` | Seconds between readiness probe retries |
| `bigip_wait_max_retries` | `30` | Max readiness probe attempts |
| `snapshot_output_dir` | `/runner/artifacts/snapshots` | EE staging dir for snapshots |
| `bigip_snapshot_save_ucs` | `true` | Save UCS backup pre-upgrade |
| `snapshot_fail_on_regression` | `true` | Fail play if VS regressions detected |
| `ansible_command_timeout` | `600` | httpapi persistent connection timeout |

### BIG-IP HA Only

| Variable | Default | Description |
|---|---|---|
| `bigip_failback` | `false` | Fail back to original active after upgrade |

---

## F5OS rSeries Upgrade

> 🚧 **Testing in progress** — role and playbook are built and structured but not yet validated against hardware.

### Key Difference from BIG-IP

rSeries **pulls** the image from a remote URL — it does not receive a push from the bastion. The image must be hosted on an HTTP/HTTPS/SCP server reachable from the rSeries management interface. The bastion can serve this role if it has an HTTP server configured.

```
BIG-IP:   bastion CIFS --> sshpass SCP push --> BIG-IP /shared/images/
rSeries:  rSeries pulls <-- HTTP/HTTPS/SCP <-- image server
```

### Upgrade Flow

```
Play 0  Pre-flight     Validate vars, check version, set timestamps
Play 1  Pre-snapshot   Collect system-info baseline
Play 2  Upgrade        f5os_system_image_import (pull from URL, two-step)
                       f5os_system_image_install (install by version, two-step)
                       f5os_wait_rseries (layered HTTPS + REST readiness)
Play 3  Post-snapshot  Collect post-upgrade state, version diff report
Play 4  Store artifacts Copy JSONs to bastion CIFS
```

### Modules Used

| Module | Purpose |
|---|---|
| `f5os_device_info` | Collect system info (version, platform) |
| `f5os_system_image_import` | Pull image from remote URL to rSeries staging |
| `f5os_system_image_install` | Install staged image by version string |

### Image URL Format

```yaml
# HTTP (simplest - serve from bastion if it has httpd)
f5os_image_url: "http://192.170.7.50/f5os/F5OS-A-1.8.0-13798.R4R5.iso"

# HTTPS
f5os_image_url: "https://files.example.com/f5os/F5OS-A-1.8.0-13798.R4R5.iso"

# SCP
f5os_image_url: "scp://user@192.170.7.50/images/F5OS-A-1.8.0-13798.R4R5.iso"
```

### AAP Job Template Setup

| Field | Value |
|---|---|
| Playbook | `playbooks/F5OS/Upgrades/rSeries/upgrade.yaml` |
| Inventory | `f5os_rseries` group |
| Credential | AAP Machine Credential (rSeries admin) |
| Extra Variables | Paste from `extra_vars/upgrade_vars.yml` |

### Key Extra Variables

```yaml
f5os_target_version: "1.8.0"
f5os_image_version: "1.8.0-13798"
f5os_image_url: "http://192.170.7.50/f5os/F5OS-A-1.8.0-13798.R4R5.iso"
bastion_backup_root: "/mnt/software/F5-Backups/f5os-backups"
ansible_command_timeout: 600
```

---

## F5OS Velos Upgrade

> 🚧 **Work in progress** — Velos has a more complex upgrade path (controller + partitions) and is not yet implemented.

Velos differs from rSeries in that it has a **controller** and separate **partitions**, each requiring their own software version management:
- Controller upgrade: `f5os_system_image_import` + `f5os_system_image_install` (same as rSeries)
- Partition upgrade: `velos_partition_image` + `velos_partition` (set `os_version`) + `velos_partition_wait`

Placeholder playbook at `playbooks/F5OS/Upgrades/Velos/upgrade.yaml`.

---

## Troubleshooting

### `ansible_command_timeout` errors
Set `ansible_command_timeout: 600` in the AAP inventory group variables for `bigip_ha` and `bigip_standalone` groups. The default 30 seconds is too short for `bigip_device_info` which makes internal AS3/package checks on initialization.

### Image transfer permission denied
The bastion needs `sshpass` installed. The BIG-IP SSH host key must not be stale in the bastion's `known_hosts` — the playbook handles this automatically with `ssh-keygen -R` before each transfer.

### Socket path errors on bigip_software_install
Known issue with `f5networks.f5_bigip` — the httpapi socket is destroyed after `state=activated`. The role works around this with URI polling. See the test playbook at `playbooks/BIG-IP/Tests/test_software_install.yaml` to validate module behaviour in your environment.

### bigip_wait timing out on SSH port down
If the device rebooted during the install pause the SSH port will already be up when `bigip_wait` runs. The role detects this and skips the port down/up layers automatically.

---

## License

Apache 2.0 — see LICENSE file.
