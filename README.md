# FYP
---
## Flow diagram

flowchart TD
    A[WhatsApp 新訊息] --> B[自動回覆: 請稍等]
    B --> L[寫入 Logs: 原始對話+時間戳]

    L --> C[AI/規則判斷意圖<br/>enquiry / new / update / cancel]

    C -->|enquiry| ENQ[通知店主手動跟進<br/> `Draft 不建立`]
    ENQ --> END1[End]

    C -->|new| N1[計算 delivery_date<br/>if now<03:00 => 今日<br/>else => 明天]
    N1 --> N2[建立 DraftOrders 草稿<br/>狀態=pending]
    N2 --> N3[推送店主: 草稿內容<br/> `確認/修改/拒絕`]

    C -->|update| U1[讀取 DraftOrders/Orders<br/>by 訂單號/客戶]
    U1 --> U2[更新草稿或已確認訂單<br/>狀態=edited]

    C -->|cancel| K{now < 03:00 ?}
    K -->|是| K1[更新 Orders 狀態=cancelled<br/>通知雙方]
    K -->|否| K2[拒絕取消 + 通知雙方]

    N3 --> D{店主回覆}
    D -->|確認| O1[將草稿寫入 Orders<br/>狀態=confirmed]
    O1 --> O2[DraftOrders 狀態=confirmed]
    O1 --> O3[通知客戶: 已確認]
    D -->|修改| N2
    D -->|拒絕| R1[DraftOrders 狀態=rejected<br/>通知客戶]

    CRON[Cron 03:00] --> R[讀 Orders: delivery_date=今日<br/>狀態=confirmed]
    R --> SUM[按產品彙總 qty_required]
    SUM --> INV[讀 Inventory qty_onhand]
    INV --> MSG[推送店主產品列表<br/>今日需出貨 vs 現存]

---
## Sequence diagram

sequenceDiagram
    participant C as 客戶
    participant W as WhatsApp Bot<br/>(n8n + AI)
    participant LG as Logs(Sheets)
    participant D as DraftOrders(Sheets)
    participant O as Orders(Sheets)
    participant I as Inventory(Sheets)
    participant S as 店主

    Note over C,S: 客戶發訊息（詢價/推介/下單）
    C->>+W: WhatsApp 訊息
    W->>-C: "請稍等，店主即將處理"
    W->>LG: 寫入 Logs(原文, time, sender)

    W->>W: Intent routing<br/>enquiry/new/update/cancel

    alt enquiry
        W->>S: 通知店主手動回覆
    else new order
        W->>W: 計算 delivery_date<br/>if now<03:00 => 今日 else 明天
        W->>D: 建立草稿(pending)
        W->>S: 推送草稿 + [確認/修改/拒絕]
        S->>W: "確認"
        W->>O: 寫入正式訂單(confirmed)
        W->>D: 更新草稿狀態=confirmed
        W->>C: 回覆 "訂單已確認"
    else update order
        C->>W: "改訂單 #123 ..."
        W->>LG: 記錄修改訊息
        W->>D: 更新草稿(如未確認) 或
        W->>O: 更新正式訂單(如已確認)
        W->>C: 回覆 "已更新"
    else cancel order
        C->>W: "取消訂單 #123"
        W->>LG: 記錄取消訊息
        W->>W: 檢查 now < 03:00 ?
        alt before 03:00
            W->>O: 狀態=cancelled
            W->>C: "已取消"
            W->>S: "訂單已取消"
        else after 03:00
            W->>C: "03:00 後已入貨，不可取消"
            W->>S: "客戶嘗試取消(已拒)"
        end
    end

    Note over W,S: 每日 03:00 入貨提醒
    W->>O: 讀今日 confirmed 訂單
    W->>W: 彙總 qty_required
    W->>I: 讀現存 qty_onhand
    W->>S: 推送產品列表(今日需出貨 vs 現存)
