title: EXM - Get xDb contact
date: 2017/01/18
categories:
- Sitecore
- EXM
tags:
- exm
- xdb

---
<img class="hero-img" src="/images/contact.jpg" alt="Contact">
---
How to retrieve xDb contacts data to use in EXM newsletter. When generating an EXM newsletter you might need to get contacts' details so you can personalise the newsletter further...

<!-- more -->
*Versions used: Sitecore 8.1 rev. 151207 (Update-1). EXM 3.2.0 rev. 151020*

When generating an EXM newsletter you might need to get contacts' details so you can personalise the newsletter further than just OOTB token replacement. For example, render different datasource depending on a contact's preference.

This post outlines how to get a contact's ID and the contact's object from xDB so you can read its data.

The following function retrieves the contact guid in all 3 EXM tabs where the contact is read: message, review and delivery. There are other ways of retrieving this (e.g. querystring) but the following works in every scenario and was recommended by an EXM developer at sitecore (sorry can't remember who it was but thank you!). __Do not forget to see the extension class required at the bottom__

``` csharp
private Guid GetContactId()
{
    var contactGuid = Guid.Empty;
    var recipientId = string.Empty;

    //get recipient ID
    var recipientIdObj = System.Web.HttpContext.Current.GetRecipientId();
    if (recipientIdObj != null)
        recipientId = recipientIdObj.ToString();

    if (!string.IsNullOrEmpty(recipientId))
    {
        recipientId = recipientId.Replace("xdb:", string.Empty);
        ShortID shortRecipientId;
        if (ShortID.TryParse(recipientId, out shortRecipientId))
            contactGuid = shortRecipientId.Guid;
    }

    return contactGuid != Guid.Empty ? contactGuid : Guid.Empty;
}
```

The function first gets the ID using sitecore's extension inside `Sitecore.Modules.EmailCampaign.Core.Extensions` and then converts it to GUID.

After you have the contact ID, you can retrieve the xDB contact like this:

``` csharp
var contactRepo = Sitecore.Configuration.Factory.CreateObject("contactRepository", true) as Sitecore.Analytics.Data.ContactRepository;

if (contactRepo == null)
    return null;

var leaseOwner = new LeaseOwner("admin", LeaseOwnerType.OutOfRequestWorker);
var xContactid = new XdbContactId(contactGuid);

var contact = contactRepo.TryLoadContact(xContactid.Value.Guid, leaseOwner, TimeSpan.FromMinutes(1));

//facets
var facets = contact.Object.Facets;
```


---

Please let me know what you think and/or if you can spot any errors.
*/eom*

