## Fake "Recent" block in `Contents.md`
This will render a block-like list with all the recently modified pages.

It's a little hacky because Logseq queries cannot return pages, so we must
make them look like blocks manually.

```clojure
@@html: <div style="display:none">@@
#+BEGIN_QUERY
{:title "Recent"
 :query [:find (pull ?p [*])
         :in $ ?start
         :where
         [?p :page/last-modified-at ?d]
         [?p :page/name ?n]
         [(>= ?d ?start)]
         [(not= ?n "contents")]]
 :inputs [:4d-before]
 :view (fn [result]
  (for [p (take 4 (sort-by :page/last-modified-at > result))]
    [:div.ls-block.flex.flex-col.pt-1
      [:div.flex-1.flex-row
        [:div.mr-2.flex.flex-row.items-center
          {:style {:height "24px" :margin-top "0" :float "left"}}
          [:a.block-control.opacity-50.hover:opacity-100
            {:style {:min-width "0" :height "16px" :margin-right "2px"}}
            [:span]]
          [:a
            {:href (str "/page/" (:page/name p))}
            [:span.bullet-container.cursor
              [:span.bullet]]]]
        [:div.flex.relative
          [:div.flex-1.flex-col.relative.block-content
            [:div
              [:span.page-reference
                [:a.page-ref
                  {:href (str "/page/" (:page/name p))}
                  (-> p :page/properties :title)]]]]]]]))}
#+END_QUERY
<style>
.custom-query { margin-top: 0; }
.custom-query .opacity-70 { opacity: 1; }
</style>
```

## The `<script-block>` element

This is a WebComponent that executes its content as JavaScript.

First, import it:

```html
<style onload="Function(this.innerHTML.slice(2, this.innerHTML.length - 2))()">/*
import("https://cdn.jsdelivr.net/gh/71/logseq-snippets@main/script-block.js")
*/</style>
```

Usage is:
```html
<script-block state='{}'><!-- return document.createElement("div"); --></script-block>
```

Within the JavaScript function, the function has access to:
- `this`, the current state; can be mutated, and takes the value of the `state` attribute (after parsing from JSON).
- `save`, a function that takes an object and:
  1. Serializes the object to JSON.
  2. Edits the source of the block so that `<script-block state='...'>` becomes `<script-block state='${json}'>`.
  3. Saves the changes to the block.
