
# Purview-driven Unity Catalog Access Automation

![Automating access requests to Azure Databricks Unity Catalog with Microsoft Purview](/images/Architecture_Purview-UC-Workflow.png)

## A step-by-step guide to enabling open data sharing of Databricks Unity Catalog assets with Microsoft Purview

Consider a scenario where an analyst using Power BI needs to securely access and query data assets in a Databricks workspace—without having direct access to the workspace itself. Non-Databricks users can take advantage of [Delta Sharing](https://learn.microsoft.com/en-ca/azure/databricks/delta-sharing/) in Databricks, which enables secure data access to data and AI assets in Databricks for users inside or outside your organization through the Databricks [open sharing protocol](https://learn.microsoft.com/en-ca/azure/databricks/delta-sharing/share-data-open). Additionally, since these users would not have access to Databricks Catalog Explorer—hindering their ability to find any Unity Catalog-managed data—Microsoft Purview can be used to allow users from across your organization to discover and request access securely to this data. By leveraging Microsoft Purview’s integration with Unity Catalog, Purview users can discover and access Unity Catalog-managed data in a governed and secure manner.

The following step-by-step guidance explains how to enable external users to explore and request access to data assets in a Databricks workspace—without having direct access to that workspace—by simply using the Microsoft Purview UI. By leveraging Purview’s [Azure Databricks Unity Catalog connector](https://learn.microsoft.com/en-us/purview/register-scan-azure-databricks-unity-catalog?tabs=MI), the metadata for the assets in Unity Catalog is made discoverable in the Purview data map. The ability to request and grant access to Unity Catalog objects in an automated fashion is made possible through Purview’s self-service access workflows, in conjunction with calling the Databricks REST API, as shown below.

### Automating access requests to Azure Databricks Unity Catalog with Microsoft Purview using:

1. [Purview Workflows (Classic Data Catalog)](/automate-using-workflows.md)
2. [Power Automate Connector](/) (Coming soon!)