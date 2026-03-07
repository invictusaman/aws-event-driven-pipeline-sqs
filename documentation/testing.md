# Testing Steps

Follow these 5 steps to verify the pipeline works end-to-end.

---

## Step 1 — Upload Dataset to S3 Raw Folder

1. Go to **S3 → sales-data-bucket-sqs → sales-data/raw/**
2. Click **Upload → Add Files**
3. Select `data/sales_data.csv`
4. Click **Upload**

**Expected:** File appears at `sales-data/raw/sales_data.csv`

Take a screenshot → save as `screenshots/s3_output.png`

---

## Step 2 — Send Message to SQS Queue

1. Go to **SQS → sales-pipeline-queue**
2. Click **Send and receive messages**
3. Paste this into the Message body field:

```json
{
  "file_name": "sales_data.csv",
  "source_path": "sales-data/raw/",
  "destination_path": "sales-data/processed/"
}
```

4. Click **Send Message**

**Expected:** Messages Available briefly shows 1, then drops to 0 as Lambda consumes it

Take a screenshot → save as `screenshots/sqs_queue.png`

---

## Step 3 — Verify Lambda Execution

1. Go to **CloudWatch → Log Groups → /aws/lambda/sales-pipeline-trigger**
2. Open the latest Log Stream
3. Confirm these log entries exist:

| Log Entry                     | Meaning                         |
| ----------------------------- | ------------------------------- |
| `Event: {"Records": [...]}`   | Lambda received the SQS message |
| `Message: {"file_name": ...}` | Message parsed correctly        |
| `Glue JobRunId: jr_...`       | Glue job started successfully   |

Take a screenshot of the log stream → save as `screenshots/lambda_trigger.png`

**Troubleshooting:** If no logs appear, check the SQS trigger is enabled on the Lambda function and the IAM role has `AWSGlueServiceRole`.

---

## Step 4 — Verify Glue Job Execution

1. Go to **AWS Glue → ETL Jobs → sales-data-processor**
2. Click the **Runs** tab
3. Find the most recent run — status should be **Succeeded**

Take a screenshot of the Runs tab → save as `screenshots/glue_job.png`

**Troubleshooting:** If the job fails, open the run logs and look for errors related to S3 paths or IAM permissions.

---

## Step 5 — Confirm Processed Output in S3

1. Go to **S3 → sales-data-bucket-sqs → sales-data/processed/**
2. Confirm one or more `.parquet` files are present

**Expected output columns:**

| Column         | Notes                            |
| -------------- | -------------------------------- |
| order_id       | Original                         |
| customer_id    | Original                         |
| product        | Original                         |
| category       | Original                         |
| quantity       | Original                         |
| price          | Original                         |
| order_date     | Original                         |
| city           | Original                         |
| payment_method | Original                         |
| status         | Standardised                     |
| total_amount   | **New** — quantity × price       |
| order_size     | **New** — Large / Medium / Small |

Take a screenshot of the processed folder → save as `screenshots/s3_output.png`

---

## Summary Checklist

| Step | Service | Pass Condition                                         |
| ---- | ------- | ------------------------------------------------------ |
| 1    | S3      | CSV visible in `raw/`                                  |
| 2    | SQS     | Message sent; queue count returns to 0                 |
| 3    | Lambda  | CloudWatch logs show Glue JobRunId                     |
| 4    | Glue    | Job run status: Succeeded                              |
| 5    | S3      | Parquet files present in `processed/` with new columns |
