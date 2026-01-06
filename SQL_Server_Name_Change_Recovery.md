# SQL Server 伺服器名稱更改修復歷程

## 📋 文件資訊
- **作業日期**: 2025-01-06
- **SQL Server 版本**: SQL Server 2017 Express (MSSQL14.SQLEXPRESS)
- **作業系統**: Windows Server
- **目標**: 將 SQL Server 實例名稱從 `BRIDGEFORAWS\SQLEXPRESS` 更改為 `BRIDGEFORDB\SQLEXPRESS`

---

## 🎯 問題背景

### 初始狀況
- Windows 主機名稱已成功從 `BRIDGEFORAWS` 更改為 `BRIDGEFORDB`
- SQL Server 內部的 `@@SERVERNAME` 仍顯示舊名稱 `BRIDGEFORAWS\SQLEXPRESS`
- 需要使用標準程序 `sp_dropserver` + `sp_addserver` 來更新伺服器名稱

### 遇到的主要障礙
執行 `sp_dropserver` 時出現錯誤：
```
訊息 20582，層級 16，狀態 1，程序 sp_MSrepl_check_server
無法卸除伺服器 'BRIDGEFORAWS\SQLEXPRESS'，因為該伺服器在複寫是作為發行者。
```

---

## 🔍 問題分析

### 根本原因
系統存在 **AWS DMS 複寫配置殘留**，導致：
1. SQL Server 被標記為「發行者」(Publisher)
2. 複寫檢查機制阻止 `sp_dropserver` 執行
3. `distribution` 資料庫已在之前被刪除，導致無法使用正常的複寫清理程序

### 技術細節
- **資料庫層級**: `poserp_ProductReplica` 的 `is_published` 標記已是 0
- **伺服器層級**: `sys.servers` 中 `is_publisher` 仍是 1（或相關元數據殘留）
- **核心矛盾**: 
  - 無法執行 `sp_dropdistributor` → 因為 distribution 不存在
  - 無法執行 `sp_adddistributor` → 因為系統認為已配置
  - 無法執行 `sp_dropserver` → 因為複寫檢查阻擋
  - 無法直接修改 `sys.servers` → SQL Server 2017+ 有系統表保護機制（錯誤 259）

---

## ❌ 嘗試過但失敗的方法

### 1. 直接使用 sp_replicationdboption
```sql
EXEC sp_replicationdboption 
    @dbname = 'poserp_ProductReplica',
    @optname = 'publish',
    @value = 'false';
```
**結果**: 失敗，因為 distribution 資料庫不存在

### 2. 直接更新系統表
```sql
UPDATE sys.databases SET is_published = 0;
UPDATE sys.servers SET is_publisher = 0;
```
**結果**: 錯誤 259 - SQL Server 2017+ 系統表保護

### 3. 使用追蹤標誌繞過
```sql
DBCC TRACEON(1224, 3608, 3609, -1);
```
**結果**: 無效，系統表保護無法繞過

### 4. EMERGENCY 模式 + allow updates
```sql
ALTER DATABASE master SET EMERGENCY;
EXEC sp_configure 'allow updates', 1;
```
**結果**: 仍被系統表保護阻擋

### 5. 直接執行 sp_dropdistributor
```sql
EXEC sp_dropdistributor @no_checks = 1, @ignore_distributor = 1;
```
**結果**: 錯誤 911 - 資料庫 'distribution' 不存在

---

## ✅ 最終成功的解決方案

### 核心策略
**「重建 distribution → 正常清理 → 更改名稱」**

由於 distribution 資料庫已被刪除但系統元數據仍認為伺服器是發行者，我們採用「恢復 distribution 備份 → 使用正常程序清理」的策略。

---

## 📝 完整執行步驟

### 階段 1: 準備工作

#### 1.1 確認當前狀態
```sql
-- 檢查資料庫複寫標記
SELECT 
    name,
    is_published,
    is_subscribed,
    is_merge_published
FROM sys.databases
WHERE name = 'poserp_ProductReplica';

-- 檢查伺服器複寫標記
SELECT 
    name,
    is_publisher,
    is_subscriber,
    is_distributor
FROM sys.servers
WHERE server_id = 0;

-- 檢查伺服器屬性
SELECT 
    SERVERPROPERTY('IsDistributor') AS IsDistributor,
    SERVERPROPERTY('IsPublisher') AS IsPublisher;
```

**檢查結果**:
- `poserp_ProductReplica`: is_published = 0 ✓
- `sys.servers`: is_publisher = 1 ✗
- 嘗試 `sp_dropserver` 出現錯誤 20582 ✗

