# Homework 4 — AI Harness Systems Design and Analysis
**專案名稱：智慧電商退換貨客服 Agent (Smart E-commerce Return Assistant)**

---

## 一、問題定義與應用背景 (Problem Definition & Context)

### 1.1 應用背景
在現代電子商務平台中，售後服務（特別是退換貨處理）是客服中心面臨的最大宗請求之一。傳統的自動化客服（如基於決策樹的 Chatbot 或關鍵字回覆系統）存在顯著的限制：
1. **缺乏彈性**：只能提供制式化的回答，無法處理用戶多變的自然語言描述。
2. **缺乏上下文理解**：無法根據用戶的具體訂單狀態、商品類型以及複雜的退貨政策進行動態的多步驟判斷。
3. **無法執行具有副作用的操作**：通常只能提供資訊，無法直接與後端物流或金流系統互動並執行退款。

### 1.2 問題定義與解決方案
為了提升客戶滿意度並大幅降低人力客服成本，本專案設計了一個「**智慧電商退換貨客服 Agent**」。
此系統將大型語言模型（LLM）作為核心的「系統控制器（System Controller）」。透過 AI Harness 的編排，Agent 能夠：
- 透過自然語言理解（NLU）精準捕捉用戶的退貨意圖。
- 自主規劃（Plan）需要哪些資訊。
- 透過 Function Calling 機制呼叫後端 API (Tools) 來查詢訂單狀態、比對退貨政策。
- 最終執行（Execute）或拒絕退款請求，實現 End-to-End 的全自動化退換貨流程。

---

## 二、AI Harness 系統設計 (AI System Architecture)

本系統採用基於 LLM 的 Agent 架構，將系統分為三個核心層次：Controller、Memory、與 Tools。

### 2.1 LLM Controller (大語言模型控制器)
LLM 扮演系統的大腦，負責意圖識別與多步驟決策。系統採用 **ReAct (Reason + Act)** 模式。
在每個對話輪次中，LLM 會被提示進行以下循環：
- **Thought (思考)**：分析當前對話狀態，判斷是否具備足夠資訊來回應。
- **Action (行動)**：若資訊不足，決定呼叫哪一個外部 Tool。
- **Observation (觀察)**：接收 Tool 的回傳結果（如 JSON 格式的 API 回應）。
- 循環上述步驟，直到 LLM 認為可以給出最終解答（Final Answer）。

### 2.2 Memory (記憶與狀態管理模組)
為了讓 Agent 能夠進行連貫的多步驟互動，系統實作了記憶模組：
- **Conversation Buffer (短期對話記憶)**：
  負責紀錄當前的對話上下文（Context），包含用戶的原始請求、LLM 的回覆、以及先前呼叫工具的回傳結果。這確保了 Agent 不會在同一個對話中重複詢問已知的訂單編號。
- **System Prompt (系統預設提示)**：
  賦予 Agent 的 Persona（例如：「你是一個專業且有禮貌的退換貨客服專家」），並嚴格規範其不可自行捏造退貨政策（防範 Hallucination）。

### 2.3 Tools Layer (工具整合層)
封裝好的 API 函數，提供給 LLM 呼叫以獲取外部資訊或執行狀態變更（Action）。LLM 透過定義明確的 OpenAPI 規格或 Function Signature 與這些工具互動。

---

## 三、工具設計 (Tools / Function Calling Design)

為了完成退換貨的完整流程，系統為 Agent 設計了以下三個核心工具。每個工具都具備明確的輸入參數與回傳結構。

### 3.1 Tool 1: `get_order_details` (查詢訂單資訊)
- **設計目的**: 用於根據訂單編號查詢使用者的訂單詳細資訊。這是啟動退貨流程的第一步。
- **輸入參數 (Input)**:
  - `order_id` (string): 用戶提供的訂單編號（格式如 ORD-XXXXX）。
- **回傳格式 (Output JSON)**:
  - `status` (string): 訂單當前狀態 (如 "delivered", "processing", "shipped")。
  - `delivery_date` (string): 送達日期（ISO 8601 格式，若未送達則為 null）。
  - `items` (array): 包含商品 ID、名稱與 `category`（類別）的列表。

### 3.2 Tool 2: `check_return_eligibility` (檢查退換貨資格)
- **設計目的**: 電商平台的退貨政策因商品而異（例如生鮮食品不可退、3C 產品限 7 天內）。此工具輸入商品類別與購買天數，查詢規則引擎以確認是否符合退貨政策。
- **輸入參數 (Input)**:
  - `item_category` (string): 從訂單資訊中獲取的商品類別。
  - `days_since_delivery` (integer): 距離商品送達已經過的天數（由 Agent 計算得出）。
