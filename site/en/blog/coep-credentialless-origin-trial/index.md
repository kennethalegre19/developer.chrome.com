---
layout: 'layouts/blog-post.njk'
title: "Load cross-origin resources without CORP headers using `COEP: credentialless`"
description: >
  `Cross-Origin-Embedder-Policy: credentialless` will be on origin trial starting Chrome 93. This new value allows web pages to enable cross-origin isolation without requiring cross-origin resources to respond with a `CORP: cross-origin` header by sending credentialless requests.
subhead: >
  Cross-origin resources served by third-parties often do not include adequate CORP
  headers. If they can be requested without credentials, now you can enable cross-origin
  isolation by marking them as such. 
date: 2021-07-29
authors:
  - agektmr
tags:
  - security
hero: 'image/YLflGBAPWecgtKJLqCJHSzHqe2J2/XCE0xAdnJfEUZokVykVE.jpg'
alt: >
  Anonymous people are walking a corridor.
---

We are experimenting with the new Cross-Origin Embedder Policy (COEP) value `credentialless` which allows the browser to load cross-origin resources without Cross-Origin Resource Policy (CORP) value `cross-origin`, by sending a request `credentialless`. This helps developers to adopt cross-origin isolation more easily.

## Why we need cross-origin isolation

Some web APIs increase the risk of side-channel attacks such as
[Spectre](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability)). To
mitigate that risk, browsers offer an opt-in-based isolated environment called
[cross-origin isolation](https://web.dev/coop-coep/). With a cross-origin
isolated state, the webpage can use privileged features including
[`SharedArrayBuffer`](https://developer.chrome.com/blog/enabling-shared-array-buffer/),
[`performance.measureUserAgentSpecificMemory()`](https://web.dev/monitor-total-page-memory-usage/),
[JS Self-Profiling API](https://github.com/WICG/js-self-profiling) and
[high-precision timers with better
resolution](https://developer.chrome.com/blog/cross-origin-isolated-hr-timers/)
while isolating the origin from others unless they are opted in.

The webpage must send two HTTP headers to enable cross-origin isolation:

```http
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Opener-Policy: same-origin
```

With a cross-origin isolated state, all cross-origin resources must be served
with CORS or `Cross-Origin-Resource-Policy: cross-origin` to be loaded.

{% Img src="image/YLflGBAPWecgtKJLqCJHSzHqe2J2/SmiNygdHglDvm1Et1hFQ.png",
alt="", width="800", height="372" %}

## Challenges with enabling cross-origin isolation

While cross-origin isolation brings webpages better security and ability to
enable powerful features, it's challenging for some of them [to meet all the
conditions required](https://web.dev/cross-origin-isolation-guide/). One of the
biggest challenges is to serve cross-origin resources with CORS or `CORP:
cross-origin` in order to be loaded. Resources without those headers will not be
loaded by the browser even if they are successfully served.

These cross-origin resources are usually served by third-parties and asking them
to make such changes turned out to be a big ask.

But what if we know the resource is safe enough to be loaded? In fact, the only
resources that are at risk are ones requested with credentials, because they
potentially include sensitive information which the attacker can't load on their
own. This means resources that can be requested without credentials are publicly
available and safe to load.

## `credentialless` to the rescue

That's where `COEP: credentialless` comes in. `credentialless` is a new value
option for the `Cross-Origin-Embedder-Policy` header. Similar to `require-corp`,
it can enable cross-origin isolation, but instead of requiring a
`CORP:cross-origin` header for no-cors cross-origin requests, they are instead
sent without credentials (for example, cookies).

You will be able to enable cross-origin isolation alternatively with the
following two headers:

```http
Cross-Origin-Embedder-Policy: credentialless
Cross-Origin-Opener-Policy: same-origin
```

This means the requested cross-origin server won't be able to respond with a
sensitive resource and the requester can always assume that the response only
contains publicly available information.

{% Img src="image/YLflGBAPWecgtKJLqCJHSzHqe2J2/lbeAQd4Zxnzti7SdHdGt.png",
alt="", width="800", height="397" %}

This is also aligned with browsers' plan of [phasing out third-party
cookies](https://blog.chromium.org/2020/01/building-more-private-web-path-towards.html).

## Demo

You can try various header options in this demo:
[https://first-party-test.glitch.me](https://first-party-test.glitch.me)

## FAQ

### Can I send a credentialed request under a `credentialless` environment?

Absolutely, at the cost of shifting the request's mode to require a CORS check
on the response. For HTML tags such as `<audio>`, `<img>`, `<link>`, `<script>`,
and `<video>`, just append `crossorigin="use-credentials"` explicitly to let the
browser inform to send credentialed requests.

For example, even if a document on `https://www.example.com` has
`Cross-Origin-Embedder-Policy: credentialless` header, `<img
src="https://images.example.com/avatar.png" crossorigin="use-credentials">` will
send a credentialed request.

For the `fetch()` API, `request.mode = 'cors'` can be used.

### Provided `COEP: credentialless`, how is `COEP: require-corp` still useful for my website?

`COEP: require-corp` doesn't require you to manually switch the request mode to
CORS if cookies are needed for some cross-origin subresources.

### Can I also load cross-origin iframes without special headers under a `credentialless` environment?

No. Loading cross-origin iframes under a `credentialless` environment still requires the same conditions as `require-corp`. iframe documents need to be served with two headers:

* `Cross-Origin-Embedder-Policy: credentialless` (or `require-corp`)
* `Cross-Origin-Resource-Policy: cross-origin`

The good news is, there's [an ongoing discussion about allowing loading
cross-origin iframes without those headers by giving iframes
`crossorigin="anonymous"`](https://github.com/camillelamy/explainers/blob/master/anonymous_iframes.md).
This will allow cross-origin iframes being loaded without headers but without
credentials.

### Will this feature be adopted by other browsers?

* Mozilla Request for position: [Worth
  prototyping](https://github.com/mozilla/standards-positions/issues/539)
* Webkit Request for position: [No
  signal](https://lists.webkit.org/pipermail/webkit-dev/2021-June/031898.html)
* [W3C TAG](https://www.w3.org/2001/tag/) Request for position:
  [Pending](https://github.com/w3ctag/design-reviews/issues/582)

## Register for an origin trial

To ensure that `COEP: credentialless` is helping developers to adopt cross
origin isolation, we are making it available in Chrome 93 as an origin trial.
You can register for it to allowlist your website to make the
`Cross-Origin-Embedder-Policy: credentialless` header to take effect.

1. [Request a
   token](https://developer.chrome.com/origintrials/#/view_trial/303992974847508481)
   for your origin.
2. Apply an `Origin-Trial` HTTP header to the document you want to apply
   `Cross-Origin-Embedder-Policy: credentialless` header. The resulting response
   header should look something like: `Origin-Trial: TOKEN_GOES_HERE`. (Adding
   the token as a meta tag is usually an option for many origin trials, but it
   won't work for this feature.)
3. Start serving `COEP: credentialless`.

For instance, to get a cross-origin isolated environment, the HTTP headers sent
are:

* `Origin-Trial: TOKEN_GOES_HERE`
* `Cross-Origin-Embedder-Policy: credentialless`
* `Cross-Origin-Opener-Policy:same-origin`

If you have any feedback on this functionality, file an issue at [the
GitHub repository](https://github.com/WICG/credentiallessness).

## What's coming next

There are two additional updates coming to mitigate other challenges related to
cross-origin isolation:

* [COOP:
  same-origin-allow-popups-plus-coep](https://github.com/camillelamy/explainers/blob/master/coi-with-popups.md)
* [Anonymous
  iframes](https://github.com/camillelamy/explainers/blob/master/anonymous_iframes.md)

Both of these changes are planned to land sometime in Q4 2021. Those who
registered for [the Chrome origin trial to extend the SharedArrayBuffer
change](https://developer.chrome.com/blog/enabling-shared-array-buffer/) due to
the above obstacles might be wondering when it will be terminated. Originally we
announced that it will be terminated in Chrome 96, but we will not do so until
these updates are in place.

## Resources

* [Making your website "cross-origin isolated" using COOP and
  COEP](https://web.dev/coop-coep/)
* [Why you need "cross-origin isolated" for powerful
  features](https://web.dev/why-coop-coep/)
* [A guide to enable cross-origin
  isolation](https://web.dev/cross-origin-isolation-guide/)
* [SharedArrayBuffer updates in Android Chrome 88 and Desktop Chrome
  92](https://developer.chrome.com/blog/enabling-shared-array-buffer/)

Photo by [Martin
Adams](https://unsplash.com/@martinadams?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
on
[Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
