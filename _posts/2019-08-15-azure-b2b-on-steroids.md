---
title: Azure B2B on Steriods
excerpt: How to improve your B2B management
date: 2019-08-15 18:44:12.000000000 +02:00
published: true
toc: true
tags: 
   - migrated
categories:
   - Identity
---

In this post I will cover some advanced ways to improve Azure B2B usage for day-to-day scenarios.

We start with some basic tweaks: Change the userType and add B2b users to Exchange Online

After this little warm up we will super charge your B2b users and put them on steroids 🙂

## Change the UserType attribute
Currently not widely known but definitely powerful is the possibility to change the attribute _userType_ from guest to member. Microsoft Office 365 services are using this attribute to restrict some access rights you normally don’t want your externals or partners to have. But what if you invite B2B users from another tenant that also belongs to your company? Wouldn’t it be great if these users could see all users in Microsoft Teams, do a search everywhere in SharePoint Online or just browse the Azure AD? By setting userType to ‘Member’ you get exactly that.

The attribute can easily be changed with the AzureAD PowerShell module:

```powershell
PS> Get-AzureAdUser -UserPrincipalName j.doe_contoso.com#EXT#@t1.onmicrosoft.com | Set-AzureAdUser -UserType 'Member'
``` 
Of course, you can change this value also back to ‘Guest’ if you want the user to be restricted again.

## Add Azure B2B users to Exchange Online Global Address List or distribution groups
When you invite a guest user to your Azure AD tenant, Microsoft will always provision an identity to your tenant. Most of the time I refer to this as a shadow account for the actual users that is authenticating within another tenant. This shadow account is also "synced" to Exchange Online because Microsoft creates this Azure B2B identity as a mail enabled user. If you want to put these identities in a distribution group or if you want to make it visible in the global address list, you can easily do this by connecting to your Exchange Online and fire up some PowerShell commands:

To make the user visible in your global address list:  

```powershell
PS> Get-MailUser j.doe@contoso.com | Set-MailUser -HiddenFromAddressListsEnabled $False
```
If you want to add John Doe to the distribution group ‘AllB2bUser’ use this command:  
```powershell
PS> Get-DistributionGroup -Identity AllB2bUser | Add-DistributionGroupMember -Member j.doe@contoso.com
```

If you are running a hybrid Exchange environment and have not yet moved all your users to Exchange Online you should be very careful with this approach. You could end up having different entries in your gal. Also while you have hybrid identities you cannot modify synced distribution groups. 

But If you want to get rid of these hurdles, my Hybrid Azure B2B user process will help you with that!

## Make your Azure B2B users hybrid
Most of you will already cover external users in your on-prem environment by creating AD users for them. These accounts often get challenging when these external users are mail enabled and you are going to use Office 365 for collaboration with your them. If you don’t want to license them with a SharePoint Online plan or an Office 365 E1 you maybe want to guest invite these external users to your Azure AD tenant. Azure B2B will help you achieve this with ease. But doing so can lead to new challenges. It is a good idea to filter the on-prem AD external accounts so you don’t end up with two identities in Azure AD for the same person. This would totally confuse your end users and reduce usability and acceptance in the business. Shown in the picture below you see what happens when you don’t filter the on-prem user. You will end up having two John Does in your tenant. If a user looks up John Doe in Microsoft Teams he will see two identities – one ‘internal’ and the guest.

But what if you have mail-enabled the external accounts in Active Directory to show up in the Exchange on-prem GAL or to be member in a distribution group? Skipping these users from syncing into Azure AD will break workflows in Exchange Online. Again a path that will lead to a decreased user acceptance.

One way out of this situation is described in this great docs article: https://docs.microsoft.com/en-us/microsoft-identity-manager/microsoft-identity-manager-2016-graph-b2b-scenario. So technically what it describes is installing MIM and syncing the Azure B2B users back to your on-premise environment.

But I think there are some flaws in this approach:

1. You have to install and maintain a MIM sync service. If not have one already running this is a huge challenge
1. The management for these users has to be done in Azure AD – so if you are currently with hybrid identites this will break your normal processes for user management
1. It gets even more complex when you want these B2b users also in your on-prem Exchange because you have to create all the provisioning rules into MIM
1. Onboarding existing AD users puts even more complexity in to the whole process

