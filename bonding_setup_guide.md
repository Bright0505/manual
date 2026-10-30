# Ubuntu 網路 Bonding 設定指南

## 概述

本指南說明如何在 Ubuntu 系統上設定網路 bonding（網路聚合），將多個網路介面合併以增加頻寬和提供冗餘性。

## 目標

- 將 4 個網路介面合併成一個邏輯介面
- 增加網路頻寬（理論上可達 4 倍）
- 提供網路連線的冗餘性

## 前置需求

- Ubuntu 系統具有 4 個網路介面
- 管理員權限 (sudo)
- 網路交換器（對於 LACP 模式需要支援 802.3ad）

## Bonding 模式說明

| 模式 | 名稱 | 頻寬增加 | 需要交換器支援 | 適用場景 |
|------|------|----------|----------------|----------|
| Mode 0 | balance-rr | ★★★★☆ | 否 | 高頻寬需求，可接受封包重排 |
| Mode 1 | active-backup | ★☆☆☆☆ | 否 | 高可用性，不需要頻寬增加 |
| Mode 4 | 802.3ad (LACP) | ★★★★★ | 是 | 最佳選擇，需要交換器配合 |
| Mode 6 | balance-alb | ★★★☆☆ | 否 | 智慧負載平衡 |

## 實施步驟

### 階段一：準備工作（無停機時間）

#### 1. 備份現有設定
```bash
# 創建備份目錄
sudo mkdir -p /etc/netplan/backup

# 備份現有設定
sudo cp /etc/netplan/*.yaml /etc/netplan/backup/
sudo cp -r /etc/netplan /root/netplan_backup
```

#### 2. 確認網路介面名稱
```bash
# 列出所有網路介面
ip link show

# 或使用
ls /sys/class/net/
```

#### 3. 安裝必要套件
```bash
sudo apt update
sudo apt install ifenslave

# 載入 bonding 模組
sudo modprobe bonding
echo 'bonding' | sudo tee -a /etc/modules
```

### 階段二：設定 Bonding（短暫停機）

#### 4. 創建 Bonding 設定檔

根據您的原始設定創建 `/etc/netplan/99-bonding.yaml`：

**Mode 0 設定（推薦先使用）：**
```yaml
network:
  version: 2
  ethernets:
    eno1:
      dhcp4: no
    eno2:  # 請根據實際介面名稱修改
      dhcp4: no
    eno3:  # 請根據實際介面名稱修改
      dhcp4: no
    eno4:  # 請根據實際介面名稱修改
      dhcp4: no
  bonds:
    bond0:
      interfaces: [eno1, eno2, eno3, eno4]  # 請根據實際介面名稱修改
      parameters:
        mode: balance-rr  # Mode 0
        mii-monitor-interval: 100
      addresses:
        - "192.168.1.100/24"
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
        search: []
      routes:
        - to: "default"
          via: "192.168.1.254"
```

**Mode 4 設定（需要交換器支援 LACP）：**
```yaml
network:
  version: 2
  ethernets:
    eno1:
      dhcp4: no
    eno2:
      dhcp4: no
    eno3:
      dhcp4: no
    eno4:
      dhcp4: no
  bonds:
    bond0:
      interfaces: [eno1, eno2, eno3, eno4]
      parameters:
        mode: 802.3ad  # Mode 4 (LACP)
        lacp-rate: fast
        mii-monitor-interval: 100
        transmit-hash-policy: layer3+4
      addresses:
        - "192.168.1.100/24"
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
        search: []
      routes:
        - to: "default"
          via: "192.168.1.254"
```

#### 5. 準備回滾腳本（安全措施）
```bash
cat > /tmp/rollback.sh << 'EOF'
#!/bin/bash
echo "執行網路設定回滾..."
sudo rm -f /etc/netplan/99-bonding.yaml
sudo netplan apply
echo "已回滾到原始設定"
EOF

chmod +x /tmp/rollback.sh
```

#### 6. 套用設定
```bash
# 檢查設定語法
sudo netplan --debug generate

# 套用設定（會短暫中斷網路）
sudo netplan apply
```

### 階段三：驗證設定

#### 7. 檢查設定狀態
```bash
# 檢查 bond0 介面
ip addr show bond0

# 檢查網路連線
ping 192.168.1.254

# 檢查 bonding 狀態
cat /proc/net/bonding/bond0

# 檢查路由表
ip route show
```

#### 8. 效能測試
```bash
# 安裝測試工具
sudo apt install iperf3

# 測試頻寬（需要遠端 iperf3 伺服器）
iperf3 -c [目標伺服器IP] -t 30
```

## 交換器設定

### 對於 Mode 4 (LACP)
交換器端需要設定：
1. 將對應的 4 個埠設定為 Port Channel
2. 啟用 802.3ad LACP
3. 設定 LACP 為 active 模式

**Cisco 範例：**
```
interface range GigabitEthernet0/1-4
 channel-group 1 mode active
 no shutdown
```

**HP/Aruba 範例：**
```
interface 1-4
 lacp active
```

## 故障排除

### 常見問題

#### 1. Bond 介面無法啟動
```bash
# 檢查錯誤訊息
dmesg | grep bond
journalctl -u systemd-networkd
```

#### 2. Mode 4 無法協商成功
- 確認交換器已設定 LACP
- 檢查網路線連接
- 確認交換器埠設定正確

#### 3. 頻寬沒有增加
- 確認使用正確的 bonding 模式
- 測試多個並行連線
- 檢查交換器設定

### 回滾步驟

如果遇到問題需要回滾：

```bash
# 方法 1：使用準備好的腳本
/tmp/rollback.sh

# 方法 2：手動回滾
sudo rm /etc/netplan/99-bonding.yaml
sudo cp /etc/netplan/backup/*.yaml /etc/netplan/
sudo netplan apply
```

## 監控和維護

### 定期檢查
```bash
# 檢查 bonding 狀態
cat /proc/net/bonding/bond0

# 檢查介面統計
ip -s link show bond0

# 檢查網路效能
sar -n DEV 1 5
```

### 日誌監控
```bash
# 監控 bonding 相關日誌
journalctl -f | grep bond
```

## 效能預期

| 模式 | 單一連線頻寬 | 多連線頻寬 | 容錯能力 |
|------|-------------|------------|----------|
| Mode 0 | 1Gbps | 2.5-3.5Gbps | 無 |
| Mode 4 | 1Gbps | 3-4Gbps | 有 |
| Mode 6 | 1Gbps | 2-3Gbps | 有 |

## 注意事項

1. **停機時間**：設定過程中會有 1-2 分鐘的網路中斷
2. **交換器相容性**：Mode 4 需要交換器支援 LACP
3. **效能測試**：實際效能會因網路環境而異
4. **備份重要性**：務必備份原始設定以便回滾

## 參考資料

- [Linux Kernel Bonding Documentation](https://www.kernel.org/doc/Documentation/networking/bonding.txt)
- [Ubuntu Netplan Documentation](https://netplan.io/)
- [IEEE 802.3ad Link Aggregation](https://standards.ieee.org/)

---

**建立日期**: 2025年8月
**版本**: 1.0
**適用系統**: Ubuntu 18.04+