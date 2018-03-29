title: Log in to sitecore 9 editor using OKTA provider
date: 2018/04/04
categories:
- Sitecore
- Authentication
tags:
- openid
- facebook
- google
- sso
- authentication

---
<img class="hero-img" src="/images/sitecore9-sso.jpg" alt="Sitecore 9 SSO">
---
How to implement federated authentication on sitecore 9 to allow content editors log in to sitecore using their okta accounts.
<!-- more -->

*Versions used: Sitecore Experience Platform 9.0 rev. 171219 (9.0 Update-1).*

This post is a continuation of my [previous post](http://josedbaez.com/2018/03/sitecore9-sso/). Instead of logging visitors in using federated authentication, in this post I'll show how to implement Okta authentication to log in to sitecore editor. It's an udpated version of my [sitecore 8 okta post](http://josedbaez.com/2017/09/sitecore-okta-login/). [Source code here.](https://github.com/josedbaez/sitecore9sso)

The good news is things have changed and the implementation is easier and shorter than before. I won't be going into details on how to configure the application in Okta because it hasn't changed much since last time. [Click here](http://josedbaez.com/2017/09/sitecore-okta-login/#OKTA-application-configuration) for previous configuration. 

Please see my [previous post](http://josedbaez.com/2018/03/sitecore9-sso/) or [download here](https://github.com/josedbaez/sitecore9sso/blob/master/App_Config/Include/SSO/SitecoreSSO.config) to enable required settings and add default services.

## Okta middleware/provider implementation ##
The following code is based on the [sample provided](https://github.com/oktadeveloper/okta-aspnet-mvc-example/blob/master/OktaAspNetExample/Startup.cs) by Okta.

``` csharp
public class OktaIdentityProvider : IdentityProvidersProcessor
{
    protected override string IdentityProviderName => "Okta";
    private const string ClientId = "your clientid here";
    private const string ClientSecret = "your ClientSecret here";
    private const string Authority = "your okta site URL here";
    private const string OauthTokenEndpoint = "/oauth2/v1/token";
    private const string OauthUserInfoEndpoint = "/oauth2/v1/userinfo";
    private const string OAuthRedirectUri = "http://sc9xp0.sc/identity/externallogincallback";
    private const string OpenIdScope = OpenIdConnectScope.OpenIdProfile + " email";

    protected IdentityProvider IdentityProvider { get; set; }

    public OktaIdentityProvider(FederatedAuthenticationConfiguration federatedAuthenticationConfiguration)
        : base(federatedAuthenticationConfiguration)
    {
    }

    protected override void ProcessCore(IdentityProvidersArgs args)
    {
        Assert.ArgumentNotNull(args, "args");
        IdentityProvider = this.GetIdentityProvider();
        var options = new OpenIdConnectAuthenticationOptions
        {
            ClientId = ClientId,
            ClientSecret = ClientSecret,
            Authority = Authority,
            RedirectUri = OAuthRedirectUri,
            ResponseType = OpenIdConnectResponseType.CodeIdToken,
            Scope = OpenIdScope,
            AuthenticationType = IdentityProvider.Name,
            TokenValidationParameters = new TokenValidationParameters
            {
                NameClaimType = "name"
            },
            Notifications = new OpenIdConnectAuthenticationNotifications
            {
                AuthorizationCodeReceived = ProcessAuthorizationCodeReceived,
                RedirectToIdentityProvider = n =>
                {
                    // If signing out, add the id_token_hint
                    if (n.ProtocolMessage.RequestType == Microsoft.IdentityModel.Protocols.OpenIdConnectRequestType.LogoutRequest )
                    {
                        var idTokenClaim = n.OwinContext.Authentication.User.FindFirst("id_token");
                        if (idTokenClaim != null)
                            n.ProtocolMessage.IdTokenHint = idTokenClaim.Value;
                    }
                    return Task.CompletedTask;
                }
            }
        };
        args.App.UseOpenIdConnectAuthentication(options);
    }

    private async Task ProcessAuthorizationCodeReceived(AuthorizationCodeReceivedNotification notification)
    {
        // Exchange code for access and ID tokens
        var tokenClient = new TokenClient(Authority + OauthTokenEndpoint, ClientId, ClientSecret);
        var tokenResponse = await tokenClient.RequestAuthorizationCodeAsync(notification.Code, notification.RedirectUri);
        if (tokenResponse.IsError)
            throw new Exception(tokenResponse.Error);

        var userInfoClient = new UserInfoClient(Authority + OauthUserInfoEndpoint);
        var userInfoResponse = await userInfoClient.GetAsync(tokenResponse.AccessToken);
        var claims = new List<Claim>();
        claims.AddRange(userInfoResponse.Claims);
        claims.Add(new Claim("id_token", tokenResponse.IdentityToken));
        claims.Add(new Claim("access_token", tokenResponse.AccessToken));

        if (!string.IsNullOrEmpty(tokenResponse.RefreshToken))
            claims.Add(new Claim("refresh_token", tokenResponse.RefreshToken));

        notification.AuthenticationTicket.Identity.AddClaims(claims);
        notification.AuthenticationTicket.Identity.ApplyClaimsTransformations(new TransformationContext(this.FederatedAuthenticationConfiguration, IdentityProvider));
    }
}
```
We first create our `OpenIdConnectAuthenticationOptions` object and set the settings required by the middleware to know who to communicate to and Okta's settings so it allows requests from our application.

`AuthenticationType` is very important because, after authentication is triggered, a 401 status code will be set to the response and each owin middleware will inspect and decide if it should action the request by matching its `AuthenticationType`.

Remember `Scope` defines the claims we are requesting from Okta. We are setting it to the enum `OpenIdConnectScope.OpenIdProfile` plus `email`. Sitecore requires email by default. See [OpenId specification](http://openid.net/specs/openid-connect-core-1_0.html#ScopeClaims) for more info on scope values. `AuthenticationType` determines the authorization processing flow to be used. See [OpenId specification](http://openid.net/specs/openid-connect-core-1_0.html#HybridAuthRequest) for more info on scope authentication request.

`/identity/externallogincallback` is the callback URL sitecore creates to process external logins after they have been authenticated on the providers.

Our AuthorizationCodeReceived task `ProcessAuthorizationCodeReceived` is making sure the token is valid and retrieving claims to be added to the context. It uses sitecore's `ApplyClaimsTransformations` helper to map claims according to configuration set in config file.


## Okta provider configuration ##
The configuration is pretty much the same as the [previous post](http://josedbaez.com/2018/03/sitecore9-sso/#Enable-and-configure-providers) so I won't go into any details. You can get it [here](https://github.com/josedbaez/sitecore9sso/blob/master/App_Config/Include/SSO/SitecoreSSO.Providers.Editors.config). Do not forget to include [main configuration file](https://github.com/josedbaez/sitecore9sso/blob/master/App_Config/Include/SSO/SitecoreSSO.config).

``` xml
<?xml version="1.0" encoding="utf-8"?>
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/" xmlns:role="http://www.sitecore.net/xmlconfig/role/">
  <sitecore role:require="Standalone or ContentDelivery or ContentManagement">
    <pipelines>
      <owin.identityProviders>
        <processor type="Sitecore9SSO.Pipelines.IdentityProviders.OktaIdentityProvider, Sitecore9SSO" resolve="true" />
      </owin.identityProviders>
    </pipelines>

    <federatedAuthentication type="Sitecore.Owin.Authentication.Configuration.FederatedAuthenticationConfiguration, Sitecore.Owin.Authentication">

      <identityProvidersPerSites hint="list:AddIdentityProvidersPerSites">
        <mapEntry name="editors" type="Sitecore.Owin.Authentication.Collections.IdentityProvidersPerSitesMapEntry, Sitecore.Owin.Authentication">
          <sites hint="list">
            <site>shell</site>
            <site>admin</site>
          </sites>

          <identityProviders hint="list:AddIdentityProvider">
            <identityProvider ref="federatedAuthentication/identityProviders/identityProvider[@id='Okta']" />
          </identityProviders>

          <externalUserBuilder type="Sitecore.Owin.Authentication.Services.DefaultExternalUserBuilder, Sitecore.Owin.Authentication">
            <param desc="isPersistentUser">false</param>
          </externalUserBuilder>
        </mapEntry>
      </identityProvidersPerSites>

      <identityProviders hint="list:AddIdentityProvider">
        <identityProvider id="Okta" type="Sitecore.Owin.Authentication.Configuration.DefaultIdentityProvider, Sitecore.Owin.Authentication">
          <param desc="name">$(id)</param>
          <param desc="domainManager" type="Sitecore.Abstractions.BaseDomainManager" resolve="true" />
          <caption>Log in with Okta</caption>
          <icon>/assets/okta.png</icon>
          <domain>sitecore</domain>
          <transformations hint="list:AddTransformation">

            <transformation name="map role to idp" type="Sitecore.Owin.Authentication.Services.DefaultTransformation, Sitecore.Owin.Authentication">
              <sources hint="raw:AddSource">
                <claim name="idp" value="Okta" />
              </sources>
              <targets hint="raw:AddTarget">
                <claim name="http://schemas.microsoft.com/ws/2008/06/identity/claims/role" value="sitecore\Developer" />
              </targets>
              <keepSource>true</keepSource>
            </transformation>

            <transformation name="fullname" type="Sitecore.Owin.Authentication.Services.DefaultTransformation,Sitecore.Owin.Authentication">
              <sources hint="raw:AddSource">
                <claim name="name" />
              </sources>
              <targets hint="raw:AddTarget">
                <claim name="FullName" />
              </targets>
            </transformation>

          </transformations>
        </identityProvider>
      </identityProviders>

    </federatedAuthentication>
  </sitecore>
</configuration>
```

====================
References:
https://coding.abel.nu/2014/06/understanding-the-owin-external-authentication-pipeline/
https://dzone.com/articles/understanding-owin-external
http://openid.net/specs/openid-connect-core-1_0.html
https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-protocols-openid-connect-code


---

Please let me know what you think and/or if you can spot any errors.
*/eom*
