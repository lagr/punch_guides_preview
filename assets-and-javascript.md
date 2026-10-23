# Assets and JavaScript

Applies to Punch 0.1.

Stylesheets, images, fonts, and JavaScript modules are files in one directory of the app. They are embedded in the binary when it compiles, served under URLs that carry a fingerprint of their content, and cached by browsers for a year. There is no build step and no Node: CSS is plain CSS, JavaScript is ES modules loaded through an import map, and Turbo comes with Punch. This guide shows how to add each kind of asset, how pages refer to them, and why inline scripts and styles are refused.

After reading this guide, you will know:

- how to add a stylesheet, an image, and a JavaScript module;
- how pages refer to assets so that a missing file is a compile error;
- how fingerprints make assets cacheable forever and deploys safe;
- how to add behaviour with JavaScript modules, with or without Stimulus;
- why inline scripts and styles are refused, and what to do instead.

Before you start: you know [Pages and Components](pages-and-components.md). The examples come from the reference app's `app/assets/` and its layout.

## The asset directory

An app declares its asset directory once, in `app/setup.cr`:

```crystal illustrative="one line of reference/app/setup.cr"
# Stylesheets and images, embedded in the binary with fingerprinted URLs
# (lib/punch/docs/assets.md). Turbo comes with Punch.
Punch.assets "assets"
```

The path is relative to the file, so this is `app/assets/`. Every file below it is an asset, named by its path there. The reference app has three:

```text
app/assets/
  app.css
  images/
    dot.svg
    mark.svg
```

The extension decides the content type: CSS, JavaScript, JSON, source maps, text, SVG, icons, web manifests, the common image formats, and web fonts. An extension Punch does not serve does not compile, and HTML is not an asset, since pages are operations. Files starting with a dot are left out.

The files are read when the app compiles and become part of the binary, so a deployed app needs no asset directory and no CDN upload. Changing an asset means compiling again; `punch dev` does that when a file under `app/` changes.

## Referring to an asset

`Punch.asset("app.css")` names an asset, and the name is checked when the app compiles:

```crystal fails=spec/fail/asset_unknown.cr
Punch.assets "../assets/fixture"
Punch.asset("app.scss")
```

```text
Error: Punch.asset: no asset "app.scss"
```

The reference app's layout loads its stylesheet and shows its mark:

```crystal illustrative="two lines of Invoicing::Layout, reference/app/components/layout.cr"
v.document(@title, stylesheets: {Punch.asset("app.css")}, morph: @morph) do
  # ...
  chrome.img src: Punch.asset("images/mark.svg").href, alt: ""
```

- **`stylesheets:`** takes stylesheets only, and **`script:`** one JavaScript module. An image passed as a stylesheet does not compile.
- **`asset.href`** is a `Punch::Href` for `src:` and `href:` attributes. A plain string there does not compile.
- **`v.img`** needs `alt:`. `alt: ""` marks an image as decoration, as the mark in the header is.

## Fingerprints and caching

Each asset is served at its path with the first 16 hex digits of its content's SHA-256 before the extension. The app's `assets` command lists them:

```console
$ bin/invoicing assets
/assets/@hotwired/turbo-b9d35d123a07614f.js    203701 bytes  (gzip 44957)
/assets/app-0805348a529b9903.css                 2066 bytes  (gzip 996)
/assets/images/dot-67db238acc0ce42a.svg           109 bytes
/assets/images/mark-1d9917f380c61145.svg          231 bytes  (gzip 171)
/assets/punch/morph-66e18472a931426a.js           245 bytes  (gzip 195)

Import map: {"imports":{"@hotwired/turbo":"/assets/@hotwired/turbo-b9d35d123a07614f.js","punch/morph":"/assets/punch/morph-66e18472a931426a.js"}}
```

- **A changed file gets a new URL**, so every asset is sent with `Cache-Control: public, max-age=31536000, immutable`. A browser never asks for the same asset twice, and never shows a stale one.
- **Text formats are sent compressed** to clients that accept gzip.
- **Only the current fingerprint is served.** The plain name and old fingerprints answer 404.
- **Deploys reload pages whose assets changed.** Every stylesheet and script is written with `data-turbo-track="reload"`, so Turbo loads the whole page again when a deploy changed them, instead of mixing old pages with new assets.

## Stylesheets

A stylesheet refers to other assets by relative paths, and Punch rewrites them to the fingerprinted URLs. The reference app's stylesheet marks overdue invoices with a dot:

```css illustrative="one rule of reference/app/assets/app.css"
tr.overdue td:first-child { padding-left: 1.4rem; background: url(images/dot.svg) no-repeat 0.4rem center / 0.5rem; }
```

`url(images/dot.svg)` is served as `url(/assets/images/dot-67db238acc0ce42a.svg)`. `@import "base.css"` works the same way. The stylesheet's own fingerprint is taken after the rewrite, so it changes when an image it uses changes. A relative reference that names no asset stays as it is and is listed by `bin/invoicing assets` under "Unresolved".

Write modern CSS as it is: custom properties, nesting, `:is()`, and grid work in every current browser, and the reference app uses all four. There is no preprocessor. If you want Sass or Tailwind, run their command line yourself and put the output in `app/assets/`; Punch embeds whatever CSS it finds.

## JavaScript modules

JavaScript is ES modules, loaded by the browser as they are. Every `.js` asset is in the page's import map under its path without `.js`, so modules import each other by those names:

```js illustrative="an app module; the reference app ships no JavaScript of its own"
// app/assets/application.js
import { showRelativeTimes } from "controllers/relative_time"

document.addEventListener("turbo:load", showRelativeTimes)
```

