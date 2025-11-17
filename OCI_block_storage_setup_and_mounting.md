# Create Block Storage in Oracle Cloud (OCI) and Mount on Ubuntu

This guide covers:
- Creating a Block Volume in OCI
- Attaching the Block Volume to an instance
- Partitioning, formatting, mounting and persisting mounts with /etc/fstab
- Exporting the mounted volume via NFS and mounting it on client servers
- Resizing a mounted Block Volume (BV)

Replace placeholders (availability domain, compartment, device names, private IPs, CIDRs, mount paths, etc.) with values from your environment.

---

## OCI: Create a Block Volume

1. Navigate to the Block Volume section:
   - Console: Storage → Block Volumes → Create Block Volume

2. Create a new Block Volume:
   - Provide a Name for the volume.
   - Compartment: select `ptpl` (root) or your target compartment.
   - Availability Domain: select appropriate AD for your tenancy region.
   - Configure Volume Size and Performance:
     - Choose Custom Configuration and specify required volume size.
     - Enable Target Volume Performance.
     - Set Default VCPUs/GB and Maximum VPUs/GB as needed.
     - Turn on Detached Volume Auto-Tune (optional).
   - Keep other settings default and click Create Block Volume.

3. After creation, attach the Block Volume to a running instance:
   - In the Block Volume resource page, go to Resources → Attached Instances → Attach to Instance.
   - Choose the running instance and attach.

---

## Mount the attached Block Volume to the server and filesystem

Important: If the volume contains existing data, do NOT run partitioning & formatting commands (fdisk/mkfs) or you will erase data. Only run fdisk/mkfs on new (empty) volumes.

1. SSH into the instance where the BV is attached.
2. Identify the device name:
   - cd to root or just run:
     ```bash
     lsblk
     ```
     Look for the newly attached device (example: `/dev/sdb`, `/dev/sdc`, etc.).
3. If this is a **new** volume (no existing partition or filesystem), create a partition and format:
   ```bash
   # create a partition interactively or use parted; example using fdisk:
   sudo fdisk /dev/sdb
   # inside fdisk: create a new partition (n), write (w)
   ```
   Then format the partition (example ext4):
   ```bash
   sudo mkfs.ext4 /dev/sdb1
   ```
   Note: If the device uses the whole disk without partition table, you may format `/dev/sdb` directly; most common is to create `/dev/sdb1`.
4. Create the mount point and mount:
   ```bash
   sudo mkdir -p /data/Prod
   sudo mount /dev/sdb1 /data/Prod
   df -h
   ```
   Confirm `/data/Prod` is listed and capacity matches the BV.

### Persist the mount across reboots

1. Get the partition UUID:
   ```bash
   sudo blkid
   ```
   Copy the UUID for `/dev/sdb1` (example: `UUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"`).

2. Edit `/etc/fstab` and add a line using the UUID (replace values):
   ```
   UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /data/Prod  ext4  defaults  0  2
   ```
   - Use the exact UUID string (no quotes) or wrap in quotes if you prefer.
   - Ensure mount options and filesystem type match what you formatted.

3. Reload mounts (test fstab):
   ```bash
   sudo mount -a
   df -h /data/Prod
   ```
   If `mount -a` fails, check `/etc/fstab` for syntax errors before rebooting.

---

## Mount the Block Volume as an NFS server and share with client servers

Prerequisite: Create a separate VM to act as the NFS server (Ubuntu) and attach the BV to it.

1. Create the NFS server VM and install required packages:
   ```bash
   sudo apt update
   sudo apt install -y nfs-server nfs-kernel-server nfs-common rpcbind rpcbind.socket
   ```
   Note package names may vary slightly by distribution; `nfs-server` is usually covered by `nfs-kernel-server`.

2. Ensure OCI Network Security (VCN) and VM-level firewall allow NFS ports:
   - Ports to allow between client and NFS server:
     - 111/tcp (rpcbind)
     - 111/udp
     - 2049/tcp (nfs)
   Configure Security Lists / Network Security Groups and OS-level firewall (ufw/iptables) accordingly.

