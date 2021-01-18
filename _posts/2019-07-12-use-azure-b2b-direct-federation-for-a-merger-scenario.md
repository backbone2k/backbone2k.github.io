---

title: Use Azure B2B direct federation for a merger scenario
date: 2019-07-12 14:09:54.000000000 +02:00
published: true
categories:
- Authentication
- Azure AD
- Migration
tags:
- migrated
excerpt: In this article you will learn how to use the new Azure B2B direct federation
  feature to quickly integrate your next M&A challenge.
---

In Jule 2019 Microsoft announced the public preview for a new and awesome feature:

**==> Azure B2B direct federation.** 

In short: This feature lets you establish a federation trust between an existing Azure AD tenant an an identity provider (IdP).

The good news is that this IdP does not need to be ADFS! Instead it can be anything that is compatible with SAML 2.0 or WS-Fed (for WS-Fed currently only ADFS and Shibboleth are successful testest by Microsoft). So technically you can build a trust between almost any IdP on the market (in theory :-D)

The main purpose of this feature is to lower the bars for working together with a partner company more closely even if this partner is not using Azure AD for AuthN. You may think, that using Azure B2B is no big deal even when you don't have Azure AD running at your partner organisations, but redeeming invitations, creating LiveIDs etc. can be challenging for a large user base. Also the users in the partner org suffer SSO when relying on Live IDs or OTP flows.

In this case the direct fedartion can tighten the collaboration and streamline the process for AuthN/AuthZ

## Migration scenario with direct federation

Another scenario I recently came up with direct federation is to speed up a merge & acquisition scenario. If company A is already using Azure AD and the newly acquired company B has not move to Azure AD yet you can easly achieve a quick first collaboration. My experience is, that after the deal of acquiring company B is official, granting access to some resources like intranet / documents hosted on SharePoint Online or inviting people into Microsoft Teams must be realized quickly. Depending on the size of company B just sending out hundreds or maybe thousands of classic Azure B2B invites can have a big impact to users and to the support organisation.

With direct federation we now have now the perfect answer for onboarding company B in a blink! And the best part - we can integrate more or less everything that we may find for authenticating users with SAML2 / WS-Fed. So if company B is using ADFS - perfect; if they are strong bound to open source and linux maybe the have Shibboleth running - Check!

![image]({{ site.baseurl }}/assets/migrated/2019/07/image.png)

As shown above the IdP at company B can be anything that supports the protocols SAML 2.0 or WS-FED

## Limitations of this scenario

Of course using direct federation as part of an integration / migration is only a small part of the puzzle. Migrations are far more complicated than just granting access to some ressources in SharePoint Online or Microsoft Teams.

The described approach does not cover how to give access in the other direction. Of course this can also be achieved with a federation between company B and Azure AD - but this can result in some changes for the whole AuthN flow.

Email coexistence is also often something that needs to be planed carefully. Direct federation of course does not solve this. So if users from company B need to share the same domain for sending and receiving emails or you have the need for calendar sharing the approach to achieve a quick collaboration stage can be totally different.

## Limitations for direct federation

Direct federation is currently in public preview and Microsoft will sure change something on the way to GA. But you always should be aware what is working, and what is not.

From my point of view the biggest challenges before you can use direct federation are:

*   The company you want to federate with does not have validated their domain via DNS with Azure AD. So if someone made some integrations to Azure AD for some of their users in the past and they validated some or all of their domains to Azure AD direct federation is maybe not the relief you where looking for.
*   If the company you want to federate with runs it's own IdP the authentication URLs must meet some criteria. The domain name for authentication must match with the authentication URL.

Another important information you should keep in mind when setting up direct federation is that already invited B2B users from company B keep using the classic AuthN Azure B2B flow. In a classic M&A scenario you will stumble across this, because it is for sure that some of the users already collaborated before ;-). The easiest way to solve this, is to delete the already invited users. But be aware: Going this path you will lose all permissions for these users. Access permissions have to be re-applied for the users that are coming through direct federation.

## More information

If you want to read more about direct federation and how to set it up with ADFS you can take a look at the documentation provided by Microsoft [https://docs.microsoft.com/en-us/azure/active-directory/b2b/direct-federation](https://docs.microsoft.com/en-us/azure/active-directory/b2b/direct-federation)

## What's next?

In my next blog I will cover how to setup direct federation with Shibboleth and how to migrate existing Azure B2B user to Azure B2B direct federation - so stay tuned.