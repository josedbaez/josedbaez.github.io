title: Log4net rolling appender
date: 2017/01/23
categories:
- Sitecore
- Log4net
tags:
- log4net
- logs

---
<img class="hero-img" src="/images/parameters.png" alt="Parameters">
---
A way to reduce log files by telling log4net to create multiple ones per day instead of one...
<!-- more -->
*Versions used: Sitecore 8.1 rev. 151207 (Update-1).*



``` xml
<appender name="LogFileAppender" type="log4net.Appender.SitecoreLogFileAppender, Sitecore.Logging">
    <patch:delete />
</appender>
<appender name="LogFileAppender" type="log4net.Appender.RollingFileAppender">
    <file value="$(dataFolder)/logs/log" />
    <appendToFile value="true" />
    <datePattern value="'.'yyyyMMdd'.txt'" />
    <rollingStyle value="Composite" />
    <maxSizeRollBackups value="-1" />
    <maximumFileSize value="10KB" />
    <preserveLogFileNameExtension value="true" />
    <staticLogFileName value="false" />
    <layout type="log4net.Layout.PatternLayout">
      <conversionPattern value="%4t %d{ABSOLUTE} %-5p %m%n" />
    </layout>
    <encoding value="utf-8" />
</appender>

```




---

Please let me know what you think and/or if you can spot any errors.
*/eom*
