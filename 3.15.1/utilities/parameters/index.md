The parameters utility provides high-level functions to retrieve one or multiple parameter values from [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html), [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/), [AWS AppConfig](https://docs.aws.amazon.com/appconfig/latest/userguide/what-is-appconfig.html), [Amazon DynamoDB](https://aws.amazon.com/dynamodb/), or bring your own.

## Key features

- Retrieve one or multiple parameters from the underlying provider
- Cache parameter values for a given amount of time (defaults to 5 minutes)
- Transform parameter values from JSON or base 64 encoded strings
- Bring Your Own Parameter Store Provider

## Getting started

Tip

All examples shared in this documentation are available within the [project repository](https://github.com/aws-powertools/powertools-lambda-python/tree/develop/examples).

By default, we fetch parameters from System Manager Parameter Store, secrets from Secrets Manager, and application configuration from AppConfig.

### IAM Permissions

This utility requires additional permissions to work as expected.

Note

Different parameter providers require different permissions.

| Provider | Function/Method | IAM Permission | | --- | --- | --- | | SSM | **`get_parameter`**, **`SSMProvider.get`** | **`ssm:GetParameter`** | | SSM | **`get_parameters`**, **`SSMProvider.get_multiple`** | **`ssm:GetParametersByPath`** | | SSM | **`get_parameters_by_name`**, **`SSMProvider.get_parameters_by_name`** | **`ssm:GetParameter`** and **`ssm:GetParameters`** | | SSM | **`set_parameter`**, **`SSMProvider.set_parameter`** | **`ssm:PutParameter`** | | SSM | If using **`decrypt=True`** | You must add an additional permission **`kms:Decrypt`** | | Secrets | **`get_secret`**, **`SecretsProvider.get`** | **`secretsmanager:GetSecretValue`** | | Secrets | **`set_secret`**, **`SecretsProvider.set`** | **`secretsmanager:PutSecretValue`** and **`secretsmanager:CreateSecret`** (if creating secrets) | | DynamoDB | **`DynamoDBProvider.get`** | **`dynamodb:GetItem`** | | DynamoDB | **`DynamoDBProvider.get_multiple`** | **`dynamodb:Query`** | | AppConfig | **`get_app_config`**, **`AppConfigProvider.get_app_config`** | **`appconfig:GetLatestConfiguration`** and **`appconfig:StartConfigurationSession`** |

### Fetching parameters

You can retrieve a single parameter using the `get_parameter` high-level function.

```
import requests

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: dict, context: LambdaContext) -> dict:
    try:
        # Retrieve a single parameter
        endpoint_comments = parameters.get_parameter("/lambda-powertools/endpoint_comments")

        # the value of this parameter is https://jsonplaceholder.typicode.com/comments/
        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()[:10], "statusCode": 200}
    except parameters.exceptions.GetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

For multiple parameters, you can use either:

- `get_parameters` to recursively fetch all parameters by path.
- `get_parameters_by_name` to fetch distinct parameters by their full name. It also accepts custom caching, transform, decrypt per parameter.

This is useful when you want to fetch all parameters from a given path, say `/dev`, e.g., `/dev/config`, `/dev/webhook/config`

To ease readability in deeply nested paths, we strip the path name. For example:

- `/dev/config` -> `config`
- `/dev/webhook/config` -> `webhook/config`

```
import requests

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Retrieve all parameters within a path e.g., /dev
        # Say, you had two parameters under `/dev`: /dev/config, /dev/webhook/config
        all_parameters: dict = parameters.get_parameters("/dev", max_age=20)
        endpoint_comments = None

        # We strip the path prefix name for readability and memory usage in deeply nested paths
        # all_parameters would then look like:
        ## all_parameters["config"] = value # noqa: ERA001
        ## all_parameters["webhook/config"] = value # noqa: ERA001
        for parameter, value in all_parameters.items():
            if parameter == "endpoint_comments":
                endpoint_comments = value

        if endpoint_comments is None:
            return {"comments": None}

        # the value of parameter is https://jsonplaceholder.typicode.com/comments/
        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()[:10]}
    except parameters.exceptions.GetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

```
from __future__ import annotations

from typing import Any

from aws_lambda_powertools.utilities.parameters.ssm import get_parameters_by_name

parameters = {
    "/develop/service/commons/telemetry/config": {"max_age": 300, "transform": "json"},
    "/no_cache_param": {"max_age": 0},
    # inherit default values
    "/develop/service/payment/api/capture/url": {},
}


def handler(event, context):
    # This returns a dict with the parameter name as key
    response: dict[str, Any] = get_parameters_by_name(parameters=parameters, max_age=60)
    for parameter, value in response.items():
        print(f"{parameter}: {value}")

```

Failing gracefully if one or more parameters cannot be fetched or decrypted.

By default, we will raise `GetParameterError` when any parameter fails to be fetched.

You can override it by setting `raise_on_error=False`. When disabled, we take the following actions:

- Add failed parameter name in the `_errors` key, *e.g.*, `{_errors: ["/param1", "/param2"]}`
- Keep only successful parameter names and their values in the response
- Raise `GetParameterError` if any of your parameters is named `_errors`

```
from __future__ import annotations

from typing import Any

from aws_lambda_powertools.utilities.parameters.ssm import get_parameters_by_name

parameters = {
    "/develop/service/commons/telemetry/config": {"max_age": 300, "transform": "json"},
    # it would fail by default
    "/this/param/does/not/exist": {},
}


def handler(event, context):
    values: dict[str, Any] = get_parameters_by_name(parameters=parameters, raise_on_error=False)
    errors: list[str] = values.get("_errors", [])

    # Handle gracefully, since '/this/param/does/not/exist' will only be available in `_errors`
    if errors:
        ...

    for parameter, value in values.items():
        print(f"{parameter}: {value}")

```

### Setting parameters

You can set a parameter using the `set_parameter` high-level function. This will create a new parameter if it doesn't exist.

```
from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: dict, context: LambdaContext) -> dict:
    try:
        # Set a single parameter, returns the version ID of the parameter
        parameter_version = parameters.set_parameter(name="/mySuper/Parameter", value="PowerToolsIsAwesome")

        return {"mySuperParameterVersion": parameter_version, "statusCode": 200}
    except parameters.exceptions.SetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

Sometimes you may be setting a parameter that you will have to update later on. Use the `overwrite` option to overwrite any existing value. If you do not set this option, the parameter value will not be overwritten and an exception will be raised.

```
from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: dict, context: LambdaContext) -> dict:
    try:
        # Set a single parameter, but overwrite if it already exists.
        # Overwrite is False by default, so we explicitly set it to True
        updating_parameter = parameters.set_parameter(
            name="/mySuper/Parameter",
            value="PowerToolsIsAwesome",
            overwrite=True,
        )

        return {"mySuperParameterVersion": updating_parameter, "statusCode": 200}
    except parameters.exceptions.SetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

