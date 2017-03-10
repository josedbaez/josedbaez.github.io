Change target framework to 4.5.2
Update sitecore.kernel reference

make sure the config is copied to your folder
Delete layout folder


Changes are on:
\Fields\LinkListField.cs
Add these 2 namespace references at the top the file
using Sitecore.Links;
using System.Xml;


\sitecore modules\Shell\FieldTypes\LinkListField.cs
Add the following namespace reference:
using Sitecore.Web;

In `public void EditLink(ClientPipelineArgs args)`, change the `else` clause to:
// Show the dialog using ShowModalDialog.
var urlString = new UrlString(args.Parameters["url"]);

UrlHandle urlHandle = new UrlHandle();
urlHandle["ro"] = Source;
urlHandle["db"] = Client.ContentDatabase.Name;
urlHandle["va"] = node.OuterXml;// this.XmlValue.ToString();
urlHandle.Add(urlString);

urlString.Append("ro", Source);
urlString.Append("sc_content", WebUtil.GetQueryString("sc_content"));
Sitecore.Context.ClientPage.ClientResponse.ShowModalDialog(urlString.ToString(), true);
args.WaitForPostBack();



In `InsertLink(ClientPipelineArgs args)`, Line 158.
Change this line
"scContent.linklistInsertLink('", ID, "', {text:'", selectText, "'})" }));

to:
"scContent.linklistInsertLink('", ID, "', {text:'", selectText, "'})"}));

and the else clause right below to:
// Show the dialog using ShowModalDialog.
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

Finally, change the `private string GetSelectText(XmlNode node)` method to the following:

string attr = GetAttribute(node, "text");
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
