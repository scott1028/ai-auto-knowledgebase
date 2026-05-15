---
source_url: https://developer.mozilla.org/en-US/blog/launching-new-front-end/
source_hash: a0d7f135
retrieved: 2026-05-16
title: MDN 新前端正式上線 (Launching MDN's new front end)
---

# MDN 新前端正式上線

> 原文發佈日期：2025-08-19,作者 The MDN Team。

## 重點摘要

- MDN 完全重寫了前端,從架構到設計都重新打造,今天正式推出。
- 技術選用策略:以 **Baseline 「Widely available」** 為主;若用到 Baseline 「Newly available」的功能,則搭配 polyfill 或漸進增強 (progressive enhancement) 處理。這讓 MDN 得以採用較新的 CSS 與 web components 來建構 UI。
- 改動目標在「精煉與一致性」,並非全面大改。

## UI 主要改進

- **字體 (typography) 改進**,提升可讀性。
- 新增 **code font**,讓不同平台上的程式碼呈現一致且高品質。
- 圖示改用 **Lucide** 圖示庫。
- 新增 **search modal**,可從 MDN 任何頁面快速搜尋內容。
- 頂部導覽 (top navigation) 重新設計,協助使用者更容易發現 MDN 的各類內容。

## 後續方向

MDN 團隊會持續迭代,歡迎讀者在 Discord 的 `#platform` 頻道交流,或到對應 GitHub repository 回報問題。

## 個人筆記

這篇文章本身較短,後續會有另一篇深入講「實作細節」的文章 (即同期的 `mdn-front-end-deep-dive`)。若要了解 MDN 採用 Baseline 標準的決策邏輯,直接看那篇 deep dive 比較划算。
