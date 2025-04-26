### Step-by-Step Implementation Guide

#### Step 1: Set Up Azure Cosmos DB

1. **Sign in to Azure Portal**:
   - Go to the [Azure Portal](https://portal.azure.com/).

2. **Create a New Azure Cosmos DB Account**:
   - Click on **"Create a resource"**.
   - Search for **"Azure Cosmos DB"** and select it.
   - Click on **"Create"**.
   - **Configure the Cosmos DB Account**:
     - **Subscription**: Choose your Azure subscription.
     - **Resource Group**: Create a new resource group (e.g., `BillingRecordsRG`).
     - **Account Name**: Provide a unique name for your Cosmos DB account (e.g., `billingrecordsdb`).
     - **API**: Choose **SQL** for the SQL API.
     - **Location**: Select the closest region for your users.
     - **Capacity Mode**: Select **Provisioned throughput** and set the initial **RU/s** (e.g., 5 RU/s).

3. **Review and Create**:
   - Review the settings and click on **"Create"**. Wait for the deployment to complete.

#### Step 2: Set Up Azure Blob Storage

1. **Create Azure Storage Account**:
   - In the Azure Portal, click on **"Create a resource"**.
   - Search for **"Storage account"** and select it.
   - Click on **"Create"**.
   - **Configure the Storage Account**:
     - **Subscription**: Choose the same subscription.
     - **Resource Group**: Select the existing resource group (`BillingRecordsRG`).
     - **Storage Account Name**: Provide a unique name (e.g., `billingrecordsblob`).
     - **Region**: Same region as Cosmos DB.
     - **Performance**: Choose Standard.
     - **Replication**: Choose Locally redundant storage (LRS).
     - **Access Tier**: Select **Cool** (for archived data).

2. **Review and Create**:
   - Review the settings and click **"Create"**. Wait for the deployment to complete.

#### Step 3: Set Up Azure Functions

1. **Create Azure Functions**:
   - In the Azure Portal, click on **"Create a resource"**.
   - Search for **"Function App"** and select it.
   - Click on **"Create"**.
   - **Configure the Function App**:
     - **Subscription**: Choose the same subscription.
     - **Resource Group**: Select `BillingRecordsRG`.
     - **Function App Name**: Provide a unique name (e.g., `billingrecordsfns`).
     - **Publish**: Choose **Code**.
     - **Runtime Stack**: Choose your preferred stack (e.g., .NET, Node.js, Python).
     - **Region**: Same region as previous resources.
     - **Hosting Plan**: Choose **Premium** for better performance.
     - **Storage**: Create a new storage account or use the existing one from Step 2.

2. **Review and Create**:
   - Review the settings and click **"Create"**. Wait for the deployment to complete.

#### Step 4: Create a Function for Data Archival

1. **Navigate to the Function App**:
   - Go to your newly created Function App in the Azure Portal.

2. **Create a New Function**:
   - Click on **"Functions"** in the left menu.
   - Click on **"Add"** to create a new function.
   - Choose the **HTTP trigger** template for the function.

3. **Write the Archival Logic**:
   - Use the provided pseudocode for archiving old records:
     ```python
     import azure.cosmos.cosmos_client as cosmos_client
     import azure.storage.blob as blob_service_client
     import datetime
     import json

     # Setup Cosmos Client
     cosmos_client = cosmos_client.CosmosClient(ENDPOINT, KEY)
     database = cosmos_client.get_database_client(DATABASE_NAME)
     container = database.get_container_client(CONTAINER_NAME)

     # Setup Blob Service Client
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
     ```

4. **Save and Test the Function**:
   - Save the function and use the **Test/Run** feature to test the archival logic.

#### Step 5: Set Up Application Insights (Optional)

1. **Create Application Insights**:
   - In the Azure Portal, click on **"Create a resource"**.
   - Search for **"Application Insights"** and select it.
   - Click on **"Create"** and configure it for your Function App.
   - Link it to your Function App to monitor performance and diagnostics.

#### Step 6: Deploy the Frontend Application (Optional)

- If there is a frontend for users to interact with the billing records, deploy it using Azure App Service or another service that suits your application's needs.

#### Step 7: Configure Access and Security

1. **Manage Access Keys**:
   - Ensure your Azure Cosmos DB and Blob Storage accounts have access keys configured properly.
   - Use **Azure Key Vault** to securely manage secrets, like connection strings, and access them in your functions.

2. **Set Up Role-Based Access Control (RBAC)**:
   - Configure RBAC to ensure that only authorized users and services can access the resources you have created.
