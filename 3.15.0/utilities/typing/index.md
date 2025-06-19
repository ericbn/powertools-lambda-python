This typing utility provides static typing classes that can be used to ease the development by providing the IDE type hints.

## Key features

- Add static typing classes
- Ease the development by leveraging your IDE's type hints
- Avoid common typing mistakes in Python

## Getting started

Tip

All examples shared in this documentation are available within the [project repository](https://github.com/aws-powertools/powertools-lambda-python/tree/develop/examples).

We provide static typing for any context methods or properties implemented by [Lambda context object](https://docs.aws.amazon.com/lambda/latest/dg/python-context.html).

## LambdaContext

The `LambdaContext` typing is typically used in the handler method for the Lambda function.

```
from aws_lambda_powertools.utilities.typing import LambdaContext


def handler(event: dict, context: LambdaContext) -> dict:
    # Insert business logic
    return event

```

## Working with context methods and properties

Using `LambdaContext` typing makes it possible to access information and hints of all properties and methods implemented by Lambda context object.

```
from time import sleep

import requests

from aws_lambda_powertools import Logger
from aws_lambda_powertools.utilities.typing import LambdaContext

logger = Logger()


def lambda_handler(event, context: LambdaContext) -> dict:
    limit_execution: int = 1000  # milliseconds

    # scrape website and exit before lambda timeout
    while context.get_remaining_time_in_millis() > limit_execution:
        comments: requests.Response = requests.get("https://jsonplaceholder.typicode.com/comments")
        # add logic here and save the results of the request to an S3 bucket, for example.

        logger.info(
            {
                "operation": "scrape_website",
                "request_id": context.aws_request_id,
                "remaining_time": context.get_remaining_time_in_millis(),
                "comments": comments.json()[:2],
            },
        )

        sleep(1)

    return {"message": "Success"}

```
