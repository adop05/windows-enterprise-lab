# 02 — Infrastructure and Windows Troubleshooting

## Overview

This document records infrastructure incidents encountered during the development of the Windows Enterprise Administration Lab, including diagnostic commands, root cause analysis, remediation procedures, and verification results.

---

# Incident 01 — Proxmox LVM-Thin Storage Exhaustion

## 1. Problem Description

During the installation and configuration of Windows 11 Enterprise on Proxmox VM 113, the virtual machine encountered a storage-related failure.

### Symptoms

- Windows installation and updates stopped progressing.
- Proxmox reported an `io-error` status.
- The VM appeared to be running, but QEMU had paused execution.
- Graceful shutdown attempts failed.

## 2. Initial Investigation

The following commands were executed on the Proxmox host to diagnose the storage failure.

### Inspect Proxmox Storage

```bash
pvesm status
```

Initial result:

```text
local-lvm    lvmthin    active    100.00%
```

### Inspect LVM Utilization

```bash
lvs -o lv_name,lv_size,data_percent,metadata_percent
```

Output:

```text
LV       LSize       Data%    Meta%
data     <143.23g    100.00   4.34
```

The thin-pool data area was completely full, while metadata utilization remained low.

### Inspect Volume Group Capacity

```bash
vgs -o vg_name,vg_size,vg_free
```

Result:

```text
VG     VSize      VFree
pve    237.47g    14.00g
```

The volume group contained 14 GiB of unallocated space that could be used to expand the thin pool.

## 3. Root Cause

Proxmox uses LVM-thin storage to allocate virtual disks.

Thin provisioning allows virtual machines to have disks whose advertised capacities exceed the physical storage currently allocated to them. Physical blocks are consumed as the guest operating systems write data.

During Windows 11 installation and updates, additional disk writes exhausted the available thin-pool capacity.

Although the virtual disk could still have free space inside Windows, the underlying Proxmox storage pool could no longer allocate additional physical blocks, causing the QEMU I/O error.

## 4. Remediation — Expand LVM-Thin Storage

The available space in the `pve` volume group was used to expand the thin pool.

```bash
lvextend -L +10G /dev/pve/data
```

Storage utilization was verified using `pvesm status`.

### Results

| Metric             | Before     | After      |
| ------------------ | ---------- | ---------- |
| Thin-pool capacity | 143.23 GiB | 153.23 GiB |
| Utilization        | 100%       | 92.17%     |

The expansion temporarily resolved the storage exhaustion.

However, the pool remained above 90% utilization and subsequently reached 100% again during additional Windows activity.

Further storage reclamation was necessary.

## 5. Remediation — Migrate the Splunk VM Disk

An existing Splunk VM had a virtual disk stored on `local-lvm`.

The objective was to migrate the disk to the directory-based `local` storage backend to reduce thin-pool utilization.

### Inspect the VM

```bash
qm config 106
qm status 106
qm listsnapshot 106
```

The VM was stopped and had no Proxmox snapshots.

Its virtual disk was configured as:

```text
scsi0: local-lvm:vm-106-disk-0,size=32G
```

### Enable Disk Images on Local Storage

The initial migration attempt failed with:

```text
storage 'local' does not support vm images
```

The `Disk image` content type was enabled for the `local` storage backend through the Proxmox web interface.

### Migrate the Disk

```bash
qm disk move 106 scsi0 local --format qcow2 --delete 0
```

The `--delete 0` option preserved the original disk during migration.

Afterward, the VM configuration showed:

```text
scsi0: local:106/vm-106-disk-0.qcow2
unused0: local-lvm:vm-106-disk-0
```

After verifying the migration, the original unused disk was removed.

### Results

| Metric                | Before | After  |
| --------------------- | ------ | ------ |
| local-lvm utilization | 92.17% | 82.06% |

Approximately **15.5 GiB** of thin-pool allocation was reclaimed.

Because both storage backends resided on the same physical drive, this operation redistributed storage usage rather than increasing total physical capacity.

## 6. Remediation — Retire Redundant Windows Server VM

Two Windows Server virtual machines had been created during lab preparation.

Neither had an established Active Directory configuration at the time, so the redundant installation (VM 110) was removed while VM 101 was retained.

