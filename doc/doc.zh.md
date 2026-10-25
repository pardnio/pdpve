# pdpve - 技術文件

最後更新：2026-10-25

> 返回 [README](./README.zh.md)

## 前置需求

- Proxmox VE 主機（使用 `qm`、`pvesm`、`/etc/pve/qemu-server/` 與 `vmbr0` 橋接網路）
- root 權限（主選單與 `kernel` 模組啟動時檢查 `EUID`）
- Bash、`getopt`（util-linux）、`curl`、`git`
- 各模組額外依賴：

| 模組 | 依賴指令 |
|------|----------|
| `fan` | `ipmitool`、`dmidecode`、支援 IPMI 的 BMC |
| `cpu` | `cpufreq-set`（cpufrequtils）、`sensors`（lm-sensors） |
| `find-ip` | `arp`（net-tools）、`ping`；廠商查詢需連線 `api.macvendors.com` |
| `qm` | `nano`（編輯設定檔）、`ssh` |
| `qm-install` | `ssh-keygen`；需能連線至 Cloud Image 下載站 |
| speedtest | `speedtest` CLI（本工具不負責安裝） |

## 安裝

### 從原始碼安裝

須在 repo 根目錄執行，安裝程序會複製當前目錄下的 `./module`：

```bash
git clone https://github.com/pardnchiu/pdpve.git
cd pdpve
bash pdpve
```

首次執行（未帶參數）時：

1. 將 `pdpve` 複製至 `/usr/local/sbin/pdpve` 並加上執行權限
2. 將 `module/` 複製至 `/usr/local/bin/pdpve/module/`
3. 之後直接執行 `pdpve` 即可進入互動選單

若系統已安裝不同版本，會顯示版本差異並詢問是否覆蓋。

### 更新

在主選單輸入 `u`（或 `update`）：從 `https://github.com/pardnchiu/pdpve.git` clone 至 `/tmp/pdpve`，覆蓋 `/usr/local/sbin/pdpve` 與 `/usr/local/bin/pdpve/module/`，完成後刪除暫存目錄並結束程式。

### 移除

在主選單輸入 `rm`（或 `remove`），確認後刪除 `/usr/local/sbin/pdpve` 與 `/usr/local/bin/pdpve`。

## 設定

本工具不讀取環境變數，使用下列固定路徑：

| 路徑 | 用途 |
|------|------|
| `/usr/local/sbin/pdpve` | 已安裝的主程式 |
| `/usr/local/bin/pdpve/module/` | 已安裝的模組；呼叫時優先使用當前目錄 `./module/`，找不到才使用此路徑 |
| `~/.ssh/id_rsa`、`~/.ssh/id_rsa.pub` | 建立虛擬機時注入的 SSH 金鑰；不存在時自動以 RSA 4096 產生 |
| `~/.pdpve_pubkey` | 選用；額外的公鑰清單，與 `id_rsa.pub` 合併後一起注入虛擬機 |
| `/tmp/<映像檔名稱>` | Cloud Image 快取；已存在時直接沿用，不重新下載 |
| `/etc/pve/qemu-server/<VMID>.conf` | 虛擬機清單與編輯功能讀取的設定檔 |

## 使用方式

### 基礎用法

```bash
# 進入互動主選單（需 root）
pdpve

# 顯示版本
pdpve -v

# 顯示 CPU 型號、核心數、溫度與各核心目前頻率
pdpve -c -i
```

### 虛擬機管理

```bash
# 進入虛擬機選單：安裝、清單、編輯、啟動、關機、重啟、停止、移除、SSH
pdpve -qm
```

在選單中輸入 `i` 會逐步詢問：VMID（已存在則重新輸入）→ 儲存區（以 `pvesm status` 驗證）→ 作業系統 → 版本 → 是否下載映像檔 → 名稱、vCPU、磁碟、RAM → IP（`dhcp` 或 CIDR 格式並填 Gateway）→ 使用者名稱與密碼。

選單的 `ssh` 會以虛擬機的 `ciuser` 連線至 `<vmbr0 網段前三碼>.<VMID>`，因此建議靜態 IP 的最後一碼與 VMID 相同，並在安裝時填入使用者名稱。

### 以參數直接建立虛擬機

`qm` 選單呼叫 `qm-install` 時不會轉傳參數；需要預先帶入參數時直接執行模組：

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

未帶入的欄位（版本、IP、Gateway、使用者、密碼）仍會互動詢問。任何 `qm create`、`importdisk`、`resize` 或 SSH 金鑰產生失敗時，會以 `qm destroy --purge` 清除該虛擬機並以 exit code `1` 結束。

### CPU 頻率

