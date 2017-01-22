title: Log4net rolling appender
date: 2017/01/23
categories:
- Sitecore
- Log4net
tags:
- log4net
- logs

---
<img class="hero-img" src="/images/logs.jpg" alt="Logs">
---
A way to reduce log file size by telling log4net to create multiple files per day instead of one...
<!-- more -->
 
*Versions used: Sitecore 8.1 rev. 151207 (Update-1).*

Have you ever had issues opening log files that are too big for text editors to process? I surely have. 

Log4net can be configured to use a [rolling file appender](https://logging.apache.org/log4net/release/config-examples.html#rollingfileappender) that creates multiple files once it reaches the maximum size per file set.

The following patch will delete default sitecore log4net appender and add a new one that tells log4net to create a log file every 50KB, per day. *Note: pick a much bigger number but not too big that editors struggle to open the files.*

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
    <maximumFileSize value="50KB" /> <!-- KB, MB, GB -->
    <preserveLogFileNameExtension value="true" />
    <staticLogFileName value="false" />
    <layout type="log4net.Layout.PatternLayout">
      <conversionPattern value="%4t %d{ABSOLUTE} %-5p %m%n" />
    </layout>
    <encoding value="utf-8" />
</appender>

```

And this is the result:

<img class="hero-img" src="/images/rolling-logs.png" alt="Rolling logs">


---

Please let me know what you think and/or if you can spot any errors.
*/eom*
