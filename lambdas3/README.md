# ****Create two Amazon S3 buckets****

```bash
aws s3api create-bucket --bucket ozgurlambdatest1234 --region eu-west-1 \
--create-bucket-configuration LocationConstraint=eu-west-1

aws s3api create-bucket --bucket ozgurlambdatest1234-resized --region eu-west-1 \
--create-bucket-configuration LocationConstraint=eu-west-1
```

****Upload a test image to your source bucket****

```bash
aws s3api put-object --bucket ozgurlambdatest1234 --key icarditest.jpg --body ./icarditest.jpg
aws s3api list-objects-v2 --bucket ozgurlambdatest1234
```

****Create a permissions policy****

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:PutLogEvents",
                "logs:CreateLogGroup",
                "logs:CreateLogStream"
            ],
            "Resource": "arn:aws:logs:*:*:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": "arn:aws:s3:::*/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::*/*"
        }
    ]
}
```

```bash
aws iam create-policy --policy-name LambdaS3Policy --policy-document file://policy.json
```

****Create an execution role****

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

```bash
aws iam create-role --role-name LambdaS3Role --assume-role-policy-document file://trust-policy.json
aws iam attach-role-policy --role-name LambdaS3Role --policy-arn arn:aws:iam::766685550559:policy/LambdaS3Policy
```

****Create the function deployment package****

```python
import boto3
import os
import sys
import uuid
from urllib.parse import unquote_plus
from PIL import Image
import PIL.Image
            
s3_client = boto3.client('s3')
            
def resize_image(image_path, resized_path):
  with Image.open(image_path) as image:
    image.thumbnail(tuple(x / 2 for x in image.size))
    image.save(resized_path)
            
def lambda_handler(event, context):
  for record in event['Records']:
    bucket = record['s3']['bucket']['name']
    key = unquote_plus(record['s3']['object']['key'])
    tmpkey = key.replace('/', '')
    download_path = '/tmp/{}{}'.format(uuid.uuid4(), tmpkey)
    upload_path = '/tmp/resized-{}'.format(tmpkey)
    s3_client.download_file(bucket, key, download_path)
    resize_image(download_path, upload_path)
    s3_client.upload_file(upload_path, '{}-resized'.format(bucket), 'resized-{}'.format(key))
```

```bash
mkdir package
pip install \
--platform manylinux2014_x86_64 \
--target=package \
--implementation cp \
--python-version 3.9 \
--only-binary=:all: --upgrade \
pillow boto3

cd package
zip -r ../lambda_function.zip .
cd ..
zip lambda_function.zip lambda_function.py
```

****Create the Lambda function****

```bash
aws lambda create-function --function-name CreateThumbnail \
--zip-file fileb://lambda_function.zip --handler lambda_function.lambda_handler \
--runtime python3.9 --timeout 10 --memory-size 1024 \
--role arn:aws:iam::766685550559:role/LambdaS3Role --region eu-west-1
```

****Configure Amazon S3 to invoke the function****

```bash
aws lambda add-permission --function-name CreateThumbnail \
--principal s3.amazonaws.com --statement-id s3invoke --action "lambda:InvokeFunction" \
--source-arn arn:aws:s3:::ozgurlambdatest1234 \
--source-account 766685550559
```

```json
{
"LambdaFunctionConfigurations": [
    {
      "Id": "CreateThumbnailEventConfiguration",
      "LambdaFunctionArn": "arn:aws:lambda:us-west-2:123456789012:function:CreateThumbnail",
      "Events": [ "s3:ObjectCreated:Put" ]
    }
  ]
}
```

```json
aws s3api put-bucket-notification-configuration --bucket ozgurlambdatest1234 --notification-configuration file://notification.json
```

****Test your function using the Amazon S3 trigger****

```json
aws s3api put-object --bucket ozgurlambdatest1234 --key icardi.jpg --body ./icardi.jpg
aws s3api list-objects-v2 --bucket ozgurlambdatest1234-resized
```
