---
layout: post
title:  "Understanding v8 Coverage Ranges"
date:   2020-10-16
author: "Ben Reich"
tags:   code-coverage v8 javascript
---
Code coverage provides engineers with a metric of how much of their codebase has
been tested. Different programming languages come equipped with different tools
to capture this code coverage. For the purpose of this post I'll be delving into
the v8 JavaScript runtime.

The v8 runtime has been widely adopted and is being used as the default JS
rendering engine in Chrome and Node. In late 2017 the v8 team released the 
ability to enable code coverage on their runtime by sending a sequence of
messages via the Chrome DevTools Protocol (CDP) to the underlying v8 isoalte.

Prior to the release of this feature well known JS code coverage libraries were
inserting incrementers into their 