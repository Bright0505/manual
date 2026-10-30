# Airflow 3.x CLI 常用指令速查手冊

## 一、DAG 管理 (`airflow dags`)

### 列出所有 DAG

```bash
airflow dags list
```

### 查看 DAG 詳細資訊

```bash
airflow dags details <dag_id>
```

### 暫停 / 恢復 DAG

```bash
# 暫停
airflow dags pause <dag_id>

# 恢復
airflow dags unpause <dag_id>
```

### 觸發 DAG 執行

```bash
# 基本觸發
airflow dags trigger <dag_id>

# 帶入自訂參數
airflow dags trigger <dag_id> -c '{"key":"value"}'
```

### 刪除 DAG

```bash
# 刪除指定 DAG 的所有資料庫記錄
airflow dags delete <dag_id>

# 跳過確認提示
airflow dags delete <dag_id> -y
```

> **注意：** 此指令只刪除資料庫中的 metadata（DAG runs、task instances 等）。若 DAG 的 `.py` 檔案仍留在 `DAGS_FOLDER`，Scheduler 會重新解析後讓 DAG 再次出現，只是歷史記錄會被清除。要完全移除 DAG，需同時刪除對應的 DAG 檔案。

### 其他常用子指令

| 指令 | 說明 |
|------|------|
| `airflow dags list-runs -d <dag_id>` | 列出指定 DAG 的所有執行記錄 |
| `airflow dags list-jobs` | 列出排程 jobs |
| `airflow dags list-import-errors` | 列出有 import 錯誤的 DAG |
| `airflow dags next-execution <dag_id>` | 查看下一次排程時間 |
| `airflow dags state <dag_id> <execution_date>` | 查看特定執行的狀態 |
| `airflow dags show <dag_id>` | 顯示 DAG 的 task 依賴圖 |
| `airflow dags reserialize` | 重新序列化 DAG |
| `airflow dags test <dag_id> <execution_date>` | 測試執行單一 DagRun |
| `airflow dags report` | 顯示 DagBag 載入報告 |

---

## 二、回填 (`airflow backfill`)

Airflow 3.x 已將 `airflow dags backfill` 移至 `airflow backfill create`。

### 基本語法

```bash
airflow backfill create --dag-id <dag_id> --from-date <start> --to-date <end>
```

### 參數說明

| 參數 | 說明 |
|------|------|
| `--dag-id` | **（必填）** 要回填的 DAG ID |
| `--from-date` | **（必填）** 回填起始日期（最早的 logical date） |
| `--to-date` | **（必填）** 回填結束日期（最晚的 logical date） |
| `--dry-run` | 只顯示會執行什麼，不實際執行 |
| `--reprocess-behavior` | 當該日期已有執行記錄時的處理方式：`none`（預設，不重跑）、`failed`（只重跑失敗的）、`completed`（重跑已完成的） |
| `--run-backwards` | 從最近的日期開始往回執行 |
| `--max-active-runs` | 限制同時執行的最大數量 |
| `--dag-run-conf` | 傳入 JSON 格式的 DAG run 設定 |
| `--run-on-latest-version` | （實驗性）使用最新版本的 DAG bundle 執行 |

### 實用範例（以 `daily_sync_data_sa390_1days` 為例）

```bash
# 基本回填（指定日期範圍）
airflow backfill create \
  --dag-id daily_sync_data_sa390_1days \
  --from-date 2025-03-01 \
  --to-date 2025-03-07

# 回填單一天
airflow backfill create \
  --dag-id daily_sync_data_sa390_1days \
  --from-date 2025-03-06 \
  --to-date 2025-03-06

# Dry run — 先看會跑什麼，不實際執行
airflow backfill create \
  --dag-id daily_sync_data_sa390_1days \
  --from-date 2025-03-01 \
  --to-date 2025-03-07 \
  --dry-run

# 只重跑失敗的
airflow backfill create \
  --dag-id daily_sync_data_sa390_1days \
  --from-date 2025-03-01 \
  --to-date 2025-03-07 \
  --reprocess-behavior failed

# 重跑所有已完成的（全部重跑）
airflow backfill create \
  --dag-id daily_sync_data_sa390_1days \
  --from-date 2025-03-01 \
  --to-date 2025-03-07 \
  --reprocess-behavior completed

# 從最近的日期開始往回跑
airflow backfill create \
  --dag-id daily_sync_data_sa390_1days \
  --from-date 2025-03-01 \
  --to-date 2025-03-07 \
  --run-backwards

# 限制同時最多跑 2 個
airflow backfill create \
  --dag-id daily_sync_data_sa390_1days \
  --from-date 2025-03-01 \
  --to-date 2025-03-07 \
  --max-active-runs 2

# 帶入自訂 config
airflow backfill create \
  --dag-id daily_sync_data_sa390_1days \
  --from-date 2025-03-01 \
  --to-date 2025-03-07 \
  --dag-run-conf '{"key":"value"}'
```

### 注意事項

- `--reprocess-behavior` 預設為 `none`，表示如果該日期已經有 DAG run 就**不會重跑**，需明確指定 `failed` 或 `completed` 才會重新處理。
- `--run-backwards` 不支援有 `depend_on_past` 設定的 task。
- 建議先用 `--dry-run` 確認回填範圍，再正式執行。
