# End to End Fedora Btfs Snapshot and Rollback Guide

This guide covers the complete pipeline: installing and configuring Snapper, setting up automated GRUB menu integration via grub-btrfs, verifying functionality, and executing a permanent system rollback using the Subvolume Swap Method required by Fedora's specific architecture.

## Phase 1: Install and Configure Snapper

### 1.Install Snapper

```bash
sudo dnf install snapper
```

### 2. Generate the Config for Root Directory

```bash
sudo snapper -c root create-config /
```

This automatically generates the configuration file at /etc/snapper/configs/root and creates a nested subvolume at /.snapshots.

## Phase 2: Install and Configure grub-btrfs

Because grub-btrfs is not in the official Fedora repositories, use a dedicated COPR repository to handle installation and automatic updates. You can also build from source (what I did).

### 1. Enable the COPR Repository and Install Components

```bash
sudo dnf copr enable -y kylegospo/grub-btrfs
sudo dnf install -y grub-btrfs inotify-tools
```

### 2. Verify and Trigger an Initial GRUB Update

```bash
# Take a manual snapshot
sudo snapper -c root create single -d "Baseline Setup"

# Regenerate your GRUB configuration
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

### 3. Enable the Automated Monitoring Daemon

Enable and start the grub-btrfsd background service. This daemon watches your /.snapshots directory using inotify and updates the GRUB menu instantly whenever a snapshot is created or destroyed:

```bash
sudo systemctl enable --now grub-btrfsd
```

## Phase 3: Enable Automated Timeline Snapshots & DNF Integration

To ensure your system is continuously protected without manual intervention, you can enable hourly timeline snapshots and install a DNF plugin that automatically snapshots before and after package management actions.

### 1. Enable Timeline Snapshots (Hourly)

Snapper uses systemd timers to handle scheduled snapshots and cleanup tasks automatically. 

#### Step 1: Verify Timeline is Enabled in Configuration
In the Snapper configuration file for root (located at `/etc/snapper/configs/root`), the timeline snapshot flag is enabled by default (`TIMELINE_CREATE="yes"`). You can verify or explicitly set this using the `snapper` command:

```bash
sudo snapper -c root set-config TIMELINE_CREATE="yes"
```

#### Step 2: Configure Retention Limits
By default, Snapper retains a large number of snapshots (e.g., 10 hourly, 10 daily, 10 weekly, 10 monthly, 10 yearly), which can consume significant Btrfs metadata and disk space. It is highly recommended to adjust these limits to more sensible defaults:

```bash
sudo snapper -c root set-config TIMELINE_LIMIT_HOURLY="5" TIMELINE_LIMIT_DAILY="7" TIMELINE_LIMIT_WEEKLY="4" TIMELINE_LIMIT_MONTHLY="12" TIMELINE_LIMIT_YEARLY="0"
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

---

## Phase 4: The System Rollback Protocol

When booting into a snapshot from the GRUB menu, Fedora loads it in a restricted environment. Because Fedora hardcodes `subvol=root` into its kernel arguments and filesystem table (`/etc/fstab`), the native `snapper rollback` command will fail with an I/O error.

To bypass this restriction and make a rollback permanent, you must manually swap the subvolumes.

### Prerequisites Checklist

1. Restart your machine, enter the GRUB menu, navigate to "Submenus of Btrfs Snapshots", and boot into your target snapshot. Use `Esc` to enter the GRUB menu during boot (or type `normal` in the GRUB CLI if you ESC past it).
2. Identify your primary drive partition layout using `findmnt /`. In the steps below, this target partition is generically represented as `/dev/sdXN` (e.g., `/dev/nvme0n1p3`, `/dev/sda3`).
3. Identify your running snapshot index number using `mount | grep 'on / '`. In the steps below, this target index is generically represented as `TARGET_NUM`.

### Step-by-Step Subvolume Swap Execution

#### 1. Open Top-Level Access to the Drive

Mount the raw top-level directory of your Btrfs partition (subvol=/) directly to the pre-existing, empty /mnt folder. Even though your active snapshot workspace is read-only, mounting the drive's base layer this way opens it with full read-write privileges:

```bash
sudo mount -o subvol=/ /dev/sdXN /mnt
cd /mnt
```

#### 2. Displace the Broken Root Subvolume

Rename the broken root subvolume to clear the path. Because Btrfs treats subvolumes like standard directory objects at the root layer, this moves your entire broken OS state and its nested snapshots safely aside:

```bash
sudo mv root root_broken
```

#### 3. Clone the Target Snapshot Into Place

Create a fresh, writeable clone of your working snapshot and position it exactly where Fedora expects it (root). Because Btrfs uses Copy-on-Write metadata linking, this layout manipulation is instantaneous and consumes no additional storage space:

```bash
sudo btrfs subvolume snapshot root_broken/.snapshots/TARGET_NUM/snapshot root
```

#### 4. Safely Unmount and Reboot

Exit the mount folder, unmount the drive access layer, and restart your computer:

```bash
cd ~
sudo umount /mnt
sudo reboot
```

## Phase 5: Final Verification and Housekeeping

Allow your computer to boot normally. Do not open the GRUB snapshot sub-menus this time; let it load the default option. Once at your desktop, open a terminal and verify your active subvolume has successfully returned to the main root target:

```bash
mount | grep 'on / '
```

The output should explicitly state `subvol=/root` or `subvol=root`.

Once you are satisfied that your system is fully functional and successfully restored, mount the top-level partition again to permanently eliminate the old broken subvolume:

```bash
sudo mount -o subvol=/ /dev/sdXN /mnt
sudo btrfs subvolume delete /mnt/root_broken
sudo umount /mnt
```