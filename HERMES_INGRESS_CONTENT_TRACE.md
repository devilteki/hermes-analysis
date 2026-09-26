# Hermes Telegram Inbound 訊息本文持久化追蹤報告

## 1. 查核 Commit 與環境對比

- **本次實際查核 Commit**: `9fd30b85f15f4f07c413691b2f4a169990bbb3f0` [已核實]
- **舊報告查核 Commit**: `9fd30b85f15f4f07c413691b2f4a169990bbb3f0` [已核實]
- **代碼差異比對**: 本次查核分支為 `jules-ingress-session-trace-12122229779866332452`，目前 HEAD 指向 `9fd30b85f15f4f07c413691b2f4a169990bbb3f0`，與舊報告 (`HERMES_INGRESS_SESSION_TRACE.md`) 所查核之 Commit 完全相同，相關傳輸與持久化程式碼無任何差異。 [已核實]

---

## 2. 情境定義與範疇說明

本報告接續 `HERMES_INGRESS_SESSION_TRACE.md`（舊報告結尾於新建 Telegram 私聊 Session 路由與 Session 實體列於 `gateway_routing`、`sessions.json` 及 `sessions` 資料表落盤）。

- **場景設定**:
  - 單一 Telegram 私聊（`chat_type = "dm"` / `"private"`）
  - 一般文字訊息（`MessageType.TEXT`）
  - 新建 Session（New Session）
- **追蹤起終點**: 從 Session Store 取得 / 建立 Session 後，追到**使用者訊息本文第一次確認完成的持久化寫入（First Durable Content Write）**為止。不展開模型 LLM 執行、結果送出、其他平台或架構建議。 [已核實]

---

## 3. 四類敘述修正前後對照表 (Itemized Revision Log)

### 修正類別 ①：`update_id` 進入 `MessageEvent.platform_update_id` 後的實際傳遞軌跡
- **修正前敘述**: 誤稱「`update_id` 未在 `MessageEvent` 中保存，自 Adapter 入口起即丟棄、未落盤」。
- **修正後敘述**: `update_id` 在 `_build_message_event` (L7082) 中傳入 `MessageEvent(platform_update_id=update_id)` 保存。隨事件在記憶體中傳遞：
  - 在 `/restart` 指令情境下（`gateway/slash_commands.py` L570），`platform_update_id` 會被讀取並寫入 `.restart_last_processed.json` 檔案落盤，供新啟動的 Gateway 程序進行重送去重（`gateway/run_busy.py` L1197）。
  - 在一般文字訊息對話情境下，`platform_update_id` 在 `base.py` 中可作為 `turn_marker` 的備用標記，隨 `MessageEvent` 物件在記憶體中傳遞，但**不會**寫入 SQLite `state.db` 的 `messages` 資料表。
- **對應原始碼位置**:
  - `plugins/platforms/telegram/adapter.py` (L7082)
  - `gateway/platforms/event.py` (L62)
  - `gateway/platforms/base.py` (L462)
  - `gateway/slash_commands.py` (L545, L570)
  - `gateway/run_busy.py` (L1197, L1230)

### 修正類別 ②：`expand_link_entities()` 與後續處理對原文一致性的影響
- **修正前敘述**: 僅簡單稱「兩者完全相同，僅清洗 Unicode surrogates」。
- **修正後敘述**:
  1. `expand_link_entities(message)` (`plugins/platforms/telegram/telegram_entities.py` L24) 檢查 Telegram Entity，若訊息含有隱藏 URL 的 `text_link` 類型實體，會將隱藏 URL 以 ` (URL)` 格式展開插在錨點文字後（如 `"Google"` 展開為 `"Google (https://google.com)"`）；純文字或無 `text_link` 時文字維持原樣。
  2. 後續在 `_hmwa_apply_message_timestamp()` (`gateway/run_turn.py` L2136) 中，若文字帶有時間戳記語法，時間戳記會被解析分離。
  3. 在 `sanitize_surrogates()` (`agent/turn_context.py` L1015) 中，無效之 UTF-16 surrogate pairs 會被清理。
  4. 經上述轉換後的文字，才是最終寫入 `messages.content` 欄位之內容。
