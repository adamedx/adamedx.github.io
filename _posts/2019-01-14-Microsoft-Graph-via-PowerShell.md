---
title: "Microsoft Graph via PowerShell Part 3"
date: 2019-01-20 21:00:00 -0700
categories: softwarengineering
tags: powershell microsoft-graph autographps aad-graph powershell-graph msgraph azure
---
*This is the third and final post in a series on PowerShell and the Graph.*

In this series' [previous entry]({{ site.baseurl }}{% post_url 2019-01-09-Microsoft-Graph-via-PowerShell %}), we dove into the details of the conceptually simple "Step 2" of getting an access token on your way to making a simple call to Microsoft Graph. It turns out that the process is fairly complicated, even when you're aiming for a mere proof-of-concept. In this post, we demonstrate a simple solution that takes advantage of PowerShell's strength around interoperability with .NET and offers a production-level solution for access tokens.

## Get your token the easy way with MSAL

A natural improvement in removing the complexity of token acquisition is to package the PowerShell function we discussed earlier, [GetGraphAccessToken](https://github.com/adamedx/PowerShellGraphDemo/blob/v1.1.0/PowerShellGraphDemo.ps1#L157) into a reusable PowerShell module. One-time improvements to uplevel the quality and capabilities of the code would be required, ongoing bug fixes and tweaks, and of course tests would be required.

Fortunately, someone *has* already done that work, though it's in the form of C# compiled to .NET rather than PowerShell. This is the open-source [Microsoft Authentication Library (MSAL)](https://github.com/AzureAD/microsoft-authentication-library-for-dotnet), and the fact that it is .NET is no problem for us as .NET-based PowerShell can easily consume any .NET assembly. Here's what it looks like to use MSAL through code from our [sample repository](https://github.com/adamedx/PowerShellGraphDemo/blob/v1.1.0/Get-GraphAccessTokenFromMSAL.ps1):

```powershell
# Assumes you've cloned this repository and dot-sourced the sample code, i.e.
# . ./PowerShellGraphDemo.ps1
# . ./Get-GraphAccessTokenFromMSAL.ps1
$accessToken = Get-GraphAccessTokenFromMSAL
$result = InvokeGraphRequest me -GraphBaseUri $accessInfo.GraphUri -GraphAccessToken $accessToken
$result.content
```

Note that it looks very similar to what we outlined in the series' [first post]({{ site.baseurl }}{% post_url 2018-08-08-Microsoft-Graph-via-PowerShell %}), with the key difference that we invoke a function `Get-GraphAccessTokenFromMSAL` instead of `GetGraphAccessToken`.

The steps below will walk us through the MSAL approach for obtaining a token -- note that the numbering of the steps is unrelated to the numbers in the previous entries -- they simply describe the process for getting a token from MSAL. To map to the previous posts, think of all the numbered steps in this post as part of "Step 2" from the previous posts, which concerned how to obtain a token in general.

### Step 0 -- install MSAL

Our MSAL-specific sample can be found [here](https://github.com/adamedx/PowerShellGraphDemo/blob/v1.1.0/Get-GraphAccessTokenFromMSAL.ps1). It's a single function, `Get-GraphAccessTokenFromMSAL`, that logically replaces `Get-GraphAccessToken` from [PowerShellGraphDemo](https://github.com/adamedx/PowerShellGraphDemo/blob/v1.1.0/Get-GraphAccessTokenFromMSAL.ps1). We'll assume that you've cloned the [PowerShellGraphDemo](https://github.com/adamedx/PowerShellGraphDemo) repository locally. To try the MSAL sample, `cd` to the root of the cloned repository and dot-source the sample:

```powershell
. ./Get-GraphAccessTokenFromMSAL.ps1
```

Note that the first time this command is executed on a particular system, it may take a minute or two -- that's because it has to download the MSAL library from [nuget.org](https://www.nuget.org/packages/Microsoft.Identity.Client/). It gets copied to a subdirectory `./lib` in the local repository so it can be accessed by the script.

### Step 1 -- load the MSAL assembly

To use .NET types from PowerShell, we'll need to load the MSAL assembly, `Microsoft.Identity.Client.dll`, into the PowerShell process -- this is accomplished with the following one-liner excerpted from [Get-GraphAccessToken](https://github.com/adamedx/PowerShellGraphDemo/blob/v1.1.0/Get-GraphAccessTokenFromMSAL.ps1):

```powershell
[System.Reflection.Assembly]::LoadFrom((gi $psscriptroot/lib/Microsoft.Identity.Client.*/lib/net45/Microsoft.Identity.Client.dll).fullname) | out-null
```

The specific type we'll want to use from this assembly is [Microsoft.Identity.Client.PublicClientApplication](https://docs.microsoft.com/en-us/dotnet/api/microsoft.identity.client.publicclientapplication?view=azure-dotnet); once the assembly is loaded, we can refer to it from PowerShell using standard .NET type syntax [Microsoft.Identity.Client.PublicClientApplication].

### Step 2 -- create a PublicClientApplication type instance

The [PublicClientApplication](https://docs.microsoft.com/en-us/dotnet/api/microsoft.identity.client.publicclientapplication?view=azure-dotnet) type encapsulates the functionality for OAuth2 protocol exchanges that allow us to obtain an access token, i.e. it is the equivalent of all the code we wrote in our previous post to get a token. To use it, we simply need to `new` it as we would in C#, using either PowerShell's `New-Object` cmdlet or invoking `New` as if it were a static method on the type -- we use the latter in the sample as it is more concise:

```powershell
$authContext = [Microsoft.Identity.Client.PublicClientApplication]::new($appId, $logonEndpoint, $null)
```

This creates the new `PublicClientApplication` and assigns it to the variables `$authContext`.

### Step 3 -- get the token

Logically, the `AcquireTokenAsync` method of `PublicClientApplication` allows us to make a single method invocation to obtain the token! However, as this is a [.NET async method](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/), there are two steps to making the call: initiating the call, and then waiting for it to complete. This design is optimized for applications to avoid blocking threads which increases throughput for server applications and for client applications improves responsiveness by moving slow network calls out of the UX execution path.

For our sample case, and even for the case of scripting and automation, we don't generally need the capabilities offered by [.NET's async programming model](https://docs.microsoft.com/en-us/dotnet/csharp/async) (we don't have a requirement to handle other work while the call is in progress for instance), so we can make what is essentially a synchronous rather than asynchronous REST request to obtain the token. To go synchronous, we need to start the call and wait for it to complete, so what is logically one line of code turns into two lines, which we consider in the sample comments to be steps "3a" and "3b":

```powershell
$asyncResult = $authContext.AcquireTokenAsync([System.Collections.Generic.List[string]] $graphScopes) #3a
$token = $asyncResult.Result # 3b
```

The first line starts the call and stores a reference to it in `$asyncResult`. As input to `AcquireTokenAsync`, we must pass the permission scopes -- because the method requires the .NET type `[System.Collections.Generic.List<string>]`, we must cast the PowerShell array `$graphScopes` to that type, using PowerShell's syntax for that type `[System.Collections.Generic.List[string]]`.

The second line waits for the call referenced by `$asyncResult` to complete and retrieves the result and assigns it to the variable `$token`. Note these lines are what will trigger a browser UX to be shown for you to sign in.

This entire step 3 is actually MSAL exercising most of the protocol code we covered in the previous post: it generates the URIs for the `authorize` and `token` endpoints, displays a browser interface for sign-in, parses the response from the browser, and sends a request to and waits for a response from the `token` endpoint. It also complies with security best-practices around hardening the protocol with `nonce` values and other techniques for avoiding capture or replay of tokens or sign-in information. As new hardening techniques arise, they will be added to MSAL either transparently or with a simple opt-in flag, requiring no or little effort for applications using MSAL to take advantage of improvements in security.

Note that the token returned from MSAL is itself a .NET type [AuthenticationResult](https://docs.microsoft.com/en-us/dotnet/api/microsoft.identity.client.authenticationresult?view=azure-dotnet). This type contains the access token that we need to set in the `Authorization` header in calls to MS Graph, so `Get-GraphAccessTokenFromMSAL` returns the `AccessToken` field of the result of the call to `AcquireTokenAsync`.

### Step 4 -- use the token

At this point, using the token we've obtained from MSAL is the same as using the token we obtaine from our custom PowerShell script-only implementation:

```powershell
$accessToken = Get-GraphAccessTokenFromMSAL # Get the token
$result = InvokeGraphRequest me -GraphBaseUri $accessInfo.GraphUri -GraphAccessToken $accessToken # use it to call Graph
```

The second line is essentially the same as in the non-MSAL case -- we just obtained the token differently. We can then use the response in the `$result` variable just as we would have in the original case -- conveniently deserialized PowerShell objects contineu to be available via `$result.content`.

## So was that easier? Yes!

Let's compare the MSAL vs. non-MSAL cases:

| Measure            | Pure PowerShell | PowerShell + MSAL |
|--------------------|-----------------|-------------------|
| Library complexity | This is currently a custom implementation not supported by any OSS project or vendor. ~ 150 lines of code, or 40 code blocks for a non-production implementation limited to public client scenarios. Double this for production quality, and add more for additional auth flows like client credentials. | N/A, essentially 0 code blocks or lines. The complexity is hidden behind the external library. |
| Maintenance cost   | Cost is significant as there is no OSS project or vendor for this at the moment. Any developer would need to delve deeply into OAuth2 protocols to understand details of secure implementation and write tests to validate such improvements. As issues arise, particularly those related to security, fast response to address them would be warranted. If support for new scenarios or updates to the protocol is needed, you'd need to do this yourself | This is essentially 0 -- the MSAL OSS project is highly-utilized and already addresses mainstream scenarios, and bug fixes of all kinds are a key responsiblity of the project. New scenarios and protocol updates are thus likely to be added before you need them, but if they aren't, MSAL is open-source and you can contribute improvements, though you'll neeed to know C#, not PowerShell. |
| Compliance Cost    | This is high cost as again there is no current maintainer of a PowerShell library, so to be sure you're implementing sensitve OAuth2 protocols correctly, you'll need to become an OAuth2  and security protocol expert. | Minimal cost -- MSAL's implementation is compliant by design and where this is not so that omission is treated as a code defect that is quickly remedied. The library's interface makes it easy to use it in a compliant fashion and hard to do otherwise, so in the vast majority of use caes just using MSAL will mean you are compliant. |
| Client complexity  | Minimal -- the library as described here exposes an easy-to-use PowerShell cmdlet `GetGraphAccessToken` that acquires a token in one line. A similarly easy interface could be provided for auth flows beyond public client if the implementation supported them. | Minimal cost, though higher in cost than non-MSAL in terms of lines of code because of the need to adhere to MSAL's C# usage pattern of constructing an object, calling an async method, and waiting for the method to complete. Still, this amounts to essentially 4 lines of code vs. 1, or creating a thin wrapper for those 4 lines that's not worth maintaining as a library. |

In summary, by using MSAL we remove the maintenance cost of the ~150 lines (and ultimately more than that) of PowerShell code in exchange for a dependency on MSAL and a handful of lines of wrapper code. **From this standpoint it's difficult to justify NOT using MSAL.**

About the only improvement we could make on MSAL is if it were integrated directly into the default PowerShell distribution and exposed as a cmdlet rather than a .NET type. And even with that, we'd save only about 4 lines of code, i.e. the following 4 lines which are the only essential parts of `Get-GraphAccessTokenFromMSAL` in [Get-GraphTokenFromMSAL.ps1](https://github.com/adamedx/PowerShellGraphDemo/blob/v1.1.0/Get-GraphAccessTokenFromMSAL.ps1) (the remaining 40 or so lines of that file consists of comments, one-time installation code, and some parameterization to make the method more generic):

```powershell
[System.Reflection.Assembly]::LoadFrom((gi $psscriptroot/lib/Microsoft.Identity.Client.*/lib/net45/Microsoft.Identity.Client.dll).fullname) | out-null # Step 1
$authContext = [Microsoft.Identity.Client.PublicClientApplication]::new($appId, $logonEndpoint, $null) # Step 2
$asyncResult = $authContext.AcquireTokenAsync([System.Collections.Generic.List[string]] $graphScopes) # Step 3a
$token = $asyncResult.Result # Step 3b
```

Again, the 4 lines of PowerShell script above essentially replace 150 lines of minimally functional PowerShell script, and those 150 lines or more are replaced by code that is managed and maintained by an active and growing community.

## Beyond the demo

So if it's not clear by now, let's say it: **if you're accessing Microsoft Graph from PowerShell, you should use MSAL to obtain access tokens.** The capabilities of MSAL go well beyond what is shown here and include the following:

* Client-credentials flows: Rather than require a user to respond to a browser prompt to sign-in, client-credentials flows allow you to use certificates provisioned on a system to perform non-interactive, unattended automation, a key DevOps scenario.
* Refresh tokens and caching: In our example, we obtained a token that expires in roughly 60 minutes. To access Graph after that expiry, you'll need to get a new token, which in the case of interactive login means more UX popups, and even for non-interactive flows requires requires addtional potentially unreliable or slow network round trips. MSAL supports the use of *refresh tokens* which allow the initial token to be renewed without UX for fairly long periods of time (days, weeks, even longer).
* Confidential client support and additional flows: Our examples focus on cases where the caller and the script are using the same device, but if the PowerShell script were being used on a system different from that of the user, or if the user's device does not have the capability to show a browser UX for sign-in, MSAL's capabilities to support such scenarios would also extend to PowerShell.
* Additional features and protocol improvements: As suggested earlier, as the protocol is strengthened or evolves to support new scenarios, the investment in MSAL will accrue to your PowerShell scenarios.

## Series recap: MS Graph in PowerShell is... easy and natural

PowerShell is a great fit for the Microsoft Graph. Let's look at what we've learned:

* The hardest part of interacting with the Graph, getting a token, is easy in PowerShell if you just use [MSAL](https://github.com/AzureAD/microsoft-authentication-library-for-dotnet) by taking advantage of PowerShell's native interop with .NET.
* The next step, simply making a REST call, is also simple -- just use `Invoke-WebRequest` and toss your token in the `Authorization` header. Note that write operations are possible in addition to the read-only `GET` scenarios we've shone -- you can use `PUT`, `POST`, `PATCH`, and `DELETE` with `Invoke-WebRequest` (see the appendix).
* PowerShell's built-in JSON serializer makes it very easy to transform Graph's JSON responses into programmable objects that look just like any PowerShell object, making it seem almost as if Graph is built-in to PowerShell itself. The process can be reversed as well when writing PowerShell objects to Graph.

For a taste of a comprehensive Graph + PowerShell experience, see the [AutoGraphPS PowerShell module project](https://github.com/adamedx/autographps), which makes use of MSAL to implement flows beyond public client. `AutoGraphPS` also componentizes the techniques describes in this series, making it useful as a way to learn these approaches or even as a set of building blocks to write your own domain-specific Graph PowerShell scripts.

And one last plug -- if you do build something interesting with Graph + PowerShell, please share what you can as open source and on [PowerShell Gallery](https://www.powershellgallery.com). This will accelerate the powers and growth of the community of Graph + PowerShell users -- I know you're out there!

# Appendix: Beyond read-only scenarios

Our examples in this series have focused on read-only use cases, i.e. `GET` REST methods against the Graph. But PowerShell is not just for read-only operations:

* Write operations to the Graph typically involve `PUT`, `POST`, `PATCH`, and `DELETE` methods, and these can be initiated in PowerShell directly from the same [Invoke-WebRequest](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest?view=powershell-6) cmdlet we used for `GET` methods in our samples. Not only that, but you can build the requisite REST request bodies for Graph methods by reversing the deserialization object we took advantage of earlier; construct objects in PowerShell with built-in `Dictionary` types and arrays, and then convert them to Graph-compatible JSON via PowerShell's [ConvertTo-JSON](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json?view=powershell-6).
* And if you're missing any other functionality in PowerShell, remember that anything you can do in C#/.NET is also accessible from PowerShell -- any .NET assemblies available to you that provide Graph functionality may be used in your PowerShell scripts.


