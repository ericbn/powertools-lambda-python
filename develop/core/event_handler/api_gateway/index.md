Event handler for Amazon API Gateway REST and HTTP APIs, Application Load Balancer (ALB), Lambda Function URLs, and VPC Lattice.

## Key Features

- Lightweight routing to reduce boilerplate for API Gateway REST/HTTP API, ALB and Lambda Function URLs
- Support for CORS, binary and Gzip compression, Decimals JSON encoding and bring your own JSON serializer
- Built-in integration with [Event Source Data Classes utilities](../../../utilities/data_classes/) for self-documented event schema
- Works with micro function (one or a few routes) and monolithic functions (all routes)
- Support for OpenAPI and data validation for requests/responses

## Getting started

Tip

All examples shared in this documentation are available within the [project repository](https://github.com/aws-powertools/powertools-lambda-python/tree/develop/examples).

### Install

This is not necessary if you're installing Powertools for AWS Lambda (Python) via [Lambda Layer/SAR](../../../#lambda-layer).

**When using the data validation feature**, you need to add `pydantic` as a dependency in your preferred tool *e.g., requirements.txt, pyproject.toml*. At this time, we only support Pydantic V2.

### Required resources

If you're using any API Gateway integration, you must have an existing [API Gateway Proxy integration](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html) or [ALB](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/lambda-functions.html) configured to invoke your Lambda function.

In case of using [VPC Lattice](https://docs.aws.amazon.com/lambda/latest/dg/services-vpc-lattice.html), you must have a service network configured to invoke your Lambda function.

This is the sample infrastructure for API Gateway and Lambda Function URLs we are using for the examples in this documentation.

There is no additional permissions or dependencies required to use this utility.

```
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31
Description: Hello world event handler API Gateway

Globals:
  Api:
    TracingEnabled: true
    Cors: # see CORS section
      AllowOrigin: "'https://example.com'"
      AllowHeaders: "'Content-Type,Authorization,X-Amz-Date'"
      MaxAge: "'300'"
    BinaryMediaTypes: # see Binary responses section
      - "*~1*" # converts to */* for any binary type
      # NOTE: use this stricter version if you're also using CORS; */* doesn't work with CORS
      # see: https://github.com/aws-powertools/powertools-lambda-python/issues/3373#issuecomment-1821144779
      # - "image~1*" # converts to image/*
      # - "*~1csv" # converts to */csv, eg text/csv, application/csv

  Function:
    Timeout: 5
    Runtime: python3.12
    Tracing: Active
    Environment:
      Variables:
        POWERTOOLS_LOG_LEVEL: INFO
        POWERTOOLS_LOGGER_SAMPLE_RATE: 0.1
        POWERTOOLS_LOGGER_LOG_EVENT: true
        POWERTOOLS_SERVICE_NAME: example

Resources:
  ApiFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: getting_started_rest_api_resolver.lambda_handler
      CodeUri: ../src
      Description: API handler function
      Events:
        AnyApiEvent:
          Type: Api
          Properties:
            # NOTE: this is a catch-all rule to simplify the documentation.
            # explicit routes and methods are recommended for prod instead (see below)
            Path: /{proxy+} # Send requests on any path to the lambda function
            Method: ANY # Send requests using any http method to the lambda function


        # GetAllTodos:
        #   Type: Api
        #   Properties:
        #     Path: /todos
        #     Method: GET
        # GetTodoById:
        #   Type: Api
        #   Properties:
        #     Path: /todos/{todo_id}
        #     Method: GET
        # CreateTodo:
        #   Type: Api
        #   Properties:
        #     Path: /todos
        #     Method: POST

        ## Swagger UI specific routes

        # SwaggerUI:
        #     Type: Api
        #     Properties:
        #         Path: /swagger
        #         Method: GET

```

```
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31
Description: Hello world event handler Lambda Function URL

Globals:
  Function:
    Timeout: 5
    Runtime: python3.12
    Tracing: Active
    Environment:
      Variables:
        POWERTOOLS_LOG_LEVEL: INFO
        POWERTOOLS_LOGGER_SAMPLE_RATE: 0.1
        POWERTOOLS_LOGGER_LOG_EVENT: true
        POWERTOOLS_SERVICE_NAME: example
    FunctionUrlConfig:
      Cors: # see CORS section
        # Notice that values here are Lists of Strings, vs comma-separated values on API Gateway
        AllowOrigins: ["https://example.com"]
        AllowHeaders: ["Content-Type", "Authorization", "X-Amz-Date"]
        MaxAge: 300

Resources:
  ApiFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: getting_started_lambda_function_url_resolver.lambda_handler
      CodeUri: ../src
      Description: API handler function
      FunctionUrlConfig:
        AuthType: NONE # AWS_IAM for added security beyond sample documentation

```

### Event Resolvers

Before you decorate your functions to handle a given path and HTTP method(s), you need to initialize a resolver. A resolver will handle request resolution, including [one or more routers](#split-routes-with-router), and give you access to the current event via typed properties.

By default, we will use `APIGatewayRestResolver` throughout the documentation. You can use any of the following:

| Resolver | AWS service | | --- | --- | | **[`APIGatewayRestResolver`](#api-gateway-rest-api)** | Amazon API Gateway REST API | | **[`APIGatewayHttpResolver`](#api-gateway-http-api)** | Amazon API Gateway HTTP API | | **[`ALBResolver`](#application-load-balancer)** | Amazon Application Load Balancer (ALB) | | **[`LambdaFunctionUrlResolver`](#lambda-function-url)** | AWS Lambda Function URL | | **[`VPCLatticeResolver`](#vpc-lattice)** | Amazon VPC Lattice |

#### Response auto-serialization

> Want full control of the response, headers and status code? [Read about `Response` object here](#fine-grained-responses).

For your convenience, we automatically perform these if you return a dictionary response:

1. Auto-serialize `dictionary` responses to JSON and trim it
1. Include the response under each resolver's equivalent of a `body`
1. Set `Content-Type` to `application/json`
1. Set `status_code` to 200 (OK)

```
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.utilities.typing.lambda_context import LambdaContext

app = APIGatewayRestResolver()


@app.get("/ping")
def ping():
    return {"message": "pong"}  # (1)!


def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

1. This dictionary will be serialized, trimmed, and included under the `body` key

```
{
    "statusCode": 200,
    "multiValueHeaders": {
        "Content-Type": [
            "application/json"
        ]
    },
    "body": "{'message':'pong'}",
    "isBase64Encoded": false
}

```

Coming from Flask? We also support tuple response

You can optionally set a different HTTP status code as the second argument of the tuple.

```
import requests
from requests import Response

from aws_lambda_powertools.event_handler import ALBResolver
from aws_lambda_powertools.utilities.typing import LambdaContext

app = ALBResolver()


@app.post("/todo")
def create_todo():
    data: dict = app.current_event.json_body
    todo: Response = requests.post("https://jsonplaceholder.typicode.com/todos", data=data)

    # Returns the created todo object, with a HTTP 201 Created status
    return {"todo": todo.json()}, 201


def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

#### API Gateway REST API

When using Amazon API Gateway REST API to front your Lambda functions, you can use `APIGatewayRestResolver`.

Here's an example on how we can handle the `/todos` path.

Trailing slash in routes

For `APIGatewayRestResolver`, we seamless handle routes with a trailing slash (`/todos/`).

```
import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver()


@app.get("/todos")
@tracer.capture_method
def get_todos():
    todos: Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    todos.raise_for_status()

    # for brevity, we'll limit to the first 10 only
    return {"todos": todos.json()[:10]}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

This utility uses `path` and `httpMethod` to route to the right function. This helps make unit tests and local invocation easier too.

```
{
    "body": "",
    "resource": "/todos",
    "path": "/todos",
    "httpMethod": "GET",
    "isBase64Encoded": false,
    "queryStringParameters": {},
    "multiValueQueryStringParameters": {},
    "pathParameters": {},
    "stageVariables": {},
    "headers": {
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8",
        "Accept-Encoding": "gzip, deflate, sdch",
        "Accept-Language": "en-US,en;q=0.8",
        "Cache-Control": "max-age=0",
        "CloudFront-Forwarded-Proto": "https",
        "CloudFront-Is-Desktop-Viewer": "true",
        "CloudFront-Is-Mobile-Viewer": "false",
        "CloudFront-Is-SmartTV-Viewer": "false",
        "CloudFront-Is-Tablet-Viewer": "false",
        "CloudFront-Viewer-Country": "US",
        "Host": "1234567890.execute-api.us-east-1.amazonaws.com",
        "Upgrade-Insecure-Requests": "1",
        "User-Agent": "Custom User Agent String",
        "Via": "1.1 08f323deadbeefa7af34d5feb414ce27.cloudfront.net (CloudFront)",
        "X-Amz-Cf-Id": "cDehVQoZnx43VYQb9j2-nvCh-9z396Uhbp027Y2JvkCPNLmGJHqlaA==",
        "X-Forwarded-For": "127.0.0.1, 127.0.0.2",
        "X-Forwarded-Port": "443",
        "X-Forwarded-Proto": "https"
    },
    "multiValueHeaders": {},
    "requestContext": {
        "accountId": "123456789012",
        "resourceId": "123456",
        "stage": "Prod",
        "requestId": "c6af9ac6-7b61-11e6-9a41-93e8deadbeef",
        "requestTime": "25/Jul/2020:12:34:56 +0000",
        "requestTimeEpoch": 1428582896000,
        "identity": {
            "cognitoIdentityPoolId": null,
            "accountId": null,
            "cognitoIdentityId": null,
            "caller": null,
            "accessKey": null,
            "sourceIp": "127.0.0.1",
            "cognitoAuthenticationType": null,
            "cognitoAuthenticationProvider": null,
            "userArn": null,
            "userAgent": "Custom User Agent String",
            "user": null
        },
        "path": "/Prod/todos",
        "resourcePath": "/todos",
        "httpMethod": "GET",
        "apiId": "1234567890",
        "protocol": "HTTP/1.1"
    }
}

```

```
{
    "statusCode": 200,
    "multiValueHeaders": {
        "Content-Type": ["application/json"]
    },
    "body": "{\"todos\":[{\"userId\":1,\"id\":1,\"title\":\"delectus aut autem\",\"completed\":false},{\"userId\":1,\"id\":2,\"title\":\"quis ut nam facilis et officia qui\",\"completed\":false},{\"userId\":1,\"id\":3,\"title\":\"fugiat veniam minus\",\"completed\":false},{\"userId\":1,\"id\":4,\"title\":\"et porro tempora\",\"completed\":true},{\"userId\":1,\"id\":5,\"title\":\"laboriosam mollitia et enim quasi adipisci quia provident illum\",\"completed\":false},{\"userId\":1,\"id\":6,\"title\":\"qui ullam ratione quibusdam voluptatem quia omnis\",\"completed\":false},{\"userId\":1,\"id\":7,\"title\":\"illo expedita consequatur quia in\",\"completed\":false},{\"userId\":1,\"id\":8,\"title\":\"quo adipisci enim quam ut ab\",\"completed\":true},{\"userId\":1,\"id\":9,\"title\":\"molestiae perspiciatis ipsa\",\"completed\":false},{\"userId\":1,\"id\":10,\"title\":\"illo est ratione doloremque quia maiores aut\",\"completed\":true}]}",
    "isBase64Encoded": false
}

```

#### API Gateway HTTP API

When using Amazon API Gateway HTTP API to front your Lambda functions, you can use `APIGatewayHttpResolver`.

Note

Using HTTP API v1 payload? Use `APIGatewayRestResolver` instead. `APIGatewayHttpResolver` defaults to v2 payload.

If you're using Terraform to deploy a HTTP API, note that it defaults the [payload_format_version](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/apigatewayv2_integration#payload_format_version) value to 1.0 if not specified.

```
import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayHttpResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayHttpResolver()


@app.get("/todos")
@tracer.capture_method
def get_todos():
    todos: Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    todos.raise_for_status()

    # for brevity, we'll limit to the first 10 only
    return {"todos": todos.json()[:10]}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_HTTP)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

#### Application Load Balancer

When using Amazon Application Load Balancer (ALB) to front your Lambda functions, you can use `ALBResolver`.

```
import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import ALBResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = ALBResolver()


@app.get("/todos")
@tracer.capture_method
def get_todos():
    todos: Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    todos.raise_for_status()

    # for brevity, we'll limit to the first 10 only
    return {"todos": todos.json()[:10]}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.APPLICATION_LOAD_BALANCER)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

#### Lambda Function URL

When using [AWS Lambda Function URL](https://docs.aws.amazon.com/lambda/latest/dg/urls-configuration.html), you can use `LambdaFunctionUrlResolver`.

```
import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import LambdaFunctionUrlResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = LambdaFunctionUrlResolver()


@app.get("/todos")
@tracer.capture_method
def get_todos():
    todos: Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    todos.raise_for_status()

    # for brevity, we'll limit to the first 10 only
    return {"todos": todos.json()[:10]}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.LAMBDA_FUNCTION_URL)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

```
{
  "version": "2.0",
  "routeKey": "$default",
  "rawPath": "/todos",
  "rawQueryString": "",
  "headers": {
    "x-amz-content-sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "x-amzn-tls-version": "TLSv1.2",
    "x-amz-date": "20220803T092917Z",
    "x-forwarded-proto": "https",
    "x-forwarded-port": "443",
    "x-forwarded-for": "123.123.123.123",
    "accept": "application/xml",
    "x-amzn-tls-cipher-suite": "ECDHE-RSA-AES128-GCM-SHA256",
    "x-amzn-trace-id": "Root=1-63ea3fee-51ba94542feafa3928745ba3",
    "host": "xxxxxxxxxxxxx.lambda-url.eu-central-1.on.aws",
    "content-type": "application/json",
    "accept-encoding": "gzip, deflate",
    "user-agent": "Custom User Agent"
  },
  "requestContext": {
    "accountId": "123457890",
    "apiId": "xxxxxxxxxxxxxxxxxxxx",
    "authorizer": {
      "iam": {
        "accessKey": "AAAAAAAAAAAAAAAAAA",
        "accountId": "123457890",
        "callerId": "AAAAAAAAAAAAAAAAAA",
        "cognitoIdentity": null,
        "principalOrgId": "o-xxxxxxxxxxxx",
        "userArn": "arn:aws:iam::AAAAAAAAAAAAAAAAAA:user/user",
        "userId": "AAAAAAAAAAAAAAAAAA"
      }
    },
    "domainName": "xxxxxxxxxxxxx.lambda-url.eu-central-1.on.aws",
    "domainPrefix": "xxxxxxxxxxxxx",
    "http": {
      "method": "GET",
      "path": "/todos",
      "protocol": "HTTP/1.1",
      "sourceIp": "123.123.123.123",
      "userAgent": "Custom User Agent"
    },
    "requestId": "24f9ef37-8eb7-45fe-9dbc-a504169fd2f8",
    "routeKey": "$default",
    "stage": "$default",
    "time": "03/Aug/2022:09:29:18 +0000",
    "timeEpoch": 1659518958068
  },
  "isBase64Encoded": false
}

```

#### VPC Lattice

When using [VPC Lattice with AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/services-vpc-lattice.html), you can use `VPCLatticeV2Resolver`.

```
import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import VPCLatticeV2Resolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = VPCLatticeV2Resolver()


@app.get("/todos")
@tracer.capture_method
def get_todos():
    todos: Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    todos.raise_for_status()

    # for brevity, we'll limit to the first 10 only
    return {"todos": todos.json()[:10]}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.APPLICATION_LOAD_BALANCER)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

```
{
  "version": "2.0",
  "path": "/todos",
  "method": "GET",
  "headers": {
    "user_agent": "curl/7.64.1",
    "x-forwarded-for": "10.213.229.10",
    "host": "test-lambda-service-3908sdf9u3u.dkfjd93.vpc-lattice-svcs.us-east-2.on.aws",
    "accept": "*/*"
  },
  "queryStringParameters": {
    "order-id": "1"
  },
  "body": "{\"message\": \"Hello from Lambda!\"}",
  "requestContext": {
      "serviceNetworkArn": "arn:aws:vpc-lattice:us-east-2:123456789012:servicenetwork/sn-0bf3f2882e9cc805a",
      "serviceArn": "arn:aws:vpc-lattice:us-east-2:123456789012:service/svc-0a40eebed65f8d69c",
      "targetGroupArn": "arn:aws:vpc-lattice:us-east-2:123456789012:targetgroup/tg-6d0ecf831eec9f09",
      "identity": {
        "sourceVpcArn": "arn:aws:ec2:region:123456789012:vpc/vpc-0b8276c84697e7339",
        "type" : "AWS_IAM",
        "principal": "arn:aws:sts::123456789012:assumed-role/example-role/057d00f8b51257ba3c853a0f248943cf",
        "sessionName": "057d00f8b51257ba3c853a0f248943cf",
        "x509SanDns": "example.com"
      },
      "region": "us-east-2",
      "timeEpoch": "1696331543569073"
  }
}

```

```
import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import VPCLatticeResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = VPCLatticeResolver()


@app.get("/todos")
@tracer.capture_method
def get_todos():
    todos: Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    todos.raise_for_status()

    # for brevity, we'll limit to the first 10 only
    return {"todos": todos.json()[:10]}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.APPLICATION_LOAD_BALANCER)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

```
{
    "raw_path": "/testpath",
    "method": "GET",
    "headers": {
      "user_agent": "curl/7.64.1",
      "x-forwarded-for": "10.213.229.10",
      "host": "test-lambda-service-3908sdf9u3u.dkfjd93.vpc-lattice-svcs.us-east-2.on.aws",
      "accept": "*/*"
    },
    "query_string_parameters": {
      "order-id": "1"
    },
    "body": "eyJ0ZXN0IjogImV2ZW50In0=",
    "is_base64_encoded": true
  }

```

### Dynamic routes

You can use `/todos/<todo_id>` to configure dynamic URL paths, where `<todo_id>` will be resolved at runtime.

Each dynamic route you set must be part of your function signature. This allows us to call your function using keyword arguments when matching your dynamic route.

Note

For brevity, we will only include the necessary keys for each sample request for the example to work.

```
from urllib.parse import quote

import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver()


@app.get("/todos/<todo_id>")
@tracer.capture_method
def get_todo_by_id(todo_id: str):  # value come as str
    todo_id = quote(todo_id, safe="")
    todos: Response = requests.get(f"https://jsonplaceholder.typicode.com/todos/{todo_id}")
    todos.raise_for_status()

    return {"todos": todos.json()}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

```
{
    "resource": "/todos/{id}",
    "path": "/todos/1",
    "httpMethod": "GET"
}

```

Tip

You can also nest dynamic paths, for example `/todos/<todo_id>/<todo_status>`.

#### Catch-all routes

Note

We recommend having explicit routes whenever possible; use catch-all routes sparingly.

You can use a [regex](https://docs.python.org/3/library/re.html#regular-expression-syntax) string to handle an arbitrary number of paths within a request, for example `.+`.

You can also combine nested paths with greedy regex to catch in between routes.

Warning

We choose the most explicit registered route that matches an incoming event.

```
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver()


@app.get(".+")
@tracer.capture_method
def catch_any_route_get_method():
    return {"path_received": app.current_event.path}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

```
{
    "resource": "/{proxy+}",
    "path": "/any/route/should/work",
    "httpMethod": "GET"
}

```

### HTTP Methods

You can use named decorators to specify the HTTP method that should be handled in your functions. That is, `app.<http_method>`, where the HTTP method could be `get`, `post`, `put`, `patch` and `delete`.

```
import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver()


@app.post("/todos")
@tracer.capture_method
def create_todo():
    todo_data: dict = app.current_event.json_body  # deserialize json str to dict
    todo: Response = requests.post("https://jsonplaceholder.typicode.com/todos", data=todo_data)
    todo.raise_for_status()

    return {"todo": todo.json()}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

```
{
    "resource": "/todos",
    "path": "/todos",
    "httpMethod": "POST",
    "body": "{\"title\": \"foo\", \"userId\": 1, \"completed\": false}"
}

```

If you need to accept multiple HTTP methods in a single function, or support a HTTP method for which no decorator exists (e.g. [TRACE](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/TRACE)), you can use the `route` method and pass a list of HTTP methods.

```
import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver()


# PUT and POST HTTP requests to the path /hello will route to this function
@app.route("/todos", method=["PUT", "POST"])
@tracer.capture_method
def create_todo():
    todo_data: dict = app.current_event.json_body  # deserialize json str to dict
    todo: Response = requests.post("https://jsonplaceholder.typicode.com/todos", data=todo_data)
    todo.raise_for_status()

    return {"todo": todo.json()}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

Note

It is generally better to have separate functions for each HTTP method, as the functionality tends to differ depending on which method is used.

### Data validation

This changes the authoring experience by relying on Python's type annotations

It's inspired by [FastAPI framework](https://fastapi.tiangolo.com/) for ergonomics and to ease migrations in either direction. We support both Pydantic models and Python's dataclass.

For brevity, we'll focus on Pydantic only.

All resolvers can optionally coerce and validate incoming requests by setting `enable_validation=True`.

With this feature, we can now express how we expect our incoming data and response to look like. This moves data validation responsibilities to Event Handler resolvers, reducing a ton of boilerplate code.

Let's rewrite the previous examples to signal our resolver what shape we expect our data to be.

```
from typing import Optional

import requests
from pydantic import BaseModel, Field

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver(enable_validation=True)  # (1)!


class Todo(BaseModel):  # (2)!
    userId: int
    id_: Optional[int] = Field(alias="id", default=None)
    title: str
    completed: bool


@app.get("/todos/<todo_id>")  # (3)!
@tracer.capture_method
def get_todo_by_id(todo_id: int) -> Todo:  # (4)!
    todo = requests.get(f"https://jsonplaceholder.typicode.com/todos/{todo_id}")
    todo.raise_for_status()

    return todo.json()  # (5)!


@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_HTTP)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

1. This enforces data validation at runtime. Any validation error will return `HTTP 422: Unprocessable Entity error`.

1. We create a Pydantic model to define how our data looks like.

1. Defining a route remains exactly as before.

1. By default, URL Paths will be `str`. Here, we are telling our resolver it should be `int`, so it converts it for us.

   Lastly, we're also saying the return should be our `Todo`. This will help us later when we touch OpenAPI auto-documentation.

1. `todo.json()` returns a dictionary. However, Event Handler knows the response should be `Todo` so it converts and validates accordingly.

```
{
  "version": "1.0",
  "resource": "/todos/1",
  "path": "/todos/1",
  "httpMethod": "GET",
  "headers": {
    "Origin": "https://aws.amazon.com"
  },
  "multiValueHeaders": {},
  "queryStringParameters": {},
  "multiValueQueryStringParameters": {},
  "requestContext": {
    "accountId": "123456789012",
    "apiId": "id",
    "authorizer": {
      "claims": null,
      "scopes": null
    },
    "domainName": "id.execute-api.us-east-1.amazonaws.com",
    "domainPrefix": "id",
    "extendedRequestId": "request-id",
    "httpMethod": "GET",
    "path": "/todos/1",
    "protocol": "HTTP/1.1",
    "requestId": "id=",
    "requestTime": "04/Mar/2020:19:15:17 +0000",
    "requestTimeEpoch": 1583349317135,
    "resourceId": null,
    "resourcePath": "/todos/1",
    "stage": "$default"
  },
  "pathParameters": null,
  "stageVariables": null,
  "body": "",
  "isBase64Encoded": false
}

```

```
{
  "statusCode": 200,
  "body": "Hello world",
  "isBase64Encoded": false,
  "multiValueHeaders": {
    "Content-Type": [
      "application/json"
    ]
  }
}

```

#### Handling validation errors

By default, we hide extended error details for security reasons *(e.g., pydantic url, Pydantic code)*.

Any incoming request or and outgoing response that fails validation will lead to a `HTTP 422: Unprocessable Entity error` response that will look similar to this:

```
{
  "statusCode": 422,
  "body": "{\"statusCode\": 422, \"detail\": [{\"type\": \"int_parsing\", \"loc\": [\"path\", \"todo_id\"]}]}",
  "isBase64Encoded": false,
  "headers": {
    "Content-Type": "application/json"
  },
  "cookies": []
}

```

You can customize the error message by catching the `RequestValidationError` exception. This is useful when you might have a security policy to return opaque validation errors, or have a company standard for API validation errors.

Here's an example where we catch validation errors, log all details for further investigation, and return the same `HTTP 422` with an opaque error.

```
from typing import Optional

import requests
from pydantic import BaseModel, Field

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver, Response, content_types
from aws_lambda_powertools.event_handler.openapi.exceptions import RequestValidationError
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver(enable_validation=True)


class Todo(BaseModel):
    userId: int
    id_: Optional[int] = Field(alias="id", default=None)
    title: str
    completed: bool


@app.exception_handler(RequestValidationError)  # (1)!
def handle_validation_error(ex: RequestValidationError):
    logger.error("Request failed validation", path=app.current_event.path, errors=ex.errors())

    return Response(
        status_code=422,
        content_type=content_types.APPLICATION_JSON,
        body="Invalid data",
    )


@app.post("/todos")
def create_todo(todo: Todo) -> int:
    response = requests.post("https://jsonplaceholder.typicode.com/todos", json=todo.dict(by_alias=True))
    response.raise_for_status()

    return response.json()["id"]


@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_HTTP)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

1. We use [exception handler](#exception-handling) decorator to catch **any** request validation errors.

   Then, we log the detailed reason as to why it failed while returning a custom `Response` object to hide that from them.

```
{
  "statusCode": 422,
  "body": "Invalid data",
  "isBase64Encoded": false,
  "headers": {
    "Content-Type": "application/json"
  },
  "cookies": []
}

```

#### Validating payloads

We will automatically validate, inject, and convert incoming request payloads based on models via type annotation.

Let's improve our previous example by handling the creation of todo items via `HTTP POST`.

What we want is for Event Handler to convert the incoming payload as an instance of our `Todo` model. We handle the creation of that `todo`, and then return the `ID` of the newly created `todo`.

Even better, we can also let Event Handler validate and convert our response according to type annotations, further reducing boilerplate.

```
from typing import List, Optional

import requests
from pydantic import BaseModel, Field

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver(enable_validation=True)  # (1)!


class Todo(BaseModel):  # (2)!
    userId: int
    id_: Optional[int] = Field(alias="id", default=None)
    title: str
    completed: bool


@app.post("/todos")
def create_todo(todo: Todo) -> str:  # (3)!
    response = requests.post("https://jsonplaceholder.typicode.com/todos", json=todo.dict(by_alias=True))
    response.raise_for_status()

    return response.json()["id"]  # (4)!


@app.get("/todos")
@tracer.capture_method
def get_todos() -> List[Todo]:
    todo = requests.get("https://jsonplaceholder.typicode.com/todos")
    todo.raise_for_status()

    return todo.json()  # (5)!


@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_HTTP)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

1. This enforces data validation at runtime. Any validation error will return `HTTP 422: Unprocessable Entity error`.

1. We create a Pydantic model to define how our data looks like.

1. We define `Todo` as our type annotation. Event Handler then uses this model to validate and inject the incoming request as `Todo`.

1. Lastly, we return the ID of our newly created `todo` item.

   Because we specify the return type (`str`), Event Handler will take care of serializing this as a JSON string.

1. Note that the return type is `List[Todo]`.

   Event Handler will take the return (`todo.json`), and validate each list item against `Todo` model before returning the response accordingly.

```
{
  "version": "1.0",
  "body": "{\"title\": \"foo\", \"userId\": \"1\", \"completed\": false}",
  "resource": "/todos",
  "path": "/todos",
  "httpMethod": "POST",
  "headers": {
    "Origin": "https://aws.amazon.com"
  },
  "multiValueHeaders": {},
  "queryStringParameters": {},
  "multiValueQueryStringParameters": {},
  "requestContext": {
    "accountId": "123456789012",
    "apiId": "id",
    "authorizer": {
      "claims": null,
      "scopes": null
    },
    "domainName": "id.execute-api.us-east-1.amazonaws.com",
    "domainPrefix": "id",
    "extendedRequestId": "request-id",
    "httpMethod": "POST",
    "path": "/todos",
    "protocol": "HTTP/1.1",
    "requestId": "id=",
    "requestTime": "04/Mar/2020:19:15:17 +0000",
    "requestTimeEpoch": 1583349317135,
    "resourceId": null,
    "resourcePath": "/todos",
    "stage": "$default"
  },
  "pathParameters": null,
  "stageVariables": null,
  "isBase64Encoded": false
}

```

```
{
  "statusCode": 200,
  "body": "2008821",
  "isBase64Encoded": false,
  "multiValueHeaders": {
    "Content-Type": [
      "application/json"
    ]
  }
}

```

##### Validating payload subset

With the addition of the [`Annotated` type starting in Python 3.9](https://docs.python.org/3/library/typing.html#typing.Annotated), types can contain additional metadata, allowing us to represent anything we want.

We use the `Annotated` and OpenAPI `Body` type to instruct Event Handler that our payload is located in a particular JSON key.

Event Handler will match the parameter name with the JSON key to validate and inject what you want.

```
from typing import Optional

import requests
from pydantic import BaseModel, Field
from typing_extensions import Annotated

from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.event_handler.openapi.params import Body  # (1)!
from aws_lambda_powertools.utilities.typing import LambdaContext

app = APIGatewayRestResolver(enable_validation=True)


class Todo(BaseModel):
    userId: int
    id_: Optional[int] = Field(alias="id", default=None)
    title: str
    completed: bool


@app.post("/todos")
def create_todo(todo: Annotated[Todo, Body(embed=True)]) -> int:  # (2)!
    response = requests.post("https://jsonplaceholder.typicode.com/todos", json=todo.dict(by_alias=True))
    response.raise_for_status()

    return response.json()["id"]


def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

1. `Body` is a special OpenAPI type that can add additional constraints to a request payload.

1. `Body(embed=True)` instructs Event Handler to look up inside the payload for a key.

   This means Event Handler will look up for a key named `todo`, validate the value against `Todo`, and inject it.

```
{
  "version": "1.0",
  "body": "{ \"todo\": {\"title\": \"foo\", \"userId\": \"1\", \"completed\": false } }",
  "resource": "/todos",
  "path": "/todos",
  "httpMethod": "POST",
  "headers": {
    "Origin": "https://aws.amazon.com"
  },
  "multiValueHeaders": {},
  "queryStringParameters": {},
  "multiValueQueryStringParameters": {},
  "requestContext": {
    "accountId": "123456789012",
    "apiId": "id",
    "authorizer": {
      "claims": null,
      "scopes": null
    },
    "domainName": "id.execute-api.us-east-1.amazonaws.com",
    "domainPrefix": "id",
    "extendedRequestId": "request-id",
    "httpMethod": "POST",
    "path": "/todos",
    "protocol": "HTTP/1.1",
    "requestId": "id=",
    "requestTime": "04/Mar/2020:19:15:17 +0000",
    "requestTimeEpoch": 1583349317135,
    "resourceId": null,
    "resourcePath": "/todos",
    "stage": "$default"
  },
  "pathParameters": null,
  "stageVariables": null,
  "isBase64Encoded": false
}

```

```
{
  "statusCode": 200,
  "body": "2008822",
  "isBase64Encoded": false,
  "multiValueHeaders": {
    "Content-Type": [
      "application/json"
    ]
  }
}

```

#### Validating responses

You can use `response_validation_error_http_code` to set a custom HTTP code for failed response validation. When this field is set, we will raise a `ResponseValidationError` instead of a `RequestValidationError`.

For a more granular control over the failed response validation http code, the `custom_response_validation_http_code` argument can be set per route. This value will override the value of the failed response validation http code set at constructor level (if any).

```
from http import HTTPStatus
from typing import Optional

import requests
from pydantic import BaseModel, Field

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver(
    enable_validation=True,
    response_validation_error_http_code=HTTPStatus.INTERNAL_SERVER_ERROR,  # (1)!
)


class Todo(BaseModel):
    userId: int
    id_: Optional[int] = Field(alias="id", default=None)
    title: str
    completed: bool


@app.get("/todos_bad_response/<todo_id>")
@tracer.capture_method
def get_todo_by_id(todo_id: int) -> Todo:
    todo = requests.get(f"https://jsonplaceholder.typicode.com/todos/{todo_id}")
    todo.raise_for_status()

    return todo.json()["title"]  # (2)!


@app.get(
    "/todos_bad_response_with_custom_http_code/<todo_id>",
    custom_response_validation_http_code=HTTPStatus.UNPROCESSABLE_ENTITY,  # (3)!
)
@tracer.capture_method
def get_todo_by_id_custom(todo_id: int) -> Todo:
    todo = requests.get(f"https://jsonplaceholder.typicode.com/todos/{todo_id}")
    todo.raise_for_status()

    return todo.json()["title"]


@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_HTTP)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

1. A response with status code set here will be returned if response data is not valid.
1. Operation returns a string as oppose to a `Todo` object. This will lead to a `500` response as set in line 16.
1. Operation will return a `422 Unprocessable Entity` response if response is not a `Todo` object. This overrides the custom http code set in line 16.

```
from http import HTTPStatus
from typing import Optional

import requests
from pydantic import BaseModel, Field

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver(
    enable_validation=True,
    response_validation_error_http_code=HTTPStatus.INTERNAL_SERVER_ERROR,  # (1)!
)


class Todo(BaseModel):
    userId: int
    id_: Optional[int] = Field(alias="id", default=None)
    title: str
    completed: bool


@app.get("/todos_bad_response/<todo_id>")
@tracer.capture_method
def get_todo_by_id(todo_id: int) -> Todo:
    todo = requests.get(f"https://jsonplaceholder.typicode.com/todos/{todo_id}")
    todo.raise_for_status()

    return todo.json()["title"]  # (2)!


@app.get(
    "/todos_bad_response_with_custom_http_code/<todo_id>",
    custom_response_validation_http_code=HTTPStatus.UNPROCESSABLE_ENTITY,  # (3)!
)
@tracer.capture_method
def get_todo_by_id_custom(todo_id: int) -> Todo:
    todo = requests.get(f"https://jsonplaceholder.typicode.com/todos/{todo_id}")
    todo.raise_for_status()

    return todo.json()["title"]


@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_HTTP)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

1. A response with status code set here will be returned if response data is not valid.
1. Operation returns a string as oppose to a `Todo` object. This will lead to a `500` response as set in line 18.

```
from http import HTTPStatus
from typing import Optional

import requests
from pydantic import BaseModel, Field

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver, content_types
from aws_lambda_powertools.event_handler.api_gateway import Response
from aws_lambda_powertools.event_handler.openapi.exceptions import ResponseValidationError
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver(
    enable_validation=True,
    response_validation_error_http_code=HTTPStatus.INTERNAL_SERVER_ERROR,
)


class Todo(BaseModel):
    userId: int
    id_: Optional[int] = Field(alias="id", default=None)
    title: str
    completed: bool


@app.get("/todos_bad_response/<todo_id>")
@tracer.capture_method
def get_todo_by_id(todo_id: int) -> Todo:
    todo = requests.get(f"https://jsonplaceholder.typicode.com/todos/{todo_id}")
    todo.raise_for_status()

    return todo.json()["title"]


@app.exception_handler(ResponseValidationError)  # (1)!
def handle_response_validation_error(ex: ResponseValidationError):
    logger.error("Request failed validation", path=app.current_event.path, errors=ex.errors())

    return Response(
        status_code=500,
        content_type=content_types.APPLICATION_JSON,
        body="Unexpected response.",
    )


@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_HTTP)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

1. The distinct `ResponseValidationError` exception can be caught to customise the response.

#### Validating query strings

We will automatically validate and inject incoming query strings via type annotation.

We use the `Annotated` type to tell the Event Handler that a particular parameter is not only an optional string, but also a query string with constraints.

In the following example, we use a new `Query` OpenAPI type to add [one out of many possible constraints](#customizing-openapi-parameters), which should read as:

- `completed` is a query string with a `None` as its default value
- `completed`, when set, should have at minimum 4 characters
- No match? Event Handler will return a validation error response

```
from typing import List, Optional

import requests
from pydantic import BaseModel, Field
from typing_extensions import Annotated  # (1)!

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.event_handler.openapi.params import Query  # (2)!
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver(enable_validation=True)


class Todo(BaseModel):
    userId: int
    id_: Optional[int] = Field(alias="id", default=None)
    title: str
    completed: bool


@app.get("/todos")
@tracer.capture_method
def get_todos(completed: Annotated[Optional[str], Query(min_length=4)] = None) -> List[Todo]:  # (3)!
    url = "https://jsonplaceholder.typicode.com/todos"

    if completed is not None:
        url = f"{url}/?completed={completed}"

    todo = requests.get(url)
    todo.raise_for_status()

    return todo.json()


@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_HTTP)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