- **對應原始碼位置**:
  - `plugins/platforms/telegram/telegram_entities.py` (L24-L80)
  - `plugins/platforms/telegram/adapter.py` (L7081)
  - `gateway/run_turn.py` (L2136)
  - `agent/turn_context.py` (L1015)

### 修正類別 ③：`mark_turn_active()` / `_mark_durable_active_turn()` 與訊息本文落盤的區隔
- **修正前敘述**: 模糊未區分活躍 Turn 標記與訊息本文落盤。
- **修正後敘述**:
  1. `_mark_durable_active_turn()` (`gateway/run_inbound.py` L1755) 呼叫 `async_session_store.mark_turn_active(session_key)` (`gateway/session_lifecycle.py` L132)。
  2. 此操作生成 UUID token 並更新 `SessionEntry` 的 `active_turn_token` 與 `active_turn_started_at`，並落盤寫入 `gateway_routing` 資料表（`mark_turn_active()` 經由 `_save_entry` UPSERT 寫入 SQLite `gateway_routing` 表成功即返回；僅在單筆 SQLite 寫入失敗或全量快照時才會作為 fallback 鏡像寫入 `sessions.json`）。
  3. **此寫入為 Turn 的運行狀態標記（Running Turn Lease Marker）**，用於崩潰重啟時的排空與救援判定，**不包含使用者訊息本文**；訊息本文的第一次持久化落盤必須等到 Step 15 的 `_persist_turn_start()` 寫入 `messages` 資料表。
- **對應原始碼位置**:
  - `gateway/run_turn.py` (L2105)
  - `gateway/run_inbound.py` (L1755)
  - `gateway/session_lifecycle.py` (L132)

### 修正類別 ④：`_persist_turn_start()`、`_persist_under_lock()` 及 DB flush 的成功／失敗條件與邊界標註
- **修正前敘述**: 將 Telegram Polling ACK、取消前必已落盤、崩潰必然永久遺失寫為絕對斷言。
- **修正後敘述**:
  1. **`_persist_under_lock()` 成功／失敗機制**: `_persist_under_lock()` (`agent/turn_context.py` L468) 包裹 `_ensure_and_persist()`，其內部有 `try...except Exception` 區塊，會捕獲寫入過程中的例外並發出警告（Log-and-Swallow）。因此，**即使 SQLite 寫入失敗，`_persist_turn_start()` 也不會拋出 Exception 中斷 Turn 流程**。
  2. **DB Flush 成功／失敗條件**:
     - **成功條件**: `_db_flush_write()` (L258) 呼叫 `SessionDB.append_messages_batch()` 成功執行 `INSERT INTO messages` 且事務提交，並為記憶體 dict 蓋上 `_DB_PERSISTED_MARKER = True` 與 `_row_id`。
     - **失敗處理**: 若 SQLite 拋出 `StateDbReplacedError` / `StateDbCorruptError`，`_db_flush_failed()` (L299) 會嘗試調用 `divert_session_transcript_jsonl()` 將未落盤訊息寫入急救 JSONL 檔。
  3. **邊界標註調整**:
     - **Telegram Polling/Webhook ACK 時點**: 改標為 **【條件成立才會發生】/【待查】**。專案代碼依賴 `python-telegram-bot` (PTB) 套件，WebHook 在 HTTP Handler 收到 Payload 即回覆 200 OK，Polling 於下一次 `getUpdates` 帶 offset 時 ACK。專案代碼內部無直接發送 ACK 的指令。
     - **取消前必已落盤**: 改標為 **【條件成立才會發生】**。僅當取消（如 `/stop` 或 SIGINT）發生在 `_persist_turn_start()` 執行**之後**且 DB 寫入成功時，本文才已落盤；若取消發生在 `_persist_turn_start()` 執行**之前**，訊息尚未寫入 `messages` 表（會經由 `carry_unadmitted_user_message` 暫存）。
     - **崩潰必然永久遺失**: 改標為 **【條件成立才會發生】**。若 Telegram 已 ACK 且進程在 `_persist_turn_start()` 寫入前崩潰（或磁碟已滿導致急救檔亦無法寫入），資料始會遺失；若 Telegram 尚未 ACK 或急救 JSONL 寫入成功，則不致永久遺失。
