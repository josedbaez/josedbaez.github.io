title: Map Monoco Link List field type with Synthesis
date: 2017/03/15
categories:
- Sitecore
- Field Types
- Synthesis
tags:
- linklist
- fieldtypes
- synthesis

---
<img class="hero-img" src="/images/synthesizing.jpg" alt="Synthesizing">
---
Map strongly typed Link List field type with Synthesis when generating objects...
<!-- more -->

*Versions used: Sitecore 8.1 rev. 160519 (8.1 Update-3)*

This post is an addendum to my previous post about [Monoco Link List field type in Sitecore 8](/2016/09/synthesis-rendering-parameters-helper).

I love using object mappers. On a previous project I was using [synthesis](https://github.com/kamsar/Synthesis) and Link List field; and we wanted to be able to map our slick link list when generating synthesis models. I'll show how to do this here. [Source code here](#source-code)

*NB: As mentioned in the previous post, this is an old code done in a rush, and might not be up to a good code standard, but it's a good start and does the job.*

The first thing we have to do is copy `Fields\LinkListField.cs` class (I'll call it `LinkListFieldSynthesis.cs` in this post), extract an interface and add a new constructor required by Synthesis for lazy loading.

The interface looks like this:
``` csharp
public interface ILinkListFieldSynthesis
{
    IEnumerable<Link> Links { get; }
    string Root { get; set; }
    XmlDocument Xml { get; }
    Field InnerField { get; }
    string Value { get; set; }
    string GetAttribute(string name);
    void SetAttribute(string name, string value);
    List<WebEditButton> GetWebEditButtons();
    void Relink(ItemLink item, Item newLink);
    void RemoveLink(ItemLink itemLink);
    void UpdateLink(ItemLink itemLink);
    void ValidateLinks(LinksValidationResult result);
}
```
Then, you add a new constructor to the class that looks like this:
``` csharp
public LinkListFieldSynthesis(Synthesis.FieldTypes.LazyField lazyField, string indexValue)
  : base(lazyField.Value, "links")
{
}
```
And finally, we tell synthesis about this new field type and how to map it by adding the following to file `Synthesis.config`
``` xml
<map field="LinkList" type="Monoco.CMS.Fields.LinkListFieldSynthesis, Monoco.CMS" interface="Monoco.CMS.Fields.ILinkListFieldSynthesis, Monoco.CMS" />
```

That's it. Compile your code and generate your synthesis models which you can then use like
``` csharp
foreach (var link in Model.MyProperty.Links)
{
    <li>
        <a href="@link.Url" target="@link.Target" alt="@link.Title">@link.Text</a>
    </li>
}
```

====================

<a name="source-code">Source code:</a> [Download here](https://github.com/josedbaez/josedbaez.github.io/blob/source/source/files/LinkListFieldSynthesis.cs)


---

Please let me know what you think and/or if you can spot any errors.
*/eom*
