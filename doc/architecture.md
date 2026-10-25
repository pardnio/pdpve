# pdpve - Architecture

Last updated: 2026-10-25

> Back to [README](../README.md)

## Overview

```mermaid
graph TB
    U[root user] --> P[pdpve entry script]
    P --> L[_CALL_MODULE loader]
    L --> QM[qm VM menu]
    QM --> QI[qm-install provisioning]
    L --> CPU[cpu frequency control]
    L --> FAN[fan control]
    L --> IP[find-ip LAN scan]
    L --> K[kernel management]
    P --> ST[speedtest]
    P --> GH[GitHub update source]
    QM --> PVE[Proxmox VE qm / pvesm]
    QI --> PVE
    QI --> IMG[Cloud image mirrors]
    CPU --> SYS[sysfs cpufreq / sensors]
    FAN --> IPMI[ipmitool / BMC]
    IP --> ARP[ping / arp / macvendors]
    K --> PKG[dpkg / apt]
```

## Module: pdpve Entry Script

Parses flags or shows the interactive main menu, installs, updates, and removes itself, and dispatches work to the modules.

```mermaid
graph TB
    subgraph pdpve
        ARGS[Flag parsing] -->|no flag| CHK[_CHECK_INSTALL]
        CHK -->|missing or different version| INS[_INSTALL]
        CHK --> INIT[_INIT root check]
        INS --> INIT
        ARGS -->|-h / -r| INIT
        INIT --> DO[_DO main menu loop]
        ARGS -->|-qm / -c / -f / -k| CALL[_CALL_MODULE]
        DO --> CALL
        DO --> UPD[_UPDATE]
        DO --> RM[_REMOVE]
        ARGS -->|-v| VER[Print version]
    end
    CALL -->|first| LOCAL[./module/name]
    CALL -->|fallback| SYSMOD["/usr/local/bin/pdpve/module/name"]
    INS --> BIN["/usr/local/sbin/pdpve"]
    UPD --> GIT[git clone into /tmp/pdpve]
    RM --> BIN
```

## Module: qm

VM menu that wraps `qm` commands and provides SSH access with an IP derived from the VMID.

```mermaid
graph TB
    subgraph qm
        MENU[_DO menu] --> LIST[_SHOW_LIST]
        MENU --> STATE[_SET_STATE]
        MENU --> EDIT[Edit config]
        MENU --> SSH[SSH connect]
        MENU --> INST[_CALL_MODULE qm-install]
        STATE --> ACT[start / shutdown / reboot / stop / remove]
    end
    LIST --> CONF["/etc/pve/qemu-server/*.conf"]
    EDIT --> NANO[nano]
    ACT --> QMCMD[qm commands]
    SSH --> CFG[qm config reads ciuser]
    SSH --> ROUTE[ip route reads vmbr0 subnet]
    ROUTE --> TARGET[subnet.VMID]
```

## Module: qm-install

Creates a VM from a cloud image and cleans up on failure.

```mermaid
graph TB
    subgraph qm-install
        IN[Flags / prompts] --> VAL[Validate VMID and storage]
        VAL --> PICK[Pick OS and release]
        PICK --> HEAD[curl --head checks URL]
        HEAD --> DL[Download to /tmp or reuse cache]
        DL --> CREATE[qm create]
        CREATE --> IMPORT[qm importdisk]
        IMPORT --> KEY[Prepare SSH keys]
        KEY --> SET[qm set user / password / keys / disks / boot]
        SET --> RESIZE[qm resize]
        RESIZE --> NET[qm set ipconfig0]
        NET --> START[qm start]
        IMPORT -->|fail| CLEAN[cleanup_vm]
        KEY -->|fail| CLEAN
        RESIZE -->|fail| CLEAN
    end
    PVESM[pvesm status] --> VAL
    MIRROR[Debian / Rocky / Ubuntu mirrors] --> DL
    SSHDIR[~/.ssh/id_rsa.pub + ~/.pdpve_pubkey] --> KEY
    CLEAN --> DESTROY[qm destroy --purge]
```

## Module: cpu

Splits cores into performance and powersave groups with frequency limits, and reports CPU info.

