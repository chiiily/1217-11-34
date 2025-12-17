# CourseTable — 規劃與實作摘要（更新）

以下為依你最新說明整理的需求與實作建議：

---

## 1. 明確的最原始核心功能（你要的）

這是你最初要求要優先完成的最小可行功能（MVP）：

- 顯示「自己的課表」。
- 新增/儲存別人的課表到你的本地。
- 當你按下查看某位同學的課表時，同時顯示「你和該同學的共同空堂時間段」。

以上三項為最優先 (P0)。第二項強調「需對方同意授權」，流程見第 3 節。

## 2. 你要的延伸功能（我之前提出並你要納入的）

- API 端點（後端）：提供查詢、授權、邀請、與共同空閒時間計算等接口。
- 通知功能：當請求被對方接受/拒絕、邀請有回覆、或事件提醒時發送通知，且需有授權控制（對方同意才給你詳細資訊）。

（API 與通知的細節見後面的 API 設計草案與授權流程。）

## 3. 你後來想到的功能（且同意授權為前提）

以下功能均在「對方允許」的前提下才會顯示或允許操作：

1) 發送邀請函（Invitation）：針對共同空堂時間發出邀請。
2) 系統內聊天（私聊 / 群聊）：可在授權範圍內與同學聊天。
3) 行事曆（Calendar）：整合課表與個人事件。
4) 顯示學分數（Credits）：課程資料顯示該課程學分，以及顯示該同學總學分（若授權）。

可行性：上述均可實作，建議分階段引入；聊天可先做非即時版本（訊息存在 DB），再升級為即時（WebSocket）。

## 4. 授權與「對方同意」流程（關鍵）

簡要流程：

1. 你在 UI 上輸入或選擇同學，按「新增/查詢同學課表」。
2. 若你尚無該同學授權，系統會：
	- 顯示提示並發出授權請求給對方（API: `POST /api/v1/permission-requests`）。
	- 在你的界面顯示「等待對方授權」的狀態。
3. 對方在其應用/系統中接收到通知（通知中心或系統推播），可選擇同意或拒絕。
4. 如果對方同意，後端標記授權關係；你即可取得該同學完整課表與學分數，並能使用邀請/聊天等功能（依授權範圍）。
5. 若拒絕，顯示適當訊息，並可提示用戶再次發送請求或聯絡對方。

授權控制建議：支援「全部授權」與「限定授權」（例如：只授權看空堂、或只授權看課程名稱但不顯示地點/學分）。

## 5. 需要的操作頁面與彈出視窗（UI 流程與頁面數）

我把應用拆成主要頁面（View）與按鍵後會跳出的彈窗（Dialog/Modal）：總共建議 **7 個主要操作頁面/視窗**（含必要的彈窗）。

主要頁面與彈窗清單：

1) **主儀表板（Main Dashboard）** — 顯示你的課表（預設畫面）。
	- 元件：週次/日切換、學分總覽、課表格子、快速搜尋欄。

2) **新增/管理同學課表視窗（Add Classmate）** — 按鈕 -> 彈窗
	- 用途：輸入同學帳號或從聯絡人/收藏選擇，提出授權請求或新增已授權之同學。
	- 按下「新增」後：如果未授權，會發送授權請求並顯示等待狀態；如果已授權則直接下載並儲存顯示。

3) **同學課表檢視（Classmate Schedule View）** — 新視窗或側欄
	- 用途：顯示選中同學的課表（受授權範圍決定顯示資訊）。
	- 同視窗包含按鈕：「顯示共同空堂」、「發送邀請」、「開始聊天」（若授權）。

4) **共同空堂視窗（Common Free Time Modal）** — 彈窗
	- 用途：在你與該同學（或多位）之間計算並列出可用時段清單，並標記在課表上。
	- 支援選擇時段粒度（30 分 / 60 分）與可立即發送邀請按鈕。

