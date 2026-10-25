# pdpve - Documentation

Last updated: 2026-10-25

> Back to [README](../README.md)

## Prerequisites

- A Proxmox VE host (the tool uses `qm`, `pvesm`, `/etc/pve/qemu-server/`, and the `vmbr0` bridge)
- root privileges (the main menu and the `kernel` module check `EUID` on start)
- Bash, `getopt` (util-linux), `curl`, `git`
- Extra dependencies per module:

| Module | Required commands |
|--------|-------------------|
| `fan` | `ipmitool`, `dmidecode`, an IPMI-capable BMC |
| `cpu` | `cpufreq-set` (cpufrequtils), `sensors` (lm-sensors) |
| `find-ip` | `arp` (net-tools), `ping`; vendor lookup needs access to `api.macvendors.com` |
| `qm` | `nano` (config editing), `ssh` |
| `qm-install` | `ssh-keygen`; network access to the cloud image mirrors |
| speedtest | `speedtest` CLI (not installed by this tool) |

## Installation

### From Source

Run it from the repository root, because the installer copies `./module` from the current directory:

```bash
git clone https://github.com/pardnchiu/pdpve.git
cd pdpve
bash pdpve
```

The first run without arguments:

1. Copies `pdpve` to `/usr/local/sbin/pdpve` and marks it executable
2. Copies `module/` to `/usr/local/bin/pdpve/module/`
3. Lets you run `pdpve` directly afterwards to open the interactive menu

When a different version is already installed, the script shows both versions and asks before overwriting.

### Update

Enter `u` (or `update`) in the main menu: the script clones `https://github.com/pardnchiu/pdpve.git` into `/tmp/pdpve`, overwrites `/usr/local/sbin/pdpve` and `/usr/local/bin/pdpve/module/`, removes the temporary directory, and exits.

### Uninstall

Enter `rm` (or `remove`) in the main menu and confirm to delete `/usr/local/sbin/pdpve` and `/usr/local/bin/pdpve`.

## Configuration

The tool reads no environment variables and uses these fixed paths:

| Path | Purpose |
|------|---------|
| `/usr/local/sbin/pdpve` | Installed main script |
| `/usr/local/bin/pdpve/module/` | Installed modules; module calls prefer `./module/` in the current directory and fall back to this path |
| `~/.ssh/id_rsa`, `~/.ssh/id_rsa.pub` | SSH key injected into new VMs; generated as RSA 4096 when missing |
| `~/.pdpve_pubkey` | Optional; extra public keys merged with `id_rsa.pub` and injected together |
| `/tmp/<image file>` | Cloud image cache; an existing file is reused without downloading again |
| `/etc/pve/qemu-server/<VMID>.conf` | Config files read by the VM list and edit actions |

## Usage

### Basic

```bash
# Open the interactive main menu (requires root)
pdpve

# Print the version
pdpve -v

# Show CPU model, core count, temperature, and current per-core frequency
pdpve -c -i
```

### VM Management

```bash
# Open the VM menu: install, list, edit, start, shutdown, reboot, stop, remove, SSH
pdpve -qm
```

Entering `i` in the menu walks through: VMID (re-prompted if it already exists) -> storage (validated with `pvesm status`) -> OS -> release -> image download confirmation -> name, vCPU, disk, RAM -> IP (`dhcp` or CIDR plus gateway) -> username and password.

The menu's `ssh` action connects as the VM's `ciuser` to `<first three octets of vmbr0>.<VMID>`, so give static IPs a last octet equal to the VMID and set a username during installation.

### Create a VM with Flags

The `qm` menu does not forward arguments to `qm-install`; run the module directly to pre-fill values:

```bash
bash /usr/local/bin/pdpve/module/qm-install \
  --id 200 \
  --storage local-lvm \
  --os debian \
  --name web \
  --cpu 4 \
  --disk 32G \
  --ram 4096
```

Fields not passed as flags (release, IP, gateway, username, password) are still prompted. When `qm create`, `importdisk`, `resize`, or SSH key generation fails, the script removes the VM with `qm destroy --purge` and exits with code `1`.

### CPU Frequency

```bash
# Lock the first 4 cores to performance at 3.0 GHz max / 1.2 GHz min; drop the rest to minimum frequency
pdpve -c -s -c 4 -x 30 -n 12
```

