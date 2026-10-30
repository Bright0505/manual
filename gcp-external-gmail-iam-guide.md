# GCP 組織授權外部 Gmail 帳號：問題排查與操作教學

> 環境：組織 `<組織網域>`（Org ID `<ORG_ID>`）、專案 `<PROJECT_ID>`

---

## 1. 背景

需求：將外部個人帳號（`@gmail.com`）加入組織 / 專案的 IAM，例如：

- `<外部帳號A>@gmail.com` → 組織管理員
- `<外部帳號B>@gmail.com` → 專案權限

組織原本啟用了 **Domain restricted sharing**（`iam.allowedPolicyMemberDomains`），只允許自家 Customer ID 的帳號，所以外部 Gmail 一律被擋。

---

## 2. 遇到的錯誤與原因

| # | 錯誤訊息 | 原因 | 解法 |
|---|---|---|---|
| 1 | `The policy contains invalid list value(s): [ is set]` | 在 `iam.allowedPolicyMemberDomains` 選了「**全部拒絕**」。此限制不支援 Deny All。 | 不要用全部拒絕，只能用「自訂 → 允許 → Customer ID」。 |
| 2 | `One or more values is invalid.` | 同一個限制選了「自訂 → **拒絕**」且值為空；另有一條「全部拒絕」規則。此限制只接受**允許**且必須有值。 | 改成「允許」並填 Customer ID（`C0xxxxxxx`），刪除多餘規則。 |
| 3 | 授予 Owner 時 `Precondition check failed.` | **Owner (Legacy)** 不能直接綁給組織外帳號（需走邀請流程）。 | 改給 `Editor` + `Project IAM Admin` 等角色；或讓對方用組織帳號。 |
| 4 | `已遭到組織政策管理員禁止 … constraints/iam.managed.allowedPolicyMembers` | 新政策已生效，但該帳號不在 `allowedMemberSubjects` 允許清單。 | 把 `user:<email>` 加進允許清單（見第 4 節）。 |
| 5 | 帳號看不到組織 | ① 登入的不是被授權的那個帳號（Chrome 多帳號）② 資源選擇器停在「最近」③ 所在專案屬於「無組織」④ 權限傳播延遲 | 確認右上角帳號 email、選擇器切「全部」或搜尋 `<組織網域>`、重新整理。 |

---

## 3. 最終架構（一次性設定）

改用新版 managed constraint 取代舊的網域限制，可以**逐一允許外部帳號**：

| 限制 | 層級 | 設定 |
|---|---|---|
| `iam.managed.allowedPolicyMembers` | 組織 `<組織網域>` | **強制執行：啟用** |
| ↳ `allowedPrincipalSets` | | `//cloudresourcemanager.googleapis.com/organizations/<ORG_ID>` |
| ↳ `allowedMemberSubjects` | | `user:<外部帳號A>@gmail.com`<br>`user:<外部帳號B>@gmail.com` |
| `iam.allowedPolicyMemberDomains` | 組織 `<組織網域>` | 改為「Google 代管預設值」（停用） |

> ⚠️ 順序：**先**啟用新政策，**再**停用舊政策，避免中間出現沒有限制的空窗期。
> ⚠️ `allowedPrincipalSets` 一定要包含自家組織，否則連組織內帳號都無法授權。
> ⚠️ 兩個限制同時開著時，舊的還是會擋 Gmail。

---

## 4. 日常操作：新增一個外部 Gmail 帳號

**使用 `<組織管理員帳號>` 帳號操作。**

1. 左上角資源選擇器 → 切到 **`<組織網域>`**（組織，不是專案）。
2. **IAM 與管理 → 組織政策**，篩選 `iam.managed.allowedPolicyMembers`，點 **Restrict allowed policy members in IAM allow policies**。
3. **管理政策** → 展開規則（顯示「已強制執行」）。
4. `allowedMemberSubjects` 右邊鉛筆 → **＋新增值** → 填入：
   ```
   user:<新帳號>@gmail.com
   ```
   （必須有 `user:` 前綴，舊的值不要刪）→ **儲存**。
5. 規則 **完成** → **設定政策**。
6. 等 1～2 分鐘 → 切到目標專案 → **IAM → 授予存取權** → 加入帳號並指派角色。

### 指令版（Cloud Shell）

```bash
gcloud org-policies describe iam.managed.allowedPolicyMembers \
  --organization=<ORG_ID> > policy.yaml

# 編輯 policy.yaml，在 allowedMemberSubjects 底下新增：
#   - user:<新帳號>@gmail.com

gcloud org-policies set-policy policy.yaml

# 授予專案角色
gcloud projects add-iam-policy-binding <PROJECT_ID> \
  --member="user:<新帳號>@gmail.com" --role="roles/editor"
```

---

## 5. 角色建議

| 需求 | 建議角色 | 備註 |
|---|---|---|
| 管理整個組織 | `roles/resourcemanager.organizationAdmin` | 在**組織層級**授予 |
| 看得到組織 | `roles/resourcemanager.organizationViewer` 或 `roles/browser` | 專案層級權限不會讓人看到組織 |
| 專案全權操作 | `roles/editor` + `roles/resourcemanager.projectIamAdmin` | 取代 Owner |
| Owner | 盡量避免 | 外部帳號會 `Precondition check failed` |

---

## 6. 快速排查清單

- [ ] 錯誤訊息是否提到 `constraints/...`？→ 是組織政策擋住，看是哪個 constraint。
- [ ] 帳號是否已在 `allowedMemberSubjects`（含 `user:` 前綴）？
- [ ] 舊的 `iam.allowedPolicyMemberDomains` 是否已停用？
- [ ] 是否在授予 **Owner**？→ 改用 Editor 等角色。
- [ ] 政策剛改完？→ 等 1～2 分鐘再試。
- [ ] 看不到組織？→ 確認登入帳號、資源選擇器切「全部」、確認權限是在**組織層級**。
- [ ] 修改組織政策時，資源選擇器是在**組織**而不是專案？

---

## 7. 安全提醒

- 外部 Gmail 不受組織的 2SV、登入政策、離職停用流程管控，被盜風險較高。
- 給予外部帳號組織管理員權限前，至少確認對方已開啟兩步驟驗證 / passkey。
- 長期使用者建議在 `<組織網域>` 建立正式組織帳號，並定期檢查 `allowedMemberSubjects` 清單，移除不再需要的帳號。
