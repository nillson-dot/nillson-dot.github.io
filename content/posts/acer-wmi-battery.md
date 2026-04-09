+++
title = '《在 Linux 上實現 Acer 筆電充電管理：從模組編譯到 DKMS 全自動化》'
date = 2026-04-08T12:00:00+08:00
draft = false
tags = ["Linux", "Fedora", "Kernel", "Automation", "CSIE"]
categories = ["Technical Projects"]
+++

在 Windows 上，Acer Sense 可以輕鬆將電池充電上限設為 80% 以延長壽命。

在 Linux 的 `sysfs (虛擬檔案系統)` 標準電源介面中，充電限制參數通常會在

`/sys/class/power_supply/BATx/`

但我在 `/sys` 下卻找不到相關參數檔案，這意味著 Linux 標準的核心驅動並未釋出 Acer 筆電的電池健康管理接口。

- 本專案在 [Github](https://github.com/nillson-dot/acer-wmi-battery) 上以開源專案 [acer-wmi-battery](https://github.com/frederik-h/acer-wmi-battery) 為基礎開發進階功能。
- 本專案在 僅在個人 Fedora 系統上測試，樣本數仍然不足，不保證一定成功！

---

## 取得驅動

在 Linux 核心中，許多硬體控制權都是透過 `sysfs 虛擬檔案系統` 釋出的。

幸運的是，Github 上已有現成且開源的專案 [acer-wmi-battery](https://github.com/frederik-h/acer-wmi-battery) ，套用後便能將硬體暫存器映射到系統路徑：

`/sys/bus/wmi/drivers/acer-wmi-battery/health_mode`

對這個檔案寫入 1 即開啟 80% 限制，寫入 0 則恢復 100% 充滿。

但在編譯和使用一段時間後，我遇到了兩個 Linux 系統常見的問題：

### 核心更新即失效

Linux 更新頻率極高，每次核心升級後，手動編譯的模組就會消失，必須在每次更新核心後重新編譯模組並載入。

### Secure Boot 簽名

自編譯模組若沒有數位簽名，會被 UEFI 拒絕載入。

---

## 解決方案

### 開始前準備

在開始之前，確保你已安裝 DKMS 和相關 Kernel Headers：

- **Debian / Ubuntu based:**

```bash
sudo apt update
sudo apt install dkms linux-headers-$(uname -r) build-essential
```

- **Fedora / RHEL based:**

```bash
sudo dnf install dkms kernel-devel kernel-headers
```

- **Arch Linux based:**

```bash
sudo pacman -S dkms linux-headers
```

#### MOK 簽名

若你的電腦有啟用「安全開機」選項，系統只會載入經過信任密鑰簽名的核心程式碼，
所以你需要建立一份 MOK (Machine Owner Key) 來為你的模組簽名。

**以下僅為範例，不一定得照著我的方法做**

建立個人密鑰對（公鑰和私鑰）：

```bash
# 建立存放密鑰的資料夾
sudo mkdir -p /root/mok
sudo cd /root/mok

# 生成 4096 位元的 RSA 密鑰對
sudo openssl req -new -x509 -newkey rsa:2048 -keyout MOK.priv -outform DER -out MOK.der -nodes -days 36500 -subj "/CN=My Battery Key/"
```

將公鑰匯入系統的 MOK 清單：

```bash
sudo mokutil --import MOK.der
```

此時系統會要求輸入一組「臨時密碼」（簡單、容易記就好），接著重新開機會進入藍底的 MOK Management 界面。

依序選擇 `Enroll MOK` -> `Continue` -> `Yes` -> `輸入臨時密碼` -> `Reboot`。

為了將 MOK 位置提供給 DKMS ，請在 `/etc/dkms/framework.conf` 加上：

```
mok_signing_key="/root/mok/MOK.priv"
mok_certificate="/root/mok/MOK.der"
```

### 1. DKMS (Dynamic Kernel Module Support)

DKMS 是用來生成 Linux 的核心模組的框架，可以在更新核心版本後，自動編譯並安裝模組。

其規定要交給它管理的原始碼必須放在 `/usr/src/` 目錄下，目錄名稱必須包含「模組名稱」與「版本號」。

原始碼目錄下須包含 C 代碼、`Makefile` 以及 `dkms.conf` (DKMS 將透過此檔案得知如何編譯)， `dkms.conf` 已附在專案中，內容參考：

```
PACKAGE_NAME="acer-wmi-battery"
PACKAGE_VERSION="1.0"
BUILT_MODULE_NAME[0]="acer-wmi-battery"
DEST_MODULE_LOCATION[0]="/extra"
AUTOINSTALL="yes"
MAKE[0]="make -C $kernel_source_dir M=$dkms_tree/$PACKAGE_NAME/$PACKAGE_VERSION/build"
CLEAN="make -C $kernel_source_dir M=$dkms_tree/$PACKAGE_NAME/$PACKAGE_VERSION/build clean"
```

從專案 [acer-wmi-battery](https://github.com/nillson-dot/acer-wmi-battery) 取得原始碼：

```bash
git clone https://github.com/nillson-dot/acer-wmi-battery
```

切換到 acer-wmi-battery 原始碼目錄，將必要檔案複製過去：

```bash
# 建立目標目錄
sudo mkdir -p /usr/src/acer-wmi-battery-1.0

# 複製必要檔案
sudo cp acer-wmi-battery.c Makefile dkms.conf /usr/src/acer-wmi-battery-1.0/

# 加入模組至dkms
sudo dkms add -m acer-wmi-battery -v 1.0

# 編譯並安裝模組：
sudo dkms install -m acer-wmi-battery -v 1.0
```

### 2. 狀態同步架構：

由於模組參數是存在揮發記憶體中，Linux 系統重啟後會造成參數遺失，
因此我們要在硬碟新建一份紀錄目前 `health_mode` 設定的實體存檔。
我考量了兩種設計模式：

- **關機時紀錄狀態**

  此方法雖然只需要在關機時進行一次存檔，卻無法保證在系統崩潰和異常斷電時還能夠正常寫入。

- **更改參數時存檔（採用）**

  簡單來說就是在開關「充電限制」時將狀態寫入實體存檔。

為了使開關「充電限制」並同時存檔，我寫了名為 `set-battery` 的 CLI 工具。它將原本複雜且難記的寫入 `sysfs` 路徑操作和存檔封裝成直觀的指令：

```
#!/bin/bash

# Define paths
SYS_FILE="/sys/bus/wmi/drivers/acer-wmi-battery/health_mode"
STATE_FILE="/var/lib/acer-battery-state"

# Check input arguments
VALUE=$1
if [[ "$VALUE" != "0" && "$VALUE" != "1" ]]; then
echo "Usage: sudo $0 [0|1]"
echo "0: Disable limit (Charge to 100%)"
echo "1: Enable limit (Charge to 80%)"
exit 1
fi

# Core logic
if [ -f "$SYS_FILE" ]; then # Write to hardware
echo $VALUE > "$SYS_FILE"

    # Write to persistent state file
    echo $VALUE > "$STATE_FILE"

    echo "State updated to: $VALUE"

else
echo "Error: Driver module not loaded. Please check DKMS status or run 'sudo modprobe acer-wmi-battery'."
exit 1
fi
```

將腳本移至 Linux 自訂軟體路徑：

```bash
sudo cp set-battery /usr/local/bin/

# 添加執行權限
sudo chmod +x /usr/local/bin/set-battery
```

### 3. Systemd 服務：

為了開機時將 `health_mode` 參數存檔自動寫入 `sysfs（虛擬檔案系統）`，
我建立了 `acer-boot-restore.service` ：

```
[Unit]
Description=Restore Acer Battery Health Mode on Boot
After=systemd-modules-load.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'if [ -f /var/lib/acer-battery-state ]; then cat /var/lib/acer-battery-state > /sys/bus/wmi/drivers/acer-wmi-battery/health_mode; fi'

[Install]
WantedBy=multi-user.target
```

將檔案移到存放自定義服務的標準位置並啟用：

```bash
sudo cp acer-boot-restore.service /etc/systemd/system/

# 讓系統重新讀取並啟用設定檔
sudo systemctl daemon-reload
sudo systemctl enable --now acer-boot-restore.service
```

---

**現在你只需要透過 set-battery　即可快速切換：**

```bash
sudo set-battery 1 # 開啟充電限制 (80%)
sudo set-battery 0 # 關閉充電限制 (100%)
```
