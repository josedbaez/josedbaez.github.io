title: Log in to Sitecore using OpenID Connect with Okta
date: 2017/09/30
categories:
- Sitecore
- Authentication
tags:
- openid
- okta
- sso
- authentication

---
<img class="hero-img" src="/images/okta-sso.png" alt="Sitecore Okta SSO">
---
How to implement OpenID Connect Single Sign-On with Okta to log in to sitecore (backend NOT client facing site) by intercepting Authorize attribute.
<!-- more -->

*Versions used: Sitecore 8.2 rev. 170614 (8.2 Update-4).*

In this post I will outline how to implement OpenID SSO with [Okta](https://www.okta.com/) to allow users to **log in to sitecore (backend NOT client facing site)** from Okta's dashboard or by being redirected to Okta's login screen when trying to directly access sitecore through custom url <sup>[custom url](#custom-url)</sup>. *Authentication logic has been copied/modified from [Okta's github example code.](https://github.com/oktadeveloper/okta-oauth-aspnet-codeflow)*

The solution provided by OKTA uses [OWIN](http://owin.org/) libraries. I integrated the OWIN middleware through a sitecore pipeline following [VyacheslavPritykin Sitecore-Owin](https://github.com/VyacheslavPritykin/Sitecore-Owin) solution.

** The code flow of this solution: **
The user requests a method decorated with `Authorize`<sup>[Authorize](#all-authorize)</sup> attribute, if the user is not authenticated, the middleware will get triggered and the user is redirected to Okta. Once the user logs in to okta, the middleware gets notified, we extract the information returned by okta and log the user in as a virtual user with `AuthenticationManager`. The original request (to method decorated with Authorize) is sent again and the user now goes through because it has been authenticated.

**The solution is divided in the following parts:**
* OKTA application configuration
* Sitecore pipeline to register OWIN middleware
* OWIN middleware to handle authentication redirects
* Sitecore login helper
* Login controller

## OKTA application configuration ##
You'll need to [Sign up for Okta](https://developer.okta.com/docs/api/getting_started/api_test_client.html) and get access to the API.

Once you have access to okta; within the Admin part of your account, add a new application with the following selections: Web as platform and OpenID Connect as Sign on method.

Go to your newly created application and configure as follows:

__General tab__
<img class="content-img" src="/images/okta-general-tab.jpg" alt="Okta general tab">

Most of the configuration is self explanatory. `Login redirect URIs` is defined by Okta as "URI where Okta will send OAuth responses". We don't really use it but it's required because a request to it will be sent once the user has logged in to Okta. `Initiate login URI` is another request we will get from Okta once the user has logged in or when the user tries to access sitecore from Okta's dashboard. This is where we will redirect the user to sitecore.

Copy <a name="client-id">`Client ID`</a> and <a name="client-secret">`Client secret`<a> as we are going to need these later.

__Sign On tab__
<img class="content-img" src="/images/okta-sign-on-tab.jpg" alt="Okta sign-on tab">
<img class="content-img" src="/images/okta-sign-on-policy.png" alt="Okta sign-on policy">

On this tab you can configure access policy and the OpenID token. We are telling okta to return all groups but here you can configure the groups claim to specify which groups you want included by entering expressions that will be used to filter groups. For example myapp.* means all groups prefixed with "myapp" will be included in the response attribute statement. [Go here](https://support.okta.com/help/Documentation/Knowledge_Article/Configuring-Okta-Template-SAML-20-application) for more information.

__Assignments__
Here you assign which users and or groups can access your app. I recommend using groups instead of users.

Okta groups are like sitecore roles; we will need to map these groups to equivalent sitecore roles to give them proper sitecore permissions. To make the mapping easier, we decided to name Okta groups the same as the Sitecore role it will be mapped to.

## Sitecore OWIN middleware pipeline ##
You can either install [VyacheslavPritykin Sitecore-Owin](https://github.com/VyacheslavPritykin/Sitecore-Owin) nuget package or do the following.

- Define a custom pipeline in sitecore
``` xml
<sitecore>
  <pipelines>
    <initOwinMiddleware>
      <processor mode="on" type="MySite.OwinMiddleware, MySite"></processor>
    </initOwinMiddleware>
  </pipelines>
</sitecore>
```

- The processor requires a custom `PipelineArgs` that sets the Owin app object on startup.
``` csharp
using Sitecore.Pipelines;
using IAppBuilder = Owin.IAppBuilder;

namespace MySite
{
    public class InitOwinMiddlewareMiddlewareArgs : PipelineArgs
    {
        public Owin.IAppBuilder App { get; set; }

        public InitializeOwinOktaMiddlewareArgs(IAppBuilder app)
        {
            this.App = app;
        }
    }
}
```

- The `initOwinMiddleware` pipeline is called on startup by setting the `owin:AppStartup` class reference in our web.config.
The following transform:
    - Adds settings `owin:AutomaticAppStartup` and `owin:AppStartup`.
    - Sets authentication to none. This is done to avoid an infinite loop from okta to sitecore.

``` xml
<appSettings xdt:Transform="InsertIfMissing">
  <add key="owin:AutomaticAppStartup" value="true" xdt:Transform="InsertIfMissing" xdt:Locator="Match(key)" />
  <add key="owin:AppStartup" value="MySite.Startup, MySite" xdt:Transform="InsertIfMissing" xdt:Locator="Match(key)" />
</appSettings>

<system.web>
  <authentication mode="None" xdt:Transform="SetAttributes">
  </authentication>
</system.web>
```

Startup class implementation.

``` csharp
using Microsoft.Owin;
using Owin;
using Sitecore.Pipelines;

[assembly: OwinStartup(typeof(Startup))]
namespace MySite
{
    public class Startup
    {
        public virtual void Configuration(IAppBuilder app)
        {
            CorePipeline.Run("initOwinMiddleware", new InitOwinMiddlewareMiddlewareArgs(app));
        }
    }
}
```

## OWIN middleware and Okta redirect implementation ##
The OWIN middleware pipeline handles the authentication configuration of the web application. It tells asp.net where to redirect the user and what to do when the authorisation is given to the user.

As stated before, this logic was extracted from [Okta's codeflow example code.](https://github.com/oktadeveloper/okta-oauth-aspnet-codeflow) and modified so it can work with latest OWIN and Claims libraries <sup>[versions used](#versions)</sup>.

``` csharp
using System;
using System.Globalization;
using System.Linq;
using System.Security.Claims;
using IdentityModel.Client;
using Microsoft.Owin.Security;
using Microsoft.Owin.Security.Cookies;
using Microsoft.Owin.Security.OpenIdConnect;
using Owin;
namespace MySite
{
    public class OwinMiddleware
    {
        public void Process(InitOwinMiddlewareMiddlewareArgs args)
        {
            var oathAuthority = "https://dev-666666.oktapreview.com"; //change this
            args.App.UseCookieAuthentication(new CookieAuthenticationOptions {AuthenticationType = "Cookies"});
            args.App.UseOpenIdConnectAuthentication(new OpenIdConnectAuthenticationOptions
            {
                ClientId = "myclientid123", //change this
                Authority = oathAuthority,
                RedirectUri = "https://Website_URL/Okta/Callback", //change this
                ResponseType = "code id_token", //
                Scope = "openid email profile address phone groups offline_access", //claims to get from okta
                SignInAsAuthenticationType = "Cookies",
                UseTokenLifetime = true,

                Notifications = new OpenIdConnectAuthenticationNotifications
                {
                    AuthorizationCodeReceived = async n =>
                    {
                        var tokenClient = new TokenClient(oathAuthority + "/oauth2/v1/token", "myclientid123", "myclientsecret", AuthenticationStyle.BasicAuthentication);

                        var tokenResponse = await tokenClient.RequestAuthorizationCodeAsync(n.Code, n.RedirectUri);
                        if (tokenResponse.IsError)
                            throw new Exception(tokenResponse.Error);

                        var userInfoClient = new UserInfoClient(oathAuthority + Constants.UserInfoEndpoint);
                        var userInfoResponse = await userInfoClient.GetAsync(tokenResponse.AccessToken);

                        var id = new ClaimsIdentity(userInfoResponse.Claims, n.AuthenticationTicket.Identity.AuthenticationType);
                        var idClaims = userInfoResponse.Claims;

                        id.AddClaim(new Claim("id_token", n.ProtocolMessage.IdToken));
                        id.AddClaim(new Claim("access_token", tokenResponse.AccessToken));
                        if (tokenResponse.RefreshToken != null)
                            id.AddClaim(new Claim("refresh_token", tokenResponse.RefreshToken));

                        id.AddClaim(new Claim("expires_at", DateTime.Now.AddSeconds(tokenResponse.ExpiresIn).ToLocalTime().ToString(CultureInfo.InvariantCulture)));
                        if (tokenResponse.RefreshToken != null)
                            id.AddClaim(new Claim("refresh_token", tokenResponse.RefreshToken));

                        var nameClaim = new Claim(ClaimTypes.Name, idClaims.FirstOrDefault(c => c.Type == "name")?.Value);
                        id.AddClaim(nameClaim);

                        n.AuthenticationTicket = new AuthenticationTicket(new ClaimsIdentity(id.Claims, n.AuthenticationTicket.Identity.AuthenticationType),
                            n.AuthenticationTicket.Properties);

                        //log user in to sitecore
                        var princ = new ClaimsPrincipal(new[] { id });
                        MySite.Helpers.LogUser(princ);
                    },
                }
            });
        }
    }
}
```
See below for `MySite.Helpers.LogUser(princ)` implementation.

## Sitecore login helper ##
This helper extracts the required information from Okta's token response, logs in the user as a [virtual user](https://briancaos.wordpress.com/2015/11/13/sitecore-virtual-users-authenticate-users-from-external-systems/) and maps okta groups to sitecore roles.
``` csharp
using System;
using System.Linq;
using System.Security.Claims;
using System.Security.Principal;
using Microsoft.AspNet.Identity;
using Sitecore;
using Sitecore.Diagnostics;
using Sitecore.Security.Accounts;
using Sitecore.Security.Authentication;
namespace MySite
{
    public class Helpers
    {
        public void LogUser(IPrincipal principal)
        {
            var identity = principal.Identity as ClaimsIdentity;
            if (identity == null || !identity.IsAuthenticated)
                return;

            var email = identity.FindFirstValue("email");
            var userName = identity.FindFirstValue("preferred_username");
            var sitecoreUsername = string.Format("{0}\\{1}", "sitecore", userName);

            var user = AuthenticationManager.BuildVirtualUser(sitecoreUsername, true);
            user.Profile.Name = identity.FindFirstValue("given_name");
            user.Profile.Email = email;
            user.Profile.FullName = identity.FindFirstValue("given_name");

            AddRoles(identity, user);
            user.Profile.Save();
            AuthenticationManager.LoginVirtualUser(user);
        }

        private void AddRoles(ClaimsIdentity claimsIdentity, User user)
        {
            var groups = claimsIdentity.FindAll("groups");
            if (groups == null || !groups.Any())
                return;

            foreach (var group in groups)
            {
                if(group.Value.ToLowerInvariant().Equals("everyone"))
                    continue;
                var role = string.Format("{0}\\{1}", "mydomain", role.Value);
                if (Role.Exists(role))
                    user.Roles.Add(Role.FromName(role));
            }
        }
    }
}
```

## Login controller ##
All we have left to do now is create a method with the `[Authorize]` attribute on it, and redirect users to this route so OWIN middleware can take over.

``` csharp
public class OktaController : Controller
{
    public ActionResult Callback()
    {
        return Redirect("/sitecore");
    }

    [Authorize]
    public ActionResult Login()
    {
        var owinContext = System.Web.HttpContext.Current.Request.GetOwinContext();
        if (owinContext?.Authentication.User != null)
            return Redirect("/sitecore");
        return Redirect("/");
    }
}
```

Do not forget to register the controller's route in route table.

====================
<a name="all-authorize">All authorize decorated methods?</a>We didn't need membership for end users so we could afford to intercept any `Authorize` attribute. You can configure a separate middleware for end users. See [Bas Lijten's federated solution](https://github.com/BasLijten/SitecoreFederatedLogin)

<a name="custom-url">Why a custom url?</a> We opted to use a custom URL because we wanted to be able to log-in to sitecore using the conventional method as well.

<a name="versions">Versions used:</a> I had lot of issues figuring out dll versions, so here's the list of the versions used on this implementation.
Newtonsoft.Json 10.0.0.0
Microsoft.IdentityModel.Protocol.Extensions 1.0.40306.1554
System.IdentityModel.Tokens.Jwt 4.0.40306.1554
Microsoft.Owin.Security 3.1.0.0
Microsoft.Owin.Security.Cookies 3.1.0.0
Microsoft.Owin 3.1.0.0

See [this](http://josedbaez.com/2017/08/binding-redirect-patch/) if you are having binding redirects issues.

---

Please let me know what you think and/or if you can spot any errors.
*/eom*