### Fetching secrets

You can fetch secrets stored in Secrets Manager using `get_secret`.

```
from typing import Any

import requests

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Usually an endpoint is not sensitive data, so we store it in SSM Parameters
        endpoint_comments: Any = parameters.get_parameter("/lambda-powertools/endpoint_comments")
        # An API-KEY is a sensitive data and should be stored in SecretsManager
        api_key: Any = parameters.get_secret("/lambda-powertools/api-key")

        headers: dict = {"X-API-Key": api_key}

        comments: requests.Response = requests.get(endpoint_comments, headers=headers)

        return {"comments": comments.json()[:10], "statusCode": 200}
    except parameters.exceptions.GetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

### Setting secrets

You can set secrets stored in Secrets Manager using `set_secret`.

Note

We strive to minimize API calls by attempting to update existing secrets as our primary approach. If a secret doesn't exist, we proceed to create a new one.

```
from typing import Any

from aws_lambda_powertools import Logger
from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext

logger = Logger(serialize_stacktrace=True)


def access_token(client_id: str, client_secret: str, audience: str) -> str:
    # example function that returns a JWT Access Token
    # add your own logic here
    return f"{client_id}.{client_secret}.{audience}"


def lambda_handler(event: dict, context: LambdaContext):
    try:
        client_id: Any = parameters.get_parameter("/aws-powertools/client_id")
        client_secret: Any = parameters.get_parameter("/aws-powertools/client_secret")
        audience: Any = parameters.get_parameter("/aws-powertools/audience")

        jwt_token = access_token(client_id=client_id, client_secret=client_secret, audience=audience)

        # set-secret will create a new secret if it doesn't exist and return the version id
        update_secret_version_id = parameters.set_secret(name="/aws-powertools/jwt_token", value=jwt_token)

        return {"access_token": "updated", "statusCode": 200, "update_secret_version_id": update_secret_version_id}
    except parameters.exceptions.SetSecretError as error:
        logger.exception(error)
        return {"access_token": "updated", "statusCode": 400}

```

### Fetching app configurations

You can fetch application configurations in AWS AppConfig using `get_app_config`.

The following will retrieve the latest version and store it in the cache.

Warning

We make two API calls to fetch each unique configuration name during the first time. This is by design in AppConfig. Please consider adjusting `max_age` parameter to enhance performance.

```
from typing import Any

import requests

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Retrieve a single parameter
        endpoint_comments: Any = parameters.get_app_config(name="config", environment="dev", application="comments")

        # the value of this parameter is https://jsonplaceholder.typicode.com/comments/
        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()[:10], "statusCode": 200}
    except parameters.exceptions.GetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

### Environment variables

The following environment variables are available to configure the parameter utility at a global scope:

| Setting | Description | Environment variable | Default | | --- | --- | --- | --- | | **Max Age** | Adjusts for how long values are kept in cache (in seconds). | `POWERTOOLS_PARAMETERS_MAX_AGE` | `300` | | **Debug Sample Rate** | Sets whether to decrypt or not values retrieved from AWS SSM Parameters Store. | `POWERTOOLS_PARAMETERS_SSM_DECRYPT` | `false` |

