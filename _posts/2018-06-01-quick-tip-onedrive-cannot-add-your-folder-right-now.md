---
layout: post
title: 'Quick tip: Onedrive cannot add your folder right now'
date: 2018-06-01 17:16:06.000000000 +02:00
type: post
published: true
categories:
- Office
- Quick tip
- OneDrive
tags:
- migrated
- issue
---

Last week I came across a problem with the Ondrive for Business NGC  was throwing the error "Onedrive cannot add your folder right now"

![472e7945-6cb0-4d1c-be7e-62993a6ad6f5]({{ site.baseurl }}/assets/migrated/2018/06/472e7945-6cb0-4d1c-be7e-62993a6ad6f5.png)

I was mislead by this error and was thinking it is client related. At the end the solution was almost to easy - for this tenant we have client restrictions for synchronization enabled. Only domain joined devices from specified domains are allowed to sync data. Some of you already know that the environment I am working in has a lot of domains, the particular domain was not included in the list.

You can check for the current domain GUIDs with the SharePoint Online Powershell module:

```powershell 
Get-SPOTenantSyncClientRestriction | Select-Object -ExpandProperty AllowedDomainList

Guid  
----  
cd4eb0e7-c5a2-4734-afd9-931fc6dbf285  
ce3b35e7-394b-4a60-b317-6a5021872993  
63b6ced5-91c5-4672-b248-e1fba2027aa6  
2e4b1128-db58-4fd9-8175-590cf0fef209  
``` 

You can add another domain to the list with

```powershell 
$ADL = (Get-SPOTenantSyncClientRestriction).AllowedDomainList  
$ADL.Add((Get-ADDomain -Identity lab01.treegardner.de).ObjectGUID)

Set-SPOTenantSyncClientRestriction -DomainGuids $ADL
```

If your list is very long (the maximum number of domain GUIDs allowed are 125!) you should check for duplicates upfront before adding the new GUID

```powershell
$NewGUID = (Get-ADDomain -Identity lab01.treegardner.de).ObjectGUID

If ($GUIDs -contains $NewGUID) {  
    	Write-Host "The GUID $NewGUID is already in the list of allowed domains!"  
} Else {  
    $GUIDs.Add($NewGUID)  
}
```

Another tipp I can give you if you are dealing with a lot of domains: maintain a source list in a format where you can look it up more easily. A simple CSV file should do the trick:

```csv
domainFQDN,domainGUID,status  
lab01.treegardner.de,cd4eb0e7-c5a2-4734-afd9-931fc6dbf285,added  
domain.local,ce3b35e7-394b-4a60-b317-6a5021872993,added  
company.lan,63b6ced5-91c5-4672-b248-e1fba2027aa6,removed  
lab02.treegardner.de,2e4b1128-db58-4fd9-8175-590cf0fef209,added
```

You can than easily built your GUID list and set this for the AllowedDomainList:

```powershell
$Data = Import-Csv -Path .\alloweddomainlist.csv -Delimiter ','  
$GUIDs = @()  
Foreach ($Domain in $Data) {  
    If ($Domain.status -eq "added") {  
        $GUIDs += [guid]$Domain.domainGUID  
    }  
}

Set-SPOTenantSyncClientRestriction -DomainGuids $GUIDs
```