```mermaid
graph TB
    subgraph cpu
        OPT[getopt flags] --> ACTION{Action}
        ACTION -->|set| SUP[check_cpu_support]
        SUP -->|EPYC| SKIP[Skip]
        SUP --> ASK[Get core count / max / min]
        ASK --> PERF[Cores 0..N-1 performance]
        ASK --> SAVE[Cores N..last powersave]
        ACTION -->|info| INFO[Model / cores / temperature / frequency]
    end
    AVAIL[scaling_available_frequencies] --> ASK
    PERF --> SYSFS[scaling_max_freq / scaling_min_freq]
    SAVE --> SYSFS
    PERF --> SETGOV[cpufreq-set]
    SAVE --> SETGOV
    MINF[cpuinfo_min_freq] --> SAVE
    INFO --> SENS[sensors Tctl / CPUTIN]
```

## Module: fan

Detects the server brand and sends the matching IPMI raw commands.

```mermaid
graph TB
    subgraph fan
        OPT[getopt flags] --> SPEED[Speed: auto or percent]
        SPEED --> HEX[Convert speed*80/100 to hex]
        OPT --> BRAND{Brand}
        BRAND -->|not set| DETECT[getBrand]
        DETECT -->|unknown| PICK[Interactive picker]
        BRAND --> RAW[Build raw command]
        HEX --> RAW
    end
    DMI[dmidecode system-manufacturer] --> DETECT
    RAW --> IPMI[ipmitool raw]
    IPMI --> BMC[Dell / Supermicro / HPE / ASRock Rack BMC]
```

## Module: find-ip

Scans the /24 subnet behind `vmbr0` and identifies devices.

```mermaid
graph TB
    subgraph find-ip
        ROUTE[Read vmbr0 subnet] --> SWEEP[Parallel ping .1 to .254]
        SWEEP --> TABLE[Read arp -n]
        TABLE --> MAC{MAC prefix}
        MAC -->|bc:24:11| PVE[Proxmox VM]
        MAC -->|00:50:56 / 00:0c:29| VMW[VMware VM]
        MAC -->|52:54:00| KVM[QEMU/KVM VM]
        MAC -->|other| LOOKUP[Vendor lookup]
    end
    LOOKUP --> API[api.macvendors.com]
```

## Module: kernel

Lists, updates, and cleans PVE kernel packages.

```mermaid
graph TB
    subgraph kernel
        MENU[_DO menu] --> LIST[_GET_LIST]
        LIST --> INST[_GET_INSTALLED]
        LIST --> RC[_GET_RC]
        MENU --> UPD[Package update]
        MENU --> CLEAN[_CLEAN]
        CLEAN --> FILTER[Exclude running kernel]
        FILTER --> CONFIRM[Confirm]
    end
    INST --> DPKG[dpkg --list]
    RC --> DPKG
    LIST --> BOOT["/boot/vmlinuz-*"]
    UPD --> APT[apt update / upgrade / autoremove / autoclean / clean]
    CONFIRM --> PURGE[dpkg --purge]
```

## Data Flow

Interaction sequence when creating a VM:

```mermaid
sequenceDiagram
    participant User
    participant pdpve
    participant qm
    participant qm-install
    participant Mirror
    participant PVE as Proxmox VE
    User->>pdpve: pdpve -qm
    pdpve->>qm: _CALL_MODULE qm
    User->>qm: i
    qm->>qm-install: _CALL_MODULE qm-install
    qm-install->>PVE: qm config / pvesm status validation
    qm-install->>Mirror: curl --head checks release
    qm-install->>Mirror: curl -L downloads image
    qm-install->>PVE: qm create
    qm-install->>PVE: qm importdisk
    qm-install->>PVE: qm set ciuser / cipassword / sshkeys / scsi0 / ide2 / boot
    qm-install->>PVE: qm resize scsi0
    qm-install->>PVE: qm set ipconfig0
    qm-install->>PVE: qm start
    alt Any step fails
        qm-install->>PVE: qm destroy --purge
    end
```

## State Machine

Installation check when the entry script starts without flags:

```mermaid
stateDiagram-v2
    [*] --> CheckInstall
    CheckInstall --> Install: not installed
    CheckInstall --> AskOverwrite: different version
    CheckInstall --> MainMenu: same version
    AskOverwrite --> Install: y
    AskOverwrite --> MainMenu: N
    Install --> CheckInstall: press Enter
    MainMenu --> MainMenu: press Enter after an action
    MainMenu --> [*]: b / bye
    MainMenu --> [*]: update finished
```

***

©️ 2025 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
