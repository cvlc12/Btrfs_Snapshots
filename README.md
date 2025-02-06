# Automatic Btrfs Snapshots with systemd

This how-to describes how to set up automatic daily Btrfs snapshots for `/` and `/home` on Fedora. Snapshots are managed in a rotating fashion, keeping only the last 10. If you start your system multiple times a day, only one snapshot per day will be created.

## Prerequisites

- Fedora 41 or newer
- Btrfs as the filesystem for `/` and `/home`
- System using `systemd`

## 1. Create the Script

Save the following script as `/usr/local/bin/btrfs_snapshot.sh`:

```bash
#!/bin/bash

# Directories for storing snapshots
ROOT_SNAPSHOT_DIR="/.snapshots"
HOME_SNAPSHOT_DIR="/home/.snapshots"
MAX_SNAPSHOTS=10

# File to track the last snapshot date
TIMESTAMP_FILE="/var/lib/btrfs_snapshot_last_run"
DATE=$(date +"%Y-%m-%d")

# Function to create and rotate snapshots
create_and_rotate_snapshots() {
    local source=$1
    local target_dir=$2
    
    # Create the snapshot
    snapshot_path="${target_dir}/${DATE}"
    echo "Creating snapshot for ${source} at ${snapshot_path}"
    sudo btrfs subvolume snapshot -r "${source}" "${snapshot_path}"
    
    # Rotate old snapshots
    snapshots=($(sudo ls -1 "${target_dir}" | sort -r))
    if [ ${#snapshots[@]} -gt $MAX_SNAPSHOTS ]; then
        for snapshot in "${snapshots[@]:$MAX_SNAPSHOTS}"; do
            echo "Deleting old snapshot: ${target_dir}/${snapshot}"
            sudo btrfs subvolume delete "${target_dir}/${snapshot}"
        done
    fi
}

# Ensure the snapshot directories exist
sudo mkdir -p "${ROOT_SNAPSHOT_DIR}" "${HOME_SNAPSHOT_DIR}"

# Check the last execution date
if [ -f "${TIMESTAMP_FILE}" ]; then
    LAST_RUN_DATE=$(cat "${TIMESTAMP_FILE}")
else
    LAST_RUN_DATE=""
fi

# Create only one snapshot per day
if [ "${LAST_RUN_DATE}" != "${DATE}" ]; then
    create_and_rotate_snapshots "/" "${ROOT_SNAPSHOT_DIR}"
    create_and_rotate_snapshots "/home" "${HOME_SNAPSHOT_DIR}"
    echo "${DATE}" | sudo tee "${TIMESTAMP_FILE}" > /dev/null
    echo "Snapshots completed successfully."
else
    echo "Snapshot already taken today (${DATE}). Skipping."
fi
```

### **Set Permissions**

Run:
```bash
sudo chmod +x /usr/local/bin/btrfs_snapshot.sh
```

## 2. Create the Systemd Service

Create the file `/etc/systemd/system/btrfs-snapshot.service` with the following content:

```ini
[Unit]
Description=Btrfs Snapshot Service

[Service]
ExecStart=/usr/local/bin/btrfs_snapshot.sh
Type=oneshot
```

## 3. Create the Systemd Timer

Create the file `/etc/systemd/system/btrfs-snapshot.timer` with the following content:

```ini
[Unit]
Description=Run Btrfs Snapshot Service on Boot

[Timer]
OnBootSec=2min
Persistent=true

[Install]
WantedBy=timers.target
```

## 4. Enable the Service and Timer

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now btrfs-snapshot.timer
```

## 5. Verification

Check the status of the timer:
```bash
systemctl list-timers --all | grep btrfs-snapshot
```

If necessary, you can manually start the service:
```bash
sudo systemctl start btrfs-snapshot.service
```

View logs:
```bash
journalctl -u btrfs-snapshot.service --no-pager
```

---

Now, automatic daily snapshots are created and managed. 🎉

