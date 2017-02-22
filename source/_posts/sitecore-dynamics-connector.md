title: Install and configure Sitecore Dynamics Connect
date: 2017/23/02
categories:
- Sitecore
- CRM
tags:
- dynamics
- xdb

---
<img class="hero-img" src="/images/contact.jpg" alt="Contact">
---
How to install and configure Sitecore Dynamics Connect module...
<!-- more -->

*Versions used: Sitecore 8.2 rev. 161221 (8.2 Update-2). Dynamics CRM Connect 1.3*

Sitecore Dynamics Connect module allows you to exchange data between Sitecore and Dynamics. In Sitecore's [words](https://dev.sitecore.net/Downloads/Dynamics_CRM_Connect/Dynamics_CRM_Connect_1/Dynamics_CRM_Connect_1_3.aspx): "Integrate data from Dynamics CRM with Sitecore, and data from Sitecore with Dynamics CRM."

On a previous project, we were using the module to keep xDb contacts in sync as they were being managed in CRM.

This post outlines how to install and configure the module so you can start using out of the box pipelines.

**Steps:**

1. [Download](https://dev.sitecore.net/Downloads/Dynamics_CRM_Connect/Dynamics_CRM_Connect_1/Dynamics_CRM_Connect_1_3.aspx) and install packages in the following order:
    1. Data Exchange Framework 1.3
    2. Exchange Framework 1.3
    3. Dynamics CRM Provider for Data Exchange Framework 1.3

2. Modify your `ConnectionStrings.config` file and add a connection string to CRM instance. *(e.g. ```<add name="CRM" connectionString="Url=https://mycrm.dynamics.com; Username=hola;Password=ppp;" />```)* [MSDN](https://msdn.microsoft.com/en-us/library/gg695810) for more info on Dynamics Connection Strings.

3. Go to item `/sitecore/system/Data Exchange` and create a new `Data Exchange Tenant for Dynamics CRM` item under it.

4. Go to item `/sitecore/system/Data Exchange/[Your Item]/Tenant Settings/Dynamics CRM/Repository Sets` and create a new `XRM Client Entity Repository Set` item under it.
    1. Set `Connection String Name` field to your connection string name from step 2
    2. Click on Test Connection <sup>[x](#assembly-error)</sup>

5. Go to item `
/sitecore/system/Data Exchange/[Your Item]]/Endpoints/Providers/Dynamics CRM/Dynamics CRM Entity Endpoint` and set field `Entity Repository Set` to your repository set (step 4). Save item.

6. Go to item `/sitecore/system/Data Exchange/[Your Item]` and check field `Enabled`

That's it. You have configured the connection to Dynamics and can now run any pipeline under `
/sitecore/system/Data Exchange/[Name Of Item]/Pipeline Batches` by clicking on the item and `Run Pipeline Batch` from the `Data Exchange` menu.



<a name="assembly-error">If you get an error like:</a> `Could not load file or assembly 'Microsoft.Xrm.Sdk, Version=7.0.0.0...`

You need to add the following dependency assembly to your web.config (This has been reported to sitecore support and the official docummentation will get amended).

<dependentAssembly>
   <assemblyIdentity name="Microsoft.Xrm.Sdk" publicKeyToken="31bf3856ad364e35" culture="neutral" />
   <bindingRedirect oldVersion="0.0.0.0-8.0.0.0" newVersion="8.0.0.0" />
</dependentAssembly>

<dependentAssembly>
   <assemblyIdentity name="Microsoft.Crm.Sdk.Proxy" publicKeyToken="31bf3856ad364e35" culture="neutral" />
   <bindingRedirect oldVersion="0.0.0.0-8.0.0.0" newVersion="8.0.0.0" />
</dependentAssembly>



---

Please let me know what you think and/or if you can spot any errors.
*/eom*

