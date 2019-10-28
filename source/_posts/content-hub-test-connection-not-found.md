title: 404 error when clicking Test Connection on Actions
date: 2019/10/28
thumbnail: /images/test-connection-404.png
categories:
- Sitecore
- Content Hub
tags:
- stylelabs
- action
- apicall

---
<img class="hero-img" src="/images/headless-police-woman.jpg" alt="Headless Police Woman">
---
Why is Test Connection button on an Action returning a not found error when clicked.
<!-- more -->

*Versions used: Content Hub 3.2*

You want to invoke an API from Content Hub and have created an `Action` that points to it. `API URL` is fine, `Method` is correct, `Headers` aren't expected so is left empty. You then click on `Test Connection` and get `Connection failed` error.

Why?!

Regardless of the selected method, and expected by the API when called, __`Test Connection` button sends a `HEAD` request to the API__. Set your API endpoint to allow [HEAD](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/HEAD) requests and you are all set.

---

Please let me know what you think and/or if you can spot any errors.
*/eom*
