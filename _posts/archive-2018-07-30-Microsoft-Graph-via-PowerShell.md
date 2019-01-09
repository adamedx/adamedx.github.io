---
title: "Microsoft Graph via PowerShell Part 1"
date: 2018-07-24 21:51:00 -0700
categories: softwarengineering
tags: powershell microsoftgraph poshgraph
---
Yes, you can absolutely access [the Microsoft Graph](https://graph.microsoft.io) through PowerShell. I've been working with Graph, the API gateway for Microsoft's vast array of services, over the last year. While PowerShell is a great fit for Graph, I ended up developing the [PoshGraph OSS project](https://github.com/adamedx/poshgraph) in an effort make the experience of Graph + PowerShell as seamless as it should be. At its heart, the implementation is quite straightforward -- however, the barriers one must hurdle to get to the easy part are one reason I made the investment in a dedicated project.

## The basics -- tokens and REST
First, there is already a good write-up on how one can [access Graph from PowerShell](https://blog.kloud.com.au/2016/09/13/leveraging-the-microsoft-graph-api-with-powershell-and-oauth-2-0/) -- it's worth browsing it before I dive into a targeted version of the approach. We'll cover the technique in this sample I've created for explanatory purposes:

* [Demonstration of Microsoft Graph through PowerShell](https://github.com/adamedx/PowerShellGraphDemo)

The sample provides cmdlets that allow you to invoke any Graph method specifying its URI (e.g. `https://graph.microsoft.com/v1.0/me`). The cmdlets will also return the JSON result of the REST method, which PowerShell can easily deserialize into objects. In itself this can function as a primitive "PowerShell Graph SDK." This forms the basis of [PoshGraph].

1. Register an Azure Active Directory application (one time only)
2. [Get an access token](https://github.com/adamedx/PowerShellGraphDemo/blob/133a1e0c2859abc8bcf31da3ce9c9372f2eb4dd3/PowerShellGraphDemo.ps1#L226) for your application with permissions for the call you'd like to make
3. [Make a REST request to documented the Graph URI](https://github.com/adamedx/PowerShellGraphDemo/blob/133a1e0c2859abc8bcf31da3ce9c9372f2eb4dd3/PowerShellGraphDemo.ps1#L227) using the relevant verb with the `Authorization` header set to the access token and the body containing any appropriate parameters
4. [Convert any JSON from the response](https://github.com/adamedx/PowerShellGraphDemo/blob/133a1e0c2859abc8bcf31da3ce9c9372f2eb4dd3/PowerShellGraphDemo.ps1#L228) to easy to manipulate PowerShell objects

The first step is done only once, and is essentially manual. Let's actually look at sample code that uses the cmdlets do perform steps 2-4 above for the URI `https://graph.microsoft.com/v1.0/me`:

```
$accessInfo = GetGraphAccessToken
$result = InvokeGraphRequest $accessInfo.GraphUri v1.0/me
$result.content
```

Assuming you are an authorized caller and you've set the parameters correctly, you'll be prompted to provide credentials to sign in, and you'll get back a successful `2xx` response containing the output from the call, in this case the profile information (e.g. name, email address, etc.) of the user who signed in. This will work for any URI, not just this one.

The sample has instructions if you'd like to try it out -- you can then execute the exact commands above. Nnote that it assumes that you specify the Graph URI, you leave off the leading `https://graph.microsoft.com` part of the URI and only specify the path relative to that, in the example above `v1.0/me`.

### Example: E-mail with Invoke-WebRequest

To understand the sample, we'll skip ahead to step 2, making the REST request. Consider the following Microsoft Graph URI:

```
GET https://graph.microsoft.com/v1.0/me/messages
```



The `me/messages` relative URI indicates that we should get the email *messages* of *me*, the signed-in user. This REST call is very easy to make from PowerShell using `Invoke-RestMethod` or `Invoke-WebRequest`. Assuming you already have an access token stored in the variable `$token`, here's how you'd make the REST call and get the result:

```
$response = Invoke-WebRequest -Method GET 'https://graph.microsoft.com/v1.0/me/messages' -Headers @{Authorization=$token}
$responseContent = ConvertFrom-Json | $response.content
```

The first line makes the `GET` call to the URI with the access token set in the `Authorization` header. The returned response will have JSON content, which is great, but even better is the fact that PowerShell makes it easy to turn the JSON text into PowerShell objects. That second line uses the `ConvertFrom-Json` cmdlet to deserialize the response into the `$response` variable.

In the case of the `me/messages` invocation, the deserialized object is a collection of messages, each with properties such as `subject`, `from`, `importance`, `body`, etc., that represents an e-mail message. The messages are easy to access from PowerShell as in the following:

```
# Dump the first item in the response
$resultContent.value[2]

# Show just the subject of all responses
$resultContent.value | select Subject

# Show from and subject of messages with high importance
$resultContent.value | where Importance -eq High | select From, Subject
```

### PowerShell + Graph is easy, right?
As you can see in the examples above, once you've retrieved the results and converted the JSON to PowerShell objects, you can filter and otherwise manipulate them like any other objects emitted by your favorite PowerShell cmdlets. This makes it simple to integrate data from Microsoft Graph into your PowerShell scripts or even into ad hoc system administration use cases. While there's no example given here, write operations (e.g `PUT`, `PATCH` and some `POST` methods) are nearly as simple, only the `http` verb given to `Invoke-WebRequest` changes, along with the additional requirement of specifying a JSON body to the cmdlet; the JSON body itself may be constructed from easy PowerShell objects via `ConvertTo-Json`, the inverse of `ConvertFrom-Json`.

This ability to make Graph calls in what is essentially a single line of code makes using Graph from PowerShell look easy. However, the examples given here leave out a key piece of complexity: how do we get that initial access token referenced via the `$token` variable earlier in the example? The referenced article actually goes into detail on this; it provides a PowerShell-based technique for communicating with the Azure Active Directory OAuth2 service to obtain a token.

## About that token detail...
Unfortunately, while that example is informative, it's not production-ready due to the following omissions:

* Token acquisition UX: In order to acquire a token, you'll need to a render a UX based on HTML returned from a response from the Authorize URI. The code from the example can be found [here](https://gist.github.com/darrenjrobinson/b74211f98c507c4acb3cdd81ce205b4f#file-ps2graphapi-ps1), and at 10+ lines, it's certainly not a one-liner.
* Token refresh and management: The example provides no facility for refreshing the token -- the token will expire fairly shortly after it is obtained (currently within 60 minutes), so users will eventually need to re-acquire the token. By using [refresh tokens](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-v2-tokens#refresh-tokens), a more reliable user experience is possible, one that doesn't at times unpredictable to the user prompt for authentication.
* One-liners for the above: The nice thing about using `Invoke-WebRequest` in concert with `ConvertFrom-Json` to access Microsoft Graph is that a single call to Graph corresponds to a single line of PowerShell code. What about the code for token acquisition and management? Unfortunately, even in the example provided, token acquistiion turns out to be several lines of PowerShell code, and token refresh would add significantly more code. Just making a single Graph call requires a lot of code for what is conceptually a simple set of operations: acquiring a token only when necessary, and using it in a Graph call.

### Easier tokens with MSAL

There is good news, however: there is a .NET authentication library that provides a relatively simple token acquisition mechanism: the [Microsoft Authentication Library (MSAL)](https://github.com/AzureAD/microsoft-authentication-library-for-dotnet) implements an efficient interface for acquiring tokens. It even provides token refresh capability. And since it's .NET, it can be used from PowerShell. Assuming you've installed MSAL (possibly via [https://nuget.org](nuget)), the PowerShell code below would acquire a token:

```
# Assuming the MSAL dll Microsoft.Client.Identity.dll is in the current directory:
$scopes = @('User.Read', 'Mail.Read')
[System.Reflection.Assembly]::LoadFrom("$pwd\Microsoft.Identity.Client.dll") | out-null
$requestedScopes = new-object System.Collections.Generic.List[string]
$scopes | foreach { $requestedScopes.Add($_) }
$authResult = $msalAuthContext.AcquireTokenAsync($requestedScopes)
$authToken = $authResult.Result

# This is the actual value you'll set for your Graph request's
# Authorization header to represent the token:
$tokenForHeader = $authToken.CreateAuthorizationHeader()
```

## Enter PoshGraph -- it's all one line

The above code is not particularly complex, AND it handles token validation. It's still far from a one-liner though -- the only way to get that level of terseness is to turn that code into a reusable function. This is exactly the tack taken by [PoshGraph](https://github.com/adamedx/poshgraph). Using it gives you one-line that performs BOTH the token acquisition and the actual Graph REST invocation:

```
# Execute 'Install-Module PoshGraph -Scope CurrentUser' if you want to try PoshGraph
Get-GraphItem me # output 'me' to the PowerShell console

# Show the subject of the most recent email messages
$messages = Get-GraphItem me/messages
$messages | select subject
```

The `Get-GraphItem` cmdlet above from the PoshGraph module does four important things:

1. The first time it is called, it displays a credential UX to the invoking user so that an access token may be obtained. It uses MSAL to do this, so the token is also validated.
2. It makes the REST call to Graph.
3. It deserializes any objects returned from the REST call.
4. The next time `Get-GraphItem` is called, it re-uses the previously acquired token to call Graph and no credential UX is displayed

In the end, PowerShell really is a natural choice for accessing the Microsoft Graph. The most challenging aspect of interacting with Graph, acquiring the token, can anyway be accomplished through .NET, and thus that ability is available to PowerShell thanks to PowerShell's excellent .NET interop.

We'll take a closer look at [PoshGraph](https://github.com/adamedx/poshgraph) and its many additional capabilities in an upcoming post.


