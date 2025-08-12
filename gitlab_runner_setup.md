# GitLab Runner Ubuntu 安裝與註冊流程

## 環境需求
- Ubuntu 18.04 或更新版本
- 具備 sudo 權限的使用者帳號
- 網路連線可存取 GitLab 實例

## 步驟一：安裝 GitLab Runner

### 方法 1：使用官方 APT 倉庫（推薦）

```bash
# 1. 下載並安裝 GitLab 官方 GPG 金鑰
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash

# 2. 更新套件列表並安裝 GitLab Runner
sudo apt-get update
sudo apt-get install gitlab-runner

# 3. 驗證安裝
gitlab-runner --version
```

### 方法 2：手動下載安裝

```bash
# 1. 下載最新版本的 deb 套件
curl -LJO "https://gitlab-runner-downloads.s3.amazonaws.com/latest/deb/gitlab-runner_amd64.deb"

# 2. 安裝套件
sudo dpkg -i gitlab-runner_amd64.deb

# 3. 如果有相依性問題，執行修復
sudo apt-get install -f
```

## 步驟二：準備註冊資訊

在 GitLab 介面中獲取以下資訊：

### 取得 Registration Token 路徑：
- **專案層級**：專案 → Settings → CI/CD → Runners → Expand → Project runners
- **群組層級**：群組 → Settings → CI/CD → Runners → Expand
- **實例層級**：Admin Area → Overview → Runners （需要管理員權限）

### 需要記錄的資訊：
- [ ] GitLab URL：`https://gitlab.example.com` 或 `https://gitlab.com`
- [ ] Registration Token：`glrt-xxxxxxxxxxxxxxxx`
- [ ] Runner 描述：`ubuntu-runner-01`
- [ ] Runner Tags：`linux, ubuntu, docker`

## 步驟三：註冊 Runner

### 互動式註冊

```bash
# 執行註冊命令
sudo gitlab-runner register
```

### 註冊過程中的輸入項目：

```
Enter the GitLab instance URL (for example, https://gitlab.com/):
→ 輸入：https://gitlab.com

Enter the registration token:
→ 輸入：glrt-xxxxxxxxxxxxxxxx

Enter a description for the runner:
→ 輸入：ubuntu-runner-01

Enter tags for the runner (comma-separated):
→ 輸入：linux,ubuntu,docker

Enter optional maintenance note for the runner:
→ 輸入：（可選，按 Enter 跳過）

Registering runner... succeeded                     runner=xxxxxxxx

Enter an executor:
→ 輸入：docker

Enter the default Docker image (for example, ruby:2.7):
→ 輸入：ubuntu:20.04
```

### 非互動式註冊（適合自動化腳本）

```bash
sudo gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.com" \
  --registration-token "glrt-xxxxxxxxxxxxxxxx" \
  --executor "docker" \
  --docker-image "ubuntu:20.04" \
  --description "ubuntu-runner-01" \
  --tag-list "linux,ubuntu,docker" \
  --run-untagged="true" \
  --locked="false" \
  --access-level="not_protected"
```

## 步驟四：設定執行器類型

### Docker 執行器（推薦）
- **適用場景**：隔離性好，支援多種環境
- **預設映像檔**：`ubuntu:20.04`、`alpine:latest`、`node:16`

### Shell 執行器
- **適用場景**：直接在主機上執行，性能較好
- **注意事項**：需要手動安裝所需的工具和相依性

### 其他執行器
- `kubernetes`：在 K8s 叢集中執行
- `docker+machine`：動態建立 Docker 主機

## 步驟五：啟動與驗證服務

```bash
# 1. 啟動 GitLab Runner 服務
sudo gitlab-runner start

# 2. 設定開機自動啟動
sudo systemctl enable gitlab-runner

# 3. 檢查服務狀態
sudo systemctl status gitlab-runner

# 4. 驗證 Runner 註冊狀態
sudo gitlab-runner verify

# 5. 列出已註冊的 Runner
sudo gitlab-runner list
```

## 步驟六：驗證 Runner 可用性

### 在 GitLab 介面確認：
1. 前往專案 → Settings → CI/CD → Runners
2. 確認 Runner 顯示綠色圓點（表示 online）
3. 檢查 Runner 的 Tags 和描述是否正確

### 建立測試 Pipeline：
建立 `.gitlab-ci.yml` 檔案測試：

```yaml
test_job:
  stage: test
  tags:
    - linux
  script:
    - echo "Hello from GitLab Runner!"
    - whoami
    - pwd
    - ls -la
```

## 常用管理命令

```bash
# 檢視 Runner 狀態
sudo gitlab-runner status

# 重啟 Runner
sudo gitlab-runner restart

# 停止 Runner
sudo gitlab-runner stop

# 檢視設定檔
sudo cat /etc/gitlab-runner/config.toml

# 重新載入設定
sudo gitlab-runner reload

# 取消註冊 Runner
sudo gitlab-runner unregister --url "https://gitlab.com" --token "RUNNER_TOKEN"

# 檢視 Runner 日誌
sudo journalctl -u gitlab-runner -f
```

## 疑難排解

### 常見問題與解決方案：

#### 1. Runner 顯示離線
```bash
# 檢查網路連線
curl -I https://gitlab.com

# 重啟服務
sudo systemctl restart gitlab-runner

# 檢查防火牆設定
sudo ufw status
```

#### 2. Docker 執行器無法正常運作
```bash
# 安裝 Docker（如果尚未安裝）
sudo apt-get update
sudo apt-get install docker.io

# 將 gitlab-runner 使用者加入 docker 群組
sudo usermod -aG docker gitlab-runner

# 重啟服務
sudo systemctl restart gitlab-runner
```

#### 3. 權限問題
```bash
# 檢查 gitlab-runner 使用者權限
sudo -u gitlab-runner whoami

# 修改檔案擁有者
sudo chown -R gitlab-runner:gitlab-runner /home/gitlab-runner
```

## 安全性考量

- [ ] 定期更新 GitLab Runner 至最新版本
- [ ] 使用專用的 Runner，避免在生產環境主機上安裝
- [ ] 設定適當的 Tags，控制 Pipeline 執行範圍
- [ ] 定期輪換 Registration Token
- [ ] 監控 Runner 的資源使用情況

## 更新 GitLab Runner

```bash
# 使用 APT 更新
sudo apt-get update
sudo apt-get install gitlab-runner

# 檢查版本
gitlab-runner --version

# 重新載入服務
sudo systemctl daemon-reload
sudo systemctl restart gitlab-runner
```

---

## 檢查清單

安裝完成後請確認：

- [ ] GitLab Runner 服務正常運作
- [ ] Runner 在 GitLab 介面顯示為 online
- [ ] 測試 Pipeline 可以正常執行
- [ ] Runner Tags 設定正確
- [ ] 相關日誌沒有錯誤訊息
- [ ] 開機自動啟動已設定

---

*最後更新日期：2025-08-12*
*適用版本：GitLab Runner 16.x 以上*