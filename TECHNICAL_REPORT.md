# CourseTable 應用 — 技術性報告

**報告日期**：2025 年 12 月 7 日  
**專案名稱**：CourseTable（揪!!!）— 大學課表協調與邀請應用  
**報告撰寫**：AI Assistant  
**狀態**：規劃階段（Phase Planning）

---

## 目錄

1. [執行摘要](#1-執行摘要)
2. [需求分析](#2-需求分析)
3. [系統架構設計](#3-系統架構設計)
4. [技術棧選擇](#4-技術棧選擇)
5. [功能模塊概覽](#5-功能模塊概覽)
6. [API 設計與實現](#6-api-設計與實現)
7. [數據庫設計](#7-數據庫設計)
8. [前端 UI 設計](#8-前端-ui-設計)
9. [認證與授權機制](#9-認證與授權機制)
10. [實施路線圖](#10-實施路線圖)
11. [風險評估與建議](#11-風險評估與建議)
12. [結論](#12-結論)

---

## 1. 執行摘要

### 1.1 項目概述

**CourseTable** 是一個基於 PyQt6 的桌面應用，旨在幫助大學生：
- 管理並展示個人課表
- 查詢同學的課表（需授權同意）
- 快速找出共同空閒時間
- 發送邀請函進行活動協調
- 系統內聊天與事件管理

### 1.2 核心價值主張

**問題**：學生難以快速協調課程安排，找到與多位同學的共同空閒時間。

**解決方案**：
- 集中式課表管理
- 授權控制機制（保護隱私）
- 智能化時間交集計算
- 一鍵邀請與通知系統

### 1.3 預期成果

**Phase 1 MVP**（4 週）：
- ✅ 自己的課表顯示
- ✅ 同學課表查詢（含授權請求）
- ✅ 共同空堂時段計算與顯示

**Phase 2-4**（後續）：
- 邀請函管理
- 行事曆整合
- 聊天系統（非即時 → 即時升級）
- 通知中心

---

## 2. 需求分析

### 2.1 功能需求（MoSCoW 優先級）

#### Must Have（第 1 週交付）
- [ ] 用戶登入系統
- [ ] 顯示個人課表
- [ ] 搜索與查詢同學帳號
- [ ] 發送授權請求
- [ ] 授權請求的接受/拒絕流程
- [ ] 在授權後顯示同學課表
- [ ] 計算共同空堂時段
- [ ] 在課表上視覺化共同空堂（高亮）

#### Should Have（第 2-3 週）
- [ ] 發送邀請函功能
- [ ] 邀請回應管理（接受/拒絕）
- [ ] 通知中心
- [ ] 學分數顯示
- [ ] 行事曆基礎版

#### Could Have（第 4 週+）
- [ ] 聊天系統（非即時版本）
- [ ] 多人課表並列
- [ ] 邀請歷史記錄

#### Won't Have（未來版本）
- [ ] WebSocket 即時聊天（Phase 4）
- [ ] 移動應用版本
- [ ] 社交功能（動態、評分等）

### 2.2 非功能性需求

| 需求 | 目標值 |
|------|--------|
| 響應時間 | API 端點 < 500ms |
| 可用性 | 99%（開發階段 90%） |
| 資料庫查詢效率 | 索引優化，查詢 < 100ms |
| 並發用戶 | 初期 100，後期 1000+ |
| 資安 | HTTPS、JWT Token、敏感資料加密 |
| 隱私 | GDPR 合規、授權機制 |

### 2.3 用戶角色與故事

#### 角色 1：學生 Alice

**故事**：
> 作為一名大四學生，我想要查看同班同學 Bob 的課表，以便找到我們共同的空閒時間一起準備畢業專題。

**驗收標準**：
- Alice 可以輸入 Bob 的帳號進行搜索
- 系統自動向 Bob 發送授權請求
- Bob 看到請求通知並同意
- Alice 收到授權確認，立即看到 Bob 的課表
- Alice 點擊「共同空堂」，看到周二 14:00-16:00 與周四 10:00-12:00 的空檔

#### 角色 2：教授/系統管理員

**需要**：
- 批量上傳課表數據
- 監控系統使用狀況
- 用戶隱私保護日誌

---

## 3. 系統架構設計

### 3.1 整體架構圖

```
┌─────────────────────────────────────────────────┐
│        PyQt6 Desktop Client（前端）              │
├─────────────────────────────────────────────────┤
│  主儀表板 │ 同學課表 │ 邀請 │ 通知 │ 行事曆     │
└────────────────────┬────────────────────────────┘
                     │ HTTP/HTTPS
                     ↓
┌─────────────────────────────────────────────────┐
│       FastAPI Web Server（後端服務層）           │
├─────────────────────────────────────────────────┤
│  認證  │ 課表 │ 授權 │ 邀請 │ 通知 │ 聊天      │
└────────────────────┬────────────────────────────┘
                     │ SQL
                     ↓
┌─────────────────────────────────────────────────┐
│       SQLite Database（數據層 - 本地開發）       │
├─────────────────────────────────────────────────┤
│ Users │ Courses │ Permissions │ Invitations │ ..│
└─────────────────────────────────────────────────┘

升級版本：將 SQLite → PostgreSQL（多用戶並發）
```

### 3.2 分層架構

```
層級              責任                    技術
────────────────────────────────────────────────
表現層 (UI)      用戶交互 GUI           PyQt6
────────────────────────────────────────────────
業務邏輯層       業務規則、計算          Python class
────────────────────────────────────────────────
API 層           REST 接口              FastAPI
────────────────────────────────────────────────
數據訪問層        ORM、數據庫查詢        SQLAlchemy
────────────────────────────────────────────────
存儲層           持久化存儲             SQLite/PostgreSQL
────────────────────────────────────────────────
```

### 3.3 授權流程時序圖

```
Alice                     Server                    Bob
  │                         │                        │
  ├─ 搜索 Bob 帳號 ────────→ │                        │
  │                         ├─ 檢查用戶存在 ──┐        │
  │                         │                 │        │
  ├─ 發送授權請求 ────────→ │ ←─ 用戶存在 ────┘        │
  │                         ├─ 建立 Permission      │
  │                         │   (status=pending)   │
  │                         │                    │
  │◄─ 等待授權中 ───────────┤                    │
  │                         ├─ 建立通知 ────────→ 收到通知
  │                         │                    │
  │                         │◄─ 點擊同意/拒絕 ────┤
  │                         ├─ 更新 Permission    │
  │                         │   (status=approved) │
  │                         │                    │
  │◄─ 授權確認 ────────────┤                    │
  │                         │                    │
  ├─ 查詢 Bob 課表 ──────→ ├─ 檢查授權 ──┐        │
  │                         │    ✓       │        │
  │◄─ 回傳課表 ────────────┤◄──────────┘        │
  │                         │                    │
```

---

## 4. 技術棧選擇

### 4.1 前端技術

| 組件 | 選擇 | 理由 |
|------|------|------|
| GUI 框架 | PyQt6 v6.4.2 | 跨平台、功能完整、易於快速開發 |
| UI 設計 | Qt Designer | 拖拽式設計，自動生成 XML |
| 網絡請求 | requests | 簡單易用，支持 async |
| 驗證 | Pydantic | 數據驗證一致 |
| 打包 | PyInstaller | 單執行檔分發 |

### 4.2 後端技術

| 組件 | 選擇 | 理由 |
|------|------|------|
| Web 框架 | FastAPI | 現代、高效能、自動 OpenAPI 文檔 |
| 伺服器 | Uvicorn | ASGI 伺服器，支持非同步 |
| ORM | SQLAlchemy 2.0 | 功能完整、支持多數據庫 |
| 認證 | JWT (PyJWT) | 無狀態、安全、易於集成 |
| 驗證 | Pydantic v2 | 型別檢查、自動文檔 |
| 日誌 | logging | Python 標準庫 |

### 4.3 數據庫技術

| 階段 | 數據庫 | 優點 | 缺點 |
|------|--------|------|------|
| 開發 | SQLite | 無需外部服務、快速測試 | 單機、並發差 |
| 測試 | SQLite + Docker | 容易部署 | 仍需升級 |
| 生產 | PostgreSQL | 高並發、功能豐富 | 需維運 |

### 4.4 開發工具

```
IDE              : VS Code / PyCharm
版本控制          : Git + GitHub
包管理            : pip / uv
虛擬環境          : venv / conda
API 文檔          : Swagger UI (自動生成)
測試框架          : pytest + pytest-cov
代碼品質          : pylint, black, mypy
部署              : Docker + 雲服務 (AWS/Azure/本地)
```

---

## 5. 功能模塊概覽

### 5.1 核心模塊清單

#### 模塊 1：認證模塊 (Authentication)
- **責任**：用戶登入、Token 生成與驗證
- **入口**：`/api/v1/auth/login`
- **依賴**：PyJWT、SQLAlchemy
- **優先級**：P0（第 1 週）

#### 模塊 2：課表管理模塊 (Schedule Management)
- **責任**：課表的 CRUD、查詢、權限過濾
- **入口**：`/api/v1/schedule/me`, `/api/v1/schedule/{username}`
- **依賴**：SQLAlchemy、Pydantic
- **優先級**：P0（第 1 週）

#### 模塊 3：授權管理模塊 (Permission Management)
- **責任**：授權請求流程、授權狀態管理
- **入口**：`/api/v1/permissions/*`
- **依賴**：SQLAlchemy、通知模塊
- **優先級**：P0（第 1 週）

#### 模塊 4：共同空堂計算模塊 (Free Time Calculator)
- **責任**：時間交集算法、空堂檢測
- **入口**：`/api/v1/common-free-time`
- **依賴**：datetime、算法庫
- **優先級**：P0（第 1 週）

#### 模塊 5：邀請管理模塊 (Invitation Management)
- **責任**：邀請發送、回應、歷史記錄
- **入口**：`/api/v1/invitations/*`
- **依賴**：SQLAlchemy、通知模塊
- **優先級**：P1（第 2 週）

#### 模塊 6：通知中心模塊 (Notification Center)
- **責任**：通知生成、儲存、已讀標記
- **入口**：`/api/v1/notifications/*`
- **依賴**：SQLAlchemy
- **優先級**：P1（第 2 週）

#### 模塊 7：聊天模塊 (Chat System - Phase 2)
- **責任**：訊息儲存與檢索
- **入口**：`/api/v1/messages/*`
- **依賴**：SQLAlchemy、WebSocket (未來)
- **優先級**：P2（第 4 週+）

---

## 6. API 設計與實現

### 6.1 API 端點設計原則

1. **RESTful 設計**：遵循 REST 架構風格
2. **版本控制**：統一前綴 `/api/v1`
3. **一致的回應格式**：JSON 格式，含 status & data
4. **錯誤處理**：HTTP 狀態碼 + 詳細錯誤訊息
5. **認證**：所有敏感端點需 JWT Token

### 6.2 API 端點清單（完整）

#### 認證相關
```
POST   /api/v1/auth/login              # 用戶登入
POST   /api/v1/auth/logout             # 登出
POST   /api/v1/auth/refresh-token      # 刷新 Token
```

#### 課表相關
```
GET    /api/v1/schedule/me             # 取得自己的課表
GET    /api/v1/schedule/{username}     # 取得他人課表（需授權）
POST   /api/v1/schedule                # 上傳課表（管理員）
PUT    /api/v1/schedule/{course_id}    # 編輯課程
DELETE /api/v1/schedule/{course_id}    # 刪除課程
POST   /api/v1/common-free-time        # 計算共同空堂
```

#### 授權相關
```
POST   /api/v1/permissions/request     # 發送授權請求
GET    /api/v1/permissions/status      # 查詢授權狀態
PUT    /api/v1/permissions/{req_id}    # 回覆授權請求（approve/decline）
GET    /api/v1/permissions/list        # 列出所有授權記錄
```

#### 邀請相關
```
POST   /api/v1/invitations             # 發送邀請
GET    /api/v1/invitations             # 取得邀請列表（sent/received）
PUT    /api/v1/invitations/{inv_id}    # 回覆邀請（accept/decline）
DELETE /api/v1/invitations/{inv_id}    # 取消邀請
```

#### 通知相關
```
GET    /api/v1/notifications           # 取得未讀通知
PUT    /api/v1/notifications/{notif_id} # 標記已讀
DELETE /api/v1/notifications/{notif_id} # 刪除通知
```

#### 聊天相關（Phase 2）
```
POST   /api/v1/messages                # 傳送訊息
GET    /api/v1/messages                # 取得聊天歷史
GET    /api/v1/groups                  # 列出群組
POST   /api/v1/groups                  # 建立群組
```

### 6.3 API 回應格式範例

**成功回應 (2xx)**
```json
{
  "status": "success",
  "data": {
    "username": "john",
    "name": "John Doe",
    "total_credits": 18,
    "courses": [...]
  }
}
```

**錯誤回應 (4xx/5xx)**
```json
{
  "status": "error",
  "error_code": "permission_denied",
  "detail": "You don't have permission to access this resource",
  "timestamp": "2025-12-07T15:30:00Z"
}
```

---

## 7. 數據庫設計

### 7.1 核心表結構

#### 表 1：users（用戶表）
```sql
CREATE TABLE users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  username VARCHAR(50) UNIQUE NOT NULL,
  email VARCHAR(100) UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  name VARCHAR(100),
  student_id VARCHAR(20),
  total_credits INTEGER DEFAULT 0,
  profile_picture_url TEXT,
  is_active BOOLEAN DEFAULT 1,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 表 2：courses（課程表）
```sql
CREATE TABLE courses (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id INTEGER NOT NULL,
  name VARCHAR(100) NOT NULL,
  instructor VARCHAR(100),
  day VARCHAR(20),  -- Monday, Tuesday, ...
  start_time VARCHAR(5),  -- HH:MM format
  end_time VARCHAR(5),
  location VARCHAR(100),
  credits INTEGER,
  course_code VARCHAR(20),
  semester VARCHAR(20),  -- 2025-Spring, 2025-Fall
  is_mandatory BOOLEAN DEFAULT 0,
  created_at TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

#### 表 3：permissions（授權表）
```sql
CREATE TABLE permissions (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  from_user_id INTEGER NOT NULL,
  to_user_id INTEGER NOT NULL,
  scope VARCHAR(20),  -- 'free_time_only' | 'limited' | 'full'
  status VARCHAR(20),  -- 'pending' | 'approved' | 'declined'
  message TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  responded_at TIMESTAMP,
  expires_at TIMESTAMP,  -- 可選：授權過期日期
  FOREIGN KEY (from_user_id) REFERENCES users(id),
  FOREIGN KEY (to_user_id) REFERENCES users(id),
  UNIQUE(from_user_id, to_user_id)  -- 防止重複請求
);
```

#### 表 4：invitations（邀請表）
```sql
CREATE TABLE invitations (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  from_user_id INTEGER NOT NULL,
  to_user_id INTEGER NOT NULL,
  time_slot VARCHAR(100),  -- "Monday 14:00-16:00"
  location VARCHAR(100),
  message TEXT,
  status VARCHAR(20),  -- 'pending' | 'accepted' | 'declined' | 'cancelled'
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  responded_at TIMESTAMP,
  FOREIGN KEY (from_user_id) REFERENCES users(id),
  FOREIGN KEY (to_user_id) REFERENCES users(id)
);
```

#### 表 5：notifications（通知表）
```sql
CREATE TABLE notifications (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id INTEGER NOT NULL,
  type VARCHAR(50),  -- 'permission_request' | 'invitation' | 'message' | 'reminder'
  message TEXT,
  related_id INTEGER,  -- 關聯的請求/邀請 ID
  is_read BOOLEAN DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  read_at TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

#### 表 6：messages（訊息表）
```sql
CREATE TABLE messages (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  sender_id INTEGER NOT NULL,
  receiver_id INTEGER,  -- 私聊對象（NULL 表示群聊）
  group_id INTEGER,      -- 群組 ID（NULL 表示私聊）
  content TEXT NOT NULL,
  is_read BOOLEAN DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (sender_id) REFERENCES users(id),
  FOREIGN KEY (receiver_id) REFERENCES users(id),
  FOREIGN KEY (group_id) REFERENCES groups(id)
);
```

#### 表 7：groups（群組表）
```sql
CREATE TABLE groups (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name VARCHAR(100) NOT NULL,
  description TEXT,
  creator_id INTEGER NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (creator_id) REFERENCES users(id)
);
```

#### 表 8：group_members（群組成員表）
```sql
CREATE TABLE group_members (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  group_id INTEGER NOT NULL,
  user_id INTEGER NOT NULL,
  joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (group_id) REFERENCES groups(id),
  FOREIGN KEY (user_id) REFERENCES users(id),
  UNIQUE(group_id, user_id)
);
```

#### 表 9：events（行事曆事件表）
```sql
CREATE TABLE events (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id INTEGER NOT NULL,
  title VARCHAR(100) NOT NULL,
  description TEXT,
  start_time TIMESTAMP NOT NULL,
  end_time TIMESTAMP NOT NULL,
  location VARCHAR(100),
  color VARCHAR(20),  -- 用於 UI 高亮
  reminder_minutes INTEGER DEFAULT 15,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

### 7.2 索引策略

```sql
-- 性能優化索引
CREATE INDEX idx_courses_user_id ON courses(user_id);
CREATE INDEX idx_courses_day ON courses(day);
CREATE INDEX idx_permissions_from_to ON permissions(from_user_id, to_user_id);
CREATE INDEX idx_permissions_status ON permissions(status);
CREATE INDEX idx_invitations_user_status ON invitations(to_user_id, status);
CREATE INDEX idx_notifications_user_read ON notifications(user_id, is_read);
CREATE INDEX idx_messages_receiver ON messages(receiver_id);
CREATE INDEX idx_group_members_group ON group_members(group_id);
```

### 7.3 ER 圖（簡化）

```
┌─────────────┐
│    users    │
├─────────────┤
│ id (PK)     │
│ username    │
│ email       │
│ name        │
└─────────────┘
       │
       ├──────────────────────┬──────────────────────┐
       │                      │                      │
       ↓                      ↓                      ↓
┌─────────────┐      ┌──────────────┐      ┌────────────────┐
│  courses    │      │ permissions  │      │  invitations   │
├─────────────┤      ├──────────────┤      ├────────────────┤
│ id (PK)     │      │ id (PK)      │      │ id (PK)        │
│ user_id(FK) │      │ from_user_id │      │ from_user_id   │
│ name        │      │ to_user_id   │      │ to_user_id     │
│ day         │      │ scope        │      │ time_slot      │
│ start_time  │      │ status       │      │ location       │
│ end_time    │      └──────────────┘      │ status         │
└─────────────┘                           └────────────────┘
       │
       ├──────────────────────┬──────────────────────┐
       │                      │                      │
       ↓                      ↓                      ↓
┌──────────────┐    ┌────────────────┐    ┌──────────────┐
│ notifications│    │    messages    │    │    events    │
├──────────────┤    ├────────────────┤    ├──────────────┤
│ id (PK)      │    │ id (PK)        │    │ id (PK)      │
│ user_id(FK)  │    │ sender_id(FK)  │    │ user_id(FK)  │
│ type         │    │ receiver_id(FK)│    │ title        │
│ message      │    │ group_id(FK)   │    │ start_time   │
└──────────────┘    │ content        │    │ end_time     │
                    └────────────────┘    └──────────────┘
```

---

## 8. 前端 UI 設計

### 8.1 頁面結構（7 個主要頁面）

#### 頁面 1：主儀表板（Main Dashboard）
**目的**：顯示使用者的課表及快速操作

**組件**：
- 用戶信息卡（頭像、名字、總學分）
- 週次/日期選擇器
- 課表網格（5 列 × 10 行，每格代表一個時間槽）
- 快速搜尋欄（搜尋同學）
- 按鈕：「新增同學」、「行事曆」、「邀請」、「通知」

**互動**：
- 點擊課程單元格 → 查看課程詳細資訊
- 點擊「新增同學」→ 彈出 Modal

**狀態顯示**：
```
自己的課程      ■ 藍色 (rgb(224, 227, 255))
共同空堂        ■ 綠色 (rgb(150, 255, 150))
待授權          ■ 灰色 (rgb(200, 200, 200))
```

#### 頁面 2：新增/管理同學（Add Classmate Modal）
**目的**：發起授權請求或選擇已授權同學

**組件**：
- 文本輸入框（輸入帳號）
- 自動完成下拉（推薦同學）
- 授權範圍選擇（單選按鈕）
  - ○ 只看空堂
  - ○ 看課程名稱（不看地點）
  - ○ 完全授權
- 按鈕：「發送請求」、「取消」

**流程**：
- 輸入帳號 → 檢查授權狀態 → 顯示「發送請求」或「已授權」

#### 頁面 3：同學課表檢視（Classmate Schedule View）
**目的**：顯示同學的課表（受授權範圍限制）

**組件**：
- 同學資訊卡（名字、學分、授權範圍）
- 課表網格（顯示同學課程）
- 按鈕：「顯示共同空堂」、「發送邀請」、「開始聊天」

**樣式差異**：
- 同學課程用不同顏色區分（例如橙色）

#### 頁面 4：共同空堂視窗（Common Free Time Modal）
**目的**：列出並選擇共同空閒時段

**組件**：
- 時段粒度選擇（30 分 / 60 分）
- 共同空堂列表（表格）
  - 日期 | 時間 | 選擇 (Checkbox)
- 在課表上視覺化高亮
- 按鈕：「發送邀請」、「關閉」

**範例輸出**：
```
共同空堂結果：
✓ 週一 14:00-16:00
✓ 週二 10:00-12:00
✓ 週三 15:00-17:00
```

#### 頁面 5：邀請函管理（Invitations）
**目的**：管理已發出與已收到的邀請

**組件**：
- 標籤頁：「已發出」| 「已收到」
- 邀請列表（表格）
  - 對方名字 | 時間 | 地點 | 狀態 | 操作
- 狀態指示（pending/accepted/declined）
- 按鈕：「新建邀請」、「刪除」、「詳情」

#### 頁面 6：通知中心（Notifications）
**目的**：集中查看所有通知

**組件**：
- 通知列表（下拉或側欄）
- 通知分類（授權請求 | 邀請回覆 | 聊天訊息）
- 未讀通知徽章（紅點）
- 標記已讀 / 全部已讀
- 通知清空選項

**通知類型範例**：
```
[授權] Bob 同意了你的授權請求
[邀請] Alice 接受了你的邀請 (周二 14:00-16:00)
[訊息] Charlie 傳來新訊息
```

#### 頁面 7：聊天視窗（Chat - Phase 2）
**目的**：與同學進行私聊或群聊

**組件**：
- 聯絡人列表（左側邊欄）
- 聊天窗口（中間）
  - 訊息氣泡（區分發送者/接收者）
  - 輸入框 + 傳送按鈕
- 群組管理（建立/邀請/移除成員）

**補充：行事曆頁（Calendar - Phase 2）**
- 月視圖 + 周視圖
- 課程自動導入
- 事件建立與編輯
- 提醒設定

### 8.2 色彩方案

```
主色調      淡紫藍 rgb(206, 207, 255)
背景        白色 rgb(255, 255, 255)
課程格       淡藍紫 rgb(224, 227, 255)
共同空堂     淡綠 rgb(150, 255, 150)
我的課程     藍色 rgb(100, 150, 255)
同學課程     橙色 rgb(255, 150, 100)
待授權/灰色   rgb(200, 200, 200)
成功/綠色     rgb(100, 200, 100)
錯誤/紅色     rgb(255, 100, 100)
```

### 8.3 UI 互動流程圖

```
啟動應用
    ↓
登入頁面 ─→ 輸入帳密 ─→ 驗證成功
    ↓
主儀表板 (顯示自己課表)
    │
    ├─ 點「新增同學」
    │   ↓
    │  新增同學 Modal ─→ 輸入帳號 ─→ 發送授權請求
    │   │
    │   └─ 等待對方同意
    │       ↓
    │   ✓ 授權成功 → 自動顯示同學課表
    │   ✗ 授權失敗 → 顯示提示
    │
    ├─ 點「通知」
    │   ↓
    │  通知中心 ─→ 查看未讀
    │   │
    │   └─ 點授權請求 → 接受/拒絕
    │       點邀請 → 接受/拒絕
    │       點訊息 → 開啟聊天
    │
    ├─ 選擇同學課表 ─→ 檢視
    │   ↓
    │  點「共同空堂」
    │   ↓
    │  列出共同時段 ─→ 選擇 ─→ 發邀請
    │   ↓
    │  邀請 Modal ─→ 填寫詳情 ─→ 傳送
    │
    └─ 點「邀請」
        ↓
       邀請管理頁 ─→ 查看已發出/已收到
```

---

## 9. 認證與授權機制

### 9.1 JWT 認證流程

```
用戶登入
  ↓
POST /api/v1/auth/login
{username, password}
  ↓
驗證密碼 (bcrypt)
  ↓
生成 JWT Token
payload: {
  "sub": "john",
  "exp": 1733595000,
  "iat": 1733508600
}
  ↓
回傳 access_token
  ↓
客戶端儲存 Token
(LocalStorage 或內存)
  ↓
後續請求
  ↓
Header: Authorization: Bearer {token}
  ↓
驗證 Token 簽名與過期時間
  ↓
允許/拒絕訪問
```

### 9.2 授權機制（Permission Scope）

**三層授權模型**：

#### Scope 1：free_time_only（只看空堂）
- ✓ 可計算共同空堂
- ✗ 不看課程名稱、地點、學分
- 用途：「只想找時間，不想透露課表」

#### Scope 2：limited（基礎授權）
- ✓ 可看課程名稱、時間
- ✗ 不看地點、教師、學分
- 用途：「一般使用」

#### Scope 3：full（完全授權）
- ✓ 看所有資訊（課程名稱、時間、地點、教師、學分）
- 用途：「親密朋友」

### 9.3 授權決策樹

```
GET /api/v1/schedule/{username}

是否為自己？
├─ Yes → 回傳完整課表
│
└─ No → 檢查 Permission 記錄
    │
    ├─ Permission 不存在？
    │  └─ 403 Forbidden + 提示發送授權請求
    │
    ├─ Permission.status = pending？
    │  └─ 403 Forbidden + 提示等待中
    │
    ├─ Permission.status = declined？
    │  └─ 403 Forbidden + 提示已拒絕
    │
    ├─ Permission.status = approved？
    │  └─ 檢查 scope
    │      ├─ free_time_only → 回傳時間資訊 (不含課程)
    │      ├─ limited → 回傳課程名稱+時間
    │      └─ full → 回傳完整課表
```

---

## 10. 實施路線圖

### 10.1 開發週期（12 週計劃）

#### Week 1-2: 核心基礎設施
- [ ] 建立項目結構與虛擬環境
- [ ] 設計數據庫 Schema
- [ ] 實現 SQLAlchemy 模型
- [ ] 實現認證系統（登入/Token）
- [ ] 編寫 API 框架（FastAPI 路由）
- [ ] 初始化前端 PyQt6 專案

**可交付**：基礎應用框架 + 登入功能

#### Week 3-4: 課表管理
- [ ] 實現課表 CRUD 端點
- [ ] 實現課表查詢端點（含授權檢查）
- [ ] 共同空堂計算演算法
- [ ] 前端課表顯示頁面
- [ ] API 客戶端類

**可交付**：顯示自己課表 + 計算共同空堂

#### Week 5-6: 授權系統
- [ ] 實現授權請求端點
- [ ] 授權狀態管理端點
- [ ] 通知系統（基礎版）
- [ ] 前端授權請求 UI
- [ ] 前端通知中心 UI

**可交付**：完整授權流程 + 查詢他人課表

#### Week 7-8: 邀請與聊天（非即時）
- [ ] 邀請管理端點
- [ ] 非即時聊天端點
- [ ] 前端邀請管理 UI
- [ ] 前端聊天 UI（基礎版）
- [ ] 邀請與聊天通知整合

**可交付**：發送邀請 + 基礎聊天

#### Week 9-10: 行事曆與學分
- [ ] 行事曆事件端點
- [ ] 學分計算邏輯
- [ ] 前端行事曆 UI
- [ ] 學分顯示集成
- [ ] 事件與課表同步

**可交付**：完整的行事曆 + 學分管理

#### Week 11-12: 測試、優化與部署
- [ ] 單元測試（API 端點）
- [ ] 集成測試（前後端）
- [ ] UI/UX 測試
- [ ] 性能優化
- [ ] Bug 修復
- [ ] 打包與部署準備

**可交付**：正式版本 v1.0

### 10.2 里程碑與交付

| 里程碑 | 完成時間 | 可交付物 | 驗收標準 |
|--------|---------|---------|---------|
| M1: MVP | 第 6 週 | 能查看自己與同學課表 | 授權流程正常 |
| M2: 邀請 | 第 8 週 | 發送邀請 + 基礎聊天 | 邀請通知正常 |
| M3: 完整 | 第 10 週 | 行事曆 + 學分 | 功能齊全 |
| M4: 產品 | 第 12 週 | 可部署版本 | 通過測試 |

### 10.3 測試策略

```
單元測試 (Unit Test)
├─ API 端點測試 (pytest)
├─ 模型驗證測試
├─ 授權邏輯測試
└─ 算法測試 (共同空堂計算)

集成測試 (Integration Test)
├─ 端到端流程測試
├─ 數據庫操作測試
└─ 前後端通信測試

UI/UX 測試 (Manual)
├─ UI 互動測試
├─ 流程驗證
└─ 錯誤提示測試

性能測試 (Performance)
├─ API 響應時間測試
├─ 數據庫查詢效率
└─ 並發用戶測試
```

---

## 11. 風險評估與建議

### 11.1 技術風險

| 風險 | 概率 | 影響 | 緩解 |
|------|------|------|------|
| 後端 API 未就緒 | 高 | 中 | 使用 mock 數據開發前端 |
| 時間交集演算法複雜 | 中 | 中 | 提前製作原型、單元測試 |
| 資料庫 Schema 變更 | 中 | 高 | 使用遷移工具 (Alembic) |
| JWT Token 過期管理 | 低 | 低 | 實現 Token 刷新機制 |

### 11.2 項目風險

| 風險 | 概率 | 影響 | 緩解 |
|------|------|------|------|
| 需求變更頻繁 | 高 | 中 | 使用敏捷方法、定期評審 |
| 人力資源不足 | 中 | 高 | 明確分工、優先級排序 |
| 時間延誤 | 中 | 中 | 緩衝計劃 + 里程碑檢查 |

### 11.3 安全風險

| 風險 | 影響 | 防護 |
|------|------|------|
| SQL 注入 | 高 | 使用 SQLAlchemy ORM |
| 授權繞過 | 高 | 每個端點檢查授權 |
| Token 洩露 | 中 | HTTPS + Token 簽名 |
| 密碼儲存不當 | 高 | bcrypt 雜湊 + Salt |

### 11.4 最佳實踐建議

```python
# 1. 密碼安全
from passlib.context import CryptContext
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
hashed_password = pwd_context.hash(password)

# 2. 環境變數管理
from dotenv import load_dotenv
import os
load_dotenv()
SECRET_KEY = os.getenv("SECRET_KEY")

# 3. API 日誌記錄
import logging
logger = logging.getLogger(__name__)
logger.info(f"User {username} accessed schedule")

# 4. 輸入驗證
from pydantic import BaseModel, Field
class PermissionRequest(BaseModel):
    to_username: str = Field(..., min_length=2, max_length=50)
    scope: str = Field(..., regex="^(free_time_only|limited|full)$")

# 5. 錯誤處理
try:
    result = get_schedule(username)
except PermissionError:
    raise HTTPException(status_code=403, detail="Permission denied")
except ValueError as e:
    raise HTTPException(status_code=400, detail=str(e))
```

---

## 12. 結論

### 12.1 項目總結

**CourseTable** 是一個可行且具有實際價值的應用，解決了大學生課表協調的真實問題。通過分階段開發、清晰的架構設計與嚴格的測試，該項目可在 12 週內交付 MVP 版本。

### 12.2 核心優勢

✅ **用戶隱私**：授權機制保護個人課表不被濫用  
✅ **易於使用**：直觀的 GUI 與簡單的邀請流程  
✅ **可擴展性**：分層架構易於添加新功能  
✅ **技術選型成熟**：使用業界標準框架，降低風險  

### 12.3 後續計劃

**Phase 2-3**：
- 即時聊天升級（WebSocket）
- 多人課表並列顯示
- 本地快取與離線支持

**Phase 4+**：
- 移動應用版本（React Native / Flutter）
- 社交功能（評分、動態）
- AI 推薦引擎

### 12.4 成功關鍵因素

1. **清晰的優先級**：確保 MVP 功能優先完成
2. **定期測試**：每週進行回歸測試
3. **用戶反饋**：邀請目標用戶進行 UAT
4. **文檔完善**：API 文檔與開發指南同步更新
5. **持續交付**：定期發布版本，收集反饋

---

## 附錄

### A. 技術參考資源

- **FastAPI 官方文檔**: https://fastapi.tiangolo.com/
- **PyQt6 文檔**: https://doc.qt.io/qt-6/
- **SQLAlchemy 教程**: https://docs.sqlalchemy.org/
- **JWT 認證指南**: https://tools.ietf.org/html/rfc7519

### B. 工具與命令

```bash
# 初始化項目
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt

# 啟動後端
cd api
uvicorn main:app --reload --host 127.0.0.1 --port 8000

# 啟動前端
python main.py

# 執行測試
pytest tests/ -v --cov=api

# 代碼品質檢查
pylint api/
black --check api/
mypy api/
```

### C. 開發環境檢查清單

- [ ] Python 3.9+ 已安裝
- [ ] 虛擬環境已建立
- [ ] 依賴包已安裝
- [ ] VS Code / IDE 已配置
- [ ] Git 已初始化
- [ ] `.env` 文件已配置
- [ ] 數據庫已初始化
- [ ] 本地後端可正常啟動
- [ ] Swagger UI 可訪問 (http://localhost:8000/docs)
- [ ] 前端應用可正常啟動

---

**報告編制日期**：2025 年 12 月 7 日  
**最後更新**：2025 年 12 月 7 日 15:30 UTC  
**作者**：AI Assistant  
**審批人**：（待指定）  
**版本**：1.0 - 初稿
