
# Automating access requests to Azure Databricks Unity Catalog with Microsoft Purview

![Automating access requests to Azure Databricks Unity Catalog with Microsoft Purview](/images/Architecture_Purview-UC-Workflow.png)

## A step-by-step guide to enabling open data sharing of Databricks Unity Catalog assets with Microsoft Purview 

Consider a scenario where an analyst using Power BI needs to securely access and query data assets in a Databricks workspace—without having direct access to the workspace itself. Non-Databricks users can take advantage of[Delta Sharing](https://learn.microsoft.com/en-ca/azure/databricks/delta-sharing/)in Databricks, which enables secure data access to data and AI assets in Databricks for users inside or outside your organization through the Databricks[open sharing protocol](https://learn.microsoft.com/en-ca/azure/databricks/delta-sharing/share-data-open). Additionally, since these users would not have access to Databricks Catalog Explorer—hindering their ability to find any Unity Catalog-managed data—Microsoft Purview can be used to allow users from across your organization to discover and request access securely to this data. By leveraging Microsoft Purview’s integration with Unity Catalog, Purview users can discover and access Unity Catalog-managed data in a governed and secure manner.

The following step-by-step guidance explains how to enable external users to explore and request access to data assets in a Databricks workspace—without having direct access to that workspace—by simply using the Microsoft Purview UI. By leveraging Purview’s[Azure Databricks Unity Catalog connector](https://learn.microsoft.com/en-us/purview/register-scan-azure-databricks-unity-catalog?tabs=MI), the metadata for the assets in Unity Catalog is made discoverable in the Purview data map. The ability to request and grant access to Unity Catalog objects in an automated fashion is made possible through Purview’s self-service access workflows, in conjunction with calling the Databricks REST API, as shown below.

![Automation Workflow](/images/Automation_Purview-UC-Workflow.png)

The following steps will automate access requests to Databricks Unity Catalog using Purview workflows and the Databricks REST API resulting in a seamless experience for the end user.

### Step 1: Setup prerequisites

- [Setup service principal for Azure Databricks authentication and add it as a credential in Purview](https://learn.microsoft.com/en-us/purview/register-scan-azure-databricks-unity-catalog?tabs=SP#tabpanel_1_SP)
- [Setup connection to Azure Databricks Unity Catalog from Microsoft Purview](https://learn.microsoft.com/en-us/purview/register-scan-azure-databricks-unity-catalog?tabs=MI)
- [Scan Azure Databricks to automatically identify assets in Microsoft Purview](https://learn.microsoft.com/en-us/purview/register-scan-azure-databricks-unity-catalog?tabs=MI#scan)
- Confirm availability of assets (e.g., Databricks tables) by[browsing and searching in the Microsoft Purview Unified Catalog](https://learn.microsoft.com/en-us/purview/register-scan-azure-databricks-unity-catalog?tabs=MI#browse-and-search-assets)

### Step 2: Create Databricks notebook

In your Azure Databricks workspace, create a new Databricks notebook with the following cells to programmatically call the Databricks REST API to create and share a Delta share for the requested data asset with the intended recipient:

#### Create widgets and import libraries

```python
# Create widgets to retrieve parameters from Purview workflow
dbutils.widgets.text("assetFQName", "")
dbutils.widgets.text("requestorID", "")

# Import libraries
import requests
import json 
```

#### Setup authentication to request token

```python
# Function to request token using service principal with Databricks scope secret
def request_token():
    client_id = dbutils.secrets.get(scope="azure_auth", key="client_id")
    client_secret = dbutils.secrets.get(scope="azure_auth", key="client_secret")
    tenant_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    scope = "2ff814a6-3304-4ab8-85cb-cd0e6f879c1d/.default" #OAuth scope used specifically for Azure Databricks
    
    url = f"https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token"
    headers = {
        'Content-Type': 'application/x-www-form-urlencoded'
    }
    data = {
        'grant_type': 'client_credentials',
        'client_id': client_id,
        'client_secret': client_secret,
        'scope': scope
    }
    response = requests.post(url, headers=headers, data=data)
    response.raise_for_status()
    return response.json()['access_token']
```

#### Extract information from Purview parameters

```python
 # Function to extract catalog.schema.table from a fully qualified asset path in Purview
def derive_asset_name(asset_fqname):
    parts = asset_fqname.split('/')
    catalog = parts[4]
    schema = parts[6]
    table = parts[8]
    return f"{catalog}.{schema}.{table}"
```

#### Create Delta share

```python
# Function to create a Delta share
def create_share(token, share_name, workspace_url):
    url = f"{workspace_url}/api/2.1/unity-catalog/shares"
    headers = {
        'Authorization': f'Bearer {token}',
        'Content-Type': 'application/json'
    }
    data = {
        'name': share_name
    }
    response = requests.post(url, headers=headers, json=data)
    if response.status_code == 200:
        return response.json()
    elif response.status_code == 400 and response.json().get('error_code') == 'SHARE_ALREADY_EXISTS':
        return None
    else:
        response.raise_for_status()
```

#### Add data object to share

```python
# Function to add data object to the share
def add_data_object_to_share(token, share_name, asset_name, workspace_url):
    url = f"{workspace_url}/api/2.1/unity-catalog/shares/{share_name}"
    headers = {
        'Authorization': f'Bearer {token}',
        'Content-Type': 'application/json'
    }
    data = {
        'name': share_name,
        'updates': [
            {
                'action': 'ADD',
                'data_object': {
                    'data_object_type': 'TABLE',
                    'history_data_sharing_status': 'ENABLED',
                    'name': asset_name,
                    'status': 'ACTIVE'
                }
            }
        ]
    }
    response = requests.patch(url, headers=headers, json=data)
    if response.status_code == 200:
        return response.json()
    elif response.status_code == 400 and response.json().get('error_code') == 'RESOURCE_ALREADY_EXISTS':
        return None
    else:
        response.raise_for_status()
```

#### Create share recipient

```python
# Function to create a recipient
def create_recipient(token, requestor_id, workspace_url):
    url = f"{workspace_url}/api/2.1/unity-catalog/recipients"
    headers = {
        'Authorization': f'Bearer {token}',
        'Content-Type': 'application/json'
    }
    data = {
        'authentication_type': 'TOKEN',
        'name': requestor_id
    }
    response = requests.post(url, headers=headers, json=data)
    if response.status_code == 200:
        return response.json()
    elif response.status_code == 400 and response.json().get('error_code') == 'RECIPIENT_ALREADY_EXISTS':
        return None
    else:
        response.raise_for_status()
```

#### Add recipient to share

```python
# Function to add recipient to the share with select permissions
def add_recipient_to_share(token, requestor_id, share_name, workspace_url):
    url = f"{workspace_url}/api/2.1/unity-catalog/shares/{share_name}/permissions"
    headers = {
        'Authorization': f'Bearer {token}',
        'Content-Type': 'application/json'
    }
    data = {
        'changes': [
            {
                'principal': requestor_id,
                'remove': [],
                'add': ['SELECT']
            }
        ]
    }
    response = requests.patch(url, headers=headers, json=data)
    response.raise_for_status()
    return response.json()
```

#### Execute API calls to the Databricks REST API

```python
# Main function to execute all API calls
def main():
    # Retrieve the values of the parameters
    asset_fqname = dbutils.widgets.get("assetFQName")
    requestor_id = dbutils.widgets.get("requestorID")
    
    workspace_url = 'https://adb-xxxxxxxxxxxxxxx.xx.azuredatabricks.net'
    asset_name = derive_asset_name(asset_fqname)
    share_name = asset_name.replace('.', '_')
    try:
        token = request_token()
        create_share(token, share_name, workspace_url)
        add_data_object_to_share(token, share_name, asset_name, workspace_url)
        create_recipient(token, requestor_id, workspace_url)
        add_recipient_to_share(token, requestor_id, share_name, workspace_url)
        print("All API calls executed successfully.")
    except requests.exceptions.RequestException as e:
        print(f"An error occurred: {e}")

if __name__ == "__main__":
    main()
```

### Step 3: Configure Databricks workflow

Create a new Databricks workflow as shown below with the notebook created above as a task.

![Configure Databricsk workflow](/images/Screenshots/0.%20Create%20Databricks%20Workflow.png)

> [!Note]
> Make sure to note down the "Job ID" as it will be required later.

### Step 4: Configure Purview workflow

1 - Create a new workflow in Microsoft Purview by going to “Solutions” and selecting “Workflows”.

![Configure Purview workflow](/images/Screenshots/1.%20Create%20Purview%20Workflow.png)

2 - When adding the new workflow, select “Governance” as theworkflow type, select “Data access request” as thetemplate, and provide aname, e.g., “Databricks access request”. A generic template of the workflow is generated for data access requests, we will need to modify this to suit our scenario.Deletethe action containing the “condition” action to clean the workflow slate as shown below.

![Clean the workflow slate](/images/Screenshots/2.%20Clean%20the%20workflow%20slate.png)

3 - Let’s start by modifying the “start and wait for an approval” step to include a more descriptive title for the email that is sent out, assign the approval to the asset owner and set any reminders.

![Modify approval step](/images/Screenshots/3.%20Modify%20approval%20step.png)

4 - Create a new step/action and choose “condition”. Select “Add dynamic content” and select “Approval.Outcome”.

![Create new condition and dynamic content](/images/Screenshots/4.%20Create%20new%20condition%20and%20add%20dynamic%20content.png)

5 - Enter “Approved” for the value as shown below.

![Set condition to approved](/images/Screenshots/5.%20Set%20condition%20to%20approved.png)

6 - Create “send email notification” action for the “If no” condition (on the right) by modifying the title, subject, message body and recipient (add dynamic content then select “Workflow.Requestor) as follows:

![Request rejected workflow](/images/Screenshots/6.%20Request%20rejected%20workflow.png)

7 - Create the following actions in the “If yes” condition (on the left) in this order:

(i) HTTP: Trigger Databricks Workflow

![Trigger Databricks workflow](/images/Screenshots/7.%20Trigger%20Databricks%20Workflow.png)

- Host: Enter the URL for the Databricks workspace attached to the Unity Catalog metastore that contains the asset associated with this request.
- Method: `POST`
- Path: `/api/2.2/jobs/run-now`
- Body:
    ```json
    {"job_id": <databricks-job-id>,"queue": { "enabled": true },"notebook_params":{"assetFQName":"⁠Asset.Fully Qualified Name⁠","requestorID":"⁠Workflow.Requestor⁠"}}Note: Use “Add dynamic content” to pass the asset’s fully qualified name (Asset.Fully Qualified Name⁠) and requestor’s object id  (⁠Workflow.Requestor⁠) as parameters to the Databricks workflow.
    ```
- Authentication: Service Principal
- Credential: `<service-principal>`
- Audience: `2ff814a6-3304-4ab8-85cb-cd0e6f879c1d/.default`
 
(ii) Delay: Set a 2-minute delay to allow for the Databricks workflow to complete

![Create delay](/images/Screenshots/8.%20Create%20delay.png)

(iii) HTTP: Get Activation URL

![Get activation URL](/images/Screenshots/9.%20Get%20Activation%20URL.png)

- Host: Enter the URL for the Databricks workspace attached to the Unity Catalog metastore that contains the asset associated with this request.
- Method: `GET`
- Path: `/api/2.1/unity-catalog/recipients/<workflow-requestor-id>`

> [!Note]
> Insert workflow requestor using the “Add dynamic content” option to use the requestor’s object id dynamically in the API call.

(iv) Parse JSON: Extract URL

![Extract URL](/images/Screenshots/10.%20Extract%20URL.png)

- Content: Use the “Add dynamic content” option to insert the “Http.Body” variable from the “Get Requestor Email” HTTP action created previously. 
    ```json
    Schema:{"type": "object","properties": {"created_by": {"type": "string"},"name": {"type": "string"},"updated_by": {"type": "string"},"securable_kind": {"type": "string"},"authentication_type": {"type": "string"},"id": {"type": "string"},"properties_kvpairs": {"type": "object","properties": {"properties": {"type": "object","properties": {"databricks.name": {"type": "string"}}}}},"owner": {"type": "string"},"updated_at": {"type": "integer"},"full_name": {"type": "string"},"securable_type": {"type": "string"},"created_at": {"type": "integer"},"tokens": {"type": "array","items": {"type": "object","properties": {"updated_by": {"type": "string"},"id": {"type": "string"},"created_at": {"type": "integer"},"updated_at": {"type": "integer"},"activation_url": {"type": "string"},"created_by": {"type": "string"},"expiration_time": {"type": "integer"}}}}}}
    ```

(v)Send email notification:

![Send approval email with URL](/images/Screenshots/11.%20Send%20approval%20email%20with%20URL.png)

- Subject: Write “Access Request – APPROVED” or something similar.
- Message body: Write the message you would like to send and use “Add dynamic content” to dynamically pass the access URL for the Delta Share to the requestor by inserting the following expression:`first(outputs('Extract URL')['tokens'])['activation_url']`

8 - Apply workflow to the appropriate scope and save it.

![Apply and save](/images/Screenshots/12.%20Apply%20and%20Save.png)

## Step 5: Verify the user experience

A user browsing delta tables in the Microsoft Purview Unified Catalog may request access directly to the table as shown below, triggering the workflow configured above.

![Request access to asset](/images/Screenshots/13.%20Request%20access%20to%20asset%20UI.png)

Once the request is submitted, this triggers the Purview workflow created above. The first action, as defined in the workflow associated with this data asset, is to send an email notification to the listed “Owner” of the asset (go to Edit > Contacts to check) to approve this access request.

![Approval email](/images/Screenshots/14.%20Approval%20email.png)

Once the request is approved, the Purview workflow triggers the Databricks workflow to make the required Databricks REST API calls for creating and sharing the data asset through Open Delta Sharing. Shortly after the Delta share is created and recipient is added in Azure Databricks, the Purview workflow retrieves the activation URL to access the share and sends it to the requestor.

![Access URL](/images/Screenshots/15.%20Access%20URL.png)

After downloading the shared credential file from the activation URL, the requestor can[read the shared data on Power BI by connecting to Azure Databricks using the Delta Sharing connector](https://learn.microsoft.com/en-ca/azure/databricks/delta-sharing/read-data-open#power-bi).
