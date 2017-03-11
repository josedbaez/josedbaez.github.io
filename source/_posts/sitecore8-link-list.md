title: Monoco Link List field type in Sitecore 8
date: 2017/03/11
categories:
- Sitecore
- Field Types
tags:
- linklist
- fieldtypes

---
<img class="hero-img" src="/images/linklist.png" alt="Link list">
---
Update Monoco Link List field type source code so it can be used in Sitecore 8...
<!-- more -->

*Versions used: Sitecore 8.2 rev. 161221 (8.2 Update-2)*

Have you ever wanted to add N number of hyperlinks to an item, but did not want to rely on traversing its children? How cool would it be to have all these links in the same field? Thanks to [Monoco's link list field](http://code.monoco.se/2012/12/a-shiny-new-field-type-linklist), we can!. The problem? It was created a while ago and it doesn't work in Sitecore 8+ versions.

This post explains how to update the original code so it can be used in Sitecore 8+ versions, adds the text property and amends code so it sets the target item when using internal links. If you just want the code, download [Source code here](#source-code)

*NB: I did this a long time ago under tight deadline and never found time to blog about it; got inspired by a question I recently saw on [SSE](http://sitecore.stackexchange.com) (which I can't find now). The solution might not be the prettiest code ever, and might have some redundant stuff (which is the reason I never sent a pull request to creator, but it works with minor changes to the original code*

**Steps:**
1. [Download](https://marketplace.sitecore.net/en/Modules/Link_List_Field_Type.aspx) and install the module from the marketplace.
    If you try to use the field type now, it won't render properly and will look like this:
    <img class="" src="/images/broken-link-list.png" alt="Broken Monoco Link List">

2. [Download](https://github.com/monoco/Monoco.CMS.FieldTypes) the original code and open the solution/project in visual studio.

3. Update `Sitecore.Kernel` reference in project.

4. Change target framework to 4.5.2 or later.

5. Delete layout folder. This comes as an example but doesn't compile properly and it's not needed.

6. Go to file `\Fields\Link.cs` and add the following property:
``` csharp
public string Text { get; set; }
```

7. Go to file `\Fields\LinkListField.cs` and make the following changes:
  1. Add these 2 namespace references:
  ``` csharp
  using Sitecore.Links;
  using System.Xml;
  ```
  2. Add this new function that will get target item for internal links. *This should definitely be refactored and re-use code used in `GetUrlFromLink(XElement link)` as it's pretty much the same.*
  ``` csharp
  private Item GetTargetItemFromLink(XElement link)
  {
      switch (GetElementAttribute(link, "linktype").ToLowerInvariant())
      {
          case "internal":
              var id = GetElementAttribute(link, "id");
              if (!String.IsNullOrWhiteSpace(id))
              {
                  Sitecore.Data.ID itemId;
                  if (Sitecore.Data.ID.TryParse(id, out itemId))
                  {
                      var item = Sitecore.Context.Database.GetItem(itemId);

                      return item ?? null;
                  }
                  return null;
              }
              return null;
          case "media":
              var id2 = GetElementAttribute(link, "id");
              if (!String.IsNullOrWhiteSpace(id2))
              {
                  Sitecore.Data.ID mediaId;
                  if (Sitecore.Data.ID.TryParse(id2, out mediaId))
                  {
                      var item = Sitecore.Context.Database.GetItem(mediaId);
                      if (item == null)
                          return null;
                      var mediaItem = (MediaItem)item;
                      return mediaItem;
                  }
              }
              return null;
          default: // all other links are considered external.
              return null;
      }
  }
  ```
  3. Modify method `ParseLinks()` so `_links` variable sets `Text` and `TargetItem`. It should look like this:
  ``` csharp
  _links = from link in document.Descendants("link")
     select new Link
     {
         Target = GetElementAttribute(link, "target"),
         Title = GetElementAttribute(link, "title"),
         Text = GetElementAttribute(link, "text"),
         LinkType = GetElementAttribute(link, "linktype"),
         Url = GetUrlFromLink(link),
         TargetItem = GetTargetItemFromLink(link)
     };
  ```

8. Go to file `\sitecore modules\Shell\FieldTypes\LinkListField.cs` and make the following changes:
  1. Add the following namespace reference:
  ``` csharp
  using Sitecore.Web;
  ```
  2. In method `EditLink(ClientPipelineArgs args)`, change the `else` clause to:
  ``` csharp
  // Show the dialog using ShowModalDialog.
  var urlString = new UrlString(args.Parameters["url"]);
  UrlHandle urlHandle = new UrlHandle();
  urlHandle["ro"] = Source;
  urlHandle["db"] = Client.ContentDatabase.Name;
  urlHandle["va"] = node.OuterXml;
  urlHandle.Add(urlString);
  urlString.Append("ro", Source);
  urlString.Append("sc_content", WebUtil.GetQueryString("sc_content"));
  Sitecore.Context.ClientPage.ClientResponse.ShowModalDialog(urlString.ToString(), true);
  args.WaitForPostBack();
  ```
  3. In method `InsertLink(ClientPipelineArgs args)`, change lines 156-158 to:
  ``` csharp
  Sitecore.Context.ClientPage.ClientResponse.Eval(string.Concat(new string[]
  {
      "scContent.linklistInsertLink('", ID, "', {text:'", selectText, "'})"
  }));
  ```
  4. And the `else` clause right below to:
  ``` csharp
  var urlString = new UrlString(args.Parameters["url"]);
  UrlHandle urlHandle = new UrlHandle();
  urlHandle["ro"] = Source;
  urlHandle["db"] = Client.ContentDatabase.Name;
  urlHandle["va"] = this.XmlValue.ToString();
  urlHandle.Add(urlString);
  urlString.Append("ro", Source);
  urlString.Append("sc_content", WebUtil.GetQueryString("sc_content"));
  Sitecore.Context.ClientPage.ClientResponse.ShowModalDialog(urlString.ToString(), true);
  args.WaitForPostBack();
  ```
  5. Lastly, the function `string GetSelectText(XmlNode node)` to the following:
  ``` csharp
  private string GetSelectText(XmlNode node)
  {
      string attr = GetAttribute(node, "text");
      var linkType = GetLinkType(node);
      string url = string.Empty;
      if (linkType.ToString().Equals("internal") || linkType.ToString().Equals("media"))
      {
          string itemId = GetAttribute(node, "id");
          if (!string.IsNullOrEmpty(itemId))
          {
              var item = Client.ContentDatabase.GetItem(new Sitecore.Data.ID(itemId));
              if (item != null)
              url = item.Paths.ContentPath;
        }
      }
      else
      {
          var urlObj = GetLinkUrl(node);
          if (urlObj != null)
          url = urlObj.ToString();
      }
      return string.Format("{0} ({1}: {2})", attr, linkType, url);
}
```
9. And that's it. Compile your code and copy all relevant files to your website folder. The link list field should now look like this:
<img class="hero-img" src="/images/working-link-list.png" alt="Broken Monoco Link List">

====================

<a name="source-code">Source code:</a> Do not forget to update the reference to `Sitecore.Kernel`. [Download here](https://github.com/josedbaez/josedbaez.github.io/raw/source/source/files/Monoco.CMS.FieldTypes.zip)


---

Please let me know what you think and/or if you can spot any errors.
*/eom*
