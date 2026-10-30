# Manual

個人維運筆記與操作手冊，每份文件獨立成篇，記錄實際操作過的步驟與排查紀錄。

## 文件索引

### 雲端 / 帳號權限

| 文件 | 說明 |
|---|---|
| [GCP 組織授權外部 Gmail 帳號](gcp-external-gmail-iam-guide.md) | 組織政策（`iam.managed.allowedPolicyMembers`）放行外部 Gmail、常見錯誤排查、角色建議 |

### Ubuntu / 系統

| 文件 | 說明 |
|---|---|
| [Ubuntu 在 VM 中擴展磁碟空間 (LVM)](ubuntu-lvm-disk-expansion.md) | VMware 虛擬機中擴展 Ubuntu LVM 磁碟 |
| [Ubuntu 網路 Bonding 設定指南](bonding_setup_guide.md) | 多網路介面聚合，增加頻寬與冗餘 |
| [Ubuntu IOPS 測試完整手冊](ubuntu_iops_testing_guide.md) | 磁碟 IOPS 效能測試 |

### 資料庫

| 文件 | 說明 |
|---|---|
| [SQL Anywhere 10 記憶體定期釋放配置指南](SQL_Anywhere_Memory_Release_Guide.md) | 依硬體規格配置快取與定期釋放記憶體 |
| [SQL Server 伺服器名稱更改修復歷程](SQL_Server_Name_Change_Recovery.md) | 主機改名後 SQL Server 的修復紀錄 |

### CI/CD 與排程

| 文件 | 說明 |
|---|---|
| [GitLab Runner Ubuntu 安裝與註冊流程](gitlab_runner_setup.md) | 在 Ubuntu 安裝並註冊 GitLab Runner |
| [Airflow 3.x CLI 常用指令速查手冊](airflow_cli_cheatsheet.md) | Airflow CLI 常用指令速查 |

## 新增文件

1. 在根目錄新增 `.md` 檔，第一行用 `# 標題`。
2. 在上方「文件索引」對應分類加一列（沒有合適分類就新增一個）。