### Results

| Metric                | Before | After  |
| --------------------- | ------ | ------ |
| local-lvm utilization | 100%   | 92.59% |

Approximately **11.4 GiB** of allocated storage was reclaimed.

## 7. Remediation — TRIM and Discard Operations

To reclaim additional storage without removing more systems, TRIM/discard operations were performed on applicable VMs and LXC containers throughout the homelab.

TRIM allows a filesystem to identify blocks that are no longer in use so the underlying storage system can reclaim their physical allocation.

For thin-provisioned virtual disks, deleting files inside a guest does not necessarily release the corresponding blocks on the Proxmox host.

The TRIM procedure depends on the virtualization method.

### LXC Containers

LXC containers can be trimmed directly from the Proxmox host:

```bash
pct fstrim <CTID>
```

TRIM was performed on the AIOStream and OpenVPN containers.

**AIOStream (CT 109):**

```bash
pct fstrim 109
```

Output:

```text
/var/lib/lxc/109/rootfs/: 6.2 GiB trimmed
```

Thin-pool utilization decreased from **92.59% to 88.68%**.

**OpenVPN (CT 108):**

```bash
pct fstrim 108
```

Output:

```text
/var/lib/lxc/108/rootfs/: 6.3 GiB trimmed
```

Thin-pool utilization decreased from **88.68% to 87.22%**.

Together, these operations reclaimed approximately **8.2 GiB** of physical thin-pool allocation.

The amount reported by `fstrim` does not necessarily equal the physical space recovered, making before-and-after storage measurements important.

### Virtual Machines

Unlike LXC containers, regular Linux VMs generally require TRIM to be executed within the guest operating system.

Discard support must also be enabled along the virtual disk's storage path.

The following command was used to inspect discard capabilities:

```bash
lsblk -D
```

Where necessary, the virtual machine was shut down and **Discard** was enabled in the Proxmox virtual disk settings.

After restarting the guest, TRIM was executed:

```bash
sudo fstrim -av
```

Periodic trimming was enabled where supported:

```bash
sudo systemctl enable --now fstrim.timer
```

### Fedora VM — Discard Troubleshooting

Fedora VM 100 used a 50 GiB virtual disk backed by `local-lvm`.

Initially, the guest reported discard support:

```text
NAME    DISC-GRAN    DISC-MAX
sda     4K           1G
```

A manual TRIM operation reported:

```text
/boot: 2 GiB trimmed
/: 48 GiB trimmed
```

However, Proxmox storage utilization did not decrease.

Investigation of the VM's disk configuration revealed that `discard=on` was missing:

```text
scsi0: local-lvm:vm-100-disk-0,iothread=1,size=50G
```

After enabling Discard in Proxmox, restarting Fedora, and executing TRIM again, physical storage allocation decreased.

### Results

| Metric                 | Before | After  |
| ---------------------- | ------ | ------ |
| Fedora disk allocation | 39.72% | 35.15% |
| Thin-pool utilization  | 87.22% | 85.73% |

Approximately **2.3 GiB** of additional thin-pool allocation was reclaimed.

This demonstrated that successful filesystem-level TRIM reporting does not guarantee that the underlying virtualization storage has released physical blocks.

## 8. Final Recorded Storage Audit

After the storage expansion, disk migration, redundant VM removal, and TRIM operations, the most recent comprehensive Proxmox audit showed:

| Storage   | Utilization | Available |
| --------- | ----------- | --------- |
| local-lvm | 86.27%      | ~21.0 GiB |
| local     | 60.43%      | ~23.3 GiB |

Additional measurements:

```text
LVM-thin metadata utilization: 4.12%
Unallocated volume-group space: 4 GiB
```

Although Windows 11 subsequently became accessible and the QEMU Guest Agent responded, a final post-recovery QEMU status check was not captured.

The following commands can be used to verify the VM and storage state:

```bash
qm status 113 --verbose | grep -E '^(status|qmpstatus):'
pvesm status
df -h /
```

The expected VM status is:

```text
qmpstatus: running
```

### Remaining Risks

- Thin-pool utilization remains relatively high.
- Additional Windows updates and VM activity may consume more allocated storage.
- Multiple virtual machines share the same physical NVMe drive.
