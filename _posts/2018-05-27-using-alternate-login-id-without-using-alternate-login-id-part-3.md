---
layout: post
title: Using alternate login id without using alternate login id – part 3
date: 2018-05-27 10:59:12.000000000 +02:00
type: post
published: true
categories: 
    - ADFS
    - Authentication
tags:
    - migrated
    - popular 
---

Welcome back to part 3 of this not so typical technical article about alternate login id. If you have missed the first two you should start reading them first to fully get my intention behind it.

In this article I will focus on the claim rules configuration in general and of  course also for Office 365 in special.

But before jumping into the claim rule language I want to visualize the actual flow of authentication with our database in place.

![AlternateLoginId-gfx03]({{ site.baseurl }}/assets/migrated/2018/05/alternateloginid-gfx032.png)

We still have the Active Directory as our first claims provider but when the relying party trust is going to the transformation rules we can query the SQL database.

### The Default SQL claim issuing rule

Of course you can use the database for any relying party trust. I have over hundred applications that are using ADFS and most of the time I use the database for getting all the claims I need. So when I add a new relying party trust I always create this first transformation rule as my starting point that looks somewhat like this:

```text
@RuleName = "Issue default claims from SqlAttributeStore"
c:[Type == "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn"]
 => issue(store = "SqlAttributeStore", types = ("http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname",
 "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname", 
 "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name", 
 "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress"), 

 query = "SELECT LastName, FirstName,Displayname, EmailAddress, from dbo.vPersons where UPN = {0}", param = c.Value);
```

Issuing from a database needs three parameters

1.  **STORE **- this is the name of the attribute store we configured in the first part
2.  **TYPES **- this is a comma separated list of claim names you want to issue
3.  **QUERY **- just a plain SQL language query. The order in the SELECT statement is important.

### Office 365 Claim Rules

When you deploy Azure AD you can do the easy start and let Azure AD Connect create the initial Office 365 claim rules for you. The modification for our alternate login id is then straight forward and only one rule has to be modified.

The first rule Azure AD connect is creating in your ADFS farm is the one to issue the user principal name and it should look like this:

