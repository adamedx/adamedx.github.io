---
title: "Microsoft Graph via PowerShell Part 1"
date: 2018-08-08 21:51:00 -0700
categories: softwarengineering
tags: powershell microsoft-graph poshgraph autographps aad-graph powershell-graph msgraph azure
---
*This is the initial post in a series on PowerShell and the Graph.*

Yes, you can absolutely access [the Microsoft Graph](https://graph.microsoft.io) through PowerShell, and you don't need to wait for an official SDK -- it (mostly) works out of the box!

I've been working with Graph, the API gateway for Microsoft's vast array of services, over the last year. While PowerShell is a great fit for Graph, I ended up developing the [AutoGraphPS OSS project](https://github.com/adamedx/poshgraph) (previously named *PoshGraph*) in an effort make the experience of Graph + PowerShell as seamless as it should be. At its heart, the implementation is quite straightforward -- however, the barriers one must hurdle to get to the easy part are one reason I made the investment in a dedicated project.

## The basics -- tokens and REST
First, there is already a good write-up on how one can [access Graph from PowerShell](https://blog.kloud.com.au/2016/09/13/leveraging-the-microsoft-graph-api-with-powershell-and-oauth-2-0/) -- it's worth browsing before I dive into a targeted version of the approach. We'll cover the technique in the following sample I've created for explanatory purposes:

* [Demonstration of Microsoft Graph through PowerShell](https://github.com/adamedx/PowerShellGraphDemo)

The sample provides cmdlets that allow you to invoke any Graph method by specifying its URI (e.g. `https://graph.microsoft.com/v1.0/me`). The cmdlets will also return the JSON in raw form or as deserializeed PowerShell objects. In itself this can function as a primitive "PowerShell Graph SDK." This forms the basis of [AutoGraphPS](https://github.com/adamedx/autographps). Here's the recipe from the demo:

1. [Register an Azure Active Directory application](https://apps.dev.microsoft.com) (one time only)
2. [Get an access token](https://github.com/adamedx/PowerShellGraphDemo/blob/133a1e0c2859abc8bcf31da3ce9c9372f2eb4dd3/PowerShellGraphDemo.ps1#L226) at runtime for your application with permissions for the call you'd like to make
3. [Make a REST request to the documented Graph URI](https://github.com/adamedx/PowerShellGraphDemo/blob/133a1e0c2859abc8bcf31da3ce9c9372f2eb4dd3/PowerShellGraphDemo.ps1#L227) using the relevant verb with the `Authorization` header set to the access token and the JSON body containing any appropriate parameters
4. [Convert any JSON from the response](https://github.com/adamedx/PowerShellGraphDemo/blob/133a1e0c2859abc8bcf31da3ce9c9372f2eb4dd3/PowerShellGraphDemo.ps1#L228) to easy to manipulate PowerShell objects

## How it works -- overview
The first step is done only once, and is essentially manual. After that, steps 2-4 are captured in the sample below that uses the cmdlets from the sample to against the URI `https://graph.microsoft.com/v1.0/me`:

```
$accessInfo = GetGraphAccessToken # step 2
$result = InvokeGraphRequest $accessInfo.GraphUri $accessInfo.token v1.0/me # step 3
$result.content # step 4
```

You can actually follow the [instructions](https://github.com/adamedx/PowerShellGraphDemo/blob/master/README.md) in the sample repository and then execute the above commands if you have an Azure Active Directory (AAD) account or Microsoft Account (MSA). When you run it, you'll be prompted to provide credentials for the account to sign in. After a successful sign-in, the Graph API will be invoked and you'll get back a `2xx` response containing the output from the call. In this case, the output is the profile information (e.g. name, email address, etc.) of the user who signed in. This will work for any Graph URI, not just this particular profile API URI.

Note that if you try it with a different URI, you must specify it by leaving off the leading `https://graph.microsoft.com` part of the URI and only specify the path relative to that, i.e. in the example above `v1.0/me`. You must also have a token with the [right permissions](https://developer.microsoft.com/en-us/graph/docs/concepts/permissions_reference) for the part of the Graph you're accessing.

### Step 1: Register your application
In order to understand the sample, you can skip this section on app registration altogether. The remainder of the section is here simply to satisfy your curiosity -- if you're new to Graph or Azure Active Directory applications, this topic may be more meaningful after you've read all the posts in the series.

Before your code can access Graph, it must obtain an access token, and a prerequisite for this is for your code to identify itself with a unique *application ID*. The application ID is obtained by performing a one-time registration in the [Application Registration Portal](https://graph.microsoft.com).

Since the sample defaults to using a pre-registered application, you'll be able to run it as-is without performing your own app registration. If you are curious or want to use this code beyond learning about Graph, you can visit the aforementioned portal to obtain an application ID; the sample cmdlets include a parameter that allows you to use your own registered application ID rather than the default built into the sample.

Note that for your own app ID to work with this sample, you must add the "native" platform when you configure the application on the portal -- the sample uses a native authentication flow as opposed to one of the "web" flows. Web flows require that the application itself prove its identity, and that's not something the sample does (it relies only on the user's proof of identity when obtaining an access token).

### Step 2: Get your access token
This part is worth a post in its own right -- we'll cover this in the next post. For now, we'll treat this as a black box -- to follow along in the next sections, assume we've executed the following command to obtain an access token:

```
$token = (GetGraphAccessToken).Token # step 2
```

### Skip to step 3: Invoke-WebRequest

To understand the sample, we'll start with step 3 where we make the REST request. Consider the following invocation of the sample's `InvokeGraphRequest` cmdlet to access `https://graph.microsoft.com/v1.0/me`, i.e. the basic profile information of the caller. Here we assume that the token from step 2 has been retrieved and is available in a variable called `$token` by calling `GetGraphAccessToken` or an equivalent command:

```
InvokeGraphRequest -GraphBaseUri https://graph.microsoft.com -GraphRelativeUri v1.0/me -GraphAccessToken $token # step 3
```

If you examine the code for `InvokeGraphRequest`, it really is just a wrapper around PowerShell's built-in `Invoke-WebRequest` cmdlet. You can see this if you re-run the above command with the `-verbose` option:

```
PS> $response = InvokeGraphRequest -GraphBaseUri https://graph.microsoft.com -GraphRelativeUri v1.0/me -GraphAccessToken $token -verbose
VERBOSE: GET https://graph.microsoft.com/v1.0/me with 0-byte payload
VERBOSE: received -1-byte response of content type
application/json;odata.metadata=minimal;odata.streaming=true;IEEE754Compatible=false;charset=utf-8
```

From this we see that `Invoke-WebRequest` performs the following functions to make the Graph call:

1. Generates a URI based off of `https://graph.microsoft.com` to save you some typing
2. Sets the appropriate content header and importantly sets the `Authorization` header to the value of the passed via the  `-GraphAccessToken` option -- witout the token, the call would fail with an `unauthorized` error.
3. Calls `Invoke-WebRequest` with the headers, URI, and specified verb and http body.

The equivalent direct call to `Invoke-WebRequest` can be seen below (the need for the `-usebasicparsing` option is explained [here](https://github.com/PowerShell/PowerShell/issues/3042)):

```
$response = Invoke-WebRequest -usebasicparsing -Method GET 'https://graph.microsoft.com/v1.0/me' -Headers @{Authorization=$accessinfo.token.access_token;'Content-Type'='application/json'}
```

This shows that at its core Graph is a fairly lightweight abstraction on top of REST -- if you know REST and you have the [Graph API reference documentation](https://developer.microsoft.com/en-us/graph/docs/concepts/v1-overview), you can use Graph.

### Step 4: Graph JSON to PowerShell objects

When you use `Invoke-WebRequest` which returns an object of type `Microsoft.PowerShell.Commands.HtmlWebResponseObject`, you can access the `Content` member of the object to get the actual JSON response from Graph. In the examples above, you'd get the output below if you evaluate the `Content` member:

```
PS> $response.Content
{"@odata.context":"https://graph.microsoft.com/v1.0/$metadata#users/$entity","id":"9dd26bb5-f20c-4ed2-9977-cb247b84bd81","businessPhones":["+1 (313) 764-1817"],"displayName":"Cosmo Jones","givenName":"Cosmo","jobTitle":"Minister of Music","mail":"cosmo@starchild.org","mobilePhone":null,"officeLocation":"A/901","preferredLanguage":null,"surname":"Jones","userPrincipalName":"cosmo@starchild.org"}
```

This JSON output is human readable, if not exactly human friendly. Fortunately, the `ConvertFrom-JSON` cmdlet built into PowerShell can tame your JSON -- just pipe in the content, as in the following:

```
PS> $response.Content | ConvertFrom-Json

@odata.context    : https://graph.microsoft.com/v1.0/$metadata#users/$entity
id                : 9dd26bb5-f20c-4ed2-9977-cb247b84bd81
businessPhones    : {+1 (313) 764-1817}
displayName       : Cosmo Jones
givenName         : Cosmo
jobTitle          : Minister of Music
mail              : cosmo@starchild.org
mobilePhone       :
officeLocation    : A/901
preferredLanguage :
surname           : Jones
userPrincipalName : cosmo@starchild.org
```

The more readable appearance when emitting objects to the console is not simply cosmetic -- it reflects the fact that `ConvertFrom-Json` deserialized the JSON string into PowerShell objects. This means you can do things like this:

```
# Append the job title to a file
echo $response.Content.jobTitle >> MyJobs.txt
```

You can access members or elements of the response using `.` or `[]` notation if the object is an array, just as you would with objects returned by PowerShell cmdlets, or C#, or objects exposed in languages like Python or Ruby. In fact, `InvokeGraphRequest` returns an object with a `Content` member that instead of exposing a JSON `string` like that returned by `Invoke-WebRequest` it emits objects deserialized from that JSON.

So by building on REST + PowerShell, we have a natural object-oriented abstraction of the Graph, and this is a key part of the magic exposed by [AutoGraphPS](https://github.com/adamedx/autographps) in its mission to make the Graph accessible to anyone with a (Power) shell.

## Up next: step 2 -- getting the token
In an upcoming post, we'll backtrack and explore the rather important piece of the story we left out -- getting an access token. Without it, we can't make any calls to the Graph API.

It turns out that as easy as it was to invoke the API once we had the token, obtaining a token is much more involved -- it's not just a simple wrapper on top of an existing PowerShell cmdlet unfortunately.

