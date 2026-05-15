---
source_url: https://developer.mozilla.org/en-US/blog/global-privacy-control/
source_hash: 4f36cf3c
retrieved: 2026-05-16
title: Implications of Global Privacy Control
---

# Implications of Global Privacy Control

## In this article




- [Trust and data collection](#trust_and_data_collection)
- [GPC and Do Not Track (DNT)](#gpc_and_do_not_track_dnt)
- [How GPC may give more control to users](#how_gpc_may_give_more_control_to_users)
- [What GPC looks like for website owners](#what_gpc_looks_like_for_website_owners)
- [What to do when receiving a GPC signal](#what_to_do_when_receiving_a_gpc_signal)
- [Trying GPC and browser support](#trying_gpc_and_browser_support)
- [Summary](#summary)








 ![Implications of Global Privacy Control title.
](./featured.png)




# Implications of Global Privacy Control



 [Lola Odelola](https://bsky.app/profile/lolaodelola.bsky.social)

 March 15, 2025


 5 minutes read








 Privacy has been a focal point for the World Wide Web Consortium (W3C) for the last few years, with their release of their [Privacy Principles](https://www.w3.org/TR/privacy-principles/) and browser vendors working on what host of tools should replace third-party cookies.
It makes sense, then, that the [Global Privacy Control](https://globalprivacycontrol.org) (GPC) has gained traction and is on a standards track, with the Privacy Working Group having recently [published the first working draft](https://www.w3.org/TR/gpc/).
This article takes a look at the draft, and how it will look for website owners and users who want to make use of GPC.




## Trust and data collection

 According to the UK Government's Center for Ethics and Innovation, 57% of respondents agree that collecting [personal data "is useful for creating products and services that benefit them as individuals"](https://www.gov.uk/government/publications/public-attitudes-to-data-and-ai-tracker-survey).
Only 46% of respondents trust that big tech companies will let them make decisions about how their data is used, and that number drops to 31% when referring to social media companies.

We can interpret this to mean that most respondents distrust social media and big tech companies regarding decision-making and consent as it relates to their data.
There's clearly a desire from users to have more control over how personal data is collected and shared, but striking a balance that's easy to control and enforce is delicate.




## GPC and Do Not Track (DNT)

 This isn't the first time that a tracking prevention mechanism has reached the W3C and been implemented by browsers.
In 2009 the [Do Not Track (DNT) header](/en-US/docs/Web/HTTP/Reference/Headers/DNT) was created as a way for web users to express their tracking preferences.
While it was widely implemented in browsers, it had a low adoption rate with websites.

The main problem with DNT was the lack of legal and regulatory backing it received.
Website owners could decide if they'd observe the DNT signal and there were no legal repercussions if they chose not to.
This is where GPC is different.

At the time of writing, the Attorney General for California has recommended observation of GPC to comply with [CCPA](https://oag.ca.gov/privacy/ccpa).
There are also intentions to work with the European Union's GDPR:

The GPC signal will be intended to communicate a Do Not Sell request from a global privacy control, as per [CCPA-REGULATIONS §999.315](https://www.oag.ca.gov/sites/all/files/agweb/pdfs/privacy/oal-sub-final-text-of-regs.pdf) for that browser or device, or, if known, the consumer.
Under the GDPR, the intent of the GPC signal is to convey a general request that data controllers limit the sale or sharing of the user's personal data to other da ta controllers ([GDPR Articles 7 & 21](https://eur-lex.europa.eu/legal-content/EN/TXT/PDF/?uri=CELEX:32016R0679)).
Over time, the GPC signal may be intended to communicate rights in other jurisdictions. – [globalprivacycontrol.org/#about](https://globalprivacycontrol.org/#about)

The signal also has support from various browsers and extensions including Mozilla's Firefox, Brave and DuckDuckGo's Privacy Browser.

GPC is different from other proposals being discussed and developed in that it gives power to the web user.
Google is working on the [User-Agent Client Hints](https://wicg.github.io/ua-client-hints/) specification in the Web Platform Incubator Community Group, and [Bounce Tracking Mitigation](https://developers.google.com/privacy-sandbox/protections/bounce-tracking-mitigations) is on the Privacy Working Group's charter.
While both proposals have their merits, neither gives the user a choice in the way that GPC does.




## How GPC may give more control to users

 Web users want to have more autonomy over their data.
They want to know who has it, where it's going and why, and they want to be able to consent to how their data moves between parties.
GPC proposes two browser signals which can be set on all HTTP requests as headers.
The two signals will be an interaction and a preference both called do-not-sell-or-share, which differ by scope.

Interaction is set per domain, for example, a web user may want to allow the National Health Service `https://nhs.uk` to share their data with their pharmacy or health insurance, so they could switch the do-not-sell-or-share interaction for `https://nhs.uk` off.
The same user may not want `https://tiktok.com` to share their data with anyone so they'd turn the interaction on for `https://tiktok.com`.

The scope of the interaction is for a domain, and, the scope for the preference encompasses all interactions per browser.
Setting a preference is setting a Global Privacy Control.




## What GPC looks like for website owners

 It's a requirement to return the GPC support resource as a JSON object from a well-known URI (`/.well-known/gpc.json`) with these members:

json
```
{
 "gpc": true,
 "lastUpdate": "1997-03-10"
}

```

Which has this meaning:

[`gpc`](#gpc)

The value of the `gpc` is either `true` (the server intends to abide by GPC requests) or `false`, to indicate that it does not.
For any other value the origin's support is unknown.

[`lastUpdate`](#lastupdate)

A full-date (`YYYY-MM-DD`) or date-time (`YYYY-MM-DDTHH:mm:ss.sssZ`) indicating the time at which the statement of support was made.
Later changes to the meaning of the GPC standard should not affect the interpretation of the resource for legal purposes.

For handling requests with a GPC signal, developers can implement a listener by checking for the [`Sec-GPC`](/en-US/docs/Web/HTTP/Reference/Headers/Sec-GPC) HTTP header in requests.
An Express app might look like this:

js
```
// Any route
app.get("/", function (req, res) {
 // Check for a `Sec-GPC` header:
 const gpcValue = req.header("Sec-GPC");
 if (gpcValue === "1") {
 // Signal detected
 optOutUser(userId);
 }
});

```

You can alternatively use the [`globalPrivacyControl`](/en-US/docs/Web/API/Navigator/globalPrivacyControl) property in the browser:

js
```
const gpcValue = navigator.globalPrivacyControl;
if (gpcValue) {
 // Signal detected
 optOutUser(userId);
}

```

The examples above uss a placeholder `optOutUser(userId)` for illustration, but the backend should call whatever logic is needed to satisfy the user's choice.

The HTTP response would look something like this if the user has set GPC to "true":

http
```
200 OK
Access-Control-Allow-Origin: *
Connection: Keep-Alive
Content-Encoding: gzip
Content-Type: text/html; charset=utf-8
…
Sec-GPC: 1

```




## What to do when receiving a GPC signal

 It's up to the developer/business to decide how to treat the signal, for example, removing the user's details from third-party tracking or marketing, following a similar procedure as to when users opt out of sharing data for marketing purposes.
If in CCPA jurisdiction, the signal must be observed to avoid legal repercussions.




## Trying GPC and browser support

 GPC is currently available in the latest versions of Firefox, Brave and DuckDuckGo's Privacy Browser and users can [turn the signal on in their settings (PDF)](https://globalprivacycontrol.org/GPC_for_Users.pdf).
In addition to browser config, [GPC extensions](https://globalprivacycontrol.org/#download) are available for Microsoft Edge and Google Chrome.

Developers can test GPC in Express apps using the [express-gpc middleware](https://www.npmjs.com/package/express-gpc), which you can to your project using `npm i express-gpc`.




## Summary

 GPC is an exciting specification, in a sea of companies hyper-focussed on how to do private advertising, it's refreshing to have a specification that prioritizes the needs of the web user.
With growing browser support and not much integration needed for developers, it's a step toward greater transparency and control for web users.

[Lola Odelola](https://bsky.app/profile/lolaodelola.bsky.social) is a web standards technologist who works to make web standards accessible to developers. Lola sits on the W3C Technical Architecture Group and participates in various W3C groups as a W3C Invited Expert.










## Previous post
 JavaScript Temporal is coming