1. If you're not using Python 3.9 or higher, you can install and use [`typing_extensions`](https://pypi.org/project/typing-extensions/) to the same effect

1. `Query` is a special OpenAPI type that can add constraints to a query string as well as document them

1. **First time seeing `Annotated`?**

   This special type uses the first argument as the actual type, and subsequent arguments as metadata.

   At runtime, static checkers will also see the first argument, but any receiver can inspect it to get the metadata.

If you don't want to validate query strings but simply let Event Handler inject them as parameters, you can omit `Query` type annotation.

This is merely for your convenience.

```
from typing import List, Optional

import requests
from pydantic import BaseModel, Field

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver(enable_validation=True)


class Todo(BaseModel):
    userId: int
    id_: Optional[int] = Field(alias="id", default=None)
    title: str
    completed: bool


@app.get("/todos")
@tracer.capture_method
def get_todos(completed: Optional[str] = None) -> List[Todo]:  # (1)!
    url = "https://jsonplaceholder.typicode.com/todos"

    if completed is not None:
        url = f"{url}/?completed={completed}"

    todo = requests.get(url)
    todo.raise_for_status()

    return todo.json()


@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_HTTP)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

1. `completed` is still the same query string as before, except we simply state it's an string. No `Query` or `Annotated` to validate it.

If you need to handle multi-value query parameters, you can create a list of the desired type.

```
from enum import Enum
from typing import List

