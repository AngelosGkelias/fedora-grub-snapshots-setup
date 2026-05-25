# End to End Fedora Btrfs Snapshot and Rollback Guide (Root & Home)

This guide covers the complete pipeline for configuring snapshots and system rollbacks on Fedora's Btrfs filesystem, covering both the `/` (root) and `/home` subvolumes. It includes installing Snapper, setting up automated GRUB menu integration via `grub-btrfs`, verifying functionality, and executing a permanent system and home rollback using the **Subvolume Swap Method** required by Fedora's specific architecture.

---

## Phase 1: Install and Configure Snapper

Because Btrfs treats `/` (root) and `/home` as independent top-level subvolumes on Fedora, they must be configured and snapshotted separately.

### 1. Install Snapper

```bash
sudo dnf install snapper
```

### 2. Generate the Config for Root Directory (`/`)

```bash
sudo snapper -c root create-config /
```

This automatically generates the configuration file at `/etc/snapper/configs/root` and creates a nested subvolume at `/.snapshots`.

### 3. Generate the Config for Home Directory (`/home`)

```bash
sudo snapper -c home create-config /home
```

This automatically generates the configuration file at `/etc/snapper/configs/home` and creates a nested subvolume at `/home/.snapshots`.

> [!NOTE]
> Since the `/home` subvolume contains user-specific documents, configurations, and personal files, separating its snapshots from the system root (`/`) allows you to restore your OS configuration without losing recent personal data, or vice versa.

---

## Phase 2: Install and Configure grub-btrfs

Because `grub-btrfs` is not in the official Fedora repositories, use a dedicated COPR repository to handle installation and automatic updates. 

### 1. Enable the COPR Repository and Install Components

```bash
sudo dnf copr enable -y kylegospo/grub-btrfs
sudo dnf install -y grub-btrfs inotify-tools
```

### 2. Verify and Trigger an Initial GRUB Update

Take manual baseline snapshots for both the root and home configurations, then update your GRUB configuration to integrate them:

```bash
# Take a manual snapshot of root
sudo snapper -c root create single -d "Baseline Root Setup"

# Take a manual snapshot of home
sudo snapper -c home create single -d "Baseline Home Setup"

# Regenerate your GRUB configuration
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

### 3. Enable the Automated Monitoring Daemon

Enable and start the `grub-btrfsd` background service. This daemon watches your `/.snapshots` directory using inotify and updates the GRUB menu instantly whenever a root snapshot is created or destroyed:

```bash
sudo systemctl enable --now grub-btrfsd
```

> [!TIP]
> The GRUB menu only displays bootable snapshots from the system root (`/`). `/home` snapshots do not appear in GRUB, as they are not bootable OS environments.

---

## Phase 3: Enable Automated Timeline Snapshots & DNF Integration

To ensure your system is continuously protected without manual intervention, you can enable hourly timeline snapshots and install a DNF plugin that automatically snapshots before and after package management actions.

### 1. Enable Timeline Snapshots (Hourly)

Snapper uses systemd timers to handle scheduled snapshots and cleanup tasks automatically. 

#### Step 1: Verify Timeline is Enabled in Configuration
In the Snapper configuration files (located at `/etc/snapper/configs/root` and `/etc/snapper/configs/home`), the timeline snapshot flag is enabled by default (`TIMELINE_CREATE="yes"`). You can verify or explicitly set this using the `snapper` command:

```bash
# For Root config
sudo snapper -c root set-config TIMELINE_CREATE="yes"

# For Home config
sudo snapper -c home set-config TIMELINE_CREATE="yes"
```

#### Step 2: Configure Retention Limits
By default, Snapper retains a large number of snapshots (e.g., 10 hourly, 10 daily, 10 weekly, 10 monthly, 10 yearly), which can consume significant Btrfs metadata and disk space. It is highly recommended to adjust these limits to more sensible defaults:

```bash
# Set sensible retention limits for Root
sudo snapper -c root set-config TIMELINE_LIMIT_HOURLY="5" TIMELINE_LIMIT_DAILY="7" TIMELINE_LIMIT_WEEKLY="4" TIMELINE_LIMIT_MONTHLY="12" TIMELINE_LIMIT_YEARLY="0"

# Set sensible retention limits for Home
sudo snapper -c home set-config TIMELINE_LIMIT_HOURLY="5" TIMELINE_LIMIT_DAILY="7" TIMELINE_LIMIT_WEEKLY="4" TIMELINE_LIMIT_MONTHLY="12" TIMELINE_LIMIT_YEARLY="0"
```

#### Step 3: Enable and Start Systemd Timers
Enable and start the Snapper timeline daemon and the cleanup daemon. The timeline service creates the snapshots hourly, and the cleanup service runs daily to purge snapshots exceeding your retention limits:

```bash
# Enable the hourly timeline snapshot timer
sudo systemctl enable --now snapper-timeline.timer

