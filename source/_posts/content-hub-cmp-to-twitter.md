title: Tweet from Sitecore Content Hub CMP
date: 2019/11/22
categories:
- Sitecore
- Content Hub
tags:
- stylelabs
- cmp
- azure
- servicebus
- twitter

---
<img class="hero-img" src="/images/sitecore-twitter-logos.png" alt="Combined Sitecore Twitter Logos">
---
How to publish text and an image from Connect Hub CMP to twitter. Sample code using azure, c# and Content Hub's Web Client SDK. 
<!-- more -->

*Versions used: Content Hub 3.2*

You can publish content to twitter (or any other channels) by pushing CMP content to an intermediate layer (e.g. azure function), or straight to the channel -using a Content Hub Action Script or API. I prefer doing it outside Content Hub. Why? to free up resources and delegate responsibility: CMP is responsible for pushing content but does not care what happens to it afterwards. 

The solution in this post uses an [Azure Service Bus topic](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-queues-topics-subscriptions) to receive messages from CMP and pass them on to subscribers. You don't necessarily need a topic but it's a good idea if you have multiple channels where the same data must be published to. [Full source code here](https://github.com/josedbaez/Sitecore.ContentHub.Twitter)

<img class="boxed" src="/images/azure-queue-topics.png" style="max-width:480px" alt="Azure Queue and Topics">
<!-- Source: https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview -->

It is also the mechanism used by  [Sitecore Connect™ for Sitecore CMP](https://dev.sitecore.net/Downloads/Sitecore_Connect_for_Sitecore_CMP/10/Sitecore_Connect_for_Sitecore_CMP_100.aspx) connector explained on my [previous post](/2019/10/sitecore-content-hub-cmp-connector), so I will be linking to that one to avoid duplicating content.

*Steps*
- Create a [twitter app](https://developer.twitter.com/en/apps) and grab all keys and tokens under `Keys and tokens`.
- Create a Content Hub trigger and action to send specific content entities to azure. [Explanation here](/2019/10/sitecore-content-hub-cmp-connector#Azure-Services-Bus#Content-Hub).
- Create an Azure Service Bus service that allows the creation of topics. For this, you will need Standard or Premium service tier. You can follow the steps listed [here](/2019/10/sitecore-content-hub-cmp-connector#Azure-Services-Bus). On the last point, name your subscription `twitter` instead of `sitecore`.
- Create a new Azure Function App.
- Create a new [Service Bus Trigger](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-service-bus#trigger) function, inside the newly created Function App, that subscribes to the topic `cmp_content` and `twitter` subscription created on step 1. Choose whichever Development Environment you wish. I'd recommend outside the portal so you can source control it. You will need to use either [REST API](https://docs.stylelabs.com/content/integrations/rest-api/about.html?v=3.2.1) or [Web Client SDK](https://docs.stylelabs.com/content/integrations/web-sdk/index.html?v=3.2.1). __You can write the azure function on any language if you use the REST API__. 

The following example uses the Web Client SDK -so it's written in C#- and uses [Tweetinvi](https://www.nuget.org/packages/TweetinviAPI/) library. I've removed some validations to make it shorter but you can find full [source code here]((https://github.com/josedbaez/Sitecore.ContentHub.Twitter)).

``` csharp
[FunctionName("TweetContent")]
public static async Task Run([ServiceBusTrigger("cmp_content", "twitter", Connection = "Manage_Send_Listen_ConnectionString")]
    ServiceBusMessageRequest chMessage, ILogger logger)
{
    var contentId = chMessage.Message.TargetId;
    var contentEntity = await GetContentEntity(contentId, logger);

    Tweetinvi.Auth.SetUserCredentials("TwitterConsumerKey", "TwitterConsumerSecret", "TwitterUserAccessToken", "TwitterUserAccessSecret");

    var tweetImage = await GetTwitterImage(contentEntity);
    var tweet = contentEntity.GetPropertyValue<string>(DescFieldId);
    Tweetinvi.Tweet.PublishTweet(tweet, new PublishTweetOptionalParameters{ Medias = { tweetImage } });
}

private static async Task<IEntity> GetContentEntity(long entityId, ILogger logger)
{
    var propertyLoad = new PropertyLoadOption(new string[] { "Advertisement_Body" });
    var relationLoad = new RelationLoadOption(new string[] { "CmpContentToLinkedAsset" });
    var loadConfig = new EntityLoadConfiguration(CultureLoadOption.Default, propertyLoad, relationLoad);
    IEntity entity = await MConnector.Client.Entities.GetAsync(entityId, loadConfig);
    return entity;
}

private static async Task<IMedia> GetTwitterImage(IEntity content)
{
    var relation = content.GetRelation<IParentToManyChildrenRelation>("CmpContentToLinkedAsset");
    if(relation != null)
    {
        var ids = relation.GetIds();
        var asset = await MConnector.Client.Entities.GetAsync(ids.First());
        var downloadReviewRenditions = asset.Renditions.Where(x => x.Name.Equals("downloadPreview"));
        if(downloadReviewRenditions != null)
        {
            var href = downloadReviewRenditions.First().Items.First().Href;
            using (var webClient = new WebClient())
            {
                byte[] imageBytes = webClient.DownloadData(href);
                var media = Upload.UploadBinary(imageBytes);
                return media;
            }
        }
    }
    return null;
}
```
Content Hub sends a JSON object that contains the EntityID (among other properties not needed), so using the SDK, the code gets the Entity and the property (Advertisement_Body) to be used.

It then logs in to twitter to be able to upload the image before it can we linked to the tweet. Retrieves the `downloadPreview` rendition of the first image linked (via attachments) to the Content entity and upload the byte array to twitter.

Finally, it publishes the tweet with the text and the image.

====================

---

Please let me know what you think and/or if you can spot any errors.
*/eom*
