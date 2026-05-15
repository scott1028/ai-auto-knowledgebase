---
source_url: https://developer.mozilla.org/en-US/blog/javascript-temporal-is-coming/
source_hash: 6699120a
retrieved: 2026-05-16
title: JavaScript Temporal 即將登場 (JavaScript Temporal is coming)
---

# JavaScript Temporal 即將登場

> 原文發佈日期：2025-01-24,作者 Brian Smith。

## 重點摘要

- `Temporal` 是 JavaScript 內建處理日期/時間的新物件,設計上是 `Date` 的「完整替代品」。
- 各家瀏覽器開始在實驗版中實作 (Firefox Nightly 進度最完整)。
- MDN 同期上架超過 270 頁的 Temporal 文件。
- 對於需要排程、國際化 (internationalization)、時區敏感資料的應用,意義重大。

## 為何需要 Temporal:`Date` 的歷史包袱

- JavaScript 1995 年誕生時,`Date` 直接複製早期有缺陷的 `java.util.Date` API。
- Java 1997 年就替換掉自己的實作,JS 卻被卡了將近 30 年。
- 已知缺陷:
  - 只支援 local time 與 UTC,**沒有時區概念**。
  - 解析行為不可靠 (parsing 不一致)。
  - `Date` 是 mutable,造成難以追蹤的 bug。
  - 夏令時間 (DST) 與歷史曆法切換的計算特別痛苦。
- 結果大家都得依賴 `Moment.js`、`date-fns` 之類的第三方庫。

## 核心概念

Temporal 把時間拆成三類:**instants** (歷史上唯一的時點)、**wall-clock times** (區域時間)、**durations** (時長)。對應的 API:

- 時長:`Temporal.Duration`
- 唯一時點:
  - 時間戳:`Temporal.Instant`
  - 含時區的日期時間:`Temporal.ZonedDateTime`
- 不帶時區 (Plain):
  - 完整日期時間:`Temporal.PlainDateTime`
  - 只有日期:`Temporal.PlainDate`
  - 年月:`Temporal.PlainYearMonth`
  - 月日:`Temporal.PlainMonthDay`
  - 只有時間:`Temporal.PlainTime`
- 取「現在」:`Temporal.Now`

## 範例

### 取得當前時間 (含時區)

```js
const dateTime = Temporal.Now.plainDateTimeISO();
console.log(dateTime); // 2025-01-22T11:46:36.144

const dateTimeInNewYork = Temporal.Now.plainDateTimeISO("America/New_York");
console.log(dateTimeInNewYork); // 2025-01-22T05:47:02.555
```

### 計算下一個農曆新年 (Chinese 曆法)

```js
const chineseNewYear = Temporal.PlainMonthDay.from({
  monthCode: "M01",
  day: 1,
  calendar: "chinese",
});
const currentYear = Temporal.Now.plainDateISO().withCalendar("chinese").year;
let nextCNY = chineseNewYear.toPlainDate({ year: currentYear });
if (Temporal.PlainDate.compare(nextCNY, Temporal.Now.plainDateISO()) <= 0) {
  nextCNY = nextCNY.add({ years: 1 });
}
console.log(
  `The next Chinese New Year is on ${nextCNY.withCalendar("iso8601").toLocaleString()}`,
);
// e.g.: 1/29/2025
```

### Unix timestamp 換算 + 計算距現在的小時數

```js
const launch = Temporal.Instant.fromEpochMilliseconds(1851222399924);
const now = Temporal.Now.instant();
const duration = now.until(launch, { smallestUnit: "hour" });
console.log(`It will be ${duration.toLocaleString("en-US")} until the launch`);
// polyfill: "It will be 31,600 hr until the launch"
// Firefox Nightly: "It will be PT31600H until the launch"
```

> ⚠️ Firefox 對 `Duration.toLocaleString` 目前還沒輸出 locale-aware 字串,輸出的是 ISO 8601 duration (`PT31600H`)。屬於設計待定,未來可能與 polyfill 收斂。

### 排序時長 —— `compare()` 靜態方法

```js
const durations = [
  Temporal.Duration.from({ hours: 1 }),
  Temporal.Duration.from({ hours: 2 }),
  Temporal.Duration.from({ hours: 1, minutes: 30 }),
  Temporal.Duration.from({ hours: 1, minutes: 45 }),
];

durations.sort(Temporal.Duration.compare);
console.log(durations.map((d) => d.toString()));
// [ 'PT1H', 'PT1H30M', 'PT1H45M', 'PT2H' ]
```

## 試用方式 & 瀏覽器支援

- **Firefox Nightly**:目前最成熟。需在 `about:config` 開啟 `javascript.options.experimental.temporal`。
- **Safari**:`[JSC] Implement Temporal` 追蹤中。
- **Chrome**:`Implement the Temporal proposal` 追蹤中。
- 任何瀏覽器都可以到 `https://tc39.es/proposal-temporal/docs/`,在 DevTools console 用 `@js-temporal/polyfill` 跑範例。

## 致謝 (摘錄)

- Eric Meyer:約 4 年前開始整理 MDN 上 Temporal 的相容性資料與文件骨架。
- Joshua Chen:接手把 MDN 文件 PR 完成。
- André Bargull:Firefox 的 Temporal 實作。

## 延伸閱讀

- "Fixing JavaScript Date – Getting Started" by Maggie Pint (2017)

## 個人筆記

- API 總方法超過 200 個,實際導入時建議先選定 3~4 個 use case 對應到 `PlainDate` / `ZonedDateTime` / `Duration` / `Instant`,而不是一次學完。
- `compare()` 設計成 static method 拿來餵 `Array.prototype.sort` 的 pattern 很乾淨,值得學起來。
- 對 server-side Node.js,可以先用 polyfill 上線,等原生穩定後再切換,介面幾乎一致。