# Enable the daily cleanup timer to enforce retention limits
sudo systemctl enable --now snapper-cleanup.timer
```

---

### 2. Configure DNF Snapper Integration (Pre/Post Transaction Snapshots)

Fedora has a DNF plugin that automatically creates a **Pre** snapshot before any DNF transaction (install, upgrade, remove) begins, and a **Post** snapshot after it finishes. If a package update breaks your GUI or system stability, you can roll back to the exact second before the transaction began.

#### Step 1: Install the DNF Snapper Plugin

Install the package `python3-dnf-plugin-snapper`:

```bash
sudo dnf install -y python3-dnf-plugin-snapper
```

That's it! The plugin works immediately out of the box. It detects the default `root` config and will automatically snapshot the `/` subvolume on every subsequent DNF command. 

> [!NOTE]
> The DNF plugin only runs on the `root` Snapper config since system packages are installed directly to the `/` partition. It does not snapshot the `home` config.

---

## Phase 4: The System & Home Rollback Protocol

When booting into a root snapshot from the GRUB menu, Fedora loads it in a restricted, read-only environment. Because Fedora hardcodes `subvol=root` (and `subvol=home`) into its kernel arguments and filesystem table (`/etc/fstab`), the native `snapper rollback` command will fail with an I/O error.

To bypass this restriction and make a rollback permanent, you must manually swap the subvolumes. This method can be applied to `root`, `home`, or both simultaneously.

### Prerequisites Checklist

1. Restart your machine, enter the GRUB menu, navigate to **"Submenus of Btrfs Snapshots"**, and boot into your target root snapshot. Use `Esc` to enter the GRUB menu during boot (or type `normal` in the GRUB CLI if you ESC past it).
2. Identify your primary drive partition layout using `findmnt /`. In the steps below, this target partition is generically represented as `/dev/sdXN` (e.g., `/dev/nvme0n1p3`, `/dev/sda3`).
3. Identify your running root snapshot index number using `mount | grep 'on / '`. In the steps below, this target index is generically represented as `TARGET_NUM_ROOT`.
4. Identify your target `/home` snapshot index number by listing home snapshots:
   ```bash
   sudo snapper -c home list
   ```
   In the steps below, this target index is generically represented as `TARGET_NUM_HOME`.

### Step-by-Step Subvolume Swap Execution

#### 1. Open Top-Level Access to the Drive

Mount the raw top-level directory of your Btrfs partition (`subvol=/`) directly to the pre-existing `/mnt` folder:

```bash
sudo mount -o subvol=/ /dev/sdXN /mnt
cd /mnt
```

#### 2. Displace the Broken/Old Subvolumes

Rename the subvolumes you wish to replace. You can choose to displace only `root`, only `home`, or both:

```bash
# To roll back Root:
sudo mv root root_broken

# To roll back Home:
sudo mv home home_broken
```

#### 3. Clone the Target Snapshots Into Place

Create a fresh, writeable clone of your working snapshots and position them exactly where Fedora expects them (`root` and `home`). Because Btrfs uses Copy-on-Write metadata linking, this layout manipulation is instantaneous and consumes no additional storage space:

```bash
# To restore Root:
sudo btrfs subvolume snapshot root_broken/.snapshots/TARGET_NUM_ROOT/snapshot root

# To restore Home:
sudo btrfs subvolume snapshot home_broken/.snapshots/TARGET_NUM_HOME/snapshot home
```

> [!IMPORTANT]
> Because `.snapshots` is a nested subvolume located inside `/root` and `/home` respectively, they will now be under `/mnt/root_broken/.snapshots` and `/mnt/home_broken/.snapshots` after the rename step. This is why we source the snapshots from the `_broken` paths.

#### 4. Safely Unmount and Reboot

Exit the mount folder, unmount the drive access layer, and restart your computer:

```bash
cd ~
sudo umount /mnt
sudo reboot
```

---

## Phase 5: Final Verification and Housekeeping

Allow your computer to boot normally. Do not open the GRUB snapshot sub-menus this time; let it load the default option. 

### 1. Verify Active Subvolumes

Once at your desktop, open a terminal and verify that your active subvolume mounts have successfully returned to the main root and home targets:

```bash
# Verify Root
mount | grep 'on / '

# Verify Home
mount | grep 'on /home'
```

The output should explicitly state `subvol=/root` (or `subvol=root`) and `subvol=/home` (or `subvol=home`).

### 2. Permanently Reclaim Disk Space

Once you are fully satisfied that your system and user directories are completely functional and successfully restored, mount the top-level partition again to permanently eliminate the old broken subvolumes:

```bash
sudo mount -o subvol=/ /dev/sdXN /mnt

# Delete root_broken if you rolled back root
sudo btrfs subvolume delete /mnt/root_broken

# Delete home_broken if you rolled back home
sudo btrfs subvolume delete /mnt/home_broken

sudo umount /mnt
```