The script appends `00000` to the `-x` / `-n` values and writes them to `scaling_max_freq` / `scaling_min_freq` (kHz), so `30` means 3.0 GHz. Without `-n`, the minimum frequency equals the maximum. CPUs whose model name contains `EPYC` are skipped.

### Fan Speed

```bash
# Detect the brand and switch to auto mode
pdpve -f -s auto

# Pin a Dell server to 40%
pdpve -f -s 40 -b dell
```

Interactive speed input is clamped to 20-80; a value passed with `-s` is sent as is. When an IPMI command fails (Dell, Supermicro, HPE), the script prints an error and exits with code `1`.

### LAN Scan and Kernel Management

```bash
# Open the kernel menu: list kernels, update packages, clean old kernels
pdpve -k
```

Run the LAN scan by entering `ip` (or `findip`) in the main menu.

## CLI Reference

### Main Flags

| Flag | Description |
|------|-------------|
| (none) | Check the installation: install when missing, ask before overwriting a different version, then open the main menu |
| `-qm`, `--qemu` | Open the `qm` VM menu |
| `-c`, `--cpu` | Run the `cpu` module and forward the remaining arguments |
| `-f`, `--fan` | Run the `fan` module and forward the remaining arguments |
| `-k`, `--kernel` | Open the `kernel` menu |
| `-s`, `--speedtest` | Run `speedtest` |
| `-v`, `--version` | Print the version (currently `v0.1.4`) |
| `-h`, `--help` | Check root privileges and open the main menu |
| `-r`, `--remove` | Check root privileges and open the main menu; enter `rm` there to uninstall |
| `-ip`, `--findip` | Calls an undefined `_FIND_IP` in v0.1.4; use the `ip` menu entry instead |

### Main Menu

| Input | Action |
|-------|--------|
| `qm` | VM menu |
| `ip`, `findip` | LAN device scan |
| `c`, `cpu` | CPU frequency control |
| `f`, `fan` | Fan control |
| `k`, `kernel` | PVE kernel menu |
| `s`, `speedtest` | Network speed test |
| `u`, `update` | Update pdpve from GitHub |
| `rm`, `remove` | Remove pdpve from the system |
| `h`, `help` | Show the main menu again |
| `b`, `bye` | Exit |

Any other input asks whether to close the program.

### `qm` Menu

| Input | Action |
|-------|--------|
| `i`, `install` | Run `qm-install` to create a VM |
| `l`, `list` | List VMIDs from `/etc/pve/qemu-server/*.conf` with `qm status` |
| `e`, `edit` | Edit `<VMID>.conf` with `nano` |
| `o`, `start` | `qm start` |
| `d`, `shutdown` | `qm shutdown` |
| `r`, `reboot` | `qm shutdown` then `qm start` |
| `s`, `stop` | `qm stop` |
| `rm`, `remove` | `qm destroy --purge` after confirmation |
| `ssh` | SSH as `ciuser` to `<vmbr0 subnet>.<VMID>` |
| `b`, `bye` | Exit |

### `qm-install` Flags

| Flag | Description | Default |
|------|-------------|---------|
| `-i`, `--id` | VM ID; re-prompted if it already exists | Prompted |
| `-s`, `--storage` | Storage ID; re-prompted if it does not exist | Prompted |
| `-o`, `--os` | `debian`, `rockylinux`, `ubuntu` | Prompted |
| `-v`, `--version` | OS release; the image URL is built only by the interactive picker, so the release menu still appears | Prompted |
| `-n`, `--name` | VM name | `qemu` |
| `-c`, `--cpu` | vCPU count | `2` |
| `-d`, `--disk` | Disk size; values below 16G become `16G` | `16G` |
| `-r`, `--ram` | Memory (MB) | `2048` |

Prompt-only fields: IP (default `dhcp`; static IPs use `x.x.x.x/nn` plus a gateway), username (left empty means no `ciuser`), password (default `passwd`).

### Supported Cloud Images

