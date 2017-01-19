title: Synthesis Rendering Parameters Helper
date: 2016/09/11
categories:
- Sitecore
- Synthesis
tags:
- synthesis
- helper

---
<img class="hero-img" src="/images/parameters.png" alt="Parameters">
---
A synthesis helper that gets a rendering/component rendering parameters. If you are not familiar with or not using rendering parameters in your solution, I’d suggest...
<!-- more -->
*Versions used: Sitecore 8.1 rev. 160519 (Update-3). Synthesis 8.2.0*

One of the features provided by Sitecore to enrich content entry and separate concerns are Rendering Parameters. Rendering parameters should be used to empower editors to customise a component's behaviour, look-and-feel, or even functionality without changing its data source.

If you are not familiar with or not using rendering parameters in your solution, I’d suggest you read about it and start using them immediately. This post is about a [Synthesis](https://github.com/kamsar/Synthesis) helper that fetches and casts rendering parameters (when using Parameters Template field); NOT on how to use or implement rendering parameters.

In one of my previous projects, we were using rendering parameters heavily and decided to create a little helper that would return a specific component’s rendering parameter. As I had used [glass](http:// http://glass.lu/) before, and knew about [GetRenderingParameters]( http://www.glass.lu/mapper/sc/tutorials/tutorial23), I decided to check the source code to see how I could recreate something similar to use with synthesis. It turned out I was able to port the code without needing to make many changes so shout-out to the Glass team.

Here's the code implementantion with comments.

``` csharp
using System;
using System.Linq;
using Sitecore.Common;
using Sitecore.Data;
using Sitecore.Data.Items;
using Sitecore.Mvc.Presentation;
using Sitecore.SecurityModel;
using Sitecore.Web;
using Synthesis;

public static T GetRenderingParameters<T>(ID templateId) where T : IStandardTemplateItem
{
    T ret;
    //get parameters from rendering. They come like key=val&
    var parameters = RenderingContext.Current.Rendering["parameters"];
    if (string.IsNullOrEmpty(parameters))
        return default(T);

    //convert to name value collection for easier handling
    var parametersCollection = WebUtil.ParseUrlParameters(parameters);

    //create a temp sitecore item to hold rendering parameter information.
    var tempId = Guid.NewGuid().ToID();
    var itemDefinition = new ItemDefinition(tempId, "renderingparamitem", templateId, ID.Null);
    var itemData = new ItemData(itemDefinition, Sitecore.Context.Language, Sitecore.Context.Item.Version, new FieldList());
    var tempItem = new Item(tempId, itemData, RenderingContext.Current.ContextItem.Database);

    using (new SecurityDisabler()) //disable security to be able to create items.
    {
        using (new EnforceVersionPresenceDisabler())
        {
            tempItem.Editing.BeginEdit();
            var fields = parametersCollection.AllKeys;
            //set field values from params received
            foreach (var fieldName in fields.Where(fieldName => tempItem[fieldName] != null))
                tempItem[fieldName] = parametersCollection[fieldName];
            tempItem.Editing.EndEdit();
            ret = (T)tempItem.AsStronglyTyped(); //cast to synthesis object
        }
    }
    return ret;
}

```

The helper can be used like this:
`var renderingParam = GetRenderingParameters<IRenderingParamItem>(RenderingParam.ItemTemplateId);`
where `IRenderingParamItem` is the interface type of your rendering parameters template and `RenderingParam.ItemTemplateId` is the ID of said template.

Note: This helper is not checking for null, you should check for null doing something like:
`if (RenderingContext.Current.Rendering != null && RenderingContext.Current.Rendering.Parameters != null && RenderingContext.Current.Rendering.Parameters.Any())`


---

Please let me know what you think and/or if you can spot any errors.
*/eom*
