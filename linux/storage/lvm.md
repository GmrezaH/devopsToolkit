# Separate or Extend /var

## Check current `/var` partition

Before proceeding, verify whether `/var` is already on a separate partition.

```bash
df -h /var
```

Additionally, check the LVM configuration:

```bash
lsblk
```

This command provides a visual representation of the disk partitions and their mount points.

## Creating a separate `/var` partition

If `/var` is **not** on a separate partition, follow these steps to create one.

1. Create a new partition

   Use `fdisk` or `gdisk` to create a new partition. Below is an example using `fdisk` (Replace `/dev/sdb` with the appropriate disk identifier as needed):

   ```bash
   fdisk /dev/sdb
   ```

   **Within the `fdisk` prompt:**

   1. **Create a New GPT Partition Table (if not already using GPT):**

      - Type `g` and press `Enter` to create a new empty GPT partition table.

   2. **Create a New Partition:**

      - Type `n` and press `Enter` to create a new partition.
      - Accept the default partition number by pressing `Enter`.
      - Press `Enter` to accept the default first sector.
      - Press `Enter` to accept the default last sector or specify the desired size.

   3. **Set Partition Type to Linux LVM:**

      - Type `t` and press `Enter`.
      - Enter the partition number (e.g., `1`) and press `Enter`.
      - Type `31` and press `Enter` to set the type to "Linux LVM".

   4. **Write Changes and Exit:**
      - Type `w` and press `Enter` to write the changes to the disk and exit `fdisk`.

1. Initialize the partition as a Physical Volume

   Inform LVM that the new partition is a Physical Volume (PV):

   ```bash
   pvcreate /dev/sdb1
   ```

1. Extend the Volume Group

   Add the new PV to an existing Volume Group (VG). First, identify your VG name:

   ```bash
   vgdisplay | grep 'VG Name'
   ```

   Assuming the VG name is `rl`, extend it with the new PV:

   ```bash
   vgextend rl /dev/sdb1
   ```

1. Create a Logical Volume for `/var`

   Create a new Logical Volume (LV) named `var` using the available free space:

   ```bash
   lvcreate -l 100%FREE -n var rl
   ```

   **Explanation:**

   - `-l 100%FREE`: Allocates all available free space in the VG to the new LV.
   - `-n var`: Names the LV as `var`.
   - `rl`: Specifies the Volume Group to use.

1. Create Filesystem and Mount `/var`

   1. Create the Filesystem

      Choose a filesystem type (e.g., XFS or EXT4) and create it on the new LV:

      ```bash
      mkfs.xfs /dev/rl/var
      ```

      _or_

      ```bash
      mkfs.ext4 /dev/rl/var
      ```

   1. Mount the new partition temporarily

      ```bash
      mkdir /mnt/newvar
      mount /dev/rl/var /mnt/newvar
      ```

   1. Copy Existing `/var` Data

      Ensure no processes are writing to `/var` during the copy:

      ```bash
      lsof /var
      ```

      If necessary, stop services that are using `/var`.

      Copy the data using `rsync` to preserve permissions and links:

      ```bash
      rsync -acxHAX --progress /var/* /mnt/newvar/
      ```

   1. Backup and Update `/etc/fstab`

      Backup the existing `fstab`:

      ```bash
      cp /etc/fstab /etc/fstab.bk
      ```

      Add the new entry to `/etc/fstab`:

      ```bash
      sh -c "echo `blkid /dev/rl/var | awk '{print $2}' | sed 's/\"//g'` /var xfs defaults 0 0 >> /etc/fstab"
      systemctl daemon-reload
      ```

   1. Mount the New `/var`

      ```bash
      umount /mnt/newvar
      mount /var
      ```

      Verify the mount:

      ```bash
      df -h /var
      ```

1. Finalize

   Reboot the system to ensure all changes take effect correctly:

   ```bash
   reboot
   ```

## Extending an Existing `/var` Partition

If `/var` is **already** on a separate partition and you need to extend its size, follow these steps.

1. Ensure There is Free Space in the Volume Group

   Check available free space in your Volume Group:

   ```bash
   vgdisplay rl
   ```

   Look for the "Free PE / Size" line to determine available space.

   If there is no free space, you may need to add a new Physical Volume as described in the [Creating a Separate `/var` Partition](#creating-a-separate-var-partition) section.

1. Extend the Logical Volume for `/var`

   Increase the size of the `/var` Logical Volume. For example, to add all available free space:

   ```bash
   lvextend -l +100%FREE /dev/rl/var
   ```

   _or specify the size to extend by:_

   ```bash
   lvextend -L +10G /dev/rl/var
   ```

1. Resize the Filesystem

   After extending the LV, resize the filesystem to utilize the new space.

   - **For XFS Filesystem:**

     ```bash
     xfs_growfs /var
     ```

   - **For EXT4 Filesystem:**

     ```bash
     resize2fs /dev/rl/var
     ```

   Verify the new size:

   ```bash
   df -h /var
   ```
