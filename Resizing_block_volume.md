# Resizing the Block Volume (Mounted Block Volume)

Before you begin:
- Increase the Block Volume size from the OCI Console first (or the cloud provider UI). Only after the volume size has been increased on the provider side should you resize partitions and filesystems on the instance.
- Back up important data or take a snapshot of the volume before performing resize operations.

## 1) Check current filesystem and mount
Confirm the filesystem type and usage for the mount point:
```bash
df -T /data/Prod
```

## 2) Install required tools (if not already installed)
growpart is provided by cloud-guest-utils on Ubuntu:
```bash
sudo apt update
sudo apt install -y cloud-guest-utils
```

## 3) Expand the partition
Use growpart to expand the partition on the device (example: /dev/sdb partition 1):
```bash
sudo growpart /dev/sdb 1
```

If growpart reports errors or the device size did not update, force a rescan of the block device:
```bash
echo 1 | sudo tee /sys/class/block/sdb/device/rescan
# then retry
sudo growpart /dev/sdb 1
```

If you don't use partition tables (you formatted the raw device), you may skip growpart and resize the filesystem directly against the device that was expanded (e.g., /dev/sdb).

## 4) Resize the filesystem
Choose the command matching your filesystem type.

- For ext4:
```bash
sudo resize2fs /dev/sdb1
```

- For XFS (grow by mount point):
```bash
sudo xfs_growfs /data/Prod
```

Note: For XFS, the filesystem must be mounted. For ext4, resize2fs can typically run while mounted for online resizing.

## 5) Verify the new size
Check that the filesystem reflects the new size:
```bash
df -h /data/Prod
```

## 6) Troubleshooting & notes
- If the expanded size is not visible in the OS, re-check that the Block Volume expansion completed successfully in the OCI Console and that the instance sees the new size with:
  ```bash
  lsblk
  sudo fdisk -l /dev/sdb
  ```
- If you used LVM on top of the block device, extend the PV/LV before resizing the filesystem:
  - pvresize /dev/sdb1
  - lvextend -r /dev/mapper/<vg-name>-<lv-name>  (or use lvextend then resize filesystem)
- Always ensure you operate on the correct device names; running mkfs or destructive commands on an existing device will erase data.
