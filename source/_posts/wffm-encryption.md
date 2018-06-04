title: WFFM data encryption - Sitecore 8
date: 2016/09/05
categories:
- Sitecore
- WFFM
tags:
- wffm
- encryption
- security

---
<img class="hero-img" src="/images/encryption-lock.jpg" alt="Encryption">
---
How to encrypt any data stored by WFFM in both mongo and SQL databases. The code provided here will encrypt values in `FormData` and `FormFieldValues` tables. Also...
<!-- more -->
 
*Versions used: Sitecore 8.1 rev. 160519 (Update-3). Web Forms For Marketers 8.1 rev. 160523 (Update-3)*

**Update/Warning:**
See Wilfred's comment regarding the encryption of 'data' column.

A client asked us to encrypt any data stored by WFFM in both mongo and SQL databases. Sitecorejunkie had posted a [solution](https://sitecorejunkie.com/2013/06/21/encrypt-web-forms-for-marketers-fields-in-sitecore/) that works in sitecore 7 but some things changed in Sitecore 8 and the solution does not work anymore.

The code provided here will encrypt values in `FormData` and `FormFieldValues` tables. Also, will decrypt values when form report is exported to csv or xml as we need the real values.

**The following solution is divided in 3 parts:**
* An encryption/decryption helper using [.NET MachineKey Class (v4.0+)](https://msdn.microsoft.com/en-us/library/system.web.security.machinekey.aspx). NB: I'm not an encryption/security expert so you should research and decide which algorithm/method suits you best.
* Encryption of data being stored in Analytics Mongo DB.
* Encryption of data being stored in Reporting SQL DB.
* Decryption of data when generating reports.

## Encryption helper
[.NET MachineKey Class (v4.0+)](https://msdn.microsoft.com/en-us/library/system.web.security.machinekey.aspx) uses [machine key](https://msdn.microsoft.com/en-us/library/ff649308.aspx) to encrypt/decrypt data. I'd recommend [setting the machine key in web.config](http://stackoverflow.com/questions/3855666/adding-machinekey-to-web-config-on-web-farm-sites) as you need this to be the same in all servers on multi-server solutions. Make sure you [generate](http://www.developerfusion.com/tools/generatemachinekey) a random machine key instead of copying one from online.


``` csharp
using System;
using System.Text;
using System.Web;
using System.Web.Security;

namespace Project.Helpers
{
    public class EncryptionHelper
    {
        private static readonly UTF8Encoding Encoder = new UTF8Encoding();

        public static string Encrypt(string unencrypted)
        {
            if (string.IsNullOrEmpty(unencrypted))
                return string.Empty;
            try
            {
                var encryptedBytes = MachineKey.Protect(Encoder.GetBytes(unencrypted));
                if (encryptedBytes != null && encryptedBytes.Length > 0)
                    return HttpServerUtility.UrlTokenEncode(encryptedBytes);
            }
            catch (Exception)
            {
                return string.Empty;
            }
            return string.Empty;
        }

        public static string Decrypt(string encrypted)
        {
            if (string.IsNullOrEmpty(encrypted))
                return string.Empty;
            try
            {
                var bytes = HttpServerUtility.UrlTokenDecode(encrypted);
                if (bytes != null && bytes.Length > 0)
                {
                    var decryptedBytes = MachineKey.Unprotect(bytes);
                    if(decryptedBytes != null && decryptedBytes.Length > 0)
                        return Encoder.GetString(decryptedBytes);
                }
            }
            catch (Exception)
            {
                return string.Empty;
            }
            return string.Empty;
        }
    }
}
```

## Mongo DB
WFFM stores data in `FormData` table when the user submits a form and the session is [abandoned](https://briancaos.wordpress.com/2016/03/02/sitecore-xdb-flush-data-to-mongodb/). We need to override methods `InsertFormData` and `GetFormData` in class `AnalyticsFormsDataProvider`.

NB: I also did exactly the same to classes `CombinedFormsDataProvider` and `SqlFormsDataProvider` in case we decide in the future to use SQL for storing data.

``` csharp
﻿using System;
using System.Collections.Generic;
using Sitecore.Diagnostics;
using Sitecore.WFFM.Abstractions.Analytics;
using Sitecore.WFFM.Abstractions.Shared;
using Sitecore.WFFM.Analytics.Dependencies;
using Sitecore.WFFM.Analytics.Providers;

namespace Project.WFFM
{
    public class EncryptedAnalyticsFormsDataProvider : AnalyticsFormsDataProvider
    {
        public EncryptedAnalyticsFormsDataProvider(ReportDataProviderWrapper reportDataProviderWrapper, ILogger logger, IAnalyticsTracker analyticsTracker, ISettings settings)
            : base(reportDataProviderWrapper, logger, analyticsTracker, settings)
        {
        }

        public override void InsertFormData(FormData form)
        {
            Assert.ArgumentNotNull(form, "form");
            Assert.ArgumentNotNull(form.Fields, "form.Fields");
            form.Fields = EncryptFields(form.Fields);
            base.InsertFormData(form);
        }

        public override IEnumerable<FormData> GetFormData(Guid formId)
        {
            Assert.ArgumentNotNull(formId, "formId");
            var entries = base.GetFormData(formId);
            entries = DecryptFields(entries);
            return entries;
        }

        private IEnumerable<FieldData> EncryptFields(IEnumerable<FieldData> fields)
        {
            var encryptFields = fields as FieldData[] ?? fields.ToArray();
            foreach (var field in encryptFields.Where(field => !string.IsNullOrEmpty(field.Value)))
                field.Value = EncryptionUtil.Encrypt(field.Value);
            return encryptFields;
        }

        private IEnumerable<FormData> DecryptFields(IEnumerable<FormData> entries)
        {
            var entriesArray = entries as FormData[] ?? entries.ToArray();
            if (entries == null || !entriesArray.Any())
                return entriesArray;
            foreach (var entry in entriesArray)
            {
                if (entry == null || entry.Fields == null || !entry.Fields.Any())
                    continue;
                foreach (var field in entry.Fields.Where(field => !string.IsNullOrEmpty(field.Value)))
                    field.Value = EncryptionUtil.Decrypt(field.Value);
            }
            return entriesArray;
        }
    }
}
```

`InsertFormData` will be triggered when data is to be stored in mongo, whereas `GetFormData` is triggered when sitecore needs to retrieve the data (e.g. when generating excel report). `EncryptFields` and `DecryptFields` are looping through all fields in the form and encrypting/decrypting their values using our helper.


Then, we need to patch the configuration file so it uses our code instead of default one.

``` xml
<?xml version="1.0" encoding="utf-8" ?>
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
  <sitecore>
    <wffm>
      <analytics>
        <analyticsFormsDataProvider patch:instead="*[@type='Sitecore.WFFM.Analytics.Providers.AnalyticsFormsDataProvider, Sitecore.WFFM.Analytics']" type="Project.WFFM.EncryptedAnalyticsFormsDataProvider, Project">
          <param name="reportDataProviderWrapper" ref="/sitecore/wffm/analytics/reportDataProviderWrapper" />
          <param name="logger" ref="/sitecore/wffm/logger" />
          <param name="analyticsTracker" ref="/sitecore/wffm/analytics/analyticsTracker" />
          <param name="settings" ref="/sitecore/wffm/settings" />
        </analyticsFormsDataProvider>
      </analytics>
    </wffm>
  </sitecore>
</configuration>
```

If you end up overriding `CombinedFormsDataProvider` and `SqlFormsDataProvider` then you will need to similarly patch `Sitecore.WFFM.Analytics.Providers.SqlFormsDataProvider, Sitecore.WFFM.Analytics` and `Sitecore.WFFM.Analytics.Providers.CombinedFormsDataProvider, Sitecore.WFFM.Analytics`

After this, WFFM data in analytics database (mongo) will be encrypted and decrypted when exporting to excel/xml. However, if you take a look at `FormFieldValues` table in SQL reporting database, you will notice the values here aren't. So the next step is to encrypt SQL server data.

## SQL server
WFFM stores data in couple SQL tables once user's session is [abandoned](https://briancaos.wordpress.com/2016/03/02/sitecore-xdb-flush-data-to-mongodb/). The one we need to encrypt is `FormFieldValues`. In order to achieve this, we need to create new pipeline and replace default `FormSummaryProcessor` to encrypt the data before it gets saved. I reflectored the class `Sitecore.WFFM.Analytics.Aggregation.Processors.FormSummary.FormSummaryProcessor`, copied the code and modified the value (line 52) we need to encrypt. If you are using a different version than the one specified at the beggining of this post, I'd suggest you reflector your version's dll and amend the code where needed.

NB: Sitecore support suggested we also modify the pipelines below but I couldn't see a reason to as there's no data we needed to encrypt on these tables.
`Sitecore.WFFM.Analytics.Aggregation.Processors.FormEvents.FormEventsProcessor, Sitecore.WFFM.Analytics`
`Sitecore.WFFM.Analytics.Aggregation.Processors.FormStatisticsByContact.FormStatisticsByContactProcessor, Sitecore.WFFM.Analytics`

Here's the full implementation:

``` csharp
using System.Collections.Generic;
using System.Linq;
using Global.Helpers;
using Sitecore.Analytics.Aggregation.Pipeline;
using Sitecore.Analytics.Model;
using Sitecore.Diagnostics;
using Sitecore.WFFM.Abstractions.Analytics;
using Sitecore.WFFM.Analytics.Aggregation.Processors.FormFieldValues;
using Sitecore.WFFM.Analytics.Aggregation.Processors.FormSummary;

namespace Project.WFFM
{
    public class EncryptedFormSummaryProcessor : AggregationProcessor
    {
        protected override void OnProcess(AggregationPipelineArgs args)
        {
            Assert.ArgumentNotNull(args, "args");
            VisitData visit = args.Context.Visit;
            if (visit.Pages == null)
            {
                return;
            }
            foreach (PageData current in visit.Pages)
            {
                if (current.PageEvents != null)
                {
                    FormSummary formSummary = null;
                    foreach (PageEventData current2 in current.PageEvents)
                    {
                        if (!(current2.PageEventDefinitionId != IDs.FormSubmitSuccessEventId) && current2.CustomValues.ContainsKey(Sitecore.WFFM.Abstractions.Analytics.Constants.WffmKey))
                        {
                            List<Sitecore.WFFM.Abstractions.Analytics.FieldData> list = current2.CustomValues[Sitecore.WFFM.Abstractions.Analytics.Constants.WffmKey] as List<Sitecore.WFFM.Abstractions.Analytics.FieldData>;
                            if (list == null)
                            {
                                IEnumerable<Sitecore.WFFM.Analytics.Model.FieldData> enumerable = current2.CustomValues[Sitecore.WFFM.Abstractions.Analytics.Constants.WffmKey] as List<Sitecore.WFFM.Analytics.Model.FieldData>;
                                if (enumerable == null)
                                {
                                    continue;
                                }
                                list = new List<Sitecore.WFFM.Abstractions.Analytics.FieldData>(enumerable);
                            }
                            foreach (Sitecore.WFFM.Abstractions.Analytics.FieldData current3 in list)
                            {
                                IEnumerable<string> enumerable2 = (current3.Values != null && current3.Values.Count > 0) ? current3.Values : ((!string.IsNullOrEmpty(current3.Value)) ? Enumerable.Repeat<string>(current3.Value, 1) : Enumerable.Repeat<string>(string.Empty, 1));
                                foreach (string current4 in enumerable2)
                                {
                                    FormSummaryKey key = new FormSummaryKey
                                    {
                                        FormId = current2.ItemId,
                                        FieldId = current3.FieldId,
                                        FieldName = current3.FieldName,
                                        FieldValueId = args.GetDimension<FormFieldValues>().Add(EncryptionUtil.Encrypt(current4))
                                    };
                                    FormSummaryValue value = new FormSummaryValue
                                    {
                                        Count = 1
                                    };
                                    if (formSummary == null)
                                    {
                                        formSummary = args.GetFact<FormSummary>();
                                    }
                                    formSummary.Emit(key, value);
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
```

**FieldValueId = args.GetDimension<FormFieldValues>().Add(EncryptionUtil.Encrypt(current4))** is what was modified using our encryption helper.


Then, we need to patch the pipeline so it uses our new pipeline instead of default one.

``` xml
﻿<?xml version="1.0" encoding="utf-8" ?>
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
  <sitecore>
    <pipelines>
      <group groupName="analytics.aggregation">
        <pipelines>
          <interactions>
            <processor patch:instead="*[@type='Sitecore.WFFM.Analytics.Aggregation.Processors.FormSummary.FormSummaryProcessor, Sitecore.WFFM.Analytics']" type="Project.WFFM.EncryptedFormSummaryProcessor, Project" />
          </interactions>
        </pipelines>
      </group>
    </pipelines>
   </sitecore>
</configuration>
```

---

Please let me know what you think and/or if you can spot any errors.
*/eom*
