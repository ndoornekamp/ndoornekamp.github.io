---
title: How I broke our application by migrating from `requests` to `httpx`
date: 2023-05-25
permalink: /requests-httpx-migration
---

Turns out `httpx` has a default timeout of 5 seconds, whereas `requests` has not.