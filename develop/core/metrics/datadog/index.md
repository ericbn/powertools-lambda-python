This observability provider creates custom metrics by flushing metrics to [Datadog Lambda extension](https://docs.datadoghq.com/serverless/installation/python/?tab=datadogcli), or to standard output via [Datadog Forwarder](https://docs.datadoghq.com/logs/guide/forwarder/?tab=cloudformation). These metrics can be visualized in the [Datadog console](https://app.datadoghq.com/metric/explore).

```
stateDiagram-v2
    direction LR
    LambdaFn: Your Lambda function
    LambdaCode: DatadogMetrics
    DatadogSDK: Datadog SDK
    DatadogExtension: Datadog Lambda Extension
    Datadog: Datadog Dashboard
    LambdaExtension: Lambda Extension

    LambdaFn --> LambdaCode
    LambdaCode --> DatadogSDK
    DatadogSDK --> DatadogExtension
    DatadogExtension --> Datadog: async

    state LambdaExtension {
        DatadogExtension
    }

```

## Key features

- Flush metrics to Datadog extension or standard output
- Validate against common metric definitions mistakes
- Support to add default tags

## Terminologies

If you're new to Datadog Metrics, there are three terminologies you must be aware of before using this utility:

- **Namespace**. It's the highest level container that will group multiple metrics from multiple services for a given application, for example `ServerlessEcommerce`.
- **Metric**. It's the name of the metric, for example: SuccessfulBooking or UpdatedBooking.
- **Tags**. Metrics metadata in key-value pair format. They help provide contextual information, and filter org organize metrics.

You can read more details in the [Datadog official documentation](https://docs.datadoghq.com/metrics/custom_metrics/).

## Getting started

Tip

All examples shared in this documentation are available within the [project repository](https://github.com/aws-powertools/powertools-lambda-python/tree/develop/examples).

### Install

> **Using Datadog Forwarder?** You can skip this step.

We recommend using [Datadog SDK](https://docs.datadoghq.com/serverless/installation/python/) and Datadog Lambda Extension with this feature for optimal results.

For Datadog SDK, you can add `aws-lambda-powertools[datadog]` as a dependency in your preferred tool, or as a Lambda Layer in the following example:

```
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31
Description: Powertools for AWS Lambda (Python) version

Globals:
  Function:
    Timeout: 5
    Runtime: python3.12
    Tracing: Active
    Environment:
      Variables:
        POWERTOOLS_METRICS_NAMESPACE: ServerlessAirline
        # [Production setup]
        # DATADOG_API_KEY_SECRET_ARN: "<AWS Secrets Manager Secret ARN containing Datadog API key>"
        # [Development only]
        DD_API_KEY: "<YOUR DATADOG API KEY>"
        # Configuration details: https://docs.datadoghq.com/serverless/installation/python/?tab=datadogcli
        DD_SITE: datadoghq.com

    Layers:
      # Find the latest Layer version in the official documentation
      # https://docs.powertools.aws.dev/lambda/python/latest/#lambda-layer
      - !Sub arn:aws:lambda:${AWS::Region}:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:15
      # Find the latest Layer version in the Datadog official documentation

      # Datadog SDK
      # Latest versions: https://github.com/DataDog/datadog-lambda-python/releases
      - !Sub arn:aws:lambda:${AWS::Region}:464622532012:layer:Datadog-Python312:78

      # Datadog Lambda Extension
      # Latest versions: https://github.com/DataDog/datadog-lambda-extension/releases
      - !Sub arn:aws:lambda:${AWS::Region}:464622532012:layer:Datadog-Extension:45

Resources:
  CaptureLambdaHandlerExample:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ../src
      Handler: capture_lambda_handler.handler

```

### Creating metrics

You can create metrics using `add_metric`.

By default, we will generate the current timestamp for you. Alternatively, you can use the `timestamp` parameter to set a custom one in epoch time.

```
from aws_lambda_powertools.metrics.provider.datadog import DatadogMetrics
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = DatadogMetrics()


@metrics.log_metrics  # ensures metrics are flushed upon request completion/failure
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_metric(name="SuccessfulBooking", value=1)

```

```
import time

from aws_lambda_powertools.metrics.provider.datadog import DatadogMetrics
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = DatadogMetrics()


@metrics.log_metrics  # ensures metrics are flushed upon request completion/failure
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_metric(name="SuccessfulBooking", value=1, timestamp=int(time.time()))

```

Warning: Do not create metrics outside the handler

Metrics added in the global scope will only be added during cold start. Disregard if you that's the intended behavior.

### Adding tags

You can add any number of tags to your metrics via keyword arguments (`key=value`). They are helpful to filter, organize, and aggregate your metrics later.

We will emit a warning for tags [beyond the 200 chars limit](https://docs.datadoghq.com/getting_started/tagging/).

```
from aws_lambda_powertools.metrics.provider.datadog import DatadogMetrics
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = DatadogMetrics()


@metrics.log_metrics  # ensures metrics are flushed upon request completion/failure
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_metric(name="SuccessfulBooking", value=1, tag1="powertools", tag2="python")

```

### Adding default tags

You can persist tags across Lambda invocations and `DatadogMetrics` instances via `set_default_tags` method, or `default_tags` parameter in the `log_metrics` decorator.

If you'd like to remove them at some point, you can use the `clear_default_tags` method.

Metric tag takes precedence over default tags of the same name

When adding tags with the same name via `add_metric` and `set_default_tags`, `add_metric` takes precedence.

```
from aws_lambda_powertools.metrics.provider.datadog import DatadogMetrics
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = DatadogMetrics()
metrics.set_default_tags(tag1="powertools", tag2="python")


@metrics.log_metrics  # ensures metrics are flushed upon request completion/failure
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_metric(name="SuccessfulBooking", value=1)

```

```
from aws_lambda_powertools.metrics.provider.datadog import DatadogMetrics
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = DatadogMetrics()

default_tags = {"tag1": "powertools", "tag2": "python"}


@metrics.log_metrics(default_tags=default_tags)  # ensures metrics are flushed upon request completion/failure
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_metric(name="SuccessfulBooking", value=1)

```

### Flushing metrics

Use `log_metrics` decorator to automatically serialize and flush your metrics (SDK or Forwarder) at the end of your invocation.

This decorator also ensures metrics are flushed in the event of an exception, including warning you in case you forgot to add metrics.

```
from aws_lambda_powertools.metrics.provider.datadog import DatadogMetrics
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = DatadogMetrics()


@metrics.log_metrics  # ensures metrics are flushed upon request completion/failure
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_metric(name="SuccessfulBooking", value=1, tag1="powertools", tag2="python")

```

```
{
   "m":"SuccessfulBooking",
   "v":1,
   "e":1691707076,
   "t":[
      "tag1:powertools",
      "tag2:python"
   ]
}

```

#### Raising SchemaValidationError on empty metrics

Use `raise_on_empty_metrics=True` if you want to ensure at least one metric is always emitted.

```
from aws_lambda_powertools.metrics.provider.datadog import DatadogMetrics
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = DatadogMetrics()


@metrics.log_metrics(raise_on_empty_metrics=True)  # ensures metrics are flushed upon request completion/failure
def lambda_handler(event: dict, context: LambdaContext):
    # no metrics being created will now raise SchemaValidationError
    return

```

Suppressing warning messages on empty metrics

If you expect your function to execute without publishing metrics every time, you can suppress the warning with **`warnings.filterwarnings("ignore", "No application metrics to publish*")`**.

### Capturing cold start metric

You can optionally capture cold start metrics with `log_metrics` decorator via `capture_cold_start_metric` param.

```
from aws_lambda_powertools.metrics.provider.datadog import DatadogMetrics
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = DatadogMetrics()


@metrics.log_metrics(capture_cold_start_metric=True)
def lambda_handler(event: dict, context: LambdaContext):
    return

```

```
{
    "m":"ColdStart",
    "v":1,
    "e":1691707488,
    "t":[
       "function_name:HelloWorldFunction"
    ]
 }

```

If it's a cold start invocation, this feature will:

- Create a separate Datadog metric solely containing a metric named `ColdStart`
- Add `function_name` metric tag

This has the advantage of keeping cold start metric separate from your application metrics, where you might have unrelated tags.

Info

We do not emit 0 as a value for ColdStart metric for cost reasons. [Let us know](https://github.com/aws-powertools/powertools-lambda-python/issues/new?assignees=&labels=feature-request%2C+triage&template=feature_request.md&title=) if you'd prefer a flag to override it.

### Environment variables

You can use any of the following environment variables to configure `DatadogMetrics`:

| Setting | Description | Environment variable | Constructor parameter | | --- | --- | --- | --- | | **Metric namespace** | Logical container where all metrics will be placed e.g. `ServerlessAirline` | `POWERTOOLS_METRICS_NAMESPACE` | `namespace` | | **Flush to log** | Use this when you want to flush metrics to be exported through Datadog Forwarder | `DD_FLUSH_TO_LOG` | `flush_to_log` | | **Disable Powertools Metrics** | Optionally, disables all Powertools metrics. | `POWERTOOLS_METRICS_DISABLED` | N/A |

Info

`POWERTOOLS_METRICS_DISABLED` will not disable default metrics created by AWS services.

## Advanced

### Flushing metrics manually

If you are using the [AWS Lambda Web Adapter](https://github.com/awslabs/aws-lambda-web-adapter) project, or a middleware with custom metric logic, you can use `flush_metrics()`. This method will serialize, print metrics available to standard output, and clear in-memory metrics data.

Warning

This does not capture Cold Start metrics, and metric data validation still applies.

Contrary to the `log_metrics` decorator, you are now also responsible to flush metrics in the event of an exception.

```
from aws_lambda_powertools.metrics.provider.datadog import DatadogMetrics
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = DatadogMetrics()


def book_flight(flight_id: str, **kwargs):
    # logic to book flight
    ...
    metrics.add_metric(name="SuccessfulBooking", value=1)


def lambda_handler(event: dict, context: LambdaContext):
    try:
        book_flight(flight_id=event.get("flight_id", ""))
    finally:
        metrics.flush_metrics()

```

### Integrating with Datadog Forwarder

Use `flush_to_log=True` in `DatadogMetrics` to integrate with the legacy [Datadog Forwarder](https://docs.datadoghq.com/logs/guide/forwarder/?tab=cloudformation).

This will serialize and flush metrics to standard output.

```
from aws_lambda_powertools.metrics.provider.datadog import DatadogMetrics
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = DatadogMetrics(flush_to_log=True)


@metrics.log_metrics  # ensures metrics are flushed upon request completion/failure
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_metric(name="SuccessfulBooking", value=1)

```

```
{
   "m":"SuccessfulBooking",
   "v":1,
   "e":1691768022,
   "t":[

   ]
}

```

## Testing your code

### Setting environment variables

Tip

Ignore this section, if:

- You are explicitly setting namespace via `namespace` parameter
- You're not instantiating `DatadogMetrics` in the global namespace

For example, `DatadogMetrics(namespace="ServerlessAirline")`

Make sure to set `POWERTOOLS_METRICS_NAMESPACE` before running your tests to prevent failing on `SchemaValidation` exception. You can set it before you run tests or via pytest plugins like [dotenv](https://pypi.org/project/pytest-dotenv/).

```
POWERTOOLS_METRICS_NAMESPACE="ServerlessAirline" DD_FLUSH_TO_LOG="True" python -m pytest # (1)!

```

1. **`DD_FLUSH_TO_LOG=True`** makes it easier to test by flushing final metrics to standard output.

### Clearing metrics

`DatadogMetrics` keep metrics in memory across multiple instances. If you need to test this behavior, you can use the following Pytest fixture to ensure metrics are reset incl. cold start:

```
import pytest

from aws_lambda_powertools.metrics.provider import cold_start
from aws_lambda_powertools.metrics.provider.datadog import DatadogMetrics


@pytest.fixture(scope="function", autouse=True)
def reset_metric_set():
    # Clear out every metric data prior to every test
    metrics = DatadogMetrics()
    metrics.clear_metrics()
    cold_start.is_cold_start = True  # ensure each test has cold start
    yield

```

### Functional testing

You can read standard output and assert whether metrics have been flushed. Here's an example using `pytest` with `capsys` built-in fixture:

```
import add_datadog_metrics


def test_log_metrics(capsys):
    add_datadog_metrics.lambda_handler({}, {})

    log = capsys.readouterr().out.strip()  # remove any extra line

    assert "SuccessfulBooking" in log  # basic string assertion in JSON str

```

```
from aws_lambda_powertools.metrics.provider.datadog import DatadogMetrics
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = DatadogMetrics()


@metrics.log_metrics  # ensures metrics are flushed upon request completion/failure
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_metric(name="SuccessfulBooking", value=1)

```

Tip

For more elaborate assertions and comparisons, check out [our functional testing for DatadogMetrics utility.](https://github.com/aws-powertools/powertools-lambda-python/blob/develop/tests/functional/metrics/datadog/test_metrics_datadog.py)
