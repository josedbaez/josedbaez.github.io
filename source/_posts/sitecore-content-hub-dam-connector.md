title: Sitecore connector for Content Hub DAM
date: 2019/08/19
categories:
- Sitecore
- Content Hub
tags:
- stylelabs
- connector
- dam

---
<img class="hero-img" src="/images/dam-connector.png" alt="Sitecore Content Hub DAM Connector">
---
How to install Sitecore's Connect Hub DAM connector "Sitecore Connect™ for Sitecore DAM 2.0.0"
<!-- more -->

*Versions used: Sitecore Experience Platform 9.2 Initial Release. Content Hub 3.2*

[Sitecore Connect™ for Sitecore DAM](https://dev.sitecore.net/Downloads/Sitecore_Plugin_for_Stylelabs_DAM/20/Sitecore_Connect_for_Sitecore_DAM_200.aspx) gives users the ability to insert Content Hub assets from Sitecore Content Management, and to upload assets from CM to Content Hub. It works on any Sitecore image and RTE field types; in both Content and Experience editors. To achieve this, the connector adds new buttons that invoke and display content hub on an iframe.

The connector will display the page set in Sitecore (explained below). Content Hub 3.2 comes with 2 pre-configured pages that can be found under: __Admin__ -> __Pages__ -> __Sitecore Plugin__. Here you can amend the pages as you find suitable or modify the search component configuration. I haven't tried yet but you might be able to create a custom page to use instead of the default ones.

The connector requires configuration in both systems (Content Hub and Sitecore CM instance) to work properly:
## Content Hub Configuration ##
- To allow a Sitecore instance to access content hub, you must add a CORS configuration entry with your Sitecore CM instance URL.
  1. Navigate to: __Manage__ -> __Settings__ -> __PortalConfiguration__ -> __CORSConfiguration__ or go to URL http://CONTENT_HUB_URL/en-us/admin/settingmanagement?setting=CORSConfiguration&settingId=8748
  2. Enter your CM instance URL (e.g. http://sc92.dev.local) and click Save.
- Then, update your SSO services entries to passive mode and add an `ExternalRedirectKeys` element with your Sitecore CM instance URL.
  1. Navigate to: __Manage__ -> __Settings__ -> __PortalConfiguration__ -> __Authentication__ or go to URL http://CONTENT_HUB_URL//en-us/admin/settingmanagement?setting=Authentication&settingId=525
  2. Find the element `Providers` and change `authentication_mode` to `Passive` on every provider present.
    **Tip:** *I find text mode (dropdown at the top) is best to edit json configuration in content hub.*
  3. Add a new element at the bottom of the document that looks like:
  ```json
    "ExternalRedirectKeys": {
      "Sitecore": "http://sc92.dev.local"
    }
  ```
    Here's an example of how these 2 elements should look:
  ```json
  ...
  ...
  "Providers": [
    {
      "$type": "Stylelabs.M.Portal.Authentication.SamlAuthenticationProviderConfigurator, Stylelabs.M.Portal",
      "metadata_location": "https://xxxxxx",
      "sp_entity_id": "urn:xxxxxxx.com",
      "idp_entity_id": "urn:xxxxxxx.com",
      "provider_name": "SSO",
      "authentication_mode": "Passive",
      "module_path": "AuthServices",
      "is_enabled": true
    }
  ],
  "ExternalRedirectKeys": {
    "Sitecore": "http://sc92.dev.local"
  }
  ```

## Sitecore Configuration ##
Now that Content Hub has been configured to allow connections from the Sitecore instance, you are ready to install and configure the connector:
- Download Sitecore Connect™ for Sitecore DAM 2.0.0 from [sitecore website](https://dev.sitecore.net/Downloads/Sitecore_Plugin_for_Stylelabs_DAM/20/Sitecore_Connect_for_Sitecore_DAM_200.aspx).
- Install the package on your CM server via Sitecore package installer.
  1. When prompted for an existing item, select overwrite -the package will create new buttons.
- Configure connector to point to your Content Hub instance.
  1. Navigate to item `/sitecore/system/Modules/DAM/Config/DAM connector`
  2. Populate the fields as follows:
  `DAMInstance` The root URL to your Content Hub instance (e.g. https://MY_CONTENT_HUB.com)
  `SearchPage` This will be the Content Hub page displayed in Sitecore. By default, it should be: https://MY_CONTENT_HUB.com/en-us/sitecore-plugin/approved-assets
  `ExternalRedirectKey` This matches the entry on element `ExternalRedirectKeys` added in the json document above. If you followed this example, should be: Sitecore

Once you have done this, you will find a new button that reads `Browse Sitecore DAM` on any image field and a new icon on rich text editor fields: <img src="/images/rte-dam-icon.png">

When selecting images via the connector, it sets the field value to the selected public link of the asset.

<img src="/images/dam-connector-demo.gif">
====================

References:
https://dev.sitecore.net/Downloads/Sitecore_Plugin_for_Stylelabs_DAM/20/Sitecore_Connect_for_Sitecore_DAM_200.aspx


---

Please let me know what you think and/or if you can spot any errors.
*/eom*
