---
source_url: https://developer.mozilla.org/en-US/blog/javascript-temporal-is-coming/
source_hash: 6699120a
retrieved: 2026-05-16
title: JavaScript Temporal is coming
---

# JavaScript Temporal is coming

## In this article




- [What is JavaScript Temporal?](#what_is_javascript_temporal)
- [Core concepts](#core_concepts)
- [Temporal examples](#temporal_examples)
- [Trying Temporal and browser support](#trying_temporal_and_browser_support)
- [Acknowledgements](#acknowledgements)
- [See also](#see_also)








 ![JavaScript Temporal is coming title. A JavaScript logo, a clock graphic and a globe with orbiting bodies symbolizing calendars and time.](./featured.png)




# JavaScript Temporal is coming



 [Brian Smith](https://bsmth.de)

 January 24, 2025


 5 minutes read








 Implementations of the new JavaScript Temporal object are starting to be shipped in experimental releases of browsers.
This is big news for web developers because working with dates and times in JavaScript will be hugely simplified and modernized.

Applications that rely on scheduling, internationalization, or time-sensitive data will be able to use built-ins for efficient, precise and consistent dates, times, durations, and calendars.
We're a long way away from stable, cross-browser support, and there may be changes as implementations develop, but we can already take a look at Temporal as it stands now, why it's exciting, and what problems it solves.

To help you get up to speed, there are over [270 pages of Temporal docs on MDN](/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal) added this week, with detailed explanations and examples.




## What is JavaScript Temporal?

 To understand Temporal, we can look at JavaScript's `Date` object.
When JavaScript was created in 1995, the [`Date`](/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date) object was copied from Java's early, flawed `java.util.Date` implementation.
Java replaced this implementation in 1997, but JavaScript is stuck with the same API for almost 30 years, despite known problems.

The major issues with JavaScript's `Date` object are that it only supports the user's local time and UTC, and there's no time zone support.
Additionally, its parsing behavior is very unreliable, and `Date` itself is mutable, which can introduce hard-to-trace bugs.
There are other problems like calculations across Daylight Saving Time (DST) and historical calendar changes, which are notoriously difficult to work with.

All of these issues make working with dates and times in JavaScript complex and prone to bugs, which can have serious consequences for some systems.
Most developers rely on dedicated libraries like [Moment.js](https://momentjs.com/) and [date-fns](https://date-fns.org/) for better handling of dates and times in their applications.

Temporal is designed as a full replacement for the [`Date`](/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date) object, making date and time management reliable and predictable.
Temporal adds support for time zone and calendar representations, many built-in methods for conversions, comparisons and computations, formatting, and more.
The API surface has over 200 utility methods, and you can find information about all of them in the [Temporal docs on MDN](/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal).




## Core concepts

 In Temporal, the key concepts are that it has instants (unique points in history), wall-clock times (regional time), and durations.
The APIs have this overall structure to handle these concepts:

- Duration: `Temporal.Duration` the difference between two points in time

- Points in time:

- Unique points in time:

- As a timestamp: `Temporal.Instant`

- A date-time with a time zone: `Temporal.ZonedDateTime`

- Time-zone-unaware date/time ("Plain"):

- Full date and time: `Temporal.PlainDateTime`

- Just the date: `Temporal.PlainDate`

- Year and month: `Temporal.PlainYearMonth`

- Month and day: `Temporal.PlainMonthDay`

- Just the time: `Temporal.PlainTime`

- Now: using `Temporal.now` to get the current time as various class instances, or in a specific format




## Temporal examples

 Some of the most basic usages of Temporal include getting current dates and times as an ISO string, but we can see from the example below, that we can now provide time zones with many methods, which takes care of complex calculations you may be doing yourself:

js
```
// The current date in the system's time zone
const dateTime = Temporal.Now.plainDateTimeISO();
console.log(dateTime); // e.g.: 2025-01-22T11:46:36.144

// The current date in the "America/New_York" time zone
const dateTimeInNewYork = Temporal.Now.plainDateTimeISO("America/New_York");
console.log(dateTimeInNewYork);
// e.g.: 2025-01-22T05:47:02.555

```

Working with different calendars is also simplified, as it's possible to create dates in calendar systems other than Gregorian, such as Hebrew, Chinese, and Islamic, for example.
The code below helps you find out when the next Chinese New Year is (which is quite soon!):

js
```
// Chinese New Years are on 1/1 in the Chinese calendar
const chineseNewYear = Temporal.PlainMonthDay.from({
 monthCode: "M01",
 day: 1,
 calendar: "chinese",
});
const currentYear = Temporal.Now.plainDateISO().withCalendar("chinese").year;
let nextCNY = chineseNewYear.toPlainDate({ year: currentYear });
// If nextCNY is before the current date, move forward by 1 year
if (Temporal.PlainDate.compare(nextCNY, Temporal.Now.plainDateISO())
Working with Unix timestamps is a very common use case as many systems (APIs, databases) use the format to represent times.
The following example shows how to take a Unix Epoch timestamp in milliseconds, create an instant from it, get the current time with `Temporal.Now`, then calculate how many hours from now until the Unix timestamp:

js
```
// 1851222399924 is our timestamp
const launch = Temporal.Instant.fromEpochMilliseconds(1851222399924);
const now = Temporal.Now.instant();
const duration = now.until(launch, { smallestUnit: "hour" });
console.log(`It will be ${duration.toLocaleString("en-US")} until the launch`);
// "It will be 31,600 hr until the launch"
Currently, `toLocaleString` [doesn't output a locale-sensitive string](https://bugzilla.mozilla.org/show_bug.cgi?id=1839694) in the Firefox implementation, so durations above (`PT31600H`) are returned as a non-locale-sensitive [duration format](/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Duration#iso_8601_duration_format).
This may change as it's more of a design decision rather than a technical limitation as [formatting the duration](/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/PlainDate/since#using_since) is possible, so the polyfill and Firefox implementations may eventually converge.

There's a lot to highlight, but one pattern that I thought was interesting in the API is the `compare()` methods, which allow you to [sort durations](/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Duration/compare#sorting_an_array_of_durations) in an elegant and efficient way:

js
```
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




## Trying Temporal and browser support

 Support is slowly starting to be included in experimental browser releases, and Firefox appears to have the most mature implementation at this point.
In Firefox, Temporal is being built into the [Nightly version](https://www.mozilla.org/en-US/firefox/channel/desktop/) behind the `javascript.options.experimental.temporal` preference.
If you want to see the full compatibility story, you can check the (quite epic) [Temporal object Browser Compatibility section](/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal#browser_compatibility).

Here are the main browser bugs that track Temporal implementations:

- Firefox: [Build temporal in Nightly by default](https://bugzilla.mozilla.org/show_bug.cgi?id=1912757)

- Safari: [[JSC] Implement Temporal](https://bugs.webkit.org/show_bug.cgi?id=223166)

- Chrome: [Implement the Temporal proposal](https://issues.chromium.org/issues/42201538)

Additionally, you can visit [https://tc39.es/proposal-temporal/docs/](https://tc39.es/proposal-temporal/docs/) which has `@js-temporal/polyfill` available.
That means, you can open the developer tools on the TC39 docs page, and try some of the examples in the console in any browser without changing flags or preferences.

With experimental implementations landing, now is a good time to try out Temporal and become familiar with what will be the modern approach to handling dates and times in JavaScript.




## Acknowledgements



- Thanks to [Eric Meyer](https://meyerweb.com/) for his work on the topic.
It's been roughly 4 years since Eric's efforts to document [browser compatibility data](https://github.com/mdn/browser-compat-data/pull/10643) and scaffold the documentation [in his fork of mdn/content](https://github.com/meyerweb/content/tree/temporal/).

- [Joshua Chen](/en-US/community/spotlight/joshua-chen) for picking up the torch from Eric and getting a pull request together for [the MDN documentation](https://github.com/mdn/content/pull/37344).

- [André Bargull](https://bugzilla.mozilla.org/user_profile?user_id=339940) for the work on the Firefox Temporal implementation.




## See also



- [Fixing JavaScript Date – Getting Started](https://maggiepint.com/2017/04/09/fixing-javascript-date-getting-started/) by Maggie Pint (2017)










## Previous post
 Fix your website's Largest Contentful Paint by optimizing image loading