3. Attach and mount the Block Volume on the NFS server using the previous mounting steps, e.g. mount at `/data/Prod`.

4. Export the directory via NFS:
   - Edit `/etc/exports` and add an export entry. Example:
     ```
     /data/Prod 10.0.1.0/24(rw,sync,no_root_squash,no_subtree_check)
     ```
     - `/data/Prod` — the mount path on NFS server
     - `10.0.1.0/24` — client subnet CIDR that may mount the share
     - Export options: `rw,sync,no_root_squash,no_subtree_check` (adjust to your security posture)

5. Apply exports:
   ```bash
   sudo exportfs -v
   sudo systemctl restart nfs-kernel-server
   sudo systemctl status nfs-kernel-server
   ```

6. On a client server, create the mount point and mount the NFS share:
   ```bash
   sudo mkdir -p /data/Prod
   sudo mount -t nfs <NFS-server-private-ip>:/data/Prod /data/Prod
   df -h /data/Prod
   ```
   To persist on client reboot, add an entry to the client `/etc/fstab`:
   ```
   <NFS-server-private-ip>:/data/Prod  /data/Prod  nfs  defaults,_netdev  0  0
   ```

---

## Resizing a Block Volume (that is mounted)

1. Check current filesystem type:
   ```bash
   df -T /data/Prod
   ```

2. Expand the partition if needed:
   - Use `growpart` from the cloud-utils package to grow the partition:
     ```bash
     sudo apt install -y cloud-guest-utils   # provides growpart on Ubuntu
     sudo growpart /dev/sdb 1
     ```
   - If `growpart` fails, trigger a rescan of the block device:
     ```bash
     echo 1 | sudo tee /sys/class/block/sdb/device/rescan
     ```

3. Resize the filesystem:
   - For ext4:
     ```bash
     sudo resize2fs /dev/sdb1
     ```
   - For xfs (if the filesystem mount point is used, grow using mount point):
     ```bash
     sudo xfs_growfs /data/Prod
     ```

4. Verify:
   ```bash
   df -h /data/Prod
   ```

Notes:
- Ensure you increase the Block Volume size from the OCI console first, then resize the partition+filesystem on the instance.
- Always take backups/snapshots before resizing or performing disk operations.

---

## Troubleshooting tips

- If a device does not appear after attachment, run `sudo lsblk` and `dmesg | tail` to see kernel messages.
- If `mount -a` fails, inspect `/var/log/syslog` or `journalctl -xe` for fstab errors.
- For NFS permission problems, check export rules, `showmount -e` on the server, and NFS server logs.
- If resizing does not reflect, ensure the cloud provider volume expansion completed successfully in the OCI console and the instance sees the new size (use `lsblk` and `fdisk -l`).

---

## Example commands summary

```bash
# Identify block devices
lsblk

# Create partition (interactive)
sudo fdisk /dev/sdb

# Format (ext4 example)
sudo mkfs.ext4 /dev/sdb1

# Mount
sudo mkdir -p /data/Prod
sudo mount /dev/sdb1 /data/Prod

# Persist via fstab (example)
sudo blkid
# add to /etc/fstab:
# UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /data/Prod  ext4  defaults  0  2
sudo mount -a

# NFS server packages (Ubuntu)
sudo apt update
sudo apt install -y nfs-kernel-server nfs-common rpcbind

# Export FS
# add to /etc/exports:
# /data/Prod 10.0.1.0/24(rw,sync,no_root_squash,no_subtree_check)
sudo exportfs -v
sudo systemctl restart nfs-kernel-server

# Client mount
sudo mount -t nfs <NFS-server-private-ip>:/data/Prod /data/Prod

# Resize partition and filesystem
sudo apt install -y cloud-guest-utils
sudo growpart /dev/sdb 1
sudo resize2fs /dev/sdb1       # ext4
# or
sudo xfs_growfs /data/Prod     # xfs
```

---

Replace all example names (device names, CIDRs, mount paths, compartment names) with your actual environment values and always back up important data before performing destructive operations.
