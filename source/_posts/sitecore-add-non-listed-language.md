title: Add a non-listed language to Sitecore
thumbnail: /images/greetings-sitecore_th.jpg
date: 2019/04/13
categories:
- Sitecore
- Globalization
tags:
- sitecore
- globalization
- multilingual

---
<img class="hero-img" src="/images/greetings-sitecore.jpg" alt="Greetings">
---
How to add Sitecore languages without patching LanguageDefinitions.config.
<!-- more -->

*Versions used: Sitecore Experience Platform 9.0 rev. 180604 (Update-2).*

Sitecore supports any language that is registered in the operating system where it runs. See my [previous post](/2019/04/supported-cultures) to see how to render a list of available languages on a server. When adding a new language in sitecore, the modal dialog loads defined languages (`\App_Config\LanguageDefinitions.config`) into the dropdown. When you select a language from the dropdown, the properties of that language definition are set to the appropriate inputs.

When you click next, sitecore will verify 2 things: 
1- If the language already exists -gets language items from cache or by querying Database by language template ID. 
2- The provided Language code combination (Language  + "-" + Region + "-" CustomCode) is a valid culture (using `System.Globalization.CultureInfo.GetCultureInfo`).

The selected option on the dropdownn is not used for the validation, hence it is not required (I'm not debating best practice here) to define new languages inside `LanguageDefinitions.config`. You can select the blank option; as long as you enter a valid culture (existing in the operating system), sitecore will create the language item. If the culture is invalid, you will get an error like `The name "xxxxxx" is not a valid or supported culture identifier`.

Sometimes it's confusing to figure out the language code combination that needs to be entered on this modal or in `LanguageDefinitions.config`. From the table on my [previous post](/2019/04/supported-cultures), the last 2 columns (`Language` and `Country/region code`) go in the inputs with the same names. 
e.g. for `Bosnian (Latin, Bosnia and Herzegovina)`, you'd enter `bs-Latn` in Language and `BA` in Country/region code.

As for the next step on the creation modal, I have not found a reliable source listing `Codepage` and `Charset` for all languages out there, so do your own research. If you do have one, please let me!. 

---

Please let me know what you think and/or if you can spot any errors.
*/eom*
