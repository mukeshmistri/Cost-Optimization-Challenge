
---

# Dynamic Data Copying Solution Using Azure Data Factory

## Summary

This document outlines a robust solution for dynamically managing the copying of data from Azure Cosmos DB to Azure Blob Storage using Azure Data Factory (ADF), Azure Functions, and Azure Monitor. The workflow is designed to ensure data integrity through hash validation, adjust processing based on current Request Unit (RU) usage, and handle errors gracefully, all while maintaining operational efficiency.

## Known Information and Context

### Service
- **Service**: Azure Cosmos DB with serverless architecture.

### Existing Setup
- **Data Volume**: 600 GB of data exists in Cosmos DB.
- **Data Breakdown**: There are 2 million records, each approximately 300 KB.

### Considerations
1. The system is read-heavy, but records older than 3 months are rarely accessed.

### Challenge
- Efficient way to reduce cost while maintaining data availability.

### Requirements
1. The proposed solution should be simple and easy to deploy and maintain.
2. No data loss and no downtime - the transition should be seamless without losing any records or requiring service downtime.
3. No changes to API contracts - the existing read/write API for billing records must remain unchanged.

### Additional Constraints
1. We have 5000 RU/s available, with almost 50% used on a day-to-day basis.
2. To process a record of 300 KB, it will consume approximately 300 RU.

### Specific Challenges that can be encountered
1. How do we migrate the data from Cosmos DB to the storage account?
2. How do we validate that the files are not corrupted?
3. How much time will it take?
4. Business should not be impacted, and we should remain within safe limits.
5. How can we avoid errors or the API becoming unavailable for clients?
6. How do we monitor the risk that, on the day we are planning to migrate data, the business is experiencing a lot of read-heavy activity?
7. What other things can go wrong?

### Additional Information (Assumed)
- The Azure Cosmos DB used is **Azure Cosmos DB for NoSQL**.
- The data in the storage account is copied in **Parquet format** to ensure lower latencies.

## Solution Overview

1. **Event-Driven RU Monitoring**:
   - Set up an Azure Monitor alert that triggers an Azure Function when RU usage exceeds a specified threshold (e.g., 80%).
   - Reference the document MonitorRUUsage.md for exact configuration.

2. **Trigger ADF Pipeline**:
   - If RU usage is below the threshold, the Azure Function triggers an ADF pipeline to copy data from Azure Cosmos DB to Azure Blob Storage.

3. **Dynamic Batch Size Adjustment**:
   - In parallel, Azure Functions monitor RU consumption and adjust the batch size of the copy activity to ensure the load remains below 80%. This is achieved using the Azure Data Factory REST API to update the `batchSize` parameter dynamically.

4. **Hash Validation**:
   - After each copy operation, an Azure Function validates the hash of the copied data against the expected hash. If the validation fails, the function logs the error, sends notifications, and implements retry logic.

5. **Halt Pipeline on Persistent Failure**:
   - If hash validation continues to fail after retries, halt the pipeline using a status variable (e.g., “Running”, “Failed”, “Halted”) and a conditional check.

6. **Continue Running the Pipeline**:
   - If the hash validation is successful, the pipeline continues to run, dynamically monitoring RU usage and resizing the batch load as necessary.

## Implementation Steps

### 1. Set Up Azure Resources

- **Azure Function App**: Create a new Function App to handle RU monitoring and hash validation.
- **Azure Data Factory**: Create an ADF instance to manage the data copy process.
- **Azure Blob Storage**: Set up a Blob Storage account for storing copied data.

### 2. Azure Function for Monitoring RU Usage

- Implement an Azure Function to check RU usage and trigger the ADF pipeline. Use Azure Monitor to create alerts based on RU consumption metrics.

### 3. Trigger ADF Pipeline and Monitor Performance

- Use the Azure Data Factory REST API to start the pipeline and adjust batch sizes based on monitored RU usage dynamically.

### 4. Hash Validation Function

- Create an Azure Function that calculates the hash of copied data and compares it with the expected hash. Implement error logging and notifications for hash validation failures.

```
import requests
from azure.identity import DefaultAzureCredential

def update_adf_batch_size(pipeline_name, resource_group, factory_name, new_batch_size):
    # Get an access token
    credential = DefaultAzureCredential()
    token = credential.get_token("https://management.azure.com/.default")
    access_token = token.token

    # Prepare the API call
    url = f"https://management.azure.com/subscriptions/<Your Subscription ID>/resourceGroups/{resource_group}/providers/Microsoft.DataFactory/factories/{factory_name}/pipelines/{pipeline_name}/createRun?api-version=2018-06-01"
    headers = {
        "Authorization": f"Bearer {access_token}",
        "Content-Type": "application/json"
    }
    body = {
        "referencePipelineRunId": "<PipelineRunID>",  # Optional: to reference a specific run
        "parameters": {
            "batchSize": new_batch_size  # Assuming your pipeline has a parameter named 'batchSize'
        }
    }

    # Make the API request
    response = requests.post(url, headers=headers, json=body)
    if response.status_code == 200:
        print(f"Successfully updated batch size to {new_batch_size}.")
    else:
        print(f"Failed to update batch size: {response.content}")
```

### 5. Pipeline Management

- Use Set Variable activities in ADF to maintain a status variable (e.g., `pipelineStatus`), and use Conditional activities to halt the pipeline if validation fails continuously.

- Implementing Terminate Logic:
  In the Condition settings, set the expression to check the variable:

  ```
  @equals(variables('pipelineStatus'), 'Failed')
  ```

### 6. Batch Size Control

- Control the batch size based on the number of records, ensuring that 300 KB per file is accounted for during the copying process.
- Example Calculation:

  If your copy operation can use up to 1,000 RU for a batch and a record size of 300 KB consumes approximately 300 RU, you could calculate:
  
  ```
  max_records = max_ru / ru_per_record  # e.g., 1000 RU / 300 RU = 3.33 records
  ```

## Risks and Mitigation Strategies

### Accepted Risks

1. **High RU Usage**:
   - **Mitigation**: Implement dynamic batch size adjustments and monitor RU usage continuously to avoid hitting limits.

2. **Data Corruption During Copy**:
   - **Mitigation**: Use robust hash validation and retry logic to ensure data integrity.

3. **Pipeline Failures or Timeouts**:
   - **Mitigation**: Establish comprehensive error handling within the ADF pipeline to manage failures gracefully.

4. **Email Notification Failures**:
   - **Mitigation**: Implement fallback mechanisms for alerts, such as logging failures to a monitoring system.

5. **Cost Overruns**:
   - **Mitigation**: Monitor costs associated with RU consumption and Azure services closely, setting budget alerts in Azure.
