---
title: Receiving signals in Docker
date: 2025-02-19
permalink: /receiving-signals-in-docker
---

In a Dockerfile, `RUN`, `CMD` and `ENTRYPOINT` instructions all have two possible forms:

- JSON array: `["command", "param1", "param2"]` (also known as 'exec form');
- Shell form: `command param1 param2`.

When you use shell form, the executable runs as a child process, which doesn't pass [signals](/receiving-signals-in-docker). Therefore, Docker [recommends](https://docs.docker.com/reference/build-checks/json-args-recommended/) using the JSON array form.

However, the shell form is more flexible. For example, it allows variable substitution. (This can be useful, for example, when you want to reuse a Dockerfile for different projects by using an instruction like `CMD python -m ${PROJECT}`.)

If you want the best of both worlds (i.e. a container that receives signals, but a Dockerfile with shell form instructions), you can use `exec` to run the command in the shell form:

```diff
- CMD python -m ${PROJECT}
+ CMD ["sh", "-c", "exec python -m ${PROJECT}"]
```

## Resources & further reading

- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/#shell-and-exec-form)
- [Why Your Dockerized Application Isn’t Receiving Signals](https://hynek.me/articles/docker-signals/)
