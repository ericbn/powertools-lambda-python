The streaming utility handles datasets larger than the available memory as streaming data.

## Key features

- Stream Amazon S3 objects with a file-like interface with minimal memory consumption
- Built-in popular data transformations to decompress and deserialize (gzip, CSV, and ZIP)
- Build your own data transformation and add it to the pipeline

## Background

Within Lambda, processing S3 objects larger than the allocated amount of memory can lead to out of memory or timeout situations. For cost efficiency, your S3 objects may be encoded and compressed in various formats (*gzip, CSV, zip files, etc*), increasing the amount of non-business logic and reliability risks.

Streaming utility makes this process easier by fetching parts of your data as you consume it, and transparently applying data transformations to the data stream. This allows you to process one, a few, or all rows of your large dataset while consuming a few MBs only.

## Getting started

### Streaming from a S3 object

With `S3Object`, you'll need the bucket, object key, and optionally a version ID to stream its content.

We will fetch parts of your data from S3 as you process each line, consuming only the absolute minimal amount of memory.

```
from typing import Dict

from aws_lambda_powertools.utilities.streaming.s3_object import S3Object
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: Dict[str, str], context: LambdaContext):
    s3 = S3Object(bucket=event["bucket"], key=event["key"])
    for line in s3:
        print(line)

```

```
from typing import Dict

from aws_lambda_powertools.utilities.streaming.s3_object import S3Object
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: Dict[str, str], context: LambdaContext):
    s3 = S3Object(bucket=event["bucket"], key=event["key"], version_id=event["version_id"])
    for line in s3:
        print(line)

```

### Data transformations

Think of data transformations like a data processing pipeline - apply one or more in order.

As data is streamed, you can apply transformations to your data like decompressing gzip content and deserializing a CSV into a dictionary.

For popular data transformations like CSV or Gzip, you can quickly enable it at the constructor level:

```
from typing import Dict

from aws_lambda_powertools.utilities.streaming.s3_object import S3Object
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: Dict[str, str], context: LambdaContext):
    s3 = S3Object(bucket=event["bucket"], key=event["key"], is_gzip=True, is_csv=True)
    for line in s3:
        print(line)

```

Alternatively, you can apply transformations later via the `transform` method. By default, it will return the transformed stream you can use to read its contents. If you prefer in-place modifications, use `in_place=True`.

When is this useful?

In scenarios where you might have a reusable logic to apply common transformations. This might be a function or a class that receives an instance of `S3Object`.

```
from typing import Dict

from aws_lambda_powertools.utilities.streaming.s3_object import S3Object
from aws_lambda_powertools.utilities.streaming.transformations import (
    CsvTransform,
    GzipTransform,
)
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: Dict[str, str], context: LambdaContext):
    s3 = S3Object(bucket=event["bucket"], key=event["key"])
    data = s3.transform([GzipTransform(), CsvTransform()])
    for line in data:
        print(line)  # returns a dict

```

Note that when using `in_place=True`, there is no return (`None`).

```
from typing import Dict

from aws_lambda_powertools.utilities.streaming.s3_object import S3Object
from aws_lambda_powertools.utilities.streaming.transformations import (
    CsvTransform,
    GzipTransform,
)
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: Dict[str, str], context: LambdaContext):
    s3 = S3Object(bucket=event["bucket"], key=event["key"])
    s3.transform([GzipTransform(), CsvTransform()], in_place=True)
    for line in s3:
        print(line)  # returns a dict

```

#### Handling ZIP files

`ZipTransform` doesn't support combining other transformations.

This is because a Zip file contains multiple files while transformations apply to a single stream.

That said, you can still open a specific file as a stream, reading only the necessary bytes to extract it:

```
from aws_lambda_powertools.utilities.streaming import S3Object
from aws_lambda_powertools.utilities.streaming.transformations import ZipTransform

s3object = S3Object(bucket="bucket", key="key")
zip_reader = s3object.transform(ZipTransform())
with zip_reader.open("filename.txt") as f:
    for line in f:
        print(line)

```

#### Built-in data transformations

We provide popular built-in transformations that you can apply against your streaming data.