from typing_extensions import Annotated

from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.event_handler.openapi.params import Query
from aws_lambda_powertools.utilities.typing import LambdaContext

app = APIGatewayRestResolver(enable_validation=True)


class ExampleEnum(Enum):
    """Example of an Enum class."""

    ONE = "value_one"
    TWO = "value_two"
    THREE = "value_three"


@app.get("/todos")
def get(
    example_multi_value_param: Annotated[
        List[ExampleEnum],  # (1)!
        Query(
            description="This is multi value query parameter.",
        ),
    ],
):
    """Return validated multi-value param values."""
    return example_multi_value_param


def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

1. `example_multi_value_param` is a list containing values from the `ExampleEnum` enumeration.

#### Validating path parameters

Just like we learned in [query string validation](#validating-query-strings), we can use a new `Path` OpenAPI type to [add constraints](#customizing-openapi-parameters).

For example, we could validate that `<todo_id>` dynamic path should be no greater than three digits.

```
from typing import Optional

import requests
from pydantic import BaseModel, Field
from typing_extensions import Annotated

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.event_handler.openapi.params import Path
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver(enable_validation=True)


class Todo(BaseModel):
    userId: int
    id_: Optional[int] = Field(alias="id", default=None)
    title: str
    completed: bool


@app.get("/todos/<todo_id>")
@tracer.capture_method
def get_todo_by_id(todo_id: Annotated[int, Path(lt=999)]) -> Todo:  # (1)!
    todo = requests.get(f"https://jsonplaceholder.typicode.com/todos/{todo_id}")
    todo.raise_for_status()

    return todo.json()


@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_HTTP)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

1. `Path` is a special OpenAPI type that allows us to constrain todo_id to be less than 999.

#### Validating headers

We use the `Annotated` type to tell the Event Handler that a particular parameter is a header that needs to be validated.

We adhere to [HTTP RFC standards](https://www.rfc-editor.org/rfc/rfc7540#section-8.1.2), which means we treat HTTP headers as case-insensitive.

In the following example, we use a new `Header` OpenAPI type to add [one out of many possible constraints](#customizing-openapi-parameters), which should read as:

- `correlation_id` is a header that must be present in the request
- `correlation_id` should have 16 characters
- No match? Event Handler will return a validation error response

```
from typing import List, Optional

import requests
from pydantic import BaseModel, Field
from typing_extensions import Annotated  # (1)!

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.event_handler.openapi.params import Header  # (2)!
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver(enable_validation=True)


class Todo(BaseModel):
    userId: int
    id_: Optional[int] = Field(alias="id", default=None)
    title: str
    completed: bool


@app.get("/todos")
@tracer.capture_method
def get_todos(correlation_id: Annotated[str, Header(min_length=16, max_length=16)]) -> List[Todo]:  # (3)!
    url = "https://jsonplaceholder.typicode.com/todos"

    todo = requests.get(url, headers={"correlation_id": correlation_id})
    todo.raise_for_status()

    return todo.json()


@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_HTTP)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

1. If you're not using Python 3.9 or higher, you can install and use [`typing_extensions`](https://pypi.org/project/typing-extensions/) to the same effect

1. `Header` is a special OpenAPI type that can add constraints and documentation to a header

1. **First time seeing `Annotated`?**

   This special type uses the first argument as the actual type, and subsequent arguments as metadata.

   At runtime, static checkers will also see the first argument, but any receiver can inspect it to get the metadata.

You can handle multi-value headers by declaring it as a list of the desired type.

```
from enum import Enum
from typing import List

from typing_extensions import Annotated

from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.event_handler.openapi.params import Header
from aws_lambda_powertools.utilities.typing import LambdaContext

app = APIGatewayRestResolver(enable_validation=True)


class CountriesAllowed(Enum):
    """Example of an Enum class."""

    US = "US"
    PT = "PT"
    BR = "BR"


@app.get("/hello")
def get(
    cloudfront_viewer_country: Annotated[
        List[CountriesAllowed],  # (1)!
        Header(
            description="This is multi value header parameter.",
        ),
    ],
):
    """Return validated multi-value header values."""
    return cloudfront_viewer_country


def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

1. `cloudfront_viewer_country` is a list that must contain values from the `CountriesAllowed` enumeration.

#### Supported types for response serialization

With data validation enabled, we natively support serializing the following data types to JSON:

| Data type | Serialized type | | --- | --- | | **Pydantic models** | `dict` | | **Python Dataclasses** | `dict` | | **Enum** | Enum values | | **Datetime** | Datetime ISO format string | | **Decimal** | `int` if no exponent, or `float` | | **Path** | `str` | | **UUID** | `str` | | **Set** | `list` | | **Python primitives** *(dict, string, sequences, numbers, booleans)* | [Python's default JSON serializable types](https://docs.python.org/3/library/json.html#encoders-and-decoders) |

See [custom serializer section](#custom-serializer) for bringing your own.

Otherwise, we will raise `SerializationError` for any unsupported types *e.g., SQLAlchemy models*.

### Accessing request details

Event Handler integrates with [Event Source Data Classes utilities](../../../utilities/data_classes/), and it exposes their respective resolver request details and convenient methods under `app.current_event`.

That is why you see `app.resolve(event, context)` in every example. This allows Event Handler to resolve requests, and expose data like `app.lambda_context` and `app.current_event`.

#### Query strings and payload

Within `app.current_event` property, you can access all available query strings as a dictionary via `query_string_parameters`.

You can access the raw payload via `body` property, or if it's a JSON string you can quickly deserialize it via `json_body` property - like the earlier example in the [HTTP Methods](#http-methods) section.

```
from typing import List, Optional

import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver()


@app.get("/todos")
@tracer.capture_method
def get_todos():
    todo_id: str = app.current_event.query_string_parameters["id"]
    # alternatively
    _: Optional[str] = app.current_event.query_string_parameters.get("id")

    # or multi-value query string parameters; ?category="red"&?category="blue"
    _: List[str] = app.current_event.multi_value_query_string_parameters["category"]

    # Payload
    _: Optional[str] = app.current_event.body  # raw str | None

    endpoint = "https://jsonplaceholder.typicode.com/todos"
    if todo_id:
        endpoint = f"{endpoint}/{todo_id}"

    todos: Response = requests.get(endpoint)
    todos.raise_for_status()

    return {"todos": todos.json()}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

#### Headers