```js illustrative="an app module; the reference app ships no JavaScript of its own"
// app/assets/controllers/relative_time.js
const format = new Intl.RelativeTimeFormat(document.documentElement.lang, { numeric: "auto" })

export function showRelativeTimes() {
  for (const time of document.querySelectorAll("time[data-relative]")) {
    const days = Math.round((Date.parse(time.dateTime) - Date.now()) / 86400000)
    time.title = time.textContent
    time.textContent = format.format(days, "day")
  }
}
```

The layout loads the app's module with `script:`:

```crystal illustrative="the layout's document call with a script"
v.document(@title, stylesheets: {Punch.asset("app.css")}, script: Punch.asset("application.js")) do
```

and a component marks the elements the module works on with a `data-` attribute, which views write from a named tuple. The page stays complete without JavaScript: the date is there, the module only rewrites it as "in 3 days":

```crystal illustrative="a due date the module rewrites"
v.time locale.date(due_at, :long), datetime: due_at, data: {relative: true}
```

- **Import by bare names**, as the import map lists them (`controllers/relative_time`), never by relative paths: a relative import resolves against the fingerprinted URL and names a file that is not served.
- **A library** vendored as one ES module file goes into the directory under the name it is imported by, such as `app/assets/@hotwired/stimulus.js` for Stimulus. Use the library's ES module build, and record where it came from.
- **Turbo is always there** as `@hotwired/turbo`. An app cannot replace it.

For more than a few behaviours, Stimulus fits Punch well: controllers are found by `data-controller` attributes, which components write with `data: {controller: "relative-time"}`, and they survive Turbo's page changes without wiring. Punch does not ship it; vendor its ES module build if you want it.

## Turbo

Every page loads Turbo, which Punch vendors unchanged from the npm package. Turbo Drive turns links and form submissions into fetches that replace the page's body, so pages feel immediate without client-side rendering. Forms follow its conventions without anything from you ([Forms](forms.md)), and Frames, Streams, and live pages build on it ([Turbo Frames, Streams, and Live Pages](turbo-frames-streams-and-live-pages.md)).

`v.document(…, turbo: false)` leaves Turbo out of a page, for a page that must work with full page loads only.

## The Content-Security-Policy

Every response carries a Content-Security-Policy that allows scripts and styles from the app's own origin only:

```text
default-src 'self'; script-src 'self' 'sha256-…'; style-src 'self' 'sha256-…'; base-uri 'none'; object-src 'none'; frame-ancestors 'none'; form-action 'self'
```

Two inline pieces are allowed by their hash: the import map, and the stylesheet of Turbo's progress bar. Nothing else inline runs. That makes a whole class of cross-site scripting harmless: even if an attacker got markup into a page, the browser would not run a script from it.

Views make the same choice at compile time, so you never write something the browser would block:

```text
Error: Punch::View: no `onclick`. Inline event handlers are blocked by the Content-Security-Policy
Error: Punch::View: no `style`. Inline styles are blocked by the Content-Security-Policy
```

Instead of `onclick`, write a module that listens for events on elements marked with a `data-` attribute. Instead of `style`, give the element a class and style it in the stylesheet. For a value that varies per element, such as a width, put it in a `data-` attribute and let a module set it, or choose from a few classes.

## Testing assets

Assets are served through the router like everything else, so a spec can fetch one and check a page's references:

```crystal illustrative="a spec shape from the assets reference"
css = Punch::Spec.client.get(Punch.asset("app.css").url)
css.headers["Content-Type"].should eq("text/css; charset=utf-8")
page.should have_element("head link[rel=stylesheet][href='#{Punch.asset("app.css").url}']")
```

`Punch::Assets.table.unresolved.should be_empty` fails when a stylesheet refers to a file that is not there. Behaviour written in JavaScript is not covered by Punch's specs; keep it small, and test it in a browser.

## Common mistakes

**A path string for an asset.** `v.img src: "/assets/logo.svg"` does not compile, and the plain name would answer 404 anyway. Use `Punch.asset("images/logo.svg").href`.

**A relative import in JavaScript.** `import "./relative_time.js"` resolves against the fingerprinted URL of the importing module and fails in the browser. Import by the name in the import map: `import "controllers/relative_time"`.

**An inline script for a small behaviour.** A `<script>` in a component cannot be written, and one added through `raw` would be blocked by the policy. Put it in a module.

**Expecting a change to an asset to show without a build.** Assets are embedded when the app compiles. `punch dev` rebuilds on save; a running `serve` keeps the assets it was built with.

**Missing `alt`.** `v.img src: …` without `alt:` does not compile. Write what the image says, or `alt: ""` for decoration.

## Limits

- No build pipeline: no bundling, minification, TypeScript, or Sass. Run such tools outside Punch and put their output in the asset directory.
- No source maps unless you add them as assets yourself.
- Uploads and images that users add are not assets; there is no image processing.
- No Subresource Integrity attributes yet; assets come from the app's own origin.

## Rules

- Put every stylesheet, image, font, and script under the directory `Punch.assets` names. ([The asset directory](#the-asset-directory))
- Refer to assets with `Punch.asset("…")`, never with a path string. ([Referring to an asset](#referring-to-an-asset))
- Refer to other assets from CSS by relative paths. ([Stylesheets](#stylesheets))
- Import JavaScript modules by their bare import-map names. ([JavaScript modules](#javascript-modules))
- No inline scripts, event handlers, or styles: modules and classes instead. ([The Content-Security-Policy](#the-content-security-policy))
- Give every image an `alt`. ([Referring to an asset](#referring-to-an-asset))

## Reference

- [Assets](../assets.md): extensions, URLs, the import map, Turbo, and the Content-Security-Policy in full.
- [HTTP kernel](../http.md#responses): the security headers every response carries.
- [Views and pages](../views.md): `v.document` and `v.img`.
