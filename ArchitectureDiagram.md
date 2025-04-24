# AzureCostOptimization
(https://github.com/user-attachments/assets/f2236519-6970-4d7c-b1bb-1d2d51a6a4f9)

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
