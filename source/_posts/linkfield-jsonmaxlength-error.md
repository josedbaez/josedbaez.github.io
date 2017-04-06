title: General Link maxJsonLength error
date: 2017/04/06
categories:
- Sitecore
- Field Types
tags:
- commerce
- generallink
- bug

---
<img class="hero-img" src="/images/general-link-bug.jpg" alt="Sitecore General Link Bug">
---
Fix error related to maxJsonLength when using general link in content and experience editor. The length of the string exceeds the value set on the maxJsonLength property...
<!-- more -->

*Versions used: Sitecore 8.1 rev. 151207 (Update-1).*

When using general link, if you try to expand an item that has a large number of children (which should be avoided, but sometimes is out of your hands; in our case, they were commerce items imported by the module) you will notice it never ends loading the children. If you open the browser console, you will see an error like the following:

``` JSON
{
   "statusCode":500,
   "error":{
      "message":"Error during serialization or deserialization using the JSON JavaScriptSerializer. The length of the string exceeds the value set on the maxJsonLength property."
   }
}
```

When you click on the `+` icon, an AJAX request is sent to an Item Web API method which is using the class `System.Web.Script.Serialization.JavaScriptSerializer` to serialize the object being returned. The issue? `maxJsonLength` default size is 2MB.  We tried adding setting `<jsonSerialization maxJsonLength="######"/>` in web.config but it didn't fix the issue.

We contacted sitecore support about this and it was flagged as a bug (ref no. 94776). The official solution is [here](https://github.com/SitecoreSupport/Sitecore.Support.94776/releases/tag/8.1.1.0). It now allows us to set the maximum allowed size in a new setting called `JsonSerialization.MaxLength`



---

Please let me know what you think and/or if you can spot any errors.
*/eom*

