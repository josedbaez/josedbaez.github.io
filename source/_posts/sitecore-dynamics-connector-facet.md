title: Add custom facet to sync with Dynamics CRM
date: 2018/07/30
categories:
- Sitecore
- CRM
tags:
- dynamics
- xdb

---
<img class="hero-img" src="/images/sitecore-dynamics.jpg" alt="Sitecore Microsoft Dynamics CRM">
---
How to define and implement a custom facet in sitecore so it can be imported when syncing CRM contacts.
<!-- more -->

*Versions used: Sitecore 8.2 rev. 161221 (8.2 Update-2). Dynamics CRM Connect 1.3*

When you configure the dynamics connector it creates predefined facets and its mappings. You need to define custom ones if you want them to be synced. I was meant to blog about this a while ago but for some reason didn't. Someone asked me about it on a [previous post](http://josedbaez.com/2018/02/sitecore-dynamics-connector/) where I explain how to configure the connector.

There are 3 things you have to do to add a custom facet:
* Create facet interface and implementation.
* Add custom facet to sitecore entities and model.
* Create facet accessor items and map them.

The example below is from an implementation where user interests (e.g. sports, computers, music) needed to be synced. These were being saved individually on the CRM as boolean -e.g. a contact would have Sports = true, Computers = false, Music = true, etc, and stored on xDB as one facet with multiple properties.

## Facet interface and implementation ##
``` csharp
using Sitecore.Analytics.Model.Framework;
using Sitecore.Data;

namespace Entities
{
    public interface IMyCrmFacet : IFacet, IElement, IValidatable
    {
        bool Computers { get; set; }
        bool Sports { get; set; }
    }
}
```
``` csharp
using Sitecore.Analytics.Model.Framework;
using Sitecore.Data;

namespace Entities
{
    [Serializable]
    public class MyCrmFacet : Facet, IMyCrmFacet, IFacet, IElement, IValidatable
    {
        private const string COMPUTERS = "Computers";
        private const string SPORTS = "Sports";

        public bool Computers
        {
            get
            {
                return base.GetAttribute<bool>(COMPUTERS);
            }
            set
            {
                base.SetAttribute<bool>(COMPUTERS, value);
            }
        }

        public bool Sports
        {
            get
            {
                return base.GetAttribute<bool>(SPORTS);
            }
            set
            {
                base.SetAttribute<bool>(SPORTS, value);
            }
        }

        public CrmInterests()
        {
            base.EnsureAttribute<bool>(COMPUTERS);
            base.EnsureAttribute<bool>(SPORTS);
        }
    }
}
```
The code above is quite simple. All we are doing is creating getters and setters for each facet properties. The name of the property will be used in sitecore to find the code, and the constant value is name it will store in mongo.

## Add custom facet to sitecore entities and model ##
Now we need to let sitecore know about this new facet and its implementation. We do this via a config patch as follows.
``` xml
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
  <sitecore>
    <model>
      <elements>
        <element interface="Entities.IMyCrmFacet, MyLibrary" implementation="Entities.MyCrmFacet, MyLibrary"/>
      </elements>
      <entities>
        <contact>
          <facets>
            <facet name="MyCrmFacet" contract="Entities.IMyCrmFacet, MyLibrary" />
          </facets>
        </contact>
      </entities>
    </model>
  </sitecore>
</configuration>
```

## Create facet accessor items and map them ##
Now that we have created our facet and have registered it in sitecore, all that is left is create accessor items to tell sitecore which CRM properties should map to our facet.

To do this, you have to create an accessor for the CRM attribute, an accessor for the xDB contact and a mapping set item to link the 2.

a) Create an item of type `Entity Attribute Value Accessor` under: 
`/sitecore/system/Data Exchange/xxxx/Data Access/Value Accessor Sets/Providers/Dynamics CRM/CRM Contact Attributes/` and set `AttributeName` to the field name of the property in CRM. This might not be the same as the label users see on the CRM form, so make sure you set the right one (e.g. in our case the form would should Computers but the name was ms_computers. Was able to get that by inspecting form html).

b) Create an item of type `xDB Contact Facet Property Value Accessor` under: `/sitecore/system/Data Exchange/xxxx/Data Access/Value Accessor Sets/Providers/Sitecore/xDB Contact Properties/` and set `ContactFacetName` to one property in your facet class (e.g. Computers)

c) Create an item of type `Value Mapping` under `/sitecore/system/Data Exchange/xxxx/Value Mapping Sets/CRM Contact to xDB Contact/`
and set `SourceAccessor` to the CRM accessor created on a), and `TargetAccessor` to the xDB accessor created on b).

Done. Your custom properties should now sync when you run the task. 

PS: You need to the steps above for each property that you want to map.

---

Please let me know what you think and/or if you can spot any errors.
*/eom*