You can also use [`POWERTOOLS_PARAMETERS_MAX_AGE`](#adjusting-cache-ttl) through the `max_age` parameter and [`POWERTOOLS_PARAMETERS_SSM_DECRYPT`](#ssmprovider) through the `decrypt` parameter to override the environment variable values.

## Advanced

### Adjusting cache TTL

Tip

`max_age` parameter is also available in underlying provider functions like `get()`, `get_multiple()`, etc.

By default, we cache parameters retrieved in-memory for 300 seconds (5 minutes). If you want to change this default value and set the same TTL for all parameters, you can set the `POWERTOOLS_PARAMETERS_MAX_AGE` environment variable. **You can still set `max_age` for individual parameters**.

You can adjust how long we should keep values in cache by using the param `max_age`, when using `get_parameter()`, `get_parameters()` and `get_secret()` methods across all providers.

```
from typing import Any

import requests

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Retrieve a single parameter with 20s cache
        endpoint_comments: Any = parameters.get_parameter("/lambda-powertools/endpoint_comments", max_age=20)

        # the value of this parameter is https://jsonplaceholder.typicode.com/comments/
        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()[:10], "statusCode": 200}
    except parameters.exceptions.GetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

```
from typing import Any

import requests

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Retrieve multiple parameters from a path prefix
        all_parameters: Any = parameters.get_parameters("/lambda-powertools/", max_age=20)
        endpoint_comments = "https://jsonplaceholder.typicode.com/noexists/"

        for parameter, value in all_parameters.items():
            if parameter == "endpoint_comments":
                endpoint_comments = value

        # the value of parameter is https://jsonplaceholder.typicode.com/comments/
        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()[:10]}
    except parameters.exceptions.GetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

```
from typing import Any

import requests

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Usually an endpoint is not sensitive data, so we store it in SSM Parameters
        endpoint_comments: Any = parameters.get_parameter("/lambda-powertools/endpoint_comments")
        # An API-KEY is a sensitive data and should be stored in SecretsManager
        api_key: Any = parameters.get_secret("/lambda-powertools/api-key", max_age=20)

        headers: dict = {"X-API-Key": api_key}

        comments: requests.Response = requests.get(endpoint_comments, headers=headers)

        return {"comments": comments.json()[:10], "statusCode": 200}
    except parameters.exceptions.GetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

```
from typing import Any

import requests

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Retrieve a single parameter
        endpoint_comments: Any = parameters.get_app_config(
            name="config",
            environment="dev",
            application="comments",
            max_age=20,
        )

        # the value of this parameter is https://jsonplaceholder.typicode.com/comments/
        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()[:10], "statusCode": 200}
    except parameters.exceptions.GetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

### Always fetching the latest

If you'd like to always ensure you fetch the latest parameter from the store regardless if already available in cache, use `force_fetch` param.

```
from typing import Any

import requests

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Retrieve a single parameter with 20s cache
        endpoint_comments: Any = parameters.get_parameter("/lambda-powertools/endpoint_comments", force_fetch=True)

        # the value of this parameter is https://jsonplaceholder.typicode.com/comments/
        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()[:10], "statusCode": 200}
    except parameters.exceptions.GetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

```
from typing import Any

import requests

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Retrieve multiple parameters from a path prefix
        all_parameters: Any = parameters.get_parameters("/lambda-powertools/", force_fetch=True)
        endpoint_comments = "https://jsonplaceholder.typicode.com/noexists/"

        for parameter, value in all_parameters.items():
            if parameter == "endpoint_comments":
                endpoint_comments = value

        # the value of parameter is https://jsonplaceholder.typicode.com/comments/
        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()[:10]}
    except parameters.exceptions.GetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

```
from typing import Any

import requests

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Usually an endpoint is not sensitive data, so we store it in SSM Parameters
        endpoint_comments: Any = parameters.get_parameter("/lambda-powertools/endpoint_comments")
        # An API-KEY is a sensitive data and should be stored in SecretsManager
        api_key: Any = parameters.get_secret("/lambda-powertools/api-key", force_fetch=True)

        headers: dict = {"X-API-Key": api_key}

        comments: requests.Response = requests.get(endpoint_comments, headers=headers)

        return {"comments": comments.json()[:10], "statusCode": 200}
    except parameters.exceptions.GetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

```
from typing import Any

import requests

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Retrieve a single parameter
        endpoint_comments: Any = parameters.get_app_config(
            name="config",
            environment="dev",
            application="comments",
            force_fetch=True,
        )

        # the value of this parameter is https://jsonplaceholder.typicode.com/comments/
        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()[:10], "statusCode": 200}
    except parameters.exceptions.GetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

### Built-in provider class

For greater flexibility such as configuring the underlying SDK client used by built-in providers, you can use their respective Provider Classes directly.

Tip

This is useful when you need to customize parameters for the SDK client, such as region, credentials, retries and others. For more information, read [botocore.config](https://botocore.amazonaws.com/v1/documentation/api/latest/reference/config.html) and [boto3.session](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/core/session.html#module-boto3.session).

#### SSMProvider

```
from typing import Any

import requests
from botocore.config import Config

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext

# changing region_name, connect_timeout and retrie configurations
# see: https://botocore.amazonaws.com/v1/documentation/api/latest/reference/config.html
config = Config(region_name="sa-east-1", connect_timeout=1, retries={"total_max_attempts": 2, "max_attempts": 5})
ssm_provider = parameters.SSMProvider(config=config)


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Retrieve a single parameter
        endpoint_comments: Any = ssm_provider.get("/lambda-powertools/endpoint_comments")

        # the value of this parameter is https://jsonplaceholder.typicode.com/comments/
        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()[:10], "statusCode": 200}
    except parameters.exceptions.GetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

```
from typing import Any

import boto3
import requests

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext

# assuming role from another account to get parameter there
# see: https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html
sts_client = boto3.client("sts")
assumed_role_object = sts_client.assume_role(
    RoleArn="arn:aws:iam::account-of-role-to-assume:role/name-of-role",
    RoleSessionName="RoleAssume1",
)
credentials = assumed_role_object["Credentials"]

# using temporary credentials in your SSMProvider provider
# see: https://boto3.amazonaws.com/v1/documentation/api/latest/reference/core/session.html#module-boto3.session
boto3_session = boto3.session.Session(
    region_name="us-east-1",
    aws_access_key_id=credentials["AccessKeyId"],
    aws_secret_access_key=credentials["SecretAccessKey"],
    aws_session_token=credentials["SessionToken"],
)
ssm_provider = parameters.SSMProvider(boto3_session=boto3_session)


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Retrieve multiple parameters from a path prefix
        all_parameters: Any = ssm_provider.get_multiple("/lambda-powertools/")
        endpoint_comments = "https://jsonplaceholder.typicode.com/noexists/"

        for parameter, value in all_parameters.items():
            if parameter == "endpoint_comments":
                endpoint_comments = value

        # the value of parameter is https://jsonplaceholder.typicode.com/comments/
        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()[:10]}
    except parameters.exceptions.GetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

