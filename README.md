<div align="center">
  <h1>Teaflask Steep</h1>
  <p>See what each person did in your web app, one journey at a time.</p>
  <p>
    <a href="https://docs.teaflask.com/steep-install">Install guide</a> ·
    <a href="https://docs.teaflask.com">Docs</a> ·
    <a href="https://app.teaflask.com/">Dashboard</a> ·
    <a href="https://github.com/teaflask/steep/issues">Issues</a>
  </p>
  <p>
    <a href="https://www.npmjs.com/package/@teaflask/steep"><img src="https://img.shields.io/npm/v/@teaflask/steep?color=111" alt="npm version"></a>
    <a href="https://github.com/teaflask/steep/blob/main/LICENSE"><img src="https://img.shields.io/npm/l/@teaflask/steep?color=111" alt="License: FSL-1.1-ALv2"></a>
  </p>
</div>

[Teaflask](https://teaflask.com) Steep captures what people do in your web app (the pages they view, what they click, the fields they change, the forms they submit and the calls your app makes for them) and sends it to Teaflask, where your team sees each person's journey. `@teaflask/steep` is the browser part. It renders nothing and shows nothing to your users, and it redacts sensitive values in the browser before anything is sent.

## Install

You need a Steep key from the Steep keys page in the [dashboard](https://app.teaflask.com/). Teaflask accepts a key only from the sites (origins) you allow on it.

### npm

```sh
npm i @teaflask/steep
```

```ts
import { startSteep } from "@teaflask/steep";

const steep = startSteep({ steepKey: "stp_live_…" });
```

Call `startSteep` once, when your app starts. Importing the package does nothing by itself. Only one Steep runs per page, so a second start is refused with a console warning.

### Script tag

Add both lines to every page:

```html
<script async src="https://app.teaflask.com/steep/v1/steep.js"></script>
<teaflask-steep steep-key="stp_live_…"></teaflask-steep>
```

Steep starts once the element is on the page with a `steep-key`, so you can add the attribute later. Changing `steep-key` or `base-url` restarts it once (a `base-url` change continues the same journey), removing or emptying `steep-key` stops it, and removing the element stops it. Writing the same value again changes nothing, so it never restarts a Steep you stopped; adding the element back after removing it (as a framework does when it re-mounts it) starts Steep again, even after `stop()`.

## Identify signed-in users

Pass `userId` and `getEndUserToken` together. `getEndUserToken` returns a short-lived token your server signs: an HS256 JWT with `user_id` and `exp`, signed with the JWT secret from Settings → Keys in Teaflask, the same token the Teaflask assistant uses. `userId` must equal its `user_id`. Set both or neither: one alone keeps the journey anonymous and logs a console warning.

```ts
const getEndUserToken = () =>
  fetch("/api/teaflask-token").then((r) => r.text());

const steep = startSteep({
  steepKey: "stp_live_…",
  userId: user.id,
  getEndUserToken,
});

// Later: on sign-in, and on sign-out
steep.identify({ userId: user.id, getEndUserToken });
steep.identify(null);
```

Signing in attaches the current journey to the person. Signing out, or a different person signing in, starts a new journey.

With the script tag, set the same two properties on the element. Assign both in the same task and Steep signs in once; values you set before the script loads are picked up. On sign-out, set both to `null` or call `identify(null)`.

```js
const steep = document.querySelector("teaflask-steep");
steep.userId = user.id;
steep.getEndUserToken = () =>
  fetch("/api/teaflask-token").then((r) => r.text());
```

The [install guide](https://docs.teaflask.com/steep-install) shows how to sign the token on your server.

## Key settings

Two settings on each key decide whom Steep captures:

- **Respect browser privacy signals** (on by default). A browser that sends [Global Privacy Control](https://globalprivacycontrol.org/) isn't captured at all: Steep sends nothing, and its state reads `privacy-signal`.
- **Only capture signed-in users** (off by default). Steep captures nothing until your app signs the person in, keeps nothing from before the sign-in, and stops capturing when they sign out.

## You decide whom Steep captures

Steep shows no banner, prompt or notice. If your site needs consent, ask for it your own way, and call `startSteep` (or add the element) only after the person agrees.

```ts
import { isOptedOut, optIn, optOut } from "@teaflask/steep";

optOut(); // stops Steep, discards what wasn't sent, stays off here
isOptedOut(); // true
optIn(); // Steep runs again the next time it starts
```

- `steep.stop()` ends capture on this page, after sending what was already captured.
- `optOut()` stops Steep at once, discards anything not yet sent, and keeps Steep off in this browser, for this site, until `optIn()`. It returns `false` when the browser refused `localStorage`, so the choice could only be kept for this tab or this page load.
- `isOptedOut()` tells you whether this browser is opted out.
- The element offers the same `optOut()` and `optIn()`, and a `stop()` that holds until its `steep-key` or `base-url` changes or the element is removed and added back. To keep Steep off for good, use `optOut()` or remove the element.

Steep's opt-out lives only in this browser's storage. It doesn't follow the person to another browser or device, and clearing site data removes it. Keep the person's choice on your side too, and don't start Steep for someone who said no.

## What Steep captures

- **Page views and navigations**, including single-page app route changes: the scrubbed page address, the title, and only the origin of the referring site.
- **Clicks**: what was clicked, described by its role, accessible name, tag, test id, link path and nearby landmarks. Never its other attributes or its full text.
- **Field changes**: a field's value when it is committed (on change, blur or submit), never per keystroke. List and checkbox choices are kept as choices.
- **Form submits**: which form, and which button submitted it.
- **Network calls** made with `fetch` or `XMLHttpRequest`, as metadata only: method, origin, path, query parameter names, status, timing, the click or input that set the call off, and a GraphQL operation name. Never headers, bodies or parameter values.
- **Page outlines**, when the key's Page outlines setting is on: a text outline of what the page shows (headings, controls, labels and visible text), with field contents put through the same redaction.
- **Attention**: whether the page is visible, focused, or idle (a minute with no input).
- **Custom events** you add with `emit(name, properties)`: flat properties up to 4 KiB, with secrets scrubbed.

Only real input counts. Events your own scripts dispatch aren't captured.

### What Steep never captures

- Screenshots, video, or a copy of your page's HTML or styles.
- The contents of passwords, card numbers, one-time codes, or any field whose type, `autocomplete`, name or label says it holds something sensitive. Steep notes only that the field changed.
- Secrets such as API keys and tokens, wherever they appear in captured text. They are replaced with a `[redacted]` marker.
- Credentials, tracking parameters (`utm_*`, `gclid`, `fbclid` and similar) and secret-looking query values in addresses.
- With the key's Typed values setting off: field text (only its length is kept), query values in page addresses, and typed text that resurfaces in a title or address.
- Anything inside an ignored region, or inside the Teaflask assistant on the same page.

## Keep part of a page out

Add `data-teaflask-ignore` to an element to keep it and everything inside it out of what Steep captures:

```html
<div data-teaflask-ignore>…</div>
```

Clicks, field changes and form submits inside it are skipped, with only a count kept that something was skipped. This reaches into open shadow roots below it. A click on an element that contains an ignored region is skipped too, since its name could repeat what's inside. Network calls your code makes from there are still captured, as metadata only. You can also add CSS selectors as ignored regions on the key in the dashboard; if a browser can't parse one, Steep turns typed values off on that page.

## Browser storage

Steep keeps a few entries under the `tf-steep:` prefix:

- `tf-steep:session` in `sessionStorage`: the current journey's id and timing, so a reload continues it in this tab. By default a journey ends after 30 minutes without activity, or 12 hours.
- `tf-steep:identity` in `sessionStorage`: the signed-in journey's current Steep token.
- `tf-steep:config` in `sessionStorage`: your key's settings, cached for a minute.
- `tf-steep:opt-out` in `localStorage` (or `sessionStorage` when `localStorage` is blocked): the opt-out flag.

Steep sets no cookies, and its requests to Teaflask carry none.

## Content Security Policy

```text
script-src https://app.teaflask.com
connect-src https://api.teaflask.com
```

`script-src` is needed only for the script tag.

## Coverage limits

- Network calls that start before Steep is capturing aren't captured: with the `async` script tag, calls made before it loads, and with either install, calls made before Steep has your key's settings.
- `navigator.sendBeacon`, WebSockets and Server-Sent Events aren't captured.
- Content inside iframes isn't captured.
- Shadow DOM: open shadow roots are covered for clicks and field changes, but a form submitted inside a shadow root isn't captured as a form submit (its fields are still captured when they lose focus). A closed shadow root is seen only as its host element.
- A GraphQL operation name is read from the request body by its first match, so an `operationName` nested in variables, or in a batched request beyond the first operation, can be attributed wrongly.
- Steep's `fetch` wrapper returns a mirror of your promise that settles exactly as yours does, so an unhandled rejection behaves as it would without Steep.

## Versions

The script tag's `/v1/` URL is a rolling channel: it always serves the latest release, so your pages get fixes without changing the tag. If you need a fixed version, install from npm and pin it there. The package is pre-1.0: a breaking change bumps the minor version, and the [changelog](https://github.com/teaflask/steep/blob/main/CHANGELOG.md) lists what changed in every release. Every public export and field is described in the package's types, so your editor and coding agent can read what each one means.

Read the [install guide](https://docs.teaflask.com/steep-install) for the full setup.

## License

[Functional Source License, Version 1.1, ALv2 Future License](https://github.com/teaflask/steep/blob/main/LICENSE) (FSL-1.1-ALv2). You can use it in your own product. You can't use it to build a competing product or service. Every release becomes Apache-2.0 two years after it ships.

Report security issues to [security@teaflask.com](mailto:security@teaflask.com); see [SECURITY.md](https://github.com/teaflask/steep/blob/main/SECURITY.md).
