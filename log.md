# 互動與設計開發紀錄 (log.md)

本文件紀錄 HW4「AI Harness Systems Design and Analysis」的 AI 輔助設計與開發過程。

## 紀錄 1：確立系統設計主題與範圍
**時間**：2026-05-31
**互動過程摘要**：
- **Prompt / Request**：要求 AI 協助生成 Task 檔案設定目標，並挑選一個「較簡單實作的主題」。
- **AI 建議與決策**：AI 提出了四個選項（客服、旅遊、資料分析、程式碼審查），並根據「簡單實作、流程清晰、易於滿足至少三個工具」的需求，選擇了 **「智慧電商退換貨客服 Agent」**。
- **設計考量**：
  - 此情境為經典的 AI Agent 應用，容易定義多步驟工作流（查詢訂單 -> 確認退換貨政策 -> 執行退貨）。
  - 需要串接的工具可以很直觀地定義為三個 API：`get_order_details`, `check_return_eligibility`, `process_refund`。
  - 易於用 ReAct (Reason & Act) 框架來解釋 LLM 的 orchestration 邏輯。

## 紀錄 2：擴充書面報告內容與字數
**時間**：2026-05-31
**互動過程摘要**：
- **Prompt / Request**：使用者回覆「好啊，可以幫我用 markdown 來撰寫我們的書面報告嗎」。
- **AI 建議與決策**：AI 判斷稍早產生的初步 `report.md` 版本可能過於精簡，無法滿足作業「2–5 頁（A4 格式）」的要求。因此，AI 重新撰寫並大幅擴充了 `report.md` 的內容。
- **設計考量**：
  - 增加了背景說明的深度。
  - 詳細闡述了 ReAct (Reason + Act) 在這三個 Tool 之間的具體推論過程（Thought, Action, Observation）。
  - 在 Evaluation 區塊補充了 Human-in-the-loop (HITL) 的防呆機制以及解決 LLM 數學計算幻覺的應對方案。