The AWS Systems Manager Parameter Store provider supports two additional arguments for the `get()` and `get_multiple()` methods:

| Parameter | Default | Description | | --- | --- | --- | | **decrypt** | `False` | Will automatically decrypt the parameter. | | **recursive** | `True` | For `get_multiple()` only, will fetch all parameter values recursively based on a path prefix. |

You can create `SecureString` parameters, which are parameters that have a plaintext parameter name and an encrypted parameter value. If you don't use the `decrypt` argument, you will get an encrypted value. Read [here](https://docs.aws.amazon.com/kms/latest/developerguide/services-parameter-store.html) about best practices using KMS to secure your parameters.

Tip

If you want to always decrypt parameters, you can set the `POWERTOOLS_PARAMETERS_SSM_DECRYPT=true` environment variable. **This will override the default value of `false` but you can still set the `decrypt` option for individual parameters**.

```
from typing import Any
from uuid import uuid4

import boto3

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext

ec2 = boto3.resource("ec2")
ssm_provider = parameters.SSMProvider()


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Retrieve the key pair from secure string parameter
        ec2_pem: Any = ssm_provider.get("/lambda-powertools/ec2_pem", decrypt=True)

        name_key_pair = f"kp_{uuid4()}"

        ec2.import_key_pair(KeyName=name_key_pair, PublicKeyMaterial=ec2_pem)

        ec2.create_instances(
            ImageId="ami-026b57f3c383c2eec",
            InstanceType="t2.micro",
            MinCount=1,
            MaxCount=1,
            KeyName=name_key_pair,
        )

        return {"message": "EC2 created", "success": True}
    except parameters.exceptions.GetParameterError as error:
        return {"message": f"Error creating EC2 => {str(error)}", "success": False}

```

```
from typing import Any

import requests

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext

ssm_provider = parameters.SSMProvider()


class ConfigNotFound(Exception): ...


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Retrieve multiple parameters from a path prefix
        # /config = root
        # /config/endpoint = url
        # /config/endpoint/query = querystring
        all_parameters: Any = ssm_provider.get_multiple("/config", recursive=False)
        endpoint_comments = "https://jsonplaceholder.typicode.com/comments/"

        for parameter, value in all_parameters.items():
            # query parameter is used to query endpoint
            if "query" in parameter:
                endpoint_comments = f"{endpoint_comments}{value}"
                break
        else:
            # scheme config was not found because get_multiple is not recursive
            raise ConfigNotFound("URL query parameter was not found")

        # the value of parameter is https://jsonplaceholder.typicode.com/comments/
        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()}
    except parameters.exceptions.GetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

#### SecretsProvider

```
from typing import Any

import requests
from botocore.config import Config

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext

config = Config(region_name="sa-east-1", connect_timeout=1, retries={"total_max_attempts": 2, "max_attempts": 5})
ssm_provider = parameters.SecretsProvider(config=config)


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Usually an endpoint is not sensitive data, so we store it in SSM Parameters
        endpoint_comments: Any = parameters.get_parameter("/lambda-powertools/endpoint_comments")
        # An API-KEY is a sensitive data and should be stored in SecretsManager
        api_key: Any = ssm_provider.get("/lambda-powertools/api-key")

        headers: dict = {"X-API-Key": api_key}

        comments: requests.Response = requests.get(endpoint_comments, headers=headers)

        return {"comments": comments.json()[:10], "statusCode": 200}
    except parameters.exceptions.GetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

#### DynamoDBProvider

The DynamoDB Provider does not have any high-level functions, as it needs to know the name of the DynamoDB table containing the parameters.

**DynamoDB table structure for single parameters**