- **對應原始碼位置**:
  - `agent/turn_context.py` (L468, L963)
  - `agent/session_persistence.py` (L258, L299, L347, L380)
  - `hermes_state_messages.py` (L364, L515)

---

## 4. 訊息本文持久化全流程追蹤 (Step 10 ~ Step 15)

### Step 10: Turn Preparation & Message Preprocessing
- **代碼路徑**: `gateway/run_turn.py:GatewayRunnerTurnMixin._hmwa_prepare_turn()` (L2069)
- **處理邏輯**:
  1. 調用 `_prepare_profile_scoped_inbound_message_text()` (L2131) 處理 Profile 作用域與多媒體/圖片等標記，取得文字 `message_text`。
  2. 調用 `_hmwa_apply_message_timestamp(event, message_text)` (L2136) 提取訊息中的時間戳記標記（若有），分離得到純文字 `message_text` 與 `persist_user_message`（若有 API-only 恢復引導）、`persist_user_timestamp`。
  3. 調用 `_mark_durable_active_turn(event, session_key)` (L2105) 更新 `SessionEntry` 活躍狀態並寫入 `gateway_routing` 資料表（此為 Turn 運行標記，非訊息本文落盤；SQLite 寫入成功即返回，僅失敗或全量快照時才後備寫入 `sessions.json`）。
  4. 封裝為 `_PreparedTurn` (L2156)，傳回 `_handle_message_with_agent()`。
- **持久化狀態**: 僅寫入 `gateway_routing` 之 Turn 活躍標記；**訊息本文無持久化**。 [已核實]

---

### Step 11: Agent Dispatch & Executor Thread Handoff
- **代碼路徑**:
  - `gateway/run_turn.py:GatewayRunnerTurnMixin._handle_message_with_agent()` (L2181)
  - `gateway/run_turn.py:GatewayRunnerTurnMixin._run_agent()` (L2944)
  - `gateway/run_turn.py:GatewayRunnerTurnMixin._run_agent_inner()` (L4241)
  - `gateway/run_turn_runner.py:TurnRunner.run_sync()` (L1884)
  - `gateway/run_turn_runner.py:TurnRunner._run_conversation_with_approval()` (L1686)
- **處理邏輯**:
  1. `_handle_message_with_agent` 取得 `_PreparedTurn` 後，將 `message_text`、`persist_user_message`、`persist_user_timestamp`、`persist_user_display_kind` 與 `inbound_message_id = str(event.message_id)` 作為 kwargs 傳給 `_run_agent()`。
  2. `_run_agent_inner` 建立 `TurnContext` 與 `TurnRunner`，並在 asyncio event loop 中啟動 worker 執行緒調用 `TurnRunner.run_sync()`。
  3. `run_sync()` 在執行器執行緒中呼叫 `_prepare_turn_message(agent_history)` (L1614)，再呼叫 `_run_conversation_with_approval()` 組裝 `kwargs`：
     - `persist_user_message`: 使用者純文字或恢復引導
     - `persist_user_timestamp`: 訊息時間戳記
     - `persist_user_platform_id`: `str(ctx.inbound_message_id)` (Telegram 平台 `message_id`)
     - `task_id`: `ctx.session_id`
  4. 調用 `agent.run_conversation(...)`。
- **持久化狀態**: **無**。 [已核實]

---

### Step 12: Turn Facade & Admission Lease
- **代碼路徑**: `agent/turn_facade.py:run_conversation()` (L22)
- **處理邏輯**:
  1. 呼叫 `admit_durable_turn_lease()` (L62) 取得 Session 執行租約。
  2. 若 Session 在此早退或中斷，會調用 `carry_unadmitted_user_message()` 保存未准入之訊息以供後續恢復。
  3. 正常情況下，設置對話與 ContextVar 作用域，調用 `agent.conversation_loop.run_conversation()` (L146)。
- **持久化狀態**: **無**。 [已核實]

---

### Step 13: Agent Conversation Loop Entry
- **代碼路徑**:
  - `agent/conversation_loop.py:run_conversation()` (L1654)
  - `agent/conversation_loop.py:_run_conversation_turn()` (L1497)