```bash
# 前 4 個核心鎖定 performance，最高 3.0 GHz、最低 1.2 GHz；其餘核心降至最低頻率
pdpve -c -s -c 4 -x 30 -n 12
```

`-x`／`-n` 的值會被串接 `00000` 後寫入 `scaling_max_freq`／`scaling_min_freq`（單位 kHz），因此 `30` 代表 3.0 GHz。未給 `-n` 時最低頻率等於最高頻率。CPU 型號含 `EPYC` 時直接跳過設定。

### 風扇轉速

```bash
# 自動偵測品牌，切換為自動模式
pdpve -f -s auto

# 指定品牌並固定轉速 40%
pdpve -f -s 40 -b dell
```

互動輸入轉速時會自動限制在 20–80；以 `-s` 帶入時不做範圍調整。IPMI 指令失敗時（Dell、Supermicro、HPE）印出錯誤並以 exit code `1` 結束。

### 區網掃描與核心管理

```bash
# 進入核心選單：列出核心、更新套件、清理舊核心
pdpve -k
```

區網掃描請在主選單輸入 `ip`（或 `findip`）。

## 命令列參考

### 主程式參數

| 參數 | 說明 |
|------|------|
| （無） | 檢查安裝狀態：未安裝則安裝，版本不同則詢問覆蓋，之後進入互動主選單 |
| `-qm`, `--qemu` | 進入 `qm` 虛擬機選單 |
| `-c`, `--cpu` | 執行 `cpu` 模組，其餘參數轉傳給模組 |
| `-f`, `--fan` | 執行 `fan` 模組，其餘參數轉傳給模組 |
| `-k`, `--kernel` | 進入 `kernel` 核心選單 |
| `-s`, `--speedtest` | 執行 `speedtest` |
| `-v`, `--version` | 印出版本號（目前 `v0.1.4`） |
| `-h`, `--help` | 檢查 root 權限後進入互動主選單 |
| `-r`, `--remove` | 檢查 root 權限後進入互動主選單，於選單輸入 `rm` 移除 |
| `-ip`, `--findip` | v0.1.4 中呼叫未定義的 `_FIND_IP`，請改用主選單 `ip` |

### 主選單

| 輸入 | 動作 |
|------|------|
| `qm` | 虛擬機選單 |
| `ip`, `findip` | 區網裝置掃描 |
| `c`, `cpu` | CPU 頻率控制 |
| `f`, `fan` | 風扇控制 |
| `k`, `kernel` | PVE 核心選單 |
| `s`, `speedtest` | 網路速度測試 |
| `u`, `update` | 從 GitHub 更新 pdpve |
| `rm`, `remove` | 從系統移除 pdpve |
| `h`, `help` | 重新顯示主選單 |
| `b`, `bye` | 離開 |

其他輸入會詢問是否關閉程式。

### `qm` 選單

| 輸入 | 動作 |
|------|------|
| `i`, `install` | 執行 `qm-install` 建立虛擬機 |
| `l`, `list` | 列出 `/etc/pve/qemu-server/*.conf` 的 VMID 與 `qm status` |
| `e`, `edit` | 以 `nano` 編輯 `<VMID>.conf` |
| `o`, `start` | `qm start` |
| `d`, `shutdown` | `qm shutdown` |
| `r`, `reboot` | `qm shutdown` 後 `qm start` |
| `s`, `stop` | `qm stop` |
| `rm`, `remove` | 確認後 `qm destroy --purge` |
| `ssh` | 以 `ciuser` SSH 至 `<vmbr0 網段>.<VMID>` |
| `b`, `bye` | 離開 |

### `qm-install` 參數

| 參數 | 說明 | 預設 |
|------|------|------|
| `-i`, `--id` | 虛擬機 ID，已存在則重新詢問 | 互動輸入 |
| `-s`, `--storage` | 儲存區 ID，不存在則重新詢問 | 互動輸入 |
| `-o`, `--os` | `debian`、`rockylinux`、`ubuntu` | 互動選擇 |
| `-v`, `--version` | 系統版本；映像檔網址只在互動選擇時產生，帶入此值仍會進入版本選單 | 互動選擇 |
| `-n`, `--name` | 虛擬機名稱 | `qemu` |
| `-c`, `--cpu` | vCPU 數量 | `2` |
| `-d`, `--disk` | 磁碟大小，小於 16G 時自動調整為 `16G` | `16G` |
| `-r`, `--ram` | 記憶體（MB） | `2048` |

互動欄位：IP（預設 `dhcp`，靜態需為 `x.x.x.x/nn` 並填 Gateway）、使用者名稱（留空則不設定 `ciuser`）、密碼（預設 `passwd`）。

### 支援的 Cloud Image