Similarly to [Query strings](#query-strings-and-payload), you can access headers as dictionary via `app.current_event.headers`. Specifically for headers, it's a case-insensitive dictionary, so all lookups are case-insensitive.

```
import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver()


@app.get("/todos")
@tracer.capture_method
def get_todos():
    endpoint = "https://jsonplaceholder.typicode.com/todos"

    api_key = app.current_event.headers.get("X-Api-Key")
    todos: Response = requests.get(endpoint, headers={"X-Api-Key": api_key})
    todos.raise_for_status()

    return {"todos": todos.json()}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

### Handling not found routes

By default, we return `404` for any unmatched route.

You can use **`not_found`** decorator to override this behavior, and return a custom **`Response`**.

```
import requests

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import (
    APIGatewayRestResolver,
    Response,
    content_types,
)
from aws_lambda_powertools.event_handler.exceptions import NotFoundError
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver()


@app.not_found
@tracer.capture_method
def handle_not_found_errors(exc: NotFoundError) -> Response:
    logger.info(f"Not found route: {app.current_event.path}")
    return Response(status_code=418, content_type=content_types.TEXT_PLAIN, body="I'm a teapot!")


@app.get("/todos")
@tracer.capture_method
def get_todos():
    todos: requests.Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    todos.raise_for_status()

    # for brevity, we'll limit to the first 10 only
    return {"todos": todos.json()[:10]}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

### Exception handling

You can use **`exception_handler`** decorator with any Python exception. This allows you to handle a common exception outside your route, for example validation errors.

```
import requests

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import (
    APIGatewayRestResolver,
    Response,
    content_types,
)
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver()


@app.exception_handler(ValueError)
def handle_invalid_limit_qs(ex: ValueError):  # receives exception raised
    metadata = {"path": app.current_event.path, "query_strings": app.current_event.query_string_parameters}
    logger.error(f"Malformed request: {ex}", extra=metadata)

    return Response(
        status_code=400,
        content_type=content_types.TEXT_PLAIN,
        body="Invalid request parameters.",
    )


@app.get("/todos")
@tracer.capture_method
def get_todos():
    # educational purpose only: we should receive a `ValueError`
    # if a query string value for `limit` cannot be coerced to int
    max_results = int(app.current_event.query_string_parameters.get("limit", 0))

    todos: requests.Response = requests.get(f"https://jsonplaceholder.typicode.com/todos?limit={max_results}")
    todos.raise_for_status()

    return {"todos": todos.json()}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

Info

The `exception_handler` also supports passing a list of exception types you wish to handle with one handler.

### Raising HTTP errors

You can easily raise any HTTP Error back to the client using `ServiceError` exception. This ensures your Lambda function doesn't fail but return the correct HTTP response signalling the error.

Info

If you need to send custom headers, use [Response](#fine-grained-responses) class instead.

We provide pre-defined errors for the most popular ones based on [AWS Lambda API Reference Common Erros](https://docs.aws.amazon.com/lambda/latest/api/CommonErrors.html).

```
import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.event_handler.exceptions import (
    BadRequestError,
    ForbiddenError,
    InternalServerError,
    NotFoundError,
    RequestEntityTooLargeError,
    RequestTimeoutError,
    ServiceError,
    ServiceUnavailableError,
    UnauthorizedError,
)
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver()


@app.get(rule="/bad-request-error")
def bad_request_error():
    raise BadRequestError("Missing required parameter")  # HTTP  400


@app.get(rule="/unauthorized-error")
def unauthorized_error():
    raise UnauthorizedError("Unauthorized")  # HTTP 401


@app.get(rule="/forbidden-error")
def forbidden_error():
    raise ForbiddenError("Access denied")  # HTTP 403


@app.get(rule="/not-found-error")
def not_found_error():
    raise NotFoundError  # HTTP 404


@app.get(rule="/request-timeout-error")
def request_timeout_error():
    raise RequestTimeoutError("Request timed out")  # HTTP 408


@app.get(rule="/internal-server-error")
def internal_server_error():
    raise InternalServerError("Internal server error")  # HTTP 500


@app.get(rule="/request-entity-too-large-error")
def request_entity_too_large_error():
    raise RequestEntityTooLargeError("Request payload too large")  # HTTP 413


@app.get(rule="/service-error", cors=True)
def service_error():
    raise ServiceError(502, "Something went wrong!")


@app.get(rule="/service-unavailable-error")
def service_unavailable_error():
    raise ServiceUnavailableError("Service is temporarily unavailable")  # HTTP 503


@app.get("/todos")
@tracer.capture_method
def get_todos():
    todos: Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    todos.raise_for_status()

    return {"todos": todos.json()[:10]}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

### Enabling SwaggerUI

This feature requires [data validation](#data-validation) feature to be enabled.

Behind the scenes, the [data validation](#data-validation) feature auto-generates an OpenAPI specification from your routes and type annotations. You can use [Swagger UI](https://swagger.io/tools/swagger-ui/) to visualize and interact with your newly auto-documented API.

There are some important **caveats** that you should know before enabling it:

| Caveat | Description | | --- | --- | | Swagger UI is **publicly accessible by default** | When using `enable_swagger` method, you can [protect sensitive API endpoints by implementing a custom middleware](#customizing-swagger-ui) using your preferred authorization mechanism. | | **No micro-functions support** yet | Swagger UI is enabled on a per resolver instance which will limit its accuracy here. | | You need to expose a **new route** | You'll need to expose the following path to Lambda: `/swagger`; ignore if you're routing this path already. | | JS and CSS files are **embedded within Swagger HTML** | If you are not using an external CDN to serve Swagger UI assets, we embed JS and CSS directly into the HTML. To enhance performance, please consider enabling the `compress` option to minimize the size of HTTP requests. | | Authorization data is **lost** on browser close/refresh | Use `enable_swagger(persist_authorization=True)` to persist authorization data, like OAuath 2.0 access tokens. |

```
from typing import List

import requests
from pydantic import BaseModel, Field

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver(enable_validation=True)
app.enable_swagger(path="/swagger")  # (1)!


class Todo(BaseModel):
    userId: int
    id_: int = Field(alias="id")
    title: str
    completed: bool


@app.post("/todos")
def create_todo(todo: Todo) -> str:
    response = requests.post("https://jsonplaceholder.typicode.com/todos", json=todo.dict(by_alias=True))
    response.raise_for_status()

    return response.json()["id"]


@app.get("/todos")
def get_todos() -> List[Todo]:
    todo = requests.get("https://jsonplaceholder.typicode.com/todos")
    todo.raise_for_status()

    return todo.json()


def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

1. `enable_swagger` creates a route to serve Swagger UI and allows quick customizations.

   You can also include middlewares to protect or enhance the overall experience.

Here's an example of what it looks like by default:

### Custom Domain API Mappings

When using [Custom Domain API Mappings feature](https://docs.aws.amazon.com/apigateway/latest/developerguide/rest-api-mappings.html), you must use **`strip_prefixes`** param in the `APIGatewayRestResolver` constructor.

**Scenario**: You have a custom domain `api.mydomain.dev`. Then you set `/payment` API Mapping to forward any payment requests to your Payments API.

**Challenge**: This means your `path` value for any API requests will always contain `/payment/<actual_request>`, leading to HTTP 404 as Event Handler is trying to match what's after `payment/`. This gets further complicated with an [arbitrary level of nesting](https://github.com/aws-powertools/powertools-lambda/issues/34).

To address this API Gateway behavior, we use `strip_prefixes` parameter to account for these prefixes that are now injected into the path regardless of which type of API Gateway you're using.

```
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver(strip_prefixes=["/payment"])


@app.get("/subscriptions/<subscription>")
@tracer.capture_method
def get_subscription(subscription):
    return {"subscription_id": subscription}


@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

```
{
    "resource": "/subscriptions/{subscription}",
    "path": "/payment/subscriptions/123",
    "httpMethod": "GET"
}

```

Note

After removing a path prefix with `strip_prefixes`, the new root path will automatically be mapped to the path argument of `/`.

For example, when using `strip_prefixes` value of `/pay`, there is no difference between a request path of `/pay` and `/pay/`; and the path argument would be defined as `/`.

For added flexibility, you can use regexes to strip a prefix. This is helpful when you have many options due to different combinations of prefixes (e.g: multiple environments, multiple versions).

```
import re

from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.utilities.typing import LambdaContext

# This will support:
# /v1/dev/subscriptions/<subscription>
# /v1/stg/subscriptions/<subscription>
# /v1/qa/subscriptions/<subscription>
# /v2/dev/subscriptions/<subscription>
# ...
app = APIGatewayRestResolver(strip_prefixes=[re.compile(r"/v[1-3]+/(dev|stg|qa)")])


@app.get("/subscriptions/<subscription>")
def get_subscription(subscription):
    return {"subscription_id": subscription}


def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

## Advanced

### CORS

You can configure CORS at the `APIGatewayRestResolver` constructor via `cors` parameter using the `CORSConfig` class.

This will ensure that CORS headers are returned as part of the response when your functions match the path invoked and the `Origin` matches one of the allowed values.

Tip

Optionally disable CORS on a per path basis with `cors=False` parameter.

```
from urllib.parse import quote

import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver, CORSConfig
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
# CORS will match when Origin is only https://www.example.com
cors_config = CORSConfig(allow_origin="https://www.example.com", max_age=300)
app = APIGatewayRestResolver(cors=cors_config)


@app.get("/todos")
@tracer.capture_method
def get_todos():
    todos: Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    todos.raise_for_status()

    # for brevity, we'll limit to the first 10 only
    return {"todos": todos.json()[:10]}


@app.get("/todos/<todo_id>")
@tracer.capture_method
def get_todo_by_id(todo_id: str):  # value come as str
    todo_id = quote(todo_id, safe="")
    todos: Response = requests.get(f"https://jsonplaceholder.typicode.com/todos/{todo_id}")
    todos.raise_for_status()

    return {"todos": todos.json()}


@app.get("/healthcheck", cors=False)  # optionally removes CORS for a given route
@tracer.capture_method
def am_i_alive():
    return {"am_i_alive": "yes"}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

```
{
    "statusCode": 200,
    "multiValueHeaders": {
        "Content-Type": ["application/json"],
        "Access-Control-Allow-Origin": ["https://www.example.com"],
        "Access-Control-Allow-Headers": ["Authorization,Content-Type,X-Amz-Date,X-Amz-Security-Token,X-Api-Key"]
    },
    "body": "{\"todos\":[{\"userId\":1,\"id\":1,\"title\":\"delectus aut autem\",\"completed\":false},{\"userId\":1,\"id\":2,\"title\":\"quis ut nam facilis et officia qui\",\"completed\":false},{\"userId\":1,\"id\":3,\"title\":\"fugiat veniam minus\",\"completed\":false},{\"userId\":1,\"id\":4,\"title\":\"et porro tempora\",\"completed\":true},{\"userId\":1,\"id\":5,\"title\":\"laboriosam mollitia et enim quasi adipisci quia provident illum\",\"completed\":false},{\"userId\":1,\"id\":6,\"title\":\"qui ullam ratione quibusdam voluptatem quia omnis\",\"completed\":false},{\"userId\":1,\"id\":7,\"title\":\"illo expedita consequatur quia in\",\"completed\":false},{\"userId\":1,\"id\":8,\"title\":\"quo adipisci enim quam ut ab\",\"completed\":true},{\"userId\":1,\"id\":9,\"title\":\"molestiae perspiciatis ipsa\",\"completed\":false},{\"userId\":1,\"id\":10,\"title\":\"illo est ratione doloremque quia maiores aut\",\"completed\":true}]}",
    "isBase64Encoded": false
}

```

```
from urllib.parse import quote

import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver, CORSConfig
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
# CORS will match when Origin is https://www.example.com OR https://dev.example.com
cors_config = CORSConfig(allow_origin="https://www.example.com", extra_origins=["https://dev.example.com"], max_age=300)
app = APIGatewayRestResolver(cors=cors_config)


@app.get("/todos")
@tracer.capture_method
def get_todos():
    todos: Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    todos.raise_for_status()

    # for brevity, we'll limit to the first 10 only
    return {"todos": todos.json()[:10]}


@app.get("/todos/<todo_id>")
@tracer.capture_method
def get_todo_by_id(todo_id: str):  # value come as str
    todo_id = quote(todo_id, safe="")
    todos: Response = requests.get(f"https://jsonplaceholder.typicode.com/todos/{todo_id}")
    todos.raise_for_status()

    return {"todos": todos.json()}


@app.get("/healthcheck", cors=False)  # optionally removes CORS for a given route
@tracer.capture_method
def am_i_alive():
    return {"am_i_alive": "yes"}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

```
{
    "statusCode": 200,
    "multiValueHeaders": {
        "Content-Type": ["application/json"],
        "Access-Control-Allow-Origin": ["https://www.example.com","https://dev.example.com"],
        "Access-Control-Allow-Headers": ["Authorization,Content-Type,X-Amz-Date,X-Amz-Security-Token,X-Api-Key"]
    },
    "body": "{\"todos\":[{\"userId\":1,\"id\":1,\"title\":\"delectus aut autem\",\"completed\":false},{\"userId\":1,\"id\":2,\"title\":\"quis ut nam facilis et officia qui\",\"completed\":false},{\"userId\":1,\"id\":3,\"title\":\"fugiat veniam minus\",\"completed\":false},{\"userId\":1,\"id\":4,\"title\":\"et porro tempora\",\"completed\":true},{\"userId\":1,\"id\":5,\"title\":\"laboriosam mollitia et enim quasi adipisci quia provident illum\",\"completed\":false},{\"userId\":1,\"id\":6,\"title\":\"qui ullam ratione quibusdam voluptatem quia omnis\",\"completed\":false},{\"userId\":1,\"id\":7,\"title\":\"illo expedita consequatur quia in\",\"completed\":false},{\"userId\":1,\"id\":8,\"title\":\"quo adipisci enim quam ut ab\",\"completed\":true},{\"userId\":1,\"id\":9,\"title\":\"molestiae perspiciatis ipsa\",\"completed\":false},{\"userId\":1,\"id\":10,\"title\":\"illo est ratione doloremque quia maiores aut\",\"completed\":true}]}",
    "isBase64Encoded": false
}

```

#### Pre-flight

Pre-flight (OPTIONS) calls are typically handled at the API Gateway or Lambda Function URL level as per [our sample infrastructure](#required-resources), no Lambda integration is necessary. However, ALB expects you to handle pre-flight requests.

For convenience, we automatically handle that for you as long as you [setup CORS in the constructor level](#cors).

#### Defaults

For convenience, these are the default values when using `CORSConfig` to enable CORS:

Warning

Always configure `allow_origin` when using in production.

Multiple origins?

If you need to allow multiple origins, pass the additional origins using the `extra_origins` key.

| Key | Value | Note | | --- | --- | --- | | **[allow_origin](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Allow-Origin)**: `str` | `*` | Only use the default value for development. **Never use `*` for production** unless your use case requires it | | **[extra_origins](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Allow-Origin)**: `List[str]` | `[]` | Additional origins to be allowed, in addition to the one specified in `allow_origin` | | **[allow_headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Allow-Headers)**: `List[str]` | `[Authorization, Content-Type, X-Amz-Date, X-Api-Key, X-Amz-Security-Token]` | Additional headers will be appended to the default list for your convenience | | **[expose_headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Expose-Headers)**: `List[str]` | `[]` | Any additional header beyond the [safe listed by CORS specification](https://developer.mozilla.org/en-US/docs/Glossary/CORS-safelisted_response_header). | | **[max_age](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Max-Age)**: `int` | \`\` | Only for pre-flight requests if you choose to have your function to handle it instead of API Gateway | | **[allow_credentials](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Allow-Credentials)**: `bool` | `False` | Only necessary when you need to expose cookies, authorization headers or TLS client certificates. |

### Middleware

```
stateDiagram
    direction LR

    EventHandler: GET /todo
    Before: Before response
    Next: next_middleware()
    MiddlewareLoop: Middleware loop
    AfterResponse: After response
    MiddlewareFinished: Modified response
    Response: Final response

    EventHandler --> Middleware: Has middleware?
    state MiddlewareLoop {
        direction LR
        Middleware --> Before
        Before --> Next
        Next --> Middleware: More middlewares?
        Next --> AfterResponse
    }
    AfterResponse --> MiddlewareFinished
    MiddlewareFinished --> Response
    EventHandler --> Response: No middleware
```

A middleware is a function you register per route to **intercept** or **enrich** a **request before** or **after** any response.

Each middleware function receives the following arguments:

1. **app**. An Event Handler instance so you can access incoming request information, Lambda context, etc.
1. **next_middleware**. A function to get the next middleware or route's response.

Here's a sample middleware that extracts and injects correlation ID, using `APIGatewayRestResolver` (works for any [Resolver](#event-resolvers)):

```
import requests

from aws_lambda_powertools import Logger
from aws_lambda_powertools.event_handler import APIGatewayRestResolver, Response
from aws_lambda_powertools.event_handler.middlewares import NextMiddleware

app = APIGatewayRestResolver()
logger = Logger()


def inject_correlation_id(app: APIGatewayRestResolver, next_middleware: NextMiddleware) -> Response:
    request_id = app.current_event.request_context.request_id  # (1)!

    # Use API Gateway REST API request ID if caller didn't include a correlation ID
    correlation_id = logger.get_correlation_id() or request_id  # (2)!

    # Inject correlation ID in shared context and Logger
    app.append_context(correlation_id=correlation_id)  # (3)!
    logger.set_correlation_id(correlation_id)

    # Get response from next middleware OR /todos route
    result = next_middleware(app)  # (4)!

    # Include Correlation ID in the response back to caller
    result.headers["x-correlation-id"] = correlation_id  # (5)!
    return result


@app.get("/todos", middlewares=[inject_correlation_id])  # (6)!
def get_todos():
    todos: Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    todos.raise_for_status()

    # for brevity, we'll limit to the first 10 only
    return {"todos": todos.json()[:10]}


@logger.inject_lambda_context(correlation_id_path='headers."x-correlation-id"')  # (7)!
def lambda_handler(event, context):
    return app.resolve(event, context)

```

1. You can access current request like you normally would.

1. Logger extracts it first in the request path, so we can use it.

   If this was available before, we'd use `app.context.get("correlation_id")`.

1. [Shared context is available](#sharing-contextual-data) to any middleware, Router and App instances.

   For example, another middleware can now use `app.context.get("correlation_id")` to retrieve it.

1. Get response from the next middleware (if any) or from `/todos` route.

1. You can manipulate headers, body, or status code before returning it.

1. Register one or more middlewares in order of execution.

1. Logger extracts correlation ID from header and makes it available under `correlation_id` key, and `get_correlation_id()` method.

```
{
    "statusCode": 200,
    "body": "{\"todos\":[{\"userId\":1,\"id\":1,\"title\":\"delectus aut autem\",\"completed\":false}]}",
    "isBase64Encoded": false,
    "multiValueHeaders": {
        "Content-Type": [
            "application/json"
        ],
        "x-correlation-id": [
            "ccd87d70-7a3f-4aec-b1a8-a5a558c239b2"
        ]
    }
}

```

#### Global middlewares

![Combining middlewares](../../media/middlewares_normal_processing-light.svg#only-light) ![Combining middlewares](../../media/middlewares_normal_processing-dark.svg#only-dark) _Request flowing through multiple registered middlewares_

You can use `app.use` to register middlewares that should always run regardless of the route, also known as global middlewares.

Event Handler **calls global middlewares first**, then middlewares defined at the route level. Here's an example with both middlewares:

> Use [debug mode](#debug-mode) if you need to log request/response.

```
import middleware_global_middlewares_module  # (1)!
import requests

from aws_lambda_powertools import Logger
from aws_lambda_powertools.event_handler import APIGatewayRestResolver, Response

app = APIGatewayRestResolver()
logger = Logger()

app.use(middlewares=[middleware_global_middlewares_module.log_request_response])  # (2)!


@app.get("/todos", middlewares=[middleware_global_middlewares_module.inject_correlation_id])
def get_todos():
    todos: Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    todos.raise_for_status()

    # for brevity, we'll limit to the first 10 only
    return {"todos": todos.json()[:10]}


@logger.inject_lambda_context
def lambda_handler(event, context):
    return app.resolve(event, context)

```

1. A separate file where our middlewares are to keep this example focused.

1. We register `log_request_response` as a global middleware to run before middleware.

   ```
   stateDiagram
       direction LR

       GlobalMiddleware: Log request response
       RouteMiddleware: Inject correlation ID
       EventHandler: Event Handler

       EventHandler --> GlobalMiddleware
       GlobalMiddleware --> RouteMiddleware
   ```

```
from aws_lambda_powertools import Logger
from aws_lambda_powertools.event_handler import APIGatewayRestResolver, Response
from aws_lambda_powertools.event_handler.middlewares import NextMiddleware

logger = Logger()


def log_request_response(app: APIGatewayRestResolver, next_middleware: NextMiddleware) -> Response:
    logger.info("Incoming request", path=app.current_event.path, request=app.current_event.raw_event)

    result = next_middleware(app)
    logger.info("Response received", response=result.__dict__)

    return result


def inject_correlation_id(app: APIGatewayRestResolver, next_middleware: NextMiddleware) -> Response:
    request_id = app.current_event.request_context.request_id

    # Use API Gateway REST API request ID if caller didn't include a correlation ID
    correlation_id = logger.get_correlation_id() or request_id  # elsewhere becomes app.context.get("correlation_id")

    # Inject correlation ID in shared context and Logger
    app.append_context(correlation_id=correlation_id)
    logger.set_correlation_id(correlation_id)

    # Get response from next middleware OR /todos route
    result = next_middleware(app)

    # Include Correlation ID in the response back to caller
    result.headers["x-correlation-id"] = correlation_id
    return result


def enforce_correlation_id(app: APIGatewayRestResolver, next_middleware: NextMiddleware) -> Response:
    # If missing mandatory header raise an error
    if not app.current_event.headers.get("x-correlation-id"):
        return Response(status_code=400, body="Correlation ID header is now mandatory.")  # (1)!

    # Get the response from the next middleware and return it
    return next_middleware(app)

```

#### Returning early

![Short-circuiting middleware chain](../../media/middlewares_early_return-light.svg#only-light) ![Short-circuiting middleware chain](../../media/middlewares_early_return-dark.svg#only-dark) _Interrupting request flow by returning early_

Imagine you want to stop processing a request if something is missing, or return immediately if you've seen this request before.

In these scenarios, you short-circuit the middleware processing logic by returning a [Response object](#fine-grained-responses), or raising a [HTTP Error](#raising-http-errors). This signals to Event Handler to stop and run each `After` logic left in the chain all the way back.

Here's an example where we prevent any request that doesn't include a correlation ID header:

```
import middleware_global_middlewares_module
import requests

from aws_lambda_powertools import Logger
from aws_lambda_powertools.event_handler import APIGatewayRestResolver, Response

app = APIGatewayRestResolver()
logger = Logger()
app.use(
    middlewares=[
        middleware_global_middlewares_module.log_request_response,
        middleware_global_middlewares_module.enforce_correlation_id,  # (1)!
    ],
)


@app.get("/todos")
def get_todos():
    todos: Response = requests.get("https://jsonplaceholder.typicode.com/todos")  # (2)!
    todos.raise_for_status()

    # for brevity, we'll limit to the first 10 only
    return {"todos": todos.json()[:10]}


@logger.inject_lambda_context
def lambda_handler(event, context):
    return app.resolve(event, context)

```

1. This middleware will raise an exception if correlation ID header is missing.
1. This code section will not run if `enforce_correlation_id` returns early.

```
from aws_lambda_powertools import Logger
from aws_lambda_powertools.event_handler import APIGatewayRestResolver, Response
from aws_lambda_powertools.event_handler.middlewares import NextMiddleware

logger = Logger()


def log_request_response(app: APIGatewayRestResolver, next_middleware: NextMiddleware) -> Response:
    logger.info("Incoming request", path=app.current_event.path, request=app.current_event.raw_event)

    result = next_middleware(app)
    logger.info("Response received", response=result.__dict__)

    return result


def inject_correlation_id(app: APIGatewayRestResolver, next_middleware: NextMiddleware) -> Response:
    request_id = app.current_event.request_context.request_id

    # Use API Gateway REST API request ID if caller didn't include a correlation ID
    correlation_id = logger.get_correlation_id() or request_id  # elsewhere becomes app.context.get("correlation_id")

    # Inject correlation ID in shared context and Logger
    app.append_context(correlation_id=correlation_id)
    logger.set_correlation_id(correlation_id)

    # Get response from next middleware OR /todos route
    result = next_middleware(app)

    # Include Correlation ID in the response back to caller
    result.headers["x-correlation-id"] = correlation_id
    return result


def enforce_correlation_id(app: APIGatewayRestResolver, next_middleware: NextMiddleware) -> Response:
    # If missing mandatory header raise an error
    if not app.current_event.headers.get("x-correlation-id"):
        return Response(status_code=400, body="Correlation ID header is now mandatory.")  # (1)!

    # Get the response from the next middleware and return it
    return next_middleware(app)

```

1. Raising an exception OR returning a Response object early will short-circuit the middleware chain.

```
{
    "statusCode": 400,
    "body": "Correlation ID header is now mandatory",
    "isBase64Encoded": false,
    "multiValueHeaders": {}
}

```

#### Handling exceptions

For catching exceptions more broadly, we recommend you use the [exception_handler](#exception-handling) decorator.

By default, any unhandled exception in the middleware chain is eventually propagated as a HTTP 500 back to the client.

While there isn't anything special on how to use [`try/catch`](https://docs.python.org/3/tutorial/errors.html#handling-exceptions) for middlewares, it is important to visualize how Event Handler deals with them under the following scenarios:

An exception wasn't caught by any middleware during `next_middleware()` block, therefore it propagates all the way back to the client as HTTP 500.

*Unhandled route exceptions propagate back to the client*

An exception was only caught by the third middleware, resuming the normal execution of each `After` logic for the second and first middleware.

*Unhandled route exceptions propagate back to the client*

The third middleware short-circuited the chain by raising an exception and completely skipping the fourth middleware. Because we only caught it in the first middleware, it skipped the `After` logic in the second middleware.

*Middleware handling short-circuit exceptions*

#### Extending middlewares

You can implement `BaseMiddlewareHandler` interface to create middlewares that accept configuration, or perform complex operations (*see [being a good citizen section](#being-a-good-citizen)*).

As a practical example, let's refactor our correlation ID middleware so it accepts a custom HTTP Header to look for.

```
import requests

from aws_lambda_powertools import Logger
from aws_lambda_powertools.event_handler import APIGatewayRestResolver, Response
from aws_lambda_powertools.event_handler.middlewares import BaseMiddlewareHandler, NextMiddleware

app = APIGatewayRestResolver()
logger = Logger()


class CorrelationIdMiddleware(BaseMiddlewareHandler):
    def __init__(self, header: str):  # (1)!
        """Extract and inject correlation ID in response

        Parameters
        ----------
        header : str
            HTTP Header to extract correlation ID
        """
        super().__init__()
        self.header = header

    def handler(self, app: APIGatewayRestResolver, next_middleware: NextMiddleware) -> Response:  # (2)!
        request_id = app.current_event.request_context.request_id
        correlation_id = app.current_event.headers.get(self.header, request_id)

        response = next_middleware(app)  # (3)!
        response.headers[self.header] = correlation_id

        return response


@app.get("/todos", middlewares=[CorrelationIdMiddleware(header="x-correlation-id")])  # (4)!
def get_todos():
    todos: requests.Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    todos.raise_for_status()

    # for brevity, we'll limit to the first 10 only
    return {"todos": todos.json()[:10]}


@logger.inject_lambda_context
def lambda_handler(event, context):
    return app.resolve(event, context)

```

1. You can add any constructor argument like you normally would
1. We implement `handler` just like we [did before](#middleware) with the only exception of the `self` argument, since it's a method.
1. Get response from the next middleware (if any) or from `/todos` route.
1. Register an instance of `CorrelationIdMiddleware`.

Class-based **vs** function-based middlewares

When registering a middleware, we expect a callable in both cases. For class-based middlewares, `BaseMiddlewareHandler` is doing the work of calling your `handler` method with the correct parameters, hence why we expect an instance of it.

#### Native middlewares

These are native middlewares that may become native features depending on customer demand.

| Middleware | Purpose | | --- | --- | | [SchemaValidationMiddleware](/lambda/python/latest/api/event_handler/middlewares/schema_validation.html) | Validates API request body and response against JSON Schema, using [Validation utility](../../../utilities/validation/) |

#### Being a good citizen

Middlewares can add subtle improvements to request/response processing, but also add significant complexity if you're not careful.

Keep the following in mind when authoring middlewares for Event Handler:

1. **Use built-in features over middlewares**. We include built-in features like [CORS](#cors), [compression](#compress), [binary responses](#binary-responses), [global exception handling](#exception-handling), and [debug mode](#debug-mode) to reduce the need for middlewares.
1. **Call the next middleware**. Return the result of `next_middleware(app)`, or a [Response object](#fine-grained-responses) when you want to [return early](#returning-early).
1. **Keep a lean scope**. Focus on a single task per middleware to ease composability and maintenance. In [debug mode](#debug-mode), we also print out the order middlewares will be triggered to ease operations.
1. **Catch your own exceptions**. Catch and handle known exceptions to your logic. Unless you want to raise [HTTP Errors](#raising-http-errors), or propagate specific exceptions to the client. To catch all and any exceptions, we recommend you use the [exception_handler](#exception-handling) decorator.
1. **Use context to share data**. Use `app.append_context` to [share contextual data](#sharing-contextual-data) between middlewares and route handlers, and `app.context.get(key)` to fetch them. We clear all contextual data at the end of every request.

### Fine grained responses

You can use the `Response` class to have full control over the response. For example, you might want to add additional headers, cookies, or set a custom Content-type.

Info

Powertools for AWS Lambda (Python) serializes headers and cookies according to the type of input event. Some event sources require headers and cookies to be encoded as `multiValueHeaders`.

Using multiple values for HTTP headers in ALB?

Make sure you [enable the multi value headers feature](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/lambda-functions.html#multi-value-headers) to serialize response headers correctly.

```
from http import HTTPStatus
from uuid import uuid4

import requests

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import (
    APIGatewayRestResolver,
    Response,
    content_types,
)
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.shared.cookies import Cookie
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver()


@app.get("/todos")
@tracer.capture_method
def get_todos():
    todos: requests.Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    todos.raise_for_status()

    custom_headers = {"X-Transaction-Id": [f"{uuid4()}"]}

    return Response(
        status_code=HTTPStatus.OK.value,  # 200
        content_type=content_types.APPLICATION_JSON,
        body=todos.json()[:10],
        headers=custom_headers,
        cookies=[Cookie(name="session_id", value="12345")],
    )


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

```
{
    "statusCode": 200,
    "multiValueHeaders": {
        "Content-Type": ["application/json"],
        "X-Transaction-Id": ["3490eea9-791b-47a0-91a4-326317db61a9"],
        "Set-Cookie": ["session_id=12345; Secure"]
    },
    "body": "{\"todos\":[{\"userId\":1,\"id\":1,\"title\":\"delectus aut autem\",\"completed\":false},{\"userId\":1,\"id\":2,\"title\":\"quis ut nam facilis et officia qui\",\"completed\":false},{\"userId\":1,\"id\":3,\"title\":\"fugiat veniam minus\",\"completed\":false},{\"userId\":1,\"id\":4,\"title\":\"et porro tempora\",\"completed\":true},{\"userId\":1,\"id\":5,\"title\":\"laboriosam mollitia et enim quasi adipisci quia provident illum\",\"completed\":false},{\"userId\":1,\"id\":6,\"title\":\"qui ullam ratione quibusdam voluptatem quia omnis\",\"completed\":false},{\"userId\":1,\"id\":7,\"title\":\"illo expedita consequatur quia in\",\"completed\":false},{\"userId\":1,\"id\":8,\"title\":\"quo adipisci enim quam ut ab\",\"completed\":true},{\"userId\":1,\"id\":9,\"title\":\"molestiae perspiciatis ipsa\",\"completed\":false},{\"userId\":1,\"id\":10,\"title\":\"illo est ratione doloremque quia maiores aut\",\"completed\":true}]}",
    "isBase64Encoded": false
}

```

Using `Response` with data validation?

When using the [data validation](#data-validation) feature with `enable_validation=True`, you must specify the concrete type for the `Response` class. This allows the validation middleware to infer the underlying type and perform validation correctly.

```
from http import HTTPStatus
from typing import Optional

import requests
from pydantic import BaseModel, Field

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver, Response, content_types
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver(enable_validation=True)


class Todo(BaseModel):
    userId: int
    id_: Optional[int] = Field(alias="id", default=None)
    title: str
    completed: bool


@app.get("/todos/<todo_id>")
@tracer.capture_method
def get_todo_by_id(todo_id: int) -> Response[Todo]:
    todo = requests.get(f"https://jsonplaceholder.typicode.com/todos/{todo_id}")
    todo.raise_for_status()
    return Response(
        status_code=HTTPStatus.OK.value,
        content_type=content_types.APPLICATION_JSON,
        body=todo.json(),
    )


@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

### Compress

You can compress with gzip and base64 encode your responses via `compress` parameter. You have the option to pass the `compress` parameter when working with a specific route or using the Response object.

Info

The `compress` parameter used in the Response object takes precedence over the one used in the route.

Warning

The client must send the `Accept-Encoding` header, otherwise a normal response will be sent.

```
 from urllib.parse import quote

 import requests

 from aws_lambda_powertools import Logger, Tracer
 from aws_lambda_powertools.event_handler import (
     APIGatewayRestResolver,
     Response,
     content_types,
 )
 from aws_lambda_powertools.logging import correlation_paths
 from aws_lambda_powertools.utilities.typing import LambdaContext

 tracer = Tracer()
 logger = Logger()
 app = APIGatewayRestResolver()


 @app.get("/todos", compress=True)
 @tracer.capture_method
 def get_todos():
     todos: requests.Response = requests.get("https://jsonplaceholder.typicode.com/todos")
     todos.raise_for_status()

     # for brevity, we'll limit to the first 10 only
     return {"todos": todos.json()[:10]}


 @app.get("/todos/<todo_id>", compress=True)
 @tracer.capture_method
 def get_todo_by_id(todo_id: str):  # same example using Response class
     todo_id = quote(todo_id, safe="")
     todos: requests.Response = requests.get(f"https://jsonplaceholder.typicode.com/todos/{todo_id}")
     todos.raise_for_status()

     return Response(status_code=200, content_type=content_types.APPLICATION_JSON, body=todos.json())


 # You can continue to use other utilities just as before
 @logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
 @tracer.capture_lambda_handler
 def lambda_handler(event: dict, context: LambdaContext) -> dict:
     return app.resolve(event, context)

```

```
 import requests

 from aws_lambda_powertools import Logger, Tracer
 from aws_lambda_powertools.event_handler import (
     APIGatewayRestResolver,
     Response,
     content_types,
 )
 from aws_lambda_powertools.logging import correlation_paths
 from aws_lambda_powertools.utilities.typing import LambdaContext

 tracer = Tracer()
 logger = Logger()
 app = APIGatewayRestResolver()


 @app.get("/todos")
 @tracer.capture_method
 def get_todos():
     todos: requests.Response = requests.get("https://jsonplaceholder.typicode.com/todos")
     todos.raise_for_status()

     # for brevity, we'll limit to the first 10 only
     return Response(status_code=200, content_type=content_types.APPLICATION_JSON, body=todos.json()[:10], compress=True)


 # You can continue to use other utilities just as before
 @logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
 @tracer.capture_lambda_handler
 def lambda_handler(event: dict, context: LambdaContext) -> dict:
     return app.resolve(event, context)

```

```
{
    "headers": {
        "Accept-Encoding": "gzip"
    },
    "resource": "/todos",
    "path": "/todos",
    "httpMethod": "GET"
}

```

```
{
    "statusCode": 200,
    "multiValueHeaders": {
        "Content-Type": ["application/json"],
        "Content-Encoding": ["gzip"]
    },
    "body": "H4sIAAAAAAACE42STU4DMQyFrxJl3QXln96AMyAW7sSDLCVxiJ0Kqerd8TCCUOgii1EmP/783pOPXjmw+N3L0TfB+hz8brvxtC5KGtHvfMCIkzZx0HT5MPmNnziViIr2dIYoeNr8Q1x3xHsjcVadIbkZJoq2RXU8zzQROLseQ9505NzeCNQdMJNBE+UmY4zbzjAJhWtlZ57sB84BWtul+rteH2HPlVgWARwjqXkxpklK5gmEHAQqJBMtFsGVygcKmNVRjG0wxvuzGF2L0dpVUOKMC3bfJNjJgWMrCuZk7cUp02AiD72D6WKHHwUDKbiJs6AZ0VZXKOUx4uNvzdxT+E4mLcMA+6G8nzrLQkaxkNEVrFKW2VGbJCoCY7q2V3+tiv5kGThyxfTecDWbgGz/NfYXhL6ePgF9PnFdPgMAAA==",
    "isBase64Encoded": true
}

```

### Binary responses

Amazon API Gateway does not support `*/*` binary media type [when CORS is also configured](https://github.com/aws-powertools/powertools-lambda-python/issues/3373#issuecomment-1821144779).

This feature requires API Gateway to configure binary media types, see [our sample infrastructure](#required-resources) for reference.

For convenience, we automatically base64 encode binary responses. You can also use in combination with `compress` parameter if your client supports gzip.

Like `compress` feature, the client must send the `Accept` header with the correct media type.

Lambda Function URLs handle binary media types automatically.

```
import os
from pathlib import Path

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler.api_gateway import (
    APIGatewayRestResolver,
    Response,
)
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()


app = APIGatewayRestResolver()
logo_file: bytes = Path(f"{os.getenv('LAMBDA_TASK_ROOT')}/logo.svg").read_bytes()


@app.get("/logo")
@tracer.capture_method
def get_logo():
    return Response(status_code=200, content_type="image/svg+xml", body=logo_file)


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

```
<?xml version="1.0" encoding="UTF-8"?>
<svg width="256px" height="256px" viewBox="0 0 256 256" version="1.1" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" preserveAspectRatio="xMidYMid">
    <title>AWS Lambda</title>
    <defs>
        <linearGradient x1="0%" y1="100%" x2="100%" y2="0%" id="linearGradient-1">
            <stop stop-color="#C8511B" offset="0%"></stop>
            <stop stop-color="#FF9900" offset="100%"></stop>
        </linearGradient>
    </defs>
    <g>
        <rect fill="url(#linearGradient-1)" x="0" y="0" width="256" height="256"></rect>
        <path d="M89.6241126,211.2 L49.8903277,211.2 L93.8354832,119.3472 L113.74728,160.3392 L89.6241126,211.2 Z M96.7029357,110.5696 C96.1640858,109.4656 95.0414813,108.7648 93.8162384,108.7648 L93.8066163,108.7648 C92.5717514,108.768 91.4491466,109.4752 90.9199187,110.5856 L41.9134208,213.0208 C41.4387197,214.0128 41.5060758,215.1776 42.0962451,216.1088 C42.6799994,217.0368 43.7063805,217.6 44.8065331,217.6 L91.654423,217.6 C92.8957027,217.6 94.0215149,216.8864 94.5539501,215.7696 L120.203859,161.6896 C120.617619,160.8128 120.614412,159.7984 120.187822,158.928 L96.7029357,110.5696 Z M207.985117,211.2 L168.507928,211.2 L105.173789,78.624 C104.644561,77.5104 103.515541,76.8 102.277469,76.8 L76.447943,76.8 L76.4768099,44.8 L127.103066,44.8 L190.145328,177.3728 C190.674556,178.4864 191.803575,179.2 193.041647,179.2 L207.985117,179.2 L207.985117,211.2 Z M211.192558,172.8 L195.071958,172.8 L132.029696,40.2272 C131.500468,39.1136 130.371449,38.4 129.130169,38.4 L73.272576,38.4 C71.5052758,38.4 70.0683421,39.8304 70.0651344,41.5968 L70.0298528,79.9968 C70.0298528,80.848 70.3634266,81.6608 70.969633,82.2624 C71.5694246,82.864 72.3841146,83.2 73.2372941,83.2 L100.253573,83.2 L163.59092,215.776 C164.123355,216.8896 165.24596,217.6 166.484032,217.6 L211.192558,217.6 C212.966274,217.6 214.4,216.1664 214.4,214.4 L214.4,176 C214.4,174.2336 212.966274,172.8 211.192558,172.8 L211.192558,172.8 Z" fill="#FFFFFF"></path>
    </g>
</svg>

```

```
{
    "headers": {
        "Accept": "image/svg+xml"
    },
    "resource": "/logo",
    "path": "/logo",
    "httpMethod": "GET"
}

```

```
{
    "body": "PD94bWwgdmVyc2lvbj0iMS4wIiBlbmNvZGluZz0iVVRGLTgiPz4KPHN2ZyB3aWR0aD0iMjU2cHgiIGhlaWdodD0iMjU2cHgiIHZpZXdCb3g9IjAgMCAyNTYgMjU2IiB2ZXJzaW9uPSIxLjEiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyIgeG1sbnM6eGxpbms9Imh0dHA6Ly93d3cudzMub3JnLzE5OTkveGxpbmsiIHByZXNlcnZlQXNwZWN0UmF0aW89InhNaWRZTWlkIj4KICAgIDx0aXRsZT5BV1MgTGFtYmRhPC90aXRsZT4KICAgIDxkZWZzPgogICAgICAgIDxsaW5lYXJHcmFkaWVudCB4MT0iMCUiIHkxPSIxMDAlIiB4Mj0iMTAwJSIgeTI9IjAlIiBpZD0ibGluZWFyR3JhZGllbnQtMSI+CiAgICAgICAgICAgIDxzdG9wIHN0b3AtY29sb3I9IiNDODUxMUIiIG9mZnNldD0iMCUiPjwvc3RvcD4KICAgICAgICAgICAgPHN0b3Agc3RvcC1jb2xvcj0iI0ZGOTkwMCIgb2Zmc2V0PSIxMDAlIj48L3N0b3A+CiAgICAgICAgPC9saW5lYXJHcmFkaWVudD4KICAgIDwvZGVmcz4KICAgIDxnPgogICAgICAgIDxyZWN0IGZpbGw9InVybCgjbGluZWFyR3JhZGllbnQtMSkiIHg9IjAiIHk9IjAiIHdpZHRoPSIyNTYiIGhlaWdodD0iMjU2Ij48L3JlY3Q+CiAgICAgICAgPHBhdGggZD0iTTg5LjYyNDExMjYsMjExLjIgTDQ5Ljg5MDMyNzcsMjExLjIgTDkzLjgzNTQ4MzIsMTE5LjM0NzIgTDExMy43NDcyOCwxNjAuMzM5MiBMODkuNjI0MTEyNiwyMTEuMiBaIE05Ni43MDI5MzU3LDExMC41Njk2IEM5Ni4xNjQwODU4LDEwOS40NjU2IDk1LjA0MTQ4MTMsMTA4Ljc2NDggOTMuODE2MjM4NCwxMDguNzY0OCBMOTMuODA2NjE2MywxMDguNzY0OCBDOTIuNTcxNzUxNCwxMDguNzY4IDkxLjQ0OTE0NjYsMTA5LjQ3NTIgOTAuOTE5OTE4NywxMTAuNTg1NiBMNDEuOTEzNDIwOCwyMTMuMDIwOCBDNDEuNDM4NzE5NywyMTQuMDEyOCA0MS41MDYwNzU4LDIxNS4xNzc2IDQyLjA5NjI0NTEsMjE2LjEwODggQzQyLjY3OTk5OTQsMjE3LjAzNjggNDMuNzA2MzgwNSwyMTcuNiA0NC44MDY1MzMxLDIxNy42IEw5MS42NTQ0MjMsMjE3LjYgQzkyLjg5NTcwMjcsMjE3LjYgOTQuMDIxNTE0OSwyMTYuODg2NCA5NC41NTM5NTAxLDIxNS43Njk2IEwxMjAuMjAzODU5LDE2MS42ODk2IEMxMjAuNjE3NjE5LDE2MC44MTI4IDEyMC42MTQ0MTIsMTU5Ljc5ODQgMTIwLjE4NzgyMiwxNTguOTI4IEw5Ni43MDI5MzU3LDExMC41Njk2IFogTTIwNy45ODUxMTcsMjExLjIgTDE2OC41MDc5MjgsMjExLjIgTDEwNS4xNzM3ODksNzguNjI0IEMxMDQuNjQ0NTYxLDc3LjUxMDQgMTAzLjUxNTU0MSw3Ni44IDEwMi4yNzc0NjksNzYuOCBMNzYuNDQ3OTQzLDc2LjggTDc2LjQ3NjgwOTksNDQuOCBMMTI3LjEwMzA2Niw0NC44IEwxOTAuMTQ1MzI4LDE3Ny4zNzI4IEMxOTAuNjc0NTU2LDE3OC40ODY0IDE5MS44MDM1NzUsMTc5LjIgMTkzLjA0MTY0NywxNzkuMiBMMjA3Ljk4NTExNywxNzkuMiBMMjA3Ljk4NTExNywyMTEuMiBaIE0yMTEuMTkyNTU4LDE3Mi44IEwxOTUuMDcxOTU4LDE3Mi44IEwxMzIuMDI5Njk2LDQwLjIyNzIgQzEzMS41MDA0NjgsMzkuMTEzNiAxMzAuMzcxNDQ5LDM4LjQgMTI5LjEzMDE2OSwzOC40IEw3My4yNzI1NzYsMzguNCBDNzEuNTA1Mjc1OCwzOC40IDcwLjA2ODM0MjEsMzkuODMwNCA3MC4wNjUxMzQ0LDQxLjU5NjggTDcwLjAyOTg1MjgsNzkuOTk2OCBDNzAuMDI5ODUyOCw4MC44NDggNzAuMzYzNDI2Niw4MS42NjA4IDcwLjk2OTYzMyw4Mi4yNjI0IEM3MS41Njk0MjQ2LDgyLjg2NCA3Mi4zODQxMTQ2LDgzLjIgNzMuMjM3Mjk0MSw4My4yIEwxMDAuMjUzNTczLDgzLjIgTDE2My41OTA5MiwyMTUuNzc2IEMxNjQuMTIzMzU1LDIxNi44ODk2IDE2NS4yNDU5NiwyMTcuNiAxNjYuNDg0MDMyLDIxNy42IEwyMTEuMTkyNTU4LDIxNy42IEMyMTIuOTY2Mjc0LDIxNy42IDIxNC40LDIxNi4xNjY0IDIxNC40LDIxNC40IEwyMTQuNCwxNzYgQzIxNC40LDE3NC4yMzM2IDIxMi45NjYyNzQsMTcyLjggMjExLjE5MjU1OCwxNzIuOCBMMjExLjE5MjU1OCwxNzIuOCBaIiBmaWxsPSIjRkZGRkZGIj48L3BhdGg+CiAgICA8L2c+Cjwvc3ZnPg==",
    "multiValueHeaders": {
        "Content-Type": ["image/svg+xml"]
    },
    "isBase64Encoded": true,
    "statusCode": 200
}

```

### Debug mode

You can enable debug mode via `debug` param, or via `POWERTOOLS_DEV` [environment variable](../../../#environment-variables).

This will enable full tracebacks errors in the response, print request and responses, and set CORS in development mode.

Danger

This might reveal sensitive information in your logs and relax CORS restrictions, use it sparingly.

It's best to use for local development only!

```
import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver(debug=True)


@app.get("/todos")
@tracer.capture_method
def get_todos():
    todos: Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    todos.raise_for_status()

    # for brevity, we'll limit to the first 10 only
    return {"todos": todos.json()[:10]}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

### OpenAPI

When you enable [Data Validation](#data-validation), we use a combination of Pydantic Models and [OpenAPI](https://www.openapis.org/) type annotations to add constraints to your API's parameters.

OpenAPI schema version depends on the installed version of Pydantic

Pydantic v1 generates [valid OpenAPI 3.0.3 schemas](https://docs.pydantic.dev/1.10/usage/schema/), and Pydantic v2 generates [valid OpenAPI 3.1.0 schemas](https://docs.pydantic.dev/latest/why/#json-schema).

In OpenAPI documentation tools like [SwaggerUI](#enabling-swaggerui), these annotations become readable descriptions, offering a self-explanatory API interface. This reduces boilerplate code while improving functionality and enabling auto-documentation.

Note

We don't have support for files, form data, and header parameters at the moment. If you're interested in this, please [open an issue](https://github.com/aws-powertools/powertools-lambda-python/issues/new?assignees=&labels=feature-request%2Ctriage&projects=&template=feature_request.yml&title=Feature+request%3A+TITLE).

#### Customizing OpenAPI parameters

Whenever you use OpenAPI parameters to validate [query strings](./#validating-query-strings) or [path parameters](./#validating-path-parameters), you can enhance validation and OpenAPI documentation by using any of these parameters:

| Field name | Type | Description | | --- | --- | --- | | `alias` | `str` | Alternative name for a field, used when serializing and deserializing data | | `validation_alias` | `str` | Alternative name for a field during validation (but not serialization) | | `serialization_alias` | `str` | Alternative name for a field during serialization (but not during validation) | | `description` | `str` | Human-readable description | | `gt` | `float` | Greater than. If set, value must be greater than this. Only applicable to numbers | | `ge` | `float` | Greater than or equal. If set, value must be greater than or equal to this. Only applicable to numbers | | `lt` | `float` | Less than. If set, value must be less than this. Only applicable to numbers | | `le` | `float` | Less than or equal. If set, value must be less than or equal to this. Only applicable to numbers | | `min_length` | `int` | Minimum length for strings | | `max_length` | `int` | Maximum length for strings | | `pattern` | `string` | A regular expression that the string must match. | | `strict` | `bool` | If `True`, strict validation is applied to the field. See [Strict Mode](https://docs.pydantic.dev/latest/concepts/strict_mode/) for details | | `multiple_of` | `float` | Value must be a multiple of this. Only applicable to numbers | | `allow_inf_nan` | `bool` | Allow `inf`, `-inf`, `nan`. Only applicable to numbers | | `max_digits` | `int` | Maximum number of allow digits for strings | | `decimal_places` | `int` | Maximum number of decimal places allowed for numbers | | `openapi_examples` | `dict[str, Example]` | A list of examples to be displayed in the SwaggerUI interface. Avoid using the `examples` field for this purpose. | | `deprecated` | `bool` | Marks the field as deprecated | | `include_in_schema` | `bool` | If `False` the field will not be part of the exported OpenAPI schema | | `json_schema_extra` | `JsonDict` | Any additional JSON schema data for the schema property |

#### Customizing API operations

Customize your API endpoints by adding metadata to endpoint definitions.

Here's a breakdown of various customizable fields:

| Field Name | Type | Description | | --- | --- | --- | | `summary` | `str` | A concise overview of the main functionality of the endpoint. This brief introduction is usually displayed in autogenerated API documentation and helps consumers quickly understand what the endpoint does. | | `description` | `str` | A more detailed explanation of the endpoint, which can include information about the operation's behavior, including side effects, error states, and other operational guidelines. | | `responses` | `Dict[int, Dict[str, OpenAPIResponse]]` | A dictionary that maps each HTTP status code to a Response Object as defined by the [OpenAPI Specification](https://swagger.io/specification/#response-object). This allows you to describe expected responses, including default or error messages, and their corresponding schemas or models for different status codes. | | `response_description` | `str` | Provides the default textual description of the response sent by the endpoint when the operation is successful. It is intended to give a human-readable understanding of the result. | | `tags` | `List[str]` | Tags are a way to categorize and group endpoints within the API documentation. They can help organize the operations by resources or other heuristic. | | `operation_id` | `str` | A unique identifier for the operation, which can be used for referencing this operation in documentation or code. This ID must be unique across all operations described in the API. | | `include_in_schema` | `bool` | A boolean value that determines whether or not this operation should be included in the OpenAPI schema. Setting it to `False` can hide the endpoint from generated documentation and schema exports, which might be useful for private or experimental endpoints. | | `deprecated` | `bool` | A boolean value that determines whether or not this operation should be marked as deprecated in the OpenAPI schema. |

To implement these customizations, include extra parameters when defining your routes:

```
import requests

from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.utilities.typing import LambdaContext

app = APIGatewayRestResolver(enable_validation=True)


@app.get(
    "/todos/<todo_id>",
    summary="Retrieves a todo item",
    description="Loads a todo item identified by the `todo_id`",
    response_description="The todo object",
    responses={
        200: {"description": "Todo item found"},
        404: {
            "description": "Item not found",
        },
    },
    tags=["Todos"],
)
def get_todo_title(todo_id: int) -> str:
    todo = requests.get(f"https://jsonplaceholder.typicode.com/todos/{todo_id}")
    todo.raise_for_status()

    return todo.json()["title"]


def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

#### Customizing OpenAPI metadata

Defining and customizing OpenAPI metadata gives detailed, top-level information about your API. Use the method `app.configure_openapi` to set and tailor this metadata:

| Field Name | Type | Description | | --- | --- | --- | | `title` | `str` | The title for your API. It should be a concise, specific name that can be used to identify the API in documentation or listings. | | `version` | `str` | The version of the API you are documenting. This could reflect the release iteration of the API and helps clients understand the evolution of the API. | | `openapi_version` | `str` | Specifies the version of the OpenAPI Specification on which your API is based. When using Pydantic v1 it defaults to 3.0.3, and when using Pydantic v2, it defaults to 3.1.0. | | `summary` | `str` | A short and informative summary that can provide an overview of what the API does. This can be the same as or different from the title but should add context or information. | | `description` | `str` | A verbose description that can include Markdown formatting, providing a full explanation of the API's purpose, functionalities, and general usage instructions. | | `tags` | `List[str]` | A collection of tags that categorize endpoints for better organization and navigation within the documentation. This can group endpoints by their functionality or other criteria. | | `servers` | `List[Server]` | An array of Server objects, which specify the URL to the server and a description for its environment (production, staging, development, etc.), providing connectivity information. | | `terms_of_service` | `str` | A URL that points to the terms of service for your API. This could provide legal information and user responsibilities related to the usage of the API. | | `contact` | `Contact` | A Contact object containing contact details of the organization or individuals maintaining the API. This may include fields such as name, URL, and email. | | `license_info` | `License` | A License object providing the license details for the API, typically including the name of the license and the URL to the full license text. |

Include extra parameters when exporting your OpenAPI specification to apply these customizations:

```
import requests

from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.event_handler.openapi.models import Contact, Server
from aws_lambda_powertools.utilities.typing import LambdaContext

app = APIGatewayRestResolver(enable_validation=True)
app.configure_openapi(
    title="TODO's API",
    version="1.21.3",
    summary="API to manage TODOs",
    description="This API implements all the CRUD operations for the TODO app",
    tags=["todos"],
    servers=[Server(url="https://stg.example.org/orders", description="Staging server")],
    contact=Contact(name="John Smith", email="john@smith.com"),
)


@app.get("/todos/<todo_id>")
def get_todo_title(todo_id: int) -> str:
    todo = requests.get(f"https://jsonplaceholder.typicode.com/todos/{todo_id}")
    todo.raise_for_status()

    return todo.json()["title"]


def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)


if __name__ == "__main__":
    print(app.get_openapi_json_schema())

```

#### Customizing Swagger UI

Customizing the Swagger metadata

The `enable_swagger` method accepts the same metadata as described at [Customizing OpenAPI metadata](#customizing-openapi-metadata).

The Swagger UI appears by default at the `/swagger` path, but you can customize this to serve the documentation from another path and specify the source for Swagger UI assets.

Below is an example configuration for serving Swagger UI from a custom path or CDN, with assets like CSS and JavaScript loading from a chosen CDN base URL.

```
from typing import List

import requests
from pydantic import BaseModel, EmailStr, Field

from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.utilities.typing import LambdaContext

app = APIGatewayRestResolver(enable_validation=True)
app.enable_swagger(path="/_swagger", swagger_base_url="https://cdn.example.com/path/to/assets/")


class Todo(BaseModel):
    userId: int
    id_: int = Field(alias="id")
    title: str
    completed: bool


@app.get("/todos")
def get_todos_by_email(email: EmailStr) -> List[Todo]:
    todos = requests.get(f"https://jsonplaceholder.typicode.com/todos?email={email}")
    todos.raise_for_status()

    return todos.json()


def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

A Middleware can handle tasks such as adding security headers, user authentication, or other request processing for serving the Swagger UI.

```
from typing import List

import requests
from pydantic import BaseModel, EmailStr, Field

from aws_lambda_powertools.event_handler import APIGatewayRestResolver, Response
from aws_lambda_powertools.event_handler.middlewares import NextMiddleware
from aws_lambda_powertools.utilities.typing import LambdaContext

app = APIGatewayRestResolver(enable_validation=True)


def swagger_middleware(app: APIGatewayRestResolver, next_middleware: NextMiddleware) -> Response:
    is_authenticated = ...
    if not is_authenticated:
        return Response(status_code=400, body="Unauthorized")

    return next_middleware(app)


app.enable_swagger(middlewares=[swagger_middleware])


class Todo(BaseModel):
    userId: int
    id_: int = Field(alias="id")
    title: str
    completed: bool


@app.get("/todos")
def get_todos_by_email(email: EmailStr) -> List[Todo]:
    todos = requests.get(f"https://jsonplaceholder.typicode.com/todos?email={email}")
    todos.raise_for_status()

    return todos.json()


def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

#### Security schemes

Does Powertools implement any of the security schemes?

No. Powertools adds support for generating OpenAPI documentation with [security schemes](https://swagger.io/docs/specification/authentication/), but it doesn't implement any of the security schemes itself, so you must implement the security mechanisms separately.

Security schemes are declared at the top-level first. You can reference them globally or on a per path *(operation)* level. **However**, if you reference security schemes that are not defined at the top-level it will lead to a `SchemaValidationError` *(invalid OpenAPI spec)*.

```
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import (
    APIGatewayRestResolver,
)
from aws_lambda_powertools.event_handler.openapi.models import (
    OAuth2,
    OAuthFlowAuthorizationCode,
    OAuthFlows,
)

tracer = Tracer()
logger = Logger()

app = APIGatewayRestResolver(enable_validation=True)
app.configure_openapi(
    title="My API",
    security_schemes={
        "oauth": OAuth2(
            flows=OAuthFlows(
                authorizationCode=OAuthFlowAuthorizationCode(
                    authorizationUrl="https://xxx.amazoncognito.com/oauth2/authorize",
                    tokenUrl="https://xxx.amazoncognito.com/oauth2/token",
                ),
            ),
        ),
    },
    security=[{"oauth": ["admin"]}],  # (1)!)
)


@app.get("/")
def helloworld() -> dict:
    return {"hello": "world"}


@logger.inject_lambda_context
@tracer.capture_lambda_handler
def lambda_handler(event, context):
    return app.resolve(event, context)


if __name__ == "__main__":
    print(app.get_openapi_json_schema())

```

1. Using the oauth security scheme defined earlier, scoped to the "admin" role.

```
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import (
    APIGatewayRestResolver,
)
from aws_lambda_powertools.event_handler.openapi.models import (
    OAuth2,
    OAuthFlowAuthorizationCode,
    OAuthFlows,
)

tracer = Tracer()
logger = Logger()

app = APIGatewayRestResolver(enable_validation=True)
app.configure_openapi(
    title="My API",
    security_schemes={
        "oauth": OAuth2(
            flows=OAuthFlows(
                authorizationCode=OAuthFlowAuthorizationCode(
                    authorizationUrl="https://xxx.amazoncognito.com/oauth2/authorize",
                    tokenUrl="https://xxx.amazoncognito.com/oauth2/token",
                ),
            ),
        ),
    },
)


@app.get("/", security=[{"oauth": ["admin"]}])  # (1)!
def helloworld() -> dict:
    return {"hello": "world"}


@logger.inject_lambda_context
@tracer.capture_lambda_handler
def lambda_handler(event, context):
    return app.resolve(event, context)


if __name__ == "__main__":
    print(app.get_openapi_json_schema())

```

1. Using the oauth security scheme defined bellow, scoped to the "admin" role.

```
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import (
    APIGatewayRestResolver,
)
from aws_lambda_powertools.event_handler.openapi.models import (
    OAuth2,
    OAuthFlowAuthorizationCode,
    OAuthFlows,
)

tracer = Tracer()
logger = Logger()

app = APIGatewayRestResolver(enable_validation=True)
app.configure_openapi(
    title="My API",
    security_schemes={
        "oauth": OAuth2(
            flows=OAuthFlows(
                authorizationCode=OAuthFlowAuthorizationCode(
                    authorizationUrl="https://xxx.amazoncognito.com/oauth2/authorize",
                    tokenUrl="https://xxx.amazoncognito.com/oauth2/token",
                ),
            ),
        ),
    },
)


@app.get("/protected", security=[{"oauth": ["admin"]}])
def protected() -> dict:
    return {"hello": "world"}


@app.get("/unprotected", security=[{}])  # (1)!
def unprotected() -> dict:
    return {"hello": "world"}


@logger.inject_lambda_context
@tracer.capture_lambda_handler
def lambda_handler(event, context):
    return app.resolve(event, context)


if __name__ == "__main__":
    print(app.get_openapi_json_schema())

```

1. To make security optional in a specific route, an empty security requirement ({}) can be included in the array.

OpenAPI 3 lets you describe APIs protected using the following security schemes:

| Security Scheme | Type | Description | | --- | --- | --- | | [HTTP auth](https://www.iana.org/assignments/http-authschemes/http-authschemes.xhtml) | `HTTPBase` | HTTP authentication schemes using the Authorization header (e.g: [Basic auth](https://swagger.io/docs/specification/authentication/basic-authentication/), [Bearer](https://swagger.io/docs/specification/authentication/bearer-authentication/)) | | [API keys](https://swagger.io/docs/specification/authentication/api-keys/) (e.g: query strings, cookies) | `APIKey` | API keys in headers, query strings or [cookies](https://swagger.io/docs/specification/authentication/cookie-authentication/). | | [OAuth 2](https://swagger.io/docs/specification/authentication/oauth2/) | `OAuth2` | Authorization protocol that gives an API client limited access to user data on a web server. | | [OpenID Connect Discovery](https://swagger.io/docs/specification/authentication/openid-connect-discovery/) | `OpenIdConnect` | Identity layer built [on top of the OAuth 2.0 protocol](https://openid.net/developers/how-connect-works/) and supported by some OAuth 2.0. | | [Mutual TLS](https://swagger.io/specification/#security-scheme-object). | `MutualTLS` | Client/server certificate mutual authentication scheme. |

Using OAuth2 with the Swagger UI?

You can use the `OAuth2Config` option to configure a default OAuth2 app on the generated Swagger UI.

```
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import (
    APIGatewayRestResolver,
)
from aws_lambda_powertools.event_handler.openapi.models import (
    OAuth2,
    OAuthFlowAuthorizationCode,
    OAuthFlows,
)
from aws_lambda_powertools.event_handler.openapi.swagger_ui import OAuth2Config

tracer = Tracer()
logger = Logger()

oauth2 = OAuth2Config(
    client_id="xxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    app_name="OAuth2 app",
)

app = APIGatewayRestResolver(enable_validation=True)
app.enable_swagger(
    oauth2_config=oauth2,
    security_schemes={
        "oauth": OAuth2(
            flows=OAuthFlows(
                authorizationCode=OAuthFlowAuthorizationCode(
                    authorizationUrl="https://xxx.amazoncognito.com/oauth2/authorize",
                    tokenUrl="https://xxx.amazoncognito.com/oauth2/token",
                ),
            ),
        ),
    },
    security=[{"oauth": []}],
)


@app.get("/")
def hello() -> str:
    return "world"


@logger.inject_lambda_context
@tracer.capture_lambda_handler
def lambda_handler(event, context):
    return app.resolve(event, context)

```

#### OpenAPI extensions

For a better experience when working with Lambda and Amazon API Gateway, customers can define extensions using the `openapi_extensions` parameter. We support defining OpenAPI extensions at the following levels of the OpenAPI JSON Schema: Root, Servers, Operation, and Security Schemes.

Warning

We do not support the `x-amazon-apigateway-any-method` and `x-amazon-apigateway-integrations` extensions.

```
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.event_handler.openapi.models import APIKey, APIKeyIn, Server

app = APIGatewayRestResolver(enable_validation=True)

servers = Server(
    url="http://example.com",
    description="Example server",
    openapi_extensions={"x-amazon-apigateway-endpoint-configuration": {"vpcEndpoint": "myendpointid"}},  # (1)!
)


@app.get(
    "/hello",
    openapi_extensions={"x-amazon-apigateway-integration": {"type": "aws", "uri": "my_lambda_arn"}},  # (2)!
)
def hello():
    return app.get_openapi_json_schema(
        servers=[servers],
        security_schemes={
            "apikey": APIKey(
                name="X-API-KEY",
                description="API KeY",
                in_=APIKeyIn.header,
                openapi_extensions={"x-amazon-apigateway-authorizer": "custom"},  # (3)!
            ),
        },
        openapi_extensions={"x-amazon-apigateway-gateway-responses": {"DEFAULT_4XX"}},  # (4)!
    )


def lambda_handler(event, context):
    return app.resolve(event, context)

```

1. Server level
1. Operation level
1. Security scheme level
1. Root level

### Custom serializer

You can instruct event handler to use a custom serializer to best suit your needs, for example take into account Enums when serializing.

```
import json
from dataclasses import asdict, dataclass, is_dataclass
from json import JSONEncoder

import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver()


@dataclass
class Todo:
    userId: str
    id: str  # noqa: A003 VNE003 "id" field is reserved
    title: str
    completed: bool


class DataclassCustomEncoder(JSONEncoder):
    """A custom JSON encoder to serialize dataclass obj"""

    def default(self, obj):
        # Only called for values that aren't JSON serializable
        # where `obj` will be an instance of Todo in this example
        return asdict(obj) if is_dataclass(obj) else super().default(obj)


def custom_serializer(obj) -> str:
    """Your custom serializer function APIGatewayRestResolver will use"""
    return json.dumps(obj, separators=(",", ":"), cls=DataclassCustomEncoder)


app = APIGatewayRestResolver(serializer=custom_serializer)


@app.get("/todos")
@tracer.capture_method
def get_todos():
    ret: Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    ret.raise_for_status()
    todos = [Todo(**todo) for todo in ret.json()]

    # for brevity, we'll limit to the first 10 only
    return {"todos": todos[:10]}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

### Split routes with Router

As you grow the number of routes a given Lambda function should handle, it is natural to either break into smaller Lambda functions, or split routes into separate files to ease maintenance - that's where the `Router` feature is useful.

Let's assume you have `split_route.py` as your Lambda function entrypoint and routes in `split_route_module.py`. This is how you'd use the `Router` feature.

We import **Router** instead of **APIGatewayRestResolver**; syntax wise is exactly the same.

Info

This means all methods, including [middleware](#middleware) will work as usual.

```
from urllib.parse import quote

import requests
from requests import Response

from aws_lambda_powertools import Tracer
from aws_lambda_powertools.event_handler.api_gateway import Router

tracer = Tracer()
router = Router()

endpoint = "https://jsonplaceholder.typicode.com/todos"


@router.get("/todos")
@tracer.capture_method
def get_todos():
    api_key = router.current_event.headers["X-Api-Key"]

    todos: Response = requests.get(endpoint, headers={"X-Api-Key": api_key})
    todos.raise_for_status()

    # for brevity, we'll limit to the first 10 only
    return {"todos": todos.json()[:10]}


@router.get("/todos/<todo_id>")
@tracer.capture_method
def get_todo_by_id(todo_id: str):  # value come as str
    api_key = router.current_event.headers["X-Api-Key"]

    todo_id = quote(todo_id, safe="")
    todos: Response = requests.get(f"{endpoint}/{todo_id}", headers={"X-Api-Key": api_key})
    todos.raise_for_status()

    return {"todos": todos.json()}

```

We use `include_router` method and include all user routers registered in the `router` global object.

Note

This method merges routes, [context](#sharing-contextual-data) and [middleware](#middleware) from `Router` into the main resolver instance (`APIGatewayRestResolver()`).

```
import split_route_module

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver()
app.include_router(split_route_module.router)  # (1)!


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

1. When using [middleware](#middleware) in both `Router` and main resolver, you can make `Router` middlewares to take precedence by using `include_router` before `app.use()`.

#### Route prefix

In the previous example, `split_route_module.py` routes had a `/todos` prefix. This might grow over time and become repetitive.

When necessary, you can set a prefix when including a router object. This means you could remove `/todos` prefix altogether.

```
import split_route_module

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver()
# prefix '/todos' to any route in `split_route_module.router`
app.include_router(split_route_module.router, prefix="/todos")


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

```
from urllib.parse import quote

import requests
from requests import Response

from aws_lambda_powertools import Tracer
from aws_lambda_powertools.event_handler.api_gateway import Router

tracer = Tracer()
router = Router()

endpoint = "https://jsonplaceholder.typicode.com/todos"


@router.get("/")
@tracer.capture_method
def get_todos():
    api_key = router.current_event.headers["X-Api-Key"]

    todos: Response = requests.get(endpoint, headers={"X-Api-Key": api_key})
    todos.raise_for_status()

    # for brevity, we'll limit to the first 10 only
    return {"todos": todos.json()[:10]}


@router.get("/<todo_id>")
@tracer.capture_method
def get_todo_by_id(todo_id: str):  # value come as str
    api_key = router.current_event.headers["X-Api-Key"]

    todo_id = quote(todo_id, safe="")
    todos: Response = requests.get(f"{endpoint}/{todo_id}", headers={"X-Api-Key": api_key})
    todos.raise_for_status()

    return {"todos": todos.json()}


# many more routes

```

#### Specialized router types

You can use specialized router classes according to the type of event that you are resolving. This way you'll get type hints from your IDE as you access the `current_event` property.

| Router | Resolver | `current_event` type | | --- | --- | --- | | APIGatewayRouter | APIGatewayRestResolver | APIGatewayProxyEvent | | APIGatewayHttpRouter | APIGatewayHttpResolver | APIGatewayProxyEventV2 | | ALBRouter | ALBResolver | ALBEvent | | LambdaFunctionUrlRouter | LambdaFunctionUrlResolver | LambdaFunctionUrlEvent |

```
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.event_handler.router import APIGatewayRouter

app = APIGatewayRestResolver()
router = APIGatewayRouter()


@router.get("/me")
def get_self():
    # router.current_event is a APIGatewayProxyEvent
    account_id = router.current_event.request_context.account_id

    return {"account_id": account_id}


app.include_router(router)


def lambda_handler(event, context):
    return app.resolve(event, context)

```

#### Sharing contextual data

You can use `append_context` when you want to share data between your App and Router instances. Any data you share will be available via the `context` dictionary available in your App or Router context.

We always clear data available in `context` after each invocation.

This can be useful for middlewares injecting contextual information before a request is processed.

```
import split_route_append_context_module

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver()
app.include_router(split_route_append_context_module.router)


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    app.append_context(is_admin=True)  # arbitrary number of key=value data
    return app.resolve(event, context)

```

```
import requests
from requests import Response

from aws_lambda_powertools import Tracer
from aws_lambda_powertools.event_handler.api_gateway import Router

tracer = Tracer()
router = Router()

endpoint = "https://jsonplaceholder.typicode.com/todos"


@router.get("/todos")
@tracer.capture_method
def get_todos():
    is_admin: bool = router.context.get("is_admin", False)
    todos = {}

    if is_admin:
        todos: Response = requests.get(endpoint)
        todos.raise_for_status()
        todos = todos.json()[:10]

    # for brevity, we'll limit to the first 10 only
    return {"todos": todos}

```

#### Sample layout

This is a sample project layout for a monolithic function with routes split in different files (`/todos`, `/health`).

```
.
├── pyproject.toml            # project app & dev dependencies; poetry, pipenv, etc.
├── poetry.lock
├── src
│       ├── __init__.py
│       ├── requirements.txt  # sam build detect it automatically due to CodeUri: src. poetry export --format src/requirements.txt
│       └── todos
│           ├── __init__.py
│           ├── main.py       # this will be our todos Lambda fn; it could be split in folders if we want separate fns same code base
│           └── routers       # routers module
│               ├── __init__.py
│               ├── health.py # /health routes. from routers import todos; health.router
│               └── todos.py  # /todos routes. from .routers import todos; todos.router
├── template.yml              # SAM. CodeUri: src, Handler: todos.main.lambda_handler
└── tests
    ├── __init__.py
    ├── unit
    │   ├── __init__.py
    │   └── test_todos.py     # unit tests for the todos router
    │   └── test_health.py    # unit tests for the health router
    └── functional
        ├── __init__.py
        ├── conftest.py       # pytest fixtures for the functional tests
        └── test_main.py      # functional tests for the main lambda handler

```

### Considerations

This utility is optimized for fast startup, minimal feature set, and to quickly on-board customers familiar with frameworks like Flask — it's not meant to be a fully fledged framework.

Event Handler naturally leads to a single Lambda function handling multiple routes for a given service, which can be eventually broken into multiple functions.

Both single (monolithic) and multiple functions (micro) offer different set of trade-offs worth knowing.

Tip

TL;DR. Start with a monolithic function, add additional functions with new handlers, and possibly break into micro functions if necessary.

#### Monolithic function

A monolithic function means that your final code artifact will be deployed to a single function. This is generally the best approach to start.

***Benefits***

- **Code reuse**. It's easier to reason about your service, modularize it and reuse code as it grows. Eventually, it can be turned into a standalone library.
- **No custom tooling**. Monolithic functions are treated just like normal Python packages; no upfront investment in tooling.
- **Faster deployment and debugging**. Whether you use all-at-once, linear, or canary deployments, a monolithic function is a single deployable unit. IDEs like PyCharm and VSCode have tooling to quickly profile, visualize, and step through debug any Python package.

***Downsides***

- **Cold starts**. Frequent deployments and/or high load can diminish the benefit of monolithic functions depending on your latency requirements, due to [Lambda scaling model](https://docs.aws.amazon.com/lambda/latest/dg/invocation-scaling.html). Always load test to pragmatically balance between your customer experience and development cognitive load.
- **Granular security permissions**. The micro function approach enables you to use fine-grained permissions & access controls, separate external dependencies & code signing at the function level. Conversely, you could have multiple functions while duplicating the final code artifact in a monolithic approach.
  - Regardless, least privilege can be applied to either approaches.
- **Higher risk per deployment**. A misconfiguration or invalid import can cause disruption if not caught earlier in automated testing. Multiple functions can mitigate misconfigurations but they would still share the same code artifact. You can further minimize risks with multiple environments in your CI/CD pipeline.

#### Micro function

A micro function means that your final code artifact will be different to each function deployed. This is generally the approach to start if you're looking for fine-grain control and/or high load on certain parts of your service.

**Benefits**

- **Granular scaling**. A micro function can benefit from the [Lambda scaling model](https://docs.aws.amazon.com/lambda/latest/dg/invocation-scaling.html) to scale differently depending on each part of your application. Concurrency controls and provisioned concurrency can also be used at a granular level for capacity management.
- **Discoverability**. Micro functions are easier to visualize when using distributed tracing. Their high-level architectures can be self-explanatory, and complexity is highly visible — assuming each function is named to the business purpose it serves.
- **Package size**. An independent function can be significant smaller (KB vs MB) depending on external dependencies it require to perform its purpose. Conversely, a monolithic approach can benefit from [Lambda Layers](https://docs.aws.amazon.com/lambda/latest/dg/invocation-layers.html) to optimize builds for external dependencies.

**Downsides**

- **Upfront investment**. You need custom build tooling to bundle assets, including [C bindings for runtime compatibility](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html). Operations become more elaborate — you need to standardize tracing labels/annotations, structured logging, and metrics to pinpoint root causes.
  - Engineering discipline is necessary for both approaches. Micro-function approach however requires further attention in consistency as the number of functions grow, just like any distributed system.
- **Harder to share code**. Shared code must be carefully evaluated to avoid unnecessary deployments when that changes. Equally, if shared code isn't a library, your development, building, deployment tooling need to accommodate the distinct layout.
- **Slower safe deployments**. Safely deploying multiple functions require coordination — AWS CodeDeploy deploys and verifies each function sequentially. This increases lead time substantially (minutes to hours) depending on the deployment strategy you choose. You can mitigate it by selectively enabling it in prod-like environments only, and where the risk profile is applicable.
  - Automated testing, operational and security reviews are essential to stability in either approaches.

**Example**

Consider a simplified micro function structured REST API that has two routes:

- `/users` - an endpoint that will return all users of the application on `GET` requests
- `/users/<id>` - an endpoint that looks up a single users details by ID on `GET` requests

Each endpoint will be it's own Lambda function that is configured as a [Lambda integration](https://docs.aws.amazon.com/apigateway/latest/developerguide/getting-started-with-lambda-integration.html). This allows you to set different configurations for each lambda (memory size, layers, etc.).

```
import json
from dataclasses import dataclass
from http import HTTPStatus

from aws_lambda_powertools import Logger
from aws_lambda_powertools.event_handler import APIGatewayRestResolver, Response
from aws_lambda_powertools.utilities.typing import LambdaContext

logger = Logger()

# This would likely be a db lookup
users = [
    {
        "user_id": "b0b2a5bf-ee1e-4c5e-9a86-91074052739e",
        "email": "john.doe@example.com",
        "active": True,
    },
    {
        "user_id": "3a9df6b1-938c-4e80-bd4a-0c966f4b1c1e",
        "email": "jane.smith@example.com",
        "active": False,
    },
    {
        "user_id": "aa0d3d09-9cb9-42b9-9e63-1fb17ea52981",
        "email": "alex.wilson@example.com",
        "active": True,
    },
]


@dataclass
class User:
    user_id: str
    email: str
    active: bool


app = APIGatewayRestResolver()


@app.get("/users")
def all_active_users():
    """HTTP Response for all active users"""
    all_users = [User(**user) for user in users]
    all_active_users = [user.__dict__ for user in all_users if user.active]

    return Response(
        status_code=HTTPStatus.OK.value,
        content_type="application/json",
        body=json.dumps(all_active_users),
    )


@logger.inject_lambda_context()
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

```
import json
from dataclasses import dataclass
from http import HTTPStatus
from typing import Union

from aws_lambda_powertools import Logger
from aws_lambda_powertools.event_handler import APIGatewayRestResolver, Response
from aws_lambda_powertools.utilities.typing import LambdaContext

logger = Logger()

# This would likely be a db lookup
users = [
    {
        "user_id": "b0b2a5bf-ee1e-4c5e-9a86-91074052739e",
        "email": "john.doe@example.com",
        "active": True,
    },
    {
        "user_id": "3a9df6b1-938c-4e80-bd4a-0c966f4b1c1e",
        "email": "jane.smith@example.com",
        "active": False,
    },
    {
        "user_id": "aa0d3d09-9cb9-42b9-9e63-1fb17ea52981",
        "email": "alex.wilson@example.com",
        "active": True,
    },
]


@dataclass
class User:
    user_id: str
    email: str
    active: bool


def get_user_by_id(user_id: str) -> Union[User, None]:
    for user_data in users:
        if user_data["user_id"] == user_id:
            return User(
                user_id=str(user_data["user_id"]),
                email=str(user_data["email"]),
                active=bool(user_data["active"]),
            )

    return None


app = APIGatewayRestResolver()


@app.get("/users/<user_id>")
def all_active_users(user_id: str):
    """HTTP Response for all active users"""
    user = get_user_by_id(user_id)

    if user:
        return Response(
            status_code=HTTPStatus.OK.value,
            content_type="application/json",
            body=json.dumps(user.__dict__),
        )

    else:
        return Response(status_code=HTTPStatus.NOT_FOUND)


@logger.inject_lambda_context()
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

```
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31
Description: >
    micro-function-example

Globals:
    Api:
        TracingEnabled: true
        Cors: # see CORS section
            AllowOrigin: "'https://example.com'"
            AllowHeaders: "'Content-Type,Authorization,X-Amz-Date'"
            MaxAge: "'300'"
        BinaryMediaTypes: # see Binary responses section
            - "*~1*" # converts to */* for any binary type
            # NOTE: use this stricter version if you're also using CORS; */* doesn't work with CORS
            # see: https://github.com/aws-powertools/powertools-lambda-python/issues/3373#issuecomment-1821144779
            # - "image~1*" # converts to image/*
            # - "*~1csv" # converts to */csv, eg text/csv, application/csv

    Function:
        Timeout: 5
        Runtime: python3.12

Resources:
    # Lambda Function Solely For /users endpoint
    AllUsersFunction:
        Type: AWS::Serverless::Function
        Properties:
            Handler: app.lambda_handler
            CodeUri: users
            Description: Function for /users endpoint
            Architectures:
                - x86_64
            Tracing: Active
            Events:
                UsersPath:
                    Type: Api
                    Properties:
                        Path: /users
                        Method: GET
            MemorySize: 128 # Each Lambda Function can have it's own memory configuration
            Environment:
                Variables:
                    POWERTOOLS_LOG_LEVEL: INFO
            Tags:
                LambdaPowertools: python

    # Lambda Function Solely For /users/{id} endpoint
    UserByIdFunction:
        Type: AWS::Serverless::Function
        Properties:
            Handler: app.lambda_handler
            CodeUri: users_by_id
            Description: Function for /users/{id} endpoint
            Architectures:
                - x86_64
            Tracing: Active
            Events:
                UsersByIdPath:
                    Type: Api
                    Properties:
                        Path: /users/{id+}
                        Method: GET
            MemorySize: 128 # Each Lambda Function can have it's own memory configuration
            Environment:
                Variables:
                    POWERTOOLS_LOG_LEVEL: INFO

```

Note

You can see some of the downsides in this example such as some code reuse. If set up with proper build tooling, the `User` class could be shared across functions. This could be accomplished by packaging shared code as a [Lambda Layer](https://docs.aws.amazon.com/lambda/latest/dg/chapter-layers.html) or [Pants](https://www.pantsbuild.org/docs/awslambda-python).

## Testing your code

You can test your routes by passing a proxy event request with required params.

```
from dataclasses import dataclass

import assert_rest_api_resolver_response
import pytest


@dataclass
class LambdaContext:
    function_name: str = "test"
    memory_limit_in_mb: int = 128
    invoked_function_arn: str = "arn:aws:lambda:eu-west-1:123456789012:function:test"
    aws_request_id: str = "da658bd3-2d6f-4e7b-8ec2-937234644fdc"


@pytest.fixture
def lambda_context() -> LambdaContext:
    return LambdaContext()


def test_lambda_handler(lambda_context):
    minimal_event = {
        "path": "/todos",
        "httpMethod": "GET",
        "requestContext": {"requestId": "227b78aa-779d-47d4-a48e-ce62120393b8"},  # correlation ID
    }
    # Example of API Gateway REST API request event:
    # https://docs.aws.amazon.com/lambda/latest/dg/services-apigateway.html#apigateway-example-event
    ret = assert_rest_api_resolver_response.lambda_handler(minimal_event, lambda_context)
    assert ret["statusCode"] == 200
    assert ret["body"] != ""

```

```
import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver()


@app.get("/todos")
@tracer.capture_method
def get_todos():
    todos: Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    todos.raise_for_status()

    return {"todos": todos.json()[:10]}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

```
from dataclasses import dataclass

import assert_http_api_response_module
import pytest


@dataclass
class LambdaContext:
    function_name: str = "test"
    memory_limit_in_mb: int = 128
    invoked_function_arn: str = "arn:aws:lambda:eu-west-1:123456789012:function:test"
    aws_request_id: str = "da658bd3-2d6f-4e7b-8ec2-937234644fdc"


@pytest.fixture
def lambda_context() -> LambdaContext:
    return LambdaContext()


def test_lambda_handler(lambda_context: LambdaContext):
    minimal_event = {
        "rawPath": "/todos",
        "requestContext": {
            "requestContext": {"requestId": "227b78aa-779d-47d4-a48e-ce62120393b8"},  # correlation ID
            "http": {
                "method": "GET",
            },
            "stage": "$default",
        },
    }
    # Example of API Gateway HTTP API request event:
    # https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-develop-integrations-lambda.html

    ret = assert_http_api_response_module.lambda_handler(minimal_event, lambda_context)
    assert ret["statusCode"] == 200
    assert ret["body"] != ""

```

```
import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayHttpResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = APIGatewayHttpResolver()


@app.get("/todos")
@tracer.capture_method
def get_todos():
    todos: Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    todos.raise_for_status()

    return {"todos": todos.json()[:10]}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_HTTP)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

```
from dataclasses import dataclass

import assert_alb_api_response_module
import pytest


@dataclass
class LambdaContext:
    function_name: str = "test"
    memory_limit_in_mb: int = 128
    invoked_function_arn: str = "arn:aws:lambda:eu-west-1:123456789012:function:test"
    aws_request_id: str = "da658bd3-2d6f-4e7b-8ec2-937234644fdc"


@pytest.fixture
def lambda_context() -> LambdaContext:
    return LambdaContext()


def test_lambda_handler(lambda_context: LambdaContext):
    minimal_event = {
        "path": "/todos",
        "httpMethod": "GET",
        "headers": {"x-amzn-trace-id": "b25827e5-0e30-4d52-85a8-4df449ee4c5a"},
    }
    # Example of Application Load Balancer request event:
    # https://docs.aws.amazon.com/lambda/latest/dg/services-alb.html

    ret = assert_alb_api_response_module.lambda_handler(minimal_event, lambda_context)
    assert ret["statusCode"] == 200
    assert ret["body"] != ""

```

```
import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import ALBResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = ALBResolver()


@app.get("/todos")
@tracer.capture_method
def get_todos():
    todos: Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    todos.raise_for_status()

    return {"todos": todos.json()[:10]}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.APPLICATION_LOAD_BALANCER)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

```
from dataclasses import dataclass

import assert_function_url_api_response_module
import pytest


@dataclass
class LambdaContext:
    function_name: str = "test"
    memory_limit_in_mb: int = 128
    invoked_function_arn: str = "arn:aws:lambda:eu-west-1:123456789012:function:test"
    aws_request_id: str = "da658bd3-2d6f-4e7b-8ec2-937234644fdc"


@pytest.fixture
def lambda_context() -> LambdaContext:
    return LambdaContext()


def test_lambda_handler(lambda_context: LambdaContext):
    minimal_event = {
        "rawPath": "/todos",
        "requestContext": {
            "requestContext": {"requestId": "227b78aa-779d-47d4-a48e-ce62120393b8"},  # correlation ID
            "http": {
                "method": "GET",
            },
            "stage": "$default",
        },
    }
    # Example of Lambda Function URL request event:
    # https://docs.aws.amazon.com/lambda/latest/dg/urls-invocation.html#urls-payloads

    ret = assert_function_url_api_response_module.lambda_handler(minimal_event, lambda_context)
    assert ret["statusCode"] == 200
    assert ret["body"] != ""

```

```
import requests
from requests import Response

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import LambdaFunctionUrlResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.utilities.typing import LambdaContext

tracer = Tracer()
logger = Logger()
app = LambdaFunctionUrlResolver()


@app.get("/todos")
@tracer.capture_method
def get_todos():
    todos: Response = requests.get("https://jsonplaceholder.typicode.com/todos")
    todos.raise_for_status()

    return {"todos": todos.json()[:10]}


# You can continue to use other utilities just as before
@logger.inject_lambda_context(correlation_id_path=correlation_paths.LAMBDA_FUNCTION_URL)
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)

```

## FAQ

**What's the difference between this utility and frameworks like Chalice?**

Chalice is a full featured microframework that manages application and infrastructure. This utility, however, is largely focused on routing to reduce boilerplate and expects you to setup and manage infrastructure with your framework of choice.

That said, [Chalice has native integration with Lambda Powertools](https://aws.github.io/chalice/topics/middleware.html) if you're looking for a more opinionated and web framework feature set.

**What happened to `ApiGatewayResolver`?**

It's been superseded by more explicit resolvers like `APIGatewayRestResolver`, `APIGatewayHttpResolver`, and `ALBResolver`.

`ApiGatewayResolver` handled multiple types of event resolvers for convenience via `proxy_type` param. However, it made it impossible for static checkers like Mypy and IDEs IntelliSense to know what properties a `current_event` would have due to late bound resolution.

This provided a suboptimal experience for customers not being able to find all properties available besides common ones between API Gateway REST, HTTP, and ALB - while manually annotating `app.current_event` would work it is not the experience we want to provide to customers.

`ApiGatewayResolver` will be deprecated in v2 and have appropriate warnings as soon as we have a v2 draft.
