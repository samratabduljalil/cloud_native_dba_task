# Linux Storage Management 


---

## Prerequisites

Before starting, add a new virtual disk to your VM:
- **VirtualBox/VMware**: VM Settings → Storage → Add new virtual hard disk
- It will appear as `/dev/nvme0n2` after reboot
- You need `sudo` or root access

---



## Mount a Disk Without LVM
> Simplest scenario. Raw disk → partition → format → mount → persist. Start here.

```bash
# Step 1: Check existing disks
# Look for /dev/sda (OS disk) and /dev/nvme0n2 (your new empty disk)
sudo lsblk

# Step 2: Partition the new disk
# Inside fdisk type: n → p → 1 → Enter → Enter → w
# n = new partition, p = primary, 1 = partition number,
# Enter = default first sector, Enter = default last sector, w = write & exit
sudo fdisk /dev/nvme0n1

# Step 3: Verify the partition was created
# You should now see nvme0n2p1 listed under /dev/nvme0n2
sudo lsblk

# Step 4: Format the partition with ext4
# Creates the filesystem — like "installing a filing cabinet" on the disk
mkfs.xfs /dev/nvme0n2p1

# Step 5: Create a mount point directory
# An empty directory where the disk will be attached
sudo mkdir /mnt/test_disk

# Step 6: Mount the partition
# Attaches /dev/nvme0n2p1 to /mnt/practice_disk
sudo mount /dev/nvme0n2p1 /mnt/test_disk

# Step 7: Verify the mount worked
# Should show the disk size and ~0% used
df -h /mnt/test_disk

# Step 8: Test it — create a file
# If no error, you are successfully writing to the new disk
sudo touch /mnt/test_disk/hello.txt
ls /mnt/test_disk

# Step 9: Get the UUID for permanent mounting
# Copy the UUID= value shown — you need it in the next step
sudo blkid /dev/nvme0n2p1

# Step 10: Add to /etc/fstab for permanent mount
# In vim: press i to insert, add the line, press Esc, then type :wq and Enter
sudo vim /etc/fstab
# ADD THIS LINE at the bottom (replace UUID with your actual value):
# UUID=YOUR-UUID-HERE  /mnt/test_disk  xfs  defaults  0  0

# Step 11: Test the fstab entry
# If no error appears, your fstab entry is correct
sudo mount -a

# Step 12: Final verification
# Confirms disk is mounted and shows size
df -h /mnt/test_disk
```



---

## Create a New LVM Setup from Scratch
> LVM flow: Disk → Partition → Physical Volume → Volume Group → Logical Volume → Filesystem → Mount

```bash
# Step 1: Confirm your new disk is available
# Look for /dev/nvme0n2 — should show no partitions yet
sudo lsblk

# Step 2: Partition the disk with fdisk
# Same as Lab 1: n → p → 1 → Enter → Enter → w
sudo fdisk /dev/nvme0n2

# Step 3: Create a Physical Volume (PV)
# Registers nvme0n2p1 with LVM — "this partition is now under LVM control"
# Without this step, you CANNOT create a Volume Group
sudo pvcreate /dev/nvme0n2p2

# Step 4: Verify the PV was created
# Should show nvme0n2p1 listed as a Physical Volume
sudo pvs

# Step 5: Create a Volume Group (VG)
# Creates a storage pool named 'myvg' from nvme0n2p1
# You can name the VG anything you want
sudo vgcreate myvg /dev/nvme0n2p2

# Step 6: Verify the VG was created
# Shows total size, free space, and number of PVs in the group
sudo vgdisplay myvg

# Step 7a: Create a Logical Volume using ALL available space
sudo lvcreate -l 100%FREE -n mylv myvg

# Step 7b (alternative): Create a fixed-size Logical Volume
# sudo lvcreate -L 5G -n mylv myvg

# Step 8: Verify the LV was created
# Should show 'mylv' inside 'myvg' with its size
sudo lvs

# Step 9: Format the LV with ext4
# Same as formatting a regular partition — creates the filesystem
sudo mkfs.xfs /dev/myvg/mylv

# Step 10: Create a mount point
sudo mkdir -p /mnt/lvm_data

# Step 11: Mount the logical volume
sudo mount /dev/myvg/mylv /mnt/lvm_data

# Step 12: Verify the mount
df -h /mnt/lvm_data

# Step 13: Test it — write a file
sudo touch /mnt/lvm_data/lvm_test.txt
ls /mnt/lvm_data

# Step 14: Add to /etc/fstab for permanent mount
sudo vim /etc/fstab
# ADD THIS LINE at the bottom:
# /dev/myvg/mylv  /mnt/lvm_data  ext4  defaults  0  0

# Step 15: Test fstab and final verify
sudo mount -a
df -h /mnt/lvm_data
```

