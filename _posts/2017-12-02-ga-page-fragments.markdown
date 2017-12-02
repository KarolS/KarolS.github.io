---
layout: post
title: "Adding support for page fragments to Google Analytics the quick and easy way"
date: 2017-11-02 23:40
categories: programming, JavaScript
---

By default, Google Analytics fires once per page load and therefore it only adds entries when the user navigates between different documents. But today, in the era of frequent aingle-page applications, this might not be enough. A web app might have multiple different sections, displayed conceptually from within the same document, and Google Analytics won't track where the use is navigating.

In search of a solution for tracking user behaviour accross the document, I found several guides that required to set up the Google Tag Manager. But I couldn't be bothered figuring yet another tool, so I decided to look for a more hacky and a simpler solution.

<!-- more -->

The solution is simple and as a prerequisite it requires that every change you wish to track causes a history event. Most SPA frameworks do that already to support the "back" button in browsers, so I'll assume it works. I'm also not giving any guarantees about cross-browser support, but it works in Firefox and Chrome.

Just paste the following just after you Google Analytics code:

```javascript
window.onpopstate = function(event) {
    ga('set', 'page', window.location.pathname + window.location.search + window.location.hash);
    ga('send','pageview');
};
```

(If you need to use more onpopstate events, you'll need to join the handlers yourself.)

This way, every time a new history event is pushed onto the history stack, GA will send an event that contains both the document path, the query parameters and the fragment name (so, something like `/document?query=parameters#fragment`). If you don't need some of those parts, or you want to track some more things, you can easily adapt the snippet for your needs.

Done! No need to learn about a new product and dive into unfamiliar territories, which would take hours, when a simple five-minute hack does the job.

A final detail: browsers differ on whether the event is fired on the initial page load, so you might want to take that into account.