For single parameters, you must use `id` as the [partition key](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html#HowItWorks.CoreComponents.PrimaryKey) for that table.

Example

DynamoDB table with `id` partition key and `value` as attribute

| id | value | | --- | --- | | my-parameter | my-value |

With this table, `dynamodb_provider.get("my-parameter")` will return `my-value`.

```
from typing import Any

import requests

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext

dynamodb_provider = parameters.DynamoDBProvider(table_name="ParameterTable")


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Usually an endpoint is not sensitive data, so we store it in DynamoDB Table
        endpoint_comments: Any = dynamodb_provider.get("comments_endpoint")

        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()[:10], "statusCode": 200}
    # general exception
    except Exception as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: 'DynamoDB Table example'
Resources:
  ParameterTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: ParameterTable
      AttributeDefinitions:
        -   AttributeName: id
            AttributeType: S
      KeySchema:
        -   AttributeName: id
            KeyType: HASH
      TimeToLiveSpecification:
        AttributeName: expiration
        Enabled: true
      BillingMode: PAY_PER_REQUEST

```

You can initialize the DynamoDB provider pointing to [DynamoDB Local](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.html) using `endpoint_url` parameter:

```
from typing import Any

import requests

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext

dynamodb_provider = parameters.DynamoDBProvider(table_name="ParameterTable", endpoint_url="http://localhost:8000")


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Usually an endpoint is not sensitive data, so we store it in DynamoDB Table
        endpoint_comments: Any = dynamodb_provider.get("comments_endpoint")

        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()[:10], "statusCode": 200}
    # general exception
    except Exception as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

**DynamoDB table structure for multiple values parameters**

You can retrieve multiple parameters sharing the same `id` by having a sort key named `sk`.

Example

DynamoDB table with `id` primary key, `sk` as sort key and `value` as attribute

| id | sk | value | | --- | --- | --- | | config | endpoint_comments | <https://jsonplaceholder.typicode.com/comments/> | | config | limit | 10 |

With this table, `dynamodb_provider.get_multiple("config")` will return a dictionary response in the shape of `sk:value`.

```
from typing import Any

import requests

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext

dynamodb_provider = parameters.DynamoDBProvider(table_name="ParameterTable")


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Retrieve multiple parameters using HASH KEY
        all_parameters: Any = dynamodb_provider.get_multiple("config")
        endpoint_comments = "https://jsonplaceholder.typicode.com/noexists/"
        limit = 2

        for parameter, value in all_parameters.items():
            if parameter == "endpoint_comments":
                endpoint_comments = value

            if parameter == "limit":
                limit = int(value)

        # the value of parameter is https://jsonplaceholder.typicode.com/comments/
        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()[limit]}
    # general exception
    except Exception as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: 'DynamoDB Table example'
Resources:
  ParameterTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: ParameterTable
      AttributeDefinitions:
        -   AttributeName: id
            AttributeType: S
        -   AttributeName: sk
            AttributeType: S
      KeySchema:
        -   AttributeName: id
            KeyType: HASH
        -   AttributeName: sk
            KeyType: RANGE
      TimeToLiveSpecification:
        AttributeName: expiration
        Enabled: true
      BillingMode: PAY_PER_REQUEST

```

**Customizing DynamoDBProvider**

DynamoDB provider can be customized at initialization to match your table structure:

| Parameter | Mandatory | Default | Description | | --- | --- | --- | --- | | **table_name** | **Yes** | *(N/A)* | Name of the DynamoDB table containing the parameter values. | | **key_attr** | No | `id` | Hash key for the DynamoDB table. | | **sort_attr** | No | `sk` | Range key for the DynamoDB table. You don't need to set this if you don't use the `get_multiple()` method. | | **value_attr** | No | `value` | Name of the attribute containing the parameter value. |

```
from typing import Any

import requests

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext

dynamodb_provider = parameters.DynamoDBProvider(
    table_name="ParameterTable",
    key_attr="IdKeyAttr",
    sort_attr="SkKeyAttr",
    value_attr="ValueAttr",
)


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Usually an endpoint is not sensitive data, so we store it in DynamoDB Table
        endpoint_comments: Any = dynamodb_provider.get("comments_endpoint")

        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()[:10], "statusCode": 200}
    # general exception
    except Exception as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: 'DynamoDB Table example'
Resources:
  ParameterTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: ParameterTable
      AttributeDefinitions:
        -   AttributeName: IdKeyAttr
            AttributeType: S
        -   AttributeName: SkKeyAttr
            AttributeType: S
      KeySchema:
        -   AttributeName: IdKeyAttr
            KeyType: HASH
        -   AttributeName: SkKeyAttr
            KeyType: RANGE
      TimeToLiveSpecification:
        AttributeName: expiration
        Enabled: true
      BillingMode: PAY_PER_REQUEST

```

#### AppConfigProvider

```
from typing import Any

import requests
from botocore.config import Config

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext

config = Config(region_name="sa-east-1")
appconf_provider = parameters.AppConfigProvider(environment="dev", application="comments", config=config)


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Retrieve a single parameter
        endpoint_comments: Any = appconf_provider.get("config")

        # the value of this parameter is https://jsonplaceholder.typicode.com/comments/
        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()[:10], "statusCode": 200}
    except parameters.exceptions.GetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

### Create your own provider

You can create your own custom parameter store provider by inheriting the `BaseProvider` class, and implementing both `_get()` and `_get_multiple()` methods to retrieve a single, or multiple parameters from your custom store.

All transformation and caching logic is handled by the `get()` and `get_multiple()` methods from the base provider class.

