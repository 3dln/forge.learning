# Authenticate (.NET)

## OAuthController.cs

Create a .NET WebAPI Controller named **OAuthController** (see [how to create a controller](environment/setup/net_controller)) and add the following content:

```csharp
using Autodesk.Forge;
using System;
using System.Threading.Tasks;
using System.Web.Configuration;
using System.Web.Http;

namespace forgesample.Controllers
{
  public class OAuthController : ApiController
  {
    // As both internal & public tokens are used for all visitors
    // we don't need to request a new token on every request, so let's
    // cache them using static variables. Note we still need to refresh
    // them after the expires_in time (in seconds)
    private static dynamic InternalToken { get; set; }
    private static dynamic PublicToken { get; set; }

    /// <summary>
    /// Get access token with public (viewables:read) scope
    /// </summary>
    [HttpGet]
    [Route("api/forge/oauth/token")]
    public async Task<dynamic> GetPublicAsync()
    {
      if (PublicToken == null || PublicToken.ExpiresAt < DateTime.UtcNow)
      {
        PublicToken = await Get2LeggedTokenAsync(new Scope[] { Scope.ViewablesRead });
        PublicToken.ExpiresAt = DateTime.UtcNow.AddSeconds(PublicToken.expires_in);
      }
      return PublicToken;
    }

    /// <summary>
    /// Get access token with internal (write) scope
    /// </summary>
    public static async Task<dynamic> GetInternalAsync()
    {
      if (InternalToken == null || InternalToken.ExpiresAt < DateTime.UtcNow)
      {
        InternalToken = await Get2LeggedTokenAsync(new Scope[] { Scope.BucketCreate, Scope.BucketRead, Scope.DataRead, Scope.DataCreate });
        InternalToken.ExpiresAt = DateTime.UtcNow.AddSeconds(InternalToken.expires_in);
      }

      return InternalToken;
    }

    /// <summary>
    /// Get the access token from Autodesk
    /// </summary>
    private static async Task<dynamic> Get2LeggedTokenAsync(Scope[] scopes)
    {
      TwoLeggedApi oauth = new TwoLeggedApi();
      string grantType = "client_credentials";
      dynamic bearer = await oauth.AuthenticateAsync(
        GetAppSetting("FORGE_CLIENT_ID"),
        GetAppSetting("FORGE_CLIENT_SECRET"),
        grantType,
        scopes);
      return bearer;
    }

    /// <summary>
    /// Reads appsettings from web.config
    /// </summary>
    private static string GetAppSetting(string settingKey)
    {
      return WebConfigurationManager.AppSettings[settingKey];
    }
  }
}
```

The **Get2LeggedTokenAsync** method connects to Autodesk Forge and get the access token. As we need a public (read-only) and an internal (write-enabled) tokens, **GetPublicAsync** exposes as an endpoint while **GetInternalAsync** is for the application. 

To avoid getting a new access token for each end-user request, which adds unnecessary latency, let's cache them in a couple `static` variables. Note we still need to refresh it after `expires_in` seconds.

!> Share access token between users is only valid in this case, where all users are accessing the same information (2-legged). If your app uses per-user data (3-legged), **DOT NOT** use this approach.

As per comments, the **GetAppSetting** simply gets the ID & Secret from the **Web.Config** file.

Next: [Upload file to OSS](/datamanagement/oss/)