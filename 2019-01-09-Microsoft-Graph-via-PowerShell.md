---
title: "Microsoft Graph via PowerShell Part 2"
date: 2018-11-20 21:00:00 -0700
categories: softwarengineering
tags: powershell microsoft-graph poshgraph aad-graph powershell-graph msgraph azure
---
*This is the second post in a series on PowerShell and the Graph.*

In this series' first post, we showed how easy it is to make a call to the Graph REST API -- *once you've acquired an access token*. In this edition we'll show what it takes to get a token. It's certainly more complicated than just calling the Graph API.

## Get the token

In the previous article about making an end-to-end call to Graph, we referred to one of the commands in the [sample](https://github.com/adamedx/PowerShellGraphDemo):

```powershell
$accessInfo = GetGraphAccessToken # step 2
```

Let's look at the implementation of `GetGraphAccessToken` -- all but the last line below are [excerpted from the sample](https://github.com/adamedx/PowerShellGraphDemo/blob/v1.0.0/PowerShellGraphDemo.ps1#L262):

```powershell
    # Get the authorization code -- step 2a
    $authCodeUri = GetAuthCodeUri $appId $redirectUri $graphUri $graphScopes $logonEndpoint
    $authCodeInfo = GetAuthCodeInfo $authCodeUri.Uri


    # Construct the access token request using the authorization code -- step 2b
    $tokenRequestBody = GetTokenRequestBody $appId $redirectUri $graphUri $graphScopes $authCodeInfo.ResponseParameters.Code

    # Make a request to the token endpoint to receive a response with the access and refresh tokens -- step 2c
    $tokenResponse = invoke-webrequest -method POST -usebasicparsing -uri $tokenUri -body $tokenRequestBody -headers @{'Content-Type'='application/x-www-form-urlencoded'} -erroraction stop

    # Deserialize the access token from the response -- step 2d
    ($tokenResponse.content | convertfrom-json).access_token
```

## Step 2a: Get the auth code -- UX required
The first step may be the most difficult. It starts off simply enough: the method `GetAuthcodeUri` generates a URI for the authorize endpoint. A `POST` to this URI will return a response that contains the auth code required for this part of the Oauth2 protocoal. The base of this URI is just Azure Active Directory's logon endpoint, 'https://login.microsoftonline.com'. It is followed by a segment that is the tenant in which our user resides -- we'll use the `common` tenant, which means it can request an auth code for any tenant OR even a non-AAD Microsoft Account (MSA, e.g. outlook.com, hotmail.com, etc. user). This is followed by `oauth2/nativeclient`. The remainder of the URI allows us to specify the parameters particular to our usage:

* **AppID (aka `clientid`):** We must specify the id of the application we registered back in [Step1].
* **Application redirect URI:** As part of the application registration, we specified at least one redirect URI, e.g. `http://localhost`. This parameter must match one of the URIs registered on the applicaiton.
* **Response type:** Since we're trying to obtain an auth code, this must be `code`
* **Response mode:** How should the auth code, and any other parameters, be communicated back to us? We will choose `query`, in which the response returns a URI in which the query uri contains a `code` parameter. Descriptions of the other options for `response_mode` may be found in the [OAuth2 documentation](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-auth-code-flow).
* **Permission scopes:** We need to specify the scope of access granted to the application via the access token we'll eventually receive. Can the token do anything and everything which the user might be authorized to do? That would probably not be a great idea. If the application just needs the ability to retrieve the user's photo for display to the user, the scope `openid` which includes photo information may be sufficient. If it needs to show the user their e-mail messages, a scope such as `mail.read` is sufficient (or more formally in the case of reading e-mail via MS Graph, i.e. `graph.microsoft.com`, the scope is `https://graph.microsoft.com/mail.read`). In this case, we'll use `openid` as well as `offline_access`, which allow us to retrieve refresh tokens for continued access to the resource after the initial token expires (typically 60 minutes or less).
* **Request State:** In general, appliations that handle multiple simultaneous requests may need to differentiate between them and / or link the request to locally maintained informaiton. The `state` parameter can be used for this purpose. In our case, we generate a unique identifier for each request. When we receive a response to that request, the parameter will also be returned in that response, allowing us to validate that we are processing the correct request. In our particular case, we are making only one request at a time, so this is not strictly necessary, but with a few modifications to the application we might need a similar behavior, so we include it here for completeness to fully demonstrate the work required in a robust authentication client-side implementation.

You can use your favorite string building / formatting technique to generate the URI. The result will look something like this:

```
https://login.microsoftonline.com/common/oauth2/v2.0/authorize?client_id=53316905-a6e5-46ed-b0c9-524a2379579e&response_type=code&redirect_uri=https://login.microsoftonline.com/common/oauth2/nativeclient&response_mode=fragment&scope=openid offline_access https://graph.microsoft.com/user.read&state=49d18579-76a9-4bbc-a826-78a69a72a2f1&nonce=Kv7XrTo2ReRkNitZJw01MCNGOmv Q7fibzpC4vr41sew=&prompt=login
```

### Show the sign-in page
The next part is more involved. The call to `GetAuthCodeInfo` will retrieve the authcode. This is accomplished by making a `POST` to the URI we just constructed. The HTML from the response to that request will render a user interface for the user to sign in and safely communicate proof of that user's identity to the logon authority. The result of the user's successful authentication is an OAuth2 authorization code.

This means though that we need to render any returned HTML and allow the user to interact with it, and also intercept the response. It might be tempting to believe we could implement a custom UX here, however the HTML returned from the authorize endpoint contains more than just nicely formatted input fields for usernames and passwords. This UX also provides protections against spoofing and phishing attacks and in general keeps the input of passwords safe from other applications in the local device or network. However, it is an HTML page, so we need to show it in a browser, right?

This is somewhat true, and fortunately, it's rather easy for us to get our hands on a browser that can render the authorize endpoint's HTML response -- let's look at the starting lines of `GetAuthCodeInfo`:

```powershell
    add-type -AssemblyName System.Windows.Forms

    $form = new-object -typename System.Windows.Forms.Form -property @{width=480;height=640}
    $browser = new-object -typeName System.Windows.Forms.WebBrowser -property @{width=440;height=640;url=$authUri }
```

We now have a browser via `System.Windows.Forms.WebBrowser`. This will let us render the HTML response from the `authorize` endpoint (passed in to the function as the `$authUri` argument), take in the user's sign-in information, send that user input back to the authority, and receive another response which includes the authcode.

The remaining code in that function configures the browser object so it can show the HTML returned from the request to `$authUri` and intercept the subsequent response with a callback supplied by us in a PowerShell Script Block when the user completes the sign-in. Our callback will be invoked once the response is received, and it parses the URL query for the auth code since our request specified a `response_mode` of `query`.

With this auth code in hand, we can move to the next step, which is to retrieve an actual access token.

## Get the token

Getting the token is relatively easy -- just `POST` to the token endpoint with a body that contains the auth code, application id, and scopes. The token URI is just

```
https://login.microsoftonline.com/common/oauth2/v2.0/token
```

As for the body, the cmdlet we're using to make the request to the token endpoint, PowerShell's `Invoke-WebRequest` makes it easy to satisfy the documented requirement for the body of the `POST` request to consist of form fields for the auth code, scopes, etc., that are set to the appropriate values. Simply pass in a PowerShell hash table where the keys of the table are the names of the form fields specified in the [protocol](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-auth-code-flow). From the sample, we have the following hash table returned by a `GetTokenRequestBody` function:

```powershell
    @{
        client_id = $appId
        scope = $scopes
        grant_type = 'authorization_code'
        code = $authCode
        redirect_uri = $redirectUri
    }
```

Now we can put the URI and the body together and actually make the request as in the sample:

```powershell
    # The $tokenUri variable is 'https://login.microsoftonline.com/common/oauth2/v2.0/token' as described previously
    # And $tokenRequestBody is the hash value also described above
    $tokenResponse = invoke-webrequest -method POST -usebasicparsing -uri $tokenUri -body $tokenRequestBody -headers @{'Content-Type'='application/x-www-form-urlencoded'} -erroraction stop
```

That's it -- a 200 response will include an [idtoken](https://www.oauth.com/oauth2-servers/openid-connect/id-tokens/), [refresh token](https://auth0.com/learn/refresh-tokens/), and our core goal in this exercies, the [access token](https://www.oauth.com/oauth2-servers/access-tokens/).

The response itself is in JSON, and contains all 3 tokens, so in the sample we deserialize it to more easily get at the access token, using something like the following:

```powershell
$tokens = $tokenResponse | ConvertFrom-JSON $tokenResponse
```

Now the tokens are deserialized into `$tokens`, and the access token may be referenced via `$tokens.access_token`.

With the access token in hand, we may now use this to make REST calls to Microsoft Graph (i.e. in our example `https://graph.microsoft.com`). As you recall from the initial post in this series, we need to pass this access token in the `Authorization` header of every REST request. If we've deserialized the tokens into a variable called `$tokens`, we can use `Invoke-WebRequest` to make the authorized REST call:

```powershell
$requestHeaders = @{
    Authorization = $tokens.access_token
    Content-Type = 'application/json'
}

$graphResponse = Invoke-WebRequest -usebasicparsing -headers $requestHeaders -method GET -Uri https://graph.microsoft.com/v1.0/me
```

And of course, your access token is good for multiple requests, so you don't have to repeat the dance with the authorize and token endpoints to obtain a new token for each call -- at least until the token expires, which at the time of this writing is about 60 minutes or so. This is typically sufficient for simple one-off scripting scenarios in PowerShell, though if you're willing to dig into the protocol some more you can use the aforementioned "refresh tokens" to obtain new access tokens without prompting the user to log in again.

We'll stop for now though, as this post's key point has been made: *Obtaining the access token for Microsoft Graph calls is far more complex in terms of code and logic than the actual step of making the Graph request and returning usable results in PowerShell.*

## Can we make this easier? Yes!

Does every Microsoft Graph PowerShell scripter need to implement the numerous web requests and browser control invocations to enable users to authenticate and make Graph requests from a PowerShell script? The existence of the demo would point to "no" -- the demo is packaged in a reusable fashion, so interested parties could simply incorporate the demo code into their own demo scripts / modules and start making Graph calls. There are numerous caveats though:

* **Missing SLA for the demo**: The demo script is just that -- a demo. It is not a maintained library, and in fact the code could simply disappear at any moment (though I promise I would never intentionally do such a thing). There is no service level agreement (SLA) on maintaining access, backwards compatibility, adoption of protocol improvements, or fixing code defects (including security issues!). Developers / automators should take no dependency on this or any code until it has a credible SLA.
* **Missing features such as token refresh**: Earlier it was mentioned that while the demo obtains a refresh token that could be used to allow access tokens to continue to function after the initial short (60 minutes or less) expiration period, this feature was not implemented. The demo's main goals are to show what client interactions are required to use the Microsoft Graph and how such a client might be implemented in the PowerShell language and runtime. Also, other authentication flows such as client credentials useful for background automation with no user present are absent as well.
* **Security is tricky**: The demo involves a sensitive area of application development: security, specifically authentication and authorization. Security is not the place to adopt "demo" code. Related to the SLA issue, ss new security issues arise, whether in the code itself or the protocol, fixes, mitigations, and improvements *must* be made available quickly and reliably, from a *trusted and credible author*. This author must view the security status of the implementation as a high-priority endeavor. Not only that, this author *must* not simply provide an implementation as a side project -- the investment must be dedicated and staffed with experts, not amateurs.

Another way to say all of the above: this conceptual demo is not enough for production use. You're welcome of course to improve upon it and make it production-ready, including investing resources to back the appropriate SLA.

Fortunately, there's a much simpler alternative to becoming an OAuth2 expert.

### Microsoft Authentication Library (MSAL) to the rescue

It turns out that the well-resourced provider of the Microsoft Graph service itself also maintains a library that provides everything we need to easily and robustly obtain access tokens for Microsoft Graph.

The [Microsoft Authentication Library (MSAL)](https://github.com/AzureAD/microsoft-authentication-library-for-dotnet) is a .Net based library that answers each of the caveats with the demo:

* **Reliable SLA**: MSAL is maintained by Microsoft itself, and its also open source. It is already used by many applications, including Microsoft's own premiere applications and services. Microsoft actively promotes the project, and continues to deliver a constant stream of improvements to it. Microsoft's standard SLA on security fixes applies to it, as well as compatibility and support for protocol improvements. As an open source project the community, including those of us using it in PowerShell use cases, can submit our own fixes and improvements. Another key benefit of Microsoft's maintenance is simply the fact that the library *must* be around for a long time -- Microsoft and its ISV community are committed to it for the long haul.
* **Full user experience and protocol support**: MSAL supports all of the flows of OAuth2, including non-interactive cases useful for automation, unlike the demo. And it supports transparent refresh of the access token and other features of the protocol. It provides the user interface for interactive flows -- there's no need to manage browser controls (if such components are even available on the platform in question).
* **Security and Compliance**: As an officially supported Microsoft component, Microsoft will own addressing vulnerabilities in the library in a timely fashion as it does for all Microsoft software. Importantly, it's obvious that Microsoft has the resources to make this security commitment, and not just that, the expertise. Microsoft's experience on the security front spans many domains, including contributions to security standards like OAuth2.

All of this sounds great, of course, though MSAL was designed for .Net applications, not PowerShell. Fortunately, any .Net component is usable, typically quite easily, from PowerShell.

So for our next and final post, we'll show how to use MSAL to accomplish everything we covered in the demo script. That part at least is short enough that in terms of code obtaining the token will be only a small part of the experience -- as it should be.


