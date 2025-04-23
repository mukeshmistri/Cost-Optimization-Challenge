# Cost-Optimization-Challenge
Cost Optimization Challenge: Managing Billing Records in Azure Serverless Architecture

### Detailed Analysis for Cost Optimization in Azure Serverless Architecture

This analysis provides a comprehensive overview comparing the existing setup with the proposed setup for managing billing records within an Azure serverless architecture. It includes a cost overview, an illustrative architecture diagram, pseudocode for critical processes, and a project timeline.

---

### 1. Cost Overview

#### Current Setup Costs

Assuming the current setup uses Azure Cosmos DB for all billing records (2 million records at 300 KB each):

- **Storage Costs**:
  - Total Data Size = 2,000,000 records * 300 KB = 600,000,000 KB = ~600 TB.
  - Azure Cosmos DB Storage Cost (approx. £0.20/GB/month) = £120,000/month.

- **Request Costs**:
  - Assume 100,000 read requests and 20,000 write requests per day:
  - Azure Cosmos DB Request Charge (approx. £0.008 per 1000 reads and £0.03 per 1000 writes).
  - Read Cost = (100,000 requests/day * 30 days / 1000) * £0.008 = £24/month.
  - Write Cost = (20,000 requests/day * 30 days / 1000) * £0.03 = £18/month.
  - **Total Operational Costs**: £120,000 + £24 + £18 = **£120,042/month**.


#### Proposed Setup Costs

In the proposed setup, records older than 3 months will be archived to Azure Blob Storage.

- **Storage Costs**:
  - **Active Records (0-3 Months)**: Assume 20% of records are less than three months old.
    - Active Data Size = 20% of 600 TB = 120 TB.
    - Storage Cost for Active Records = 120 TB * £0.20/GB = £24,000/month.
  
  - **Archived Records (Older than 3 Months)**:
    - Archived Data Size = 80% of 600 TB = 480 TB.
    - Blob Storage Cost (approx. £0.0184/GB/month) = 480 TB * £0.0184/GB = £8,832/month.

- **Request Costs** (Assuming a similar request pattern):
  - Requests for active records from Cosmos DB remain similar: £24/month (100,000 reads, 20,000 writes).
  - Requests for archived data will be reduced due to infrequent access.
  - Estimate archived read requests to be 10% of active ones = 10,000/month.
  - Blob Storage Read Cost (approx. £0.005 per 1000 requests):
  - Archived Read Cost = (10,000 requests/month / 1000) * £0.005 = £0.05/month.

- **Additional Costs**:
  - **Azure Application Insights**: £100/month for monitoring and diagnostics to ensure system reliability and performance.
  - **Azure Functions Premium Plan**: £200/month to reduce latency and ensure consistent performance.
  - **Azure Logic Apps**: £50/month to automate workflows for data processing and integration.
  - **Azure Storage Account (Standard)**: £20/month for logs and temporary data storage management.
  - **Azure Key Vault**: £10/month for securely managing secrets and sensitive information.

- **Total Proposed Operational Costs**:
  - Active Records Cost: £24,000
  - Archived Records Cost: £8,832
  - Request Costs: £24 (active) + £0.05 (archived) = £24.05
  - Additional Costs: £100 (App Insights) + £200 (Premium Plan) + £50 (Logic Apps) + £20 (Storage Account) + £10 (Key Vault) = £400
  - **Total Proposed Costs**: £24,000 + £8,832 + £24.05 + £400 = **£33,280.10/month**.

#### **Cost Summary**:

| Cost Category                    | Current Setup (Monthly) | Proposed Setup (Monthly) |
|----------------------------------|-------------------------|--------------------------|
| Storage Costs                    | £120,000                | £32,856.05               |
| Request Costs                    | £42                      | £24.05                   |
| Additional Component Costs        |                         | £400                     |
| **Total Costs**                  | **£120,042**            | **£33,280.10**           |


### 2. Illustrative Architecture Diagram