- **處理邏輯**:
  1. `run_conversation` 在 `native_turn_images` context manager 內呼叫 `_run_conversation_turn()`。
  2. `_run_conversation_turn` 調用 `build_turn_context()` (L1532) 初始化該 Turn 的所有上下文與數據準備。
- **持久化狀態**: **無**。 [已核實]

---

### Step 14: Turn Context Assembly & User Message Staging
- **代碼路徑**: `agent/turn_context.py:build_turn_context()` (L980)
- **處理邏輯**:
  1. `_bind_turn_identity()` (L1022) 綁定 `task_id` 與 `turn_id`。
  2. `_stage_turn_user_message()` (L1053):
     - 將 `user_message`（原始文字）與 `persist_user_message` 整理為標準 message dict 格式：
       ```python
       user_msg = {
           "role": "user",
           "content": user_message,  # 或 persist_user_message
           "timestamp": persist_user_timestamp,
           "platform_message_id": persist_user_platform_id,
           "display_kind": persist_user_display_kind,
           "display_metadata": persist_user_display_metadata,
       }
       ```
  3. `append_message(messages, user_msg)` (L1058) 將訊息加入記憶體訊息串列 `messages` 中，記錄位置 `current_turn_user_idx = len(messages) - 1`。
  4. 恢復/建造 System Prompt (`restore_or_build_system_prompt`) 並確認 `_ensure_session_row()`。
  5. 若有 Sidecar/Plugin/Prefetch/Gateway 上下文，呼叫 `_stamp_api_content_sidecar()` (L1146) 寫入 `api_content`（此欄位保存包含加強 context 的完整模型 Prompt，而 `content` 維持使用者乾淨文字）。
  6. **呼叫 `_persist_turn_start(agent, messages, conversation_history, pending_cli_message)` (L1152) 觸發本文落盤**。
- **持久化狀態**: **進行中（於 Step 15 完成第一次持久化寫入）**。 [已核實]

---

### Step 15: 訊息本文第一次確認完成持久化寫入 (First Content Persistence Write)
- **代碼路徑**:
  - `agent/turn_context.py:_persist_turn_start()` (L963)
  - `agent/session_persistence.py:AgentSessionPersistenceMixin._persist_session()` (L347)
  - `agent/session_persistence.py:AgentSessionPersistenceMixin._flush_messages_to_session_db()` (L380)
  - `agent/session_persistence.py:_db_flush_collect()` (L200) & `_db_flush_write()` (L258)
  - `hermes_state_messages.py:SessionDB.append_messages_batch()` (L364)
  - `hermes_state_messages.py:SessionDB._insert_message_rows()` (L515)
- **處理邏輯**:
  1. `_persist_turn_start()` 經由 `_persist_under_lock()` (L468) 調用 `agent._persist_session(messages, conversation_history)`。
  2. `_persist_session()` 取得 `_persist_lock(self)` 鎖，並呼叫 `_flush_messages_to_session_db()`。
  3. `_db_flush_collect()` 掃描 `messages` 列表中未標記 `_DB_PERSISTED_MARKER` 的新訊息，組裝為 SQLite ROW dict：
     ```python
     row = {
         "role": "user",
         "content": content,                  # 使用者訊息本文 (經 expand_link_entities 與 surrogates 清理)
         "api_content": api_content,          # 若與 content 相同則為 None
         "timestamp": timestamp,              # float epoch 時間戳記
         "platform_message_id": platform_id,  # Telegram message_id (字串)
         "display_kind": display_kind,
         "display_metadata": display_metadata,
         ...
     }
     ```
  4. `_db_flush_write()` 調用 `SessionDB.append_messages_batch()`，在 **單一 SQLite 資料庫事務 (Transaction)** 內執行 SQL：
     ```sql
     INSERT INTO messages (
         session_id, role, content, tool_call_id, tool_calls, tool_name,
         effect_disposition, timestamp, token_count, finish_reason, reasoning,
         reasoning_content, reasoning_details, codex_reasoning_items, codex_message_items,
         platform_message_id, observed, _compressed_summary, active, api_content,
         display_kind, display_metadata, display_identity
     ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
     ```
  5. 寫入成功後，SQLite 傳回 `lastrowid`，系統在記憶體 message dict 上蓋章 `msg["_DB_PERSISTED_MARKER"] = True` 並設置 `msg["_row_id"] = cur.lastrowid`。
