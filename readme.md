
# A Guide to Enabling Open Data Sharing of Databricks Unity Catalog Assets with Microsoft Purview

Consider a scenario where a Power BI analyst needs to securely access and query data assets in a Databricks workspace. However, this analyst does not have direct access to the workspace. In such cases, users without direct access to Databricks like this analyst can take advantage of [Delta Sharing](https://learn.microsoft.com/en-ca/azure/databricks/delta-sharing/) in Databricks which offers a secure and efficient solution. It enables secure access to data and AI assets in Databricks for users inside or outside your organization through the Databricks [open sharing protocol](https://learn.microsoft.com/en-ca/azure/databricks/delta-sharing/share-data-open).

Since these users would not have access to Databricks Catalog Explorer—making it difficult to locate Unity Catalog-managed data—**Microsoft Purview** becomes essential. Purview allows users across your organization to securely discover and request access to this data. By leveraging its [integration with Unity Catalog](https://learn.microsoft.com/en-us/purview/register-scan-azure-databricks-unity-catalog?tabs=MI), Purview users can discover and access Unity Catalog-managed data in a governed and secure manner.

Once the data is discoverable, Purview enables automated access provisioning through its workflow capabilities. Microsoft Purview allows its users to explore and request access to data assets in a Databricks workspace. Because Unity Catalog metadata is discoverable in the Purview data map, users can request and receive access to Unity Catalog objects in an automated fashion. Purview’s self-service workflows enable this automation. These workflows can be configured to automatically trigger access provisioning by calling the Databricks REST API as shown in the diagram below. This automation streamlines the approval process and ensures secure, auditable access to data.

![Automating access requests to Azure Databricks Unity Catalog with Microsoft Purview](/images/Architecture_Purview-UC-Workflow.png)

To enable open data sharing and automate access to Unity Catalog assets, follow the step-by-step guidance below:

1. Use [Purview Workflows (Classic Data Catalog)](/automate-using-workflows.md) to manage access to Unity Catalog assets
2. Use the [Power Automate Connector](/) in Purview (Coming soon!)