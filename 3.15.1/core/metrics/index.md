Metrics creates custom metrics asynchronously by logging metrics to standard output following [Amazon CloudWatch Embedded Metric Format (EMF)](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Embedded_Metric_Format.html).

These metrics can be visualized through [Amazon CloudWatch Console](https://console.aws.amazon.com/cloudwatch/).

## Key features

- Aggregate up to 100 metrics using a single CloudWatch EMF object (large JSON blob)
- Validate against common metric definitions mistakes (metric unit, values, max dimensions, max metrics, etc)
- Metrics are created asynchronously by CloudWatch service, no custom stacks needed
- Context manager to create a one off metric with a different dimension

## Terminologies

If you're new to Amazon CloudWatch, there are five terminologies you must be aware of before using this utility:

- **Namespace**. It's the highest level container that will group multiple metrics from multiple services for a given application, for example `ServerlessEcommerce`.
- **Dimensions**. Metrics metadata in key-value format. They help you slice and dice metrics visualization, for example `ColdStart` metric by Payment `service`.
- **Metric**. It's the name of the metric, for example: `SuccessfulBooking` or `UpdatedBooking`.
- **Unit**. It's a value representing the unit of measure for the corresponding metric, for example: `Count` or `Seconds`.
- **Resolution**. It's a value representing the storage resolution for the corresponding metric. Metrics can be either Standard or High resolution. Read more [here](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/publishingMetrics.html#high-resolution-metrics).

Metric terminology, visually explained

## Getting started

Tip

All examples shared in this documentation are available within the [project repository](https://github.com/aws-powertools/powertools-lambda-python/tree/develop/examples).

Metric has two global settings that will be used across all metrics emitted:

| Setting | Description | Environment variable | Constructor parameter | | --- | --- | --- | --- | | **Metric namespace** | Logical container where all metrics will be placed e.g. `ServerlessAirline` | `POWERTOOLS_METRICS_NAMESPACE` | `namespace` | | **Service** | Optionally, sets **service** metric dimension across all metrics e.g. `payment` | `POWERTOOLS_SERVICE_NAME` | `service` |

Info

`POWERTOOLS_METRICS_DISABLED` will not disable default metrics created by AWS services.

Tip

Use your application or main service as the metric namespace to easily group all metrics.

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
        POWERTOOLS_SERVICE_NAME: booking
        POWERTOOLS_METRICS_NAMESPACE: ServerlessAirline
        POWERTOOLS_METRICS_FUNCTION_NAME: my-function-name

    Layers:
      # Find the latest Layer version in the official documentation
      # https://docs.powertools.aws.dev/lambda/python/latest/#lambda-layer
      - !Sub arn:aws:lambda:${AWS::Region}:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18

Resources:
  CaptureLambdaHandlerExample:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ../src
      Handler: capture_lambda_handler.handler

```

Note

For brevity, all code snippets in this page will rely on environment variables above being set.

This ensures we instantiate `metrics = Metrics()` over `metrics = Metrics(service="booking", namespace="ServerlessAirline")`, etc.

### Creating metrics

You can create metrics using `add_metric`, and you can create dimensions for all your aggregate metrics using `add_dimension` method.

Tip

You can initialize Metrics in any other module too. It'll keep track of your aggregate metrics in memory to optimize costs (one blob instead of multiples).

```
from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = Metrics()


@metrics.log_metrics  # ensures metrics are flushed upon request completion/failure
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_metric(name="SuccessfulBooking", unit=MetricUnit.Count, value=1)

```

```
import os

from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

STAGE = os.getenv("STAGE", "dev")
metrics = Metrics()


@metrics.log_metrics  # ensures metrics are flushed upon request completion/failure
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_dimension(name="environment", value=STAGE)
    metrics.add_metric(name="SuccessfulBooking", unit=MetricUnit.Count, value=1)

```

Tip: Autocomplete Metric Units

`MetricUnit` enum facilitate finding a supported metric unit by CloudWatch. Alternatively, you can pass the value as a string if you already know them *e.g. `unit="Count"`*.

Note: Metrics overflow

CloudWatch EMF supports a max of 100 metrics per batch. Metrics utility will flush all metrics when adding the 100th metric. Subsequent metrics (101th+) will be aggregated into a new EMF object, for your convenience.

Warning: Do not create metrics or dimensions outside the handler

Metrics or dimensions added in the global scope will only be added during cold start. Disregard if that's the intended behavior.

### Adding high-resolution metrics

You can create [high-resolution metrics](https://aws.amazon.com/about-aws/whats-new/2023/02/amazon-cloudwatch-high-resolution-metric-extraction-structured-logs/) passing `resolution` parameter to `add_metric`.

When is it useful?

High-resolution metrics are data with a granularity of one second and are very useful in several situations such as telemetry, time series, real-time incident management, and others.

```
from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricResolution, MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = Metrics()


@metrics.log_metrics  # ensures metrics are flushed upon request completion/failure
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_metric(name="SuccessfulBooking", unit=MetricUnit.Count, value=1, resolution=MetricResolution.High)

```

Tip: Autocomplete Metric Resolutions

`MetricResolution` enum facilitates finding a supported metric resolution by CloudWatch. Alternatively, you can pass the values 1 or 60 (must be one of them) as an integer *e.g. `resolution=1`*.

### Adding multi-value metrics

You can call `add_metric()` with the same metric name multiple times. The values will be grouped together in a list.

```
import os

from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

STAGE = os.getenv("STAGE", "dev")
metrics = Metrics()


@metrics.log_metrics  # ensures metrics are flushed upon request completion/failure
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_dimension(name="environment", value=STAGE)
    metrics.add_metric(name="TurbineReads", unit=MetricUnit.Count, value=1)
    metrics.add_metric(name="TurbineReads", unit=MetricUnit.Count, value=8)

```

```
{
    "_aws": {
        "Timestamp": 1656685750622,
        "CloudWatchMetrics": [
            {
                "Namespace": "ServerlessAirline",
                "Dimensions": [
                    [
                        "environment",
                        "service"
                    ]
                ],
                "Metrics": [
                    {
                        "Name": "TurbineReads",
                        "Unit": "Count"
                    }
                ]
            }
        ]
    },
    "environment": "dev",
    "service": "booking",
    "TurbineReads": [
        1.0,
        8.0
    ]
}

```

### Adding default dimensions

You can use `set_default_dimensions` method, or `default_dimensions` parameter in `log_metrics` decorator, to persist dimensions across Lambda invocations.

If you'd like to remove them at some point, you can use `clear_default_dimensions` method.

```
import os

from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

STAGE = os.getenv("STAGE", "dev")
metrics = Metrics()
metrics.set_default_dimensions(environment=STAGE, another="one")


@metrics.log_metrics  # ensures metrics are flushed upon request completion/failure
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_metric(name="TurbineReads", unit=MetricUnit.Count, value=1)
    metrics.add_metric(name="TurbineReads", unit=MetricUnit.Count, value=8)

```

```
import os

from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

STAGE = os.getenv("STAGE", "dev")
metrics = Metrics()
DEFAULT_DIMENSIONS = {"environment": STAGE, "another": "one"}


# ensures metrics are flushed upon request completion/failure
@metrics.log_metrics(default_dimensions=DEFAULT_DIMENSIONS)
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_metric(name="TurbineReads", unit=MetricUnit.Count, value=1)
    metrics.add_metric(name="TurbineReads", unit=MetricUnit.Count, value=8)

```

**Note:** Dimensions with empty values will not be included.

### Changing default timestamp

When creating metrics, we use the current timestamp. If you want to change the timestamp of all the metrics you create, utilize the `set_timestamp` function. You can specify a datetime object or an integer representing an epoch timestamp in milliseconds.

Note that when specifying the timestamp using an integer, it must adhere to the epoch timezone format in milliseconds.

Info

If you need to use different timestamps across multiple metrics, opt for [single_metric](#working-with-different-timestamp).

```
import datetime

from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = Metrics()


@metrics.log_metrics  # ensures metrics are flushed upon request completion/failure
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_metric(name="SuccessfulBooking", unit=MetricUnit.Count, value=1)

    metric_timestamp = int((datetime.datetime.now() - datetime.timedelta(days=2)).timestamp() * 1000)
    metrics.set_timestamp(metric_timestamp)

```

### Flushing metrics

As you finish adding all your metrics, you need to serialize and flush them to standard output. You can do that automatically with the `log_metrics` decorator.

This decorator also **validates**, **serializes**, and **flushes** all your metrics. During metrics validation, if no metrics are provided then a warning will be logged, but no exception will be raised.

```
from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = Metrics()


@metrics.log_metrics  # ensures metrics are flushed upon request completion/failure
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_metric(name="SuccessfulBooking", unit=MetricUnit.Count, value=1)

```

```
{
    "_aws": {
        "Timestamp": 1656686788803,
        "CloudWatchMetrics": [
            {
                "Namespace": "ServerlessAirline",
                "Dimensions": [
                    [
                        "service"
                    ]
                ],
                "Metrics": [
                    {
                        "Name": "SuccessfulBooking",
                        "Unit": "Count"
                    }
                ]
            }
        ]
    },
    "service": "booking",
    "SuccessfulBooking": [
        1.0
    ]
}

```

Tip: Metric validation

If metrics are provided, and any of the following criteria are not met, **`SchemaValidationError`** exception will be raised:

- Maximum of 29 user-defined dimensions
- Namespace is set, and no more than one
- Metric units must be [supported by CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_MetricDatum.html)

#### Raising SchemaValidationError on empty metrics

If you want to ensure at least one metric is always emitted, you can pass `raise_on_empty_metrics` to the **log_metrics** decorator:

```
from aws_lambda_powertools.metrics import Metrics
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = Metrics()


@metrics.log_metrics(raise_on_empty_metrics=True)
def lambda_handler(event: dict, context: LambdaContext):
    # no metrics being created will now raise SchemaValidationError
    ...

```

Suppressing warning messages on empty metrics

If you expect your function to execute without publishing metrics every time, you can suppress the warning with **`warnings.filterwarnings("ignore", "No application metrics to publish*")`**.

### Capturing cold start metric

You can optionally capture cold start metrics with `log_metrics` decorator via `capture_cold_start_metric` param.

```
from aws_lambda_powertools import Metrics
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = Metrics()


@metrics.log_metrics(capture_cold_start_metric=True)
def lambda_handler(event: dict, context: LambdaContext): ...

```

```
{
    "_aws": {
        "Timestamp": 1656687493142,
        "CloudWatchMetrics": [
            {
                "Namespace": "ServerlessAirline",
                "Dimensions": [
                    [
                        "function_name",
                        "service"
                    ]
                ],
                "Metrics": [
                    {
                        "Name": "ColdStart",
                        "Unit": "Count"
                    }
                ]
            }
        ]
    },
    "function_name": "test",
    "service": "booking",
    "ColdStart": [
        1.0
    ]
}

```

If it's a cold start invocation, this feature will:

- Create a separate EMF blob solely containing a metric named `ColdStart`
- Add `function_name` and `service` dimensions

This has the advantage of keeping cold start metric separate from your application metrics, where you might have unrelated dimensions.

Info

We do not emit 0 as a value for ColdStart metric for cost reasons. [Let us know](https://github.com/aws-powertools/powertools-lambda-python/issues/new?assignees=&labels=feature-request%2C+triage&template=feature_request.md&title=) if you'd prefer a flag to override it.

#### Customizing function name for cold start metrics

When emitting cold start metrics, the `function_name` dimension defaults to `context.function_name`. If you want to change the value you can set the `function_name` parameter in the metrics constructor, or define the environment variable `POWERTOOLS_METRICS_FUNCTION_NAME`.

The priority of the `function_name` dimension value is defined as:

1. `function_name` constructor option
1. `POWERTOOLS_METRICS_FUNCTION_NAME` environment variable
1. `context.function_name` property

```
from aws_lambda_powertools import Metrics
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = Metrics(function_name="my-function-name")


@metrics.log_metrics(capture_cold_start_metric=True)
def lambda_handler(event: dict, context: LambdaContext): ...

```

### Environment variables

The following environment variable is available to configure Metrics at a global scope:

| Setting | Description | Environment variable | Default | | --- | --- | --- | --- | | **Namespace Name** | Sets **namespace** used for metrics. | `POWERTOOLS_METRICS_NAMESPACE` | `None` | | **Service** | Sets **service** metric dimension across all metrics e.g. `payment` | `POWERTOOLS_SERVICE_NAME` | `None` | | **Function Name** | Function name used as dimension for the **ColdStart** metric. | `POWERTOOLS_METRICS_FUNCTION_NAME` | `None` | | **Disable Powertools Metrics** | **Disables** all metrics emitted by Powertools. | `POWERTOOLS_METRICS_DISABLED` | `None` |

`POWERTOOLS_METRICS_NAMESPACE` is also available on a per-instance basis with the `namespace` parameter, which will consequently override the environment variable value.

## Advanced

### Adding metadata

You can add high-cardinality data as part of your Metrics log with `add_metadata` method. This is useful when you want to search highly contextual information along with your metrics in your logs.

Info

**This will not be available during metrics visualization** - Use **dimensions** for this purpose

```
from uuid import uuid4

from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = Metrics()


@metrics.log_metrics
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_metric(name="SuccessfulBooking", unit=MetricUnit.Count, value=1)
    metrics.add_metadata(key="booking_id", value=f"{uuid4()}")

```

```
{
    "_aws": {
        "Timestamp": 1656688250155,
        "CloudWatchMetrics": [
            {
                "Namespace": "ServerlessAirline",
                "Dimensions": [
                    [
                        "service"
                    ]
                ],
                "Metrics": [
                    {
                        "Name": "SuccessfulBooking",
                        "Unit": "Count"
                    }
                ]
            }
        ]
    },
    "service": "booking",
    "booking_id": "00347014-341d-4b8e-8421-a89d3d588ab3",
    "SuccessfulBooking": [
        1.0
    ]
}

```

### Single metric

CloudWatch EMF uses the same dimensions and timestamp across all your metrics. Use `single_metric` if you have a metric that should have different dimensions or timestamp.

#### Working with different dimensions

Generally, using different dimensions would be an edge case since you [pay for unique metric](https://aws.amazon.com/cloudwatch/pricing).

Keep the following formula in mind: **unique metric = (metric_name + dimension_name + dimension_value)**

```
import os

from aws_lambda_powertools import single_metric
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

STAGE = os.getenv("STAGE", "dev")


def lambda_handler(event: dict, context: LambdaContext):
    with single_metric(name="MySingleMetric", unit=MetricUnit.Count, value=1) as metric:
        metric.add_dimension(name="environment", value=STAGE)

```

```
{
    "_aws": {
        "Timestamp": 1656689267834,
        "CloudWatchMetrics": [
            {
                "Namespace": "ServerlessAirline",
                "Dimensions": [
                    [
                        "environment",
                        "service"
                    ]
                ],
                "Metrics": [
                    {
                        "Name": "MySingleMetric",
                        "Unit": "Count"
                    }
                ]
            }
        ]
    },
    "environment": "dev",
    "service": "booking",
    "MySingleMetric": [
        1.0
    ]
}

```

By default it will skip all previously defined dimensions including default dimensions. Use `default_dimensions` keyword argument if you want to reuse default dimensions or specify custom dimensions from a dictionary.

```
import os

from aws_lambda_powertools import single_metric
from aws_lambda_powertools.metrics import Metrics, MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

STAGE = os.getenv("STAGE", "dev")

metrics = Metrics()
metrics.set_default_dimensions(environment=STAGE)


def lambda_handler(event: dict, context: LambdaContext):
    with single_metric(
        name="RecordsCount",
        unit=MetricUnit.Count,
        value=10,
        default_dimensions=metrics.default_dimensions,
    ) as metric:
        metric.add_dimension(name="TableName", value="Users")

```

```
import os

from aws_lambda_powertools import single_metric
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

STAGE = os.getenv("STAGE", "dev")


def lambda_handler(event: dict, context: LambdaContext):
    with single_metric(
        name="RecordsCount",
        unit=MetricUnit.Count,
        value=10,
        default_dimensions={"environment": STAGE},
    ) as metric:
        metric.add_dimension(name="TableName", value="Users")

```

#### Working with different timestamp

When working with multiple metrics, customers may need different timestamps between them. In such cases, utilize `single_metric` to flush individual metrics with specific timestamps.

```
from aws_lambda_powertools import Logger, single_metric
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

logger = Logger()


def lambda_handler(event: dict, context: LambdaContext):
    for record in event:
        record_id: str = record.get("record_id")
        amount: int = record.get("amount")
        timestamp: int = record.get("timestamp")

        with single_metric(name="Orders", unit=MetricUnit.Count, value=amount, namespace="Powertools") as metric:
            logger.info(f"Processing record id {record_id}")
            metric.set_timestamp(timestamp)

```

```
[
    {
        "record_id": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
        "amount": 10,
        "timestamp": 1648195200000
    },
    {
        "record_id": "6ba7b811-9dad-11d1-80b4-00c04fd430c8",
        "amount": 30,
        "timestamp": 1648224000000
    },
    {
        "record_id": "6ba7b812-9dad-11d1-80b4-00c04fd430c8",
        "amount": 25,
        "timestamp": 1648209600000
    },
    {
        "record_id": "6ba7b813-9dad-11d1-80b4-00c04fd430c8",
        "amount": 40,
        "timestamp": 1648177200000
    },
    {
        "record_id": "6ba7b814-9dad-11d1-80b4-00c04fd430c8",
        "amount": 32,
        "timestamp": 1648216800000
    }
]

```

### Flushing metrics manually

If you are using the [AWS Lambda Web Adapter](https://github.com/awslabs/aws-lambda-web-adapter) project, or a middleware with custom metric logic, you can use `flush_metrics()`. This method will serialize, print metrics available to standard output, and clear in-memory metrics data.

Warning

This does not capture Cold Start metrics, and metric data validation still applies.

Contrary to the `log_metrics` decorator, you are now also responsible to flush metrics in the event of an exception.

```
from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = Metrics()


def book_flight(flight_id: str, **kwargs):
    # logic to book flight
    ...
    metrics.add_metric(name="SuccessfulBooking", unit=MetricUnit.Count, value=1)


def lambda_handler(event: dict, context: LambdaContext):
    try:
        book_flight(flight_id=event.get("flight_id", ""))
    finally:
        metrics.flush_metrics()

```

### Metrics isolation

You can use `EphemeralMetrics` class when looking to isolate multiple instances of metrics with distinct namespaces and/or dimensions.

This is a typical use case is for multi-tenant, or emitting same metrics for distinct applications.

```
from aws_lambda_powertools.metrics import EphemeralMetrics, MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = EphemeralMetrics()


@metrics.log_metrics
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_metric(name="SuccessfulBooking", unit=MetricUnit.Count, value=1)

```

**Differences between `EphemeralMetrics` and `Metrics`**

`EphemeralMetrics` has only one difference while keeping nearly the exact same set of features:

| Feature | Metrics | EphemeralMetrics | | --- | --- | --- | | **Share data across instances** (metrics, dimensions, metadata, etc.) | Yes | - |

Why not changing the default `Metrics` behaviour to not share data across instances?

This is an intentional design to prevent accidental data deduplication or data loss issues due to [CloudWatch EMF](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Embedded_Metric_Format_Specification.html) metric dimension constraint.

In CloudWatch, there are two metric ingestion mechanisms: [EMF (async)](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Embedded_Metric_Format_Specification.html) and [`PutMetricData` API (sync)](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/cloudwatch.html#CloudWatch.Client.put_metric_data).

The former creates metrics asynchronously via CloudWatch Logs, and the latter uses a synchronous and more flexible ingestion API.

Key concept

CloudWatch [considers a metric unique](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_concepts.html#Metric) by a combination of metric **name**, metric **namespace**, and zero or more metric **dimensions**.

With EMF, metric dimensions are shared with any metrics you define. With `PutMetricData` API, you can set a [list](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_MetricDatum.html) defining one or more metrics with distinct dimensions.

This is a subtle yet important distinction. Imagine you had the following metrics to emit:

| Metric Name | Dimension | Intent | | --- | --- | --- | | **SuccessfulBooking** | service="booking", **tenant_id**="sample" | Application metric | | **IntegrationLatency** | service="booking", function_name="sample" | Operational metric | | **ColdStart** | service="booking", function_name="sample" | Operational metric |

The `tenant_id` dimension could vary leading to two common issues:

1. `ColdStart` metric will be created multiple times (N * number of unique tenant_id dimension value), despite the `function_name` being the same
1. `IntegrationLatency` metric will be also created multiple times due to `tenant_id` as well as `function_name` (may or not be intentional)

These issues are exacerbated when you create **(A)** metric dimensions conditionally, **(B)** multiple metrics' instances throughout your code instead of reusing them (globals). Subsequent metrics' instances will have (or lack) different metric dimensions resulting in different metrics and data points with the same name.

Intentional design to address these scenarios

**On 1**, when you enable [capture_start_metric feature](#capturing-cold-start-metric), we transparently create and flush an additional EMF JSON Blob that is independent from your application metrics. This prevents data pollution.

**On 2**, you can use `EphemeralMetrics` to create an additional EMF JSON Blob from your application metric (`SuccessfulBooking`). This ensures that `IntegrationLatency` operational metric data points aren't tied to any dynamic dimension values like `tenant_id`.

That is why `Metrics` shares data across instances by default, as that covers 80% of use cases and different personas using Powertools. This allows them to instantiate `Metrics` in multiple places throughout their code - be a separate file, a middleware, or an abstraction that sets default dimensions.

### Observability providers

> An observability provider is an [AWS Lambda Partner](https://docs.aws.amazon.com/lambda/latest/dg/extensions-api-partners.html) offering a platform for logging, metrics, traces, etc.

We provide a thin-wrapper on top of the most requested observability providers. We strive to keep a similar UX as close as possible while keeping our value add features.

Missing your preferred provider? Please create a [feature request](https://github.com/aws-powertools/powertools-lambda-python/issues/new?assignees=&labels=feature-request%2Ctriage&projects=&template=feature_request.yml&title=Feature+request%3A+TITLE).

Current providers:

| Provider | Notes | | --- | --- | | [Datadog](./datadog) | Uses Datadog SDK and Datadog Lambda Extension by default |

## Testing your code

### Setting environment variables

Tip

Ignore this section, if:

- You are explicitly setting namespace/default dimension via `namespace` and `service` parameters
- You're not instantiating `Metrics` in the global namespace

For example, `Metrics(namespace="ServerlessAirline", service="booking")`

Make sure to set `POWERTOOLS_METRICS_NAMESPACE` and `POWERTOOLS_SERVICE_NAME` before running your tests to prevent failing on `SchemaValidation` exception. You can set it before you run tests or via pytest plugins like [dotenv](https://pypi.org/project/pytest-dotenv/).

```
POWERTOOLS_SERVICE_NAME="booking" POWERTOOLS_METRICS_NAMESPACE="ServerlessAirline" python -m pytest

```

### Clearing metrics

`Metrics` keep metrics in memory across multiple instances. If you need to test this behavior, you can use the following Pytest fixture to ensure metrics are reset incl. cold start:

```
import pytest

from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics.provider import cold_start


@pytest.fixture(scope="function", autouse=True)
def reset_metric_set():
    # Clear out every metric data prior to every test
    metrics = Metrics()
    metrics.clear_metrics()
    cold_start.is_cold_start = True  # ensure each test has cold start
    metrics.clear_default_dimensions()  # remove persisted default dimensions, if any
    yield

```

### Functional testing

You can read standard output and assert whether metrics have been flushed. Here's an example using `pytest` with `capsys` built-in fixture:

```
import json

import add_metrics


def test_log_metrics(capsys):
    add_metrics.lambda_handler({}, {})

    log = capsys.readouterr().out.strip()  # remove any extra line
    metrics_output = json.loads(log)  # deserialize JSON str

    # THEN we should have no exceptions
    # and a valid EMF object should be flushed correctly
    assert "SuccessfulBooking" in log  # basic string assertion in JSON str
    assert "SuccessfulBooking" in metrics_output["_aws"]["CloudWatchMetrics"][0]["Metrics"][0]["Name"]

```

```
from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = Metrics()


@metrics.log_metrics  # ensures metrics are flushed upon request completion/failure
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_metric(name="SuccessfulBooking", unit=MetricUnit.Count, value=1)

```

This will be needed when using `capture_cold_start_metric=True`, or when both `Metrics` and `single_metric` are used.

```
import json
from dataclasses import dataclass

import assert_multiple_emf_blobs_module
import pytest


@dataclass
class LambdaContext:
    function_name: str = "test"
    memory_limit_in_mb: int = 128
    invoked_function_arn: str = "arn:aws:lambda:eu-west-1:809313241:function:test"
    aws_request_id: str = "52fdfc07-2182-154f-163f-5f0f9a621d72"


@pytest.fixture
def lambda_context() -> LambdaContext:
    return LambdaContext()


def capture_metrics_output_multiple_emf_objects(capsys):
    return [json.loads(line.strip()) for line in capsys.readouterr().out.split("\n") if line]


def test_log_metrics(capsys, lambda_context: LambdaContext):
    assert_multiple_emf_blobs_module.lambda_handler({}, lambda_context)

    cold_start_blob, custom_metrics_blob = capture_metrics_output_multiple_emf_objects(capsys)

    # Since `capture_cold_start_metric` is used
    # we should have one JSON blob for cold start metric and one for the application
    assert cold_start_blob["ColdStart"] == [1.0]
    assert cold_start_blob["function_name"] == "test"

    assert "SuccessfulBooking" in custom_metrics_blob

```

```
from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

metrics = Metrics()


@metrics.log_metrics(capture_cold_start_metric=True)
def lambda_handler(event: dict, context: LambdaContext):
    metrics.add_metric(name="SuccessfulBooking", unit=MetricUnit.Count, value=1)

```

Tip

For more elaborate assertions and comparisons, check out [our functional testing for Metrics utility.](https://github.com/aws-powertools/powertools-lambda-python/blob/develop/tests/functional/metrics/required_dependencies/test_metrics_cloudwatch_emf.py)
