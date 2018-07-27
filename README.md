# victorops-azure-integration
For a tutorial with a visual guide, please view the [full write-up.](https://justintranjt.github.io/projects/2018-07-27-victorops-azure-manual-integration/)

## Steps 
### Logic App (on Azure Logic Apps)
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
5. Save. Back in the Logic App Designer and under the "When an HTTP Request is Received", the URL has now been generated. Copy this URL to the clipboard.
    
Note that the JSON Body will require some tweaking in the future to get the data we absolutely want in the incident. Once again, view the [Logic Apps Workflow Definition Language](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-workflow-definition-language) article for more information. Most of the incident information sent to VictorOps is found in the data field.
    
### Alerts (on Azure Monitor)
1. From the left menu pane, select Monitoring >> Alerts >> New Alert Rule
2. Define the alert however you would prefer to monitor things. For testing purposes, I find it easiest to monitor all administrative operations a condition (see Step 3)
Whenever the "Administrative Activity Log All Administrative operations" has "any" level, with "any" status and event is initiated by "<admin_email_addr>"
3. Define the alert details with any name and description 
4. For the last step, select a New Action Group, this action group will fire a webhook towards
    1. For the action, select webhook
    2. For the URL of the webhook, paste the URL copied earlier from the Logic App
5. Save