So I was thinking: why not using what is already there? If you have deployed a hybrid setup with AAD Connect why not use this tool for linking the Azure B2B identities to extisting on-premises identites? Of course I do not want AAD Connect to create new on-premise accounts, I just want to do some minor changes to Azure AD connect rules. With minimal effort the goal should look something like this:

## Import B2B user into AAD Connect Metaverse
First things first – the following description only works if your on-premises external accounts are **not** synced to Azure AD. If this is the case you can just go on. Otherwise you first have to deprovision these users from Azure AD and make sure your empty the AAD recycle bin.

Normally the Azure B2B identites are not imported into the Metaverse by the  Azure AD Connect AAD Import task. This is because these identities do not have any sourceAnchor or ImmutableId attribute set. So to make a first step into linking the B2B identity with an on-premises user account, we have to stamp an immutableId to the B2B accounts. To set a ImmutableId to an existing Azure AD identity all you need is the AzureAD Powershell module:  

```powershell
PS> Get-AzureAdUser -UserPrincipalName j.doe_contoso.com#EXT#@fabrikam.onmicrosoft.com | Set-AzureAdUser -ImmutableId <string>
```

But of course you have to not just pick a random string, you have to use a value that will be linking the corresponding on-prem account with this particular Azure B2B identity. If you not have changed the default AAD Connect setup, normally the on-prem ObjectGUID is the default source anchor. But we have to convert this value in a base64 representation. You can use the following lines of code to do the job

```powershell
PS> $AzureB2BUpn = 'j.doe_contoso.com#EXT#@fabrikam.onmicrosoft.com'
PS> $OnPremUsername = 'jdoe'

PS> $UserGuid =  ([GUID](Get-ADuser $OnPremUsername -Properties ObjectGUID).ObjectGUID).ToByteArray()
PS> $ImmutableID = [System.Convert]::ToBase64String()

PS> Get-AzureAdUser -UserPrincipalName $AzureB2BUpn | Set-AzureAdUser -ImmutableId $ImmutableId
```

After setting the ImmutableId to the B2B identity your next AAD Connect sync cycle will import this account and you should see an Add into the Metaverse.

**Adding logic to avoid accidential sync of external before an Azure B2B account is prepared**  
The next part in the puzzle is, we have to etablish a process in AD/ AAD Connect that only exports external users when a corresponding Azure B2B object is available and prepared with an immutableId. For this I recommend the usage of two attributes we use for the flow. One should be an attribut that helps us identify external user accounts. Maybe you already have something in place for that. The second one will be populated after the Azure B2B preparations are done. For this post I will use the following two attributes:

* employeeType
* customAttribute1

With this two attributes we can now create a custom rule in AAD Connect that will make sure that external users that have no corresponding and prepared Azure B2B identity get not synced. The details why this is important will covered later on.

To filter out these users we can use the build in Metaverse attribute cloudFiltered. If this value is True for a given user the rules shipping with AAD Connect will not Export the user to Azure AD. So creating create a new Inbound rule:

```text
Basic settings  

Name: In from AD – Cloud filter Non-Employee
Connected system: AD
Connected system type: user
Metaverse type: person
Link type: Join
Precedence: <100
Scoping filter

employeeType EQUALS ‘External’ && CustomAttribute1 ISNULL 
Tranformations

Flow type: Constant
Target attribute: cloudFiltered
Source: True
```

So what this rules does is setting the attribute cloudFiltered to True for all users that have employeeType ‘External’ and not set a value for CustomAttribute1. In reverse this means, as soon as our preparation is done and we write back something into customAttribute1 the sync and export will start to flow exactly like in this illustration:


### Advantages of Hybrid Azure B2B identites

In that moment AAD Connect established the attribute flow between on-prem and Azure AD you instantly get a lot of benefits. This list is not a complete list of features. Instead I just wanted to highlight some of them.

* **Lifecycle** Most companies have some sort of working lifecycle to their accounts in Active Directory. So maybe some of your external user accounts gets deactivated or deleted automatically. After connecting the B2B object to your external AD accounts you will extend this lifecycle to the B2B object. Something a lot of my customers demand.
* **Mail attributes** If your on-prem accounts are mail enabled and part of distribution groups, AAD connect will now reflect this to the Azure B2B object.
* **Attributes** Most companies maintain some attributes of their external accounts. Beside first and last name you will find phone numbers, managers and address attributes to be filled correctly. Now these attributes getting applied to the Azure B2B user and you have one source of truth.
* **Azure AD app proxy** If you are running on-prem services like a web application running on IIS you now can publish this app through the app proxy in Azure AD. With this setup you are now able to let your Azure B2B users access these application with SSO! This is possible through Kerberos constraint delegation. How to set this up is described at the end of this page