Here are two examples of implementing a custom parameter store. One using an external service like [Hashicorp Vault](https://www.vaultproject.io/), a widely popular key-value and secret storage and the other one using [Amazon S3](https://aws.amazon.com/s3/?nc1=h_ls), a popular object storage.

```
from typing import Any

import hvac
import requests
from custom_provider_vault import VaultProvider

from aws_lambda_powertools import Logger
from aws_lambda_powertools.utilities.typing import LambdaContext

logger = Logger()

# In production you must use Vault over HTTPS and certificates.
vault_provider = VaultProvider(vault_url="http://192.168.68.105:8200/", vault_token="YOUR_TOKEN")


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Retrieve a single parameter
        endpoint_comments: Any = vault_provider.get("comments_endpoint")

        # you can get all parameters using get_multiple and specifying vault mount point
        # # for testing purposes we will not use it
        all_parameters: Any = vault_provider.get_multiple("/")
        logger.info(all_parameters)

        # the value of this parameter is https://jsonplaceholder.typicode.com/comments/
        comments: requests.Response = requests.get(endpoint_comments["url"])

        return {"comments": comments.json()[:10], "statusCode": 200}
    except hvac.exceptions.InvalidPath as error:
        return {"comments": None, "message": str(error), "statusCode": 400}
    # general exception
    except Exception as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

```
from typing import Any, Dict

from hvac import Client

from aws_lambda_powertools.utilities.parameters import BaseProvider


class VaultProvider(BaseProvider):
    def __init__(self, vault_url: str, vault_token: str) -> None:
        super().__init__()

        self.vault_client = Client(url=vault_url, verify=False, timeout=10)
        self.vault_client.token = vault_token

    def _get(self, name: str, **sdk_options) -> Dict[str, Any]:
        # for example proposal, the mountpoint is always /secret
        kv_configuration = self.vault_client.secrets.kv.v2.read_secret(path=name)

        return kv_configuration["data"]["data"]

    def _get_multiple(self, path: str, **sdk_options) -> Dict[str, str]:
        list_secrets = {}
        all_secrets = self.vault_client.secrets.kv.v2.list_secrets(path=path)

        # for example proposal, the mountpoint is always /secret
        for secret in all_secrets["data"]["keys"]:
            kv_configuration = self.vault_client.secrets.kv.v2.read_secret(path=secret)

            for key, value in kv_configuration["data"]["data"].items():
                list_secrets[key] = value

        return list_secrets

```

```
from typing import Any

import requests
from custom_provider_s3 import S3Provider

from aws_lambda_powertools import Logger
from aws_lambda_powertools.utilities.typing import LambdaContext

logger = Logger()

s3_provider = S3Provider(bucket_name="bucket_name")


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Retrieve a single parameter using key
        endpoint_comments: Any = s3_provider.get("comments_endpoint")
        # you can get all parameters using get_multiple and specifying a bucket prefix
        # # for testing purposes we will not use it
        all_parameters: Any = s3_provider.get_multiple("/")
        logger.info(all_parameters)

        # the value of this parameter is https://jsonplaceholder.typicode.com/comments/
        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()[:10], "statusCode": 200}
    # general exception
    except Exception as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

```
import copy
from typing import Dict

import boto3

from aws_lambda_powertools.utilities.parameters import BaseProvider


class S3Provider(BaseProvider):
    def __init__(self, bucket_name: str):
        # Initialize the client to your custom parameter store
        # E.g.:

        super().__init__()

        self.bucket_name = bucket_name
        self.client = boto3.client("s3")

    def _get(self, name: str, **sdk_options) -> str:
        # Retrieve a single value
        # E.g.:

        sdk_options["Bucket"] = self.bucket_name
        sdk_options["Key"] = name

        response = self.client.get_object(**sdk_options)
        return response["Body"].read().decode()

    def _get_multiple(self, path: str, **sdk_options) -> Dict[str, str]:
        # Retrieve multiple values
        # E.g.:

        list_sdk_options = copy.deepcopy(sdk_options)

        list_sdk_options["Bucket"] = self.bucket_name
        list_sdk_options["Prefix"] = path

        list_response = self.client.list_objects_v2(**list_sdk_options)

        parameters = {}

        for obj in list_response.get("Contents", []):
            get_sdk_options = copy.deepcopy(sdk_options)

            get_sdk_options["Bucket"] = self.bucket_name
            get_sdk_options["Key"] = obj["Key"]

            get_response = self.client.get_object(**get_sdk_options)

            parameters[obj["Key"]] = get_response["Body"].read().decode()

        return parameters

```

### Deserializing values with transform parameter

For parameters stored in JSON or Base64 format, you can use the `transform` argument for deserialization.

Info

The `transform` argument is available across all providers, including the high level functions.

```
from typing import Any

import requests

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: dict, context: LambdaContext) -> dict:
    try:
        # Retrieve a single parameter
        endpoint_comments: Any = parameters.get_parameter("/lambda-powertools/endpoint_comments", transform="json")

        # the value of this parameter is https://jsonplaceholder.typicode.com/comments/
        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()[:10], "statusCode": 200}
    except parameters.exceptions.GetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

```
from typing import Any

import requests
from botocore.config import Config

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext

config = Config(region_name="sa-east-1")
appconf_provider = parameters.AppConfigProvider(environment="dev", application="comments", config=config)


def lambda_handler(event: dict, context: LambdaContext):
    try:
        # Retrieve a single parameter
        endpoint_comments: Any = appconf_provider.get("config", transform="json")

        # the value of this parameter is https://jsonplaceholder.typicode.com/comments/
        comments: requests.Response = requests.get(endpoint_comments)

        return {"comments": comments.json()[:10], "statusCode": 200}
    except parameters.exceptions.GetParameterError as error:
        return {"comments": None, "message": str(error), "statusCode": 400}

```

#### Partial transform failures with `get_multiple()`

If you use `transform` with `get_multiple()`, you can have a single malformed parameter value. To prevent failing the entire request, the method will return a `None` value for the parameters that failed to transform.

You can override this by setting the `raise_on_transform_error` argument to `True`. If you do so, a single transform error will raise a **`TransformParameterError`** exception.

For example, if you have three parameters, */param/a*, */param/b* and */param/c*, but */param/c* is malformed:

```
from typing import Any

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext

