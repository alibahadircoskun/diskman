# disk.py: Disk Health, Format & Progress Monitor

CLI tool for health checking, formatting, and monitoring disks in a multi-slot SAS backplane. Supports health scanning via HDSentinel, low-level formatting with `sg_format`, and a real-time ANSI progress monitor, all via an interactive menu or direct subcommands.

---

## Features

- **Health check**: Scans all connected disks via [HDSentinel](https://www.hdsentinel.com/hard_disk_sentinel_linux.php) and reports status.
- **Interactive formatter**: Selects disks by slot, confirms destructively, and launches parallel `sg_format` jobs (full or fast mode).
- **Live progress monitor**: Full-screen ANSI TUI showing per-disk progress bars, ETA, elapsed time, and status (no curses required).
- **Slot-aware**: Maps physical SAS expander PHY paths to human-readable bay/slot numbers.
- **Persistent logging**: All format start, complete, and failure events are written to `/var/log/diskops.log`.

---

## Requirements

| Dependency | Purpose |
|---|---|
| Python 3.8+ | Runtime |
| [`sg3_utils`](https://sg.danny.cz/sg/sg3_utils.html) | `sg_format`, `sg_requests` commands |
| [`HDSentinel`](https://www.hdsentinel.com/hard_disk_sentinel_linux.php) | Disk health reporting |
| `lsblk` | Disk model/serial enrichment (standard on Linux) |

Install `sg3_utils` on Debian/Ubuntu:

```bash
sudo apt install sg3-utils
```

Place the HDSentinel binary at `/root/HDSentinel` (or update the `HDSENTINEL` path in the script).

---

## Installation

```bash
git clone https://github.com/<your-username>/disk-manager.git
cd disk-manager
sudo bash install.sh
```

The install script will:
- Install `python-is-python3`, `sg3-utils`, `wget`, and `unzip` via apt
- Download and install HDSentinel to `/root/HDSentinel` (skipped if already present)
- Symlink `disk.py` to `/usr/local/bin/disk` so it can be run from anywhere

> Must be run as **root**. Both the install script and the tool itself will exit if run without root privileges.

---

## Configuration

At the top of `disk.py`, update three things to match your hardware:

```python
PORTS = [
    "/dev/disk/by-path/pci-<PCI_ADDR>-sas-exp0x<EXPANDER_WWN>-phy6-lun-0",
    ...
]

SLOTS = [2, 5, 8, 11, 1, 4, 7, 10, 0, 3, 6, 9]
```

- **`PORTS`**: Ordered list of stable `by-path` symlinks for each disk bay. See below for how to discover these on your system.
- **`SLOTS`**: Corresponding physical slot/bay numbers, mapped index-for-index with `PORTS`. Adjust to match your enclosure's labelling.
- **`HDSENTINEL`**: Path to the HDSentinel binary (default: `/root/HDSentinel`).
- **`LOGFILE`**: Path for the operations log (default: `/var/log/diskops.log`).

---

## Why by-path Instead of /dev/sdX?

Linux assigns `/dev/sdX` names dynamically based on the order devices are enumerated. On a live system with hotpluggable disks, this is a problem: every time a disk is removed and reinserted (or any disk in the backplane is swapped), the kernel reassigns names in arrival order. A disk that was `/dev/sdb` before the swap may come back as `/dev/sdf`, and the names of other disks may shift too.

For a tool like HDSentinel that needs to be pointed at specific devices, this means it can end up scanning the wrong disk entirely, or missing one, with no obvious error. The names just quietly no longer match the physical bays.

`/dev/disk/by-path/` symlinks solve this cleanly. They are tied to the **physical port** on the SAS expander (the PHY number) rather than the kernel's enumeration order. As long as a disk is in the same bay, its `by-path` link is always the same, regardless of what `/dev/sdX` name it was assigned that boot.

By passing explicit `by-path` targets to HDSentinel (via `-onlydevs`) rather than letting it scan by name, the tool reliably finds the right disk every time. The `discover()` function in this script resolves each symlink to its current `/dev/sdX` name at runtime, so the rest of the tooling works normally. The stable physical path is always the source of truth.

---

## Finding Your PHY Paths

The `by-path` symlinks are stable device identifiers that encode the physical path through your HBA and SAS expander to each disk bay. They survive reboots and re-enumeration, unlike `/dev/sdX` names which can change.

### Step 1: List all current by-path links

```bash
ls -la /dev/disk/by-path/ | grep -v part
```

For a SAS expander backplane you'll see entries like:

```
pci-0000:02:00.0-sas-exp0x<EXPANDER_WWN>-phy3-lun-0 -> ../../sdb
pci-0000:02:00.0-sas-exp0x<EXPANDER_WWN>-phy4-lun-0 -> ../../sdc
...
```

The components of each path are:

| Component | Meaning |
|---|---|
| `pci-0000:02:00.0` | PCI address of your HBA (host bus adapter) |
| `exp0x<EXPANDER_WWN>` | World Wide Name of your SAS expander (unique per device) |
| `phyN` | PHY number on the expander (corresponds to a physical bay) |
| `lun-0` | LUN (almost always 0 for direct-attached disks) |

### Step 2: Map PHY numbers to physical bays

PHY numbers don't necessarily match bay numbers. The wiring inside the backplane determines the relationship. The most reliable approach is to insert one disk at a time and observe which PHY path appears:

```bash
# Watch for new by-path entries as you insert each disk
watch -n1 'ls /dev/disk/by-path/ | grep exp'
```

Insert a disk into bay 0, note the new PHY entry, eject it, then repeat for each bay. Record the mapping as you go.

If your enclosure supports SES (SCSI Enclosure Services), you can also query slot info directly:

```bash
# List enclosure devices
ls /sys/class/enclosure/

# Show slot info for a specific enclosure
cat /sys/class/enclosure/<device>/*/slot
```

### Step 3: Identify your PCI address and expander WWN

```bash
# Find HBA PCI address
lspci | grep -i 'sas\|scsi\|raid'

# Find expander WWN from existing by-path entries
ls /dev/disk/by-path/ | grep exp | head -1
```

The expander WWN is the hex value after `exp0x` and it's the same across all entries for the same backplane.

### Step 4: Update PORTS and SLOTS in disk.py

Once you have the full bay-to-PHY mapping, update the script:

```python
PORTS = [
    "/dev/disk/by-path/pci-0000:02:00.0-sas-exp0x<YOUR_WWN>-phy3-lun-0",  # Bay 0
    "/dev/disk/by-path/pci-0000:02:00.0-sas-exp0x<YOUR_WWN>-phy4-lun-0",  # Bay 1
    # ... one entry per bay
]

SLOTS = [0, 1, 2, ...]  # Physical bay labels, index-matched to PORTS
```

> **Note:** If a bay is empty, its `by-path` symlink simply won't exist. The `discover()` function skips any path that isn't a block device, so empty bays are silently ignored at runtime.

---

## Usage

### Interactive menu

```bash
sudo ./disk.py
```

Launches a menu with three options: health check, format, and progress monitor.

### Subcommands

```bash
# Disk health scan
sudo ./disk.py health
sudo ./disk.py health --dump          # Full detailed report

# Format disks interactively
sudo ./disk.py format
sudo ./disk.py format --fast          # Use fast format mode (ffmt=1)

# Monitor format progress
sudo ./disk.py progress               # Interactive device selection
sudo ./disk.py progress /dev/sda /dev/sdb   # Monitor specific devices
```

---

## Why a Separate Progress Monitor?

`sg_format` prints progress output to the terminal as it runs. When formatting multiple disks in parallel, all of those processes write to the same terminal session and the output gets interleaved, making it impossible to tell which line belongs to which disk or how far along each one actually is.

The progress monitor solves this by polling each disk independently via `sg_requests --progress` and displaying them together in a structured per-disk view with individual progress bars, elapsed time, and ETA. It can be launched automatically at the end of a format job, or run standalone against any in-progress devices if you need to attach to a job that is already underway.

---

## Progress Monitor Controls

| Key | Action |
|-----|--------|
| `r` | Force refresh / re-poll immediately |
| `p` | Pause / resume polling |
| `+` / `-` | Increase / decrease poll interval (1–60s) |
| `q` / `Esc` / `Ctrl+C` | Quit |

---

## Log Format

Events are appended to `/var/log/diskops.log` in a structured key=value format:

```
FORMAT_STARTED   mode=full slot=3 dev=/dev/sdb model=ST4000NM0023 serial=Z1X0ABCD
FORMAT_COMPLETE  mode=full slot=3 dev=/dev/sdb model=ST4000NM0023 serial=Z1X0ABCD
FORMAT_FAILED    slot=3 dev=/dev/sdb model=ST4000NM0023 serial=Z1X0ABCD
FORMAT_ABORTED   devices: /dev/sdb /dev/sdc
```

---

## Environment

Developed and tested on Ubuntu Server with Dell PowerEdge hardware and a SAS expander backplane. Terminal output is compatible with any SSH session, no GUI or X11 required.

---