- **持久化狀態**: **【已核實】訊息本文首次成功寫入 SQLite `state.db` 的 `messages` 資料表**。

---

## 5. 訊息本文內容一致性核對

| 階段 / 物件 | 變數與欄位名稱 | 內容表現形式 | 是否與原始內容相同 | 原始碼位置與行號 |
| :--- | :--- | :--- | :--- | :--- |
| **1. 原始 Telegram Update** | `update.message.text` | 原始 Telegram 使用者輸入文字字串 | **基準原貌** | `plugins/platforms/telegram/adapter.py` (L6420, L6446) [已核實] |
| **2. 整理後 MessageEvent** | `event.text` | 字串（由 `expand_link_entities(message)` 回傳） | **條件成立才會不同**（若含 `text_link` 隱藏 URL，會在錨點後展開 ` (URL)`；無則相同） | `plugins/platforms/telegram/adapter.py` (L7081), `plugins/platforms/telegram/telegram_entities.py` (L24) [已核實] |
| **3. 寫入 DB 的 `messages.content`** | `messages.content` 資料表欄位 | 使用者文字（經 `expand_link_entities` 展開、時間戳記分離與 `sanitize_surrogates` 清洗） | **條件成立才會不同**（經超連結展開、時間戳剝離與 surrogate 清理） | `agent/turn_context.py` (L1015), `hermes_state_messages.py` (L267) [已核實] |
| **4. 寫入 DB 的 `messages.api_content`**| `messages.api_content` 資料表欄位 | 帶有 sidecar / plugin / prefetch 等 context 注入後的模型 Prompt 全文。若無額外 context 且與 `content` 相同，SQLite 中存為 `NULL` | **內含擴充上下文**（若有 Sidecar）或 **`NULL`** | `agent/turn_context.py` (L1146), `agent/session_persistence.py` (L200) [已核實] |

---

## 6. 身分與關鍵欄位實際傳遞軌跡表

| 欄位名稱 | 是否實際傳遞 / 傳遞至何處 | 最終落盤位置（檔案 / 資料表 / 欄位） | 備註與原始內容保存狀態 | 原始碼位置 |
| :--- | :--- | :--- | :--- | :--- |
| **`update_id`** | **是** (`MessageEvent.platform_update_id`) | 特定指令 (`/restart`): **`.restart_last_processed.json`**；一般訊息: **無 (不入 DB)** | 在 `/restart` 指令時寫入 JSON 做重啟去重；一般訊息隨 `MessageEvent` 在記憶體傳遞，未寫入 SQLite `messages` 表。 [已核實] | `plugins/platforms/telegram/adapter.py` (L7082), `gateway/slash_commands.py` (L570) |
| **`platform_message_id`** | **是** (`event.message_id` -> `inbound_message_id` -> `persist_user_platform_id`) | SQLite `state.db` -> **`messages.platform_message_id`** (`TEXT`) | 完整保存 Telegram 的 `message_id` (字串化)，用於重啟恢復與去重。 [已核實] | `gateway/run_turn.py` (L2181), `agent/turn_context.py` (L1022), `hermes_state_messages.py` (L263) |
| **`chat_id`** | **是** (`event.source.chat_id` -> `SessionSource.chat_id`) | SQLite `state.db` -> **`sessions.chat_id`** (`TEXT`) | 於 Session 主檔建立時落盤（舊報告 Step 9）；`messages` 表不重複記錄，靠 `session_id` 關聯。 [已核實] | `gateway/session_recovery.py` (L402), `hermes_state_sessions.py` (L297) |
| **`user_id`** | **是** (`event.source.user_id` -> `SessionSource.user_id`) | SQLite `state.db` -> **`sessions.user_id`** (`TEXT`) | 於 Session 主檔建立時落盤（舊報告 Step 9）；`messages` 表不重複記錄。 [已核實] | `gateway/session_recovery.py` (L402), `hermes_state_sessions.py` (L297) |
| **`session_key`** | **是** (`"agent:main:telegram:dm:<chat_id>"`) | SQLite `state.db` -> **`gateway_routing.session_key`** & **`sessions.session_key`** | 於 Session 建立與 Turn 活躍標記 (`_mark_durable_active_turn`) 時落盤；`TurnRunner` 設定 ContextVar `_SESSION_KEY`。 [已核實] | `gateway/session_persistence.py` (L440), `gateway/run_inbound.py` (L1755) |
| **`session_id`** | **是** (如 `"20260330-120000-abcd"`) | SQLite `state.db` -> **`messages.session_id`** (`TEXT`) | 作為外鍵與主關聯 key，於每一列 `messages` 寫入時落盤。 [已核實] | `agent/turn_context.py` (L980), `hermes_state_messages.py` (L26) |
| **`timestamp`** | **是** (`msg.date.timestamp()` -> `event.timestamp` -> `persist_user_timestamp`) | SQLite `state.db` -> **`messages.timestamp`** (`REAL`) | 儲存為 Unix Epoch 時間戳記 (float)，若訊息內含時間戳語法優先採用內文時間。 [已核實] | `plugins/platforms/telegram/adapter.py` (L7047), `gateway/run_turn.py` (L2136), `hermes_state_messages.py` (L523) |