ssm_provider = parameters.SSMProvider()


def lambda_handler(event: dict, context: LambdaContext):
    # This will display:
    # /param/a: [some value]
    # /param/b: [some value]
    # /param/c: None
    values: Any = ssm_provider.get_multiple("/param", transform="json")
    for key, value in values.items():
        print(f"{key}: {value}")

    try:
        # This will raise a TransformParameterError exception
        values = ssm_provider.get_multiple("/param", transform="json", raise_on_transform_error=True)
    except parameters.exceptions.TransformParameterError:
        ...

```

#### Auto-transform values on suffix

If you use `transform` with `get_multiple()`, you might want to retrieve and transform parameters encoded in different formats.

You can do this with a single request by using `transform="auto"`. This will instruct any Parameter to to infer its type based on the suffix and transform it accordingly.

Info

`transform="auto"` feature is available across all providers, including the high level functions.

```
from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext

ssm_provider = parameters.SSMProvider()


def lambda_handler(event: dict, context: LambdaContext):
    values = ssm_provider.get_multiple("/param", transform="auto")

    return values

```

For example, if you have two parameters with the following suffixes `.json` and `.binary`:

| Parameter name | Parameter value | | --- | --- | | /param/a.json | [some encoded value] | | /param/a.binary | [some encoded value] |

The return of `ssm_provider.get_multiple("/param", transform="auto")` call will be a dictionary like:

```
{
    "a.json": [some value],
    "b.binary": [some value]
}

```

### Passing additional SDK arguments

You can use arbitrary keyword arguments to pass it directly to the underlying SDK method.

```
from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext

secrets_provider = parameters.SecretsProvider()


def lambda_handler(event: dict, context: LambdaContext):
    # The 'VersionId' argument will be passed to the underlying get_secret_value() call.
    value = secrets_provider.get("my-secret", VersionId="e62ec170-6b01-48c7-94f3-d7497851a8d2")

    return value

```

Here is the mapping between this utility's functions and methods and the underlying SDK:

| Provider | Function/Method | Client name | Function name | | --- | --- | --- | --- | | SSM Parameter Store | `get_parameter` | `ssm` | [get_parameter](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ssm.html#SSM.Client.get_parameter) | | SSM Parameter Store | `get_parameters` | `ssm` | [get_parameters_by_path](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ssm.html#SSM.Client.get_parameters_by_path) | | SSM Parameter Store | `SSMProvider.get` | `ssm` | [get_parameter](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ssm.html#SSM.Client.get_parameter) | | SSM Parameter Store | `SSMProvider.get_multiple` | `ssm` | [get_parameters_by_path](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ssm.html#SSM.Client.get_parameters_by_path) | | Secrets Manager | `get_secret` | `secretsmanager` | [get_secret_value](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/secretsmanager.html#SecretsManager.Client.get_secret_value) | | Secrets Manager | `SecretsProvider.get` | `secretsmanager` | [get_secret_value](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/secretsmanager.html#SecretsManager.Client.get_secret_value) | | DynamoDB | `DynamoDBProvider.get` | `dynamodb` | ([Table resource](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/dynamodb.html#table)) | | DynamoDB | `DynamoDBProvider.get_multiple` | `dynamodb` | ([Table resource](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/dynamodb.html#table)) | | App Config | `get_app_config` | `appconfigdata` | [start_configuration_session](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/appconfigdata.html#AppConfigData.Client.start_configuration_session) and [get_latest_configuration](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/appconfigdata.html#AppConfigData.Client.get_latest_configuration) |

### Bring your own boto client

You can use `boto3_client` parameter via any of the available [Provider Classes](#built-in-provider-class). Some providers expect a low level boto3 client while others expect a high level boto3 client, here is the mapping for each of them:

| Provider | Type | Boto client construction | | --- | --- | --- | | [SSMProvider](#ssmprovider) | low level | `boto3.client("ssm")` | | [SecretsProvider](#secretsprovider) | low level | `boto3.client("secrets")` | | [AppConfigProvider](#appconfigprovider) | low level | `boto3.client("appconfigdata")` | | [DynamoDBProvider](#dynamodbprovider) | high level | `boto3.resource("dynamodb")` |

Bringing them together in a single code snippet would look like this:

```
import boto3
from botocore.config import Config

from aws_lambda_powertools.utilities import parameters

config = Config(region_name="us-west-1")

# construct boto clients with any custom configuration
ssm = boto3.client("ssm", config=config)
secrets = boto3.client("secretsmanager", config=config)
appconfig = boto3.client("appconfigdata", config=config)
dynamodb = boto3.resource("dynamodb", config=config)

ssm_provider = parameters.SSMProvider(boto3_client=ssm)
secrets_provider = parameters.SecretsProvider(boto3_client=secrets)
appconf_provider = parameters.AppConfigProvider(boto3_client=appconfig, environment="my_env", application="my_app")
dynamodb_provider = parameters.DynamoDBProvider(boto3_client=dynamodb, table_name="my-table")

```

When is this useful?

Injecting a custom boto3 client can make unit/snapshot testing easier, including SDK customizations.

### Customizing boto configuration

The **`boto_config`** , **`boto3_session`**, and **`boto3_client`** parameters enable you to pass in a custom [botocore config object](https://botocore.amazonaws.com/v1/documentation/api/latest/reference/config.html), [boto3 session](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/core/session.html), or a [boto3 client](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/core/boto3.html) when constructing any of the built-in provider classes.

Tip

You can use a custom session for retrieving parameters cross-account/region and for snapshot testing.

When using VPC private endpoints, you can pass a custom client altogether. It's also useful for testing when injecting fake instances.

```
import boto3

