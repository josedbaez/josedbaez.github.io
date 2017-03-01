title: CRM connector not removing contacts from marketing lists
date: 2017/03/02
categories:
- Sitecore
- CRM
tags:
- dynamics
- xdb
- bug

---
<img class="hero-img" src="/images/sitecore-dynamics.jpg" alt="Sitecore Microsoft Dynamics CRM">
---
Fix issue with CRM connector not removing contacts from marketing lists after syncing...
<!-- more -->

*Versions used: Sitecore 8.1 rev. 151207 (Update-1). Dynamics CRM Connect 1.1*

When [syncing marketing lists from CRM](http://integrationsdn.sitecore.net/DynamicsCrmConnect/v1.1/pipeline-batches/crm-marketinglist-sync/overview.html), you need the connector to add users to marketing lists but also to [remove them from lists](http://integrationsdn.sitecore.net/DynamicsCrmConnect/v1.1/pipeline-batches/crm-marketinglist-sync/about-reset-marketing-lists.html) they do not belong to anymore.

The removal of contacts from marketing lists happens in pipeline `Reset Existing xDB Contacts Pipeline`, but it does NOT WORK :(, hence contacts stay in a marketing list forever.

We contacted sitecore support about this and it was flagged as a bug (ref no. 139990) which is luckily easy to fix.

**Solution**

Go to item `/sitecore/system/Data Exchange/{Your CRM Tenant}/Value Mapping Sets/Clear CRM List Membership on xDB Contact/Marketing Lists` and untick field `Ignore null values`.

---

Please let me know what you think and/or if you can spot any errors.
*/eom*