5) **邀請函發送/管理視窗（Invitations）** — 弹窗或頁籤
	- 用途：建立邀請（選擇時段、地點、訊息），檢視已發邀請與其狀態（pending/accepted/declined）。

6) **通知中心（Notifications）** — 側欄或下拉彈窗
	- 用途：顯示授權請求、邀請回覆、聊天新訊息、事件提醒等。

7) **聊天視窗（Chat）** — 彈窗或獨立頁面
	- 用途：私聊或群聊（若雙方/多人同意聊天功能），起初可採用非即時模式（訊息以 API 存取），未來升級 WebSocket。

補充：**行事曆頁（Calendar）** 可作為第 8 頁（整合課表/事件顯示），用於事件建立與檢視。

簡要互動路徑示例（查看同學 + 發邀請）：
 - 主儀表板 → 點「新增同學」→ 新增視窗輸入帳號 → 發出授權請求 → 等待同意 → 同意後自動在「同學課表檢視」顯示 → 點「共同空堂」 → 開啟「共同空堂視窗」 → 選時段並「發邀請」→ 被邀者於「通知中心」收到邀請。

## 6. 需要實作的後端 API（詳細設計）

### 6.1 技術選擇

推薦方案：**FastAPI + SQLite**

- **FastAPI**：自動生成 Swagger 文檔、型別提示驗證、非同步支援、開發快速
- **SQLite**：無需獨立數據庫服務、易於本地開發與測試

### 6.2 項目結構

```
qt_hello/
├── api/
│   ├── __init__.py
│   ├── main.py              # FastAPI 應用入口
│   ├── models.py            # SQLAlchemy ORM 模型
│   ├── schemas.py           # Pydantic 數據模型
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── schedules.py     # 課表端點
│   │   ├── permissions.py   # 授權端點
│   │   ├── invitations.py   # 邀請端點
│   │   ├── notifications.py # 通知端點
│   │   └── messages.py      # 聊天端點
│   ├── database.py          # DB 連線與初始化
│   ├── auth.py              # 認證邏輯
│   └── utils.py             # 工具函式
├── api_client.py            # 前端 API 客戶端
└── requirements.txt
```

### 6.3 依賴安裝

`requirements.txt`：

```txt
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.5.0
sqlalchemy==2.0.23
python-dateutil==2.8.2
pyjwt==2.8.1
```

安裝：`pip install -r requirements.txt`

### 6.4 完整端點清單與實作

#### 端點概覽表

| 功能 | 方法 | 端點 | 優先級 | 說明 |
|------|------|------|--------|------|
| 登入 | POST | `/api/v1/auth/login` | P0 | 取得 JWT Token |
| 取得自己的課表 | GET | `/api/v1/schedule/me` | P0 | 不需授權 |
| 查詢他人課表 | GET | `/api/v1/schedule/{username}` | P0 | 需授權檢查 |
| 查詢共同空堂 | POST | `/api/v1/common-free-time` | P0 | 時間交集計算 |
| 發送授權請求 | POST | `/api/v1/permissions/request` | P0 | 發起授權流程 |
| 查詢授權狀態 | GET | `/api/v1/permissions/status` | P0 | 查詢所有授權 |
| 回覆授權請求 | PUT | `/api/v1/permissions/{request_id}` | P0 | 接受或拒絕 |
| 發送邀請 | POST | `/api/v1/invitations` | P1 | 邀請同學 |
| 取得邀請列表 | GET | `/api/v1/invitations` | P1 | 已發出/已收到 |
| 回覆邀請 | PUT | `/api/v1/invitations/{invitation_id}` | P1 | 接受或拒絕邀請 |
| 取得通知 | GET | `/api/v1/notifications` | P1 | 未讀通知列表 |
| 標記已讀 | PUT | `/api/v1/notifications/{notification_id}` | P1 | 標記通知已讀 |
| 傳送訊息 | POST | `/api/v1/messages` | P2 | 私聊或群聊 |
| 取得訊息 | GET | `/api/v1/messages` | P2 | 聊天歷史 |

