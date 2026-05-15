---
source_url: https://react.dev/blog/2025/10/07/react-compiler-1
source_hash: 20c7bca2
retrieved: 2026-05-16
title: React Compiler 1.0 正式發佈 (React Compiler v1.0)
---

# React Compiler 1.0 正式發佈

> 原文發佈日期：2025-10-07,作者 Lauren Tan、Joe Savona、Mofei Zhang。

## TL;DR

- **React Compiler 1.0** 今日 stable 釋出 —— 在 build time 對 component / hook 進行**自動 memoization**,完全不需改寫程式碼。
- 適用於 React 與 React Native;已在 Meta 大型 production app (例如 Meta Quest Store) 戰鬥過,**production ready**。
- 內含 compiler-powered lint rules,透過 `eslint-plugin-react-hooks` 的 `recommended` / `recommended-latest` 預設直接套用。
- 與 Expo、Vite、Next.js 合作:**新 app 從一開始就可預設啟用 compiler**。

## 為什麼這件事重要

過去開發者需要手動用 `useMemo` / `useCallback` / `React.memo` 才能避免不必要的 re-render;Compiler 把這件事**自動化**,而且做得**比手動更精準**:可以對「early return 之後」的值做 memoize —— 這是 `useMemo` / `useCallback` 辦不到的。

範例:即使 component 中途 return,後續的 expression 也可以被 memoize:

```jsx
import { use } from 'react';

export default function ThemeProvider(props) {
  if (!props.children) {
    return null;
  }
  // The compiler can still memoize code after a conditional return
  const theme = mergeTheme(props.theme, use(ThemeContext));
  return (
    <ThemeContext value={theme}>
      {props.children}
    </ThemeContext>
  );
}
```

## 工作原理 (Deep Dive)

```
+---------------+      +---------+      +---------+      +-------------+
| your source   | ---> | Babel   | ---> | AST     | ---> | React       |
| (.jsx / .tsx) |      | plugin  |      |         |      | Compiler    |
+---------------+      +---------+      +---------+      +------+------+
                                                                |
                                                                v
                                                         +-------------+
                                                         |  novel HIR  |
                                                         | (CFG-based) |
                                                         +------+------+
                                                                |
                                  +-----------------------------+
                                  |                             |
                                  v                             v
                          +----------------+          +-------------------+
                          | data-flow &    |          | Rules of React    |
                          | mutability     |          | validation passes |
                          | analysis       |          | (diagnostics)     |
                          +-------+--------+          +-------------------+
                                  |
                                  v
                          +------------------+
                          | granular         |
                          | auto memoization |
                          | (incl. after     |
                          |  conditionals)   |
                          +------------------+
```

- 目前**以 Babel plugin 形式實作**,但 compiler 本身與 Babel 大幅解耦 —— 把 AST 降階成自家的 **HIR (High-Level Intermediate Representation)**,基於 **CFG (Control Flow Graph)**。
- 多個 compiler pass 仔細分析 data flow 與 mutability,以決定哪些值可以被 memoize。
- 同時有 **validation passes** 編碼 Rules of React;違反規則時透過 `eslint-plugin-react-hooks` 拋出診斷 —— 經常能挖出 React code 裡的潛在 bug。

## 安裝

```sh
npm install --save-dev --save-exact babel-plugin-react-compiler@latest
# 或
pnpm add --save-dev --save-exact babel-plugin-react-compiler@latest
# 或
yarn add --dev --exact babel-plugin-react-compiler@latest
```

新增於 1.0 的 compiler 優化:**支援 optional chain 與 array index 作為依賴**,減少 re-render、提升 UI 反應速度。

## Production 數據 (Meta Quest Store)

- Initial load + cross-page navigation 改善 **最高 12%**
- 某些 interaction **快 2.5× 以上**
- Memory 使用維持中性 (neutral)

> 「Your mileage may vary」—— 建議用自己的 app 實測。

## 向後相容

- 相容 **React 17 起**。
- 還沒升到 React 19?在 compiler config 設定 minimum target,加上 `react-compiler-runtime` 依賴即可使用。

## ESLint 整合 — Rules of React

React Compiler 內含 ESLint rule,**不需安裝 compiler 即可使用**(linter 與 compiler 解耦),所以升級 `eslint-plugin-react-hooks` 沒有副作用。

- **如果已裝 `eslint-plugin-react-compiler`,現在可移除**,改用 `eslint-plugin-react-hooks@latest` 即可(感謝 `@michaelfaith` 的貢獻)。

安裝:

```sh
npm install --save-dev eslint-plugin-react-hooks@latest
# 或
pnpm add --save-dev eslint-plugin-react-hooks@latest
# 或
yarn add --dev eslint-plugin-react-hooks@latest
```

Flat Config:

```js
// eslint.config.js
import reactHooks from 'eslint-plugin-react-hooks';
import { defineConfig } from 'eslint/config';

export default defineConfig([
  reactHooks.configs.flat.recommended,
]);
```

Legacy Config:

```json
// .eslintrc.json
{
  "extends": ["plugin:react-hooks/recommended"]
}
```

新規則(由 React Compiler 啟用)範例:

| Rule | 抓什麼 |
|---|---|
| `set-state-in-render` | 在 render 過程中 `setState` 造成 render loop |
| `set-state-in-effect` | effect 內昂貴的工作 |
| `refs` | render 期間不安全地存取 refs |

## `useMemo` / `useCallback` / `React.memo` 還用嗎?

Compiler 預設會自動 memoize,**多數情況比手動寫的還精準**(且能涵蓋 early return 等情境)。但下列場景**仍有需要手動 hooks**:

- **作為 effect dependency 的 memoized 值** —— 需要嚴格控制何時被視為「改變」以避免重複 fire。

策略建議:

| 新程式碼 | 既有程式碼 |
|---|---|
| 預設仰賴 compiler;只在需要「精準控制」時補 `useMemo` / `useCallback` | **保留現有 memoization**(移除可能改變 compilation 輸出);或仔細測試後再移除 |

## 新 app 預設啟用

與 Expo / Vite / Next.js 合作:

- **Expo SDK 54** 以上 → compiler 預設啟用

```sh
npx create-expo-app@latest
```

- **Vite / Next.js** → 在 `create-vite` / `create-next-app` 中可選擇 compiler-enabled template

```sh
npm create vite@latest
npx create-next-app@latest
```

## 既有 app 漸進式導入

官方有 **incremental adoption guide**,涵蓋:

- gating strategies
- compatibility checks
- rollout tooling

可控速啟用 compiler,不需一次切換。

## swc / oxc 支援 (experimental)

- 與 swc team (Kang Dongyoon, `@kdy1dev`) 合作中:**Next.js 啟用 React Compiler 時 build 效能已顯著提升**。建議使用 Next.js **15.3.1 以上**獲得最佳 build 效能。
- 與 oxc team 合作中。等 rolldown 在 Vite 中穩定支援、oxc compiler 支援上線後,docs 會更新遷移指引。
- 在那之前 Vite 用戶繼續用 `vite-plugin-react`(把 compiler 當 Babel plugin)。

## 升級時的注意事項

Compiler 把「auto-memoization 完全當作效能優化」設計;未來版本可能改變 memoization 的精細度,**極少數**情況下可能造成行為變化(例如某個被 memoize 的值同時被某 `useEffect` 當 dependency,memoization 改變後 effect 觸發次數隨之改變)。

實務建議:

- 遵守 Rules of React(linter 抓得到大部分違規)。
- 持續 E2E test。
- **若無 test coverage**:用 `--save-exact` (npm/pnpm) 或 `--exact` (yarn) **pin 死版本** (`1.0.0` 而非 `^1.0.0`),手動升級並逐次驗證。

## 歷史

近十年的工程努力:

```
2017 ─── Prepack          (探索 compiler;後關閉)
  │
  │       Hooks 設計時即考慮「未來會有 compiler」
  ▼
2021 ─── Xuan Huang 展示 React Compiler 新原型
  │
  │       原型最終被重寫,但驗證可行性
  ▼
       Joe Savona / Sathya Gunasekaran / Mofei Zhang / Lauren Tan
       重寫成 CFG-based HIR 架構 → 精準分析與 type inference
  │
  ▼
2025-10 ── React Compiler 1.0 stable
```

## 個人筆記

- 1.0 釋出的核心訊號:**「Meta 已內部驗證」+「Vite / Next.js / Expo 預設整合」** —— 採用障礙幾乎為零。
- 「early return 後也能 memoize」這點對複雜 component 特別有價值,以前要把 hook 拉到 return 之前的 workaround 可以收回了。
- 但對 effect dependency 的 case 還是要小心 —— compiler 改變 memoization 的精細度可能讓 effect 觸發次數變動,**E2E test 是升級安全網**。
- 若無 test coverage,務必 pin 死 `1.0.0`(而非 `^1.0.0`),這是這篇文章罕見地給出「保守作法」的地方。
- 取代 `eslint-plugin-react-compiler`:現在直接用 `eslint-plugin-react-hooks@latest` 即可,設定簡化。