| 作業系統 | 版本 | 來源 |
|----------|------|------|
| Debian | `11`（bullseye）、`12`（bookworm）、`13`（trixie） | `https://cloud.debian.org/images/cloud/<代號>/latest/debian-<版本>-generic-amd64.qcow2` |
| Rocky Linux | `8`、`9`、`10` | `https://dl.rockylinux.org/pub/rocky/<版本>/images/x86_64/Rocky-<版本>-GenericCloud-Base.latest.x86_64.qcow2` |
| Ubuntu | `20.04`、`22.04`、`24.04` | `https://mirror.twds.com.tw/ubuntu-cloud-images/releases/<版本>/release/ubuntu-<版本>-server-cloudimg-amd64.img` |

下載前會以 `curl --head` 確認網址存在。

### 虛擬機固定規格

| 項目 | 值 |
|------|----|
| CPU 類型 | `x86-64-v2-AES` |
| SCSI 控制器 | `virtio-scsi-pci` |
| OS 類型 | `l26` |
| QEMU Guest Agent | 啟用（`--agent 1`） |
| 網路 | `virtio`，橋接 `vmbr0` |
| Serial | `serial0 socket`（避免 Debian cloud-init 初始化時 kernel panic） |
| 磁碟 | 匯入的映像檔掛為 `scsi0`，`ide2` 為 cloud-init 磁碟，由 `scsi0` 開機 |

### `cpu` 參數

| 參數 | 說明 |
|------|------|
| `-s`, `--set` | 設定頻率 |
| `-i`, `--info` | 顯示 CPU 型號、核心數、溫度（優先 `Tctl`，其次 `CPUTIN`）與各核心頻率 |
| `-c`, `--core` | 套用 performance 的核心數 N（核心 `0` 至 `N-1`） |
| `-x`, `--max` | 最高頻率，單位 100 MHz（`30` = 3.0 GHz） |
| `-n`, `--min` | 最低頻率，單位同上；省略時等於最高頻率 |
| `-h`, `--help` | 顯示說明 |

未帶 `-s`／`-i` 時顯示選單：`s`／`set`、`i`／`info`、`h`／`help`、`bye`。其餘核心（`N` 至最後一核）設為 powersave，最高與最低頻率皆為 `cpuinfo_min_freq`（讀取失敗時為 `800000` kHz）。

### `fan` 參數

| 參數 | 說明 |
|------|------|
| `-s`, `--speed` | `auto` 或轉速百分比 |
| `-b`, `--brand` | `dell`、`supermicro`、`hpe`、`asrockrack`；省略時以 `dmidecode -s system-manufacturer` 偵測，偵測失敗則互動選擇 |

轉速百分比會換算為 `speed * 80 / 100` 的十六進位值（`<hex>`）後送出：

| 品牌 | 自動模式 | 手動模式 |
|------|----------|----------|
| Dell | `0x30 0x30 0x01 0x01` | `0x30 0x30 0x01 0x00` → `0x30 0x30 0x02 0xff 0x<hex>` |
| Supermicro | `0x30 0x45 0x01 0x01` | `0x30 0x45 0x01 0x00` → `0x30 0x70 0x66 0x01 0x00 0x<hex>` |
| HPE | `0x3a 0x07 0x01 0x01` | `0x3a 0x07 0x01 0x00` → `0x3a 0x07 0x02 0x<hex>` |
| ASRock Rack | `0x3a 0x01 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00` | `0x3a 0x01 0x00 0x<hex> 0x<hex> 0x<hex> 0x00 0x00 0x00 0x00` |

### `kernel` 選單

| 輸入 | 動作 |
|------|------|
| `l`, `list` | 列出已安裝的 `pve-kernel`／`proxmox-kernel` 套件（標示 `[當前]`、`[可清理]`）、`rc` 狀態套件與 `/boot/vmlinuz-*` |
| `u`, `update` | `apt update` → `apt upgrade -y` → `apt autoremove -y` → `apt autoclean -y` → `apt clean` |
| `c`, `clean` | 確認後以 `dpkg --purge` 清除 `rc` 狀態的核心套件，排除當前核心版本 |
| `b`, `bye` | 離開 |

### `find-ip` 行為

1. 由 `ip route` 取得 `vmbr0` 的 `scope link` 網段，取前三碼視為 /24
2. 對 `.1` 至 `.254` 清除 ARP 快取並平行 `ping -c 1 -W 1`
3. 列出 `arp -n` 結果的 IP、MAC 與廠商

| MAC 前綴 | 標示 |
|----------|------|
| `bc:24:11` | `Proxmox VM` |
| `00:50:56`、`00:0c:29` | `VMware VM` |
| `52:54:00` | `QEMU/KVM VM` |
| 其他 | 查詢 `https://api.macvendors.com/<MAC>`，失敗則留空 |

***

©️ 2025 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
