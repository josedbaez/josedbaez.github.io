title: Media.AlwaysAppendRevision - Unrecognized Guid
date: 2018/11/19
categories:
- Sitecore
- Azure
tags:
- bug
- cdn
- azure

---
<img class="hero-img" src="/images/alwaysappendrevision-bug.jpg" alt="Always Append Revision.jpg">
---
Unrecognized Guid format error when Media.AlwaysAppendRevision is set to true.
<!-- more -->

*Versions used: Sitecore Experience Platform 9.0 rev. 171219 (9.0 Update-1).*

After reading [Kam's post](https://jammykam.wordpress.com/2017/02/13/sitecore-azure-cdn/) about changes needed to serve media library items via Azure's CDN, I implemented a custom Media Provider to append the last modified date to the media URL -so we don't need to purge URLs on azure when a media item is modified. Then a colleague pointed out Sitecore provides a setting `Media.AlwaysAppendRevision` that does exactly this (but with revision number instead).

Unfortunately, this setting breaks the default Media Provider. An `[FormatException: Unrecognized Guid format.]` error if shown if you try to add a language version to a media item. 

Sitecore has released an official patch to fix the see. Grab it from [here](https://github.com/sitecoresupport/Sitecore.Support.219261/releases)


---

Please let me know what you think and/or if you can spot any errors.
*/eom*
