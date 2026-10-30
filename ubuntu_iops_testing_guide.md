# Ubuntu IOPS 測試完整手冊

## 目錄
1. [安裝準備](#安裝準備)
2. [安裝 FIO 工具](#安裝-fio-工具)
3. [基本測試命令](#基本測試命令)
4. [詳細測試腳本](#詳細測試腳本)
5. [測試結果分析](#測試結果分析)
6. [進階測試選項](#進階測試選項)
7. [注意事項與建議](#注意事項與建議)

---

## 安裝準備

### 系統需求
- Ubuntu 18.04 或更高版本
- root 或 sudo 權限
- 充足的磁碟空間（建議至少 8GB 可用空間）

### 預先檢查
```bash
# 檢查系統版本
lsb_release -a

# 檢查可用磁碟空間
df -h

# 檢查記憶體使用情況
free -h

# 檢查磁碟資訊
lsblk
```

---

## 安裝 FIO 工具

### 更新系統套件
```bash
sudo apt-get update
sudo apt-get upgrade -y
```

### 安裝 FIO
```bash
sudo apt-get install fio -y
```

### 驗證安裝
```bash
# 檢查版本
fio --version

# 檢查說明
fio --help
```

---

## 基本測試命令

### 1. 隨機讀取測試
```bash
fio --name=randread \
    --ioengine=libaio \
    --iodepth=16 \
    --rw=randread \
    --bs=4k \
    --direct=1 \
    --size=4G \
    --numjobs=4 \
    --runtime=60 \
    --group_reporting
```

### 2. 隨機寫入測試
```bash
fio --name=randwrite \
    --ioengine=libaio \
    --iodepth=16 \
    --rw=randwrite \
    --bs=4k \
    --direct=1 \
    --size=4G \
    --numjobs=4 \
    --runtime=60 \
    --group_reporting
```

### 3. 順序讀取測試
```bash
fio --name=seqread \
    --ioengine=libaio \
    --iodepth=32 \
    --rw=read \
    --bs=1M \
    --direct=1 \
    --size=4G \
    --numjobs=1 \
    --runtime=60 \
    --group_reporting
```

### 4. 順序寫入測試
```bash
fio --name=seqwrite \
    --ioengine=libaio \
    --iodepth=32 \
    --rw=write \
    --bs=1M \
    --direct=1 \
    --size=4G \
    --numjobs=1 \
    --runtime=60 \
    --group_reporting
```

---

## 詳細測試腳本

### 完整測試腳本
建立一個測試腳本檔案：

```bash
# 建立測試腳本
nano iops_test.sh
```

腳本內容：
```bash
#!/bin/bash

# IOPS 測試腳本
echo "開始 IOPS 測試..."
echo "測試時間：$(date)"
echo "================================"

# 建立測試目錄
TEST_DIR="/tmp/fio_test"
mkdir -p $TEST_DIR

echo "1. 隨機讀取測試 (4K)"
fio --name=randread_4k \
    --directory=$TEST_DIR \
    --ioengine=libaio \
    --iodepth=16 \
    --rw=randread \
    --bs=4k \
    --direct=1 \
    --size=2G \
    --numjobs=4 \
    --runtime=60 \
    --group_reporting \
    --output-format=normal

echo ""
echo "2. 隨機寫入測試 (4K)"
fio --name=randwrite_4k \
    --directory=$TEST_DIR \
    --ioengine=libaio \
    --iodepth=16 \
    --rw=randwrite \
    --bs=4k \
    --direct=1 \
    --size=2G \
    --numjobs=4 \
    --runtime=60 \
    --group_reporting \
    --output-format=normal

echo ""
echo "3. 混合讀寫測試 (70%讀取/30%寫入)"
fio --name=randrw_7030 \
    --directory=$TEST_DIR \
    --ioengine=libaio \
    --iodepth=16 \
    --rw=randrw \
    --rwmixread=70 \
    --bs=4k \
    --direct=1 \
    --size=2G \
    --numjobs=4 \
    --runtime=60 \
    --group_reporting \
    --output-format=normal

echo ""
echo "4. 順序讀取測試 (1M)"
fio --name=seqread_1m \
    --directory=$TEST_DIR \
    --ioengine=libaio \
    --iodepth=32 \
    --rw=read \
    --bs=1M \
    --direct=1 \
    --size=4G \
    --numjobs=1 \
    --runtime=60 \
    --group_reporting \
    --output-format=normal

# 清理測試檔案
rm -rf $TEST_DIR

echo ""
echo "測試完成時間：$(date)"
echo "================================"
```

### 執行腳本
```bash
# 賦予執行權限
chmod +x iops_test.sh

# 執行測試
./iops_test.sh
```

---

## 測試結果分析

### 主要指標解讀

#### 1. IOPS (每秒輸入/輸出操作數)
- **SSD**: 通常 > 10,000 IOPS
- **HDD**: 通常 < 500 IOPS
- **NVMe SSD**: 可達 100,000+ IOPS

#### 2. 延遲 (Latency)
- **SSD**: < 1ms
- **HDD**: 5-15ms
- **NVMe SSD**: < 0.1ms

#### 3. 頻寬 (Bandwidth)
- 順序讀寫的傳輸速度
- SSD 通常 > 500 MB/s
- NVMe SSD 可達 3,000+ MB/s

### 結果範例分析

#### 高性能 SSD 範例：
```
IOPS=342k, BW=1335MiB/s
lat (usec): avg=185.15
```
- 極高的 IOPS
- 微秒級延遲
- 高頻寬

#### 傳統 HDD 範例：
```
IOPS=970, BW=3880KiB/s
lat (msec): avg=65.9
```
- 較低的 IOPS
- 毫秒級延遲
- 較低頻寬

---

## 進階測試選項

### 測試特定磁碟
```bash
# 測試特定磁碟 (注意：會破壞資料！)
fio --name=disk_test \
    --filename=/dev/sdb \
    --ioengine=libaio \
    --iodepth=16 \
    --rw=randread \
    --bs=4k \
    --direct=1 \
    --size=1G \
    --runtime=60
```

### 不同區塊大小測試
```bash
# 測試不同區塊大小
for bs in 4k 8k 16k 32k 64k 128k; do
    echo "測試區塊大小: $bs"
    fio --name=test_$bs \
        --ioengine=libaio \
        --iodepth=16 \
        --rw=randread \
        --bs=$bs \
        --direct=1 \
        --size=1G \
        --runtime=30 \
        --group_reporting
done
```

### 調整 I/O 深度
```bash
# 測試不同 I/O 深度
for depth in 1 4 8 16 32 64; do
    echo "測試 I/O 深度: $depth"
    fio --name=test_depth_$depth \
        --ioengine=libaio \
        --iodepth=$depth \
        --rw=randread \
        --bs=4k \
        --direct=1 \
        --size=1G \
        --runtime=30
done
```

### 輸出結果到檔案
```bash
# 將結果輸出到檔案
fio --name=test \
    --ioengine=libaio \
    --iodepth=16 \
    --rw=randread \
    --bs=4k \
    --direct=1 \
    --size=2G \
    --runtime=60 \
    --output=test_results.txt \
    --output-format=normal
```

---

## 注意事項與建議

### ⚠️ 重要警告
1. **資料安全**：測試可能會覆蓋資料，請在測試環境或備份後執行
2. **系統影響**：測試會佔用大量系統資源，建議在低負載時執行
3. **磁碟壽命**：頻繁的寫入測試可能影響 SSD 壽命

### 🔧 測試前準備
```bash
# 檢查磁碟健康狀態
sudo smartctl -a /dev/sda

# 清理檔案系統快取
sudo sync
echo 3 | sudo tee /proc/sys/vm/drop_caches

# 檢查系統負載
top
iostat 1 5
```

### 📊 效能基準參考

#### 隨機 4K 讀取 IOPS：
- **企業級 NVMe SSD**: 500,000+ IOPS
- **消費級 NVMe SSD**: 50,000-200,000 IOPS  
- **SATA SSD**: 10,000-90,000 IOPS
- **7200 RPM HDD**: 150-300 IOPS
- **5400 RPM HDD**: 75-150 IOPS

#### 順序讀取頻寬：
- **PCIe 4.0 NVMe**: 7,000+ MB/s
- **PCIe 3.0 NVMe**: 3,500+ MB/s
- **SATA SSD**: 550 MB/s
- **HDD**: 100-200 MB/s

### 🚀 優化建議
1. **SSD 優化**：
   - 啟用 TRIM：`sudo fstrim -av`
   - 檢查對齊：`sudo parted /dev/sda align-check optimal 1`

2. **系統優化**：
   - 調整 I/O 排程器：`echo noop | sudo tee /sys/block/sda/queue/scheduler`
   - 增加 I/O 佇列深度

3. **監控工具**：
   - 即時監控：`iostat -x 1`
   - 系統監控：`iotop`
   - 詳細分析：`blktrace`

---

## 故障排除

### 常見錯誤
1. **權限不足**：使用 `sudo` 執行
2. **空間不足**：檢查 `df -h`，清理或減少測試大小
3. **記憶體不足**：減少 `numjobs` 或 `iodepth`

### 驗證結果
```bash
# 多次測試確保一致性
for i in {1..3}; do
    echo "第 $i 次測試"
    fio --name=verify_test \
        --ioengine=libaio \
        --iodepth=16 \
        --rw=randread \
        --bs=4k \
        --direct=1 \
        --size=1G \
        --runtime=30
done
```

---

*最後更新：2025年2月*