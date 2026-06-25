

自動化部署管理介面
Ops Dashboard — 使用與安裝說明
版本 1.0　　建立日期：2026/6/5




目錄
1. 系統架構總覽
2. Ansible 跳板機 — 初始化安裝
3. API Bridge 啟動與設定
4. Web 介面使用說明
   4.1  TiDB 頁（資料庫指令）
   4.2  Redis 頁（Kuboard 連結）
   4.3  部署更新頁（GitLab Image Tag）
5. PROD 環境安全機制
6. 故障排除
7. 安全建議

1. 系統架構總覽
本系統由三個核心模組組成，涵蓋資料庫操作、Redis 管理及 GitLab 部署更新：

  ┌─────────────────────────────────────────────────────┐
  │          Ops Dashboard（瀏覽器 Web 介面）            │
  │  TiDB 頁  │  Redis 頁  │  部署更新頁               │
  └──────┬────────────┬──────────────┬──────────────────┘
         │            │              │
         ▼            │              ▼
  ┌─────────────┐     │    ┌────────────────────┐
  │ Ansible     │     │    │  GitLab API        │
  │ API Bridge  │     │    │  git.trevi.cc      │
  │ :8765       │     │    │  (修改 YAML + push) │
  └──────┬──────┘     │    └────────────────────┘
         │            │
         ▼            ▼
  ┌─────────────┐  ┌──────────────────────────┐
  │ Ansible     │  │  Kuboard（開啟瀏覽器）    │
  │ Playbook    │  │  UAT: dev-to-kuboard...  │
  │ mysql-cli   │  │  PROD: prod-to-kuboard.. │
  └──────┬──────┘  └──────────────────────────┘
         │
  ┌──────┴──────────────────────┐
  │         TiDB 叢集           │
  │  UAT-PP  UAT-Game          │
  │  PROD-PP PROD-Game         │
  └─────────────────────────────┘

環境對照表
環境	項目	主機	Port
UAT	PP (tidb-backend)	dev-to-tidb-backend.reelx.fun	30040
UAT	Game (tidb-game)	dev-to-tidb-game.reelx.fun	30020
PROD	PP (tidb-backend)	prod-to-tidb-backend.reelx.fun	30040
PROD	Game (tidb-game)	prod-to-tidb-game.reelx.fun	30020



2. Ansible 跳板機 — 初始化安裝
將 setup.sh、api_bridge.py、ansible-api-bridge.service 上傳到跳板機後依序執行。

2.1 系統需求
項目	需求
作業系統	Ubuntu 20.04 / 22.04、Debian 11/12、CentOS/RHEL 8+
Python	3.8 以上（setup.sh 自動安裝 venv）
網路	可連至 TiDB 主機（4 組端點）及 GitLab
執行權限	root 或 sudo
磁碟空間	約 500 MB（Python venv + Ansible）

2.2 一鍵安裝
# 上傳三個檔案到跳板機後執行：
sudo bash setup.sh

# 安裝成功後輸出：
# ✅ 安裝完成！
# 安裝路徑：/opt/ansible-deploy

安裝完成後，目錄結構如下：
/opt/ansible-deploy/
├── venv/                     # Python 虛擬環境
├── ansible.cfg               # Ansible 全局設定
├── group_vars/all.yml        # TiDB 連線資訊（4 組）
├── inventories/hosts.ini     # Inventory（localhost）
├── roles/tidb_query/         # 核心 Role
│   └── tasks/
│       ├── main.yml          # 執行 SQL
│       └── ping.yml          # 連線測試
├── playbooks/
│   ├── run_sql.yml           # 單次查詢
│   ├── run_sql_batch.yml     # 批次查詢
│   └── ping_all.yml          # 連線測試
└── logs/sql_results/         # 查詢結果自動存檔

2.3 驗證安裝
cd /opt/ansible-deploy
source venv/bin/activate

# 測試所有 DB 連線
ansible-playbook playbooks/ping_all.yml

# 手動執行單筆 SQL
./run_query.sh uat_pp backend "SELECT 1;"



3. API Bridge 啟動與設定
3.1 安裝 Flask
source /opt/ansible-deploy/venv/bin/activate
pip install flask