```plaintext
+--------------------------------------------------------------+
|                          API Layer                           |
|                 (Azure API Management)                       |
|                       +-------------+                       |
|                       |  Frontend   |                       |
|                       | (Web/App)   |                       |
|                       +-------------+                       |
+--------------------------------------------------------------+
                           |               |                  
                           |               |                  
                           |               |                  
                           |               |                  
                           |               |                  
               +-----------+               +------------+      
               |                                               |  
               |                                               |  
               |                                               |  
        +------+-------+                            +---------+---------+  
        |  Azure       |                            |  Azure Queue      |
        |  Functions   |                            |  Storage          |
        |  (Data       |                            |  (Request Buffer) |
        |  Processing) |                            +-------------------+
        +------+-------+                                            |
               |                                                    |
               |                                                    |
               |                                                    |
        +------+-------+                                 +----------+---------+     
        |  Azure       |                                 |  Azure Durable     |
        |  Cosmos DB   |                                 |  Functions         |
        |  (Active     |                                 |  (Workflow         |
        |  Records)    |                                 |  Management)       |
        +--------------+                                 +-------------------+
                                                           |
                                                           |
                                                           |
                                                     +-----+-----+
                                                     |  Azure Blob  |
                                                     |  Storage     |
                                                     | (Archived    |
                                                     |  Records)    |
                                                     +--------------+
```

### Key Components Explained

1. **API Layer (Azure API Management)**:
   - Acts as a façade for the frontend applications, ensuring that all API contracts remain intact and providing a single point of access for different clients.

2. **Frontend (Web/App)**:
   - The user interface where users interact with the billing records. It communicates with the API layer.

3. **Azure Functions (Data Processing)**:
   - Serverless functions handle incoming requests to read and write billing records. These functions can also trigger the archival process.

4. **Azure Queue Storage (Request Buffer)**:
   - Buffers requests and ensures that they are processed asynchronously. This helps to manage spikes in load without losing requests.

5. **Azure Durable Functions (Workflow Management)**:
   - Orchestrates complex workflows associated with processing billing records, such as managing the lifecycle of data archival.

6. **Azure Cosmos DB (Active Records)**:
   - Stores active records (those less than three months old) for quick access and high throughput.

7. **Azure Blob Storage (Archived Records)**:
   - Stores older billing records (those older than three months) at a lower cost while remaining accessible for occasional retrieval.


### 3. Pseudocode for Data Archival and Retrieval Logic

#### **Azure Function for Archiving Data**

```python
import azure.cosmos.cosmos_client as cosmos_client
import azure.storage.blob as blob_service_client
import datetime
import json

# Setup clients
cosmos_client = cosmos_client.CosmosClient(ENDPOINT, KEY)
database = cosmos_client.get_database_client(DATABASE_NAME)
container = database.get_container_client(CONTAINER_NAME)

blob_service = blob_service_client.BlobServiceClient.from_connection_string(BLOB_CONNECTION_STRING)
blob_container = blob_service.get_container_client(BLOB_CONTAINER_NAME)

def archive_old_records():
    current_date = datetime.datetime.now()
    query = "SELECT * FROM c WHERE c.timestamp < @three_months_ago"
    parameters = [{"name": "@three_months_ago", "value": (current_date - datetime.timedelta(days=90)).isoformat()}]
    
    old_records = list(container.query_items(query=query, parameters=parameters, enable_cross_partition_queries=True))

    for record in old_records:
        blob_name = f"{record['id']}.json"
        blob_container.upload_blob(blob_name, json.dumps(record))
        container.delete_item(item=record['id'], partition_key=record['id'])

# Trigger this function on a schedule
```

#### **API Layer Logic for Data Retrieval**

```python
def get_billing_record(record_id):
    try:
        record = container.read_item(item=record_id, partition_key=record_id)
        return record
    except Exception as e:
        # If not found in Cosmos DB, check Blob Storage
        blob_name = f"{record_id}.json"
        blob_client = blob_container.get_blob_client(blob_name)
        blob_data = blob_client.download_blob().readall()
        return json.loads(blob_data)
```

### 4. Project Timeline

| Phase                              | Duration       | Start Date    | End Date      |
|------------------------------------|----------------|----------------|----------------|
| **Requirements Gathering**          | 2 weeks        | Week 1        | Week 2        |
| **Design Architecture**             | 2 weeks        | Week 3        | Week 4        |
| **Implementation**                  | 4 weeks        | Week 5        | Week 8        |
| **Testing and Validation**          | 2 weeks        | Week 9        | Week 10       |
| **Deployment**                      | 1 week         | Week 11       | Week 11       |
| **Continuous Monitoring & Review**  | Ongoing        | Week 12       | Ongoing        |

### Conclusion

This detailed analysis outlines the financial benefits of transitioning from the existing Azure Cosmos DB-centric architecture to a cost-optimised serverless model that leverages Azure Blob Storage for archival. The outlined architecture diagram, pseudocode, and project timeline provide a clear roadmap for implementation, ensuring that the critical aspects of data archival and retrieval are addressed efficiently while preserving API contracts. The proposed solution not only significantly reduces operational costs but also enhances the overall manageability and scalability of the billing records system.
