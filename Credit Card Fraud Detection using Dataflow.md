---

# Credit Card Fraud Detection Data Pipeline

This repository demonstrates the development of a batch processing pipeline using **Apache Beam** and **Google Cloud Dataflow** to process and load data into **BigQuery**. The pipeline generates fake credit card transaction data, processes it, and loads it into BigQuery for further analysis.

---

## Project Outline

1. **Setup Environment**
    - Create a Google Cloud project.
    - Enable necessary APIs.
    - Create service accounts and assign permissions.
    - Install required tools and dependencies.

2. **Data Generation**
    - Generate a dataset with 50,000 rows of fake credit card transaction data.

3. **Google Cloud Setup**
    - Create Google Cloud Storage (GCS) buckets.
    - Create BigQuery datasets and tables.

4. **Pipeline Development**
    - Build the Apache Beam pipeline to:
        - Read data from GCS.
        - Parse and transform it.
        - Write processed data to BigQuery.

5. **Execution**
    - Run the pipeline using Google Cloud Dataflow.

6. **Verification**
    - Verify data in BigQuery.
    - Query and validate results.

7. **Challenges Faced**
    - Parsing issues.
    - Permission issues for BigQuery jobs.

---

## Step-by-Step Guide

### 1. Setup Environment

1. **Create a Google Cloud Project**:
   - Navigate to [Google Cloud Console](https://console.cloud.google.com/).
   - Create a new project (e.g., `big-data-services`).

2. **Enable Necessary APIs**:
   ```bash
   gcloud services enable dataflow.googleapis.com
   gcloud services enable bigquery.googleapis.com
   gcloud services enable storage.googleapis.com
   ```

3. **Create a Service Account**:
   ```bash
   gcloud iam service-accounts create dataflow-batch-sa \
     --description="Service Account for Dataflow Batch Jobs" \
     --display-name="Dataflow Batch SA"
   ```

4. **Assign Roles to the Service Account**:
   ```bash
   gcloud projects add-iam-policy-binding <PROJECT_ID> \
     --member="serviceAccount:dataflow-batch-sa@<PROJECT_ID>.iam.gserviceaccount.com" \
     --role="roles/dataflow.admin"
   gcloud projects add-iam-policy-binding <PROJECT_ID> \
     --member="serviceAccount:dataflow-batch-sa@<PROJECT_ID>.iam.gserviceaccount.com" \
     --role="roles/bigquery.admin"
   gcloud projects add-iam-policy-binding <PROJECT_ID> \
     --member="serviceAccount:dataflow-batch-sa@<PROJECT_ID>.iam.gserviceaccount.com" \
     --role="roles/storage.admin"
   ```

5. **Install Required Tools**:
   - Install the [Google Cloud CLI](https://cloud.google.com/sdk/docs/install).
   - Install Python and dependencies:
     ```bash
     pip install apache-beam[gcp]
     pip install faker
     ```

---

### 2. Data Generation

Use Python's **Faker** library to generate fake credit card transaction data. Below is a script to generate a CSV file with 50,000 rows:

```python
import csv
from faker import Faker
import random

fake = Faker()

def generate_data(file_name, num_rows):
    with open(file_name, mode='w', newline='') as file:
        writer = csv.writer(file)
        writer.writerow(["transaction_id", "card_number", "transaction_date", "amount", "merchant", "is_fraud"])
        merchants = ["Amazon", "Ebay", "Best Buy", "Apple Store"]
        for _ in range(num_rows):
            writer.writerow([
                fake.uuid4(),
                fake.credit_card_number(),
                fake.date_time_this_month(),
                round(random.uniform(10.0, 1000.0), 2),
                random.choice(merchants),
                random.choice([True, False])
            ])

generate_data("fraud_transactions.csv", 50000)
```

---

### 3. Google Cloud Setup

1. **Create GCS Buckets**:
   ```bash
   gsutil mb gs://credit-card-fraud-bucket
   gsutil mb gs://credit-card-fraud-bucket/tmp
   ```

2. **Upload the CSV File to GCS**:
   ```bash
   gsutil cp fraud_transactions.csv gs://credit-card-fraud-bucket/
   ```

3. **Create BigQuery Dataset and Table**:
   ```bash
   bq mk fraud_detection_dataset
   bq mk --table fraud_detection_dataset.transactions \
   transaction_id:STRING,card_number:STRING,transaction_date:STRING,amount:FLOAT,merchant:STRING,is_fraud:BOOLEAN
   ```

---

### 4. Pipeline Development

Create the following Python script (`fraud_pipeline.py`):

```python
import argparse
import logging
import apache_beam as beam
from apache_beam.io import ReadFromText
from apache_beam.options.pipeline_options import PipelineOptions
import csv

class ParseCSV(beam.DoFn):
    def process(self, element):
        try:
            reader = csv.reader([element])
            for row in reader:
                return [{
                    "transaction_id": row[0],
                    "card_number": row[1],
                    "transaction_date": row[2],
                    "amount": float(row[3]),
                    "merchant": row[4],
                    "is_fraud": row[5].lower() == "true"
                }]
        except Exception as e:
            logging.error(f"Error parsing row: {element}, Error: {e}")
            return []

def run():
    parser = argparse.ArgumentParser()
    parser.add_argument('--input', dest='input', required=True, help='Input file path')
    known_args, pipeline_args = parser.parse_known_args()

    with beam.Pipeline(options=PipelineOptions(pipeline_args)) as pipeline:
        (
            pipeline
            | 'Read CSV' >> ReadFromText(known_args.input, skip_header_lines=1)
            | 'Parse CSV' >> beam.ParDo(ParseCSV())
            | 'Write to BigQuery' >> beam.io.WriteToBigQuery(
                table='transactions',
                dataset='fraud_detection_dataset',
                project='<PROJECT_ID>',
                schema='transaction_id:STRING,card_number:STRING,transaction_date:STRING,amount:FLOAT,merchant:STRING,is_fraud:BOOLEAN',
                create_disposition=beam.io.BigQueryDisposition.CREATE_IF_NEEDED,
                write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND
            )
        )

if __name__ == '__main__':
    logging.getLogger().setLevel(logging.INFO)
    run()
```

---

### 5. Execution

Run the pipeline using the following command:

```bash
python fraud_pipeline.py \
  --input gs://credit-card-fraud-bucket/fraud_transactions.csv \
  --project big-data-services-449202 \
  --region us-central1 \
  --temp_location gs://credit-card-fraud-bucket/tmp \
  --runner DataflowRunner \
  --job_name fraud-detection-job
```

---

### 6. Verification

1. **Verify Table Exists**:
   ```bash
   bq show --format=prettyjson fraud_detection_dataset.transactions
   ```

2. **Count Rows**:
   ```bash
   bq query --use_legacy_sql=false "SELECT COUNT(*) AS total_rows FROM \`big-data-services-449202.fraud_detection_dataset.transactions\`"
   ```

3. **Preview Data**:
   ```bash
   bq query --use_legacy_sql=false "SELECT * FROM \`big-data-services-449202.fraud_detection_dataset.transactions\` LIMIT 10"
   ```

---

### 7. Challenges Faced

1. **CSV Parsing Errors**:
   - **Issue**: Rows were skipped due to invalid CSV parsing.
   - **Fix**: Added error handling in the `ParseCSV` function.

2. **BigQuery Permission Error**:
   - **Issue**: `User does not have bigquery.jobs.create permission`.
   - **Fix**: Updated IAM roles for the service account with `BigQuery Admin` permissions.

3. **Missing Data in BigQuery**:
   - **Issue**: Data was not appearing in BigQuery.
   - **Fix**: Verified pipeline execution logs and adjusted pipeline logic for schema and transformations.

---

### Repository Structure

```
├── fraud_pipeline.py          # Apache Beam pipeline script
├── fraud_transactions.csv     # Generated fake data
├── README.md                  # Project documentation
└── requirements.txt           # Python dependencies
```

---

### Conclusion

This project demonstrates how to design, implement, and troubleshoot a batch data processing pipeline using Google Cloud services. The resulting data in BigQuery can now be used for fraud detection analysis. 🎉