- **回傳格式 (Output JSON)**:
  - `is_eligible` (boolean): 系統判定是否可退貨。
  - `reason` (string): 具體的政策說明，供 Agent 引用以向用戶解釋。

### 3.3 Tool 3: `process_refund` (執行退貨與退款)
- **設計目的**: 啟動物流與金流的退款流程。**這是一個具有副作用 (Side-effect) 的操作**，Agent 必須在嚴格確認 `is_eligible` 為 true 後才能呼叫。
- **輸入參數 (Input)**:
  - `order_id` (string): 確認要退貨的訂單編號。
  - `refund_reason` (string): 用戶提供的退貨原因（歸檔用）。
- **回傳格式 (Output JSON)**:
  - `success` (boolean): 系統處理是否成功。
  - `return_tracking_number` (string): 產生的退貨寄件單號，供用戶去超商或物流站寄件。

---

## 四、Agent Workflow (多步驟任務執行與編排邏輯)

本系統的 Orchestration (流程控制) 完全交由 LLM 基於 ReAct Prompt 進行自主導航，其標準運作流程如下：

1. **意圖識別與資訊收集**：
   - 用戶提出：「我前幾天買的鍵盤壞了，想要退貨，訂單是 ORD-99887。」
   - LLM 識別為退貨意圖，且提取出 `order_id="ORD-99887"`。
2. **第一階段推論 (Action 1)**：
   - **Thought**: 我需要確認這個訂單的詳細內容以及它送達了多久。
   - **Action**: 呼叫 `get_order_details(order_id="ORD-99887")`。
   - **Observation**: 收到 JSON，得知 `delivery_date` 是 5 天前，`items` 中包含類別為 `3C_Electronics` 的鍵盤。
3. **第二階段推論 (Action 2)**：
   - **Thought**: 商品是 3C 產品，已經送達 5 天。我需要確認它是否符合退貨政策。
   - **Action**: 呼叫 `check_return_eligibility(item_category="3C_Electronics", days_since_delivery=5)`。
   - **Observation**: 收到 JSON `{is_eligible: true, reason: "3C產品於7天鑑賞期內可無條件退貨"}`。
4. **第三階段推論 (Action 3 - Side Effect)**：
   - **Thought**: 該商品符合退貨資格，我可以協助用戶執行退款並產生退貨單號。
   - **Action**: 呼叫 `process_refund(order_id="ORD-99887", refund_reason="鍵盤損壞")`。
   - **Observation**: 收到 `{success: true, return_tracking_number: "RTN-5566"}`。
5. **最終回應 (Final Answer)**：
   - Agent 將收集到的所有資訊轉化為自然語言回覆給用戶：「您的鍵盤符合 7 天鑑賞期的退貨政策。我已經為您申請退款，請使用退貨單號 **RTN-5566** 將商品寄回，我們收到後會盡快處理！」

---

## 五、系統評量與優化方法 (Evaluation & Optimization)

為了確保 AI 系統在正式上線前的可靠性，我們設計了以下 Evaluation 框架：

### 5.1 評量指標 (Metrics)
1. **Tool Selection Accuracy (工具選擇準確率)**：
   - 衡量 Agent 是否在正確的時間點呼叫了正確的工具。例如：Agent 絕對不可以在未呼叫 `check_return_eligibility` 的情況下，直接呼叫 `process_refund`。
2. **Task Completion Rate (任務完成率)**：
   - 透過模擬真實用戶對話（包含提供錯誤訂單號碼、超時退貨等 Edge Cases），計算 Agent 成功引導用戶至合理終點（成功退貨或合理拒絕）的百分比。
3. **Hallucination Rate (幻覺率)**：
   - 嚴格檢驗 Agent 在拒絕用戶時的解釋，是否 100% 來自 `check_return_eligibility` 的 `reason` 欄位，而非 LLM 自行編造的政策。

### 5.2 系統優化策略
- **Human-in-the-loop (HITL)**：對於金額大於特定門檻的退款（例如超過 3 萬台幣），在呼叫 `process_refund` 之前，系統會先中斷 Agent 流程，發送確認請求給真人客服進行覆核。
- **Prompt Engineering 優化**：若發現 Agent 計算日期天數容易出錯，可以再增加一個名為 `calculate_date_difference` 的小工具，將數學計算也交由程式碼執行，進一步降低 LLM 的幻覺。
