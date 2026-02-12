---
title: "AI Agent 開發的演進與新法"
date: 2026-02-12T00:43:28+08:00
draft: false
categories: ["AI"]
tags: ["AI Agent", "MCP"]
description: ""
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

看到一篇文章寫的不錯，紀錄留存一下，並且簡單的重點整理。

本文僅做紀錄與整理，觀點本身皆來自原文作者。

原文：https://www.facebook.com/100063744608911/posts/1526846902783449/?mibextid=wwXIfr&rdid=ABv4HxePsdb0LmOe


# **Agent 開發的三個階段演變**

1. **第一階段：工具連接期 (Function Calling / MCP)**
- **作法**：利用 LLM 的 API 與 Function Calling 能力，將 AI 與後端系統串接。後來演變成標準化的 MCP (Model Context Protocol)。
- **痛點**：技術門檻過高，能獨立完成的人很少。


2. **第二階段：多代理協作期 (A2A / Multi-Agent)**
- **作法**：類似 Google 提出的 Agent to Agent，讓多個 AI 各司其職。
- **痛點**：類似「微服務」亂象，過度設計 (Over-engineering)。管理多個 Agent 的 Context 很困難，且往往「一個 Agent + 多個 Tools」就能解決問題。


3. **第三階段：Coding Agent 與 SDK 成熟期 (現在)**
- **作法**：使用 Claude Code 或 GitHub Copilot 這類高度可自訂的 Coding Agent。
- **突破**：透過自訂 Instructions 和 SKILLs (流程知識)，不僅能寫程式，還能處理非開發任務。這是最直覺有效的路徑。

# **AI 時代的 MVP (最小可行性產品) 開發新路徑**

過去的軟體開發往往是「從輪子造車」(從底層一行行寫)，風險在於最後一刻才知道客戶喜不喜歡。AI 讓「理想的 MVP 路徑」(滑板車 -> 腳踏車 -> 汽車) 變得成本極低且可行：

1. **驗證概念 (滑板車)**：用 ChatGPT + Prompt 確認流程是否在 AI 能力範圍內。
2. **原型試用 (腳踏車)**：用 Coding Agent (如 Claude Code) + Instructions + MCP + Skills 組合出流程，讓內部專家或用戶試用。
3. **規模化產品 (汽/機車)**：利用 Agent SDK 將步驟 2 驗證好的邏輯移植並部署，提供給終端客戶使用。

# **小結**

- **Kent Beck 的預言成真**：AI 讓 90% 的傳統編碼技能價值歸零，但剩下的 10% (架構設計、產品思維、工程方法) 價值放大了 1000 倍。
- **角色轉變**：開發者不再只是寫 Code，更像是架構師，利用 AI 快速迭代產品體驗。
