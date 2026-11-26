# aws

AWS for [Telvax](https://github.com/telvax/telvax): Signature Version 4, the credential chain of the AWS SDKs, and an S3
client. It is a package of the Telvax registry, written in Telvax over `std.crypto`, `std.bytes` and `std.http.client`.

```sh
tvpm add aws
```

```telvax
from std.bytes import Bytes
from aws.s3 import client, put_object, get_object, list_objects

fn upload(year: Int, csv: Bytes) -> Result[Int, Text]:
    let s3 = client()?                                   # credentials and region from the environment
    put_object(s3, "my-bucket", f"bndes/{year}/data.csv", csv, "text/csv")?
    Ok(list_objects(s3, "my-bucket", "bndes/")?.len())
```

## S3 (`aws.s3`)

| Function | What it does |
|---|---|
| `client() -> Result[S3, Text]` | A client from the environment: the credential chain below, `AWS_REGION` (or `AWS_DEFAULT_REGION`), and `AWS_ENDPOINT_URL_S3` (or `AWS_ENDPOINT_URL`) for an S3-compatible server. |
| `S3(region:, credentials:, endpoint:)` | A client made by hand. `endpoint` is `""` for AWS, or the address of MinIO, LocalStack or another compatible server, which is reached path-style. |
| `put_object(s3, bucket, key, body: Bytes, content_type)` | Writes an object. |
| `get_object(s3, bucket, key) -> Result[Bytes, Text]` | Reads an object. |
| `head_object(s3, bucket, key) -> Result[Object?, Text]` | Size, ETag and date, or `None` if there is no such object. |
| `delete_object(s3, bucket, key)` | Deletes an object; a missing one is not an error. |
| `list_objects(s3, bucket, prefix) -> Result[List[Object], Text]` | Every object under `prefix`, following the pages of ListObjectsV2. |
| `create_bucket(s3, bucket)` | Creates a bucket in the region of the client. |

An error answer of S3 is an `Err` with its status, code and message (`S3 answered 403: SignatureDoesNotMatch: ...`).
Keys are encoded as S3 wants, so spaces, `&` and accented letters work.

## Credentials (`aws.credentials`)

`load()` looks where the AWS SDKs look, in the same order, and takes the first it finds:

1. `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` and `AWS_SESSION_TOKEN`. This covers Lambda, which sets them for the
   function's role.
2. The shared file: `AWS_SHARED_CREDENTIALS_FILE` or `~/.aws/credentials`, profile `AWS_PROFILE` or `default`.
3. The container endpoint: `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI` or `..._FULL_URI`, with
   `AWS_CONTAINER_AUTHORIZATION_TOKEN` (`..._FILE`). This covers ECS and EKS Pod Identity.
4. The instance metadata of EC2, with an IMDSv2 token. `AWS_EC2_METADATA_DISABLED=true` skips it.

Temporary credentials are not refreshed: a long-running service should call `client()` again when its credentials expire.

## Signing other services (`aws.sigv4`)

`sign(req: Request, creds, region, service, std.time.now())` gives the headers to send with a request to any AWS service
(`authorization`, `x-amz-date`, and `x-amz-security-token` for temporary credentials). `signature(...)` also gives the
canonical request, to debug a `SignatureDoesNotMatch`.

## Tests

* `telvax test src/sigv4_test.tvx` checks the signer against the examples AWS publishes: `get-vanilla` of the SigV4 test
  suite, and the GET of the S3 documentation.
* `telvax test src/credentials_test.tvx` checks the shared file and the JSON of a role.
* `tests/minio` runs every S3 call against a MinIO server: CI starts one, and `tvpm run main` in that folder runs the
  calls.

Licensed under MIT or Apache-2.0, at your choice.
