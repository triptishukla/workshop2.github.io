---
layout: post
published: true
title: 'IIS, ARR & Rewrite Rules (Part 1 - Cookies and Legacy System)'
subtitle: ARR is hard...
date: '2016-12-11'
tags:
  - dotnet
  - iis
  - arr
  - rewrite
  - cookies
bigimg: /img/rewrite.jpg
---
**Update:** I have released a Nuget library which enables you to test re-write rules easily. | [Nuget](https://www.nuget.org/packages/RewriteRuleTestHarness/) | [Github](http://github.com/workshop2/rewriteruletestharness/)

The project I have been working on at TJG is to add a routing layer into our Recruiter website to enable the platform to become more agile.

We have previously accomplished this by creating a [Reverse Proxy](https://en.wikipedia.org/wiki/Reverse_proxy) with [ARR](https://www.iis.net/downloads/microsoft/application-request-routing) and [Rewrite Rules](https://www.iis.net/learn/extensions/url-rewrite-module/creating-rewrite-rules-for-the-url-rewrite-module) to allow us to send certain requests to independent apps - these can then be released and evolve on their own. The problem I have been dealing with is the legacy/aging platform that has evolved of the years. 

Reverse proxies are desirable because:

- The routing is an additive process (we _shouldn’t_ have to modify existing code; however, this might be a little naive)
- We can test the routing layer for specific brands (a brand is a single site of our portfolio) without affecting the other sites
- We should be able to easily flip back/undo the routing at the load balancer level (super safe)
  
Dealing with rewrite rules in IIS can be super painful due to the lack of debugging that is available - however I will try and address this in later post.

Today I want to talk about **Cookies and Legacy Systems**.

 
# ARR + Cookies + Legacy System = Mess
To give some background as to why I am having to mess with cookies I am going to have to point a finger to the past and say _"I genuinely don't know why or how this happened, I am just having to deal with it"_. 

The recruiter side of TJG stores its cookies against **totaljobs.com** instead of **recruiter.totaljobs.com** - but why does this matter?

## Domains matter
On my first attempt of trying to get my routing layer deployed we started receiving many bugs and errors on the website - we were sensible and turned off the routing layer to ensure other customers weren't being hurt. I had tested my routing layer over and over locally, in INT and in PAT - HOW DID THIS HAPPEN?

Well, when testing internally I always started with a fresh browser with no cookies and everything worked :thumbsup:. In the real world many customers currently have their account cookies all set and when we turned on the routing they found they were getting logged out with the error **"Someone else is already logged in"** - my routing layer was for some reason setting all the cookies with to the sub-domain instead of the TLD (a hidden IIS 'feature').

Our customers now had duplicate cookies under both **totaljobs.com** and **recruiter.totaljobs.com** - eek :worried:. Luckily they just had to close their bowsers, clear their cookies or wait a few hours for the cookies to expire and everything was ok.

## Solving the problem
There are years of code and legacy behind how cookies are stored and read in the recruiter platform - I am just one person with the goal of adding routing (I don't have the resource to update all the cookie code).

IIS Rewrite Rules allow you to define rules for both in-bound traffic and out-bound traffic which enables us to modify HTTP responses.

The problem I needed to solve was: 
**"Ensure the domain is set for all _Set-Cookie_ headers to the TLD."**

I quickly identified the following **Set-Cookie** situations I needed to solve:

  1. No domain is set in the header
  1. The wrong domain is set in the header
  
### Solving problem no.1
Using REGEX I aimed to detect the **lack** of ``domain=`` in the header and then append the correct value onto the end of the Set-Cookie header.

First I created a pre-condition to ensure the rule only triggered for missing values in the Set-Cookie header:

```xml
<outboundRules>
  <preConditions>
    <preCondition name="set-cookie-is-missing-domain">
      <add input="RESPONSE_Set_Cookie" pattern="domain=" negate="true" />
    </preCondition>
  </preConditions>
</outboundRules>
```

Next I created a rule that accomplished the following:

  - Used the pre-condition we created above
  - Capture the correct domain we want to use (without ``recruiter.`` at the start)
  - Write out the new Set-Cookie value
  
To capture data in rewrite rules you put the data into a **regex group** which can then be accessed later using curly braces e.g. **{R:*}** or **{C:*}**.

  - R = Request
  - C = Condition

Next I created a condition to parse out the data I needed from the _HTTP_HOST_ header. The rule I created is very lenient as it doesn't require the header to start with `recruiter.` and also ensures that port numbers are not used.
```regex
^(recruiter.|)(.*?)(:|$)
```

Using the above regex means we can access the TLD via the **(.*?)** selector using the **{C:2}** variable.

Finally, we want to write out the new Set-Cookie header by appending a **; domain=** onto the end of the exiting value.

```xml
<outboundRules>
  <rule name="Ensure cookies domain is set" preCondition="set-cookie-is-missing-domain">
    <match serverVariable="RESPONSE_Set_Cookie" pattern="(.*)" negate="false" />
    <conditions trackAllCaptures="false">
      <add input="{R:0}" pattern="domain=" negate="true" />
      <add input="{HTTP_HOST}" pattern="^(recruiter.|)(.*?)(:|$)" />
    </conditions>
    <action type="Rewrite" value="{R:0}; domain={C:2}" />
  </rule>
</outboundRules>
```

Beautiful :smile:


### Solving problem no.2
This next rule is more of a filter which strips out domains that we don't want - in our case if a cookie is being stored against **"recruiter."**. 

Again, we need to create a pre-condition to detect a dodgy cookie domain:

```xml
<preCondition name="contains-recruiter-sub-domain-set-cookie-header">
  <add input="{RESPONSE_Set_Cookie}" pattern=".*?domain=recruiter.*?" />
</preCondition>
```

Then using the REGEX variable matching magic we are able to store down the Set-Cookie header before the domain (R:1) and after the dodgy domain (R:2) allowing us to re-assemble the header on the Rewrite action:

```xml
<rule name="Ensure cookies are not stored on recruiter sub-domain" preCondition="contains-recruiter-sub-domain-set-cookie-header">
  <match serverVariable="RESPONSE_Set_Cookie" pattern="^(.*?domain=)recruiter\.(.*?)$" negate="false" />
  <action type="Rewrite" value="{R:1}{R:2}" /> 
</rule>
```

### Testing
You will want to test these rules...really really well - coding at this level of your platform is super important to get right.

I am working on a testing library for IIS rules (simple XML parsing) which I will hopefully soon release.


## The end
Next post I will try and walk you through some in-bound header changes and how to auto-configure your IIS site to allow you to do this.

### Resulting OutBoundRules:

```xml
<system.webServer>
    <rewrite>
        <outboundRules>
            <rule name="Ensure cookies are not stored on recruiter sub-domain" preCondition="contains-recruiter-sub-domain-set-cookie-header">
                <match serverVariable="RESPONSE_Set_Cookie" pattern="^(.*?domain=)recruiter\.(.*?)$" negate="false" />
                <action type="Rewrite" value="{R:1}{R:2}" />
            </rule>
            <rule name="Ensure cookies domain is set" preCondition="set-cookie-is-missing-domain">
                <match serverVariable="RESPONSE_Set_Cookie" pattern="(.*)" negate="false" />
                <action type="Rewrite" value="{R:0}; domain={C:2}" />
                <conditions trackAllCaptures="false">
                    <add input="{R:0}" pattern="domain=" negate="true" />
                    <add input="{HTTP_HOST}" pattern="^(recruiter.|)(.*?)(:|$)" />
                </conditions>
            </rule>
            <preConditions>
                <preCondition name="contains-recruiter-sub-domain-set-cookie-header">
                    <add input="{RESPONSE_Set_Cookie}" pattern=".*?domain=recruiter.*?" />
                </preCondition>
                <preCondition name="set-cookie-is-missing-domain">
                    <add input="RESPONSE_Set_Cookie" pattern="domain=" negate="true" />
                </preCondition>
            </preConditions>
        </outboundRules>
    </rewrite>
</system.webServer>
```
