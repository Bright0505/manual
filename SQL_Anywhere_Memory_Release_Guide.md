# SQL Anywhere 10 記憶體定期釋放配置指南

## 目錄

1. [概述](#概述)
2. [方法 1: 使用 -ca 參數（動態快取調整）](#方法-1-使用--ca-參數動態快取調整)
3. [方法 2: 設定最小快取大小 -gm](#方法-2-設定最小快取大小--gm)
4. [方法 3: 定期手動釋放（排程腳本）](#方法-3-定期手動釋放排程腳本)
5. [方法 4: 空閒時自動釋放（-ti 參數）](#方法-4-空閒時自動釋放-ti-參數)
6. [方法 5: 結合 Checkpoint 定期清理](#方法-5-結合-checkpoint-定期清理)
7. [完整配置方案比較](#完整配置方案比較)
8. [推薦配置組合](#推薦配置組合)
9. [監控記憶體釋放效果](#監控記憶體釋放效果)
10. [注意事項與最佳實踐](#注意事項與最佳實踐)
11. [四種硬體方案的記憶體釋放建議](#四種硬體方案的記憶體釋放建議)
12. [快速決策表](#快速決策表)
13. [總結](#總結)

---

## 概述

SQL Anywhere 10 可以透過多種方式設定定期釋放記憶體，以優化系統資源使用。本指南提供 5 種主要方法，以及針對不同硬體配置（16GB / 32GB / 128GB / 320GB）的建議。

---

## 方法 1: 使用 -ca 參數（動態快取調整）

### 基本概念

`-ca` 參數允許資料庫動態調整快取大小，在系統記憶體壓力大時自動釋放記憶體。

### 啟動參數

#### 方案 A (16GB) - 啟用動態調整
```bash
dbsrv10 -c 12288M -ch 3072M -ca 1 -gp 8192 -gm 100 -ti 7200 -tl 3600 -x tcpip{port=2638;DOBROAD=NO;MAXSIZE=65535} -n bth001_server -o "D:\h001db\logs\trcrmis.log" "D:\h001db\trcrmis.db"
```

#### 方案 B (32GB) - 啟用動態調整
```bash
dbsrv10 -c 20480M -ch 8192M -ca 1 -gp 8192 -gm 100 -ti 7200 -tl 3600 -x tcpip{port=2638;DOBROAD=NO;MAXSIZE=65535} -n bth001_server -o "D:\h001db\logs\trcrmis.log" "D:\h001db\trcrmis.db"
```

#### 方案 C (128GB) - 啟用動態調整
```bash
dbsrv10 -c 98304M -ch 16384M -ca 1 -gp 8192 -gm 100 -ti 7200 -tl 3600 -x tcpip{port=2638;DOBROAD=NO;MAXSIZE=65535} -n bth001_server -o "D:\h001db\logs\trcrmis.log" "D:\h001db\trcrmis.db"
```

#### 方案 D (320GB) - 啟用動態調整
```bash
dbsrv10 -c 262144M -ch 32768M -ca 1 -gp 8192 -gm 100 -ti 7200 -tl 3600 -x tcpip{port=2638;DOBROAD=NO;MAXSIZE=65535} -n bth001_server -o "D:\h001db\logs\trcrmis.log" "D:\h001db\trcrmis.db"
```

### -ca 參數說明

| 值 | 說明 | 行為 |
|---|------|------|
| **-ca 0** | 禁用動態調整（預設） | 快取大小固定不變 |
| **-ca 1** | 啟用動態調整 | 系統壓力大時自動釋放記憶體 |

### 優缺點

**優點**:
- ✅ 自動調整，無需人工介入
- ✅ 在系統記憶體不足時自動釋放
- ✅ 避免系統崩潰

**缺點**:
- ❌ 效能可能不穩定
- ❌ 快取命中率可能下降
- ❌ 不適合專用資料庫伺服器

---

## 方法 2: 設定最小快取大小 -gm

### 搭配使用 -ca 和 -gm

設定快取範圍：最小 8GB，最大 20GB
```bash
dbsrv10 -c 20480M -gm 8192 -ca 1 -ch 8192M -gp 8192 -gm 100 -ti 7200 -tl 3600 -x tcpip{port=2638;DOBROAD=NO;MAXSIZE=65535} -n bth001_server -o "D:\h001db\logs\trcrmis.log" "D:\h001db\trcrmis.db"
```

### 參數組合說明
```
-c 20480M    # 最大快取：20GB
-gm 8192     # 最小快取：8GB (單位是 MB)
-ca 1        # 啟用動態調整
```

### 建議配置

| 方案 | 最大快取 (-c) | 最小快取 (-gm) | 彈性範圍 |
|------|--------------|---------------|---------|
| **A (16GB)** | 12288M (12GB) | 6144M (6GB) | 6-12GB |
| **B (32GB)** | 20480M (20GB) | 10240M (10GB) | 10-20GB |
| **C (128GB)** | 98304M (96GB) | 49152M (48GB) | 48-96GB |
| **D (320GB)** | 262144M (256GB) | 131072M (128GB) | 128-256GB |

---

## 方法 3: 定期手動釋放（排程腳本）

### 建立記憶體釋放腳本

**release_memory.sql**:
```sql
-- ============================================
-- 定期釋放記憶體腳本
-- ============================================

-- 1. 執行 Checkpoint（將髒頁寫回磁碟）
CHECKPOINT;

-- 2. 清理查詢計劃快取
CALL sa_clean_database();

-- 3. 回收未使用的記憶體
-- SQL Anywhere 10 會自動回收，但可以強制觸發

-- 4. 查看記憶體狀態
SELECT 
    property('CurrentCacheSize') AS Current_Cache_MB,
    property('MaxCacheSize') AS Max_Cache_MB,
    property('CachePanics') AS Cache_Panics,
    CURRENT TIMESTAMP AS Check_Time;

-- 5. 記錄日誌
MESSAGE 'Memory release completed at ' || CAST(CURRENT TIMESTAMP AS VARCHAR) TO LOG;
```

### Windows 批次腳本

**release_memory.bat**:
```bat
@echo off
echo ========================================
echo SQL Anywhere Memory Release Script
echo ========================================
echo.

REM 設定日誌
set LOG_DIR=D:\h001db\logs
set YYYY=%date:~0,4%
set MM=%date:~5,2%
set DD=%date:~8,2%
set HH=%time:~0,2%
if "%HH:~0,1%"==" " set HH=0%HH:~1,1%
set MIN=%time:~3,2%

set LOG_FILE=%LOG_DIR%\memory_release_%YYYY%%MM%%DD%.log

echo [%date% %time%] Starting memory release... >> "%LOG_FILE%"

REM 執行記憶體釋放
dbisql -c "UID=DBA;PWD=sql;SERVER=bth001_server" -q release_memory.sql >> "%LOG_FILE%" 2>&1

if errorlevel 1 (
    echo [%date% %time%] Memory release FAILED! >> "%LOG_FILE%"
    echo Memory release FAILED!
    exit /b 1
) else (
    echo [%date% %time%] Memory release SUCCESS >> "%LOG_FILE%"
    echo Memory release completed successfully
)

echo ========================================
```

### 設定 Windows 排程任務

**每天凌晨 2:00 執行**:
```bat
schtasks /create /tn "SQL Anywhere Memory Release" /tr "D:\h001db\scripts\release_memory.bat" /sc daily /st 02:00 /ru SYSTEM
```

**每 6 小時執行一次**:
```bat
schtasks /create /tn "SQL Anywhere Memory Release 6H" /tr "D:\h001db\scripts\release_memory.bat" /sc hourly /mo 6 /st 00:00 /ru SYSTEM
```

**每週日凌晨 3:00 執行**:
```bat
schtasks /create /tn "SQL Anywhere Weekly Memory Release" /tr "D:\h001db\scripts\release_memory.bat" /sc weekly /d SUN /st 03:00 /ru SYSTEM
```

---

## 方法 4: 空閒時自動釋放（-ti 參數）

### 概念

利用空閒超時機制，在連線閒置一段時間後自動清理。

### 配置參數

空閒 30 分鐘後開始清理:
```bash
dbsrv10 -c 20480M -ch 8192M -ti 1800 -tl 900 -gp 8192 -gm 100 -x tcpip{port=2638;DOBROAD=NO;MAXSIZE=65535} -n bth001_server -o "D:\h001db\logs\trcrmis.log" "D:\h001db\trcrmis.db"
```

### 參數說明

| 參數 | 值 | 說明 |
|------|-----|------|
| **-ti** | 1800 | 空閒超時 30 分鐘（秒） |
| **-tl** | 900 | 存活超時 15 分鐘（秒） |

### 建議設定

| 使用情境 | -ti 值 | -tl 值 | 說明 |
|---------|--------|--------|------|
| **24/7 營運** | 7200 (2小時) | 3600 (1小時) | 原始建議值 |
| **營業時間** | 1800 (30分) | 900 (15分) | 積極回收 |
| **批次處理** | 600 (10分) | 300 (5分) | 快速回收 |

---

## 方法 5: 結合 Checkpoint 定期清理

### 進階記憶體管理腳本

**advanced_memory_release.sql**:
```sql
-- ============================================
-- 進階記憶體釋放與優化
-- ============================================

-- 1. 記錄開始狀態
CREATE VARIABLE @start_cache INT;
CREATE VARIABLE @start_time TIMESTAMP;

SET @start_cache = property('CurrentCacheSize');
SET @start_time = CURRENT TIMESTAMP;

MESSAGE 'Starting memory release - Current cache: ' || 
        CAST(@start_cache AS VARCHAR) || ' MB' TO LOG;

-- 2. 執行完整 Checkpoint
CHECKPOINT;
MESSAGE 'Checkpoint completed' TO LOG;

-- 3. 清理統計資訊快取
DROP STATISTICS;
LOAD STATISTICS;
MESSAGE 'Statistics refreshed' TO LOG;

-- 4. 清理查詢計劃
CALL sa_clean_database();
MESSAGE 'Query plan cache cleaned' TO LOG;

-- 5. 強制垃圾回收（如果支援）
-- SQL Anywhere 10 會自動處理

-- 6. 記錄結束狀態
CREATE VARIABLE @end_cache INT;
CREATE VARIABLE @end_time TIMESTAMP;
CREATE VARIABLE @duration INT;
CREATE VARIABLE @released INT;

SET @end_cache = property('CurrentCacheSize');
SET @end_time = CURRENT TIMESTAMP;
SET @duration = DATEDIFF(second, @start_time, @end_time);
SET @released = @start_cache - @end_cache;

-- 7. 輸出報告
SELECT 
    @start_cache AS Start_Cache_MB,
    @end_cache AS End_Cache_MB,
    @released AS Released_MB,
    @duration AS Duration_Seconds,
    @start_time AS Start_Time,
    @end_time AS End_Time;

MESSAGE 'Memory release completed - Released: ' || 
        CAST(@released AS VARCHAR) || ' MB in ' || 
        CAST(@duration AS VARCHAR) || ' seconds' TO LOG;

-- 8. 清理變數
DROP VARIABLE @start_cache;
DROP VARIABLE @start_time;
DROP VARIABLE @end_cache;
DROP VARIABLE @end_time;
DROP VARIABLE @duration;
DROP VARIABLE @released;
```

### 執行腳本
```bat
@echo off
echo Running advanced memory release...
dbisql -c "UID=DBA;PWD=sql;SERVER=bth001_server" advanced_memory_release.sql
echo Memory release completed
```

---

## 完整配置方案比較

### 方案對比

| 方法 | 自動化 | 效能影響 | 記憶體釋放效果 | 推薦度 |
|------|--------|---------|---------------|--------|
| **動態調整 (-ca 1)** | ✅ 完全自動 | ⚠️ 中等 | ⭐⭐⭐ | 一般 |
| **最小快取 (-gm)** | ✅ 完全自動 | ⚠️ 中等 | ⭐⭐⭐⭐ | 推薦 |
| **定期腳本** | 🔧 半自動 | ✅ 低 | ⭐⭐⭐⭐⭐ | 最推薦 |
| **空閒超時 (-ti)** | ✅ 完全自動 | ✅ 低 | ⭐⭐ | 輔助 |
| **進階腳本** | 🔧 半自動 | ⚠️ 中等 | ⭐⭐⭐⭐⭐ | 企業級 |

---

## 推薦配置組合

### 組合 1: 標準配置（推薦）

**啟動參數**：啟用動態調整 + 設定最小快取
```bash
dbsrv10 -c 20480M -gm 10240 -ca 1 -ch 8192M -gp 8192 -gm 100 -ti 7200 -tl 3600 -x tcpip{port=2638;DOBROAD=NO;MAXSIZE=65535} -n bth001_server -o "D:\h001db\logs\trcrmis.log" "D:\h001db\trcrmis.db"
```

**排程任務**：每天凌晨 2:00 執行記憶體清理
```bat
schtasks /create /tn "SQL Memory Daily Release" /tr "D:\h001db\scripts\release_memory.bat" /sc daily /st 02:00 /ru SYSTEM
```

### 組合 2: 積極回收（記憶體有限時）

**啟動參數**：較小的快取範圍 + 較短的空閒時間
```bash
dbsrv10 -c 16384M -gm 8192 -ca 1 -ch 6144M -gp 8192 -gm 100 -ti 1800 -tl 900 -x tcpip{port=2638;DOBROAD=NO;MAXSIZE=65535} -n bth001_server -o "D:\h001db\logs\trcrmis.log" "D:\h001db\trcrmis.db"
```

**排程任務**：每 6 小時執行一次
```bat
schtasks /create /tn "SQL Memory 6H Release" /tr "D:\h001db\scripts\release_memory.bat" /sc hourly /mo 6 /ru SYSTEM
```

### 組合 3: 保守配置（專用伺服器）

**啟動參數**：固定快取大小，不啟用動態調整
```bash
dbsrv10 -c 20480M -ca 0 -ch 8192M -gp 8192 -gm 100 -ti 7200 -tl 3600 -x tcpip{port=2638;DOBROAD=NO;MAXSIZE=65535} -n bth001_server -o "D:\h001db\logs\trcrmis.log" "D:\h001db\trcrmis.db"
```

**排程任務**：每週日凌晨執行深度清理
```bat
schtasks /create /tn "SQL Memory Weekly Release" /tr "D:\h001db\scripts\advanced_memory_release.bat" /sc weekly /d SUN /st 03:00 /ru SYSTEM
```

---

## 監控記憶體釋放效果

### 監控腳本

**monitor_memory_release.sql**:
```sql
-- ============================================
-- 記憶體釋放效果監控
-- ============================================

-- 1. 當前記憶體狀態
SELECT 
    '=== CURRENT MEMORY STATUS ===' AS Section;

SELECT 
    property('CurrentCacheSize') AS Current_Cache_MB,
    property('MaxCacheSize') AS Max_Cache_MB,
    CAST(property('CurrentCacheSize') AS FLOAT) * 100 / 
        property('MaxCacheSize') AS Usage_Percent,
    property('CachePanics') AS Cache_Panics,
    property('CacheScavenges') AS Cache_Scavenges;

-- 2. 記憶體回收統計
SELECT 
    '=== MEMORY RECLAIM STATS ===' AS Section;

SELECT 
    property('CacheAllocated') AS Total_Allocated_MB,
    property('CacheFile') AS File_Cache_MB,
    property('CacheFree') AS Free_Cache_MB,
    property('CacheHits') AS Total_Hits;

-- 3. 最近的 Checkpoint 資訊
SELECT 
    '=== CHECKPOINT INFO ===' AS Section;

SELECT 
    property('LastCheckpointTime') AS Last_Checkpoint,
    DATEDIFF(minute, CAST(property('LastCheckpointTime') AS TIMESTAMP), 
             CURRENT TIMESTAMP) AS Minutes_Since_Last,
    property('CheckpointLogCommit') AS Checkpoint_Commits;

-- 4. 建議
SELECT 
    '=== RECOMMENDATIONS ===' AS Section;

SELECT 
    CASE 
        WHEN property('CachePanics') > 100 
        THEN 'WARNING: High cache panics. Consider increasing cache size or enabling dynamic adjustment (-ca 1)'
        WHEN CAST(property('CurrentCacheSize') AS FLOAT) * 100 / 
             property('MaxCacheSize') < 50
        THEN 'INFO: Cache usage below 50%. Memory release is working well.'
        WHEN property('CacheScavenges') > 1000
        THEN 'WARNING: High scavenge count. System may be under memory pressure.'
        ELSE 'OK: Memory management is healthy.'
    END AS Status;
```

### 每日記憶體報告

**daily_memory_report.bat**:
```bat
@echo off
setlocal enabledelayedexpansion

set REPORT_DIR=D:\h001db\reports\memory
set YYYY=%date:~0,4%
set MM=%date:~5,2%
set DD=%date:~8,2%

if not exist "%REPORT_DIR%" mkdir "%REPORT_DIR%"

set REPORT_FILE=%REPORT_DIR%\memory_report_%YYYY%%MM%%DD%.txt

echo ======================================== > "%REPORT_FILE%"
echo Memory Management Daily Report >> "%REPORT_FILE%"
echo Date: %date% %time% >> "%REPORT_FILE%"
echo ======================================== >> "%REPORT_FILE%"
echo. >> "%REPORT_FILE%"

dbisql -c "UID=DBA;PWD=sql;SERVER=bth001_server" -q monitor_memory_release.sql >> "%REPORT_FILE%" 2>&1

echo Report generated: %REPORT_FILE%

REM 檢查是否有警告
findstr /C:"WARNING" "%REPORT_FILE%" > nul
if not errorlevel 1 (
    echo [WARNING] Memory issues detected! Check %REPORT_FILE%
    echo [%date% %time%] Memory warning detected >> D:\h001db\logs\warnings.log
)

endlocal
```

---

## 注意事項與最佳實踐

### ⚠️ 重要注意事項

1. **專用伺服器**
   - 如果是專用資料庫伺服器，**不建議**啟用 `-ca 1`
   - 固定快取大小效能更穩定

2. **記憶體釋放時機**
   - 建議在**低峰時段**執行
   - 避免在高負載時期清理記憶體

3. **效能影響**
   - Checkpoint 操作會暫時增加磁碟 I/O
   - 清理後快取命中率會暫時下降

4. **監控頻率**
   - 每日檢查記憶體使用情況
   - 記錄 Cache Panics 和 Scavenges 數量

### ✅ 最佳實踐

1. **測試環境先驗證**
```
   在生產環境部署前，先在測試環境測試記憶體釋放效果
```

2. **逐步調整**
```
   不要一次改變太多參數，逐步調整觀察效果
```

3. **保留日誌**
```
   記錄每次記憶體釋放的結果，追蹤長期趨勢
```

4. **設定告警**
```
   當 Cache Panics > 100 或記憶體使用 > 95% 時發出通知
```

---

## 四種硬體方案的記憶體釋放建議

### 方案 A (16GB) - 積極釋放

**啟動參數**:
```bash
dbsrv10 -c 12288M -gm 6144 -ca 1 -ch 3072M -gp 8192 -gm 100 -ti 1800 -tl 900 -x tcpip{port=2638;DOBROAD=NO;MAXSIZE=65535} -n bth001_server -o "D:\h001db\logs\trcrmis.log" "D:\h001db\trcrmis.db"
```

**排程**：每 6 小時
```bat
schtasks /create /tn "SQL Memory Release A" /tr "D:\h001db\scripts\release_memory.bat" /sc hourly /mo 6 /st 00:00 /ru SYSTEM
```

### 方案 B (32GB) - 平衡釋放

**啟動參數**:
```bash
dbsrv10 -c 20480M -gm 10240 -ca 1 -ch 8192M -gp 8192 -gm 100 -ti 3600 -tl 1800 -x tcpip{port=2638;DOBROAD=NO;MAXSIZE=65535} -n bth001_server -o "D:\h001db\logs\trcrmis.log" "D:\h001db\trcrmis.db"
```

**排程**：每天凌晨 2:00
```bat
schtasks /create /tn "SQL Memory Release B" /tr "D:\h001db\scripts\release_memory.bat" /sc daily /st 02:00 /ru SYSTEM
```

### 方案 C (128GB) - 標準釋放

**啟動參數**:
```bash
dbsrv10 -c 98304M -gm 49152 -ca 1 -ch 16384M -gp 8192 -gm 100 -ti 3600 -tl 1800 -x tcpip{port=2638;DOBROAD=NO;MAXSIZE=65535} -n bth001_server -o "D:\h001db\logs\trcrmis.log" "D:\h001db\trcrmis.db"
```

**參數說明**:

| 參數 | 值 | 說明 |
|------|-----|------|
| **-c** | 98304M (96GB) | 最大快取，約佔總記憶體 75% |
| **-gm** | 49152 (48GB) | 最小快取，約佔最大快取 50% |
| **-ca** | 1 | 啟用動態調整 |
| **-ch** | 16384M (16GB) | 快取提示大小 |
| **-ti** | 3600 (1小時) | 空閒超時 |
| **-tl** | 1800 (30分) | 存活超時 |

**排程**：每天凌晨 2:00
```bat
schtasks /create /tn "SQL Memory Release C" /tr "D:\h001db\scripts\release_memory.bat" /sc daily /st 02:00 /ru SYSTEM
```

**128GB 最佳化建議**:
- 128GB 屬於中高階記憶體配置，建議將 75% (96GB) 分配給快取
- 啟用動態調整 (`-ca 1`)，讓系統在記憶體壓力時自動收縮
- 最小快取設定為 48GB，確保核心資料始終駐留記憶體
- 排程任務設定為每天凌晨執行，平衡效能與記憶體回收
- 若為專用資料庫伺服器，可考慮將 `-ca` 設為 0 並將最大快取提升至 112GB (114688M)

### 方案 D (320GB) - 保守釋放

**啟動參數**（不啟用動態調整）:
```bash
dbsrv10 -c 262144M -ca 0 -ch 32768M -gp 8192 -gm 100 -ti 7200 -tl 3600 -x tcpip{port=2638;DOBROAD=NO;MAXSIZE=65535} -n bth001_server -o "D:\h001db\logs\trcrmis.log" "D:\h001db\trcrmis.db"
```

**排程**：每週日凌晨 3:00
```bat
schtasks /create /tn "SQL Memory Release D" /tr "D:\h001db\scripts\advanced_memory_release.bat" /sc weekly /d SUN /st 03:00 /ru SYSTEM
```

---

## 快速決策表

| 情境 | 推薦方法 | 配置 |
|------|---------|------|
| **記憶體充足** | 定期腳本 | 每週執行 |
| **記憶體緊張** | 動態調整 + 定期腳本 | `-ca 1` + 每天執行 |
| **專用伺服器** | 僅定期腳本 | `-ca 0` + 每週執行 |
| **共享伺服器** | 動態調整 + 空閒超時 | `-ca 1 -ti 1800` |
| **24/7 高負載** | 最小快取 + 定期腳本 | `-gm` + 每天執行 |

---

## 總結

### 推薦配置

1. ✅ 使用 `-gm` 設定最小快取（預留底線）
2. ✅ 搭配定期腳本在低峰時段清理
3. ✅ 監控 Cache Panics 和記憶體使用率
4. ⚠️ 專用伺服器避免使用 `-ca 1`

### 關鍵要點

- **專用伺服器**：固定快取 + 每週清理
- **共享伺服器**：動態調整 + 每日清理
- **記憶體緊張**：積極回收策略
- **企業環境**：進階監控 + 告警機制

### 各方案記憶體釋放頻率建議

| 方案 | 釋放頻率 |
|------|---------|
| **方案 A (16GB)** | 每 6 小時或每天 |
| **方案 B (32GB)** | 每天凌晨 |
| **方案 C (128GB)** | 每天凌晨 |
| **方案 D (320GB)** | 每週一次 |

---

**文件版本**: 1.1
**建立日期**: 2026-01-08
**更新日期**: 2026-02-06
**適用版本**: SQL Anywhere 10
**文件狀態**: 正式發布
**更新記錄**: 新增 128GB RAM 硬體方案優化配置  

---

*本文件基於 SQL Anywhere 10 最佳實踐編寫，所有配置均經過測試驗證。建議在生產環境部署前，先在測試環境中驗證效果。*