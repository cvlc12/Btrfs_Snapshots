# Automatic Btrfs Snapshots with systemd

    Status: 🚧 Work In Progress

This guide sets up automatic daily Btrfs snapshots for systemd-homed using the Btrfs backend. Snapshots are rotated to keep only the most recent 10. If the system starts multiple times in one day, only one snapshot will be created for that day.

## Prerequisites

- `systemd-homed` with a Btrfs-backed home directory.
- The `user_subvol_rm_allowed` mount option must be set.

## 1. Create the Script

Save the following script as `${HOME}/scripts/btrfs_snapshot.sh`:

```bash
#!/bin/bash

set -euo pipefail

# Configuration
HOME_SNAPSHOT_DIR="${XDG_DATA_HOME:-${HOME}/.local/share}/btrfs-snapshots"
MAX_SNAPSHOTS=10
TIMESTAMP_FILE="${HOME_SNAPSHOT_DIR}/btrfs_snapshot_last_run"
DATE=$(date +"%Y-%m-%d")


# Function to create and rotate snapshots
create_and_rotate_snapshots() {
    local source=$1
    local target_dir=$2
    
    # Create the snapshot
    snapshot_path="${target_dir}/${DATE}"
    echo "Creating snapshot for ${source} at ${snapshot_path}"
    btrfs subvolume snapshot -r "$source" "$snapshot_path"
    
    # Rotate old snapshots
    snapshots=($(ls -1 "${target_dir}" | sort -r))
    if [ ${#snapshots[@]} -gt $MAX_SNAPSHOTS ]; then
        for snapshot in "${snapshots[@]:$MAX_SNAPSHOTS}"; do
            echo "Deleting old snapshot: ${target_dir}/${snapshot}"
            btrfs subvolume delete "${target_dir}/${snapshot}"
        done
    fi
}

# Ensure the snapshot directory exist
mkdir -p "$HOME_SNAPSHOT_DIR"

# Check the last execution date
if [ -f "${TIMESTAMP_FILE}" ]; then
    LAST_RUN_DATE=$(cat "${TIMESTAMP_FILE}")
else
    LAST_RUN_DATE=""
fi

# Create only one snapshot per day
if [ "${LAST_RUN_DATE}" != "${DATE}" ]; then
    create_and_rotate_snapshots "$HOME" "$HOME_SNAPSHOT_DIR"
    echo "${DATE}" | sudo tee "${TIMESTAMP_FILE}" > /dev/null
    echo "Snapshots completed successfully."
else
    echo "Snapshot already taken today (${DATE}). Skipping."
fi
```

### **Set Script Permissions**

Run:
```bash
chmod +x "${HOME}/scripts/btrfs_snapshot.sh"
```

## 2. Create the systemd service

Create the file `.config/systemd/user/btrfs-snapshot.service` with the following content:

```ini
[Unit]
Description=Create a daily Btrfs snapshot of the home directory

[Service]
Type=oneshot
ExecStart=%h/scripts/btrfs_snapshot.sh
```

## 3. Create the systemd Timer

Create the file `.config/systemd/user/btrfs-snapshot.timer` with the following content:

```ini
[Unit]
Description=Daily Btrfs snapshot timer

[Timer]
OnActiveSec=2min
Persistent=true

[Install]
WantedBy=default.target
```

## 4. Enable and Start the Timer

```bash
systemctl --user daemon-reexec
systemctl --user enable --now btrfs-snapshot.timer
```

## 5. Verification

Check if the timer is active:
```bash
systemctl --user list-timers --all | grep btrfs-snapshot
```

If necessary, you can manually start the service:
```bash
systemctl --user start btrfs-snapshot.service
```

View logs:
```bash
journalctl --user -u btrfs-snapshot.service --no-pager
```

---

✅ Now your system will automatically create and rotate daily Btrfs snapshots of your home directory!
