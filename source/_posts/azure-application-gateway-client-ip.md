title: Reading user's ip address behind Azure Application Gateway
date: 2018/10/19
categories:
- Sitecore
- Azure
tags:
- azure
- gateway
- xff

---
<img class="hero-img" src="/images/application-gateway-xforwardedfor.jpg" alt="Azure application gateway xforwardedfor.jpg">
---
How to patch sitecore pipeline to properly read client's ip address behind Azure Application Gateway.
<!-- more -->

*Versions used: Sitecore Experience Platform 9.0 rev. 171219 (9.0 Update-1).*

A load balancer will typically send the client's IP through a custom header, so the application server can read this -if needed- instead of the load balancer's internal ip. A common header for this is `X-Forwarded-For` aka `XFF`. Sitecore provides an easy and customisable way to specify which header to use to read client's ip: `Analytics.ForwardedRequestHttpHeader` setting.

[Azure Application Gateway](https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-faq) is commonly used on Sitecore PaaS implementations because it provides more features than just load balancing. Web Application Firewall for an instance.

Azure's application gateway inserts the client's IP on the `XFF` header, but in a different format than sitecore expects. `The format for x-forwarded-for header is a comma-separated list of IP:Port.` Sitecore's `Sitecore.Analytics.Pipelines.CreateVisits.XForwardedFor` class does not expect the port to be included, hence it fails when parsing it to an `IPAddress`.

Luckily the fix for this is easy. You just need to patch pipeline `Sitecore.Analytics.Pipelines.CreateVisits.XForwardedFor, Sitecore.Analytics` with a method that excludes the port and parses the IP address properly.

Create a class that inherits from Sitecore's XForwardedFor class and override function `GetIpFromHeader`.
``` csharp
public class MyCustomXFF : Sitecore.Analytics.Pipelines.CreateVisits.XForwardedFor
{
    protected override string GetIpFromHeader(string header)
    {
        Assert.ArgumentNotNull(header, "header");
        string[] array = header.Split(',');
        int headerIpIndex = HeaderIpIndex;
        string text = (headerIpIndex < array.Length) ? array[headerIpIndex] : array.LastOrDefault();
        if (string.IsNullOrEmpty(text))
        {
            return null;
        }
        return text.Split(':')[0].Trim();
    }
}
```

Then apply config patch.
``` xml
<sitecore>
  <pipelines>
    <createVisit>
      <processor type="Namespace.MyCustomXFF, MyAssembly" patch:instead="*[@type='Sitecore.Analytics.Pipelines.CreateVisits.XForwardedFor, Sitecore.Analytics']">
        <HeaderIpIndex>0</HeaderIpIndex>
      </processor>
    </createVisit>
  </pipelines>
</sitecore>
```

---

Please let me know what you think and/or if you can spot any errors.
*/eom*