#### 認證端點示例

```python
# routers/auth.py
POST /api/v1/auth/login
Request: {"username": "john", "password": "xxx"}
Response (200): {
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
  "token_type": "bearer",
  "username": "john"
}

Response (401): {"detail": "Invalid credentials"}
```

#### 課表端點示例

```python
# GET /api/v1/schedule/me
Headers: Authorization: Bearer {token}
Response (200): {
  "username": "john",
  "name": "John Doe",
  "total_credits": 18,
  "courses": [
    {
      "id": 1,
      "name": "線性代數",
      "instructor": "Prof. Smith",
      "day": "Monday",
      "start_time": "08:00",
      "end_time": "10:00",
      "location": "A101",
      "credits": 3
    },
    ...
  ]
}

# GET /api/v1/schedule/{username}
# 需檢查授權；若無授權回傳 403 Forbidden
Response (403): {"detail": "Permission denied - authorization pending"}
```

#### 授權端點示例

```python
# POST /api/v1/permissions/request
Request: {
  "to_username": "jane",
  "scope": "full",  // "free_time_only" | "limited" | "full"
  "message": "Hi, can I see your schedule?"
}
Response (201): {
  "request_id": 5,
  "from_username": "john",
  "to_username": "jane",
  "status": "pending",
  "scope": "full",
  "created_at": "2025-12-07T10:30:00"
}

# GET /api/v1/permissions/status
Response (200): {
  "sent": [
    {"request_id": 1, "to_username": "jane", "status": "pending", ...}
  ],
  "received": [
    {"request_id": 2, "from_username": "bob", "status": "pending", ...}
  ]
}

# PUT /api/v1/permissions/{request_id}?action=approve
Response (200): {"status": "approved", "message": "Permission granted"}
```

#### 共同空堂端點示例

```python
# POST /api/v1/common-free-time
Request: {
  "usernames": ["john", "jane"],
  "start_date": "2025-12-06",
  "end_date": "2025-12-12"
}
Response (200): {
  "common_free_times": [
    {"day": "Monday", "time_slot": "14:00-16:00"},
    {"day": "Tuesday", "time_slot": "10:00-12:00"},
    {"day": "Wednesday", "time_slot": "15:00-17:00"}
  ]
}
```

#### 邀請端點示例

```python
# POST /api/v1/invitations
Request: {
  "to_username": "jane",
  "time_slot": "Monday 14:00-16:00",
  "location": "Library",
  "message": "Want to study together?"
}
Response (201): {
  "invitation_id": 1,
  "from_username": "john",
  "to_username": "jane",
  "time_slot": "Monday 14:00-16:00",
  "location": "Library",
  "status": "pending",
  "message": "Want to study together?",
  "created_at": "2025-12-07T14:00:00"
}

# GET /api/v1/invitations
Response (200): {
  "sent": [...],  // 你發出的邀請
  "received": [...] // 你收到的邀請
}

# PUT /api/v1/invitations/{invitation_id}?action=accept
Response (200): {"status": "accepted", "message": "Invitation accepted"}
```

#### 通知端點示例

```python
# GET /api/v1/notifications
Response (200): {
  "unread_count": 3,
  "notifications": [
    {
      "notification_id": 1,
      "type": "permission_request",  // permission_request | invitation | message
      "message": "john wants to access your schedule",
      "is_read": false,
      "created_at": "2025-12-07T10:30:00"
    },
    ...
  ]
}

# PUT /api/v1/notifications/{notification_id}
Response (200): {"status": "marked as read"}
```

### 6.5 認證機制（JWT）

```python
# auth.py
SECRET_KEY = "your-secret-key-change-in-production"
ALGORITHM = "HS256"

def create_access_token(username: str, expires_delta=None):
    if expires_delta is None:
        expires_delta = timedelta(days=7)
    
    expire = datetime.utcnow() + expires_delta
    payload = {"sub": username, "exp": expire}
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def get_current_user(authorization: str = Header(None)):
    # 從 Header 取 Token
    # 驗證 Token 有效性
    # 回傳 username
    ...
```

