# pdpve - 架構

最後更新：2026-10-25

> 返回 [README](./README.zh.md)

## 概覽

```mermaid
graph TB
    U[root 使用者] --> P[pdpve 主程式]
    P --> L[_CALL_MODULE 模組載入]
    L --> QM[qm 虛擬機選單]
    QM --> QI[qm-install 佈建]
    L --> CPU[cpu 頻率控制]
    L --> FAN[fan 風扇控制]
    L --> IP[find-ip 區網掃描]
    L --> K[kernel 核心管理]
    P --> ST[speedtest]
    P --> GH[GitHub 更新來源]
    QM --> PVE[Proxmox VE qm / pvesm]
    QI --> PVE
    QI --> IMG[Cloud Image 下載站]
    CPU --> SYS[sysfs cpufreq / sensors]
    FAN --> IPMI[ipmitool / BMC]
    IP --> ARP[ping / arp / macvendors]
    K --> PKG[dpkg / apt]
```

## Module: pdpve 主程式

解析參數或顯示互動主選單，安裝、更新、移除自身，並把工作分派給各模組。

```mermaid
graph TB
    subgraph pdpve
        ARGS[參數解析] -->|無參數| CHK[_CHECK_INSTALL]
        CHK -->|未安裝或版本不同| INS[_INSTALL]
        CHK --> INIT[_INIT root 檢查]
        INS --> INIT
        ARGS -->|-h / -r| INIT
        INIT --> DO[_DO 主選單迴圈]
        ARGS -->|-qm / -c / -f / -k| CALL[_CALL_MODULE]
        DO --> CALL
        DO --> UPD[_UPDATE]
        DO --> RM[_REMOVE]
        ARGS -->|-v| VER[印出版本]
    end
    CALL -->|優先| LOCAL[./module/名稱]
    CALL -->|備援| SYSMOD["/usr/local/bin/pdpve/module/名稱"]
    INS --> BIN["/usr/local/sbin/pdpve"]
    UPD --> GIT[git clone 至 /tmp/pdpve]
    RM --> BIN
```

## Module: qm

虛擬機選單，包裝 `qm` 指令並提供以 VMID 推算 IP 的 SSH 連線。

```mermaid
graph TB
    subgraph qm
        MENU[_DO 選單] --> LIST[_SHOW_LIST]
        MENU --> STATE[_SET_STATE]
        MENU --> EDIT[編輯設定檔]
        MENU --> SSH[SSH 連線]
        MENU --> INST[_CALL_MODULE qm-install]
        STATE --> ACT[start / shutdown / reboot / stop / remove]
    end
    LIST --> CONF["/etc/pve/qemu-server/*.conf"]
    EDIT --> NANO[nano]
    ACT --> QMCMD[qm 指令]
    SSH --> CFG[qm config 取 ciuser]
    SSH --> ROUTE[ip route 取 vmbr0 網段]
    ROUTE --> TARGET[網段.VMID]
```

## Module: qm-install

以 Cloud Image 建立虛擬機，失敗時清除殘留。

```mermaid
graph TB
    subgraph qm-install
        IN[參數 / 互動輸入] --> VAL[驗證 VMID 與儲存區]
        VAL --> PICK[選擇作業系統與版本]
        PICK --> HEAD[curl --head 確認網址]
        HEAD --> DL[下載至 /tmp 或沿用快取]
        DL --> CREATE[qm create]
        CREATE --> IMPORT[qm importdisk]
        IMPORT --> KEY[SSH 金鑰準備]
        KEY --> SET[qm set 使用者 / 密碼 / 金鑰 / 磁碟 / 開機]
        SET --> RESIZE[qm resize]
        RESIZE --> NET[qm set ipconfig0]
        NET --> START[qm start]
        IMPORT -->|失敗| CLEAN[cleanup_vm]
        KEY -->|失敗| CLEAN
        RESIZE -->|失敗| CLEAN
    end
    PVESM[pvesm status] --> VAL
    MIRROR[Debian / Rocky / Ubuntu 下載站] --> DL
    SSHDIR[~/.ssh/id_rsa.pub + ~/.pdpve_pubkey] --> KEY
    CLEAN --> DESTROY[qm destroy --purge]
```

## Module: cpu

依核心數分段設定 governor 與頻率上下限，並提供 CPU 資訊。

