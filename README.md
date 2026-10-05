# Linux LVM Storage Expansion -- WSL & AWS EC2

A hands-on Linux storage project demonstrating **LVM (Logical Volume
Manager)**, storage expansion, filesystem growth, and adding additional
disks to an existing Volume Group.

This project was completed in two environments:

-   Local WSL2 using loopback disk images
-   AWS EC2 using EBS volumes

The AWS lab demonstrates how to expand a mounted Linux filesystem online
without unmounting or rebooting the server.

------------------------------------------------------------------------

## LVM Architecture

``` text
Disk / EBS Volume
       ↓
Physical Volume (PV)
       ↓
Volume Group (VG)
       ↓
Logical Volume (LV)
       ↓
Filesystem
       ↓
Mount Point
```

The basic LVM flow is:

**Disk → PV → VG → LV → Filesystem → Mount Point**

------------------------------------------------------------------------

# Lab 1 -- Local WSL2

## Environment

-   Ubuntu WSL2
-   LVM2
-   Loopback disk images
-   ext4 filesystem

Separate disk image files were used so the existing WSL system disks
were not modified.

## LVM Setup

Created a 2 GB loopback disk:

``` bash
dd if=/dev/zero of=lvm-disk.img bs=1M count=2048
sudo losetup -fP lvm-disk.img
lsblk
```

The disk appeared as:

``` text
/dev/loop0
```

Created the Physical Volume:

``` bash
sudo pvcreate /dev/loop0
```

Created the Volume Group:

``` bash
sudo vgcreate devops-vg /dev/loop0
```

Created a 1 GB Logical Volume:

``` bash
sudo lvcreate -L 1G -n app-data devops-vg
```

Created the ext4 filesystem:

``` bash
sudo mkfs.ext4 /dev/devops-vg/app-data
```

Mounted it:

``` bash
sudo mkdir -p /mnt/devops-data
sudo mount /dev/devops-vg/app-data /mnt/devops-data
```

Verified storage:

``` bash
df -h /mnt/devops-data
```

## Expand the Local LVM Storage

Extended the Logical Volume:

``` bash
sudo lvextend -L +500M /dev/devops-vg/app-data
```

Then expanded the ext4 filesystem:

``` bash
sudo resize2fs /dev/devops-vg/app-data
```

A second 2 GB loopback disk was then created and added to the existing
Volume Group:

``` bash
sudo pvcreate /dev/loop1
sudo vgextend devops-vg /dev/loop1
```

This demonstrated how additional storage can be added to an existing LVM
storage pool.

------------------------------------------------------------------------

# Lab 2 -- AWS EC2 + EBS + LVM

## Environment

-   AWS EC2
-   Region: `ap-south-1` (Mumbai)
-   t3.micro
-   Amazon Linux
-   AWS EBS gp3
-   LVM2
-   ext4

The EC2 root filesystem was XFS.

The practice LVM filesystem mounted at `/data` was created as ext4 so
that filesystem expansion could also be demonstrated.

------------------------------------------------------------------------

## Step 1 -- Identify the EBS Disk

Checked the EC2 storage:

``` bash
lsblk
```

The first EBS volume appeared inside Linux as:

``` text
/dev/nvme1n1
```

AWS may show an attachment such as `/dev/sdf`, while Linux on
Nitro-based EC2 instances commonly exposes the disk as an NVMe device.

------------------------------------------------------------------------

## Step 2 -- Install LVM

``` bash
sudo dnf install lvm2 -y
```

------------------------------------------------------------------------

## Step 3 -- Create the Physical Volume

``` bash
sudo pvcreate /dev/nvme1n1
```

Verify:

``` bash
sudo pvs
```

------------------------------------------------------------------------

## Step 4 -- Create the Volume Group

``` bash
sudo vgcreate aws-data-vg /dev/nvme1n1
```

Verify:

``` bash
sudo vgs
```

------------------------------------------------------------------------

## Step 5 -- Create the Logical Volume

``` bash
sudo lvcreate -L 1G -n app-data aws-data-vg
```

Verify:

``` bash
sudo lvs
```

------------------------------------------------------------------------

## Step 6 -- Create the ext4 Filesystem

``` bash
sudo mkfs.ext4 /dev/aws-data-vg/app-data
```

------------------------------------------------------------------------

## Step 7 -- Mount the Filesystem

``` bash
sudo mkdir -p /data
sudo mount /dev/aws-data-vg/app-data /data
```

Verify:

``` bash
df -h /data
```

Created test application data:

``` bash
echo "Application data before disk expansion" | sudo tee /data/app-test.txt
```

------------------------------------------------------------------------

# Online Storage Expansion

The Logical Volume and filesystem were expanded while the filesystem
remained mounted.

Extended the LV:

``` bash
sudo lvextend -L +500M /dev/aws-data-vg/app-data
```

Then grew the ext4 filesystem:

``` bash
sudo resize2fs /dev/aws-data-vg/app-data
```

No unmount or reboot was required.

------------------------------------------------------------------------

# Add a Second EBS Volume

A second 2 GB EBS volume was attached to the EC2 instance.

AWS attachment:

``` text
/dev/sdg
```

Linux device:

``` text
/dev/nvme2n1
```

Verified with:

``` bash
lsblk
```

Created a Physical Volume:

``` bash
sudo pvcreate /dev/nvme2n1
```

Added it to the existing Volume Group:

``` bash
sudo vgextend aws-data-vg /dev/nvme2n1
```

Verified:

``` bash
sudo vgs
```

The Volume Group now contained two Physical Volumes:

``` text
/dev/nvme1n1
/dev/nvme2n1
```

------------------------------------------------------------------------

