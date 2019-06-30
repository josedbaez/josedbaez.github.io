title: Supported cultures on server
date: 2019/04/10
categories:
- Globalization
tags:
- globalization
- multilingual

---
<img class="hero-img" src="/images/greetings.png" alt="Greetings">
---
How to generate a table listing all supported .NET cultures on the server.
<!-- more -->

*Versions used: Sitecore Experience Platform 9.0 rev. 180604 (Update-2).*

Windows comes with predefined cultures and also supports adding custom cultures. I struggled to find a definite list whilst I was researching if languages needed by the client were available in our environments -so we could use them without issues. 

Moreover, our solution was hosted on Azure App Services and [Azure PaaS does not support adding custom cultures](https://blogs.msdn.microsoft.com/benjaminperkins/2018/06/11/implementing-custom-cultures-cultureinfo-localize-azure-app-service/). Benjamin Perkins does post a [link](http://customcultures.azurewebsites.net/) that lists all cultures on an Azure App Service but I wanted to double check all our environments could support the cultures we needed.

The easiest way I found was to create a razor file I could drop on a server and request it to get a table of supported cultures. [Full list here](https://josedbaez.com/files/supported-cultures.html)

The code is pretty basic. [Gist here](https://gist.github.com/josedbaez/2065bf10c9fbc2ba279ed69c80a3bf7b):
``` csharp
@foreach (var culture in System.Globalization.CultureInfo.GetCultures(System.Globalization.CultureTypes.SpecificCultures))
{
    <tr>
        <td>@culture.DisplayName</td>
        <td>@culture.NativeName</td>
        <td>@culture.TextInfo.CultureName</td>
        <td>@culture.Parent.Name</td>
        <td>@culture.TextInfo.CultureName.Replace(culture.Parent.Name, string.Empty).TrimStart('-')</td>
    </tr>
}
```

And you will get a [pretty table](https://divtable.com/table-styler/) like:
<img src="/images/cultures-sample.png" alt="Example of supported cultures table">


====================
References:
https://blogs.msdn.microsoft.com/benjaminperkins/2018/06/11/implementing-custom-cultures-cultureinfo-localize-azure-app-service/

---

Please let me know what you think and/or if you can spot any errors.
*/eom*
