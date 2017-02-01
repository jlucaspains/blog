---
layout: post
title: "MVC with Windows Authentication and WIF"
date: 2016-10-20
comments: true
sharing: true
categories: [auth]
tags: [auth]
---

## Overview
In the previous post, I proposed a "better" way to handle authentication and authorization in web applications for Enterprise. 

What I'm going to demonstrate in this post is something rather common in corporate intranets: How to integrate Windows Authentication for intranet web applications using MVC,  Windows Identity Foundation 4.5 (WIF) and Claims based auth. 

## WIF 4.5
From [MSDN](https://msdn.microsoft.com/en-us/library/hh291066(v=vs.110).aspx):

> Windows Identity Foundation 4.5 is a set of .NET Framework classes for implementing claims-based identity in your applications.

What the doco above doesn't mention is that it is very painful to use WIF and there is little material out there on how to do it. I'm no specialist by any means, but I've used WIF to some extent and hope to shed some light with this post.  

## Windows Authentication
In enterprise environments it is rather common to have a ERP like SAP or Microsoft Dynamics and a collection (sometimes large collection) of smaller applications to achieve all the digital processing your business needs. Top this with a restrictive firewall and proxy, and Windows Authentication suddenly becomes very attractive.

## Roles
If there is one security concept that is widely misused is the concept of roles. I've seen so many applications that were designed to ask "Is user jdoe in role Admin?". The problem with this question is the fact that over time applications change, simple as that. When a role change, you will have to change the software to support it.

Roles are not a bad thing, on contraire my friend, they are a great way to organize and drive security using business terms and business roles. In fact I would go as far as to say they are essential to good security setup. What should change is how business applications use them. Instead of asking "Is user jdoe in role Admin?", we should ask "Does user jdoe have access to resource App.Page?". As far as the application is concerned, there is no role Admin, it doesn't matter which role grants or revokes the access all that matter is whether the user have access to the resource or not. 

## Claim
Disclaimer: The below is an extract of [MSDN](https://msdn.microsoft.com/en-us/library/ee517291.aspx)

A claim is a piece of information used to identity something. It can be a name, e-mail, resource access or anything describing such thing. It is issued and signed by an issuer and you should only trust the claim as much as you trust the issuer itself. WIF represents claims with a Claim type, which has an Issuer property that allows you to find out who issued the claim.

Normally, a claim is issued by a Security Token Service (STS), in this example I will merge the STS funcionality with the application in order to make my life easier (yes, I am lazy!).

## Enabling Windows Authentication
Start with creating yourself a brand new ASP.Net 4.5.2 web application, ensure to use the MVC template and select Windows Authentication for Authentication.

![New ASP.Net Web App]({{ site.url }}/images/posts/NewAspNetWebApp.png)

If you already have a web application, change the following in the web.config:

<pre class="brush:xml">
&lt;system.web&gt;
    &lt;authentication mode="Windows" /&gt;
&lt;/system.web&gt;
</pre>

Also, if you a running the website on IIS, don't forget to enable Windows Authentication and disable all other authentication methods.

## The Authentication Manager
After authentication a ClaimsPrincipal object is created with Claims from AD with user name and group membership information out of the box. Most likely these Claims won't be enough for your application authorization. This is where we create ourselves an implementation in order to transform our claims (remove claims or add custom claims). Given we are already relying on the Windows Authentication, we can derive an implementation of ClaimsAuthenticationManager and do the Claim processing.

<pre class="brush:csharp">
using System;
using System.Collections.Generic;
using System.IdentityModel.Services;
using System.IdentityModel.Tokens;
using System.Linq;
using System.Security.Claims;
using System.Web.Helpers;

namespace CustomAuth
{
    public class CustomAuthenticationManager : ClaimsAuthenticationManager
    {
        private readonly ICustomClaimsProvider _provider;

        public AuthenticationManager()
        {
            // Initialize your custom claims provider. Maybe use some sort or IoC here.
            _provider = New CustomClaimsProvider();
        }

        // Override the default authentication method so we can create a new ClaimsIdentity with the 
        // claims important to us
        public override ClaimsPrincipal Authenticate(string resourceName, ClaimsPrincipal incomingPrincipal)
        {
            var baseAuthenticate = base.Authenticate(resourceName, incomingPrincipal);
            var claimsIdentity = ((ClaimsIdentity)baseAuthenticate.Identity);

            var newClaims = new List&lt;Claim&gt;();
            newClaims.AddRange(GetClaimsToKeep(claimsIdentity));

            // inject any extra claim the user might have
            // this is the perfect moment to call your claims provider service and get all
            // application specific claims
            newClaims.AddRange(_provider.GetClaimsForUser(incomingPrincipal.Identity.Name));

            // you cannot change an identity's claims, so we need to create a new one
            var ci = new ClaimsIdentity(newClaims, "Negotiate");

            var result = new ClaimsPrincipal(ci);

            CreateSession(result);

            return result;
        }

        private static IEnumerable&lt;Claim&gt; GetClaimsToKeep(ClaimsIdentity claimsIdentity)
        {
            // select the claims you do want
            yield return claimsIdentity.Claims.FirstOrDefault(c =&gt; c.Type == ClaimTypes.Name);
            yield return claimsIdentity.Claims.FirstOrDefault(c =&gt; c.Type == ClaimTypes.PrimarySid);
            yield return claimsIdentity.Claims.FirstOrDefault(c =&gt; c.Type == ClaimTypes.PrimaryGroupSid);
        }

        private void CreateSession(ClaimsPrincipal transformedPrincipal)
        {
            // Create token and write a cookie with it
            // IMPORTANT: in order to use session token you must use SSL
            var sessionSecurityToken = new SessionSecurityToken(transformedPrincipal, TimeSpan.FromHours(1));
            FederatedAuthentication.SessionAuthenticationModule.WriteSessionTokenToCookie(sessionSecurityToken);
        }
    }
}
</pre> 

## Prevent continuous re-authentication
In above code, there is nothing checking for a session token and preventing re-authentication. Also, we need something to set the MVC Context with the new user. In order to achieve this we will need a HttpHandler:

<pre class="brush: csharp">
using System;
using System.IdentityModel.Services;
using System.Security.Claims;
using System.Threading;
using System.Web;

namespace CustomAuth
{
    public class CacheCustomAuthenticationHttpModule : IHttpModule
    {
        public void Dispose()
        { 
            // nothing to dispose...
        }

        public void Init(HttpApplication context)
        {
            // wire up our custom authentication after default Windows Authentication
            context.PostAuthenticateRequest += Context_PostAuthenticateRequest;
        }

        void Context_PostAuthenticateRequest(object sender, EventArgs e)
        {
            var context = ((HttpApplication)sender).Context;

            // no need to call transformation if session already exists
            if (FederatedAuthentication.SessionAuthenticationModule != null &&
                FederatedAuthentication.SessionAuthenticationModule.ContainsSessionTokenCookie(context.Request.Cookies))
            {
                return;
            }

            // find our CustomAuthenticationManager instance
            var transformer = FederatedAuthentication.FederationConfiguration.IdentityConfiguration.ClaimsAuthenticationManager;
            if (transformer != null)
            {
                // Invoke authentication which will process/transform claims
                var transformedPrincipal = transformer.Authenticate(context.Request.RawUrl, context.User as ClaimsPrincipal);

                // set context and thread user
                context.User = transformedPrincipal;
                Thread.CurrentPrincipal = transformedPrincipal;
            }
        }
    }
}
</pre>

## Wire everything together
So we have a custom authentication manager and a HttpModule, all we  need to enable them is to change our web.config 

<pre class="brush: xml">
&lt;configuration&gt;
    ...
    &lt;system.identityModel&gt;
        &lt;identityConfiguration&gt;
            &lt;claimsAuthenticationManager type="CustomAuth.CustomAuthenticationManager, CustomAuth" /&gt;
        &lt;/identityConfiguration&gt;
    &lt;/system.identityModel&gt;
    ...
    &lt;system.webServer&gt;
        &lt;modules&gt;
            &lt;add name="ClaimsTransformationHttpModule" type="Jbssa.Core.Mvc.Auth.ClaimsTransformationHttpModule" /&gt;
            &lt;add name="SessionAuthenticationModule" type="System.IdentityModel.Services.SessionAuthenticationModule, System.IdentityModel.Services, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089" /&gt;
        &lt;/modules&gt;
    &lt;/system.webServer&gt;
    ...
&lt;/configuration&gt;
</pre>

## Checking for permissions Server Side
The default ResourceAttribute uses Role to authorize an action, let's improve upon that and create a custom ClaimAuthorizeAttribute. This will allow us to decorate a controller action and assert the user has a specific claim before proceding.

<pre class="brush: csharp">
using System.Security.Claims;
using System.Web;
using System.Web.Mvc;

namespace CustomAuth
{
    public class ClaimAuthorizeAttribute : AuthorizeAttribute
    {
        string Claim { get; set; }
        string ClaimType { get; set; }

        public ResourceAuthorizeAttribute(string claimType, string claim)
        {
            Claim = claim;
            ClaimType = claimType;
        }

        protected override bool AuthorizeCore(HttpContextBase httpContext)
        {
            var principal = (ClaimsPrincipal)httpContext.User;

            // check for claim of type custom and content requested in ClaimAuthorize usage 
            return principal.HasClaim(ClaimType, Claim);
        }

        protected override void HandleUnauthorizedRequest(System.Web.Mvc.AuthorizationContext filterContext)
        {
            if (filterContext.HttpContext.Request.IsAuthenticated)
                filterContext.Result = new HttpStatusCodeResult((int)System.Net.HttpStatusCode.Forbidden); 
            else
                base.HandleUnauthorizedRequest(filterContext);
        }
    }
}
</pre>

Usage:

<pre class="brush: csharp">
// Anything inside this controller will only be invoked if user has ReportController claim
[ClaimAuthorizeAttribute("CustomClaim", "ReportController")]
public class ReportController : Controller
{
    // To invoke the Index action the user must to have both ReportController and ReportController.Index claims
    [ClaimAuthorizeAttribute("CustomClaim", "ReportController.Index")]
    public ActionResult Index()
    {
        return View();
    }

    // To invoke the Edit action the user must have both ReportController and ReportController.Edit claims
    [ClaimAuthorizeAttribute("CustomClaim", "ReportController.Edit")]
    public ActionResult Edit()
    {
        return View();
    }
}
</pre>

## Checking for permissions Client Side
In a razor view, the easiest way to give the user a visual indication of access is to hide the resources they don't have access to:

<pre class="brush: csharp">
@if(User.HasClaim("ClaimType", "ClaimName"))
{
    &lt;button type="submit"&gt;
        &lt;span class="glyphicon glyphicon-save"&gt;&lt;/span&gt; Save
    &lt;/button>
}
</pre>

Or, if you would rather disable the resource:

<pre class="brush: csharp">
&lt;button type="submit" @(User.HasClaim("ClaimType", "ClaimName") ? "disabled" : null)&gt;
    &lt;span class="glyphicon glyphicon-save"&gt;&lt;/span&gt; Save
&lt;/button&gt;
</pre>

What about "smart" users?, Couldn't they post the data using another tool?

Nope. Even if the user circumvents the UI limitations, the server is also protected and will return a 403 error. Isn't that a sweet perk? 


Hope this was helpful. Thoughts?