---

## 7. 第一個本文落盤點細節 (First Persistence Point Details)

- **目標數據庫與資料表**: SQLite 數據庫 `state.db` -> **`messages` 資料表** [已核實]
- **呼叫順序 (Call Stack Sequence)**:
  1. `gateway/run_turn.py:GatewayRunnerTurnMixin._handle_message_with_agent()` (L2181)
  2. `gateway/run_turn.py:GatewayRunnerTurnMixin._run_agent_inner()` (L4241)
  3. `gateway/run_turn_runner.py:TurnRunner.run_sync()` (L1884)
  4. `gateway/run_turn_runner.py:TurnRunner._run_conversation_with_approval()` (L1686)
  5. `agent/turn_facade.py:run_conversation()` (L22)
  6. `agent/conversation_loop.py:run_conversation()` (L1654)
  7. `agent/conversation_loop.py:_run_conversation_turn()` (L1497)
  8. `agent/turn_context.py:build_turn_context()` (L980)
  9. `agent/turn_context.py:_persist_turn_start()` (L963)
  10. `agent/turn_context.py:_persist_under_lock()` (L468)
  11. `agent/session_persistence.py:AgentSessionPersistenceMixin._persist_session()` (L347)
  12. `agent/session_persistence.py:AgentSessionPersistenceMixin._flush_messages_to_session_db()` (L380)
  13. `agent/session_persistence.py:_db_flush_write()` (L258)
  14. `hermes_state_messages.py:SessionDB.append_messages_batch()` (L364)
  15. `hermes_state_messages.py:SessionDB._insert_message_rows()` (L515) -> 執行 `_INSERT_MESSAGE_SQL` (L26) [已核實]
- **寫入欄位與型別**:
  - `session_id` (`TEXT`): Session ID 字串
  - `role` (`TEXT`): `"user"`
  - `content` (`TEXT`): 使用者原始輸入訊息本文 (經超連結展開與 surrogates 清洗)
  - `api_content` (`TEXT` / `NULL`): 擴充 Prompt (若與 `content` 相同則存 `NULL`)
  - `timestamp` (`REAL`): Unix Epoch float
  - `platform_message_id` (`TEXT`): Telegram `message_id` 字串
  - `display_kind` (`TEXT` / `NULL`): UI 渲染類型
  - `display_metadata` (`TEXT` / `NULL`): 渲染元資料 JSON 字串
  - `active` (`INTEGER`): `1`
- **成功條件**:
  - 在 `SessionDB._execute_write()` (L386) 中，SQLite 成功取得 WAL 鎖定並執行 `COMMIT` 事務成功。
  - 記憶體中 message dict 被賦予 `_DB_PERSISTED_MARKER = True` 與 SQLite `_row_id`。 [已核實]

---

## 8. 緩衝區間、取消/失敗與平台 Update 接受邊界

