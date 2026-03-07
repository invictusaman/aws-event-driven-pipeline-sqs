# Setup Steps

Follow these steps in order to configure the pipeline in your AWS account.

---

## A. Create the S3 Bucket

1. Go to **AWS Console → S3 → Create Bucket**
2. Bucket name: `sales-data-bucket-sqs` (must be globally unique)
3. Region: `us-east-1` — keep this consistent across all services
4. Leave all other settings default → click **Create Bucket**
5. Open the bucket → **Create Folder** → name it `sales-data`
6. Inside `sales-data`, create two sub-folders: `raw` and `processed`
7. Upload `data/sales_data.csv` to `sales-data/raw/`

---

## B. Create the SQS Queue

1. Go to **AWS Console → SQS → Create Queue**
2. Type: **Standard**
3. Name: `sales-pipeline-queue`
4. Leave all settings default → click **Create Queue**
5. Note the **Queue URL** and **Queue ARN** for later

---

## C. Create the IAM Role for Lambda

1. Go to **IAM → Roles → Create Role**
2. Trusted entity: **AWS Service** → Use case: **Lambda** → Next
3. Attach these policies:

| Policy                 | Purpose                             |
| ---------------------- | ----------------------------------- |
| `AmazonSQSFullAccess`  | Read and delete SQS messages        |
| `AmazonS3FullAccess`   | Read raw data, write processed data |
| `AWSGlueServiceRole`   | Start Glue jobs                     |
| `CloudWatchFullAccess` | Write Lambda logs                   |

4. Role name: `lambda-pipeline-role` → **Create Role**

---

## D. Create the AWS Glue Job

1. Go to **AWS Glue → ETL Jobs → Script Editor**
2. Engine: **Spark** → **Create**
3. Paste the Glue script below into the editor:

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.sql.functions import col, when, round as spark_round

args = getResolvedOptions(sys.argv, [
    'JOB_NAME', 'file_name', 'source_path', 'destination_path', 'S3_BUCKET'
])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

source = f"s3://{args['S3_BUCKET']}/{args['source_path']}{args['file_name']}"
dest   = f"s3://{args['S3_BUCKET']}/{args['destination_path']}"

df = spark.read.option("header","true").option("inferSchema","true").csv(source)

df = df.withColumn("total_amount", spark_round(col("quantity") * col("price"), 2))
df = df.withColumn("order_size",
    when(col("total_amount") >= 500, "Large")
    .when(col("total_amount") >= 100, "Medium")
    .otherwise("Small"))

df_out = df.filter(col("status") == "Completed")
df_out.write.mode("overwrite").option("header","true").parquet(dest)

print("Done. Written to:", dest)
job.commit()
```

4. Job name: `sales-data-processor`
5. IAM Role: create/select a Glue service role with `AmazonS3FullAccess`
6. Click **Save**

---

## E. Create the Lambda Function

1. Go to **Lambda → Create Function → Author from scratch**
2. Function name: `sales-pipeline-trigger`
3. Runtime: **Python 3.12**
4. Execution role: Use existing → select `lambda-pipeline-role`
5. Click **Create Function**
6. In the **Code** tab, replace the default code with:

```python
import json
import boto3

glue = boto3.client('glue')

def lambda_handler(event, context):
    print("Event:", json.dumps(event))
    for record in event['Records']:
        msg = json.loads(record['body'])
        print("Message:", msg)
        response = glue.start_job_run(
            JobName='sales-data-processor',
            Arguments={
                '--file_name':        msg['file_name'],
                '--source_path':      msg['source_path'],
                '--destination_path': msg['destination_path'],
                '--S3_BUCKET':        'sales-data-bucket-sqs'
            }
        )
        print("Glue JobRunId:", response['JobRunId'])
    return {'statusCode': 200, 'body': 'Success'}
```

7. Update `sales-data-bucket-sqs` with your actual bucket name
8. Click **Deploy**

### Add SQS Trigger

1. Click **Add Trigger** inside the Lambda function
2. Source: **SQS**
3. Queue: `sales-pipeline-queue`
4. Batch size: `1`
5. Click **Add**

---

## F. Verify All Resources

Before testing, confirm:

- S3 bucket has `sales-data/raw/sales_data.csv`
- SQS queue `sales-pipeline-queue` exists
- Lambda function has SQS trigger active
- Glue job `sales-data-processor` is saved and shows status _Ready_
- IAM role attached to Lambda includes all four policies

See `testing.md` for end-to-end verification.
