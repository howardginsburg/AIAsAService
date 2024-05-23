# Azure OpenAI Usage Tracking

This repo serves as a reference architecture for tracking usage of OpenAI models on Azure. Many organizations want to understand OpenAI metrics, including what models are being used, by whom, and how often.  They also want to track tokens being consumed, and the prompts being passed in.  This leads to the ability to create chargeback models for consuming applications and users and enables analysis to be done on prompt usage and best practices.  Azure API Management recently announced a new [policy](https://learn.microsoft.com/azure/api-management/azure-openai-emit-token-metric-policy) to send token information to App Insights.  This is a great feature, but doesn't enable long term usage analysis.  This architecture provides a way to track all of the data needed to understand OpenAI usage in a scalable and cost-effective manner.

## Architecture Overview

Azure API Management serves as the cornerstone for this architecture as it enables different consumer access to the same OpenAI endpoint through the use of subscriptions or jwt tokens.  APIM policy also allow the [logging](https://learn.microsoft.com/azure/api-management/api-management-howto-log-event-hubs) of request/response data to Event Hubs so that it can be processed outside of the request/response path.  The data generated is suitable for analytics queries, so rather than land it in a traditional database, Microsoft Fabric becomes a cost-effective and scalable solution for storing the data.  Power BI can then be used to create reports on the data in the Lakehouse. 

The architecture consists of the following components:

1. **Azure OpenAI**: These are the models that are exposed as APIs using Azure API Management. 
1. **Azure API Management**: This is used to expose the OpenAI models as APIs and track usage data.
1. **Event Hubs**: This is used to ingest usage data from Azure API Management.
1. **Microsoft Fabric**: This is used to process and store the usage data in a scalable and cost-effective manner.

![Architecture Diagram](/images/architecture.png)

Flow:
1. A client makes a request to an OpenAI model through Azure API Management using a subscription key.
1. Azure API Management forwards the request to the OpenAI model.
1. Azure API Management logs the subscription id and request/response data to Event Hubs using an EventLogger policy.
1. An Eventstream processor in Microsoft Fabric reads the data from Event Hubs.
1. The output of the stream is writen to a delta table in a Lakehouse.
1. The data in the delta table is then queried via a Power BI report or a Notebook.

Note, you can easily swap out the subscription key for tracking an individual user by using JWT tokens and associating the user with the token.  This would allow you to track usage at the user level.

## Setup

This tutorial assumes you have familiarity with the technologies used in this architecture and have deployed instances of each. If you are new to any of the technologies, please refer to the documentation provided by Microsoft.

### Azure OpenAI
1. Create a model deployment in Azure OpenAI and note the endpoint.
1. Enable the System Assigned Identity on your Azure API Management instance and grant it 'Cognitive Services OpenAI User' role on your OpenAI instance.  This will allow APIM to call the OpenAI endpoint without needing to rely on the subscription key for OpenAI.

### Azure Event Hubs
1. Create an Event Hub called 'openai-usage' within your Event Hub instance.
1. Grant the APIM managed identity the 'Azure Event Hubs Data Sender' role on the 'openai-usage' Event Hub so that it can write to the Event Hub without needing a connection string.

### Azure API Management
1. To create the EventLogger for APIM that is using the managed identity, we have to use the Rest API to create it.  The easiest way to do this is the [Try It](https://learn.microsoft.com/en-us/rest/api/apimanagement/logger/create-or-update?view=rest-apimanagement-2022-08-01&tabs=HTTP#code-try-0) feature in the docs.  You will need to provide the following information:
    - loggerId: openai-usage
    - resourceGroupName: your_resource_group
    - serviceName: your_apim_instance_name
    - subscriptionId: your_subscription_id
    - body:
        ```json
        {
            "properties": {
                "loggerType": "azureEventHub",
                "description": "eventhub logger for openai usage",
                "credentials": {
                    "endpointAddress":"your_eventhub_namespace.servicebus.windows.net",
                    "identityClientId":"SystemAssigned",
                    "name":"openai-usage"
                }
            }
        }
        ```
1. [Import](https://learn.microsoft.com/azure/api-management/azure-openai-api-from-specification) the OpenAI inference specification into Azure API Management.
1. Update the API settings to rename the Subscription Header Name from 'Ocp-Apim-Subscription-Key' to 'api-key'.  The OpenAI API expects the subscription key to be passed in as 'api-key'.  Developers will set this value to their APIM subscription key.
1. Add the following policy to the 'All Operations' section of your OpenAI API in APIM.  This will allow APIM to call Open AI using its managed identity, and then log the request/response data to Event Hubs.
```xml
<policies>
    <inbound>
        <base />
        <authentication-managed-identity resource="https://cognitiveservices.azure.com" output-token-variable-name="msi-access-token" ignore-error="false" />
        <set-header name="Authorization" exists-action="override">
            <value>@("Bearer " + (string)context.Variables["msi-access-token"])</value>
        </set-header>
        <set-variable name="requestBody" value="@(context.Request.Body.As<string>(preserveContent: true))" />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
        <choose>
            <when condition="@(context.Response.StatusCode == 200)">
                <log-to-eventhub logger-id="openai-usage">@{
                    var responseBody = context.Response.Body?.As<string>(true);
                    var requestBody = (string)context.Variables["requestBody"];             
                    return new JObject(
                        new JProperty("EventTime", DateTime.UtcNow),
                        new JProperty("AppSubscriptionKey", context.Request.Headers.GetValueOrDefault("api-key",string.Empty)),                     
                        new JProperty("Request", requestBody),
                        new JProperty("Response",responseBody )  
                    ).ToString();
                }</log-to-eventhub>
            </when>
        </choose>
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```
1. Test that your APIM instance can call the OpenAI endpoint by using the 'Test' tab in the APIM portal.

### Microsoft Fabric

At the time of this writing, connecting to an Event Hub from Fabric must be done using a Shared Access Key, so we cannot use a managed identity to connect to it.  This means the connection string will be stored in the Event Hub stream configuration.

#### Ingestion

1. Create a new Workspace.
1. Create a Lakehouse to store the data.
1. In your Event Hub instance, create a create a Shared access policy for the openai-usage Event Hub that has 'Listen' permissions.  Copy the Primary Key.
1. Create a new Event stream in the workspace
    - The source will be the 'openai-usage' Event Hub.
    - The destination will be a new managed Delta table in the Lakehouse called 'OpenAIData'.
    ![Eventstream](/images/eventstream.png)
1. Invoke your OpenAI APIM endpoint several times to send some test data in.  You should see the Delta table created and data in it.
    ![Delta Table](/images/deltatable.png)

#### Reporting

1. Switch to the SQL Analytics endpoint.
1. Run the following query to create a view that makes it easier to see the token usage by subscription key.
```sql
CREATE OR ALTER VIEW [dbo].[OpenAIUsageView] AS
SELECT CAST(EventTime AS DateTime2) AS [EventTime],
[AppSubscriptionKey],
JSON_VALUE([Response], '$.object') AS [Operation],
JSON_VALUE([Response], '$.model') AS [Model],
[Request], 
[Response],
CAST(JSON_VALUE([Response], '$.usage.completion_tokens') AS INT) AS [CompletionTokens],
CAST(JSON_VALUE([Response], '$.usage.prompt_tokens') AS INT) AS [PromptTokens],
CAST(JSON_VALUE([Response], '$.usage.total_tokens') AS INT) AS [TotalTokens]
FROM 
[YOUR_LAKEHOUSE_NAME].[dbo].[OpenAIData]
```
![SQL View](/images/sqlview.png)
1. Refresh the Views to see the new one created.
1. Click on the Reporting tab and select 'Automatically update semantic model'
1. Create a new report using the OpenAIUsageView as the data source.
![Report](/images/report.png)

#### Data Science

1. Create a new Notebook.
1. Load the managed delta table into a dataframe.
```pyspark
df = spark.sql("SELECT * FROM YOUR_LAKEHOUSE_NAME.OpenAIData LIMIT 1000")
display(df)
```
![Notebook](/images/notebook.png)

