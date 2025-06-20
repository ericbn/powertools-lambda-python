Powertools for AWS Lambda (Python) is a developer toolkit to implement Serverless best practices and increase developer velocity.

- **Features**

  ______________________________________________________________________

  Adopt one, a few, or all industry practices. **Progressively**.

  [All features](#features)

- **Support this project**

  ______________________________________________________________________

  Become a public reference customer, share your work, contribute, use Lambda Layers, etc.

  [Support](#support-powertools-for-aws-lambda-python)

- **Available languages**

  ______________________________________________________________________

  Powertools for AWS Lambda is also available in other languages

  [Java](https://docs.powertools.aws.dev/lambda/java/), [TypeScript](https://docs.powertools.aws.dev/lambda/typescript/latest/), and [.NET](https://docs.powertools.aws.dev/lambda/dotnet/)

## Install

You can install Powertools for AWS Lambda (Python) using your favorite dependency management, or Lambda Layers:

Most features use Python standard library and the AWS SDK *(boto3)* that are available in the AWS Lambda runtime.

- **pip**: **`pip install "aws-lambda-powertools"`**
- **poetry**: **`poetry add "aws-lambda-powertools"`**
- **pdm**: **`pdm add "aws-lambda-powertools"`**

### Extra dependencies

However, you will need additional dependencies if you are using any of the features below:

| Feature | Install | Default dependency | | --- | --- | --- | | **[Tracer](core/tracer/#install)** | **`pip install "aws-lambda-powertools[tracer]"`** | `aws-xray-sdk` | | **[Validation](utilities/validation/#install)** | **`pip install "aws-lambda-powertools[validation]"`** | `fastjsonschema` | | **[Parser](utilities/parser/#install)** | **`pip install "aws-lambda-powertools[parser]"`** | `pydantic` *(v2)* | | **[Data Masking](utilities/data_masking/#install)** | **`pip install "aws-lambda-powertools[datamasking]"`** | `aws-encryption-sdk`, `jsonpath-ng` | | **All extra dependencies at once** | **`pip install "aws-lambda-powertools[all]"`** | | | **Two or more extra dependencies only, not all** | **`pip install "aws-lambda-powertools[tracer,parser,datamasking]"`** | |

[Lambda Layer](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html) is a .zip file archive that can contain additional code, pre-packaged dependencies, data, or configuration files. We compile and optimize [all dependencies](#install), and remove duplicate dependencies [already available in the Lambda runtime](https://github.com/aws-powertools/powertools-lambda-python/blob/develop/layer_v3/docker/Dockerfile#L34) to achieve the most optimal size.

For the latter, make sure to replace `{region}` with your AWS region, e.g., `eu-west-1`, and the `{python_version}` without the period (.), e.g., `python313` for `Python 3.13`.

| Architecture | Layer ARN | | --- | --- | | x86_64 | **arn:aws:lambda:{region}:017000801446:layer:AWSLambdaPowertoolsPythonV3-{python_version}-x86_64:7** | | ARM | **arn:aws:lambda:{region}:017000801446:layer:AWSLambdaPowertoolsPythonV3-{python_version}-arm64:7** |

You can add our layer using the [AWS Lambda Console *(direct link)*](https://console.aws.amazon.com/lambda/home#/add/layer):

- Under Layers, choose `AWS layers` or `Specify an ARN`
- Click to copy the [correct ARN](#lambda-layer) value based on your AWS Lambda function architecture and region

We offer Parameter Store aliases for releases too, allowing you to specify either specific versions or use the latest version on every deploy. To use these you can add these snippets to your AWS CloudFormation or Terraform projects:

**CloudFormation**

Sample Placeholders:

- `{arch}` is either `arm64` (Graviton based functions) or `x86_64`
- `{python_version}` is the Python runtime version, e.g., `python3.13` for `Python 3.13`.
- `{version}` is the semantic version number (e,g. 3.1.0) for a release or `latest`

```
MyFunction:
    Type: "AWS::Lambda::Function"
    Properties:
        ...
        Layers:
        - {{resolve:ssm:/aws/service/powertools/python/{arch}/{python_version}/{version}}}

```

**Terraform**

Using the [`aws_ssm_parameter`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ssm_parameter) data provider from the AWS Terraform provider allows you to lookup the value of parameters to use later in your project.

```
data "aws_ssm_parameter" "powertools_version" {
    name = "/aws/service/powertools/python/{arch}/{python_version}/{version}"
}

resource "aws_lambda_function" "test_lambda" {
    ...

    runtime = "python3.13"

    layers = [data.aws_ssm_parameter.powertools_version.value]
}

```

> Are we missing a framework? please create [a documentation request](https://github.com/aws-powertools/powertools-lambda-python/issues/new?assignees=&labels=documentation%2Ctriage&projects=&template=documentation_improvements.yml&title=Docs%3A+TITLE).

Thanks to the community, we've covered most popular frameworks on how to add a Lambda Layer to an existing function.

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
    MyLambdaFunction:
        Type: AWS::Serverless::Function
        Properties:
            Runtime: python3.12
            Handler: app.lambda_handler
            Layers:
            - !Sub arn:aws:lambda:${AWS::Region}:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18

```

```
service: powertools-lambda

provider:
  name: aws
  runtime: python3.12
  region: us-east-1

functions:
  powertools:
    handler: lambda_function.lambda_handler
    architecture: arm64
    layers:
    - arn:aws:lambda:${aws:region}:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18

```

```
from aws_cdk import Aws, Stack, aws_lambda
from constructs import Construct


class SampleApp(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)

        powertools_layer = aws_lambda.LayerVersion.from_layer_version_arn(
            self,
            id="lambda-powertools",
            layer_version_arn=f"arn:aws:lambda:{Aws.REGION}:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18",
        )
        aws_lambda.Function(
            self,
            "sample-app-lambda",
            runtime=aws_lambda.Runtime.PYTHON_3_12,
            layers=[powertools_layer],
            code=aws_lambda.Code.from_asset("lambda"),
            handler="hello.handler",
        )

```

```
terraform {
  required_version = "~> 1.0.5"
  required_providers {
    aws = "~> 3.50.0"
  }
}

provider "aws" {
  region = "{region}"
}

resource "aws_iam_role" "iam_for_lambda" {
  name = "iam_for_lambda"

  assume_role_policy = <<EOF
    {
        "Version": "2012-10-17",
        "Statement": [
        {
            "Action": "sts:AssumeRole",
            "Principal": {
            "Service": "lambda.amazonaws.com"
            },
            "Effect": "Allow"
        }
        ]
    }
    EOF
}

resource "aws_lambda_function" "test_lambda" {
  filename      = "lambda_function_payload.zip"
  function_name = "lambda_function_name"
  role          = aws_iam_role.iam_for_lambda.arn
  handler       = "index.test"
  runtime       = "python3.12"
  layers        = ["arn:aws:lambda:{region}:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18"]

  source_code_hash = filebase64sha256("lambda_function_payload.zip")
}

```

```
import json

import pulumi
import pulumi_aws as aws

role = aws.iam.Role(
    "role",
    assume_role_policy=json.dumps(
        {
            "Version": "2012-10-17",
            "Statement": [
                {"Action": "sts:AssumeRole", "Principal": {"Service": "lambda.amazonaws.com"}, "Effect": "Allow"},
            ],
        },
    ),
    managed_policy_arns=[aws.iam.ManagedPolicy.AWS_LAMBDA_BASIC_EXECUTION_ROLE],
)

lambda_function = aws.lambda_.Function(
    "function",
    layers=[
        pulumi.Output.concat(
            "arn:aws:lambda:",
            aws.get_region_output().name,
            ":017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18",
        ),
    ],
    tracing_config={"mode": "Active"},
    runtime=aws.lambda_.Runtime.PYTHON3D12,
    handler="index.handler",
    role=role.arn,
    architectures=["x86_64"],
    code=pulumi.FileArchive("lambda_function_payload.zip"),
)

```

```
# Create a new one with the layer
❯ amplify add function
? Select which capability you want to add: Lambda function (serverless function)
? Provide an AWS Lambda function name: <NAME-OF-FUNCTION>
? Choose the runtime that you want to use: Python
? Do you want to configure advanced settings? Yes
...
? Do you want to enable Lambda layers for this function? Yes
? Enter up to 5 existing Lambda layer ARNs (comma-separated): arn:aws:lambda:eu-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18
❯ amplify push -y


# Updating an existing function and add the layer
❯ amplify update function
? Select the Lambda function you want to update test2
General information
- Name: <NAME-OF-FUNCTION>
? Which setting do you want to update? Lambda layers configuration
? Do you want to enable Lambda layers for this function? Yes
? Enter up to 5 existing Lambda layer ARNs (comma-separated): arn:aws:lambda:eu-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18
? Do you want to edit the local lambda function now? No

```

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  MyLambdaFunction:
      Type: AWS::Serverless::Function
      Properties:
          Architectures: [arm64]
          Runtime: python3.12
          Handler: app.lambda_handler
          Layers:
          - !Sub arn:aws:lambda:${AWS::Region}:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18

```

```
service: powertools-lambda

provider:
  name: aws
  runtime: python3.12
  region: us-east-1

functions:
  powertools:
    handler: lambda_function.lambda_handler
    architecture: arm64
    layers:
    - arn:aws:lambda:${aws:region}:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18

```

```
from aws_cdk import Aws, Stack, aws_lambda
from constructs import Construct


class SampleApp(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)

        powertools_layer = aws_lambda.LayerVersion.from_layer_version_arn(
            self,
            id="lambda-powertools",
            layer_version_arn=f"arn:aws:lambda:{Aws.REGION}:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18",
        )
        aws_lambda.Function(
            self,
            "sample-app-lambda",
            runtime=aws_lambda.Runtime.PYTHON_3_12,
            layers=[powertools_layer],
            architecture=aws_lambda.Architecture.ARM_64,
            code=aws_lambda.Code.from_asset("lambda"),
            handler="hello.handler",
        )

```

```
terraform {
  required_version = "~> 1.0.5"
  required_providers {
    aws = "~> 3.50.0"
  }
}

provider "aws" {
  region = "{region}"
}

resource "aws_iam_role" "iam_for_lambda" {
  name = "iam_for_lambda"

  assume_role_policy = <<EOF
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Action": "sts:AssumeRole",
          "Principal": {
            "Service": "lambda.amazonaws.com"
          },
          "Effect": "Allow"
        }
      ]
    }
    EOF
}

resource "aws_lambda_function" "test_lambda" {
  filename      = "lambda_function_payload.zip"
  function_name = "lambda_function_name"
  role          = aws_iam_role.iam_for_lambda.arn
  handler       = "index.test"
  runtime       = "python3.12"
  layers        = ["arn:aws:lambda:{region}:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18"]
  architectures = ["arm64"]

  source_code_hash = filebase64sha256("lambda_function_payload.zip")
}

```

```
import json

import pulumi
import pulumi_aws as aws

role = aws.iam.Role(
    "role",
    assume_role_policy=json.dumps(
        {
            "Version": "2012-10-17",
            "Statement": [
                {"Action": "sts:AssumeRole", "Principal": {"Service": "lambda.amazonaws.com"}, "Effect": "Allow"},
            ],
        },
    ),
    managed_policy_arns=[aws.iam.ManagedPolicy.AWS_LAMBDA_BASIC_EXECUTION_ROLE],
)

lambda_function = aws.lambda_.Function(
    "function",
    layers=[
        pulumi.Output.concat(
            "arn:aws:lambda:",
            aws.get_region_output().name,
            ":017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18",
        ),
    ],
    tracing_config={"mode": "Active"},
    runtime=aws.lambda_.Runtime.PYTHON3D12,
    handler="index.handler",
    role=role.arn,
    architectures=["arm64"],
    code=pulumi.FileArchive("lambda_function_payload.zip"),
)

```

```
# Create a new one with the layer
❯ amplify add function
? Select which capability you want to add: Lambda function (serverless function)
? Provide an AWS Lambda function name: <NAME-OF-FUNCTION>
? Choose the runtime that you want to use: Python
? Do you want to configure advanced settings? Yes
...
? Do you want to enable Lambda layers for this function? Yes
? Enter up to 5 existing Lambda layer ARNs (comma-separated): arn:aws:lambda:eu-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18
❯ amplify push -y


# Updating an existing function and add the layer
❯ amplify update function
? Select the Lambda function you want to update test2
General information
- Name: <NAME-OF-FUNCTION>
? Which setting do you want to update? Lambda layers configuration
? Do you want to enable Lambda layers for this function? Yes
? Enter up to 5 existing Lambda layer ARNs (comma-separated): arn:aws:lambda:eu-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18
? Do you want to edit the local lambda function now? No

```

You can use AWS CLI to generate a pre-signed URL to download the contents of our Lambda Layer.

```
aws lambda get-layer-version-by-arn --arn arn:aws:lambda:eu-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18 --region eu-west-1

```

You'll find the pre-signed URL under `Location` key as part of the CLI command output.

[Lambda Layer](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html) is a .zip file archive that can contain additional code, pre-packaged dependencies, data, or configuration files. We compile and optimize [all dependencies](#install), and remove duplicate dependencies [already available in the Lambda runtime](https://github.com/aws-powertools/powertools-lambda-python/blob/develop/layer_v3/docker/Dockerfile#L34) to achieve the most optimal size.

For the latter, make sure to replace `{python_version}` without the period (.), e.g., `python313` for `Python 3.13`.

**AWS GovCloud (us-gov-east-1)**

| Architecture | Layer ARN | | --- | --- | | x86_64 | **arn:aws-us-gov:lambda:us-gov-east-1:165087284144:layer:AWSLambdaPowertoolsPythonV3-{python_version}-x86_64:7** | | ARM | **arn:aws-us-gov:lambda:us-gov-east-1:165087284144:layer:AWSLambdaPowertoolsPythonV3-{python_version}-arm64:7** |

**AWS GovCloud (us-gov-west-1)**

| Architecture | Layer ARN | | --- | --- | | x86_64 | **arn:aws-us-gov:lambda:us-gov-west-1:165093116878:layer:AWSLambdaPowertoolsPythonV3-{python_version}-x86_64:7** | | ARM | **arn:aws-us-gov:lambda:us-gov-west-1:165093116878:layer:AWSLambdaPowertoolsPythonV3-{python_version}-arm64:7** |

We provide a SAR App that deploys a CloudFormation stack with a copy of our Lambda Layer in your AWS account and region.

Compared with the [public Layer ARN](#lambda-layer) option, the advantage is being able to use a semantic version. Make sure to replace `{python_version}` without the period (.), e.g., `python313` for `Python 3.13`.

| App | ARN | Architecture | | --- | --- | --- | | aws-lambda-powertools-python-layer-v3-{python_version}-x86-64 | arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-{python_version}-x86-64 | X86_64 | | aws-lambda-powertools-python-layer-v3-{python_version}-arm64 | arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-{python_version}-arm64 | ARM64 |

Don't have enough permissions? Expand for a least-privilege IAM policy example

Credits to [mwarkentin](https://github.com/mwarkentin) for providing the scoped down IAM permissions.

```
  AWSTemplateFormatVersion: "2010-09-09"
  Resources:
      PowertoolsLayerIamRole:
        Type: "AWS::IAM::Role"
        Properties:
            AssumeRolePolicyDocument:
                Version: "2012-10-17"
                Statement:
                    - Effect: "Allow"
                      Principal:
                        Service:
                          - "cloudformation.amazonaws.com"
                        Action:
                          - "sts:AssumeRole"
                Path: "/"

      PowertoolsLayerIamPolicy:
        Type: "AWS::IAM::Policy"
        Properties:
            PolicyName: PowertoolsLambdaLayerPolicy
            PolicyDocument:
                Version: "2012-10-17"
                Statement:
                    - Sid: CloudFormationTransform
                      Effect: Allow
                      Action: cloudformation:CreateChangeSet
                      Resource:
                        - arn:aws:cloudformation:us-east-1:aws:transform/Serverless-2016-10-31
                    - Sid: GetCfnTemplate
                      Effect: Allow
                      Action:
                        - serverlessrepo:CreateCloudFormationTemplate
                        - serverlessrepo:GetCloudFormationTemplate
                      Resource:
                        # this is arn of the Powertools for AWS Lambda (Python) SAR app
                        - arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python313-x86-64
                    - Sid: S3AccessLayer
                      Effect: Allow
                      Action:
                        - s3:GetObject
                      Resource:
                        # AWS publishes to an external S3 bucket locked down to your account ID
                        # The below example is us publishing Powertools for AWS Lambda (Python)
                        # Bucket: awsserverlessrepo-changesets-plntc6bfnfj
                        # Key: *****/arn:aws:serverlessrepo:eu-west-1:057560766410:applications-aws-lambda-powertools-python-layer-v3-python313-x86-64-3.0.9/aeeccf50-****-****-****-*********
                        - arn:aws:s3:::awsserverlessrepo-changesets-*/*
                    - Sid: GetLayerVersion
                      Effect: Allow
                      Action:
                        - lambda:PublishLayerVersion
                        - lambda:GetLayerVersion
                      Resource:
                        - !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:layer:aws-lambda-powertools-python-layer-v3*
            Roles:
            - Ref: "PowertoolsLayerIamRole"

```

If you're using Infrastructure as Code, here are some excerpts on how to use SAR:

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  AwsLambdaPowertoolsPythonLayer:
      Type: AWS::Serverless::Application
      Properties:
          Location:
              ApplicationId: arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python313-x86-64
              SemanticVersion: 3.0.9 # change to latest semantic version available in SAR

  MyLambdaFunction:
      Type: AWS::Serverless::Function
      Properties:
        Runtime: python3.13
        Handler: app.lambda_handler
        Layers:
            # fetch Layer ARN from SAR App stack output
            - !GetAtt AwsLambdaPowertoolsPythonLayer.Outputs.LayerVersionArn

```

```
service: powertools-lambda

provider:
  name: aws
  runtime: python3.13
  region: us-east-1

functions:
    powertools:
      handler: lambda_function.lambda_handler
      layers:
          - !GetAtt AwsLambdaPowertoolsPythonLayer.Outputs.LayerVersionArn

resources:
  - AwsLambdaPowertoolsPythonLayer:
      Type: AWS::Serverless::Application
      Properties:
        Location:
          ApplicationId:  arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python313-x86-64
          SemanticVersion: 3.0.9

```

```
from aws_cdk import Stack, aws_lambda, aws_sam
from constructs import Construct

POWERTOOLS_BASE_NAME = "AWSLambdaPowertools"
# Find latest from github.com/aws-powertools/powertools-lambda-python/releases
POWERTOOLS_VER = "3.0.9"
POWERTOOLS_ARN = (
    "arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python313-x86-64"
)


class SampleApp(Stack):
    def __init__(self, scope: Construct, id_: str) -> None:
        super().__init__(scope, id_)

        # Launches SAR App as CloudFormation nested stack and return Lambda Layer
        powertools_app = aws_sam.CfnApplication(
            self,
            f"{POWERTOOLS_BASE_NAME}Application",
            location={"applicationId": POWERTOOLS_ARN, "semanticVersion": POWERTOOLS_VER},
        )

        powertools_layer_arn = powertools_app.get_att("Outputs.LayerVersionArn").to_string()
        powertools_layer_version = aws_lambda.LayerVersion.from_layer_version_arn(
            self,
            f"{POWERTOOLS_BASE_NAME}",
            powertools_layer_arn,
        )

        aws_lambda.Function(
            self,
            "sample-app-lambda",
            runtime=aws_lambda.Runtime.PYTHON_3_13,
            function_name="sample-lambda",
            code=aws_lambda.Code.from_asset("lambda"),
            handler="hello.handler",
            layers=[powertools_layer_version],
        )

```

> Credits to [Dani Comnea](https://github.com/DanyC97) for providing the Terraform equivalent.

```
terraform {
  required_version = "~> 0.13"
  required_providers {
    aws = "~> 3.50.0"
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_serverlessapplicationrepository_cloudformation_stack" "deploy_sar_stack" {
  name = "aws-lambda-powertools-python-layer"

  application_id   = data.aws_serverlessapplicationrepository_application.sar_app.application_id
  semantic_version = data.aws_serverlessapplicationrepository_application.sar_app.semantic_version
  capabilities = [
    "CAPABILITY_IAM",
    "CAPABILITY_NAMED_IAM"
  ]
}

data "aws_serverlessapplicationrepository_application" "sar_app" {
  application_id   = "arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python313-x86-64"
  semantic_version = var.aws_powertools_version
}

variable "aws_powertools_version" {
  type        = string
  default     = "3.0.9"
  description = "The Powertools for AWS Lambda (Python) release version"
}

output "deployed_powertools_sar_version" {
  value = data.aws_serverlessapplicationrepository_application.sar_app.semantic_version
}

# Fetch Powertools for AWS Lambda (Python) Layer ARN from deployed SAR App
output "aws_lambda_powertools_layer_arn" {
  value = aws_serverlessapplicationrepository_cloudformation_stack.deploy_sar_stack.outputs.LayerVersionArn
}

```

Every morning during business days *(~8am UTC)*, we publish a `prerelease` to PyPi to accelerate customer feedback on **unstable** releases / bugfixes until they become production ready.

Here's how you can use them:

- **Pip**: [**`pip install --pre "aws-lambda-powertools"`**](#)
- **Poetry**: [**`poetry add --allow-prereleases "aws-lambda-powertools" --group dev`**](#)
- **Pdm**: [**`pdm add -dG --prerelease "aws-lambda-powertools"`**](#)

### Local development

Using Lambda Layer? Simply add [**`"aws-lambda-powertools[all]"`**](#) as a development dependency.

Powertools for AWS Lambda (Python) relies on the [AWS SDK bundled in the Lambda runtime](https://docs.aws.amazon.com/lambda/latest/dg/lambda-python.html). This helps us achieve an optimal package size and initialization. However, when developing locally, you need to install AWS SDK as a development dependency to support IDE auto-completion and to run your tests locally:

- **Pip**: [**`pip install "aws-lambda-powertools[aws-sdk]"`**](#)
- **Poetry**: [**`poetry add "aws-lambda-powertools[aws-sdk]" --group dev`**](#)
- **Pdm**: [**`pdm add -dG "aws-lambda-powertools[aws-sdk]"`**](#)

**A word about dependency resolution**

In this context, `[aws-sdk]` is an alias to the `boto3` package. Due to dependency resolution, it'll either install:

- **(A)** the SDK version available in [Lambda runtime](https://docs.aws.amazon.com/lambda/latest/dg/lambda-python.html)
- **(B)** a more up-to-date version if another package you use also depends on `boto3`, for example [Powertools for AWS Lambda (Python) Tracer](core/tracer/)

### Lambda Layer

[Lambda Layer](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html) is a .zip file archive that can contain additional code, pre-packaged dependencies, data, or configuration files. We compile and optimize [all dependencies](#install) for Python versions from **3.9 to 3.13**, as well as for both **arm64 and x86_64** architectures, to ensure compatibility. We also remove duplicate dependencies [already available in the Lambda runtime](https://github.com/aws-powertools/powertools-lambda-layer-cdk/blob/d24716744f7d1f37617b4998c992c4c067e19e64/layer/Python/Dockerfile#L36) to achieve the most optimal size.

Click to expand and copy any regional Lambda Layer ARN

| Region | Layer ARN | | --- | --- | | **`af-south-1`** | **arn:aws:lambda:af-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`ap-east-1`** | **arn:aws:lambda:ap-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`ap-northeast-1`** | **arn:aws:lambda:ap-northeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`ap-northeast-2`** | **arn:aws:lambda:ap-northeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`ap-northeast-3`** | **arn:aws:lambda:ap-northeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`ap-south-1`** | **arn:aws:lambda:ap-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`ap-south-2`** | **arn:aws:lambda:ap-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`ap-southeast-1`** | **arn:aws:lambda:ap-southeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`ap-southeast-2`** | **arn:aws:lambda:ap-southeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`ap-southeast-3`** | **arn:aws:lambda:ap-southeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`ap-southeast-4`** | **arn:aws:lambda:ap-southeast-4:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`ap-southeast-5`** | **arn:aws:lambda:ap-southeast-5:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`ap-southeast-7`** | **arn:aws:lambda:ap-southeast-7:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`ca-central-1`** | **arn:aws:lambda:ca-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`ca-west-1`** | **arn:aws:lambda:ca-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`eu-central-1`** | **arn:aws:lambda:eu-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`eu-central-2`** | **arn:aws:lambda:eu-central-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`eu-north-1`** | **arn:aws:lambda:eu-north-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`eu-south-1`** | **arn:aws:lambda:eu-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`eu-south-2`** | **arn:aws:lambda:eu-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`eu-west-1`** | **arn:aws:lambda:eu-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`eu-west-2`** | **arn:aws:lambda:eu-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`eu-west-3`** | **arn:aws:lambda:eu-west-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`il-central-1`** | **arn:aws:lambda:il-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`me-central-1`** | **arn:aws:lambda:me-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`me-south-1`** | **arn:aws:lambda:me-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`mx-central-1`** | **arn:aws:lambda:mx-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`sa-east-1`** | **arn:aws:lambda:sa-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`us-east-1`** | **arn:aws:lambda:us-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`us-east-2`** | **arn:aws:lambda:us-east-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`us-west-1`** | **arn:aws:lambda:us-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** | | **`us-west-2`** | **arn:aws:lambda:us-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-x86_64:18** |

| Region | Layer ARN | | --- | --- | | **`af-south-1`** | **arn:aws:lambda:af-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`ap-east-1`** | **arn:aws:lambda:ap-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`ap-northeast-1`** | **arn:aws:lambda:ap-northeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`ap-northeast-2`** | **arn:aws:lambda:ap-northeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`ap-northeast-3`** | **arn:aws:lambda:ap-northeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`ap-south-1`** | **arn:aws:lambda:ap-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`ap-south-2`** | **arn:aws:lambda:ap-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`ap-southeast-1`** | **arn:aws:lambda:ap-southeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`ap-southeast-2`** | **arn:aws:lambda:ap-southeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`ap-southeast-3`** | **arn:aws:lambda:ap-southeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`ap-southeast-4`** | **arn:aws:lambda:ap-southeast-4:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`ap-southeast-5`** | **arn:aws:lambda:ap-southeast-5:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`ap-southeast-7`** | **arn:aws:lambda:ap-southeast-7:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`ca-central-1`** | **arn:aws:lambda:ca-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`ca-west-1`** | **arn:aws:lambda:ca-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`eu-central-1`** | **arn:aws:lambda:eu-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`eu-central-2`** | **arn:aws:lambda:eu-central-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`eu-north-1`** | **arn:aws:lambda:eu-north-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`eu-south-1`** | **arn:aws:lambda:eu-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`eu-south-2`** | **arn:aws:lambda:eu-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`eu-west-1`** | **arn:aws:lambda:eu-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`eu-west-2`** | **arn:aws:lambda:eu-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`eu-west-3`** | **arn:aws:lambda:eu-west-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`il-central-1`** | **arn:aws:lambda:il-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`me-central-1`** | **arn:aws:lambda:me-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`me-south-1`** | **arn:aws:lambda:me-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`mx-central-1`** | **arn:aws:lambda:mx-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`sa-east-1`** | **arn:aws:lambda:sa-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`us-east-1`** | **arn:aws:lambda:us-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`us-east-2`** | **arn:aws:lambda:us-east-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`us-west-1`** | **arn:aws:lambda:us-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** | | **`us-west-2`** | **arn:aws:lambda:us-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-x86_64:18** |

| Region | Layer ARN | | --- | --- | | **`af-south-1`** | **arn:aws:lambda:af-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`ap-east-1`** | **arn:aws:lambda:ap-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`ap-northeast-1`** | **arn:aws:lambda:ap-northeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`ap-northeast-2`** | **arn:aws:lambda:ap-northeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`ap-northeast-3`** | **arn:aws:lambda:ap-northeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`ap-south-1`** | **arn:aws:lambda:ap-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`ap-south-2`** | **arn:aws:lambda:ap-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`ap-southeast-1`** | **arn:aws:lambda:ap-southeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`ap-southeast-2`** | **arn:aws:lambda:ap-southeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`ap-southeast-3`** | **arn:aws:lambda:ap-southeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`ap-southeast-4`** | **arn:aws:lambda:ap-southeast-4:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`ap-southeast-5`** | **arn:aws:lambda:ap-southeast-5:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`ap-southeast-7`** | **arn:aws:lambda:ap-southeast-7:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`ca-central-1`** | **arn:aws:lambda:ca-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`ca-west-1`** | **arn:aws:lambda:ca-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`eu-central-1`** | **arn:aws:lambda:eu-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`eu-central-2`** | **arn:aws:lambda:eu-central-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`eu-north-1`** | **arn:aws:lambda:eu-north-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`eu-south-1`** | **arn:aws:lambda:eu-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`eu-south-2`** | **arn:aws:lambda:eu-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`eu-west-1`** | **arn:aws:lambda:eu-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`eu-west-2`** | **arn:aws:lambda:eu-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`eu-west-3`** | **arn:aws:lambda:eu-west-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`il-central-1`** | **arn:aws:lambda:il-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`me-central-1`** | **arn:aws:lambda:me-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`me-south-1`** | **arn:aws:lambda:me-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`mx-central-1`** | **arn:aws:lambda:mx-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`sa-east-1`** | **arn:aws:lambda:sa-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`us-east-1`** | **arn:aws:lambda:us-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`us-east-2`** | **arn:aws:lambda:us-east-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`us-west-1`** | **arn:aws:lambda:us-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** | | **`us-west-2`** | **arn:aws:lambda:us-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-x86_64:18** |

| Region | Layer ARN | | --- | --- | | **`af-south-1`** | **arn:aws:lambda:af-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`ap-east-1`** | **arn:aws:lambda:ap-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`ap-northeast-1`** | **arn:aws:lambda:ap-northeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`ap-northeast-2`** | **arn:aws:lambda:ap-northeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`ap-northeast-3`** | **arn:aws:lambda:ap-northeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`ap-south-1`** | **arn:aws:lambda:ap-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`ap-south-2`** | **arn:aws:lambda:ap-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`ap-southeast-1`** | **arn:aws:lambda:ap-southeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`ap-southeast-2`** | **arn:aws:lambda:ap-southeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`ap-southeast-3`** | **arn:aws:lambda:ap-southeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`ap-southeast-4`** | **arn:aws:lambda:ap-southeast-4:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`ap-southeast-5`** | **arn:aws:lambda:ap-southeast-5:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`ap-southeast-7`** | **arn:aws:lambda:ap-southeast-7:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`ca-central-1`** | **arn:aws:lambda:ca-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`ca-west-1`** | **arn:aws:lambda:ca-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`eu-central-1`** | **arn:aws:lambda:eu-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`eu-central-2`** | **arn:aws:lambda:eu-central-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`eu-north-1`** | **arn:aws:lambda:eu-north-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`eu-south-1`** | **arn:aws:lambda:eu-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`eu-south-2`** | **arn:aws:lambda:eu-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`eu-west-1`** | **arn:aws:lambda:eu-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`eu-west-2`** | **arn:aws:lambda:eu-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`eu-west-3`** | **arn:aws:lambda:eu-west-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`il-central-1`** | **arn:aws:lambda:il-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`me-central-1`** | **arn:aws:lambda:me-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`me-south-1`** | **arn:aws:lambda:me-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`mx-central-1`** | **arn:aws:lambda:mx-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`sa-east-1`** | **arn:aws:lambda:sa-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`us-east-1`** | **arn:aws:lambda:us-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`us-east-2`** | **arn:aws:lambda:us-east-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`us-west-1`** | **arn:aws:lambda:us-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** | | **`us-west-2`** | **arn:aws:lambda:us-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18** |

| Region | Layer ARN | | --- | --- | | **`af-south-1`** | **arn:aws:lambda:af-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`ap-east-1`** | **arn:aws:lambda:ap-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`ap-northeast-1`** | **arn:aws:lambda:ap-northeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`ap-northeast-2`** | **arn:aws:lambda:ap-northeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`ap-northeast-3`** | **arn:aws:lambda:ap-northeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`ap-south-1`** | **arn:aws:lambda:ap-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`ap-south-2`** | **arn:aws:lambda:ap-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`ap-southeast-1`** | **arn:aws:lambda:ap-southeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`ap-southeast-2`** | **arn:aws:lambda:ap-southeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`ap-southeast-3`** | **arn:aws:lambda:ap-southeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`ap-southeast-4`** | **arn:aws:lambda:ap-southeast-4:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`ap-southeast-5`** | **arn:aws:lambda:ap-southeast-5:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`ap-southeast-7`** | **arn:aws:lambda:ap-southeast-7:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`ca-central-1`** | **arn:aws:lambda:ca-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`ca-west-1`** | **arn:aws:lambda:ca-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`eu-central-1`** | **arn:aws:lambda:eu-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`eu-central-2`** | **arn:aws:lambda:eu-central-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`eu-north-1`** | **arn:aws:lambda:eu-north-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`eu-south-1`** | **arn:aws:lambda:eu-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`eu-south-2`** | **arn:aws:lambda:eu-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`eu-west-1`** | **arn:aws:lambda:eu-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`eu-west-2`** | **arn:aws:lambda:eu-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`eu-west-3`** | **arn:aws:lambda:eu-west-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`il-central-1`** | **arn:aws:lambda:il-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`me-central-1`** | **arn:aws:lambda:me-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`me-south-1`** | **arn:aws:lambda:me-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`mx-central-1`** | **arn:aws:lambda:mx-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`sa-east-1`** | **arn:aws:lambda:sa-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`us-east-1`** | **arn:aws:lambda:us-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`us-east-2`** | **arn:aws:lambda:us-east-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`us-west-1`** | **arn:aws:lambda:us-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** | | **`us-west-2`** | **arn:aws:lambda:us-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-x86_64:18** |

Click to expand and copy any regional Lambda Layer ARN

| Region | Layer ARN | | --- | --- | | **`af-south-1`** | **arn:aws:lambda:af-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`ap-east-1`** | **arn:aws:lambda:ap-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`ap-northeast-1`** | **arn:aws:lambda:ap-northeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`ap-northeast-2`** | **arn:aws:lambda:ap-northeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`ap-northeast-3`** | **arn:aws:lambda:ap-northeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`ap-south-1`** | **arn:aws:lambda:ap-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`ap-south-2`** | **arn:aws:lambda:ap-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`ap-southeast-1`** | **arn:aws:lambda:ap-southeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`ap-southeast-2`** | **arn:aws:lambda:ap-southeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`ap-southeast-3`** | **arn:aws:lambda:ap-southeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`ap-southeast-4`** | **arn:aws:lambda:ap-southeast-4:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`ap-southeast-5`** | **arn:aws:lambda:ap-southeast-5:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`ap-southeast-7`** | **arn:aws:lambda:ap-southeast-7:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`ca-central-1`** | **arn:aws:lambda:ca-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`eu-central-1`** | **arn:aws:lambda:eu-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`eu-central-2`** | **arn:aws:lambda:eu-central-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`eu-north-1`** | **arn:aws:lambda:eu-north-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`eu-south-1`** | **arn:aws:lambda:eu-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`eu-south-2`** | **arn:aws:lambda:eu-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`eu-west-1`** | **arn:aws:lambda:eu-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`eu-west-2`** | **arn:aws:lambda:eu-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`eu-west-3`** | **arn:aws:lambda:eu-west-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`il-central-1`** | **arn:aws:lambda:il-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`me-central-1`** | **arn:aws:lambda:me-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`me-south-1`** | **arn:aws:lambda:me-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`mx-central-1`** | **arn:aws:lambda:mx-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`sa-east-1`** | **arn:aws:lambda:sa-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`us-east-1`** | **arn:aws:lambda:us-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`us-east-2`** | **arn:aws:lambda:us-east-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`us-west-1`** | **arn:aws:lambda:us-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** | | **`us-west-2`** | **arn:aws:lambda:us-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python39-arm64:18** |

| Region | Layer ARN | | --- | --- | | **`af-south-1`** | **arn:aws:lambda:af-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`ap-east-1`** | **arn:aws:lambda:ap-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`ap-northeast-1`** | **arn:aws:lambda:ap-northeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`ap-northeast-2`** | **arn:aws:lambda:ap-northeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`ap-northeast-3`** | **arn:aws:lambda:ap-northeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`ap-south-1`** | **arn:aws:lambda:ap-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`ap-south-2`** | **arn:aws:lambda:ap-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`ap-southeast-1`** | **arn:aws:lambda:ap-southeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`ap-southeast-2`** | **arn:aws:lambda:ap-southeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`ap-southeast-3`** | **arn:aws:lambda:ap-southeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`ap-southeast-4`** | **arn:aws:lambda:ap-southeast-4:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`ap-southeast-5`** | **arn:aws:lambda:ap-southeast-5:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`ap-southeast-7`** | **arn:aws:lambda:ap-southeast-7:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`ca-central-1`** | **arn:aws:lambda:ca-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`eu-central-1`** | **arn:aws:lambda:eu-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`eu-central-2`** | **arn:aws:lambda:eu-central-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`eu-north-1`** | **arn:aws:lambda:eu-north-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`eu-south-1`** | **arn:aws:lambda:eu-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`eu-south-2`** | **arn:aws:lambda:eu-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`eu-west-1`** | **arn:aws:lambda:eu-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`eu-west-2`** | **arn:aws:lambda:eu-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`eu-west-3`** | **arn:aws:lambda:eu-west-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`il-central-1`** | **arn:aws:lambda:il-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`me-central-1`** | **arn:aws:lambda:me-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`me-south-1`** | **arn:aws:lambda:me-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`mx-central-1`** | **arn:aws:lambda:mx-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`sa-east-1`** | **arn:aws:lambda:sa-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`us-east-1`** | **arn:aws:lambda:us-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`us-east-2`** | **arn:aws:lambda:us-east-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`us-west-1`** | **arn:aws:lambda:us-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** | | **`us-west-2`** | **arn:aws:lambda:us-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python310-arm64:18** |

| Region | Layer ARN | | --- | --- | | **`af-south-1`** | **arn:aws:lambda:af-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`ap-east-1`** | **arn:aws:lambda:ap-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`ap-northeast-1`** | **arn:aws:lambda:ap-northeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`ap-northeast-2`** | **arn:aws:lambda:ap-northeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`ap-northeast-3`** | **arn:aws:lambda:ap-northeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`ap-south-1`** | **arn:aws:lambda:ap-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`ap-south-2`** | **arn:aws:lambda:ap-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`ap-southeast-1`** | **arn:aws:lambda:ap-southeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`ap-southeast-2`** | **arn:aws:lambda:ap-southeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`ap-southeast-3`** | **arn:aws:lambda:ap-southeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`ap-southeast-4`** | **arn:aws:lambda:ap-southeast-4:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`ap-southeast-5`** | **arn:aws:lambda:ap-southeast-5:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`ap-southeast-7`** | **arn:aws:lambda:ap-southeast-7:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`ca-central-1`** | **arn:aws:lambda:ca-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`eu-central-1`** | **arn:aws:lambda:eu-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`eu-central-2`** | **arn:aws:lambda:eu-central-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`eu-north-1`** | **arn:aws:lambda:eu-north-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`eu-south-1`** | **arn:aws:lambda:eu-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`eu-south-2`** | **arn:aws:lambda:eu-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`eu-west-1`** | **arn:aws:lambda:eu-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`eu-west-2`** | **arn:aws:lambda:eu-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`eu-west-3`** | **arn:aws:lambda:eu-west-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`il-central-1`** | **arn:aws:lambda:il-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`me-central-1`** | **arn:aws:lambda:me-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`me-south-1`** | **arn:aws:lambda:me-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`mx-central-1`** | **arn:aws:lambda:mx-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`sa-east-1`** | **arn:aws:lambda:sa-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`us-east-1`** | **arn:aws:lambda:us-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`us-east-2`** | **arn:aws:lambda:us-east-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`us-west-1`** | **arn:aws:lambda:us-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** | | **`us-west-2`** | **arn:aws:lambda:us-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python311-arm64:18** |

| Region | Layer ARN | | --- | --- | | **`af-south-1`** | **arn:aws:lambda:af-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`ap-east-1`** | **arn:aws:lambda:ap-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`ap-northeast-1`** | **arn:aws:lambda:ap-northeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`ap-northeast-2`** | **arn:aws:lambda:ap-northeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`ap-northeast-3`** | **arn:aws:lambda:ap-northeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`ap-south-1`** | **arn:aws:lambda:ap-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`ap-south-2`** | **arn:aws:lambda:ap-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`ap-southeast-1`** | **arn:aws:lambda:ap-southeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`ap-southeast-2`** | **arn:aws:lambda:ap-southeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`ap-southeast-3`** | **arn:aws:lambda:ap-southeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`ap-southeast-4`** | **arn:aws:lambda:ap-southeast-4:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`ap-southeast-5`** | **arn:aws:lambda:ap-southeast-5:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`ap-southeast-7`** | **arn:aws:lambda:ap-southeast-7:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`ca-central-1`** | **arn:aws:lambda:ca-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`eu-central-1`** | **arn:aws:lambda:eu-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`eu-central-2`** | **arn:aws:lambda:eu-central-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`eu-north-1`** | **arn:aws:lambda:eu-north-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`eu-south-1`** | **arn:aws:lambda:eu-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`eu-south-2`** | **arn:aws:lambda:eu-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`eu-west-1`** | **arn:aws:lambda:eu-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`eu-west-2`** | **arn:aws:lambda:eu-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`eu-west-3`** | **arn:aws:lambda:eu-west-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`il-central-1`** | **arn:aws:lambda:il-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`me-central-1`** | **arn:aws:lambda:me-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`me-south-1`** | **arn:aws:lambda:me-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`mx-central-1`** | **arn:aws:lambda:mx-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`sa-east-1`** | **arn:aws:lambda:sa-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`us-east-1`** | **arn:aws:lambda:us-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`us-east-2`** | **arn:aws:lambda:us-east-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`us-west-1`** | **arn:aws:lambda:us-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** | | **`us-west-2`** | **arn:aws:lambda:us-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-arm64:18** |

| Region | Layer ARN | | --- | --- | | **`af-south-1`** | **arn:aws:lambda:af-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`ap-east-1`** | **arn:aws:lambda:ap-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`ap-northeast-1`** | **arn:aws:lambda:ap-northeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`ap-northeast-2`** | **arn:aws:lambda:ap-northeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`ap-northeast-3`** | **arn:aws:lambda:ap-northeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`ap-south-1`** | **arn:aws:lambda:ap-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`ap-south-2`** | **arn:aws:lambda:ap-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`ap-southeast-1`** | **arn:aws:lambda:ap-southeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`ap-southeast-2`** | **arn:aws:lambda:ap-southeast-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`ap-southeast-3`** | **arn:aws:lambda:ap-southeast-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`ap-southeast-4`** | **arn:aws:lambda:ap-southeast-4:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`ap-southeast-5`** | **arn:aws:lambda:ap-southeast-5:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`ap-southeast-7`** | **arn:aws:lambda:ap-southeast-7:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`ca-central-1`** | **arn:aws:lambda:ca-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`eu-central-1`** | **arn:aws:lambda:eu-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`eu-central-2`** | **arn:aws:lambda:eu-central-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`eu-north-1`** | **arn:aws:lambda:eu-north-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`eu-south-1`** | **arn:aws:lambda:eu-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`eu-south-2`** | **arn:aws:lambda:eu-south-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`eu-west-1`** | **arn:aws:lambda:eu-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`eu-west-2`** | **arn:aws:lambda:eu-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`eu-west-3`** | **arn:aws:lambda:eu-west-3:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`il-central-1`** | **arn:aws:lambda:il-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`me-central-1`** | **arn:aws:lambda:me-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`me-south-1`** | **arn:aws:lambda:me-south-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`mx-central-1`** | **arn:aws:lambda:mx-central-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`sa-east-1`** | **arn:aws:lambda:sa-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`us-east-1`** | **arn:aws:lambda:us-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`us-east-2`** | **arn:aws:lambda:us-east-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`us-west-1`** | **arn:aws:lambda:us-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** | | **`us-west-2`** | **arn:aws:lambda:us-west-2:017000801446:layer:AWSLambdaPowertoolsPythonV3-python313-arm64:18** |

**Want to inspect the contents of the Layer?**

The pre-signed URL to download this Lambda Layer will be within `Location` key in the CLI output. The CLI output will also contain the Powertools for AWS Lambda version it contains.

```
aws lambda get-layer-version-by-arn --arn arn:aws:lambda:eu-west-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:18 --region eu-west-1

```

#### SAR

Serverless Application Repository (SAR) App deploys a CloudFormation stack with a copy of our Lambda Layer in your AWS account and region.

Compared with the [public Layer ARN](#lambda-layer) option, SAR allows you to choose a semantic version and deploys a Layer in your target account.

| App | ARN | Python version | Architecture | | --- | --- | --- | --- | | [aws-lambda-powertools-python-layer-v3-python39-x86-64](https://serverlessrepo.aws.amazon.com/applications/eu-west-1/057560766410/aws-lambda-powertools-python-layer-v3-python39-x86-64) | [arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python39-x86-64](#) | Python 3.9 | X86_64 | | [aws-lambda-powertools-python-layer-v3-python310-x86-64](https://serverlessrepo.aws.amazon.com/applications/eu-west-1/057560766410/aws-lambda-powertools-python-layer-v3-python310-x86-64) | [arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python310-x86-64](#) | Python 3.10 | X86_64 | | [aws-lambda-powertools-python-layer-v3-python311-x86-64](https://serverlessrepo.aws.amazon.com/applications/eu-west-1/057560766410/aws-lambda-powertools-python-layer-v3-python11-x86-64) | [arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python311-x86-64](#) | Python 3.11 | X86_64 | | [aws-lambda-powertools-python-layer-v3-python312-x86-64](https://serverlessrepo.aws.amazon.com/applications/eu-west-1/057560766410/aws-lambda-powertools-python-layer-v3-python12-x86-64) | [arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python312-x86-64](#) | Python 3.12 | X86_64 | | [aws-lambda-powertools-python-layer-v3-python313-x86-64](https://serverlessrepo.aws.amazon.com/applications/eu-west-1/057560766410/aws-lambda-powertools-python-layer-v3-python313-x86-64) | [arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python313-x86-64](#) | Python 3.13 | X86_64 | | [aws-lambda-powertools-python-layer-v3-python39-arm64](https://serverlessrepo.aws.amazon.com/applications/eu-west-1/057560766410/aws-lambda-powertools-python-layer-v3-python39-arm64) | [arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python39-arm64](#) | Python 3.9 | ARM64 | | [aws-lambda-powertools-python-layer-v3-python310-arm64](https://serverlessrepo.aws.amazon.com/applications/eu-west-1/057560766410/aws-lambda-powertools-python-layer-v3-python310-arm64) | [arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python310-arm64](#) | Python 3.10 | ARM64 | | [aws-lambda-powertools-python-layer-v3-python311-arm64](https://serverlessrepo.aws.amazon.com/applications/eu-west-1/057560766410/aws-lambda-powertools-python-layer-v3-python11-arm64) | [arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python311-arm64](#) | Python 3.11 | ARM64 | | [aws-lambda-powertools-python-layer-v3-python312-arm64](https://serverlessrepo.aws.amazon.com/applications/eu-west-1/057560766410/aws-lambda-powertools-python-layer-v3-python12-arm64) | [arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python312-arm64](#) | Python 3.12 | ARM64 | | [aws-lambda-powertools-python-layer-v3-python313-arm64](https://serverlessrepo.aws.amazon.com/applications/eu-west-1/057560766410/aws-lambda-powertools-python-layer-v3-python313-arm64) | [arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python313-arm64](#) | Python 3.13 | ARM64 |

Click to expand and copy SAR code snippets for popular frameworks

You can create a shared Lambda Layers stack and make this along with other account level layers stack.

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  AwsLambdaPowertoolsPythonLayer:
      Type: AWS::Serverless::Application
      Properties:
          Location:
              ApplicationId: arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python313-x86-64
              SemanticVersion: 3.0.9 # change to latest semantic version available in SAR

  MyLambdaFunction:
      Type: AWS::Serverless::Function
      Properties:
        Runtime: python3.13
        Handler: app.lambda_handler
        Layers:
            # fetch Layer ARN from SAR App stack output
            - !GetAtt AwsLambdaPowertoolsPythonLayer.Outputs.LayerVersionArn

```

```
service: powertools-lambda

provider:
  name: aws
  runtime: python3.13
  region: us-east-1

functions:
    powertools:
      handler: lambda_function.lambda_handler
      layers:
          - !GetAtt AwsLambdaPowertoolsPythonLayer.Outputs.LayerVersionArn

resources:
  - AwsLambdaPowertoolsPythonLayer:
      Type: AWS::Serverless::Application
      Properties:
        Location:
          ApplicationId:  arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python313-x86-64
          SemanticVersion: 3.0.9

```

```
from aws_cdk import Stack, aws_lambda, aws_sam
from constructs import Construct

POWERTOOLS_BASE_NAME = "AWSLambdaPowertools"
# Find latest from github.com/aws-powertools/powertools-lambda-python/releases
POWERTOOLS_VER = "3.0.9"
POWERTOOLS_ARN = (
    "arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python313-x86-64"
)


class SampleApp(Stack):
    def __init__(self, scope: Construct, id_: str) -> None:
        super().__init__(scope, id_)

        # Launches SAR App as CloudFormation nested stack and return Lambda Layer
        powertools_app = aws_sam.CfnApplication(
            self,
            f"{POWERTOOLS_BASE_NAME}Application",
            location={"applicationId": POWERTOOLS_ARN, "semanticVersion": POWERTOOLS_VER},
        )

        powertools_layer_arn = powertools_app.get_att("Outputs.LayerVersionArn").to_string()
        powertools_layer_version = aws_lambda.LayerVersion.from_layer_version_arn(
            self,
            f"{POWERTOOLS_BASE_NAME}",
            powertools_layer_arn,
        )

        aws_lambda.Function(
            self,
            "sample-app-lambda",
            runtime=aws_lambda.Runtime.PYTHON_3_13,
            function_name="sample-lambda",
            code=aws_lambda.Code.from_asset("lambda"),
            handler="hello.handler",
            layers=[powertools_layer_version],
        )

```

> Credits to [Dani Comnea](https://github.com/DanyC97) for providing the Terraform equivalent.

```
terraform {
  required_version = "~> 0.13"
  required_providers {
    aws = "~> 3.50.0"
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_serverlessapplicationrepository_cloudformation_stack" "deploy_sar_stack" {
  name = "aws-lambda-powertools-python-layer"

  application_id   = data.aws_serverlessapplicationrepository_application.sar_app.application_id
  semantic_version = data.aws_serverlessapplicationrepository_application.sar_app.semantic_version
  capabilities = [
    "CAPABILITY_IAM",
    "CAPABILITY_NAMED_IAM"
  ]
}

data "aws_serverlessapplicationrepository_application" "sar_app" {
  application_id   = "arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python313-x86-64"
  semantic_version = var.aws_powertools_version
}

variable "aws_powertools_version" {
  type        = string
  default     = "3.0.9"
  description = "The Powertools for AWS Lambda (Python) release version"
}

output "deployed_powertools_sar_version" {
  value = data.aws_serverlessapplicationrepository_application.sar_app.semantic_version
}

# Fetch Powertools for AWS Lambda (Python) Layer ARN from deployed SAR App
output "aws_lambda_powertools_layer_arn" {
  value = aws_serverlessapplicationrepository_cloudformation_stack.deploy_sar_stack.outputs.LayerVersionArn
}

```

Credits to [mwarkentin](https://github.com/mwarkentin) for providing the scoped down IAM permissions below.

```
  AWSTemplateFormatVersion: "2010-09-09"
  Resources:
      PowertoolsLayerIamRole:
        Type: "AWS::IAM::Role"
        Properties:
            AssumeRolePolicyDocument:
                Version: "2012-10-17"
                Statement:
                    - Effect: "Allow"
                      Principal:
                        Service:
                          - "cloudformation.amazonaws.com"
                        Action:
                          - "sts:AssumeRole"
                Path: "/"

      PowertoolsLayerIamPolicy:
        Type: "AWS::IAM::Policy"
        Properties:
            PolicyName: PowertoolsLambdaLayerPolicy
            PolicyDocument:
                Version: "2012-10-17"
                Statement:
                    - Sid: CloudFormationTransform
                      Effect: Allow
                      Action: cloudformation:CreateChangeSet
                      Resource:
                        - arn:aws:cloudformation:us-east-1:aws:transform/Serverless-2016-10-31
                    - Sid: GetCfnTemplate
                      Effect: Allow
                      Action:
                        - serverlessrepo:CreateCloudFormationTemplate
                        - serverlessrepo:GetCloudFormationTemplate
                      Resource:
                        # this is arn of the Powertools for AWS Lambda (Python) SAR app
                        - arn:aws:serverlessrepo:eu-west-1:057560766410:applications/aws-lambda-powertools-python-layer-v3-python313-x86-64
                    - Sid: S3AccessLayer
                      Effect: Allow
                      Action:
                        - s3:GetObject
                      Resource:
                        # AWS publishes to an external S3 bucket locked down to your account ID
                        # The below example is us publishing Powertools for AWS Lambda (Python)
                        # Bucket: awsserverlessrepo-changesets-plntc6bfnfj
                        # Key: *****/arn:aws:serverlessrepo:eu-west-1:057560766410:applications-aws-lambda-powertools-python-layer-v3-python313-x86-64-3.0.9/aeeccf50-****-****-****-*********
                        - arn:aws:s3:::awsserverlessrepo-changesets-*/*
                    - Sid: GetLayerVersion
                      Effect: Allow
                      Action:
                        - lambda:PublishLayerVersion
                        - lambda:GetLayerVersion
                      Resource:
                        - !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:layer:aws-lambda-powertools-python-layer-v3*
            Roles:
            - Ref: "PowertoolsLayerIamRole"

```

## Quick getting started

```
sam init --app-template hello-world-powertools-python --name sam-app --package-type Zip --runtime python3.11 --no-tracing

```

## Features

Core utilities such as Tracing, Logging, Metrics, and Event Handler will be available across all Powertools for AWS Lambda languages. Additional utilities are subjective to each language ecosystem and customer demand.

| Utility | Description | | --- | --- | | [**Tracing**](core/tracer/) | Decorators and utilities to trace Lambda function handlers, and both synchronous and asynchronous functions | | [**Logger**](core/logger/) | Structured logging made easier, and decorator to enrich structured logging with key Lambda context details | | [**Metrics**](core/metrics/) | Custom Metrics created asynchronously via CloudWatch Embedded Metric Format (EMF) | | [**Event handler: AppSync**](core/event_handler/appsync/) | AppSync event handler for Lambda Direct Resolver and Amplify GraphQL Transformer function | | [**Event handler: API Gateway, ALB and Lambda Function URL**](https://docs.powertools.aws.dev/lambda/python/latest/core/event_handler/api_gateway/) | Amazon API Gateway REST/HTTP API and ALB event handler for Lambda functions invoked using Proxy integration, and Lambda Function URL | | [**Middleware factory**](utilities/middleware_factory/) | Decorator factory to create your own middleware to run logic before, and after each Lambda invocation | | [**Parameters**](utilities/parameters/) | Retrieve parameter values from AWS Systems Manager Parameter Store, AWS Secrets Manager, or Amazon DynamoDB, and cache them for a specific amount of time | | [**Batch processing**](utilities/batch/) | Handle partial failures for AWS SQS batch processing | | [**Typing**](utilities/typing/) | Static typing classes to speedup development in your IDE | | [**Validation**](utilities/validation/) | JSON Schema validator for inbound events and responses | | [**Event source data classes**](utilities/data_classes/) | Data classes describing the schema of common Lambda event triggers | | [**Parser**](utilities/parser/) | Data parsing and deep validation using Pydantic | | [**Idempotency**](utilities/idempotency/) | Idempotent Lambda handler | | [**Data Masking**](utilities/data_masking/) | Protect confidential data with easy removal or encryption | | [**Feature Flags**](utilities/feature_flags/) | A simple rule engine to evaluate when one or multiple features should be enabled depending on the input | | [**Streaming**](utilities/streaming/) | Streams datasets larger than the available memory as streaming data. |

## Environment variables

Info

Explicit parameters take precedence over environment variables

| Environment variable | Description | Utility | Default | | --- | --- | --- | --- | | **POWERTOOLS_SERVICE_NAME** | Sets service name used for tracing namespace, metrics dimension and structured logging | All | `"service_undefined"` | | **POWERTOOLS_METRICS_NAMESPACE** | Sets namespace used for metrics | [Metrics](core/metrics/) | `None` | | **POWERTOOLS_METRICS_FUNCTION_NAME** | Function name used as dimension for the **ColdStart** metric metrics | [Metrics](core/metrics/) | `None` | | **POWERTOOLS_METRICS_DISABLED** | **Disables** all metrics emitted by Powertools metrics | [Metrics](core/metrics/) | `None` | | **POWERTOOLS_TRACE_DISABLED** | Explicitly disables tracing | [Tracing](core/tracer/) | `false` | | **POWERTOOLS_TRACER_CAPTURE_RESPONSE** | Captures Lambda or method return as metadata. | [Tracing](core/tracer/) | `true` | | **POWERTOOLS_TRACER_CAPTURE_ERROR** | Captures Lambda or method exception as metadata. | [Tracing](core/tracer/) | `true` | | **POWERTOOLS_TRACE_MIDDLEWARES** | Creates sub-segment for each custom middleware | [Middleware factory](utilities/middleware_factory/) | `false` | | **POWERTOOLS_LOGGER_LOG_EVENT** | Logs incoming event | [Logging](core/logger/) | `false` | | **POWERTOOLS_LOGGER_SAMPLE_RATE** | Debug log sampling | [Logging](core/logger/) | `0` | | **POWERTOOLS_LOG_DEDUPLICATION_DISABLED** | Disables log deduplication filter protection to use Pytest Live Log feature | [Logging](core/logger/) | `false` | | **POWERTOOLS_PARAMETERS_MAX_AGE** | Adjust how long values are kept in cache (in seconds) | [Parameters](utilities/parameters/#adjusting-cache-ttl) | `5` | | **POWERTOOLS_PARAMETERS_SSM_DECRYPT** | Sets whether to decrypt or not values retrieved from AWS SSM Parameters Store | [Parameters](utilities/parameters/#ssmprovider) | `false` | | **POWERTOOLS_DEV** | Increases verbosity across utilities | Multiple; see [POWERTOOLS_DEV effect below](#optimizing-for-non-production-environments) | `false` | | **POWERTOOLS_LOG_LEVEL** | Sets logging level | [Logging](core/logger/) | `INFO` |

### Optimizing for non-production environments

We will emit a warning when this feature is used to help you detect misuse in production.

Whether you're prototyping locally or against a non-production environment, you can use `POWERTOOLS_DEV` to increase verbosity across multiple utilities.

When `POWERTOOLS_DEV` is set to a truthy value (`1`, `true`), it'll have the following effects:

| Utility | Effect | | --- | --- | | **Logger** | Increase JSON indentation to 4. This will ease local debugging when running functions locally under emulators or direct calls while not affecting unit tests. However, Amazon CloudWatch Logs view will degrade as each new line is treated as a new message. | | **Event Handler** | Enable full traceback errors in the response, indent request/responses, and CORS in dev mode (`*`). | | **Tracer** | Future-proof safety to disables tracing operations in non-Lambda environments. This already happens automatically in the Tracer utility. | | **Metrics** | Disables Powertools metrics emission by default. However, this can be overridden by explicitly setting POWERTOOLS_METRICS_DISABLED=false, which takes precedence over the dev mode setting. |

## Debug mode

As a best practice for libraries, Powertools module logging statements are suppressed.

When necessary, you can use `POWERTOOLS_DEBUG` environment variable to enable debugging. This will provide additional information on every internal operation.

## Support Powertools for AWS Lambda (Python)

There are many ways you can help us gain future investments to improve everyone's experience:

- **Become a public reference**

  ______________________________________________________________________

  Add your company name and logo on our [landing page](https://powertools.aws.dev).

  [GitHub Issue template](https://github.com/aws-powertools/powertools-lambda-python/issues/new?assignees=&labels=customer-reference&template=support_powertools.yml&title=%5BSupport+Lambda+Powertools%5D%3A+%3Cyour+organization+name%3E)

- **Share your work**

  ______________________________________________________________________

  Blog posts, video, and sample projects about Powertools for AWS Lambda.

  [GitHub Issue template](https://github.com/aws-powertools/powertools-lambda-python/issues/new?assignees=&labels=community-content&template=share_your_work.yml&title=%5BI+Made+This%5D%3A+%3CTITLE%3E)

- **Join the community**

  ______________________________________________________________________

  Connect, ask questions, and share what features you use.

  [Discord invite](https://discord.gg/B8zZKbbyET)

### Becoming a reference customer

Knowing which companies are using this library is important to help prioritize the project internally. The following companies, among others, use Powertools:

[**Alma Media**](https://www.almamedia.fi/en/)

[**Banxware**](https://www.banxware.com)

[**Brsk**](https://www.brsk.co.uk/)

[**BusPatrol**](https://buspatrol.com/)

[**Capital One**](https://www.capitalone.com/)

[**Caylent**](https://caylent.com/)

[**CHS Inc.**](https://www.chsinc.com/)

[**CPQi (Exadel Financial Services)**](https://cpqi.com/)

[**CloudZero**](https://www.cloudzero.com/)

[**CyberArk**](https://www.cyberark.com/)

[**Flyweight**](https://flyweight.io/)

[**globaldatanet**](https://globaldatanet.com/)

[**Guild**](https://guild.com/)

[**Instil**](https://instil.co/)

[**IMS**](https://ims.tech/)

[**Jit Security**](https://www.jit.io/)

[**LocalStack**](https://www.localstack.cloud/)

[**Propellor.ai**](https://www.propellor.ai/)

[**Pushpay**](https://pushpay.com/)

[**Recast**](https://getrecast.com/)

[**TopSport**](https://www.topsport.com.au/)

[**Transformity**](https://transformity.tech/)

[**Trek10**](https://www.trek10.com/)

[**Vertex Pharmaceuticals**](https://www.vrtx.com/)

### Using Lambda Layers

Layers help us understand who uses Powertools for AWS Lambda (Python) in a non-intrusive way.

When [using Layers](#lambda-layer), you can add Powertools for AWS Lambda (Python) as a dev dependency to not impact the development process. For Layers, we pre-package all dependencies, compile and optimize for storage and both x86_64 and ARM architecture.

## Tenets

These are our core principles to guide our decision making.

- **AWS Lambda only**. We optimise for AWS Lambda function environments and supported runtimes only. Utilities might work with web frameworks and non-Lambda environments, though they are not officially supported.
- **Eases the adoption of best practices**. The main priority of the utilities is to facilitate best practices adoption, as defined in the AWS Well-Architected Serverless Lens; all other functionality is optional.
- **Keep it lean**. Additional dependencies are carefully considered for security and ease of maintenance, and prevent negatively impacting startup time.
- **We strive for backwards compatibility**. New features and changes should keep backwards compatibility. If a breaking change cannot be avoided, the deprecation and migration process should be clearly defined.
- **We work backwards from the community**. We aim to strike a balance of what would work best for 80% of customers. Emerging practices are considered and discussed via Requests for Comment (RFCs)
- **Progressive**. Utilities are designed to be incrementally adoptable for customers at any stage of their Serverless journey. They follow language idioms and their community’s common practices.