---

## Add a New Disk to Existing LVM & Expand Storage
> Add a second disk to an existing VG, grow the LV, and expand the filesystem — no downtime.

> **Note:** Requires Lab 2 to be completed. Add a second new disk `/dev/sdc` to your VM first.

```bash
# Step 1: Check current disk layout
# Confirm /dev/nvme0n2 (existing, from Lab 2) and /dev/sdc (new empty disk)
sudo lsblk

# Step 2: Check current disk usage — note the size BEFORE expanding
df -h /mnt/lvm_data

# Step 3: Partition the new disk
# Same as always: n → p → 1 → Enter → Enter → w → creates /dev/nvme0n3p1
sudo fdisk /dev/nvme0n3

# Step 4: Verify new partition created
sudo lsblk

# Step 5: Create a Physical Volume on the new disk
# Same first step as before — register with LVM
sudo pvcreate /dev/nvme0n3p1

# Step 6: Verify both PVs are now listed
# Should show both nvme0n2p1 and /dev/nvme0n3p1
sudo pvs

# Step 7: Add the new PV to your existing Volume Group
# KEY COMMAND — adds /dev/nvme0n3p1's space to the existing 'myvg' pool
sudo vgextend myvg /dev/nvme0n3p1

# Step 8: Verify the VG is now larger
# 'VG Size' and 'Free PE' should be bigger — showing combined space from both disks
sudo vgdisplay myvg

# Step 9: Check current LV size before extending
sudo lvdisplay /dev/myvg/mylv

# Step 10: Extend the LV to use ALL new free space
# The '+' means "add this much on top of current size"
sudo lvextend -l +100%FREE /dev/myvg/mylv

# Alternative — extend by a specific amount:
# sudo lvextend -L +5G /dev/myvg/mylv

# Step 11: Verify LV is now larger
sudo lvs

# Step 12: CRITICAL — Resize the filesystem to use new space
# The LV grew, but ext4 still thinks it's the OLD size.
# This tells the filesystem to expand and fill the new LV size.
# WITHOUT this step, df -h will still show the OLD size!
#sudo resize2fs /dev/myvg/mylv #for ext4
sudo xfs_growfs /dev/myvg/mylv

# Step 13: Verify the filesystem grew — compare with Step 2
# Available space should now be larger. This is your proof it worked!
df -h /mnt/lvm_data
```


---

##  Common Issues You can face

```bash
# Problem: "mount: nvme0n2p1 already mounted"
sudo umount /mnt/practice_disk
sudo mount nvme0n2p1 /mnt/practice_disk

# Problem: "mount -a fails" after editing fstab
# Fix the bad line and test again — a bad fstab can prevent booting!
sudo vim /etc/fstab
sudo mount -a

# Problem: "resize2fs says Nothing to do"
# The LV wasn't extended yet. Run lvextend first:
sudo lvextend -l +100%FREE /dev/myvg/mylv
sudo resize2fs /dev/myvg/mylv

# Problem: "pvcreate: Device nvme0n2p1 not found"
# Partition wasn't saved. Redo fdisk and press 'w' to write:
sudo fdisk /dev/nvme0n2
sudo lsblk

# Problem: df -h shows no change after resize2fs
# Make sure you used the correct device path:
sudo lvs
sudo resize2fs /dev/myvg/mylv

# Problem: forgot to unmount before removing disk
sudo umount /mnt/lvm_data
# If busy:
sudo fuser -m /mnt/lvm_data     # See which process is using it
sudo fuser -km /mnt/lvm_data    # Force kill processes using it
sudo umount /mnt/lvm_data
```

---

