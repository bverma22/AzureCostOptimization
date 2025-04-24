# AzureCostOptimization
(https://github.com/user-attachments/assets/f2236519-6970-4d7c-b1bb-1d2d51a6a4f9)


Here's a **100-word summary** of the proposed cost optimization solution:

To reduce costs in a read-heavy Azure serverless billing system, older billing records (over 90 days) are archived from Cosmos DB to Azure Blob Storage. An Azure Function periodically moves this data to lower-cost storage without changing the API interface or causing downtime. When a record is requested, the system first checks Cosmos DB; if not found, it retrieves the record from Blob Storage—ensuring seamless access with minimal latency. This read-through fallback pattern preserves performance while significantly reducing Cosmos DB storage and throughput costs. The approach is simple to implement, cost-efficient, and requires no changes to existing APIs.

# Azure Billing Record Archival

This repo contains:
- An Azure Function that archives Cosmos DB records >90 days to Blob Storage
- A read API that falls back to Blob Storage if records aren't found in Cosmos DB
- Sample billing record data

## Setup

1. Replace keys in `shared/config.py`.
2. Deploy `archive_function` as a timer-triggered Azure Function.
3. Include `read_api.py` logic in your billing record read API.

## Run Locally

Install dependencies:

```bash
pip install -r archive_function/requirements.txt

## Provide workflow for cost optimization in azure (chatgpt)

🔐 shared/config.py
python
Copy
Edit
COSMOS_ENDPOINT = "https://<your-cosmos-db>.documents.azure.com:443/"
COSMOS_KEY = "<your-cosmos-db-key>"
DATABASE_NAME = "billing"
CONTAINER_NAME = "records"

BLOB_CONNECTION_STRING = "<your-blob-storage-connection-string>"
BLOB_CONTAINER_NAME = "billing-archive"
📤 archive_function/__init__.py
python
Copy
Edit
import datetime
import json
import logging
from azure.cosmos import CosmosClient
from azure.storage.blob import BlobServiceClient
from shared.config import *

def main(mytimer) -> None:
    logging.info("Starting archival process")

    cutoff = datetime.datetime.utcnow() - datetime.timedelta(days=90)

    cosmos = CosmosClient(COSMOS_ENDPOINT, COSMOS_KEY)
    container = cosmos.get_database_client(DATABASE_NAME).get_container_client(CONTAINER_NAME)

    blob_service = BlobServiceClient.from_connection_string(BLOB_CONNECTION_STRING)
    blob_container = blob_service.get_container_client(BLOB_CONTAINER_NAME)

    query = "SELECT * FROM c WHERE c.timestamp < @cutoff"
    params = [{"name": "@cutoff", "value": cutoff.isoformat()}]

    for item in container.query_items(query=query, parameters=params, enable_cross_partition_query=True):
        record_id = item["id"]
        timestamp = item["timestamp"]
        folder = timestamp[:7]  # e.g. "2023-11"
        blob_path = f"{folder}/{record_id}.json"

        blob_container.upload_blob(blob_path, json.dumps(item), overwrite=True)
        container.delete_item(record_id, partition_key=item["partitionKey"])

    logging.info("Archival completed.")
🕒 archive_function/function.json
json
Copy
Edit
{
  "scriptFile": "__init__.py",
  "bindings": [
    {
      "name": "mytimer",
      "type": "timerTrigger",
      "direction": "in",
      "schedule": "0 0 1 * * *"  // Runs daily at 1 AM
    }
  ]
}
🔄 api/read_api.py
python
Copy
Edit
import json
from azure.cosmos import CosmosClient
from azure.storage.blob import BlobServiceClient
from shared.config import *

cosmos = CosmosClient(COSMOS_ENDPOINT, COSMOS_KEY)
container = cosmos.get_database_client(DATABASE_NAME).get_container_client(CONTAINER_NAME)

blob_service = BlobServiceClient.from_connection_string(BLOB_CONNECTION_STRING)
blob_container = blob_service.get_container_client(BLOB_CONTAINER_NAME)

def get_billing_record(record_id, timestamp_hint=None):
    try:
        return container.read_item(record_id, partition_key=record_id)
    except:
        if not timestamp_hint:
            raise Exception("Record not found and timestamp_hint missing for Blob lookup.")
        blob_path = f"{timestamp_hint[:7]}/{record_id}.json"
        blob_client = blob_container.get_blob_client(blob_path)
        blob_data = blob_client.download_blob().readall()
        return json.loads(blob_data)
🧪 archive_function/sample_data/billing_sample.json
json
Copy
Edit
{
  "id": "record123",
  "userId": "user456",
  "amount": 49.99,
  "currency": "USD",
  "timestamp": "2023-11-15T10:00:00Z",
  "partitionKey": "user456"
}
📦 archive_function/requirements.txt
txt
Copy
Edit
azure-functions
azure-cosmos
azure-storage-blob
