---

title: Hybrid-Join the safe way with ClientSideSCP
date: 2019-07-16 08:28:56.000000000 +02:00
published: true
status: publish
tags:
  - migrated
categories:
  - Workplace
excerpt: If your organisation is transforming to Office 365 and wants to hybrid join
  Windows devices, this post will help you rolling out your first pilot devices without
  to worry for to much impacts. This approach also scales to a huge multi-forest environment.
---

If your organisation is transforming to Office 365 and starting to consume more and more of its services, sooner or later (I recommend sooner is better 😁) you will think about joining your domain-joined devices also to Azure AD. There are a lot of benefits you are getting from doing this which I do not want to cover in this post. Top reasons from my experience are:

*   smooth and secure way of authentication against Azure AD including SSO
*   better control for securing your Office 365 and Azure AD connected applications through more conditional access policies controls

So joining your devices to Azure AD is a good idea! How to do this is covered in countless documentations. The steps for a controlled rollout in most companies consists of two steps:

1.  Create a GPO to disable automatic join for Windows 10 and Windows Server 2016 for the majority of your devices. Only for a few pilot workplaces you want the GPO not be applied
2.  Add an SCP in your local Active Directory pointing to one of your registered domains and the Azure AD tenant ID

If you are supporting a complex and quite large Active Directory forest or you even have multiple forests/domains, setting up GPOs and waiting for them to be replicated and applied can give you some headaches.

The next hurdle oftens starts with the rights you need to create both the GPO and the SCP. Often the workplace team needs an domain or enterprise admin to assist in this task. And if you have more than one forest that should be part in your hybrid-join pilot maybe you have to talk to different IT guys from different subsidiaries to get the job done.

But there is a very easy way to hybrid-join single devices without even touching your on-premise Active Directory now! The so called ClientSideSCP method. The only two requirements you need to fulfill are

*   have local administrative rights on the machines you want to join. (You need to edit HKLM hive in the registry)
*   At least Win 10 1703 - better 1803 for different reasons (See my other blog: [https://treeforest.cloud/2018/05/19/azure-hybrid-join-with-alternate-login-id-and-adfs/](https://treeforest.cloud/2018/05/19/azure-hybrid-join-with-alternate-login-id-and-adfs/)). Microsoft is writing in their documentation, that you need at least Windows 10 1607 to get ClientSideSCP to work, but my experience is that 1607 did not work. (I had a customer with Win 10 LTSB devices along with LTSC and the the LTSC did not work)

Against the Microsoft documentation I recommend for the first rollout to not apply the registry keys by a GPO, but instead add them manually to eliminate any dependencies/issues with GPOs.

### Implementing ClientSideSCP

To have ClientSideSCP setup you have two (as a bonus I give you three 😊) options

*   **Method 1** Create a GPO with the registry keys, scope it to the pilot machines, wait for AD replication to happen and run gpupdate /force. I already wrote in the chapter before, that I do not recommend this way personally
*   **Method 2** Add the regkey and values by hand directly to the client(s)
*   **BONUS Method 3** Use my [script](https://github.com/backbone2k/setupClientSideSCP) that is doing the hard work for you (including collecting tenant id and domain name from Azure AD if you wish)

#### Method 2

just run regedit in elevated mode on your pilot devices and create the following key and add the two values:

```
Hive: **HKEY_LOCAL_MACHINE**  
Key Path: **SOFTWARE\Microsoft\Windows\CurrentVersion\CDJ\AAD**

Value1 name: **TenantId**  
Value1 type: **REG_SZ**  
Value1 data: The GUID or **Directory ID** of your Azure AD instance</pre>

Value2 name: **TenantName**  
Value2 type: **REG_SZ**  
Value2 data: One verified **domain name** in Azure AD 
```

#### BONUS Method 3

If you as lazy as I am, just grab my super-duper-setupClientSideSCP-Powershell-Script from [GitHub](https://github.com/backbone2k/setupClientSideSCP) and fire up an **elevated** Powershell 5+. If you want to run in full automatic mode you also need an Azure AD admin account and the AzureAD Powershell module.

Starting in auto-mode is as simple as this (I love simple things ;-) )

```powershell 
$ SetupClientSideSCP.ps1
```
The semi-automatic mode can be startet with the following command:

```powershell 
$ SetupClientSideSCP.ps1 -TenantId <YOUR AZURE AD TENANT ID> -TenantName <ONE VERIFED DOMAIN NAME>
```

If you want to find out more just call:

```powershell
$ Get-Help .\SetupClientSideSCP.ps1
```

#### Finally

**ADFS -** The [Microsoft documentation](https://docs.microsoft.com/en-us/azure/active-directory/devices/hybrid-azuread-join-control) states that you have to apply the reg key first to your ADFS server(s) (if you are using ADFS). Personally I do not fully understand and agree on this requirement and I cannot remember that this was documented before.

But if possible just hybrid-join your ADFS Server(s). Hybrid-joining Windows Server is only working for Windows Server 2016+ / ADFS 4.0+ (Windows Server 2012 and below cannot be hybrid joined).

**Restart -** After you have added the reg key you should restart your clients. The rest of the setup for a successful hybrid-join pilot is well documented by Microsoft and I assume you have covered these steps before. Just a quick checklist what must be in place:

*   If you are using a proxy for connecting to the internet the config must be applied through WPAD to your clients. If using proxy authentication this must be disabled for the needed endpoints to successfully join a device to Azure AD
*   Azure AD connect is setup correctly and syncing the your (pilot) clients. This is needed if you want the lifecycle for your devices synced from on-premise to the cloud.
*   If you are using non routable upn-suffixes or using alternate login id you should check the required Windows 10 Version is installed on your pilot machines