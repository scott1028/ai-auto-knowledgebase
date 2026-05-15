---
source_url: https://developer.mozilla.org/en-US/blog/global-privacy-control/
source_hash: 4f36cf3c
retrieved: 2026-05-16
title: Global Privacy Control (GPC) 的意涵 (Implications of Global Privacy Control)
---

# Global Privacy Control (GPC) 的意涵

> 原文發佈日期：2025-03-15,作者 Lola Odelola (W3C TAG 成員)。

## 重點摘要

**GPC (Global Privacy Control)** 是 W3C Privacy Working Group 推出的新草案,用一個 HTTP signal 讓使用者表達「不要販賣或分享我的個資」的偏好。和 2009 年失敗的 DNT (Do Not Track) 最大差別:GPC **有法律後盾** —— 加州 Attorney General 已建議遵守 GPC 以符合 CCPA,並計畫對接歐盟 GDPR。

## 為何 GPC ≠ DNT

- DNT 雖被多數瀏覽器實作,但網站採用率極低 —— 因為沒有法律強制力,網站可選擇忽略。
- GPC 不同:CCPA 規範下,接收到 GPC signal 等同收到「Do Not Sell」請求,網站若忽略可能有法律後果。

## GPC 的兩種 signal

兩者都叫 `do-not-sell-or-share`,差別在範圍 (scope):

| 類型 | 範圍 | 例子 |
|---|---|---|
| **Interaction** | 單一網域 | 對 `https://nhs.uk` 關閉 (允許分享給藥房/保險);對 `https://tiktok.com` 開啟 (拒絕分享) |
| **Preference** | 整個瀏覽器全域 | 設定 preference 等於開啟 Global Privacy Control |

## 對網站擁有者的實作要求

### 1. 公告支援狀態:`/.well-known/gpc.json`

```json
{
  "gpc": true,
  "lastUpdate": "1997-03-10"
}
```

- `gpc`: `true` 表示伺服器承諾遵守 GPC 請求;`false` 表示不遵守;其他值代表狀態未知。
- `lastUpdate`: 採用 `YYYY-MM-DD` 或 ISO 8601 date-time。

### 2. 偵測 signal

伺服器端檢查 `Sec-GPC` header (Express 範例):

```js
app.get("/", function (req, res) {
  const gpcValue = req.header("Sec-GPC");
  if (gpcValue === "1") {
    optOutUser(userId);
  }
});
```

瀏覽器端可用 `navigator.globalPrivacyControl`:

```js
const gpcValue = navigator.globalPrivacyControl;
if (gpcValue) {
  optOutUser(userId);
}
```

帶有 GPC 的 HTTP request 會包含 header `Sec-GPC: 1`。

### 3. 收到 signal 後

由開發者/業務決定如何處理 —— 例如將該使用者從第三方追蹤、行銷流程中移除,流程類似一般的退出機制。CCPA 司法管轄下「必須」遵守。

## 瀏覽器與工具支援

- **原生支援**:Firefox、Brave、DuckDuckGo Privacy Browser (最新版,可在設定中開啟)。
- **擴充套件**:Microsoft Edge、Google Chrome 透過 GPC 擴充。
- **Express 中介層**:`npm i express-gpc`。

## 觀察

GPC 在「企業聚焦於 private advertising」的氛圍中,屬於少數真正把選擇權還給使用者的規範。對開發者整合成本低,瀏覽器支援也在成長。

## 個人筆記

實務上若服務目標市場包含加州/歐盟,優先實作 `/.well-known/gpc.json` 與 `Sec-GPC` header 偵測;同時也代表既有的 cookie consent banner 與 GPC 處理流程需要對齊,避免兩邊邏輯衝突。