| OS | Releases | Source |
|----|----------|--------|
| Debian | `11` (bullseye), `12` (bookworm), `13` (trixie) | `https://cloud.debian.org/images/cloud/<codename>/latest/debian-<release>-generic-amd64.qcow2` |
| Rocky Linux | `8`, `9`, `10` | `https://dl.rockylinux.org/pub/rocky/<release>/images/x86_64/Rocky-<release>-GenericCloud-Base.latest.x86_64.qcow2` |
| Ubuntu | `20.04`, `22.04`, `24.04` | `https://mirror.twds.com.tw/ubuntu-cloud-images/releases/<release>/release/ubuntu-<release>-server-cloudimg-amd64.img` |

The script checks each URL with `curl --head` before downloading.

### Fixed VM Spec

| Item | Value |
|------|-------|
| CPU type | `x86-64-v2-AES` |
| SCSI controller | `virtio-scsi-pci` |
| OS type | `l26` |
| QEMU Guest Agent | Enabled (`--agent 1`) |
| Network | `virtio` on bridge `vmbr0` |
| Serial | `serial0 socket` (prevents a kernel panic during Debian cloud-init) |
| Disks | Imported image as `scsi0`, cloud-init drive on `ide2`, boot from `scsi0` |

### `cpu` Flags

| Flag | Description |
|------|-------------|
| `-s`, `--set` | Set frequencies |
| `-i`, `--info` | Show CPU model, core count, temperature (`Tctl` first, then `CPUTIN`), and per-core frequency |
| `-c`, `--core` | Number of performance cores N (cores `0` to `N-1`) |
| `-x`, `--max` | Maximum frequency in 100 MHz units (`30` = 3.0 GHz) |
| `-n`, `--min` | Minimum frequency in the same units; defaults to the maximum |
| `-h`, `--help` | Show help |

Without `-s` / `-i`, the module shows a menu: `s` / `set`, `i` / `info`, `h` / `help`, `bye`. The remaining cores (`N` to the last core) switch to powersave with both limits set to `cpuinfo_min_freq` (`800000` kHz when unreadable).

### `fan` Flags

| Flag | Description |
|------|-------------|
| `-s`, `--speed` | `auto` or a speed percentage |
| `-b`, `--brand` | `dell`, `supermicro`, `hpe`, `asrockrack`; detected with `dmidecode -s system-manufacturer` when omitted, with an interactive picker as fallback |

The script converts the percentage to `speed * 80 / 100` in hex (`<hex>`) and sends:

| Brand | Auto mode | Manual mode |
|-------|-----------|-------------|
| Dell | `0x30 0x30 0x01 0x01` | `0x30 0x30 0x01 0x00` -> `0x30 0x30 0x02 0xff 0x<hex>` |
| Supermicro | `0x30 0x45 0x01 0x01` | `0x30 0x45 0x01 0x00` -> `0x30 0x70 0x66 0x01 0x00 0x<hex>` |
| HPE | `0x3a 0x07 0x01 0x01` | `0x3a 0x07 0x01 0x00` -> `0x3a 0x07 0x02 0x<hex>` |
| ASRock Rack | `0x3a 0x01 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00` | `0x3a 0x01 0x00 0x<hex> 0x<hex> 0x<hex> 0x00 0x00 0x00 0x00` |

### `kernel` Menu

| Input | Action |
|-------|--------|
| `l`, `list` | List installed `pve-kernel` / `proxmox-kernel` packages (tagged `[當前]` / `[可清理]`), `rc` packages, and `/boot/vmlinuz-*` |
| `u`, `update` | `apt update` -> `apt upgrade -y` -> `apt autoremove -y` -> `apt autoclean -y` -> `apt clean` |
| `c`, `clean` | After confirmation, purge `rc` kernel packages with `dpkg --purge`, excluding the running kernel |
| `b`, `bye` | Exit |

### `find-ip` Behavior

1. Read the `scope link` subnet of `vmbr0` from `ip route` and treat its first three octets as a /24
2. Clear the ARP entry and run `ping -c 1 -W 1` in parallel for `.1` to `.254`
3. Print IP, MAC, and vendor for each `arp -n` entry

| MAC prefix | Label |
|------------|-------|
| `bc:24:11` | `Proxmox VM` |
| `00:50:56`, `00:0c:29` | `VMware VM` |
| `52:54:00` | `QEMU/KVM VM` |
| Others | Query `https://api.macvendors.com/<MAC>`; empty on failure |

***

©️ 2025 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
