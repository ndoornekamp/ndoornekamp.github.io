---
title: API response compression
date: 2025-12-27
permalink: /api-response-compression
---

API responses quite often contain a lot of repetitive data. Think of JSON responses with repeated object keys or HTML responses with repeated tags. Data like this often lends itself well to compression, resulting in smaller response sizes and therefore less network traffic. This comes at the cost of some CPU time spent compressing and decompressing the data.

(For me personally, the concrete reason for looking in to this was that AWS Lambda has a maximum response size of 6 MB; compressing responses helped me stay within that limit.)

For example, consider this dummy FastAPI application:

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/data")
def get_data():
    return {f"item{i}": {"key1": i, "key2": i * 2} for i in range(10_000)}
```

A client can fetch data from the `/data` endpoint and receive an uncompressed JSON response using a tool like `curl`. The following command fetches the data and prints the size of the response in bytes:

```bash
curl -s -o /dev/null -w "%{size_download}\n" http://localhost:8000/data
> 372226
```

Enabling compression in this example is not difficult with the `GZipMiddleware` that is shipped with FastAPI:

```diff
from fastapi import FastAPI
+ from fastapi.middleware.gzip import GZipMiddleware

app = FastAPI()
+ app.add_middleware(GZipMiddleware, minimum_size=100)

@app.get("/data")
def get_data():
    return {f"item{i}": {"key1": i, "key2": i * 2} for i in range(10_000)}
```

Now clients can indicate that they support compressed responses by including an `Accept-Encoding` header in their request. In most cases, clients automatically decompress the response if it is compressed. The value for this header specifies which compression algorithms the client supports, e.g. `gzip`, `deflate`, or `br` (Brotli).

When a server sees an `Accept-Encoding` header, it can choose to compress the response body using one of the supported algorithms and include a `Content-Encoding` header in the response to indicate which compression method was used. The client (e.g. a web browser) then decompresses the response body based on this header.

With compression:

```bash
curl -s -o /dev/null -w "%{size_download}\n" http://localhost:8000/data -H "Accept-Encoding: gzip"
> 69162
```

Ta-da! The response size is more than 5 times smaller.

## Resources and further reading

- [FastAPI GZipMiddleware documentation](https://fastapi.tiangolo.com/tutorial/middleware/#gzipmiddleware)
- [MDN web docs on Content-Encoding](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Encoding)
- [ZSTD vs Brotli vs Gzip](https://speedvitals.com/blog/zstd-vs-brotli-vs-gzip/)
