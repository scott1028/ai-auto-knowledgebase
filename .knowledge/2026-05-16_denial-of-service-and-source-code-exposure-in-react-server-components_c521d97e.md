---
source_url: https://react.dev/blog/2025/12/11/denial-of-service-and-source-code-exposure-in-react-server-components
source_hash: c521d97e
retrieved: 2026-05-16
title: RSC 後續漏洞:DoS 與 Source Code Exposure (CVE-2025-55183/55184/67779/2026-23864)
---

# React Server Components 後續漏洞:DoS 與 Source Code Exposure

> 原文發佈日期：2025-12-11,後續更新 2026-01-26。作者 The React Team。這是 `critical-security-vulnerability-in-react-server-components` (CVE-2025-55182, RCE) 的「後續公告」 —— 研究人員在嘗試繞過該 patch 時又發現了 3 個新漏洞。

## TL;DR

- 上一輪 RCE patch 公開後,安全研究人員針對 patch 周邊路徑進行壓力測試,**陸續發現 3 個新漏洞**。
- 新漏洞**不會**造成 RCE(原 RCE 修補仍然有效),但會引發 **Denial of Service** 與 **Source Code Exposure**。
- **如果你已經為前次漏洞升級過,還是要再升級一次。**
- 升到 `19.0.3` / `19.1.4` / `19.2.3` 的 patch 仍**不完整**,需再升到 `19.0.4` / `19.1.5` / `19.2.4`。

## 漏洞列表

| CVE | 嚴重度 | CVSS | 類型 |
|---|---|---|---|
| `CVE-2025-55183` | Medium | 5.3 | Source Code Exposure |
| `CVE-2025-55184` | High | 7.5 | DoS — infinite loop / 高 CPU |
| `CVE-2025-67779` | High | 7.5 | DoS(內部發現的遺漏 case) |
| `CVE-2026-23864` | High | 7.5 | DoS(後續發現,2026-01-26 patched) |

## 受影響的套件與版本

與 CVE-2025-55182 完全相同的三個套件:

- `react-server-dom-webpack`
- `react-server-dom-parcel`
- `react-server-dom-turbopack`

受影響版本:`19.0.0` ~ `19.0.3`、`19.1.0` ~ `19.1.3`、`19.2.0` ~ `19.2.3`。

**修補版本:`19.0.4` / `19.1.5` / `19.2.4`**(往回 backport 到三條 release line)。

## 不受影響條件 (與前次相同)

- 純 client-side,React code 完全不跑在 server。
- 沒有用支援 RSC 的 framework / bundler / bundler plugin。

## 受影響的 framework 與 bundler

同前次:`next`、`react-router`、`waku`、`@parcel/rsc`、`@vite/rsc-plugin`、`rwsdk`。升級指引參照前一篇 (`critical-security-vulnerability-in-react-server-components`)。

## High Severity: 多個 DoS (CVE-2025-55184、CVE-2025-67779、CVE-2026-23864)

對 Server Function endpoint 送出特製 HTTP request,React 反序列化時觸發無限迴圈 / out-of-memory / 過高 CPU 使用:

```
+--------+   crafted HTTP    +----------------+
|attacker| ----------------> | Server         |
+--------+                   | Function       |
                             | endpoint       |
                             +----------------+
                                     |
                                     v
                             +----------------+
                             | React deserialize
                             +----------------+
                                     |
                  +------------------+------------------+
                  |                  |                  |
                  v                  v                  v
            +----------+      +----------+      +-------------+
            | infinite |      | out-of-  |      | excessive   |
            |  loop    |      | memory   |      | CPU         |
            +----------+      +----------+      +-------------+
                  |                  |                  |
                  +------------------+------------------+
                                     |
                                     v
                             server unresponsive
```

- 即使 app 自己沒有實作 Server Function,只要支援 RSC 就可能 vulnerable。
- 2025-12-11 的 patch 修了前 3 個 CVE;2026-01-26 後又發現「**第一次的 fix 還是不完整**」,於是有了 `CVE-2026-23864`,需再升一次。

## Medium Severity: Source Code Exposure (CVE-2025-55183)

