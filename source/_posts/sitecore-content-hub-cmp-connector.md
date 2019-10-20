title: Sitecore connector for Content Hub CMP
date: 2019/10/20
categories:
- Sitecore
- Content Hub
tags:
- stylelabs
- connector
- cmp
- azure
- servicebus

---
<img class="hero-img" src="/images/cmp-connector.jpg" alt="Sitecore Content Hub CMP Connector">
---
How to install Sitecore's Connect Hub CMP connector "Sitecore Connect™ for Sitecore CMP 1.0.0"
<!-- more -->

*Versions used: Sitecore Experience Platform 9.2 Initial Release. Content Hub 3.2*

[Sitecore Connect™ for Sitecore CMP](https://dev.sitecore.net/Downloads/Sitecore_Connect_for_Sitecore_CMP/10/Sitecore_Connect_for_Sitecore_CMP_100.aspx) pulls content from Content Hub and stores them in Sitecore buckets. The connector requires an azure service bus that supports topics. See below.

What gets synced and when is configured in Content Hub via triggers. Content Hub sends configured content to an Azure Topic. On Sitecore side, the connector is listening to this topic and creates/updates items when messages are added. It then creates a message on another topic with information about what was done.

**For the connector to work, you need to configure the following 3 places::**
* Azure Services Bus
* Content Hub
* Sitecore Content Management

## Azure Services Bus ##
As mentioned before, you need a subscription that allows to select either Standard or Premium Pricing tier. This is because the connector uses Topics which are not avaiable on Basic tier.
<img class="boxed" src="/images/azure-service-bus-tiers.png" style="max-width:480px" alt="Azure Service Bus Tiers">

- On azure portal, go to Resources and click on Add. Search for `Service Bus`, then click on create.
- Enter desired name for your Service Bus service. Select Subscription, Resource Group, Location and either __Standard or Premium pricing tier__.
- Once created, go to the service and click on `Shared access policies`.
-- Click on Add. Create a new Policy named `ManageSendAccessKey` with `Send` and `Listen` access.
-- Click on newly created `ManageSendAccessKey` policy and copy one of the `connection strings`. You will need this later. <sup><a name="service-bus-connection-string">[connection string]</a></sup>
- Now click on `Topics`.
-- Create a topic and call it `cmp_content`. Set desired configuration on the topic. <sup><a name="cmp-topic">cmp topic</a></sup>
-- Create another topic and call it `sitecore_notifications`. Set desired configuration on the topic. <sup><a name="sitecore-notifications-topic">sitecore notifications topic</a></sup>
-- Go inside the `cmp_content` topic and create a subscription called `sitecore`. <sup><a name="cmp-topic-subscription">cmp topic subscription</a></sup>

The service bus is now ready. CMP will send messages to `cmp_content` topic and the connector will subscribe to `sitecore` subscription and act when messages are added to the topic.

## Content Hub ##
In Content Hub, you need an `Action` of type `Azure Service Bus` and a `Trigger` that invokes this action when the desired content should send a message to the service bus.

### Action ###
- Log in to Content Hub and go to  __Manage__ -> __Actions__.
- Click on new Action. Enter desired name (e.g. `CMP - Content to Service Bus`) and label. Under type, select `Azure Service Bus`.
-- Enter [copied](#service-bus-connection-string) Service Bus connection string.
-- Select `Topic` as `Destination type`.
-- Enter `cmp_content` <sup>[cmp topic](#cmp-topic)</sup> in `Destination` field.
-- Test Connection and click Save when done.

### Trigger ###
- Log in to Content Hub and go to  __Manage__ -> __Triggers__.
- Click on new Trigger. Enter desired name (e.g. `CMP - On Published Content`) and description. Under objective, select `Entity creation` and `Entity modification`.
- Set execution type to `In background`.
- Click on `Conditions` tab and add a `Definition` with `Content (M.Content)`. If left like this, the trigger will execute whenever an entity of type `M.Content`, CMP's default, is created or modified. You can add as many conditions as you wish if you want to restrict which content is sent to the service bus. For example, if you want only Published content to be sent to the bus, add a condition like:  `Active state (MContentToActiveState)` `added item` `contains` `any` `Published`.
- Click on `Actions` tab and add created action `CMP - Content to Service Bus` to `Post Actions`.
- Save trigger and make sure it's enabled.

## Sitecore Configuration ##
Now that Azure and Content Hub have been configured, you are ready to install and configure the connector; and then, create Content Hub to Item mappings.

### Connector Configuration ###
- Download Sitecore Connect™ for Sitecore CMP 1.0.0 from [sitecore website](https://dev.sitecore.net/Downloads/Sitecore_Connect_for_Sitecore_CMP/10/Sitecore_Connect_for_Sitecore_CMP_100.aspx).
- Install the package on your CM server via Sitecore package installer.
- Navigate to item `/sitecore/system/Modules/CMP/Config`
-- Populate the fields as follows:
  `Client Id` and `Client Secret` __:__ from a Content Hub `OAuth client`. Find or create one under __Manage__ -> __OAuth Clients__.
  `User Name` and `Password` __:__ from a Content Hub `User` with sufficient permissions to read content from entities being pushed.
  `Content Hub URI` __:__ the full URL of your Content Hub instance without trailing slash.
  `Connection string` __:__ [service bus](#service-bus-connection-string) connection string.
  `Incoming topic name` __:__ <em>cmp_content</em> <sup>[cmp topic](#cmp-topic)</sup>.
  `Incoming subscription name` __:__ <em>sitecore</em> <sup>[cmp topic subscription](#cmp-topic-subscription)</sup>.
  `Outgoing topic name` __:__ <em>cmp_content</em> <sup>[sitecore notifications topic](#sitecore-notifications-topic)</sup>.
  `Default Language` __:__ <em>en</em>. This item language will be used if no language is specified in Content Hub. Note: `en` is default in Sitecore whereas `en-US` is default in Content Hub.
  
### Templates and Mappings ###
<ul><li> Create a new template that contains `/sitecore/templates/CMP/Content Hub Entity` as a Base template under desired templates folder. e.g. `Advertisement`. <sup><a name="sitecore-template">sitecore template</a></sup><ul><li>Add the fields you want to sync from Content Hub. Select appropriate field types (Only text fields are supported). </li> <li>Mark as `Bucketable`. </li></ul></li><li>Create a new Bucket under `/sitecore/content/CMP`. e.g. `Advertisements`.</li><li>Navigate to item `/sitecore/system/Modules/CMP/Config` and create a new `/sitecore/templates/CMP/Entity Mapping` item under it. e.g. `Advertisement`.<ul><li>Populate the fields as follows:<ul><li>`Content Type Id` __:__ <em>M.ContentType.Advertisement</em>. To find this id, go to your Content Hub instance and navigate to __Manage__ -> __Taxonomy__ -> __M.ContentType__, and click on the info icon next to the desired type. The `identifier` is what you need.</li><li>`Bucket` __:__ <em>/sitecore/content/CMP/Advertisements</em>. The parent location where you want the items to be created.</li><li>`Template` the newly created sitecore template containing the fields to map to<sup>[sitecore template](#sitecore-template)</sup>.</li></ul></li><li>Create an item of type `/sitecore/templates/CMP/Field Mapping` under it. e.g. `Title`.<ul><li>Populate the fields as follows:<ul><li>`CMP Field Name` __:__ <em>Advertisement_Title</em>. To find this id, you can either find the field on the `M.Content` entity definition under __Manage__ -> __Schema__ -> __M.Content__, or find a content entity with the desired type, grab the entity id from the URL and navigate to `https://content-hub-instance/api/entities/entity_id` to find the property id.</li><li>`Sitecore Field Name` __:__ <em>Title</em>. The field name where the data should map to. </li></ul></li><li>Create as many items as fields are required.</li></ul></li></ul></li></ul>

Voilà! Now your connector has been configured. Restart your sitecore instance and items should be created as soon as the Content Hub trigger executes. 
Hint: Look at your sitecore logs to find errors and it logs a lot when something fails.


====================

References:
https://dev.sitecore.net/Downloads/Sitecore_Connect_for_Sitecore_CMP/10/Sitecore_Connect_for_Sitecore_CMP_100.aspx


---

Please let me know what you think and/or if you can spot any errors.
*/eom*