### Two-leg authentication

One thing I have not mentioned before but it is really awesome: With the Hybrid Azure B2B user you get something I call the two-leg authentication. The two-leg authentication or TLA is basically a nice side effect of this setup – but lets explain this a little bit more in detail.

When you create an guest user invite and this invitation is redeemed by the user Microsoft will stamp an alternateSecurityId to the Azure B2B object in your tenant. This is the link for the object between your tenant and the home tenant of the Azure B2B user:


**The actual values are not show like this – just for explanation purpose**  

So every time a user authenticates in the home tenant and switch to your tenant to access a shared document, Microsoft will map the user with the help of this attribute. Currently there is no way to manually update or remove the alternateSecurityId that is pointing to the source user. This is why it so important to first create the Azure AD B2b object by invitation. This is currently the only way to stamp an alternateSecurityId onto it!

But hey – that is no a “two-leg”, right? Absolutely! But in that moment, we link our on-prem account to the Azure B2B identity a second way of authentication will be established. And it does not matter how you are authenticating against Azure AD – this works for all three scenarios: ADFS, Password hash sync and Passthrough authentication!


So to make things clear: You now have a super charged user in Azure AD that can authenticate from two sides: Through an username and password that is provided by you and through the Azure B2B authentication process. Cool, isn’t it?

But why we should have two ways of authentication you may think? One big benefit of this approach is for a merge or acquisition scenario. Before the users are migrated you start collaborating with them through Azure B2B. When time goes on, you link them as described. The user now can access Office 365 through another way without loosing their access rights or data (e.g. Teams 1:1 chats are preserved). If you like you can even strengthen the collaboration more by setting the userType from Guest to Member (You learned this in the warming up at the beginning of this post). If you successfully migrated the user you can raise a ticket at Microsoft to remove the alternateSecurityId at the end. So actually the hybrid Azure B2B is an awesome way to migrate users from one Azure AD tenant to another!

## Polish the process

To make this whole hybrid B2B process more reliable you should setup a process that will deal with onboarding new external accounts on-prem. I personally create an Azure B2B identity for every new external user automatically. By using GraphAPI / invite manager API you can build a logic to invite a user without actually sending an invitation mail every time. With this flow you can pre-stage your Azure AD object to be onboarded later via Azure B2B. A sample call to the invitation API could look like the following code (I will not cover how to obtain an access token – there are a ton of blogs covering this task 🙂 ). If you want to dig deeper into the invitations API you will find everything in the official docs from Microsoft.

```powershell
$HeaderParameters = @{
   "Authorization"="Bearer $(GetAccessToken)"
} 

#Specify the URI to call and method
$InviteApiUri = "https://graph.microsoft.com/v1.0/invitations"
$method = "POST"
$body = @{
    "invitedUserEmailAddress"= "john.doe@contoso.com",
    "sendInvitationMessage": false
}

# Run Graph API query

$query = Invoke-WebRequest -Method $method -Uri $uri 
   -ContentType "application/json" -Headers $HeaderParameters `
   -Body ($body | ConvertTo-Json)
```

After creating the B2B object, the invoke will give us the object id of the newly created object. This can easily be written back to the on-prem AD account

```powershell
$B2bObjectId = ($query.Content | ConvertFrom-Json).invitedUser.Id


Set-AdUser -Identity jdoe -Replace @{customAttribute1=$B2bObjectId}
```

## Special scenario: Cloud only / no AAD Connect

In case your tenant is already 100% cloud and you do not have an AAD Connect you can still pimp your Azure B2B identities! In this case I am sure you already have some sort of user lifecycle in place. If you want to provide an alternate way to sign into your tenant you can just replace the UPN and set a password for your Azure B2B users with the AzureAD PowerShell Module

```powershell
PS> $B2bUser = Get-AzureAdUser -UserPrincipalName j.doe_contoso.com#EXT#@fabrikam.onmicrosoft.com

PS> $B2bUser | Set-AzureAdUser -UserPrincipalName 'john.doe@fabrikam.com'
PS> Set-AzureADUserPassword -ObjectId $B2bUser.ObjectId -Password ('MyPassword' | ConvertTo-SecureString -AsPlainText -Force)
```