#### 1.2 確認備份文件位置
```
E:\Backup\distribution.MDF.bak
E:\Backup\distribution.LDF.bak
E:\Backup\master.bak
```

---

### 階段 2: 恢復 distribution 資料庫

#### 2.1 停止 SQL Server 服務
```cmd
NET STOP "MSSQL$SQLEXPRESS"
```

#### 2.2 複製備份文件
```cmd
COPY "E:\Backup\distribution.MDF.bak" "C:\Program Files\Microsoft SQL Server\MSSQL14.SQLEXPRESS\MSSQL\Data\distribution.MDF"
COPY "E:\Backup\distribution.LDF.bak" "C:\Program Files\Microsoft SQL Server\MSSQL14.SQLEXPRESS\MSSQL\Data\distribution.LDF"
```

#### 2.3 啟動 SQL Server 服務
```cmd
NET START "MSSQL$SQLEXPRESS"
```

#### 2.4 附加 distribution 資料庫
```sql
USE master;
GO

-- 方法 A: 如果 LDF 完整
CREATE DATABASE distribution
ON (FILENAME = 'C:\Program Files\Microsoft SQL Server\MSSQL14.SQLEXPRESS\MSSQL\Data\distribution.MDF'),
   (FILENAME = 'C:\Program Files\Microsoft SQL Server\MSSQL14.SQLEXPRESS\MSSQL\Data\distribution.LDF')
FOR ATTACH;
GO

-- 方法 B: 如果 LDF 損壞或不完整，重建日誌
CREATE DATABASE distribution
ON (FILENAME = 'C:\Program Files\Microsoft SQL Server\MSSQL14.SQLEXPRESS\MSSQL\Data\distribution.MDF')
FOR ATTACH_REBUILD_LOG;
GO
```

#### 2.5 驗證 distribution 狀態
```sql
SELECT 
    name,
    state_desc,
    recovery_model_desc
FROM sys.databases 
WHERE name = 'distribution';
```

**預期結果**: `distribution | ONLINE | SIMPLE`

---

### 階段 3: 清理複寫配置

#### 3.1 檢查 distribution 資料庫內容
```sql
USE distribution;
GO

-- 檢查核心表
SELECT name FROM sys.tables
WHERE name IN ('MSdistribution_agents', 'MSdistribution_history', 'MSdistpublishers');

-- 檢查發行者註冊（如果表存在）
IF EXISTS (SELECT * FROM sys.tables WHERE name = 'MSdistpublishers')
    SELECT * FROM MSdistpublishers;
```

**實際情況**: 
- MSdistribution_agents, MSdistribution_history 存在
- MSdistpublishers 不存在（結構不完整）

#### 3.2 檢查 poserp_ProductReplica 的發行集
```sql
USE poserp_ProductReplica;
GO

IF EXISTS (SELECT * FROM sys.tables WHERE name = 'syspublications')
    SELECT * FROM syspublications;

SELECT 
    name,
    is_published,
    is_subscribed,
    is_merge_published
FROM sys.databases
WHERE name = 'poserp_ProductReplica';
```

**檢查結果**: 
- syspublications 不存在或為空
- is_published = 0

#### 3.3 嘗試移除分散式發佈者
```sql
USE master;
GO

EXEC sp_dropdistributor 
    @no_checks = 1,
    @ignore_distributor = 1;
GO
```

**結果**: 錯誤 21043 - "並未安裝散發者"
**分析**: 這是好消息！表示伺服器層級已不認為自己是分散式發佈者

#### 3.4 刪除 distribution 資料庫
```sql
USE master;
GO

DROP DATABASE distribution;
GO

-- 驗證
SELECT name FROM sys.databases WHERE name = 'distribution';
-- 應該沒有結果
```

#### 3.5 驗證清理完成
```sql
-- 檢查伺服器屬性
SELECT 
    SERVERPROPERTY('IsDistributor') AS IsDistributor,
    SERVERPROPERTY('IsPublisher') AS IsPublisher,
    SERVERPROPERTY('IsSubscriber') AS IsSubscriber;

-- 檢查 sys.servers
SELECT 
    name,
    is_publisher,
    is_subscriber,
    is_distributor
FROM sys.servers
WHERE server_id = 0;
```

**驗證結果**: 
- IsDistributor = 0 ✓
- IsPublisher = 0 ✓
- is_publisher = 0 ✓
- is_distributor = 0 ✓

---

### 階段 4: 更改伺服器名稱

#### 4.1 執行伺服器名稱變更
```sql
USE master;
GO

-- 刪除舊伺服器名稱
EXEC sp_dropserver 'BRIDGEFORAWS\SQLEXPRESS';
GO

-- 添加新伺服器名稱
EXEC sp_addserver 'BRIDGEFORDB\SQLEXPRESS', 'local';
GO
```

