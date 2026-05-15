---
source_url: https://developer.mozilla.org/en-US/blog/mdn-front-end-deep-dive/
source_hash: 270a4638
retrieved: 2026-05-16
title: Under the hood of MDN's new frontend
---

# Under the hood of MDN's new frontend

## In this article




- [MDN's architecture](#mdns_architecture)
- [Why we rebuilt MDN's frontend](#why_we_rebuilt_mdns_frontend)
- [Enter web components](#enter_web_components)
- [Server components and web components (on the server)](#server_components_and_web_components_on_the_server)
- [Shipping only what's required](#shipping_only_whats_required)
- [Baseline](#baseline)
- [The development environment](#the_development_environment)
- [Get involved](#get_involved)








 ![MDN search modal showing results for HTML, CSS, JavaScript performance, Web APIs, and SVG, overlaid on the site's HTML source code, with the MDN logo in the corner.](./featured.png)




# Under the hood of MDN's new frontend



 [Leo McArdle](https://github.com/LeoMcA)

 April 8, 2026


 23 minutes read








 Last year, we [launched a new frontend for MDN](/en-US/blog/launching-new-front-end/).
The most noticeable changes were adjustments to our styles; we simplified and unified the MDN design across all of our pages.
In truth, the biggest changes were not reader-visible, but rather in the overhauled code that powers our frontend.
This post describes what we've done, the technologies we chose, and why we did it at all.




## MDN's architecture

 To fully understand the changes we made to MDN's frontend, I should provide some context on how MDN content is assembled into the website you all know and love.
MDN's architecture is probably worthy of its own blog post, but to simplify for the sake of this post, pages are published to the site via these major steps:

- The documentation is written and maintained in Markdown, across a couple of git repositories, by our fantastic team of technical writers, partners, and invited experts, alongside our enormous community of contributors and translators.

- A build tool ingests these Markdown files, converts them into HTML, and saves them as a set of JSON files with supplemental metadata about each page.

- Our frontend traverses these JSON files and compiles fully-featured pages, complete with browser compatibility tables, l10n support, navigation menus, and so on - in a step we name (or perhaps misname) [server-side rendering](/en-US/docs/Glossary/SSR) (SSR).

- At this point, the resulting HTML, CSS, and JavaScript files are uploaded to cloud buckets and delivered to our readers globally.

![](/en-US/blog/mdn-front-end-deep-dive/architecture.svg)




## Why we rebuilt MDN's frontend

 Rebuilding our frontend had been a long time coming because of how tricky it was to work on MDN's UI.
Our previous frontend (called yari) was a React app that, unfortunately, had accumulated quite a lot of technical debt.
Maintenance wasn't exactly impossible, but was certainly painful to undertake.
Whenever we fixed issues or added new site functionality, we inevitably ended up piling on more technical debt.
But how did we get there?

The React app had started life as a "Create React App", but a number of the built-in defaults didn't work for us.
Of course, this led to a series of workarounds, and we eventually had to ["eject" the configuration](https://create-react-app.dev/docs/available-scripts/#npm-run-eject). We ended up with an extremely complicated Webpack config as well as some very hacky build scripts.

On the CSS side as well, things were starting to get out of control.
We used [Sass](https://sass-lang.com/) extensively, then added modern CSS features like CSS variables, which meant we had a bizarre mix of both idioms spread across our files.

The CSS was also incredibly entangled, with poor or nonexistent scoping.
When we made a change in one UI component, we'd frequently spot unintended changes in others.
These issues, and a lack of build tools to split up the CSS, meant we had to ship a large render-blocking CSS blob to our users, complete with styles for components they might never load.

But by far, the biggest issue was that our React app was merely a wrapper around our static content.
To make the React app aware of the HTML content that our build tool generated would have required expensive reparsing of the HTML and an extraordinary amount of logic which we'd have to ship to users in our client-side JavaScript.
We didn't want to do this, so the React app boundary essentially ended where our documentation began – we used React's `dangerouslySetInnerHTML` to insert the content.

Our content is mostly static (prose and code examples), but there were a number of places within this static content where we needed to add interactivity (think things like the "Copy" button on code blocks).
For these interactive parts, we ended up using regular DOM APIs, which wasn't very elegant, particularly when the rest of the site was written in React.
We couldn't use [JSX](https://react.dev/learn/writing-markup-with-jsx) (React's HTML-like syntax), which limited the maintainability of more complex pieces of interactivity, and we occasionally faced the worst-case scenario of maintaining duplicate implementations - one using React and another using DOM APIs.




## Enter web components

 As a possible solution to this problem, in 2024, we started experimenting with [Lit](https://lit.dev/) and [web components](/en-US/docs/Web/API/Web_components) to see whether they could improve the developer experience when working on this kind of interactivity within content.
Our first proper prototype, and eventual production implementation, came out of our work on the [MDN Curriculum](/en-US/curriculum/) when we partnered with [Scrimba](https://scrimba.com).




### Scrims as a proof-of-concept

 Scrimba has a feature called "Scrims" - an interactive learning environment we embed on MDN via an `<iframe>`. Scrims let learners watch a short coding tutorial and then edit the code themselves, all within the same view — think of them as interactive screencasts.

On our pages, we didn't want to send any user data to Scrimba until a user chose to interact with their content; so we didn't load the `<iframe>` until after a user clicks to open it.
We also wanted to be able to expand the Scrim to fullscreen, without a user leaving MDN, so we used the [`<dialog>`](/en-US/docs/Web/HTML/Reference/Elements/dialog) element.

We figured that building a web component would allow us to use a custom element to insert these Scrims directly into our content, thereby skipping a number of rendering steps and avoiding the tricky-to-maintain DOM API implementation.

Our component starts life by extending `LitElement`:

js
```
export class MDNScrimInline extends LitElement {}

```

Within that, we need to define some state.
In Lit, we can do this through a static properties attribute:

js
```
static properties = {
 // the url to the scrim, set as an attribute on our custom element
 url: { type: String },
 // internal state tracking if the element is fullscreen
 _fullscreen: { state: true },
 // internal state tracking if we’ve rendered the yet
 _scrimLoaded: { state: true },
};

```

And we set defaults in our class constructor:

js
```
constructor() {
 super();
 this.url = undefined;
 this._fullscreen = false;
 this._scrimLoaded = false;
}

```

We want to manipulate the URL which has been provided as an attribute to our custom element.
Lit provides us with [lifecycle methods](https://lit.dev/docs/components/lifecycle/) to do this. We want to compute a value once we know an update to the component will be rendered:

js
```
willUpdate(changedProperties) {
 if (changedProperties.has("url")) {
 if (this.url) {
 const url = new URL(this.url);
 url.searchParams.set("via", "mdn");
 url.searchParams.set("embed", "");
 this._fullUrl = url.toString();
 } else {
 this._fullUrl = undefined;
 }
 }
}

```

We can then use this state to render our component:

js
```
return html`



 Clicking will load content from scrimba.com

 Toggle fullscreen

 Open on Scrimba


 ${this._scrimLoaded
 ? html`

 `
 : html`
 Load scrim and open dialog.
 `}



`;

```

Lit's `html` template literal is just as convenient as JSX in allowing us to write HTML-ish syntax in JavaScript.
The huge advantage over JSX is it doesn't require any compilation to use: it's native JavaScript.

I say "HTML-ish" because you'll notice a few annotations before certain attributes in the template above, namely `@close` and `@click`.
This is Lit syntax that allows us to bind event listeners to these elements: the close event on the `<dialog>` and click events on a couple of buttons.
We define these in the class too:

js
```
#toggle(e) {
 if (this._fullscreen) {
 this.#close();
 } else {
 this.#open();
 }
}

#open() {
 const dialog = this.renderRoot.querySelector("dialog");
 if (dialog) {
 dialog.showModal();
 this._scrimLoaded = true;
 this._fullscreen = true;
 }
}

#close() {
 const dialog = this.renderRoot.querySelector("dialog");
 dialog?.close();
}

#dialogClosed() {
 this._fullscreen = false;
}

```

When a user clicks to open the Scrim, the `#open` method fires, which updates the values of `_scrimLoaded` and `_fullscreen`.
Lit notices the changes to these properties, because we'd defined them in `static properties`, and automatically re-renders the component, loading the `<iframe>` and the Scrim inside.

I've simplified the component a little for brevity, you can see the [`MDNScrimInline` source on GitHub](https://github.com/mdn/fred/blob/main/components/scrim-inline/element.js).
There's a number of additions in there like telemetry, and a dynamically-rendered thumbnail (it was fewer bytes and a simpler implementation than pre-rendering a bunch of images).
As you can imagine, this was very straightforward to develop, thanks to the convenience functions we get with Lit; this would've been a massive headache to implement directly with traditional DOM APIs.

In many ways, I found that the implementation in Lit was simpler than in React: you may notice the state we're dealing with isn't particularly complex, and doesn't require a complex component architecture to reflect it.
But more importantly, it gives us that custom element we can insert into our curriculum content wherever we need to add a Scrim:

html
```

```




### Interactive examples

 The Scrimba implementation was a good introduction for the team to writing a small component, but what about something more complex?
Interactive examples are the components that appear under "Try it" sections at the top of many CSS, JavaScript, and HTML pages.

Improving the infrastructure for these had been on the engineering team backlog for a while; they were difficult for our technical writers and community to maintain and author.
The existing implementation was split across four git repositories, and authoring or debugging an example could require synchronizing changes across them all.
Worse still, examples had to be written in isolation from the content they'd be included in, so it wasn't possible to show a live preview containing both changes to an example and the content of the MDN page it would be included on.

This complexity came about for good reasons: these interactive examples were too complex to easily engineer and maintain with DOM APIs directly.
Instead, we had a separate build system and examples repository which rendered these examples into separate HTML pages which we could load in an `<iframe>` directly.

We wanted to simplify this architecture, to make writing interactive examples easier for our authors, so again: we reached for Lit to build a web component we could include directly in our content.
This was a much more technically-complex implementation than Scrims.
Firstly, we needed a number of templates for the different ways interactive examples are displayed:

- A code editor and console for JavaScript examples, with [the addition of tabs for WASM](/en-US/docs/WebAssembly/Reference/Memory/load) examples.

- A tabbed code editor with rendered output for HTML examples (usually [tabs for HTML and CSS](/en-US/docs/Web/HTML/Reference/Elements/p#try_it)).

- A table of code editors that can be selected with rendered output for CSS examples (see the [CSS `background-clip` property](/en-US/docs/Web/CSS/Reference/Properties/background-clip#try_it), for example).

Secondly, we needed a way to render the examples, and the edits users made to them.
We had already written that logic to create our interactive [Playground](/en-US/blog/introducing-the-mdn-playground/), but it was in React: so we needed to port that too.

So we set about doing all that.
What made things a lot simpler was that [Lit's React integration](https://lit.dev/docs/frameworks/react/) allowed us to render these web components in our existing React app.
So we could port the elements of the Playground we needed piece-by-piece to web components, without having to port the entire thing all at once, and without having to maintain dual implementations.

At a high level, we split our single entangled Playground React component into a series of custom elements:

- `<play-editor>`: A [CodeMirror](https://codemirror.net/)-powered editor.

- `<play-console>`: An element to format and render console messages.

- `<play-runner>`: An element responsible for rendering the current state of each editor.

- `<play-controller>`: An element responsible for passing events and state between each of the above elements.

This made the logic in the Playground simpler and decoupled, which allowed us to reuse these elements in the `<interactive-example>` element we created.
This included logic to search the page for `<code>` elements the interactive example component should ingest, sending their contents to a `<play-controller>`, and working out which of the above templates it needed to render: using some combination of the `<play-*>` elements to do this.

This meant that authors could now add a macro to content - which renders our `<interactive-example>` custom element behind the scenes - followed by the code blocks the example should use:

md
```
{{InteractiveExample("CSS Demo: background-repeat")}}

```css interactive-example-choice
background-repeat: space;
```

```css interactive-example-choice
background-repeat: repeat;
```

```html interactive-example



```

```css interactive-example
#example-element {
 background-color: #cccccc;
}
```

```

You can see the full source for this example on the [CSS background-repeat page GitHub](https://github.com/mdn/content/blob/616b1da6696a833451891ad8c767ff15474b08f7/files/en-us/web/css/background-repeat/index.md?plain=1#L11-L50).
We're not quite sure where we stand on putting custom elements directly in our non-curriculum Markdown content, which could make our architecture even simpler; that's a discussion for another day.




## Server components and web components (on the server)

 So this is all well and good, web components seem pretty cool and solve some of our problems around adding interactivity within our static content. But wasn't this blog post supposed to be about how we rewrote our entire frontend stack: what happened to that? To answer that, let's get into another problem we had with our old frontend:




### React server components

 I've mentioned our problem with our React app being a "wrapper" and not being able to interact with our content.
The fundamental problem is that (at least classically) React apps are Single Page Applications (SPAs), which you figure out how to render on the server, then attempt to figure out how to avoid shipping an absolutely enormous JavaScript bundle to your users.

That last part is necessary. That's because everything you render in an SPA, even if it could be rendered statically on a server or in a compilation step, has to be shipped in your client-side JavaScript bundle and re-rendered in the client – just to verify nothing has changed.
The React documentation itself summarises it better than I could:

This pattern means users need to download and parse an additional 75K (gzipped) of libraries, and wait for a second request to fetch the data after the page loads, just to render static content that will not change for the lifetime of the page.

[Server Components](https://react.dev/reference/rsc/server-components) on react.dev

That's a quote from the documentation for React Server Components (RSC): so the project recognizes that this is a problem, and is working hard on solving it.
Unfortunately, using RSC effectively requires using a framework that we aren't already using.
Migrating to that would require rewriting a lot of the frontend anyway.

So since a large rewrite was required to address this fundamental issue, we could also re-evaluate what kind of a site MDN is, and how complex it needed to be.
Really, MDN isn't a particularly complex site, at least from a "things that require interactivity" standpoint.
The vast majority of content on an MDN documentation page is HTML and CSS: we don't need a complex app powering the majority of the site.
We essentially have islands of interactivity, which could easily all be implemented as web components.

And if we're implementing all our functionality in isolated web components, it doesn't really matter how they're assembled: we just need to template HTML together, and that can happen multiple times, in multiple places in our overall build system.
There's no higher-level "app" that needs to understand the state of the entire page, so there never could be a "wrapper" problem – our markdown to HTML build tool is just as first class a citizen as whatever templating we need to do in our frontend.

This approach solved all three problems at once: there's no SPA shipping redundant JavaScript to re-render static content, there's no "wrapper" that can't reach into our documentation HTML, and each piece of interactivity is a self-contained web component that only loads when it's needed. What remained was deciding how to do the static templating that assembles everything else.




### Our server components

 We considered using a dedicated templating language, such as EJS, to do templating in our frontend, but realized there's a lot of benefits to a component-based architecture.
While doing static HTML templating on the server avoids shipping logic to users in a client-side JavaScript bundle, this HTML still requires styling.
And as you may recall, the CSS in our old frontend was a mess, and we wanted to avoid shipping unnecessary CSS if it wasn't necessary to render the current page.

We first built our own concept of Server Components, using Lit's HTML template literal, which we were already comfortable with.
Here's an example of our top navigation bar component:

js
```
export class Navigation extends ServerComponent {
 render(context) {
 return html`

 ${Logo.render(context)}


 ${Menu.render(context)}






 `;
 }
}

```

You'll see the logic is handled in a `render` method, much like it would be in a Lit component.
We don't need any lifecycle methods because this only ever runs once.
This component can render other server components like `Logo` and `Menu`, as well as web components like `<mdn-search-button>` and `<mdn-search-modal>`.

We render this to HTML in NodeJS, using a convenient function Lit provides.
This also renders these Lit web components to [Declarative Shadow DOM](/en-US/docs/Web/API/Web_components/Using_shadow_DOM) so, in compatible browsers, the Shadow DOM and CSS of our custom elements is rendered before the JavaScript gets loaded.




## Shipping only what's required

 As I mentioned before, one big problem with an SPA-approach to building websites is everything necessary to render the page needs to be included in the client-side JavaScript bundle.
Another similar problem is that it's very easy, and therefore very common, to ship a whole load of code not necessary for rendering the current page, and only necessary for rendering other pages, in one huge client-side bundle.
We fell into this trap with our old frontend, both with our JavaScript and our CSS.

Over time we did partition off certain routes into separate chunks, but this was only possible with our JavaScript. Our CSS was too entangled to do this, and our build tool wasn't configured to do it either.

And, it was only possible to do this per route, not for components within the same page. If there was a chance a certain route might load a component, it needed to be included in the bundle if it was to be server-side rendered, even if it wasn't, and that JavaScript was never executed client side.

We wanted to avoid all this in our new frontend: only loading the most minimal CSS and JavaScript bundles required to render the page, and making it interactive; and I wanted to achieve this on an architectural level, so it was nearly impossible to not do. We achieved this in a few ways, but the key to unlocking them all was a flat name-based component structure.

Every component lives in a flat hierarchy under the `./components/` directory, with the following file names reserved for certain pieces of the component:

```

```

- `element.js` - A web component, exporting a `MDNExampleComponent` class and defining a `<mdn-example-component>` element.

- `server.js` - A server component, extending `ServerComponent` from [`components/server/index.js`](https://github.com/mdn/fred/blob/main/components/server/index.js).

- `server.css` - CSS for a server component that will be automatically loaded for this component from the server.

- `global.css` - CSS for the component that gets loaded everywhere all the time.

We enforce parts of this via linting, and some of this by throwing errors if you don't adhere to the naming requirements.




### Web components

 Because we know where each web component lives based on name, we can do some clever things.
When our page loads, we run logic like this client-side:

js
```
for (const element of document.querySelectorAll("*")) {
 const tag = element.tagName.toLowerCase();
 if (tag.startsWith("mdn-")) {
 const component = tag.replace("mdn-", "");
 import(`../components/${component}/element.js`);
 }
}

```

This results in lazy-loading every custom element present in the DOM at load time, entirely async and in parallel.
The advantages here are numerous:

- Engineers don't have to remember to import web components in server components; they can use them as if they are normal HTML elements.

- We can add custom elements to our content markdown (either directly or through a macro), without having to wire up an export elsewhere.

- We only ever load the JavaScript for each web component if it's present on the page, automatically.
It's not necessary for engineers to think about whether their new component increases the bundle size - if it's not present on the page the user is viewing, that code won't end up being loaded by the users' browser.

- Changes to one component should only have minimal impact on the rest of the bundle: a bugfix in one component will require that component's JavaScript to be reloaded, but other components should be cached by the browser and will become interactive almost immediately, as they load in parallel asynchronously.

We also automatically load every web component in our SSR bundle, where Lit renders them into a Declarative Shadow DOM (unless the component opts out because it doesn't make sense to render them on the server).
This helps ensure we don't have layout shifts when the JavaScript loads.
The result is that the slight delay to interactivity when we load its JavaScript after initial render is imperceptible because the component is already on the page, just not interactive yet.

At least, not entirely interactive: this architecture is flexible enough for us to be very clever with certain components.
One of these is `<mdn-dropdown>`.

This is a component which, as the name might suggest, implements a dropdown.
The render method is rather simple:

js
```
render() {
 return html`


 `;
}

```

We render two slots, which can be used like so:

html
```

 Click me
 Hello world!

```

Since we use slots here, `<mdn-dropdown>`'s shadow DOM is almost irrelevant, and any children of it can be styled entirely as normal: the element only adds interactivity.
It does have some styles attached, but those aren't for styling: they're for interactivity too.
See, we also have a lifecycle method:

js
```
firstUpdated() {
 this.loaded = true;
}

```

And define our loaded property like so:

js
```
static properties = {
 loaded: { type: Boolean, reflect: true },
};

```

If you trace the logic through, you'll see that the dropdown slot isn't hidden by default, and therefore, is visible when we render to DSD on the server.
And once the component is `loaded=true`, we reflect that attribute into the DOM, so it appears like:

html
```

```

The reason we want this is because we also attach the following CSS to the element:

css
```
:host(:not([loaded], :focus-within)) {
 slot[name="dropdown"] {
 display: none;
 }
}

```

The logic here is a little hard to parse - and I say that as the person who wrote it - but it effectively translates to:

- If JavaScript for the `mdn-dropdown` has loaded, do no styling.

- If the JavaScript for the `mdn-dropdown` hasn't loaded, then:

- If the focus is outside the element, hide the dropdown slot.

- If the focus is within the element, show the dropdown slot.
And what does this mean? Well, we have a CSS native dropdown as soon as the page is rendered, which progressively enhances to a JavaScript dropdown, once that code has loaded; other engineers need not know any of this is going on, just that the dropdown component works.

This is hugely important in our top navigation menu, where most of our links are behind dropdowns: those are completely usable as soon as they're rendered to the page.
In other components, such as in our theme switcher, while choosing between themes isn't possible until its JavaScript loads, the dropdown being interactive already gives us a few seconds longer load time for that JavaScript before the user clicks on anything requiring it.

#### What if we don't have DSD

Now, Declarative Shadow DOM isn't widely available yet, so we have to ensure things also work on slightly older browsers.
This is where the `global.css` file comes in: any CSS written in one of these is included on all pages, all the time.

This is obviously necessary for setting things like global CSS variables, global reset styles, and the like.
But for components, when DSD isn't available, before its JavaScript has loaded, they'll appear to the browser as an empty inline element with no styling attached.
This isn't always optimal, and can cause layout shifts when the JavaScript gets loaded, so we set global styles for certain elements, for example, for our button component:

css
```
mdn-button {
 display: inline-flex;
 vertical-align: middle;
}

```

This is just enough to ensure buttons don't shift the layout before being loaded.
There's a small optimisation to be had by only loading this style when an `mdn-button` is present on the page, rather than on every page: but it's so minimal it's probably not worth the added complexity.
It's also important for our aforementioned dropdown component: we still want this to be interactive if DSD hasn't loaded, so we include a similar style in a `global.css` file.




### Server components

 For server components, the considerations are a little different. The JavaScript itself doesn't need to be lazy loaded or cut down, since it's only being used to SSR our HTML. But what does require careful loading is the CSS used in each server component. We only want to load this if the component gets rendered to the page. But how do we know this?

Our server components extend our `ServerComponent` class, so we place some tracking logic in its static render method, which runs before and after instantiating each server component:

js
```
export class ServerComponent {
 static render(...args) {
 const { componentsUsed } = asyncLocalStorage.getStore();
 const componentUsedBefore = componentsUsed.has(this.name);
 componentsUsed.add(this.name);
 const renderResult = new this().render(...args);
 if ((!renderResult || renderResult === nothing) && !componentUsedBefore) {
 componentsUsed.delete(this.name);
 return nothing;
 }
 return renderResult;
 }
}

```

This gives us a `Set`, `componentsUsed`, which only contains the components that rendered anything. We then use this in our `OuterLayout` component:

js
```
const { componentsUsed, compilationStats } = asyncLocalStorage.getStore();
const styles = componentsUsed
 .flatMap((component) =>
 compilationStats.assets.filter(
 (name) => name === `${component.toLowerCase()}.css`,
 ),
 )
 .map((path) => html`
- `);

```

The `compilationStats` object here comes from our build tool, Rspack (more on that later), and this code block gives us a list of `<link>` tags to include in our `<head>` containing only the CSS that's necessary to render the page. We also load a CSS file with all the `global.css` files mentioned before, bundled into one.

Again, this is a simplified version of our [`ServerComponent`](https://github.com/mdn/fred/blob/main/components/server/index.js) and [`OuterLayout`](https://github.com/mdn/fred/blob/main/components/outer-layout/server.js) classes, which you can see in their entirety in our repository if you're interested.




### Performance

 You'll note that what I've suggested here results in a number of quite small CSS and JavaScript files being loaded on the page. This goes against the classical wisdom that there's an optimal bundle size where you combine multiple assets into one to reduce the number of round trips a browser needs to make to load them all.

I can't claim to be a performance expert here, but [HTTP/2](/en-US/docs/Glossary/HTTP_2) and [HTTP/3](/en-US/docs/Glossary/HTTP_3) do a lot to change that wisdom. As we can now download assets in parallel, and reuse connections, multiple small assets don't have the overhead they did before, and can be advantageous, particularly given how our web components load.

As I described before, since we load our web components asynchronously and independently after the page has rendered, it's faster to fire these down the wire component by component - so the browser can act on adding interactivity as soon as the code has loaded - rather than all in one blob where the browser has to parse the code for multiple components, perhaps to only add interactivity for one.

Once you throw caching into the mix, things get even faster, and this extends to our CSS too: an update to one component in many cases won't touch the bundled code of the others. So for a user re-visiting MDN, they'll get interactivity for components that have been cached near-instantly, and for any that have changed - or for any server component CSS that has changed - they'll only be waiting for that changed component to download.

Benchmarking these things is always required, and we could always do more, but what we've done so far showed that bundling things together was only as good or slower with a cold cache. We do also have a few levers in our build config to pull to easily bundle smaller components together if future benchmarking shows that that would be better.




## Baseline

 We're using quite a number of more modern web technologies here, and we needed an easy way to determine if we could use something, and if we could do without polyfills or progressive enhancement. Luckily enough, we've spent the last few years working on the Baseline project with a cross-vendor range of partners in the [WebDX group](https://www.w3.org/community/webdx/).

This gave us a fantastically easy way to determine whether to use an API or not. The advice I gave the other engineers was: if it's "Baseline Widely Available", just use it; if it's "Baseline Newly Available", come talk to me first, and we'll figure out a polyfill or if it can be used as a progressive enhancement; and if it's "Baseline Limited Availability" or you need to do something there's no API for yet, think some more about if you really need to do it, then come talk to me.

We ended up using a range of technologies, across all these statuses:

Things like Custom Elements and Shadow DOM have been supported cross-browser for a surprisingly long time these days, and are solidly Baseline Widely Available.

Declarative Shadow DOM, as I mentioned earlier, is supported cross browser but hasn't been in the web platform long enough to be Widely Available, so we use it as a progressive enhancement, with fallbacks in place for older browsers which don't support it yet, like our use of `global.css` stylesheets.
There's also a few things we wanted which are at the very bleeding edge: one of those was extending `light-dark` to images, where we used PostCSS to define a custom mixin, allowing us to use syntax like this:

css
```
@mixin light-dark --baseline-img, url("./icons/status/limited.svg"),
 url("./icons/status/limited-dark.svg");

```

Using Baseline allows us to build confidently - knowing the vast majority of users can use a feature once it's reached Widely Available - and also keep our set of polyfills (and their overhead) small as we automatically remove them when features move from Newly Available to Widely Available.




## The development environment

 I've saved what I think our best improvement is until last: but I'm a little biased as an engineer working on MDN. Our old frontend development environment pained me every time I had to use it.

The big problems were:

- It was slow: the default start command took about two minutes to present you with a functional locally running SPA, not including the time to download npm packages and so on.

- It was complex: there were an enormous number of commands in our `package.json`, some of which gave you a development environment faster by skipping elements of the build, but that required an intricate knowledge of what exactly these complex commands were doing to know if you needed them or not.

- It didn't reliably restart: frequently changes, even simple ones like adding a new image, would require restarting the development server - and waiting another two minutes - to see the change.

- By default, we only rendered the SPA, with no SSR: that required running a separate command, which only created a production bundle and took even longer to run, which made debugging certain issues with SSR exceptionally difficult to do.

We were very keen to improve this - obviously for our own sake - but also to make the contribution process easier, and we have: the new frontend takes 2 seconds to start, and there's really only one command you need:

bash
```
npm run start

```

An enormous amount of this speed comes from using [Rspack](https://rspack.rs/) as our build tool.
Webpack was what the old frontend used, and to its credit is fantastically configurable - something we needed given the approaches we took with our architecture.
Rspack has a webpack-compatible API, but it's written in Rust and is incredibly fast.

Though [our Rspack config](https://github.com/mdn/fred/blob/main/rspack.config.js), currently standing at 650 LOC isn't necessarily simple, I would argue it's straightforward. There's very little logic hidden away, magically happening in our build tool. This config does a lot for us, including all the bundling, polyfilling, mixins, and optimizations I described before.

Our architecture is simple enough to rely on a single command that does almost everything. Unlike before, there aren't really separate things to build independently. There's no SPA that we can render without SSR, as server components are fundamental to our architecture. We don't need multiple commands because there's only one way our website is assembled.

This gives us an environment far more similar to our production environment than before, with the main difference being whether we reload the page on every change, and reload and re-render our server components on each request. This means we almost never have to restart our development environment, unless we're making changes to our Rspack config itself or applying various production-level optimizations. We have another command to build a production build with those optimizations and without the dynamic loading, which we can easily run if we need.

Developing in this new environment has been an absolute joy for me, and I'm so glad we managed to make these improvements.




## Get involved

 I think you'll agree that this blog post is long enough, and I hear the Slack pings coming in from our content team telling me to finish this post already, but I don't feel like I've even told you half the story of our new frontend architecture!

If you're interested in learning more, or have any questions about anything we've done, please feel free to come chat with us in the [#platform channel](https://discord.com/channels/1009925603572600863/1170042997212184576) on our Discord.

If you spot any issues, please raise them in [the fred GitHub repository](https://github.com/mdn/fred/issues). And if anything there looks like something you'd like to fix, have a go and submit a PR.

Building this new frontend was a complete pleasure: it's been a privilege to be able to use new web technologies to build the website that documents them.










## Previous post
 Image formats: Codecs and compression tools
