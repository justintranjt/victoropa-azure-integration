# victorops-azure-integration
## Background. Why not use the VictorOps integration listed on their website?
Microsoft Azure and the incident management platform VictorOps do not have an automatically working integration at the moment. In fact, the integration they provide works with Microsoft OMS, a deprecated platform in the Azure ecosystem. This manual integration aims to include all the essential features that would include in a regular integration.

Note: This works for ALL forms of Azure alerts. You simply have to connect either alert to the action group with the logic app’s webhook (which will send the message to VictorOps).

For the time being, my fix is to use a Logic App to mold the payload into one VictorOps will accept from its [REST API](https://help.victorops.com/knowledge-base/victorops-restendpoint-integration/). Here is a summary of what we’re about to do:

Consider any alert created by the Azure Monitor. Oftentimes we would simply link this alert to an action group with a webhook that links directly to an incident management platform. For our workaround, we will instead route the webhook URL to a logic app which will handle the alert payload (because we also cannot construct a custom JSON with the current version of Azure monitor).

In a support case I opened with Azure in July 2018, a VictorOps support engineer told me that there are plans on the horizon that with deprecation/merging of OMS, custom JSON could be added to all Azure Monitor alert types/rules rendering this manual integration obsolete (but for the time being this is how we'll handle integration).

In our logic app, whenever an HTTP request is received at our logic app’s URI, a HTTP 200 response is triggered and sent back to the original requester. In this case, the requester is the alert rule’s action group firing an HTTP request. In the logic app, we create an HTTP POST request and send it to VictorOps’ generic REST API. This POST request contains a JSON body that is written with the Logic Apps Workflow Definition Language and contains the necessary JSON fields required by the VictorOps REST endpoint. More JSON fields can be added as necessary to make the incident descriptions more readable.

## Steps 
### Logic App (on Azure)
1. Create a Logic App. The Logic App will serve as the central structure for the integration with VictorOps. Follow these steps:
2. Create a new Logic App by clicking the Create Resource button in the top left corner of the Azure Portal. You can equivalently follow the first couple steps of this documentation.
    1. Name the application.
    2. Select an existing Resource Group.
    3. Create the logic app.
3. From the dashboard, select the Logic App you have just created.
4. From the Logic App blade, select Logic App Designer
    1. For the trigger condition, select "When an HTTP Request is received"
    2. Click New Step, select Add an Action
    3. Select Request Response and leave it as responding with a 200 status code (OK)
    4. Click New Step, select Add an Action
    5. Select HTTP - HTTP
        1. Method: POST
        2. URL:  This can be found in your VictorOps account under Settings > Alert Behavior > Integrations > Generic REST
        3. Headers: Content-Type | application/json
        
        4. Body:
```json
{
  "data": "@triggerBody()",
  "entity_display_name": "@{triggerBody()?['context']?['name']}",
  "entity_id": "@{triggerBody()?['context']?['id']}",
  "message_type": "@{if(equals(triggerBody()?['data']?['status'],'Activated'),'recovery','critical')}",
  "state_message": "@{triggerBody()?['context']?['description']}",
  "monitoring_tool":"Azure"
}
```
Note that this JSON Body will require some tweaking in the future to get the data we absolutely want in the incident. Once again, view the [Logic Apps Workflow Definition Language](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-workflow-definition-language) article for more information. Most of the incident information sent to VictorOps is found in the data field.
    
    5. Save. Back in the Logic App Designer and under the "When an HTTP Request is Received", the URL has now been generated. Copy this URL to the clipboard.
    
### Alerts (on VictorOps)
1. From the left menu pane, select Monitoring >> Alerts >> New Alert Rule
2. Define the alert however you would prefer to monitor things. For testing purposes, I find it easiest to monitor all administrative operations a condition (see Step 3)
Whenever the "Administrative Activity Log All Administrative operations" has "any" level, with "any" status and event is initiated by "<admin_email_addr>"
3. Define the alert details with any name and description 
4. For the last step, select a New Action Group, this action group will fire a webhook towards
    1. For the action, select webhook
    2. For the URL of the webhook, paste the URL copied earlier from the Logic App
5. Save