from aws_lambda_powertools.utilities import parameters

boto3_session = boto3.session.Session()
ssm_provider = parameters.SSMProvider(boto3_session=boto3_session)


def handler(event, context):
    # Retrieve a single parameter
    value = ssm_provider.get("/my/parameter")

    return value

```

```
from botocore.config import Config

from aws_lambda_powertools.utilities import parameters

boto_config = Config()
ssm_provider = parameters.SSMProvider(boto_config=boto_config)


def handler(event, context):
    # Retrieve a single parameter
    value = ssm_provider.get("/my/parameter")

    return value

```

```
import boto3

from aws_lambda_powertools.utilities import parameters

boto3_client = boto3.client("ssm")
ssm_provider = parameters.SSMProvider(boto3_client=boto3_client)


def handler(event, context):
    # Retrieve a single parameter
    value = ssm_provider.get("/my/parameter")

    return value

```

## Testing your code

### Mocking parameter values

For unit testing your applications, you can mock the calls to the parameters utility to avoid calling AWS APIs. This can be achieved in a number of ways - in this example, we use the [pytest monkeypatch fixture](https://docs.pytest.org/en/latest/how-to/monkeypatch.html) to patch the `parameters.get_parameter` method:

```
from src import single_mock


def test_handler(monkeypatch):
    def mockreturn(name):
        return "mock_value"

    monkeypatch.setattr(single_mock.parameters, "get_parameter", mockreturn)
    return_val = single_mock.handler({}, {})
    assert return_val.get("message") == "mock_value"

```

```
from aws_lambda_powertools.utilities import parameters


def handler(event, context):
    # Retrieve a single parameter
    value = parameters.get_parameter("my-parameter-name")
    return {"message": value}

```

If we need to use this pattern across multiple tests, we can avoid repetition by refactoring to use our own pytest fixture:

```
import pytest
from src import single_mock


@pytest.fixture
def mock_parameter_response(monkeypatch):
    def mockreturn(name):
        return "mock_value"

    monkeypatch.setattr(single_mock.parameters, "get_parameter", mockreturn)


# Pass our fixture as an argument to all tests where we want to mock the get_parameter response
def test_handler(mock_parameter_response):
    return_val = single_mock.handler({}, {})
    assert return_val.get("message") == "mock_value"

```

Alternatively, if we need more fully featured mocking (for example checking the arguments passed to `get_parameter`), we can use [unittest.mock](https://docs.python.org/3/library/unittest.mock.html) from the python stdlib instead of pytest's `monkeypatch` fixture. In this example, we use the [patch](https://docs.python.org/3/library/unittest.mock.html#unittest.mock.patch) decorator to replace the `aws_lambda_powertools.utilities.parameters.get_parameter` function with a [MagicMock](https://docs.python.org/3/library/unittest.mock.html#unittest.mock.MagicMock) object named `get_parameter_mock`.

```
from unittest.mock import patch

from src import single_mock


# Replaces "aws_lambda_powertools.utilities.parameters.get_parameter" with a Mock object
@patch("aws_lambda_powertools.utilities.parameters.get_parameter")
def test_handler(get_parameter_mock):
    get_parameter_mock.return_value = "mock_value"

    return_val = single_mock.handler({}, {})
    get_parameter_mock.assert_called_with("my-parameter-name")
    assert return_val.get("message") == "mock_value"

```

### Clearing cache

Parameters utility caches all parameter values for performance and cost reasons. However, this can have unintended interference in tests using the same parameter name.

Within your tests, you can use `clear_cache` method available in [every provider](#built-in-provider-class). When using multiple providers or higher level functions like `get_parameter`, use `clear_caches` standalone function to clear cache globally.

```
import pytest
from src import app


@pytest.fixture(scope="function", autouse=True)
def clear_parameters_cache():
    yield
    app.ssm_provider.clear_cache()  # This will clear SSMProvider cache


@pytest.fixture
def mock_parameter_response(monkeypatch):
    def mockreturn(name):
        return "mock_value"

    monkeypatch.setattr(app.ssm_provider, "get", mockreturn)


# Pass our fixture as an argument to all tests where we want to mock the get_parameter response
def test_handler(mock_parameter_response):
    return_val = app.handler({}, {})
    assert return_val.get("message") == "mock_value"

```

```
import pytest
from src import app

from aws_lambda_powertools.utilities import parameters


@pytest.fixture(scope="function", autouse=True)
def clear_parameters_cache():
    yield
    parameters.clear_caches()  # This will clear all providers cache


@pytest.fixture
def mock_parameter_response(monkeypatch):
    def mockreturn(name):
        return "mock_value"

    monkeypatch.setattr(app.ssm_provider, "get", mockreturn)


# Pass our fixture as an argument to all tests where we want to mock the get_parameter response
def test_handler(mock_parameter_response):
    return_val = app.handler({}, {})
    assert return_val.get("message") == "mock_value"

```

```
from botocore.config import Config

from aws_lambda_powertools.utilities import parameters

ssm_provider = parameters.SSMProvider(config=Config(region_name="us-west-1"))


def handler(event, context):
    value = ssm_provider.get("/my/parameter")
    return {"message": value}

```
