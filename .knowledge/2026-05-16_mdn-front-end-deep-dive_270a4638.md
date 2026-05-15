---
source_url: https://developer.mozilla.org/en-US/blog/mdn-front-end-deep-dive/
source_hash: 270a4638
retrieved: 2026-05-16
title: MDN 新前端架構深度解析 (Under the hood of MDN's new frontend)
---

# MDN 新前端架構深度解析

> 原文發佈日期：2026-04-08,作者 Leo McArdle。配對於同期 `launching-new-front-end` 那篇官宣;本篇是真正的技術細節。

## TL;DR

MDN 把舊有的 React SPA (yari) 完全重寫,改採:

- **Lit web components**(處理「島嶼式互動」)
- **自製 Server Components**(用 Lit 的 `html` template literal 在 Node.js 端 SSR)
- **Declarative Shadow DOM (DSD)** 漸進增強
- **扁平 component 目錄**搭配自動 lazy-load
- **Rspack** 取代 Webpack 作為建置工具

成果:啟動時間從 ~2 分鐘 → **2 秒**,bundle 只載當下頁面需要的 CSS/JS,UI 改動牽動範圍最小。

## MDN 架構速覽 (內容如何上線)

1. **Markdown 原稿**:技術寫手、合作夥伴、社群在數個 git repository 中維護。
2. **Build tool**:把 Markdown 轉成 HTML,並輸出附帶 metadata 的 JSON 檔。
3. **Frontend SSR**:遍歷 JSON,生成完整頁面 (含瀏覽器相容性表格、l10n、導覽選單)。
4. **靜態檔輸出**:HTML / CSS / JS 上傳到 cloud bucket → CDN → 全球讀者。

## 為什麼要重寫:舊架構 (yari) 的痛點

- 一開始用 Create React App,後來 `eject` 掉,Webpack 設定變得超複雜,build script 也很 hacky。
- CSS 同時混用 Sass + CSS variables,沒有 scoping,改 A 元件常意外動到 B。
- 無法切分 CSS,只能 ship 一大塊 render-blocking 的 CSS。
- 核心問題:**React app 是內容的 wrapper**。文件主體 (build tool 產的 HTML) 是 `dangerouslySetInnerHTML` 塞進去,React 無法觸及。內容中需要互動的小區塊 (例如 code block 的 Copy 按鈕) 只能用原生 DOM API 手刻 —— 結果就是 React + DOM 兩套並存,有時甚至雙實作。

## 過渡方案:web components (Lit)

### 案例 1 —— Scrim (與 Scrimba 合作的互動教學嵌入)

需求:預設不發送 user data 給 Scrimba (在使用者主動點開前不 load `<iframe>`),且要能在頁內 fullscreen ↔ inline 之間切換。

做法:用 Lit 寫成自訂元素 `<scrim-inline>`:

```js
export class MDNScrimInline extends LitElement {}

static properties = {
  url: { type: String },
  _fullscreen: { state: true },
  _scrimLoaded: { state: true },
};
```

在 `willUpdate` lifecycle 內把 `url` 加上 `via=mdn&embed=`;`render()` 回傳一個 `<dialog>`,內部依 `_scrimLoaded` 決定要顯示按鈕還是真的 `<iframe>`。互動以 Lit 的 `@click` / `@close` 綁定。

使用方式只是塞一個 custom element:

```html
<scrim-inline
  url="https://v2.scrimba.com/the-frontend-developer-career-path-c0j/~0lr"
  scrimtitle="The Request-Response Cycle"></scrim-inline>
```

> 作者觀察:**對於狀態不複雜的 UI,Lit 比 React 更簡潔**,而且 Lit 的 `html` template literal 不需要編譯。

### 案例 2 —— Interactive examples (CSS/JS/HTML 頁面上的「Try it」區塊)

舊架構分散在 4 個 git repo,authoring 體驗很糟。新做法把 React 版的 Playground 拆成多個 web components:

- `<play-editor>`:CodeMirror 編輯器
- `<play-console>`:console 訊息呈現
- `<play-runner>`:渲染當前編輯器狀態
- `<play-controller>`:在以上元件間傳遞事件與狀態

外層包成 `<interactive-example>`,內部會掃描頁面上的 `<code>` blocks 餵給 controller。

關鍵手法:**用 Lit 的 React 整合,可以把 web components 直接放進舊的 React app**,所以遷移可以「逐元件」進行,不必一次大改。

Markdown 端的 authoring 是:

```md
{{InteractiveExample("CSS Demo: background-repeat")}}

```css interactive-example-choice
background-repeat: space;
```
```

## React Server Components 不適用,所以自己做

作者引用了 React 官方文件:傳統 SPA 「為了驗證沒變化,需要下載並 parse 額外 ~75 KB 的 library」。RSC 本身解了這問題,但**強制配特定 framework**,遷移成本等同重寫。

那既然反正要重寫,就重新思考 MDN 的本質 —— 大部分內容是 HTML/CSS,**真正需要互動的只是「孤島」(islands)**。

→ 結論:**每個互動是一個獨立 web component;其餘是純靜態 HTML,在 server 端組裝即可。** 沒有 SPA wrapper,也沒有 wrapper-邊界問題。

## 自製的 Server Components

採用 component-based(而非 EJS 之類的 template language),理由是讓 CSS 也能按元件切分。Server Component 也用 Lit 的 `html` template literal:

```js
export class Navigation extends ServerComponent {
  render(context) {
    return html`
      <nav class="navigation" data-open="false">
        <div class="navigation__logo">${Logo.render(context)}</div>
        <button class="navigation__button" type="button"
          aria-expanded="false" aria-controls="navigation__popup"
          aria-label="Toggle navigation"></button>
        <div class="navigation__popup" id="navigation__popup">
          <div class="navigation__menu">${Menu.render(context)}</div>
          <div class="navigation__search" data-view="desktop">
            <mdn-search-button></mdn-search-button>
          </div>
        </div>
      </nav>
      <mdn-search-modal id="search"></mdn-search-modal>
    `;
  }
}
```

在 Node 端用 Lit 提供的工具把上述產出渲染成 HTML,**順便把 Lit web components 渲染成 Declarative Shadow DOM** —— 相容瀏覽器在 JS 載入前就已經有 Shadow DOM + CSS。

## 「只 ship 必要的東西」—— 扁平 component 結構

`./components/` 下每個元件是扁平目錄,固定檔名:

```
components/example-component
├── element.css   # 給 web component (Shadow DOM 內)
├── element.js    # web component (export MDNExampleComponent, define <mdn-example-component>)
├── global.css    # 一律全站載入
├── server.css    # 用到此 server component 時才載
└── server.js     # server component (extend ServerComponent)
```

部分規則靠 linter 強制,部分靠執行時拋錯。

### Web component 自動 lazy-load

頁面載入時跑:

```js
for (const element of document.querySelectorAll("*")) {
  const tag = element.tagName.toLowerCase();
  if (tag.startsWith("mdn-")) {
    const component = tag.replace("mdn-", "");
    import(`../components/${component}/element.js`);
  }
}
```

效果:

- 工程師不必手動 import 元件;當 HTML 元素用就好。
- 內容 Markdown 可以直接放 custom element 或透過 macro 生成。
- **只有頁面上存在的元件 JS 才會載**。
- 修一個元件,其他元件 cache 不破。

SSR 端也預設把每個 web component 都渲染成 DSD(除非元件 opt-out),避免 JS 載入後造成 layout shift。

### 進階小技巧:`<mdn-dropdown>` 的 CSS-only fallback

```js
render() {
  return html`
    <slot name="button" @click=${this._toggleDropDown}></slot>
    <slot name="dropdown" ?hidden=${!this.open && this.loaded}></slot>
  `;
}

firstUpdated() {
  this.loaded = true;
}

static properties = {
  loaded: { type: Boolean, reflect: true },
};
```

搭配 CSS:

```css
:host(:not([loaded], :focus-within)) {
  slot[name="dropdown"] {
    display: none;
  }
}
```

語意:**JS 還沒載入時,純靠 CSS `:focus-within` 提供 dropdown 行為**;JS 一載入就反射 `loaded` 屬性接管。導覽列裡的下拉選單因此在頁面渲染當下就可用。

### 沒有 DSD 怎麼辦

DSD 還沒 Widely Available。針對 fallback:`global.css` 載到每一頁,給特定元件最低限度的 placeholder 樣式以避免 layout shift,例如:

```css
mdn-button {
  display: inline-flex;
  vertical-align: middle;
}
```

### Server component CSS 的精準載入

`ServerComponent` 基類在 `render` 前後記錄哪些元件實際輸出了內容到 `componentsUsed: Set`,`OuterLayout` 再依此產生 `<link rel="stylesheet">`:

```js
const { componentsUsed, compilationStats } = asyncLocalStorage.getStore();
const styles = componentsUsed
  .flatMap((component) =>
    compilationStats.assets.filter(
      (name) => name === `${component.toLowerCase()}.css`,
    ),
  )
  .map((path) => html`<link rel="stylesheet" href=${path} />`);
```

→ 每頁只載「本頁實際用到的元件 CSS」。

## 效能取捨:為何採用「小檔大量」

HTTP/2、HTTP/3 並行下載 + connection reuse 翻轉了「合併資產」傳統智慧。配合 web components 非同步獨立載入,可以一邊載一邊「逐元件」變互動。重複造訪時 cache 命中率也更高 —— 改了某元件不會影響其他元件的 cache。

benchmark 顯示「打包成一大包」**只有在 cold cache 才打成平手或稍快**。build 設定保留了把小元件再合併成 bundle 的能力,以備未來 benchmark 需要。

## Baseline 作為決策依據

決策原則 (在團隊內推):

- **Baseline 「Widely Available」**:直接用。
- **Baseline 「Newly Available」**:先討論,看是 polyfill 還是漸進增強。
- **Baseline 「Limited Availability」**:認真想是不是真的需要。

用到範例:
- Custom Elements、Shadow DOM 已是 Widely Available。
- DSD 在 newer 範圍 → 漸進增強搭配 `global.css` fallback。
- 對 `light-dark()` 想擴用到 `<img>` 上,寫了 PostCSS mixin:

```css
@mixin light-dark --baseline-img, url("./icons/status/limited.svg"),
  url("./icons/status/limited-dark.svg");
```

當 newly → widely 後,自動移除 polyfill,避免長期累積負擔。

## 開發環境

### 舊架構痛點

- 預設啟動約 2 分鐘。
- `package.json` 命令一堆,要懂哪個跳過哪些步驟。
- 增加圖片這種小改動常需重啟 server。
- 預設不跑 SSR,要另一條更慢的命令才能除錯 SSR 問題。

### 新架構

- **2 秒**啟動。基本只要一條:

```bash
npm run start
```

- 速度提升主因:**Rspack 取代 Webpack**(Rust 重寫,API 相容)。
- Rspack 設定 650 LOC,雖不算少但邏輯直接,不依賴黑盒。
- SSR 與 dev 共用同一條路徑,行為更接近 production;只有改 Rspack config 才需重啟。

## 個人筆記

- **「islands of interactivity」是這次重寫的核心心智模型**。如果你正在 SPA 與 SSR 之間抉擇,但內容本來就 80%+ 是靜態,值得參考這條路徑(Lit + 自製 SC + DSD)。
- 元件 lazy-load 的關鍵是**命名約定**(`mdn-` prefix → `components/<name>/element.js`)。在自家專案實作時,這個約定可以更早建立,後續可省下大量 import 維護。
- `<mdn-dropdown>` 那段 CSS-only fallback 是值得學的 pattern:**用 `:focus-within` + `loaded` 屬性反射,讓元件在 JS 未到位前就可用**。
- Rspack 已可作為 Webpack 大型遷移目標;對 600+ LOC 的 webpack 設定遷移友善。