```text
@RuleName = "Issue UPN"
c:[Type == "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname"]
 => issue(store = "Active Directory", types =
("http://schemas.xmlsoap.org/claims/UPN"), query =
"samAccountName={0};userPrincipalName;{1}", param = regexreplace(c.Value,
"(?<domain>[^\\]+)\\(?<user>.+)", "${user}"), param = c.Value);

The query method used in this rule accepts three parameters:

QUERY = "<QUERY_FILTER>;<ATTRIBUTES>;<DOMAIN_NAME>\<USERNAME>"

1.  **QUERY_FILTER** - samAccountName={0}. ADFS replaces the {0} term with the result from the first parameter:

    regexreplace(c.Value, "(?<domain>[^\\]+)\\(?<user>.+)", "${user}"

    _(This expression will strip the domain and the '\' from the windowsaccountname claim value and use only the samAccountName as query filter.)_

2.  **ATTRIBUTES** - the query returns the attribute samAccountName.
3.  **DOMAIN_NAME>\<USERNAME** - The last parameter {1} will be replaced by **c.Value** - the actual WindowsAccountName. The query method needs this information to determine the correct domain to run the query against.

#### Example

So if your Accountname is DOMAIN\johndoe the query will resolve into:

query = "samAccountName=johndow;userPrincipalName;DOMAIN\johndoe"

and the result will be something like **johndow@domain.local**.

So maybe you are now asking why we have done all this effort with this database thingy. If we want to use the email address as the upn for Office 365, we could just replace the attribute _userPrincipalName_ with _mail_ in the query above, right?. This works fine if you have processes in place that ensure that you have filled mail in every domain 100% correct and are absolutely sure, that no other process changes this value out of a sudden - because if it changes, your next authentication with this user will not work.

Because of this (and a lot other very special reasons) I prefer to have the database as my source for looking up the mail attribute - to have full control over the data

The following claim rule will pull the email address out of the database and put into the upn claim. You can just swap the default rule for issuing the UPN with this one

```text
@RuleName = "Issue UPN" 
c:[Type == "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn"]
=> issue(store = "SqlAttributeStore", 
types = ("http://schemas.xmlsoap.org/claims/UPN"), 
query = "SELECT EmailAddress, from dbo.vPersons where UPN = {0}", 
param = c.Value);
```

So that's it! - after applying your changes, ADFS will issue the email address as UPN and send back the claim to the Microsoft login endpoint. In theory we are now done.... but as always, the devil comes in many forms  :-)

### Challenge when redirected from Office 365

When I first implemented this alternative alternate login id, my excitement lastet only a few seconds after I pointed my browser to portal.office.com and typed in my federated username. Microsoft redirected me to my ADFS server correctly but they have also pre-filled in the username I typed in to the portal.office.com login form. They are achieving this by using a parameter in the redirect uri:

username=robert.johnson%40company.com

From a user experience perspective this is great - you have to type in your username only once. But because ADFS is not configured for the classic alternate attribute in this case it is only looking for upn / samAccountname and the mail does not match. So the login will fail. To make this work the user has to change the username always to the internal UPN to login successfully. I immediately was thinking that no user in the world would understand this and we would end up with a ton of tickets every week.

I can remember the feeling we had at this moment, realizing that we maybe end up going the hard way - change UPNs in all our 60+ domains and have 60+ registered and federated domains in our Azure AD.

But to give up was not an option and so we did some brainstorming. One idea  immediately came up was to do some replacement in the ADFS logon process on the username. But how can we achieve this? In ADFS 3.0 Microsoft changed the underling http server from a full blown IIS to a more secure and lightweight http.sys implementation. Also all the pages you are getting from ADFS created at runtime. So there is no swapping out of some ASP.NET pages in the background.

The only official modification you are allowed to do in ADFS 3.0+ are custom CSS and JavaScripts. And that was a starting point! We quickly developed the idea of having some sort of asynchronous client side lookup against a database to replace the mail with the corresponding UPN. This is a high level flow of this idea:

![AlternateLoginId-gfx02]({{ site.baseurl }}/assets/migrated/2018/05/alternateloginid-gfx02.png)

This is part of the function we integrate into the login process:

```javascript
var Identity = function () {  
    var self = this;  
    self.UPN = undefined;  
    self.success = undefined;  
    self.done = undefined;  
    self.failed = undefined;

    self.resolve = function () {  
        if (typeof Login != 'undefined') {  
            var userName = document.getElementById(Login.userNameInput);

            var xhttp = new XMLHttpRequest();

            xhttp.onreadystatechange = function () {  
            if (this.readyState == 4 && this.status == 200) {  
                var data = JSON.parse(xhttp.responseText);  
                userName.value = data.upn;  
                self.UPN = userName.value;  
                if (self.success !== undefined) {  
                    self.success();  
                }  
           }  
        };

            xhttp.open("GET", "https://some.api.com/api/id/" + userName.value, true);  
            xhttp.send();  
        }  
    }  
}
```
### Conclusion

At the end we have created and functional process to use mail as our username without having to touch any of the domains. Was it worth the effort? Definitely! The time needed for this whole setup was about two weeks. In comparison to this touching all the Active Directories was estimated to 6 to 12 months.

The constraints in this implementation are the same as using the classic alternate login id. So every time you hit a situation where basic authentication pops up you have to enter your samAccountName or userPrincipalName into it. The only way out of this is to change as much application as you can to modern authentication.

### What's next

I will add a fourth article to this series where I will describe a way to fill the SQL database without having a identity management solution. I personally to not recommend this in production, but for a lab environment this can come in handy.

I hope you liked my articles so far and I could give you some new perspectives on how to implement Azure AD federated authentication in a different way.