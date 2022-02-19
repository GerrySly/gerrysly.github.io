---
author: "Ben"
title: "CSS coverage"
date: "2022-02-19"
description: "CSS coverage and it's implication on performance"
tags: ["shortcodes", "privacy"]
ShowToc: false
draft: true
---
One typical workflow in my team is the development of a thing, the adoptance of said thing and a subsequent turn down or deprecation. Typically in the deprecation phase various code paths are removed and space is reclaimed. This is super useful as bloat in our JavaScript or C++ can cause load times (and sometimes lead to various tricky bugs).

An area that is often missed during this deprecation phase is the identification of CSS rules which will be orphaned by the removal of all this code. This leads to a proliferation of unused rules and can make the CSS rather unwieldy and depending on how long this problem goes unchecked, can lead to slower load times and bloated builds.

There are a number of open source tools out there that perform an analysis of the code base to identify unused CSS and report on it. For example [PurgeCSS](https://purgecss.com/) which uses Puppeteer and the Chrome Devtools CSS coverage tool to analyze and [PurifyCSS](https://github.com/purifycss/purifycss) which leverages static analysis.

At Google (specifically for my team) we're not using Puppeteer for our tests and static analysis fails when JavaScript dynamically appends and removes CSS classes at runtime. Our team uses a framework call [Browser Tests](https://www.chromium.org/developers/testing/browser-tests/) which sets up a Chrome browser process, loads all the test assets and instrumenting the browser behaves as a user would behave.

## Browser Tests

One of the nifty things with browser tests is you have deep access to the constituents that make Chrome so powerful, such as Devtools. We can hook into the Chrome Devtools Protocol (in a very similar way to Puppeteer) and pull out the CSS coverage information.