- `html` and `svg` from [htl](https://observablehq.com/@observablehq/htl).

This element therefore essentially allows you to define custom components based on JavaScript whose state
can be mutated and persisted. Since their state is saved to the underlying file, it will also be saved in
the Git repository.

For an example, see the [counter](#counter).

### Counter

Increments by one everytime it is clicked.

```html
@@html: <script-block state='{"count":0}'><!-- return html`<button onclick=${() => save({ count: this.count + 1 })}>${this.count ?? 0}`; --></script-block>@@
```

## The `<define-script-block>` element

This is a WebComponent used to define reusable `<script-block>` elements. For instance, let's implement
the [counter](#counter) again!

### Reusable counter

Put this anywhere:

```html
<define-script-block name="x-counter"><!--
  return html`
    <button onclick=${() => save({ count: +this.count + 1 })}>
      ${this.count}
  `;
--></define-script-block>
```

And use it like this:

```html
@@html: <x-counter count="0"></x-counter>@@
```

Alternatively to `<define-script-block>`:

```html
<style onload="Function(this.innerHTML.slice(2, this.innerHTML.length - 2))()">/*
import("https://cdn.jsdelivr.net/gh/71/logseq-snippets@main/script-block.js")
  .then(({ defineElement }) => defineElement("x-counter", ({ save, html, count }) => html`<a onclick=${() => save({ count: +count + 1 })}>${count}`))
*/</style>
```

## RSS page

RSS page which fetches content when a "Refresh" button is clicked. The contents are fetched at a given interval.

```markdown
---
title: RSS
---

## @@html: <button onclick="Function(document.getElementById('refresh-rss-feed').innerHTML)()()">Refresh</button>@@
<script id="refresh-rss-feed">
return async function(forceRefresh) {
  document.querySelector("a[href^='/file/pages']").click();

  const title = document.querySelector("h1.title").innerText;

  document.getElementById(title).click();

  const textarea = document.getElementById(title),
        now = new Date()
  let md = textarea.value;

  const feeds = [...md.matchAll(/^### (\[(.+?)\]\((.+?)\)\s*\nSCHEDULED: <([\d-]+) \w+ ([\d:]+)) \.\+(\d+)(\w)>\n(?:<!-- REGEXP: \/(.+?)\/ -->\n)?/gm)].flatMap((match) => {
    const date = new Date(match[4] + " " + match[5]),
          title = match[2],
          url = match[3],
          interval = match[6],
          unit = match[7],
          intervalMultiplier = { h: 3600, d: 3600*24, w: 3600*24*7, m: 3600*24*30, y: 3600*24*365 }[unit],
          [re, selector] = (match[8] ?? "(.+)/$1").split("/");

    return { title, url, date, interval: interval * intervalMultiplier * 1000, toReplace: match[1], re: new RegExp(re), selector };
  });

  const items = [...md.matchAll(/^### <([\d-: ]+)> (.+?): \[(.+)\]\((.+)\)\n/gm)].map((match) => match[0]);

  for (const feed of feeds) {
    if (!forceRefresh && feed.date.valueOf() > now.valueOf()) {
      continue;
    }

    const f = window.fetchNoCors ?? window.fetch;
    const data = await f(feed.url)
        .then((x) => x.text())
        .then((x) => new DOMParser().parseFromString(x, "application/xml"));
    const feedItems = [];

    if (data.firstElementChild.tagName === "feed") {
      for (const item of data.querySelectorAll("entry")) {
        const title = item.querySelector("title").textContent,
              url = item.querySelector("link").getAttribute("href"),
              date = new Date(item.querySelector("updated").textContent);

        feedItems.push({ title, url, date });
      }
    } else {
      for (const item of data.querySelectorAll("item")) {
        const title = item.querySelector("title").textContent,
              url = item.querySelector("link").textContent,
              date = new Date(item.querySelector("pubDate, date").textContent);
        feedItems.push({ title, url, date });
      }
    }

    for (const { title, url, date } of feedItems) {
      const selectedTitle = title.replace(feed.re, feed.selector),
            markdown = `### <${date.toISOString().replace("T", " ").replace(/:\d{2}\..+$/, "")}> [[${feed.title}]]: [${selectedTitle}](${url})\n`;

      if (items.indexOf(markdown) === -1) {
        items.push(markdown);
      }
    }

    let nextDate = feed.date.valueOf();
    while (nextDate < now) {
      nextDate += feed.interval;
    }
    const next = new Date(nextDate);
    const nextString = `<${next.toISOString().substr(0, 10)} ${next.toDateString().substr(0, 3)} ${next.getHours()}:${next.getMinutes()}`

    md = md.replace(feed.toReplace, feed.toReplace.substr(0, feed.toReplace.indexOf("<")) + nextString);
  }

  items.sort().reverse();

  textarea.value = md.slice(0, md.indexOf("##" + " Items")) + "##" + " Items\n" + items.slice(0, 50).join("");
  await new Promise(r => setTimeout(r, 100));
  textarea.dispatchEvent(new KeyboardEvent("keydown", { keyCode: 27 }));

  document.querySelector("a[href^='/page']").click();
};
</script>
## Feeds
### [XKCD](https://xkcd.com/atom.xml) 
SCHEDULED: <2020-11-21 Sat 11:0 .+2d>
### [The Rust Blog](https://blog.rust-lang.org/feed.xml) 
SCHEDULED: <2020-11-20 Fri 17:0 .+1d>
<!-- REGEXP: /^Announcing // -->
## Items
### <2020-11-19 00:00> [[The Rust Blog]]: [Rust 1.48.0](https://blog.rust-lang.org/2020/11/19/Rust-1.48.html)
### <2020-11-18 00:00> [[XKCD]]: [Blair Witch](https://xkcd.com/2387/)
```

To bypass CORS, I use the following [ViolentMonkey](https://github.com/violentmonkey/violentmonkey) script:
```js
// ==UserScript==
// @match       https://logseq.com/*
// @grant       GM_xmlhttpRequest
// @inject-into page
// ==/UserScript==

unsafeWindow.fetchNoCors = (url) => new Promise((resolve, reject) => GM_xmlhttpRequest({
  url,
  method: 'GET',

  onabort: () => reject(),
  onerror: () => reject(),

  onloadend: (res) => resolve({ async text() { return res.responseText; } }),
}));
```

## Execution scripts

I use the following script to execute JS scripts automatically.

```js
const watchedElements = [];
const observer = new MutationObserver((mutationsList) => {
  for (const mutation of mutationsList) {
    if (mutation.target.classList.contains("extensions__code") &&
        mutation.target.firstChild.textContent === "js,run") {
      const code = mutation.target.children[1].value,
            dispose = Function(code)();

      if (typeof dispose === "function") {
        watchedElements.push([mutation.target, dispose]);
      }
    } else if (mutation.removedNodes.length === 1) {
      const removedNode = mutation.removedNodes[0];

      for (const [watchedElement, dispose] of watchedElements) {
        if (removedNode.contains(watchedElement)) {
          dispose();
        }
      }
    }
  }
});

observer.observe(
  document.getElementById("main-content-container"),
  { subtree: true, childList: true },
);
```

And then:

````markdown
```js,run
console.log("Script loaded.");

return () => console.log("Script unloaded.");
```
````
