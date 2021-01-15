---
layout: post
title: Using alternate login id without using alternate login id - part 1
date: 2018-05-21 02:37:56.000000000 +02:00
type: post
published: true
categories: 
    - ADFS
    - Authentication
tags:
    - migrated
    - popular 
#permalink: "/2018/05/21/using-alternate-login-id-without-using-alternate-login-id-part-1/"
---

When you are working with complex Active Directory environments that have been more or less grown uncontrolled, it is always a challenge to move your workload to the cloud. Most administrators are faced with a challenge from start when it comes to the sync with Azure AD Connect and the user principal names.

Wouldn't we all love to undo our errors from the past? Mostly the ones we were forced to do by software companies like Microsoft?

One of this ancient mistakes most of us made  was relation to naming your Active Directory forest/domain. A lot of selected a non routable domain name like for example @contoso.local. Also using your companies name or an acronym is something you will regret latest when the CEO decides to merge with another company and not keeping the name or when marketing decides the brand will do better with a new name :-)

But hey, most of the time you lived with this blemishes and just ignored them. But the cloud is not forgiving and so all this little scratches in the paint are making  admin life very hard, nowadays.

My personal situation is similar - I am managing an environment with over 50 different AD forests which belong to one company and are trusted to a single resource forest. And most of the forests and domains in this zoo are having a naming issue.

So if you want to make your first steps to Office 365 and you start deploying Azure AD Connect you will stumble across the challenge with the UPN suffixes. The recommended way is to create a new suffix and change all your users that you want to sync to Azure AD to. For a single forest / single domain environment this may be an easy task. In my case this option was off the table because of three reasons:

1.  We have approx. 3000+ applications in the wild - you cannot imagine what will happen with all these fancy LDAP integrated Java-Tomcat applications when you change the UPN over night :-)
2.  Because we have a central resource forest and needed a unique username across all these domains we made a decision to use the userPrincipalName attribute - so changing UPN names is even more challenging
3.  Because we have applications in our central resource forest that are providing interactive login (e.g. Citrix) you cannot use the same UPN suffix across all your domains and have name suffix routing working properly.

The last point was a real issue for us and we could not solve easily. For user experience the plan was to use the users primary email address as user name in Azure AD - in our case this is only one domain that is used by all the users across these 50 forests.

![AlternateLoginId-gfx01]({{ site.baseurl }}/assets/migrated/2018/05/alternateloginid-gfx011.png) The ADFS farm is installed in the resource forest that is trusted to each of the over fifty account forests

#### Use an alternate user identifier

In theory Microsoft provides a very good solution for this challenge - the alternate login id feature in ADFS. So basically what it does when you configure it, is to look into a custom defined alternative attribute to search for the user object in the Active Directory.

If you are using this feature and already configured it successful you can stop reading :-) For all others that are standing in front of a complex environment and struggle how to implement the alternate logon id I will try to give you some ideas how to configure it with a lot less drawbacks than the Microsoft way.

In this first part I will cover a little the issues, mechanics and requirements you need to implement an alternative alternate login id (alt²-login-id)

#### The dark side of the Microsoft alternate logon id implementation

As I said before, if you are dealing with an environment that is straightforward the feature that comes out of the box is sufficient. But there are situations you have to think twice before implementing this:

*   If you have lots of domains that also change on a regular base (for example by merge and acquisition)  you have to keep in mind that you have to keep the ADFS alt-login-id config up to date
*   You have to be sure that the attribute you are using is indexed in all domains
*   My observation was, that ADFS is searching the domains in sequential order, so the time it takes is the summary of all domains involved. This is fine when all of your AD sites are maintained across all forests and also the latency and bandwidth are ok. In my case - even if you have a response in 0.5s from each domain it would took nearly 30s to full fill the search.
*   You have to be absolutely sure that the value in the attribute you are searching is unique across all domains. Otherwise it will fail
*   When configuring the alternate login id, ADFS will always search this attribute before trying the upn / samAccountName