| Name | Description | Class name | | --- | --- | --- | | **Gzip** | Gunzips the stream of data using the [gzip library](https://docs.python.org/3/library/gzip.html) | GzipTransform | | **Zip** | Exposes the stream as a [ZipFile object](https://docs.python.org/3/library/zipfile.html) | ZipTransform | | **CSV** | Parses each CSV line as a CSV object, returning dictionary objects | CsvTransform |

## Advanced

### Skipping or reading backwards

`S3Object` implements [Python I/O interface](https://docs.python.org/3/tutorial/inputoutput.html). This means you can use `seek` to start reading contents of your file from any particular position, saving you processing time.

#### Reading backwards

For example, let's imagine you have a large CSV file, each row has a non-uniform size (bytes), and you want to read and process the last row only.

```
id,name,location
1,Ruben Fonseca, Denmark
2,Heitor Lessa, Netherlands
3,Leandro Damascena, Portugal

```

You found out the last row has exactly 30 bytes. We can use `seek()` to skip to the end of the file, read 30 bytes, then transform to CSV.

```
import io
from typing import Dict

from aws_lambda_powertools.utilities.streaming.s3_object import S3Object
from aws_lambda_powertools.utilities.streaming.transformations import CsvTransform
from aws_lambda_powertools.utilities.typing import LambdaContext

LAST_ROW_SIZE = 30
CSV_HEADERS = ["id", "name", "location"]


def lambda_handler(event: Dict[str, str], context: LambdaContext):
    sample_csv = S3Object(bucket=event["bucket"], key="sample.csv")

    # From the end of the file, jump exactly 30 bytes backwards
    sample_csv.seek(-LAST_ROW_SIZE, io.SEEK_END)

    # Transform portion of data into CSV with our headers
    sample_csv.transform(CsvTransform(fieldnames=CSV_HEADERS), in_place=True)

    # We will only read the last portion of the file from S3
    # as we're only interested in the last 'location' from our dataset
    for last_row in sample_csv:
        print(last_row["location"])

```

#### Skipping

What if we want to jump the first N rows?

You can also solve with `seek`, but let's take a large uniform CSV file to make this easier to grasp.

```
reading,position,type
21.3,5,+
23.4,4,+
21.3,0,-

```

You found out that each row has 8 bytes, the header line has 21 bytes, and every new line has 1 byte. You want to skip the first 100 lines.

```
import io
from typing import Dict

from aws_lambda_powertools.utilities.streaming.s3_object import S3Object
from aws_lambda_powertools.utilities.streaming.transformations import CsvTransform
from aws_lambda_powertools.utilities.typing import LambdaContext

"""
Assuming the CSV files contains rows after the header always has 8 bytes + 1 byte newline:

reading,position,type
21.3,5,+
23.4,4,+
21.3,0,-
...
"""

CSV_HEADERS = ["reading", "position", "type"]
ROW_SIZE = 8 + 1  # 1 byte newline
HEADER_SIZE = 21 + 1  # 1 byte newline
LINES_TO_JUMP = 100


def lambda_handler(event: Dict[str, str], context: LambdaContext):
    sample_csv = S3Object(bucket=event["bucket"], key=event["key"])

    # Skip the header line
    sample_csv.seek(HEADER_SIZE, io.SEEK_SET)

    # Jump 100 lines of 9 bytes each (8 bytes of data + 1 byte newline)
    sample_csv.seek(LINES_TO_JUMP * ROW_SIZE, io.SEEK_CUR)

    sample_csv.transform(CsvTransform(), in_place=True)
    for row in sample_csv:
        print(row["reading"])

```

### Custom options for data transformations

We will propagate additional options to the underlying implementation for each transform class.

| Name | Available options | | --- | --- | | **GzipTransform** | [GzipFile constructor](https://docs.python.org/3/library/gzip.html#gzip.GzipFile) | | **ZipTransform** | [ZipFile constructor](https://docs.python.org/3/library/zipfile.html#zipfile.ZipFile) | | **CsvTransform** | [DictReader constructor](https://docs.python.org/3/library/csv.html#csv.DictReader) |

For instance, take `ZipTransform`. You can use the `compression` parameter if you want to unzip an S3 object compressed with `LZMA`.

```
import zipfile
from typing import Dict

from aws_lambda_powertools.utilities.streaming.s3_object import S3Object
from aws_lambda_powertools.utilities.streaming.transformations import ZipTransform
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: Dict[str, str], context: LambdaContext):
    s3 = S3Object(bucket=event["bucket"], key=event["key"])

    zf = s3.transform(ZipTransform(compression=zipfile.ZIP_LZMA))

    print(zf.nameslist())
    zf.extract(zf.namelist()[0], "/tmp")

```

Or, if you want to load a tab-separated file (TSV), you can use the `delimiter` parameter in the `CsvTransform`:

```
from typing import Dict

from aws_lambda_powertools.utilities.streaming.s3_object import S3Object
from aws_lambda_powertools.utilities.streaming.transformations import CsvTransform
from aws_lambda_powertools.utilities.typing import LambdaContext


def lambda_handler(event: Dict[str, str], context: LambdaContext):
    s3 = S3Object(bucket=event["bucket"], key=event["key"])

    tsv_stream = s3.transform(CsvTransform(delimiter="\t"))
    for obj in tsv_stream:
        print(obj)

```

### Building your own data transformation

You can build your own custom data transformation by extending the `BaseTransform` class. The `transform` method receives an `IO[bytes]` object, and you are responsible for returning an `IO[bytes]` object.

```
import io
from typing import IO, Optional

import ijson

from aws_lambda_powertools.utilities.streaming.transformations import BaseTransform


# Using io.RawIOBase gets us default implementations of many of the common IO methods
class JsonDeserializer(io.RawIOBase):
    def __init__(self, input_stream: IO[bytes]):
        self.input = ijson.items(input_stream, "", multiple_values=True)

    def read(self, size: int = -1) -> Optional[bytes]:
        raise NotImplementedError(f"{__name__} does not implement read")

    def readline(self, size: Optional[int] = None) -> bytes:
        raise NotImplementedError(f"{__name__} does not implement readline")

    def read_object(self) -> dict:
        return self.input.__next__()

    def __next__(self):
        return self.read_object()


class JsonTransform(BaseTransform):
    def transform(self, input_stream: IO[bytes]) -> JsonDeserializer:
        return JsonDeserializer(input_stream=input_stream)

```

## Testing your code

### Asserting data transformations

Create an input payload using `io.BytesIO` and assert the response of the transformation:

```
import io

import boto3
from assert_transformation_module import UpperTransform
from botocore import stub

from aws_lambda_powertools.utilities.streaming import S3Object
from aws_lambda_powertools.utilities.streaming.compat import PowertoolsStreamingBody


def test_upper_transform():
    # GIVEN
    data_stream = io.BytesIO(b"hello world")
    # WHEN
    data_stream = UpperTransform().transform(data_stream)
    # THEN
    assert data_stream.read() == b"HELLO WORLD"


def test_s3_object_with_upper_transform():
    # GIVEN
    payload = b"hello world"
    s3_client = boto3.client("s3")
    s3_stub = stub.Stubber(s3_client)
    s3_stub.add_response(
        "get_object",
        {"Body": PowertoolsStreamingBody(raw_stream=io.BytesIO(payload), content_length=len(payload))},
    )
    s3_stub.activate()

    # WHEN
    data_stream = S3Object(bucket="bucket", key="key", boto3_client=s3_client)
    data_stream.transform(UpperTransform(), in_place=True)

    # THEN
    assert data_stream.read() == b"HELLO WORLD"

```

```
import io
from typing import IO, Optional

from aws_lambda_powertools.utilities.streaming.transformations import BaseTransform


class UpperIO(io.RawIOBase):
    def __init__(self, input_stream: IO[bytes], encoding: str):
        self.encoding = encoding
        self.input_stream = io.TextIOWrapper(input_stream, encoding=encoding)

    def read(self, size: int = -1) -> Optional[bytes]:
        data = self.input_stream.read(size)
        return data.upper().encode(self.encoding)


class UpperTransform(BaseTransform):
    def transform(self, input_stream: IO[bytes]) -> UpperIO:
        return UpperIO(input_stream=input_stream, encoding="utf-8")

```

## Known limitations

### AWS X-Ray segment size limit

We make multiple API calls to S3 as you read chunks from your S3 object. If your function is decorated with [Tracer](../../core/tracer/), you can easily hit [AWS X-Ray 64K segment size](https://docs.aws.amazon.com/general/latest/gr/xray.html#limits_xray) when processing large files.

Use tracer decorators in parts where you don't read your `S3Object` instead.
