# HW4 Infographics: 智慧電商退換貨客服 Agent

本文件提供 AI Harness
系統的視覺化設計，包含系統架構、工具呼叫序列以及決策流程圖。

## 1. 系統架構圖 (System Architecture)

此架構展示了使用者、LLM 控制器、記憶體與外部工具之間的關係。

```mermaid
graph TD
    User([User]) -->|Natural Language Request| LLM[LLM Controller]
    
    subgraph AI Harness System
        LLM <-->|Read / Write Context| Memory[(Conversation Memory)]
        LLM <-->|Function Calling| ToolsLayer{Tools Layer}
        
        ToolsLayer --> T1[get_order_details]
        ToolsLayer --> T2[check_return_eligibility]
        ToolsLayer --> T3[process_refund]
    end
    
    T1 -.->|API Request| DB[(Order Database)]
    T2 -.->|API Request| PolicyDB[(Policy Database)]
    T3 -.->|API Request| Logistics[(Logistics System)]
    
    LLM -->|Natural Language Response| User
```

## 2. 序列圖 (Sequence Diagram)

展示 Agent 如何運用工具處理用戶的退貨請求。

```mermaid
sequenceDiagram
    actor User
    participant Agent as LLM Agent
    participant T1 as Tool: get_order_details
    participant T2 as Tool: check_return_eligibility
    participant T3 as Tool: process_refund

    User->>Agent: "我想退貨，訂單編號是 ORD-123"
    Agent->>T1: get_order_details(order_id="ORD-123")
    T1-->>Agent: {status: "delivered", items: ["Laptop"], delivery_date: "2026-05-25"}
    
    Note over Agent: Reason: 已送達 6 天，商品是 Laptop。<br/>需要檢查 3C 產品退貨政策。
    
    Agent->>T2: check_return_eligibility(item_category="3C", days_since_delivery=6)
    T2-->>Agent: {is_eligible: true, reason: "3C 產品 7 天內可退貨"}
    
    Note over Agent: Reason: 資格符合，準備執行退款流程。
    
    Agent->>T3: process_refund(order_id="ORD-123", refund_reason="user request")
    T3-->>Agent: {success: true, return_tracking_number: "RTN-999"}
    
    Agent->>User: "沒問題！您的筆電符合 7 天內退貨資格。我已為您安排退貨，您的寄件單號為 RTN-999。"
```

## 3. 工作流程圖 (Workflow Flowchart)

展示 Agent 內部決策與多步驟執行的邏輯樹。

```mermaid
flowchart TD
    Start([接收用戶訊息]) --> Intent{意圖識別}
    Intent -->|非退貨請求| GeneralChat[進行一般對話回覆]
    Intent -->|退貨請求| CheckOrder[呼叫 get_order_details]
    
    CheckOrder --> OrderExist{訂單是否存在?}
    OrderExist -->|否| AskOrder[請用戶提供正確訂單編號]
    OrderExist -->|是| CalcDays[計算送達天數與商品類別]
    
    CalcDays --> CallPolicy[呼叫 check_return_eligibility]
    CallPolicy --> Eligible{符合退貨資格?}
    
    Eligible -->|否| Reject[拒絕退貨並說明政策原因]
    Eligible -->|是| CallRefund[呼叫 process_refund]
    
    CallRefund --> ProvideTracking[提供退貨單號給用戶]
    
    GeneralChat --> End([結束對話回合])
    AskOrder --> End
    Reject --> End
    ProvideTracking --> End
```
