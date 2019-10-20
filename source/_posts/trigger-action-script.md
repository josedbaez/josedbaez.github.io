title: Content Hub Triggers, Actions and Scripts
date: 2019/08/18
thumbnail: /images/trigger-action-script_th.png
categories:
- Sitecore
- Content Hub
tags:
- stylelabs
- trigger
- action
- script

---
<img class="hero-img" src="/images/trigger-action-script.jpg" alt="Content Hub Trigger, Action, Script">
---
What are Content Hub Triggers, Actions and Scripts, and what are they used for?
<!-- more -->

In Content Hub, triggers, actions and scripts are usually used together: a trigger invokes an action which executes a script<sup> An action can execute more things than just scripts</sup>.

## Triggers: ##
> A trigger is a set of actions that are automatically executed after specific events and under specific conditions. <sup>[1](#triggers)</sup>

Triggers are configured in a similar way to Sitecore Rules Engine. You specify the event it responds to (`Objective`), the conditions that specify if the trigger should run, and the actions that will run if the conditions are met.

Triggers can be configured using the UI and are found under __Admin__ -> __Triggers__.

They can be executed `In process` or `In background`. __In process__ triggers are executed synchronously and can target more execution phases (e.g. `Validation`), whereas __In background__ triggers are executed asynchronously and only support `Post Execution` phase. As a rule of thumb, `In Process` should not be chosen for heavy operations that are time-consuming.

When creating a `Trigger Condition`, you first select on which `Definition` the condition will be assessed on (e.g. `M.Asset`). Then you set the Condition by selecting the property/relation of the selected Definition and the value you are targeting. For example, a condition could be: When the `Current Value` of `Title` property of `M.Asset` definition `equals` to `Hola`.

<img class="boxed" src="/images/trigger-condition.png" style="max-width:400px" alt="Trigger Condition Configuration">


## Actions: ##
> In Sitecore Content Hub, Actions are components that perform a specific task. <sup>[2](#actions)</sup>

As of Content Hub 3.2, an Action can be created as any of these types: `Action script`, `API call`, `Azure event hub`, `Azure service bus`, `M Azure service bus`, `Print entity generation`,  `Reporting channel`, and `Start state machine`. Actions can be configured using the UI and are found under __Admin__ -> __Actions__.

Most of these are configured by entering an endpoint URL (e.g. API Call) or a connection string (e.g. Azure event hub). Others execute tasks that are specific to Content Hub and has been pre-configured (e.g. Reporting Channel). `Action script` is an action that will execute a Content Hub script of type action -see scripts section below. 

Actions usually send a payload in the request body (as json) or headers. This usually includes a `TargetId` specifying the entity being processed.  

__Note:__ Some action types require a response (e.g. API call). Not returning a response before the request times out might cause the action to send multiple requests. A queue type should be considered for heavy processes.

<img class="boxed" src="/images/action-api-call.png" style="max-width:400px" alt="Action API Call Configuration">


## Scripts: ##
> Sitecore Content Hub allows users to integrate custom scripts in their business logic. Scripts can be manually triggered by end-users or automatically triggered by the application (e.g. using Triggers) depending on the script type and the use case. Sitecore Content Hub offers different script types, with each type having different context properties. <sup>[3](#scripts)</sup>

A script is a piece of C# code that executes when invoked (via URL or trigger) and compiled in Content Hub. They can be configured using the UI and are found under Admin->Scripts.

As of Content Hub 3.2, a Script can be created as any of these types: `Action`, `Metadata processing`, `User pre-registration`, `User post-registration` and `User sign-in`.

The type of script dictates the context where it's run and the properties available. The most commonly used is Action.

When invoked by triggers, action scripts can be executed synchronously (`restricted`) and asynchronously (`unrestricted`). Restricted scripts have a limited range of libraries than can be referenced/used <sup>[4](#libraries-restriction)</sup>. 
 
<img class="boxed" src="/images/script-action.png" style="max-width:600px" alt="Action Script">


====================

References:
[1] <a name="triggers" href="https://docs.stylelabs.com/content/integrations/intergration-components/triggers/overview.html"> https://docs.stylelabs.com/content/integrations/intergration-components/triggers/overview.html </a>
[2] <a name= "scripts" href="https://docs.stylelabs.com/content/integrations/scripting-api/scripting-api-overview.html"> https://docs.stylelabs.com/content/integrations/scripting-api/scripting-api-overview.html </a>
[3] <a name= "actions" href="https://docs.stylelabs.com/content/integrations/intergration-components/actions/overview.html"> https://docs.stylelabs.com/content/integrations/intergration-components/actions/overview.html </a>
[4] <a name="libraries-restriction" href="https://docs.stylelabs.com/content/integrations/scripting-api/restricted-scripts.html"> https://docs.stylelabs.com/content/integrations/scripting-api/restricted-scripts.html </a>


---

Please let me know what you think and/or if you can spot any errors.
*/eom*