3.2 設定 systemd 服務
# 1. 修改 API Token（必做，不要用預設值）
nano ansible-api-bridge.service
# → 找到 API_BRIDGE_TOKEN=請改成強密碼
# → 改成你的強密碼，例如：API_BRIDGE_TOKEN=My$tr0ngP@ss!

# 2. 安裝並啟動
sudo cp ansible-api-bridge.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now ansible-api-bridge

# 3. 確認狀態
sudo systemctl status ansible-api-bridge
curl http://localhost:8765/health

3.3 API 端點說明
端點	方法	說明
GET /health	GET	健康檢查，無需 Token
POST /api/sql/run	POST	執行 SQL，非同步回傳 job_id
GET /api/sql/result/{id}	GET	查詢 job 執行結果
POST /api/sql/ping	POST	測試所有 TiDB 連線

⚠  API Bridge 預設只監聽 localhost:8765，若 Web 介面與跳板機不在同一網路，需建立 SSH Tunnel 或設定 Nginx 反代（詳見 service 檔案內附設定）。



4. Web 介面使用說明
4.1 TiDB 頁（資料庫指令）
適用場景：對 UAT / PROD 的 TiDB 資料庫執行 SQL 指令，結果由跳板機的 Ansible 回傳。

操作步驟
1.	填入「跳板機 API 位址」，例如：http://10.0.1.50:8765
2.	填入設定於 systemd 的 API Token
3.	點擊「測試所有連線」確認 4 組 DB 皆可連通
4.	選擇環境（UAT / PROD）、項目（PP / Game）、資料庫
5.	在 SQL 輸入框貼上指令，選擇輸出格式（table / json / csv）
6.	點擊「執行」，結果以非同步輪詢方式顯示於下方結果框

連線對應 Key
環境	項目	Ansible Env Key
UAT	PP	uat_pp
UAT	Game	uat_game
PROD	PP	prod_pp
PROD	Game	prod_game

實際執行流程（後台）
瀏覽器 POST /api/sql/run
  → API Bridge 建立 job（非同步）
  → ansible-playbook playbooks/run_sql.yml
      -e tidb_env=uat_pp
      -e tidb_database=backend
      -e tidb_sql="SELECT * FROM users LIMIT 10;"
  → 跳板機 mysql-client 直連 TiDB 執行
  → 結果寫入 logs/sql_results/result_*.txt
  → 瀏覽器輪詢 GET /api/sql/result/{job_id}

4.2 Redis 頁（Kuboard 連結）
Redis 沒有對外暴露端口，透過 Kuboard Web UI 操作。本頁依選擇的環境 / 項目 / 叢集名稱，自動產生對應的 Kuboard 連結並開啟瀏覽器。

PP 項目（固定叢集）
叢集	UAT 連結	PROD 連結
redis-cluster-pp	dev-to-kuboard.../redis-cluster-pp	prod-to-kuboard.../redis-cluster-pp
redis-cluster-wallet	dev-to-kuboard.../redis-cluster-wallet	prod-to-kuboard.../redis-cluster-wallet

Game 項目（動態叢集）
在「叢集名稱」欄位輸入叢集 ID（逗號分隔），系統自動組合連結：
範例輸入：tg102,tg108,tg115

產生連結（UAT）：
  .../namespace/redis-cluster-tg102/workload/view/StatefulSet/redis-cluster-tg102
  .../namespace/redis-cluster-tg108/workload/view/StatefulSet/redis-cluster-tg108
  .../namespace/redis-cluster-tg115/workload/view/StatefulSet/redis-cluster-tg115

4.3 部署更新頁（GitLab Image Tag）
直接呼叫 GitLab Files API，更新 patch-deployment.yaml 中的 image tag，並 commit 到 main branch 觸發後續 CI/CD 流程。

操作步驟
7.	填入 GitLab API URL（https://git.trevi.cc）
8.	填入 Project ID（到 GitLab 專案頁面 → Settings → General 查詢）
9.	填入 Personal Access Token（需要 api scope）
10.	選擇環境、類型（Game / PP）、專案
11.	點擊模塊 chip 選擇要更新的服務（支援多選）
12.	填入 Image Tag（格式：YYYYMMDDHHmmSS-{shortSHA}-{seq}）
13.	點擊「預覽變更」確認路徑正確
14.	點擊「提交 GitLab」（PROD 需額外輸入「確認部署」）

