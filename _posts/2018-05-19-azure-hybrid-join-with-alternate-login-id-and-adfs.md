---

title: Azure hybrid join with alternate login id and ADFS
date: 2018-05-19 19:57:40.000000000 +02:00
published: true
categories:
    - Workplace
tags:
    - migrated
---

A cool feature when you are dealing with Office 365 and Azure AD and you also still have a lot on-prem stuff in your business is to hybrid join your devices. Your Windows device is joined not only into the old fashioned Active Directory but also into Azure AD simultaneously. There are several reasons why you should consider to join the device also into Azure AD:

*   If you plan to manage the devices with Intune
*   If you want to give a real SSO to your users when accessing Application protected through Azure AD (for example O365 apps like Outlook or SharePoint Online)
*   If you want to be more flexible when using conditional access

In my case the SSO was one of the major reason why I was playing around with this for some time now. Unfortunately I stumbled across a nasty problem with exactly this feature.

So as always in such a case I start to read all the documentation, look into the event logs and try to search within the community to see if anyone else came across the same problem.

One very good resource (and sadly the only one except the official documentation) is the blog of Jairo Cadena ([https://jairocadena.com](https://jairocadena.com)). Reading the information how hybrid join is working behind the scenes was very helpful.

But even with this knowledge I was not able to solve the issue by myself - but lets start from the beginning....

#### Let me explain the issue

We have a registered and federated domain @company.com that we are using for the username in Azure AD and we have a different, non public UPN suffix @domain.local in our on premise Active Directory.

After successfully logging into the hybrid joined computer with an Active Directory user (ex. john@domain.local) running the command

```powershell
dsregcmd /status
```
I always get an odd result, that two important values in the user state, wich are signaling if SSO is working or not, are saying

```powershell
WamDefaultSet : NO  
AzureAdPrt : NO
```

There is only one [Microsoft article](https://docs.microsoft.com/en-us/azure/active-directory/device-management-troubleshoot-hybrid-join-windows-current) that gives you a clue what the reason could be - but no definitive answer how to find out what the actual problem is.

*   Bad storage key (STK) in TPM associated with the device upon registration (check the KeySignTest while running elevated).
*   Alternate Login ID
*   HTTP Proxy not found

I was a little surprised about the second point - alt-login-id. Does this mean, if you are using alternate login id in your environment, single sign on is not working at all? I could not believe this and so I asked some guys in my network who are more experienced than I am and they said to me that it should work also with alt-login-id. Ok so now I am confused and also excited to solve this problem in my configuration ;-)

Always a good start when dealing with this sort of problem is to install Fiddler and Wireshark. But this time capturing traffic was not so easy - because also LOCAL SYSTEM is involved making calls to public and ADFS endpoints

### Setup the lab

I don't want to go into every detail how to setup Fiddler and Wireshark but I will give some valuable tips to setup up your environment right. First grab the following tools

*   Fiddler
*   Wireshark (you can also use netsh trace and Microsoft Message Analyzer - I prefer Wireshark)
*   psexec from the Sysinternals Suite

Install all the tools and extract the psexec executable to some location of your choice

Before we start to capture the traffic we have to prepare the environment to pass through Windows authentication to ADFS correctly. This is only needed if Fiddler is running on the same machine the requests are coming from. There is a security measure in place called LoopbackCheck - we have to disable it to get authentication through Fiddler to work. You will get more information about DisableLoopCheck at this [Microsoft support article](https://support.microsoft.com/en-us/help/926642/error-message-when-you-try-to-access-a-server-locally-by-using-its-fqd). When you have opened the registry go the following subkey:

```
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Lsa
```

and create a new DWORD Value. Give it the name **DisableLoopbackCheck a**nd set the value to **1**

Restart your computer to apply the change.

Afterwards you start a command shell as administrator, go to the extracted psexec and run the following command

```bash
psexec.exe -i -s cmd.exe
```

This will launch a cmd.exe in the context of SYSTEM. Next we have to point the system to use the fiddler proxy. So type into the new shell the following

```bash
netsh winhttp set proxy 127.0.0.1:888
``` 

Start Fiddler in the background and check that "Automatic authentication" and capture https traffic is enabled.

Back in the shell of local system run the commands

```bash
dsregcmd /leave
```

This will unregister the device from Azure AD. I had sometimes errors while leaving when capturing traffic with Fiddler. Just ignore them. Afterwards we run the registration again. I would suggest to increase the buffer values for the console window - otherwise you will not be able to scroll up again.

```bash
dsregcmd /debug
``` 

You should see something happen in your Fiddler and now you can excaimine the whole communication.

If you wan't to catch the traffic from user login just switch the user and let Fiddler run in the background.

### Analyzing the results

What I could observe with Fiddler was a difference in the behavior from what I have learned from Jairo's blog - the Cloud AP Azure AD plug-in for authentication was not going to ADFS diectly (to get the SAML token) but instead is pointing to login.microsoftonline.com and passing the on-prem UPN suffix to it.

```
GET https://login.microsoftonline.com/common/UserRealm/user%40domain.local?api-version=1.0 HTTP/1.1
```

Of course Azure AD does not know the user realm @domain.local and cannot proceed and redirect the user to our local ADFS STS-IDP. My first thoughts where that the hybrid join was not done correctly and so the local system is not pointing the plug-in directly to ADFS.

### The Service Connection Point

A key role in the hybrid join process plays the SCP you have to create in your Active Directory. This SCP with the well known guid 62a0ff2e-97b9-4513-943f-0d221bd30080 contains two keyword attributes:

*   azureADName:company.com
*   azureADId:72f988bf-86f1-41af-91ab-2d7cd051db47

AzureADName is one of your registered domains in your Azure AD tenant with the id stored in azureADId. With this information the Windows client knows to which tenant it belongs while trying to join Azure Ad.

When you analyze the registration process in Fiddler you will see the following communication.

First it is going to the enterpriseregistration endpoint and is putting the domain from the keyword azureADName in the request URI:

```
GET https://enterpriseregistration.windows.net/company.com/EnrollmentServer/contract?api-version=1.3 HTTP/1.1  
GET https://login.microsoftonline.com/common/UserRealm/user%40company.com?api-version=1.0 HTTP/1.1
```

So now login.microsoftonline.com knows that the user realm @company.com is a federated domain and so we get redirected to our local ADFS STS-IDP for authentication

```
GET https://sts.company.com/adfs/services/trust/mex HTTP/1.1  
POST https://sts.company.com/adfs/services/trust/13/windowstransport HTTP/1.1  
POST https://sts.company.com/adfs/services/trust/13/windowstransport HTTP/1.1
```

After successful authentication we pass the SAML token to the oAuth2 endpoint to receive our refresh and access tokens for the computer at the end

```
POST https://login.microsoftonline.com/common/oauth2/token HTTP/1.1  
... SNIP ...
```

So this looks very good - but why is SSO still not working and AzrueADPrt and WamDefaultSet still yelling no at me?

### The "solution"

At the end I gave up. So I contacted Jairo and he helped me out very quickly - thanks a lot again for the assist! In the end it is a problem in Windows 10 RS3 and below when it comes to alternate login id. Microsoft fixed the behavior in RS4 to use the SCP also for the user login. So instead just passing the UPN suffix to the login.microsoftonline.com endpoint the process now takes the value from the keyword AzureADName in the SCP as the user realm value.

That leaves one question open: Why I was told that alternate login id should work before RS4? Of course you can use alt-login-id but still have a registered UPN suffix. Why you do this? I only can think of one reason why you do this because you don't want to change your UPNs but still want to be able to use your email address as the username in Azure AD. But this only works if you are using public top level domain names as UPN suffix. In my scenario this is not the case.

UPDATE

This a dsregcmd status from a successful joined and SSO capable computer/user. You can use this to compare it to your status

```bash
dsregcmd /status

+----------------------------------------------------------------------+  
| Device State |  
+----------------------------------------------------------------------+

AzureAdJoined : YES  
EnterpriseJoined : NO  
DeviceId : 81c1a816-1306-xxxx-b788-yyyyyyyyyy  
Thumbprint : A712F0CB3A65B6B6CB08C83DF1C9FA79FC1BCE1C  
KeyContainerId : d1dc3688-xxxx-yyyy-zzzzz-0f4072ef5158  
KeyProvider : Microsoft Platform Crypto Provider  
TpmProtected : YES  
KeySignTest: : MUST Run elevated to test.  
Idp : login.windows.net  
TenantId : 98c8eb766-xxxx-yyyy-zzzz-26d4a9d3147  
TenantName :  
AuthCodeUrl : https://login.microsoftonline.com/98c8eb766-xxxx-yyyy-zzzz-26d4a9d3147/oauth2/authorize  
AccessTokenUrl : https://login.microsoftonline.com/98c8eb766-xxxx-yyyy-zzzz-26d4a9d3147/oauth2/token  
MdmUrl :  
MdmTouUrl :  
MdmComplianceUrl :  
SettingsUrl :  
JoinSrvVersion : 1.0  
JoinSrvUrl : https://enterpriseregistration.windows.net/EnrollmentServer/device/  
JoinSrvId : urn:ms-drs:enterpriseregistration.windows.net  
KeySrvVersion : 1.0  
KeySrvUrl : https://enterpriseregistration.windows.net/EnrollmentServer/key/  
KeySrvId : urn:ms-drs:enterpriseregistration.windows.net  
WebAuthNSrvVersion : 1.0  
WebAuthNSrvUrl : https://enterpriseregistration.windows.net/webauthn/98c8eb766-xxxx-yyyy-zzzz-26d4a9d3147/  
WebAuthNSrvId : urn:ms-drs:enterpriseregistration.windows.net  
DeviceManagementSrvVersion : 1.0  
DeviceManagementSrvUrl : https://enterpriseregistration.windows.net/manage/98c8eb766-xxxx-yyyy-zzzz-26d4a9d3147/  
DeviceManagementSrvId : urn:ms-drs:enterpriseregistration.windows.net  
DomainJoined : YES  
DomainName : DOMAIN

+----------------------------------------------------------------------+  
| User State |  
+----------------------------------------------------------------------+

NgcSet : NO  
WorkplaceJoined : NO  
WamDefaultSet : YES  
WamDefaultAuthority : organizations  
WamDefaultId : https://login.microsoft.com  
WamDefaultGUID : {B16898C6-xxxx-zzzz-yyyy-64D755DA8520} (AzureAd)  
AzureAdPrt : YES  
AzureAdPrtAuthority : https://login.microsoftonline.com/98c8eb766-xxxx-yyyy-zzzz-26d4a9d3147  
EnterprisePrt : NO  
EnterprisePrtAuthority :

+----------------------------------------------------------------------+  
| Ngc Prerequisite Check |  
+----------------------------------------------------------------------+

IsUserAzureAD : YES  
PolicyEnabled : NO  
PostLogonEnabled : YES  
DeviceEligible : YES  
SessionIsNotRemote : YES  
CertEnrollment : none  
AadRecoveryNeeded : NO  
PreReqResult : WillNotProvision
```

In my next post I will explain how to use alternate login id without using alternate login id :-)