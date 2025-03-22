---
title: Working with SIGTINT, SIGTERM and SIGKILL signals in Kubernetes and Python
date: 2025-02-12
permalink: /sigint-sigterm-sigkill-signals/
---

Signals are a way to (asynchronously) communicate with processes. They can originate from a user, another process or the kernel. They may be sent to a process to notify it of an event, or to request it to do something - for example, to terminate, which is the focus of this post.

| Signal  | Description                                                                                       | Example command          |
|---------|---------------------------------------------------------------------------------------------------|--------------------------|
| `SIGINT`| A polite request to terminate, specifically intended for use in a terminal. E.g. pressing `ctrl+C` in a terminal process sends a `SIGINT` (INT for 'interrupt') signal to the process. | `ctrl+C`                 |
| `SIGTERM`| The normal way of asking a process to terminate. It can still be caught and handled (or potentially ignored) by the process. | `kill <pid>`             |
| `SIGKILL`| Not catchable - the process is ruthlessly murdered.                                            | `kill -9 <pid>`          |

To illustrate: a system administrator could send a `SIGTERM` signal first and then give programs some time to shut down. Programs can use this time to, for example, close database connections or delete temporary files. Only if the process ignored it, escalate to a `SIGKILL` signal.

## Kubernetes

Kubernetes uses termination signals to manage containers. It sends a `SIGTERM` signal to a container when it wants to stop it. This could happen, for example, to replace it with a new version or because the spot node it runs on will shut down. By default, the container has 30 seconds to shut down after receiving a `SIGTERM` signal. This can be increased by setting `terminationGracePeriodSeconds`. After this period, a `SIGKILL` signal is sent.

## Handling signals in Python

When a Python process receives `SIGINT`, it raises a `KeyboardInterrupt` exception, which can be caught and handled e.g. with a `try`/`except`/`finally` block.

When receiving `SIGTERM`, no such thing happens by default and the process terminates immediately. In most cases that is fine, but in some cases you might want to handle it. For example, if the process that is being terminated is a job, you may need to store some state so you can pick up where you left off on the next run. To do so, you can use the `signal` module to register a handler for the `SIGTERM` signal:

```python
import time
import signal
from types import FrameType

class GracefulTerminationJob:
    def __init__(self):
        self.keep_running = True
        signal.signal(signal.SIGINT, self.log_signal)
        signal.signal(signal.SIGTERM, self.log_signal)

    def log_signal(self, signal: int, frame: FrameType):
        print(f"Received signal {signal} - stopping the job")
        self.keep_running = False

    def run(self):
        while self.keep_running:
            print("Doing some work...")
            time.sleep(1)
        print("Job stopped gracefully")

if __name__ == "__main__":
    job = GracefulTerminationJob()
    job.run()
```

Note that in this example, termination signals cause the loop in the `run` method to exit, after which the program reaches it's end and exits. Consider the following example:

```python
import time
import signal
import sys
from types import FrameType

def log_signal(signal: int, frame: Frametype):
    print(f"Received signal {signal}")
    sys.exit(1)  # Exit the program

def main():
    signal.signal(signal.SIGTERM, log_signal)

    while True:
        print("Doing some work...")
        time.sleep(1)

if __name__ == "__main__":
    main()
```

Without the `sys.exit(1)` call, the program catches and handles the `SIGTERM` signal but then continues running, probably until it's finally  killed by a `SIGKILL` signal.