**執行結果**: Command(s) completed successfully. ✓

#### 4.2 驗證 sys.servers 變更
```sql
SELECT 
    server_id,
    name,
    is_publisher,
    is_subscriber,
    is_distributor
FROM sys.servers 
WHERE server_id = 0;
```

**預期結果**:
```
server_id: 0
name: BRIDGEFORDB\SQLEXPRESS
is_publisher: 0
is_subscriber: 0
is_distributor: 0
```

---

### 階段 5: 重啟服務與最終驗證

#### 5.1 重啟 SQL Server 服務
```cmd
REM 以管理員身份執行

NET STOP "MSSQL$SQLEXPRESS"
NET START "MSSQL$SQLEXPRESS"
```

⚠️ **重要**: 不重啟服務，`@@SERVERNAME` 不會更新

#### 5.2 最終驗證
```sql
-- 1. 檢查 @@SERVERNAME
SELECT @@SERVERNAME AS NewServerName;
-- 應該顯示: BRIDGEFORDB\SQLEXPRESS

-- 2. 檢查 sys.servers
SELECT 
    server_id,
    name,
    is_publisher,
    is_subscriber,
    is_distributor
FROM sys.servers 
WHERE server_id = 0;

-- 3. 檢查所有資料庫的複寫狀態
SELECT 
    name,
    is_published,
    is_subscribed,
    is_merge_published,
    is_distributor
FROM sys.databases
WHERE is_published = 1 
   OR is_subscribed = 1 
   OR is_merge_published = 1 
   OR is_distributor = 1;
-- 應該沒有任何結果

-- 4. 檢查伺服器屬性
SELECT 
    SERVERPROPERTY('ServerName') AS ServerName,
    SERVERPROPERTY('IsDistributor') AS IsDistributor,
    SERVERPROPERTY('IsPublisher') AS IsPublisher,
    SERVERPROPERTY('IsSubscriber') AS IsSubscriber;
```

**最終驗證結果**: 全部符合預期 ✓

---

## 🔑 關鍵學習點

### 1. 複寫配置的兩層結構
- **資料庫層級**: `sys.databases` 的 `is_published` 標記
- **伺服器層級**: `sys.servers` 的 `is_publisher` 標記
- **重要**: 兩層都必須清理，否則 `sp_dropserver` 會被阻擋

### 2. distribution 資料庫的特殊性
- distribution 不存在時，正常的複寫清理程序無法執行
- 但系統元數據仍可能認為伺服器是發行者
- **解決方法**: 恢復 distribution → 正常清理 → 刪除

### 3. SQL Server 2017+ 的系統表保護
- 無法直接 UPDATE sys.databases, sys.servers
- 追蹤標誌和 allow updates = 1 無效
- **唯一方法**: 使用官方預存程序或單一使用者模式

### 4. sp_dropdistributor 的行為
- 錯誤 911 (distribution 不存在) = 致命錯誤，需要恢復
- 錯誤 21043 (並未安裝散發者) = 好消息，表示已清理

### 5. 伺服器名稱變更的必要步驟
```
sp_dropserver → sp_addserver → 重啟服務
```
缺少任何一步，`@@SERVERNAME` 都不會更新

---

## 🛡️ 預防措施

### 未來如何避免此問題

#### 1. 正確的複寫卸載順序
```sql
-- 1. 先刪除訂閱
EXEC sp_dropsubscription ...

-- 2. 再刪除發行集
EXEC sp_droppublication ...

-- 3. 停用發行功能
EXEC sp_replicationdboption @optname = 'publish', @value = 'false'

-- 4. 移除發行者
EXEC sp_dropdistpublisher ...

-- 5. 最後移除分散式發佈者
EXEC sp_dropdistributor @no_checks = 1

-- 6. 確認 distribution 已刪除
DROP DATABASE distribution
```

#### 2. 變更伺服器名稱前的檢查清單
```sql
-- ✓ 檢查複寫狀態
SELECT * FROM sys.databases WHERE is_published = 1 OR is_subscribed = 1;
SELECT * FROM sys.servers WHERE is_publisher = 1 OR is_distributor = 1;

-- ✓ 檢查 distribution 資料庫
SELECT name FROM sys.databases WHERE name = 'distribution';

-- ✓ 檢查複寫作業
SELECT * FROM msdb.dbo.sysjobs WHERE category_id IN (
    SELECT category_id FROM msdb.dbo.syscategories WHERE name LIKE '%repl%'
);
```

