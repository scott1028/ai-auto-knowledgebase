---
source_url: https://developer.mozilla.org/en-US/blog/view-transitions-beginner-guide/
source_hash: f544cc39
retrieved: 2026-05-16
title: A beginner-friendly guide to view transitions in CSS
---

# A beginner-friendly guide to view transitions in CSS

## In this article




- [Quick refresher on MPAs vs SPAs](#quick_refresher_on_mpas_vs_spas)
- [Browser support for view transitions](#browser_support_for_view_transitions)
- [Creating your first view transition](#creating_your_first_view_transition)
- [Going beyond the default transition](#going_beyond_the_default_transition)
- [Wrapping up](#wrapping_up)








 ![](./featured.png)




# A beginner-friendly guide to view transitions in CSS



 [Yash Raj Bharti](https://github.com/yashrajbharti)

 October 9, 2025


 5 minutes read








 Imagine if your site could animate smoothly between pages – taking you from say `index.html` to `about.html` – without a jarring reload. Well, this is possible now, thanks to the support for [View Transition API](/en-US/docs/Web/API/View_Transition_API) in modern browsers.

View transitions were once exclusive to single-page applications ([SPAs](/en-US/docs/Glossary/SPA)). In this post, we'll explore how view transitions bring smooth, animated navigation to multi-page applications (MPAs).




## Quick refresher on MPAs vs SPAs

 Multi-page applications (MPAs) and single-page applications (SPAs) are two common approaches to web development, each with its own advantages and disadvantages.

- An MPA loads a new page from the server for each navigation, resulting in a full page reload. MPAs are simpler to build for large apps with many distinct pages.

- An SPA, on the other hand, loads a single HTML page and updates its content dynamically via JavaScript, offering faster interactions and a smoother user experience, but often requiring more complex client-side routing and state management.

For a long time, smooth animated transitions were possible only in SPAs. With the View Transition API – and now the CSS `@view-transition` at-rule – MPAs can now achieve similar effects. CSS view transitions are designed with progressive enhancement in mind. So if a browser doesn't support them, the site will still work because the CSS is treated as a hint and applied only when supported.




## Browser support for view transitions

 Before we dive into the specifics of how to use this feature, it's worth noting where view transitions are supported today. The CSS View Transitions specification has two levels:

- [Level 1](https://drafts.csswg.org/css-view-transitions-1/) enables transitions within a single page using the [View Transition API](/en-US/docs/Web/API/View_Transition_API). This is already supported in Chrome, Edge, and Safari. Firefox support is available in version 144 (currently in beta).

- [Level 2](https://drafts.csswg.org/css-view-transitions-2/) allows transitions across multiple pages via the [`@view-transition`](/en-US/docs/Web/CSS/Reference/At-rules/@view-transition) at-rule. It is currently supported in Chrome 126+, Edge 126+, and Safari 18.2+. At the time of writing this article, Level 2 is still a [work in progress in Firefox](https://bugzilla.mozilla.org/show_bug.cgi?id=1860854), but support is expected in the future.

In browsers without view transition support, sites will continue to work normally with standard page navigation as the transitions are treated as a progressive enhancement and won't break your site.

![](/en-US/blog/view-transitions-beginner-guide/baseline.png)




## Creating your first view transition

 Let's dive deeper and see view transitions in action.

As the simplest example, you can enable transition by adding just a few lines of CSS to your code:

css
```
@view-transition {
 navigation: auto;
}

```

And that's it! The basic version is just a few lines of CSS, which is incredible if you ask me.

To see the effect of this line of code, let's build two demo pages. You can also try adding this code to any existing multi-page application.

The first page, `index.html`, just contains a heading, few paragraphs, and a link to navigate to the second page:

html
```

# 🏡 Homepage

Hello, my name is Mrs. Whiskers. Welcome to my personal website!

 Everyone needs a place on the web, even a cat. If you're curious, you can find
 out more about my [hobbies](hobbies.html).

```

Let's get our second page, `hobbies.html`, ready. This too is a simple page with a heading, two short paragraphs, and a link back to the first page:

html
```

# 🧶 My hobbies

 When I'm not busy napping, I love to play with yarn and chase laser pointers.
 I'm also an avid bird watcher.

 Thank you for your interest! You can return to the
 [homepage](index.html) now.

```

Finally, add the following recipe to your `style.css` file:

css
```
@view-transition {
 navigation: auto;
}

```

```
body {
 font-family: system-ui;
 max-inline-size: 400px;
 padding: 20px;
 text-align: center;
 margin: auto;
 box-sizing: border-box;
 line-height: 1.5;

 &.index {
 background-color: oklch(0.9529 0.0444 332);
 }

 &.hobbies {
 background-color: oklch(0.9588 0.0617 184.24);
 }
}

a {
 color: oklch(0.0867 0.0419 261.53);
}

```

And that's all you need to create a view transition. With just this one line of CSS, you'll see a seamless transition between the two pages. In the past, this kind of smooth transition was possible only in SPAs using JavaScript. With the CSS `@view-transition` at-rule, the browser can now handle transitions across a multi-page application natively, without needing JavaScript.






## Going beyond the default transition

 Now let's go a step further and customize the view transition. Here we'll create a slide-in and slide-out effect, a simple animation to understand how view transitions work under the hood.

Let's add two different style sheets to our two HTML pages. Add this to `index.html`:

html
```

-

```

…and this to `hobbies.html`:

html
```

-

```

Time to get familiar with some CSS selectors (pseudo-elements) that control view transitions:

css
```
::view-transition-old(root),
::view-transition-new(root) {
 /* Write some cool CSS here */
}

```

Let's understand how these pseudo-elements work: [::view-transition-old](/en-US/docs/Web/CSS/Reference/Selectors/::view-transition-old) and [::view-transition-new](/en-US/docs/Web/CSS/Reference/Selectors/::view-transition-new) pseudo-elements reference the old and new pages, respectively, allowing us to style the transition between them. When we navigate from `index.html` to `hobbies.html`, the old page is the one we are coming from (`index.html`), and the new page is the one we are going to (`hobbies.html`).

Let's add a slide animation so that `hobbies.html` enters with a smooth sliding effect. We'll call this animation slide-from-right. Add the following code to `hobbies.css`:

css
```
@keyframes slide-from-right {
 from {
 /* Arrive from the right */
 transform: translateX(100vw);
 }
 to {
 /* Come into view */
 transform: translateX(0);
 }
}

::view-transition-old(root) {
 animation: none;
}

::view-transition-new(root) {
 /* Apply animation to hobbies.html */
 animation: slide-from-right 0.3s;
}

```

Note:
By default, the browser's [view transition animation](/en-US/docs/Web/API/View_Transition_API/Using#the_view_transition_process) creates a cross-fade between the old and new pages by animating their `opacity` and applying [`mix-blend-mode: plus-lighter`](/en-US/docs/Web/CSS/Reference/Selectors/::view-transition-old).
In this example, we've cleared the browser's default animation (`animation: none`) and replaced it with a `slide-from-right` animation. If you keep the default cross-fade, you can experiment with different [`mix-blend-mode`](/en-US/docs/Web/CSS/Reference/Properties/mix-blend-mode) values to see various transition effects.

Here's how the slide-in effect looks:



Now let's complete this effect by adding another slide animation to ensure that the `hobbies.html` page slides out the same way it came in. In this case, we will be navigating from `hobbies.html` to `index.html`; so, `hobbies.html` is now our old page and `index.html` is the new page.

We'll call this animation `slide-to-right`. Add this code to `index.css`:

css
```
@keyframes slide-to-right {
 from {
 /* Start from screen view */
 transform: translateX(0);
 }

 to {
 /* Move to right */
 transform: translateX(100vw);
 }
}

::view-transition-old(root) {
 /* Apply @keyframes to hobbies.html */
 animation: slide-to-right 0.3s;
 z-index: 2;
}

::view-transition-new(root) {
 animation: none;
}

```

Notice how I've animated only the `hobbies.html` in both directions and not added any animation to `index.html`. You can animate both if you like.

Here's how the transition will finally look, including both the slide-in and slide-out effects for `hobbies.html`:



[Check out the demo page](https://mdn.github.io/dom-examples/view-transitions/mpa-homepage/) to see the result in your browser. And if you'd like to extend this demo, feel free to use the code in [mdn/dom-examples](https://github.com/mdn/dom-examples/tree/main/view-transitions/mpa-homepage) repository and add an animation for the `index.html` page (look for `animation: none` in the `index.css` file).

I'd be happy to see all the cool view transitions you come up with.




## Wrapping up

 You can watch the video of [my talk](https://mastodon.social/@mdn/114811367802817335) on this topic on MDN's Mastodon.

Happy view transitioning! May your multi-page apps breeze from page to page.










## Previous post
 Launching MDN's new front end