### 6.6 前端 API 客戶端

```python
# api_client.py
class ScheduleAPIClient:
    def __init__(self, base_url="http://127.0.0.1:8000", token=None):
        self.base_url = base_url
        self.token = token
        self.headers = {"Authorization": f"Bearer {token}"} if token else {}
    
    def login(self, username, password):
        resp = requests.post(f"{self.base_url}/api/v1/auth/login",
                            json={"username": username, "password": password})
        if resp.status_code == 200:
            data = resp.json()
            self.token = data["access_token"]
            self.headers = {"Authorization": f"Bearer {self.token}"}
        return resp.json()
    
    def get_my_schedule(self):
        return requests.get(f"{self.base_url}/api/v1/schedule/me",
                           headers=self.headers).json()
    
    def get_user_schedule(self, username):
        return requests.get(f"{self.base_url}/api/v1/schedule/{username}",
                           headers=self.headers).json()
    
    def send_permission_request(self, to_username, scope="full"):
        data = {"to_username": to_username, "scope": scope}
        return requests.post(f"{self.base_url}/api/v1/permissions/request",
                            json=data, headers=self.headers).json()
    
    def get_common_free_time(self, usernames, start_date, end_date):
        data = {"usernames": usernames, "start_date": start_date, "end_date": end_date}
        return requests.post(f"{self.base_url}/api/v1/common-free-time",
                            json=data, headers=self.headers).json()
    
    def send_invitation(self, to_username, time_slot, location, message=""):
        data = {"to_username": to_username, "time_slot": time_slot,
                "location": location, "message": message}
        return requests.post(f"{self.base_url}/api/v1/invitations",
                            json=data, headers=self.headers).json()
    
    def get_notifications(self):
        return requests.get(f"{self.base_url}/api/v1/notifications",
                           headers=self.headers).json()
    
    def get_permission_status(self):
        return requests.get(f"{self.base_url}/api/v1/permissions/status",
                           headers=self.headers).json()
```

### 6.7 本地開發與測試

啟動後端伺服器：

```bash
cd qt_hello/api
uvicorn main:app --reload --host 127.0.0.1 --port 8000
```

訪問 http://127.0.0.1:8000/docs 即可看到自動生成的 Swagger UI 文檔。

## 7. 實作建議與優先順序（依你調整後）

優先級：

1. 必做（MVP）
	- 主儀表板（自己的課表）
	- 新增/管理同學（含發送授權請求）
	- 同學課表檢視（在授權下可見）
	- 共同空堂計算與顯示（按下同學課表即可）

2. 必要支援功能
	- API：permission-requests, schedule, common-free-time
	- 通知中心（用於授權請求/邀請回覆）
	- 資料庫：users, courses, permissions, invitations, notifications

3. 次階段（可在 MVP 後推出）
	- 行事曆整合
	- 邀請函完整 UI（管理、歷史記錄）
	- 學分顯示（授權下）
	- 聊天：先非即時，再升級即時

## 8. 其他建議

- 授權細分：支援「只看空堂」「只看課程名稱」「完整授權」三種等級，提升隱私與使用者接受度。
- UX 提示：在每一步加入清楚狀態（等待授權 / 已授權 / 被拒絕），避免使用者混淆。
- 開發策略：前端先以 mock API 驗證流程與 UI，再串接後端。
- 資安建議：所有 API 使用 HTTPS；敏感資料（如教室座標）依授權裁剪。

---

如果你同意這個修改，我會把 `ai_docs.md` 保留這個版本，接下來可以：

- 我現在開始產生 Phase 1 的程式樣板（models + api skeleton）並建立範例視窗與簡易 mock API；或
- 我先把 `ai_docs` 拆為多個細檔（例如 `api_design.md`, `ui_pages.md`, `db_schema.md`）。

請告訴我要先做哪一項（我建議先產生 Phase 1 程式樣板）。

