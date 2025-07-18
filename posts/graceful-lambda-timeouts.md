---
title: Handling Lambda timeouts with grace
date: 2025-06-18
permalink: /graceful-lambda-timeouts
---

AWS Lambda is meant for quick workloads and has a configurable maximum execution time which can't be set to more than 15 minutes. While this is well-known, you may still end up with Lambdas that sometimes take longer than their configured maximum excecution time. Such workloads are terminated abruptly as soon as they exceed the specified timeout, without an opportunity to log what happened or clean up after themselves.

This note does not cover ways to actually run the workload successfully, such as using a different technology (an [ECS Task](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html) if you want to stick with AWS, for example) or optimizing your workload to run faster. It only aims to provide a way to detect the imminent timeout and handle it gracefully, allowing for cleanup and logging.

## How to

In short: the Lambda context provides the method `context.get_remaining_time_in_millis()`, which can be used to see how much time is left before your Lambda is terminated. The code below shows how to set an alarm (a [SIGALRM signal](https://www.man7.org/linux/man-pages/man2/alarm.2.html)) just before that moment, so we have some time to exit gracefully.

```python
import signal
import logging
from types import FrameType
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from aws_lambda_typing.context import Context as LambdaContextType

logger = logging.getLogger(__name__)
lambda_timeout_buffer_seconds = 5


def handler(event, context: "LambdaContextType"):
    _setup_timeout_handler(context)

    try:
        pass  # Your workload here
    except TimeoutError:
        logger.error("handler is about to exceed its timeout")
    finally:
        signal.alarm(0)  # clean up the alarm


def _setup_timeout_handler(context: "LambdaContextType"):
    """
    Sets up:
     1. A SIGALRM signal to be sent just before the Lambda times out, giving us some time to handle it
     2. A handler for the SIGALRM signal
    """
    remaining_execution_time_seconds = context.get_remaining_time_in_millis() // 1000
    signal.alarm(max(1, remaining_execution_time_seconds - lambda_timeout_buffer_seconds))

    signal.signal(signal.SIGALRM, _timeout_handler)


def _timeout_handler(_signal_code: int, _frame: FrameType | None) -> None:
    raise TimeoutError("Execution is about to time out")
```

## Resources and further reading

- [CloudFormation custom resources](https://github.com/stelligent/cloudformation-custom-resources/blob/master/lambda/python/customresource.py)
- [Using the Lambda context object to retrieve Python function information](https://docs.aws.amazon.com/lambda/latest/dg/python-context.html)