```mermaid
graph TB
    subgraph cpu
        OPT[getopt 參數] --> ACTION{動作}
        ACTION -->|set| SUP[check_cpu_support]
        SUP -->|EPYC| SKIP[跳過]
        SUP --> ASK[取得核心數 / 最高 / 最低頻率]
        ASK --> PERF[核心 0..N-1 performance]
        ASK --> SAVE[核心 N..末 powersave]
        ACTION -->|info| INFO[型號 / 核心數 / 溫度 / 頻率]
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

偵測伺服器品牌並送出對應的 IPMI raw 指令。

```mermaid
graph TB
    subgraph fan
        OPT[getopt 參數] --> SPEED[轉速：auto 或百分比]
        SPEED --> HEX[換算 speed*80/100 十六進位]
        OPT --> BRAND{品牌}
        BRAND -->|未指定| DETECT[getBrand]
        DETECT -->|失敗| PICK[互動選擇]
        BRAND --> RAW[組合 raw 指令]
        HEX --> RAW
    end
    DMI[dmidecode system-manufacturer] --> DETECT
    RAW --> IPMI[ipmitool raw]
    IPMI --> BMC[Dell / Supermicro / HPE / ASRock Rack BMC]
```

## Module: find-ip

掃描 `vmbr0` 所在 /24 網段並辨識裝置。

```mermaid
graph TB
    subgraph find-ip
        ROUTE[取得 vmbr0 網段] --> SWEEP[.1 至 .254 平行 ping]
        SWEEP --> TABLE[讀取 arp -n]
        TABLE --> MAC{MAC 前綴}
        MAC -->|bc:24:11| PVE[Proxmox VM]
        MAC -->|00:50:56 / 00:0c:29| VMW[VMware VM]
        MAC -->|52:54:00| KVM[QEMU/KVM VM]
        MAC -->|其他| LOOKUP[廠商查詢]
    end
    LOOKUP --> API[api.macvendors.com]
```

## Module: kernel

列出、更新與清理 PVE 核心套件。

```mermaid
graph TB
    subgraph kernel
        MENU[_DO 選單] --> LIST[_GET_LIST]
        LIST --> INST[_GET_INSTALLED]
        LIST --> RC[_GET_RC]
        MENU --> UPD[套件更新]
        MENU --> CLEAN[_CLEAN]
        CLEAN --> FILTER[排除當前核心版本]
        FILTER --> CONFIRM[確認]
    end
    INST --> DPKG[dpkg --list]
    RC --> DPKG
    LIST --> BOOT["/boot/vmlinuz-*"]
    UPD --> APT[apt update / upgrade / autoremove / autoclean / clean]
    CONFIRM --> PURGE[dpkg --purge]
```

## 資料流

建立虛擬機時的互動序列：

```mermaid
sequenceDiagram
    participant 使用者
    participant pdpve
    participant qm
    participant qm-install
    participant 下載站
    participant PVE as Proxmox VE
    使用者->>pdpve: pdpve -qm
    pdpve->>qm: _CALL_MODULE qm
    使用者->>qm: i
    qm->>qm-install: _CALL_MODULE qm-install
    qm-install->>PVE: qm config / pvesm status 驗證
    qm-install->>下載站: curl --head 確認版本
    qm-install->>下載站: curl -L 下載映像檔
    qm-install->>PVE: qm create
    qm-install->>PVE: qm importdisk
    qm-install->>PVE: qm set ciuser / cipassword / sshkeys / scsi0 / ide2 / boot
    qm-install->>PVE: qm resize scsi0
    qm-install->>PVE: qm set ipconfig0
    qm-install->>PVE: qm start
    alt 任一步驟失敗
        qm-install->>PVE: qm destroy --purge
    end
```

## 狀態機

主程式無參數啟動時的安裝檢查：

```mermaid
stateDiagram-v2
    [*] --> 檢查安裝
    檢查安裝 --> 安裝: 未安裝
    檢查安裝 --> 詢問覆蓋: 版本不同
    檢查安裝 --> 主選單: 版本相同
    詢問覆蓋 --> 安裝: y
    詢問覆蓋 --> 主選單: N
    安裝 --> 檢查安裝: 按 Enter 繼續
    主選單 --> 主選單: 執行動作後按 Enter
    主選單 --> [*]: b / bye
    主選單 --> [*]: update 完成
```

***

©️ 2025 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
