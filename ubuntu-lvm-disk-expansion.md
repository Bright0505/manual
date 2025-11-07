# Ubuntu 在 VM 中擴展磁碟空間 (LVM)

本文檔說明如何在 VMware 虛擬機中為 Ubuntu 的 LVM 系統擴展磁碟空間。

## 前提條件

- Ubuntu 系統使用 LVM (Logical Volume Manager)
- 已在 VMware 中新增一顆硬碟 (例如: 100GB)
- 具有 root 或 sudo 權限

## 操作流程

### 步驟 1: 檢查目前磁碟狀態

```bash
# 列出所有磁碟裝置
ls /dev/sd*

# 查看所有磁碟資訊
sudo fdisk -l | grep "Disk /dev"
```

**預期輸出範例:**
```
/dev/sda  /dev/sda1  /dev/sda2  /dev/sda5  /dev/sdb
Disk /dev/sda: 512 GiB
Disk /dev/sdb: 100 GiB  ← 新增的硬碟
```

### 步驟 2: 使用 fdisk 建立分區

```bash
sudo fdisk /dev/sdb
```

**在 fdisk 互動介面中執行:**

```
Command (m for help): n        # 建立新分區
Partition type
Select (default p): p          # 選擇 primary 主分區
Partition number (1-4, default 1): 1    # 分區編號 1
First sector: [按 Enter]       # 使用預設起始位置
Last sector: [按 Enter]        # 使用預設結束位置 (最大)

Command (m for help): w        # 寫入變更並退出
```

**成功訊息:**
```
Created a new partition 1 of type 'Linux' and of size 100 GiB.
The partition table has been altered.
```

### 步驟 3: 將分區類型設定為 Linux LVM

```bash
sudo fdisk /dev/sdb
```

**在 fdisk 互動介面中執行:**

```
Command (m for help): t        # 改變分區類型
Selected partition 1
Partition type (type L to list all types): 8e    # Linux LVM 類型
Command (m for help): w        # 寫入變更並退出
```

**成功訊息:**
```
Changed type of partition 'Linux' to 'Linux LVM'.
```

### 步驟 4: 確認分區建立成功

```bash
ls /dev/sd*
```

**預期輸出:**
```
/dev/sda  /dev/sda1  /dev/sda2  /dev/sda5  /dev/sdb  /dev/sdb1  ← 新增的分區
```

### 步驟 5: 建立 Physical Volume (PV)

```bash
sudo pvcreate /dev/sdb1
```

**預期輸出:**
```
Physical volume "/dev/sdb1" successfully created
```

### 步驟 6: 擴展 Volume Group (VG)

```bash
# 將新的 PV 加入到現有的 VG (根據你的系統名稱調整)
sudo vgextend graylog-vg /dev/sdb1
```

**預期輸出:**
```
Volume group "graylog-vg" successfully extended
```

### 步驟 7: 確認 VG 擴展成功

```bash
sudo vgdisplay graylog-vg
```

**重要資訊:**
- **VG Size**: 總容量 (應該增加了 100 GiB)
- **Free PE / Size**: 可用空間 (應該顯示約 100 GiB)

**輸出範例:**
```
--- Volume group ---
VG Name               graylog-vg
VG Size               419.28 GiB      ← 增加了
Free PE / Size        25599 / 100.00 GiB  ← 可用空間
```

### 步驟 8: 擴展 Logical Volume (LV)

```bash
# 方式 1: 使用所有可用空間擴展根分區
sudo lvextend -l +100%FREE /dev/graylog-vg/root

# 方式 2: 指定增加的容量 (例如 100G)
# sudo lvextend -L +100G /dev/graylog-vg/root
```

**預期輸出:**
```
Size of logical volume graylog-vg/root changed from 318.6 GiB to 418.6 GiB.
Logical volume graylog-vg/root successfully resized.
```

### 步驟 9: 調整檔案系統大小

```bash
# 對於 ext4 檔案系統
sudo resize2fs /dev/graylog-vg/root

# 對於 XFS 檔案系統
# sudo xfs_growfs /
```

**預期輸出:**
```
Resizing the filesystem on /dev/graylog-vg/root to xxx blocks.
The filesystem on /dev/graylog-vg/root is now xxx blocks long.
```

### 步驟 10: 驗證結果

```bash
# 查看檔案系統使用狀況
df -h

# 查看完整磁碟結構
lsblk

# 查看 LVM 各層狀態
sudo pvs    # Physical Volume
sudo vgs    # Volume Group
sudo lvs    # Logical Volume
```

**預期 lsblk 輸出:**
```
NAME                   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda                      8:0    0   512G  0 disk
├─sda1                   8:1    0   487M  0 part /boot
├─sda2                   8:2    0     1K  0 part
└─sda5                   8:5    0 319.5G  0 part
  ├─graylog--vg-root   254:0    0 418.6G  0 lvm  /      ← 已擴展
  └─graylog--vg-swap_1 254:1    0   980M  0 lvm  [SWAP]
sdb                      8:16   0   100G  0 disk
└─sdb1                   8:17   0   100G  0 part
  └─graylog--vg-root   254:0    0 418.6G  0 lvm  /      ← 新增的硬碟
```

## 常見問題

### Q1: 為什麼不直接擴展原硬碟的分區?

**A:** 新增硬碟的方式更安全,原因如下:
- 不需要刪除或重建現有分區
- 零資料遺失風險
- 可以在系統運行時操作
- 如果需要,未來可以輕鬆移除

### Q2: 操作過程中需要重新開機嗎?

**A:** 不需要,整個過程都可以在系統運行時完成。

### Q3: 如何確認檔案系統類型?

```bash
df -T
```

常見類型: ext4, xfs

### Q4: 如果未來想移除 sdb 怎麼辦?

需要先將資料搬移:
```bash
sudo pvmove /dev/sdb1
sudo vgreduce graylog-vg /dev/sdb1
sudo pvremove /dev/sdb1
```

## 注意事項

1. ✅ 操作前建議先做 VM 快照或備份
2. ✅ 所有指令都需要 root 或 sudo 權限
3. ✅ 確認 VG 名稱 (使用 `vgs` 查看)
4. ✅ 確認 LV 名稱 (使用 `lvs` 查看)
5. ✅ 注意區分 /dev/sda 和 /dev/sdb

## 參考資料

- LVM 管理指令: pvs, vgs, lvs, pvdisplay, vgdisplay, lvdisplay
- 原文出處: https://billxu.net/blog/2022/07/25/ubuntu在vm中擴展ubuntu的磁碟空間lvm/

---

**文檔版本:** 1.0  
**最後更新:** 2025-11-07  
**適用系統:** Ubuntu 16.04+ with LVM
