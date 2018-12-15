title: Application insights logger
date: 2017/11/23
categories:
- Sitecore
- Azure
tags:
- azure
- insights
- helper

---
<img class="hero-img" src="/images/app-insights.png" alt="Application Insights">
---
A little helper to log custom messages or errors to azure application insights.
<!-- more -->

*Versions used: Sitecore 8.2 rev. 170614 (8.2 Update-4). Microsoft.ApplicationInsights 2.4.0*

You can configure your sitecore website or any web application to log any exceptions to [Azure Application Insights](https://azure.microsoft.com/en-us/services/application-insights/), but sometimes you want to specifically log something when an action happens. The following helper allows you to send custom messages to application insights.

``` csharp
using System;
using System.Collections.Generic;
using System.Linq;
using Microsoft.ApplicationInsights;

public class InsightsLogger
{
    private readonly TelemetryClient telemetry = new TelemetryClient();

    public void LogMessage(Dictionary<string, string> message)
    {
        if(message!=null && message.Any())
            telemetry.TrackEvent("MyEvent", message);
    }

    public void LogException(Exception e)
    {
        telemetry.TrackException(e);
    }
}
```
Don't forget to add nuget package `Microsoft.ApplicationInsights`.

If you want to avoid logging every sql request, remove the SQL telemetry by doing this. 

Create a new processor to filter out SQL instances as follows (extracted from https://stackoverflow.com/a/38321304)

``` csharp
using Microsoft.ApplicationInsights.Channel;
using Microsoft.ApplicationInsights.DataContracts;
using Microsoft.ApplicationInsights.Extensibility;

public class NoSqlDependenciesProcessor : ITelemetryProcessor
{
    private ITelemetryProcessor Next { get; set; }
    public NoSqlDependenciesProcessor(ITelemetryProcessor next)
    {
        this.Next = next;
    }

    public void Process(ITelemetry item)
    {
        if (IsSQLDependency(item))
            return;
        this.Next.Process(item);
    }

    private bool IsSQLDependency(ITelemetry item)
    {
        var dependency = item as DependencyTelemetry;
        switch (dependency?.Type)
        {
            case null:
                return false;
            case "SQL":
                return true;
            default:
                return false;
        }
        return false;
    }
}
```

Open `ApplicationInsights.config` and add a new type (inside `<TelemetryProcessors>`) pointing to the processor above.

``` xml
...
...
    <Add Type="MyApp.NoSqlDependenciesProcessor, MyApp" />
</TelemetryProcessors>
```
---

Please let me know what you think and/or if you can spot any errors.
*/eom*
