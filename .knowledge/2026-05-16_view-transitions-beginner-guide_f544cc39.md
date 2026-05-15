---
source_url: https://developer.mozilla.org/en-US/blog/view-transitions-beginner-guide/
source_hash: f544cc39
retrieved: 2026-05-16
title: CSS View Transitions 入門指南 (A beginner-friendly guide to view transitions in CSS)
---

# CSS View Transitions 入門指南

> 原文發佈日期：2025-10-09,作者 Yash Raj Bharti。

## 一句話

**View Transition API** (尤其是 Level 2 的 CSS `@view-transition` at-rule) 讓多頁應用 (MPA) 也能像 SPA 一樣,在切換頁面時擁有流暢的動畫過場 —— 而且最少只要「一行 CSS」。

## MPA 與 SPA 的差異複習

- **MPA**:每次導覽都向 server 取新頁,整頁 reload。容易實作大量獨立頁面。
- **SPA**:只載入一個 HTML,後續由 JavaScript 動態更新內容。互動較快但需要複雜的 client-side routing 與狀態管理。

以前「流暢的頁面過場」是 SPA 的專利。View Transition API + CSS `@view-transition` 讓 MPA 也辦得到,而且由於採用漸進增強 (progressive enhancement),不支援的瀏覽器只會略過動畫,網站照常運作。

## 瀏覽器支援現況

CSS View Transitions 規範分兩個 level:

| Level | 範圍 | 支援 |
|---|---|---|
| **Level 1** | 同一頁面內的過場 (透過 View Transition API) | Chrome / Edge / Safari ✅;Firefox 144 (beta 中) |
| **Level 2** | 跨頁面過場 (`@view-transition` at-rule) | Chrome 126+ / Edge 126+ / Safari 18.2+ ✅;Firefox 進行中 |

## 最簡範例 —— 一行就會動

```css
@view-transition {
  navigation: auto;
}
```

加在 stylesheet 就能在頁面切換時得到預設的 crossfade 過場。

### HTML

`index.html`:

```html
<h1>🏡 Homepage</h1>
<p>Hello, my name is Mrs. Whiskers...</p>
<p>...find out more about my <a href="hobbies.html">hobbies</a>.</p>
```

`hobbies.html`:

```html
<h1>🧶 My hobbies</h1>
<p>...</p>
<p>Thank you for your interest! You can return to the <a href="index.html">homepage</a> now.</p>
```

### style.css

```css
@view-transition {
  navigation: auto;
}

body {
  font-family: system-ui;
  max-inline-size: 400px;
  padding: 20px;
  text-align: center;
  margin: auto;
  box-sizing: border-box;
  line-height: 1.5;

  &.index    { background-color: oklch(0.9529 0.0444 332); }
  &.hobbies  { background-color: oklch(0.9588 0.0617 184.24); }
}
```

## 自訂過場 —— 用 `::view-transition-old` / `::view-transition-new`

兩個 pseudo-element 分別代表「離開的舊頁面」與「進來的新頁面」。

### 由右側滑入 (套用在 `hobbies.html`,進入時觸發)

```css
@keyframes slide-from-right {
  from { transform: translateX(100vw); }
  to   { transform: translateX(0); }
}

::view-transition-old(root) {
  animation: none;
}

::view-transition-new(root) {
  animation: slide-from-right 0.3s;
}
```

> **預設動畫**:瀏覽器原本會用 opacity + `mix-blend-mode: plus-lighter` 做 cross-fade。範例中用 `animation: none` 把它關掉,改用 `slide-from-right`。若想保留 cross-fade,可換不同的 `mix-blend-mode` 玩看看。

### 由右側滑出 (從 `hobbies.html` 回到 `index.html`)

```css
@keyframes slide-to-right {
  from { transform: translateX(0); }
  to   { transform: translateX(100vw); }
}

::view-transition-old(root) {
  animation: slide-to-right 0.3s;
  z-index: 2;
}

::view-transition-new(root) {
  animation: none;
}
```

## 關鍵心智模型

- `::view-transition-old(root)` = **要離開的頁面**
- `::view-transition-new(root)` = **將顯示的新頁面**
- 從 A 到 B 時:A 是 old、B 是 new。從 B 回到 A 時:角色互換。
- 預設動畫是 cross-fade;要客製就明確覆寫(必要時用 `animation: none` 取消預設)。

## 個人筆記

- 跨頁過場屬於 progressive enhancement —— 寫了不會壞,只是舊瀏覽器看不到動畫,適合直接加進現有 MPA。
- 完整 demo 程式碼可在 `mdn/dom-examples` repository 找到。