影響檔案路徑對照
環境	類型	patch-deployment.yaml 路徑
UAT	Game	game_project/{proj}/game_project/overlays/uat/{module}/patch-deployment.yaml
UAT	PP	game_project/pp/game_project/overlays/stage/{module}/patch-deployment.yaml
PROD	Game	game_project/{proj}/game_project/overlays/prod/{module}/patch-deployment.yaml
PROD	PP	game_project/pp/game_project/overlays/prod/{module}/patch-deployment.yaml

Commit 格式
# UAT
uat: update {proj} image tag to {tag} [auto-commit]

# PROD
prod: update {proj} image tag to {tag} [auto-commit]



5. PROD 環境安全機制
5.1 雙重確認設計
模組	PROD 額外確認	說明
TiDB	輸入「確認執行」	前端驗證，未通過無法送出 API
TiDB	API Bridge 攔截	DROP TABLE / DROP DATABASE / TRUNCATE 直接回 403
部署更新	輸入「確認部署」	前端驗證，未通過無法呼叫 GitLab API
Redis	PROD 警告橫幅	視覺提醒，連結開啟前顯示紅色警告

5.2 Ansible PROD 防護（api_bridge.py 層）
# api_bridge.py 中的 PROD 硬體攔截：
BLOCKED_KEYWORDS = ["DROP TABLE", "DROP DATABASE", "TRUNCATE"]

if "prod" in env:
    for danger in BLOCKED_KEYWORDS:
        if danger in sql.upper():
            return 403  # 直接拒絕，不進入 Ansible



6. 故障排除
症狀	可能原因	解決方式
「無法連線至跳板機」	API Bridge 未啟動或 IP/Port 錯誤	systemctl status ansible-api-bridge；確認填入的 API 位址
「Unauthorized」401	API Token 不符	確認 Web 介面 Token 與 systemd 環境變數一致
TiDB 連線測試失敗（❌）	跳板機無法路由到 TiDB 主機	跳板機上執行：nc -zv host port 確認網路連通
GitLab 提交失敗（403）	Token 無 api scope 或已過期	GitLab → User Settings → Access Tokens 重新產生
GitLab 提交失敗（404）	Project ID 錯誤或路徑不存在	確認 Project ID；確認 patch-deployment.yaml 已存在
SQL 結果為空或 timeout	SQL 語法錯誤或執行超時（120s）	先在 mysql-client 手動驗證 SQL；複雜查詢加 LIMIT
Ansible log 無輸出	venv 未正確啟動	source /opt/ansible-deploy/venv/bin/activate 後重試



7. 安全建議
7.1 API Token
•	使用至少 32 字元的隨機字串作為 API Token
•	定期輪換 Token（建議每 90 天）
•	不要將 Token 存入版本控制或截圖分享

7.2 Ansible Vault（加密 DB 密碼）
group_vars/all.yml 中的密碼預設為明文，建議使用 Ansible Vault 加密：
cd /opt/ansible-deploy
source venv/bin/activate

# 加密整個 group_vars/all.yml
ansible-vault encrypt group_vars/all.yml

# 執行時使用密碼檔（chmod 600）
echo "your_vault_pass" > ~/.ansible_vault_pass
chmod 600 ~/.ansible_vault_pass

7.3 網路隔離
•	API Bridge 僅監聽 127.0.0.1，對外需透過 SSH Tunnel 或 Nginx + TLS
•	跳板機本身應限制 SSH 存取 IP（AllowUsers / firewall）
•	PROD TiDB 密碼應與 UAT 不同（已符合，^PwdOfProd2025$ vs ^PwdOfDev2020$）

7.4 GitLab Token
•	Token 僅存在瀏覽器記憶體，不經過跳板機
•	建議使用 Project-scoped Token 而非 Personal Token，限制 scope 為 api
•	每次使用完畢可於 GitLab 手動 revoke




自動生成於 2026/6/5　　DevOps Team