對 vulnerable Server Function 送出惡意 HTTP request,可能讓 React **回傳 Server Function 的原始碼字串**。攻擊前提:該 function 顯式或隱式地把參數做了字串化 (stringify)。

範例 vulnerable Server Function:

```js
'use server';
export async function serverFunction(name) {
  const conn = db.createConnection('SECRET KEY');
  const user = await conn.createUser(name); // 隱式字串化,洩漏在 db
  return {
    id: user.id,
    message: `Hello, ${name}!`              // 顯式字串化,洩漏在 reply
  }
}
```

攻擊者可能拿到的回傳資料:

```
0:{"a":"$@1","f":"","b":"Wy43RxUKdxmr5iuBzJ1pN"}
1:{"id":"tva1sfodwq","message":"Hello, async function(a){console.log(\"serverFunction\");let b=i.createConnection(\"SECRET KEY\");return{id:(await b.createUser(a)).id,message:`Hello, ${a}!`}}!"}
```

→ Server Function 的原始碼被 stringify 後夾帶在 response 內。Patch 直接阻止把 Server Function 的 source code 字串化。

### 範圍說明

- **僅 source code 內 hardcoded 的 secret** 會被洩漏。
- **Runtime secret(如 `process.env.SECRET`)不受影響。**
- 洩漏範圍視 bundler 的 inlining 程度而定;**請以 production bundle 為準驗證**,不要只看 dev build。

## 為何 critical CVE 常引出 follow-up

原文有特別點出這個 pattern:

> 重大漏洞公開後,研究人員會仔細審視「相鄰程式碼路徑」尋找變體攻擊,確認初版 mitigation 是否能被繞過。這在整個業界都看得到 —— 例如 Log4Shell 之後也陸續報出多個 CVE。

> Additional disclosures can be frustrating, but they are generally a sign of a healthy response cycle.

## React Native 升級指引

- 沒有 monorepo / 沒裝 `react-dom` → `react` 版本只要 pin 住,不需額外動作。
- monorepo 環境 → 只升級下列(若有安裝):
  - `react-server-dom-webpack`
  - `react-server-dom-parcel`
  - `react-server-dom-turbopack`

不需升 `react` / `react-dom`,以避免 version mismatch。

## 時間線

- **12/03**:Andrew MacPherson 透過 Vercel 與 Meta Bug Bounty 回報 Source Code Exposure。
- **12/04**:RyotaK 回報初版 DoS 到 Meta Bug Bounty。
- **12/06**:React team 確認雙方問題並開始調查。
- **12/07**:初版 fix 完成,團隊驗證並規劃 patch。
- **12/08**:通知受影響的 hosting providers / OSS 專案。
- **12/10**:Hosting provider mitigation 上線,patch 驗證完成。
- **12/11**:Shinsaku Nomura 補回報另一個 DoS。
- **12/11**:Patch publish 到 npm,公開為 `CVE-2025-55183` + `CVE-2025-55184`;內部又發現遺漏 case,補上 `CVE-2025-67779`。
- **2026-01-26**:再發現 DoS,公開為 `CVE-2026-23864` 並 patch。

## 致謝

- **Source Code Exposure**:Andrew MacPherson (AndrewMohawk)
- **DoS (初版)**:RyotaK (GMO Flatt Security Inc.)、Shinsaku Nomura (Bitforest Co., Ltd.)
- **DoS (後續)**:Mufeed VH (Winfunc Research)、Joachim Viide、RyotaK、Xiangwei Zhang (Tencent Security YUNDING LAB)

## 個人筆記

- 若已套 `19.0.3` / `19.1.4` / `19.2.3`,**請立即再升到 `19.0.4` / `19.1.5` / `19.2.4`**,前者 patch 已被證實不完整。
- Source code exposure 雖然限定 hardcoded secret 才洩漏,但實務上 **bundler inlining** 可能把相鄰函式也一起 stringify;production bundle 是唯一可靠的判斷依據。
- 安全 patch 「複數輪 follow-up」是常態,Log4Shell 的歷史可以當教材;對應的 dependency 自動化應該對 RSC 套件設為「**即時告警 + 自動 PR**」而非月度 batch。