# Expand the Existing Logical Volume

The existing `app-data` LV was expanded by another 1 GB:

``` bash
sudo lvextend -L +1G /dev/aws-data-vg/app-data
```

Then the ext4 filesystem was expanded:

``` bash
sudo resize2fs /dev/aws-data-vg/app-data
```

Final verification:

``` bash
df -h /data
```

Result:

``` text
Filesystem                           Size  Used Avail Use% Mounted on
/dev/mapper/aws--data--vg-app--data  3.0G   28K  2.8G   1% /data
```

The mounted filesystem successfully grew to approximately **3 GB without
unmounting or rebooting the server**.

------------------------------------------------------------------------

# Final AWS Architecture

``` text
EBS Volume 1 (2 GB) → PV ──┐
                           ├──→ aws-data-vg → app-data LV → ext4 → /data
EBS Volume 2 (2 GB) → PV ──┘
```

## Final Storage

``` text
EBS Volume 1       2 GB
EBS Volume 2       2 GB
-------------------------
Volume Group       ~4 GB

Logical Volume     ~3 GB
Filesystem         ~3 GB
VG Free Space      ~1 GB
Mount Point        /data
```

------------------------------------------------------------------------

# Important LVM Commands

## Check filesystem usage

``` bash
df -h
```

## Check disks

``` bash
lsblk
```

## Check Physical Volumes

``` bash
sudo pvs
```

## Check Volume Groups

``` bash
sudo vgs
```

## Check Logical Volumes

``` bash
sudo lvs
```

## Create a Physical Volume

``` bash
sudo pvcreate /dev/<device>
```

## Create a Volume Group

``` bash
sudo vgcreate <vg-name> /dev/<device>
```

## Add a disk to an existing VG

``` bash
sudo vgextend <vg-name> /dev/<device>
```

## Create a Logical Volume

``` bash
sudo lvcreate -L <size> -n <lv-name> <vg-name>
```

## Extend a Logical Volume

``` bash
sudo lvextend -L +<size> /dev/<vg-name>/<lv-name>
```

## Grow an ext4 filesystem

``` bash
sudo resize2fs /dev/<vg-name>/<lv-name>
```

## Grow an XFS filesystem

``` bash
sudo xfs_growfs <mount-point>
```

------------------------------------------------------------------------

# ext4 vs XFS

Both are common Linux filesystems, but their growth commands are
different.

### ext4

``` bash
sudo resize2fs /dev/<vg>/<lv>
```

### XFS

``` bash
sudo xfs_growfs <mount-point>
```

Important differences:

-   ext4 can grow online and can be shrunk offline.
-   XFS can grow online but cannot be shrunk.
-   Always identify the filesystem before choosing the expansion
    command.

------------------------------------------------------------------------

# Production Troubleshooting Approach

If a Linux server reports that a filesystem is full, first check:

``` bash
df -h
df -i
lsblk
sudo pvs
sudo vgs
sudo lvs
```

Typical LVM troubleshooting flow:

1.  Check filesystem usage.
2.  Check the disk layout.
3.  Check whether the Volume Group has free space.
4.  If VG free space exists, extend the Logical Volume.
5.  Grow the filesystem.
6.  If the VG has no free space, attach additional storage.
7.  Identify the new Linux block device.
8.  Create a Physical Volume.
9.  Add the PV to the existing VG.
10. Extend the Logical Volume.
11. Grow the filesystem.
12. Verify with `df -h`.

A "full disk" problem is not always caused by capacity. Inode exhaustion
or deleted files still held open by running processes can also cause
storage issues.

------------------------------------------------------------------------

# Key Learning

The most important concept from this project is:

``` text
Disk → PV → VG → LV → Filesystem → Mount Point
```

Increasing the Logical Volume size and increasing the filesystem size
are **two separate operations**.

For example:

``` bash
lvextend
```

increases the Logical Volume size.

Then:

``` bash
resize2fs
```

grows the ext4 filesystem so it can use the additional space.

Understanding this distinction is important when troubleshooting Linux
storage in production environments.

------------------------------------------------------------------------

# What I Practiced

-   Linux disk and filesystem inspection
-   LVM architecture
-   Physical Volumes
-   Volume Groups
-   Logical Volumes
-   Logical Volume expansion
-   Adding additional disks to an existing Volume Group
-   ext4 filesystem expansion
-   XFS filesystem growth
-   AWS EC2 storage management
-   AWS EBS
-   NVMe device mapping
-   Online filesystem expansion
-   Linux storage troubleshooting
-   AWS CLI
-   EC2 Instance Connect
-   AWS resource cleanup

------------------------------------------------------------------------

# Technologies

-   Linux
-   Ubuntu WSL2
-   Amazon Linux
-   AWS EC2
-   AWS EBS
-   LVM2
-   ext4
-   XFS
-   Bash
-   AWS CLI

------------------------------------------------------------------------

# AWS Cost Safety

The AWS practice resources were deleted after completing the lab to
avoid unnecessary ongoing charges.

Final cleanup confirmed:

``` text
Running Instances: 0
EBS Volumes:       0
Snapshots:         0
Elastic IPs:       0
Load Balancers:    0
Auto Scaling:      0
```

------------------------------------------------------------------------

# Project Structure

``` text
aws-linux-lvm-storage-expansion/
├── README.md
├── local-wsl-lvm/
├── aws-lvm/
└── screenshots/
```

Screenshots from the labs can be added to the `screenshots/` directory
as the project documentation is completed.

------------------------------------------------------------------------

# Author

**Sathya**

Linux \| AWS \| Cloud Support \| DevOps

GitHub: [PillaiSathya](https://github.com/PillaiSathya)