For sure, this list is not complete and maybe some of these behaviors may change with future releases.

So as you can see there are some reasons why the built in functionality maybe not a solution for you. But even the alternative I will present to you is not dealing with all the problems an alternate login id brings with it - so for example in an Exchange hybrid environment you still will have some situations where basic authentication will popup and the user has to remember to use his UPN and not the alternate login id.

#### Requirements

The solution I will characterize in this series is working with ADFS 2012 R2 and 2016\. I will not go into details how to setup a working ADFS farm - this is covered by hundreds of excellent blogs.

Secondly we need an additional attribute store. In ADFS 2012 R2 you can leverage a SQL database for this purpose, in ADFS 2016 you could also use an LDAP directory (e.g. ADLDS). If your ADFS farm is already running on SQL (not a WID - Windows Integrated Database) you can use this SQL instance also for the attribute store.

The last thing we need is some sort of sync engine or a script that is able to pull information out of the Active Directory domains where your user accounts are stored and put it into the SQL database or LDAP. We will cover this in one of the upcoming parts with an easy approach.

In the production environment I am working on, we have an enterprise identity management solution that is doing all the hard work of syncing :-) If you are working in really complex environments you should consider to think about an identity access management (IAM) application like for example Microsoft Identity Manager or One Identity.

#### Adding an SQL attribute store to your ADFS farm

Adding another attribute store ADFS is straight forward. Just customize the variables in the following script to your needs and run it in an elevated PowerShell console on one of your ADFS servers (If you are using ADFS with WID you have to run it on your primary ADFS server)

You can grab the latest version of this script directly from my [github repo](https://github.com/backbone2k/adfsTools/blob/master/addSqlAttributeStore.ps1)

```powershell
$AttributeStoreName = "SQLAttributeStore"

$ServerName = "SqlServer.mydomain.net"  
$SQLPort = 1433  
$DatabaseName = "AttributeDB"

#I recommend to use integrated security. The service account for the ADFS farm needs READ and CONNECT rights to the database  
#If you don't use integrated security you have to privde username and password  
$UseIntegratedSecurity = $True  
$Username = ''  
$Password = ''

#If the ServerName is an availabilty group listener you should set this to $true  
$AGListener = $False

$ConnectionString = ''

If ($SQLPort -ne 1433) {  
$ServerConnectionString = $ServerName+","+$SQLPort.ToString()  
} Else {  
$ServerConnectionString = $ServerName  
}

$ConnectionString += "Server=$ServerConnectionString;Database=$DatabaseName"

If ($UseIntegratedSecurity) {  
$ConnectionString += ";Integrated Security=True"  
} Else {  
$ConnectionString += ";User Id=$Username;Password=$Password"  
}

If ($AGListener) {  
$ConnectionString += ";MultiSubnetFailover=Yes"  
}

Add-AdfsAttributeStore -Name $AttributeStoreName -StoreType SQL -Configuration @{"connection"=$ConnectionString}  
```

You can create the attribute store without having created the database because ADFS is not checking connectivity when adding a new store. After running the script you should check that the store was created either by running PowerShell or by looking in the management console:

![AlternateLoginId-screenshot1-attributestore]({{ site.baseurl }}/assets/migrated/2018/05/alternateloginid-screenshot1-attributestore.png)

Or you can use PowerShell to check:

```powershell
Get-AdfsAttributeStore SqlAttributeStore | Format-Custom -Depth 2

class AttributeStore  
{  
Configuration =  
[  
class DictionaryEntry  
{  
Key = connection  
Value = Server=SqlServer.mydomain.net;Database=AttributeDB;Integrated&nbsp;Security=True  
Name = connection  
}  
]

Name = SqlAttributeStore  
StoreClassification = SQL  
StoreTypeQualifiedName = Microsoft.IdentityServer.ClaimsPolicy.Engine.AttributeStore.Sql.SqlAttributeStore, Microsoft.IdentityServer.ClaimsPolicy  
}  
```

Ok that is all for the first part. In the next part I will explain how to setup the database and some tables.