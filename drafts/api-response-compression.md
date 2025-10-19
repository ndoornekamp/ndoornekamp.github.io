---
title: API response compression
date:
permalink: /api-response-compression
---

API responses quite often contain a lot of repetitive data. Think of JSON responses with repeated object keys or HTML responses with repeated tags.

Clients can indicate that they support compressed responses by including an `Accept-Encoding` header in their request. Common compression algorithms include `gzip` and `zstd`. When a server sees this header, it can choose to compress the response body using one of the supported algorithms and include a `Content-Encoding` header in the response to indicate which compression method was used. The client (e.g. a web browser) then decompresses the response body based on this header.

Side note: an API in an AWS Lambda has a maximum response size of 6 MB (bumping in to this limit prompted me to look in to this topic for that project). Compressing responses can help stay within that limit.