### 8.1 本文落盤前的緩衝區間 (Buffer Window)
從 Telegram Adapter 收到 Update (Step 1) 開始，經歷：
1. `_pending_text_batches` (記憶體 Debounce/Batching 緩衝)
2. `asyncio.create_task` 背景 Task 調度
3. Gateway Inbound 門閥與 Session 建立
4. `TurnRunner` Executor 執行緒切換
5. `build_turn_context` 組合 Prompt

在 **Step 1 到 Step 14 期間，訊息本文完全僅存在於 Python 記憶體（Memory Objects）中**，尚未寫入 SQLite `messages` 資料表。 [已核實]

### 8.2 Telegram Update 接受邊界 (Acknowledgement Boundary)
- **Polling / Webhook 機制**:
  - Telegram Bot API ACK 屬於底层 HTTP / `python-telegram-bot` (PTB) 套件之通訊行為（Webhook 於 HTTP Server 回覆 200 OK 時 ACK；Polling 於帶有 `offset` 的下次 `getUpdates` 請求發出時 ACK）。
  - **專案程式碼層級無法直接控制 PTB 的 exact ACK 發送瞬間**。 [條件成立才會發生] / [待查]

### 8.3 取消與失敗應對機制
1. **中斷／取消 (Interrupt / Cancel)**:
   - 若 Turn 在准入完成後、LLM 呼叫前被取消（例如收到 `/stop` 或 SIGINT）：
     - **若取消發生於 Step 15 `_persist_turn_start()` 成功執行之後**：訊息本文已落盤於 SQLite `messages` 表，後續 Turn 標記為 `interrupted` 但 user 列安全保留。 [已核實]
     - **若取消發生於 Step 15 `_persist_turn_start()` 執行之前**（例如在 `admit_durable_turn_lease` 階段）：訊息本文尚未寫入 `messages` 表，改由 `carry_unadmitted_user_message` 於記憶體中暫存。 [條件成立才會發生]
2. **資料庫寫入異常與救援 (DB Persistence Exception / Divert)**:
   - `_persist_under_lock()` (L468) 捕獲 Exceptions 並 Log-and-Swallow，因此 DB 寫入失敗不會中斷 Turn；若 SQLite 遭遇 `StateDbReplacedError` 或 `StateDbCorruptError`，`_db_flush_failed()` (L299) 會嘗試調用 `divert_session_transcript_jsonl()` 將未落盤的訊息寫入急救 JSONL 檔。 [條件成立才會發生]

### 8.4 程式碼證明的「可能遺失資料的區間」(Data Loss Risk Windows)

依據程式碼實際寫入時機，以下區間若發生異常，資料將無法自動恢復：

1. **區間一：Telegram 已 ACK 但 SQLite `_persist_turn_start` 尚未成功執行前的硬崩潰**
   - **觸發條件**: Telegram 已收到 ACK，且系統在 Step 1 ~ Step 15 之間遭遇 `SIGKILL`、Hard OOM Kicker 或斷電。 [條件成立才會發生]
   - **結果分析**: 因 Telegram 已收到 ACK，Telegram 伺服器**不會重新發送**該 Update；而 Hermes 本地 SQLite 尚未執行 `INSERT INTO messages`。**該筆使用者訊息本文將永久遺失**。
2. **區間二：磁碟空間已滿 (Disk Full) 導致 SQLite 寫入與急救 JSONL 目錄皆無法寫入**
   - **觸發條件**: Step 15 執行 SQLite INSERT 時磁碟空間 100% 滿，且急救檔 `divert_session_transcript_jsonl` 也因空間不足無法建立。 [條件成立才會發生]
   - **結果分析**: 訊息本文無法落盤，記憶體釋放後資料遺失。

---

## 9. 狀態標註說明總結

- **【已核實】**: 經過源碼直接對照，邏輯與行號完全符合，為程式碼明確保證之行為。
- **【條件成立才會發生】**: 程式碼中存在對應 Exception 處理或特定分支邏輯，僅在硬體/環境/時間點條件滿足時觸發。
- **【待查】**: 本範疇外或涉及外部套件 (`python-telegram-bot`) / 模型 Provider 內部運作之未探討區域。
