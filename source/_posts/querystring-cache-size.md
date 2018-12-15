title: Increase querystring cache size
date: 2018/12/09
thumbnail: /images/querystring-cache_th.jpg
categories:
- Sitecore
tags:
- logs
- cache
- patch

---
<img class="hero-img" src="/images/querystring-cache.jpg" alt="Querystring cache">
---
How to increase WebUtil.QueryStringCache via configuration patch.
<!-- more -->

*Versions used: Sitecore Experience Platform 9.0 rev. 171219 (9.0 Update-1).*

I was going through sitecore logs and stumbled across a warning message that was coming up very often. Usually, many Warning messages are ignored or caused by something else that should be looked into. In this case the recurrent warning message was related to `WebUtil.QueryStringCache` being cleared:

___WARN  WebUtil.QueryStringCache cache is cleared by Sitecore.Caching.Generics.Cache`1+DefaultScavengeStrategy[[System.String, mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089]] strategy. Cache running size was 17.7 KB___

My first thought was I could easily look for the sitecore setting and patch the number to something larger as the implementation I was working on was big. But... I couldn't find such setting. When the site is started you can find a message of the size that reads: 
_INFO  Cache created: 'WebUtil.QueryStringCache' (max size: 19KB, running total: 5596MB)_

As I couldn't find any setting where this 19KB was being set, I fired up ILSpy and started digging around. I found `Sitecore.Caching.CacheManager.GetNamedInstance("WebUtil.QueryStringCache", 20000L, true)` in `WebUtil.cs`, but couldn't figure out how to override this value. I started discussing with my colleague [Viacheslav](https://community.sitecore.net/members/viacheslavsedletskiy_5f00_1037964896) and he also starting digging through Sitecore's dlls and found out this can be changed through a named `cacheContainer`.

Without further ado, the patch to increase `WebUtil.QueryStringCache`:
``` xml
<sitecore>
  <cacheContainerConfiguration>
    <cacheContainer name="WebUtil.QueryStringCache" type="Sitecore.Caching.Cache, Sitecore.Kernel" singleInstance="true">
      <param desc="name">WebUtil.QueryStringCache</param>
        <param desc="cacheSize">3500KB</param>
    </cacheContainer>
  </cacheContainerConfiguration>
</sitecore>
```
To confirm your new value is being set, you can read the startup message which should now look like:
    
__INFO  Cache created: 'WebUtil.QueryStringCache' (max size: 3MB, running total: 5280MB)__

PS: As I couldn't find any official documentation about this, I contacted sitecore support to confirm this patch was Ok. This is an excerpt from the conversation: 

> This cache cannot be changed from the configuration. In most cases, this cache should not be tuned. (The notification should be ignored)... The best practice of tuning Sitecore caches is via cache max size settings. However, as there is currently no such setting which would control the QueryStringCache size, this approach is indeed acceptable... The issue with the QueryStringCache tuning is now addressed in the wish #220607.

---

Please let me know what you think and/or if you can spot any errors.
*/eom*