#### 3. 備份策略
- ✓ 變更前完整備份 master 資料庫
- ✓ 記錄 distribution 資料庫的位置和大小
- ✓ 如需刪除 distribution，先完整備份

#### 4. AWS DMS 使用建議
- 使用完畢後，徹底清理複寫配置
- 不要手動刪除 distribution，使用 `sp_dropdistributor`
- 記錄所有發行集和訂閱的配置

---

## 🚨 故障排除指南

### 問題 1: 錯誤 20582 - 伺服器作為發行者
**原因**: 伺服器層級仍有複寫標記  
**解決**: 
1. 檢查 `sys.servers` 的 `is_publisher`
2. 恢復 distribution 資料庫
3. 使用 `sp_dropdistributor` 正常清理

### 問題 2: 錯誤 911 - distribution 不存在
**原因**: distribution 已刪除但元數據仍存在  
**解決**: 
1. 從備份恢復 distribution
2. 或使用單一使用者模式清理元數據

### 問題 3: 錯誤 259 - 無法更新系統表
**原因**: SQL Server 2017+ 系統表保護  
**解決**: 
1. 不要嘗試直接 UPDATE 系統表
2. 使用官方預存程序
3. 或進入單一使用者模式

### 問題 4: @@SERVERNAME 仍顯示舊名稱
**原因**: 未重啟服務  
**解決**: 
```cmd
NET STOP "MSSQL$SQLEXPRESS"
NET START "MSSQL$SQLEXPRESS"
```

---

## 📊 執行時間統計

| 階段 | 預估時間 | 實際時間 |
|------|----------|----------|
| 問題診斷與分析 | 30 分鐘 | 45 分鐘 |
| 嘗試各種解決方案 | 1 小時 | 2 小時 |
| 恢復 distribution | 10 分鐘 | 5 分鐘 |
| 清理複寫配置 | 15 分鐘 | 10 分鐘 |
| 更改伺服器名稱 | 5 分鐘 | 3 分鐘 |
| 重啟與驗證 | 5 分鐘 | 2 分鐘 |
| **總計** | **2 小時 5 分鐘** | **3 小時 5 分鐘** |

---

## 📚 參考資料

### Microsoft 官方文件
- [Rename a Computer that Hosts a Stand-Alone Instance of SQL Server](https://docs.microsoft.com/en-us/sql/database-engine/install-windows/rename-a-computer-that-hosts-a-stand-alone-instance-of-sql-server)
- [Disable Publishing and Distribution](https://docs.microsoft.com/en-us/sql/relational-databases/replication/disable-publishing-and-distribution)
- [sp_dropserver (Transact-SQL)](https://docs.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-dropserver-transact-sql)
- [sp_dropdistributor (Transact-SQL)](https://docs.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-dropdistributor-transact-sql)

### 相關技術文章
- SQL Server Replication Troubleshooting
- AWS DMS Best Practices
- System Table Protection in SQL Server 2017+

---

## 📝 變更記錄

| 日期 | 版本 | 變更內容 | 作者 |
|------|------|----------|------|
| 2025-01-06 | 1.0 | 初始版本 - 完整記錄修復過程 | Bright |

---

## ✅ 檢查清單

使用此文件時的檢查清單：

### 修復前
- [ ] 完整備份 master 資料庫
- [ ] 備份 distribution 資料庫（如果存在）
- [ ] 記錄當前 `@@SERVERNAME`
- [ ] 記錄所有複寫配置
- [ ] 確認沒有正在執行的複寫作業

### 修復中
- [ ] 確認 distribution 資料庫狀態
- [ ] 驗證 sys.servers 的複寫標記
- [ ] 驗證 sys.databases 的複寫標記
- [ ] 測試 sp_dropserver 是否可執行

### 修復後
- [ ] 驗證 `@@SERVERNAME` 顯示新名稱
- [ ] 驗證所有複寫標記為 0
- [ ] 測試應用程式連線
- [ ] 檢查 SQL Agent 作業是否正常
- [ ] 更新監控系統的伺服器名稱

---

## 🎯 結論

本次修復成功解決了因 AWS DMS 複寫配置殘留導致無法更改 SQL Server 伺服器名稱的問題。核心解決方案是「恢復 distribution 資料庫 → 正常清理複寫配置 → 更改伺服器名稱」，避免了使用單一使用者模式等高風險方法。

**關鍵成功因素**:
1. 保留了 distribution 資料庫的物理備份
2. 系統層級的複寫元數據清理完整
3. 遵循正確的清理順序

**未來建議**:
- 使用 AWS DMS 後徹底清理複寫配置
- 定期檢查複寫狀態，避免殘留
- 伺服器名稱變更前做好完整的檢查和備份

---

**文件結束**
