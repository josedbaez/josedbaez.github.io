title: Extending Sitecore connector for Content Hub CMP
date: 2020/02/05
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
How to extend Sitecore's Connect Hub CMP connector to synchronize images.
<!-- more -->

*Versions used: Sitecore Experience Platform 9.2 Initial Release. Content Hub 3.2.0*

This post is way overdue as I was supposed to release it after my last talk in december at [The Sitecore Sessions](https://www.meetup.com/The-Sitecore-Sessions/). Sorry for the delay 😞.

On a [previous post](/2019/11/content-hub-cmp-to-twitter) I explained how to install [Sitecore Connect™ for Sitecore CMP](https://dev.sitecore.net/Downloads/Sitecore_Connect_for_Sitecore_CMP/10/Sitecore_Connect_for_Sitecore_CMP_100.aspx) to automatically send entities from CMP to Sitecore. Alas, the connector only synchronizes text fields. 

This post explains the [CMP connector extension]() that synchronizes Content Hub images set on `M.Content` entities. Using the same methodology, this could grow into a module and include other field types (e.g. references to other entities). [MISSING Source Code here MISSING]().

This solution extends the existing CMP connector by adding a new processor to process a new mapping field type that maps an image relation to a Sitecore field using the same format supported by [Sitecore connector for Content Hub DAM](https://dev.sitecore.net/Downloads/Sitecore_Plugin_for_Stylelabs_DAM/20/Sitecore_Connect_for_Sitecore_DAM_200.aspx); so DAM connector needs to be [installed](/2019/08/sitecore-content-hub-dam-connector/). 

The default connector will process items of template type `/sitecore/templates/CMP/Field Mapping`, so we need a new template item (`/sitecore/templates/CMP/Image Field Mapping`) that our custom processor can use to identify image mapping configurations. This template has `Field Mapping` as its base template plus 2 extra fields: `AssetIndex` to specify the index of the asset relation to grab (if multiple are allowed), and `RenditionToUse` to specify which asset rendition public link to use. `CMP Field Name` is used as the relation to find the asset on (`CmpContentToLinkedAsset` is avaiable OOTB). 

The new processor runs after the default ones. [MISSING LINK TO XML FILE MISSING]()
``` xml
<pipelines>          
    <cmp.importEntity>                
        <processor type="Sitecore.SharedSource.CMP.Connector.CustomFields.Pipelines.SaveImageFieldValues, Sitecore.SharedSource.CMP.Connector.CustomFields" patch:after="processor[@type='Sitecore.Connector.CMP.Pipelines.ImportEntity.SaveFieldValues, Sitecore.Connector.CMP']"/>
    </cmp.importEntity>
</pipelines>
```

This processor [MISSIN LINK TO SaveImageFieldValues.cs MISSING]() fetches all `Field Mapping` items (from `ImportEntityPipelineArgs` object) under the `Entity Mapping` item being processed, reads required field values to find the rendition from the relation and stores the formatted public link URL into the specified sitecore field.

``` csharp
foreach (Item item in from i in args.EntityMappingItem.Children
                                              where i.TemplateID == Constants.ImageFieldMappingTypeID
                                              select i)
{
    var cmpFieldName = item[Sitecore.Connector.CMP.Constants.FieldMappingCmpFieldNameFieldId];
    var sitecoreFieldName = item[Sitecore.Connector.CMP.Constants.FieldMappingSitecoreFieldNameFieldId];
        
    var renditionField = item[Constants.FieldMappingSitecoreRenditionFieldID];
    var publicLink = GetPublicLinkData(contentHubHost, args.Entity, cmpFieldName, renditionField, assetIndexField).GetAwaiter().GetResult();

    if(publicLink != null && !string.IsNullOrEmpty(publicLink.URL))
    {
        var imgElement = GetContentHubDamImageElement(contentHubHost, publicLink);
        args.Item[item[Sitecore.Connector.CMP.Constants.FieldMappingSitecoreFieldNameFieldId]] = imgElement;
    }
}
```
As you can see, the code is quite simple. It loops through `ImageFieldMapping` items and  reads the pertinent fields; gets or creates a public link to the defined rendition, and stores the formatted image tag into the image field value.

`<image stylelabs-content-type="Image" mediaid="" src="https://HOST/api/public/content/364944895?v=779a" height="1100" alt="" stylelabs-content-id="1235" width="790" thumbnailsrc="https://HOST/api/gateway/1235/thumbnail" />`


====================
References:
MISSING LINK TO SOURCE CODE

---

Please let me know what you think and/or if you can spot any errors.
*/eom*
