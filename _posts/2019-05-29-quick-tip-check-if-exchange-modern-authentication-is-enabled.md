---
layout: post
title: 'Quick tip: Check if Exchange modern authentication is enabled'
date: 2019-05-29 21:20:38.000000000 +02:00
type: post
published: true
categories:
- Authentication
- Exchange
- Quick tip
tags:
- migrated

---

In my day to day business I often need to know if a tenant or an on-premise Exchange 2016 environment is enabled for modern authentication. Most of the time I need this information at a point in time, where I do not have access to the customers Exchange (Online) environment - and most of the time even the customer does not know if the tenant or the on-premise environment are running modern or not.

For Exchange Online this is often an issue, because older Tenants had the modern authentication turned off by default. Only tenants created at the end of 2017 or later have it enabled by default. But to check if this is still the case is always a good idea.

With this little script, you can check Exchange Online and every Exchange on-premise where you can reach the EWS endpoint if modern authentication (OAuth2) is enabled or not, without having any credentials!

It is important to understand for on-premise Exchange Server: This method will not tell you if everything is set up correctly for Hybrid modern authnetication to work - but it is a starting point :-)

## How does it work?

Ok, sounds like magic to check a setting without even have credentials? No - not really - we just use the fact, that trying to send a request for modern authentication to Exchange will be handled different depending if modern authentication is enabled or not.

The endpoint we are using is the EWS endpoint. For Office 365 this is always

**https://outlook.office365.com/EWS/Exchange.asmx**

If you are sending a simple POST request to this endpoint with some defined headers you can actually learn what configuration is set. The following headers must be sent to EWS

    "Authorization"="Bearer"

You do not need an existing user to put in the X-User-Identity header. But the domain is important part. If you want to check the tenant contos.com you have to provide a username something@contoso.com

In my script i will just generate a GUID for the local part.

If you send these headers as a POST request to the O365 EWS endpoint, Exchange will tell us if it can deal with bearer authentication or not. Exchange should answer this request with a 401 unauthenticated response - but that is not the part we are interested. We have to look more closely into the response headers. If there is a header x-ms-diagnostics send back to our client, chances are high that modern auth is disabled. But to be 100% sure we have to evaluate the value in this header. If it tells us, that "flighting is not enabled" you know for sure, that this tenant/Exchange on-premise is not configured for handling modern auth!

## Using Invoke-WebRequest in PowerShell

If you want to do this check in PowerShell you will mostly use invoke-webrequest. This is fine as long as you know how to receive the response headers. Because the invoke-webrequest will throw an error if Exchange requires authentication (401) we can not get directly to the header. So we have to wrap up the request in a try-catch-block:

```powershell 
#We want to use a random username - not trying to hit a real user.
$randomUser = [GUID]::NewGuid().ToString()
$Domain = '<enter your domain name here - eg. contoso.com>'

$headers = @{
    "Authorization"="Bearer"
    "Content-Type"="text/xml"
    "X-User-Identity"="$randomUser@$Domain"
    "X-EWS-TargetVersion"="2016_11_20"
}

$uri = "https://outlook.office365.com/EWS/Exchange.asmx"

$error.clear()

Try {
    $result = Invoke-WebRequest -Uri $URI -Headers $headers -Method Post

} Catch [System.Net.WebException]{

    #... with calling the Response in the System.Net.HttpWebResponse class.
    [System.Net.HttpWebResponse]$result = [System.Net.HttpWebResponse] $_.Exception.Response
    $httperrorcode  = $result.StatusCode.Value__

    #we are interessted in the 401 NotAuthenticated
    If ($httperrorcode = 401) {

        Write-Host "Server returned 401 - lets take a look into response headers" -ForegroundColor Yellow

        If ($result.Headers -contains "x-ms-diagnostics") {
            Write-Host "x-ms-diagnostics header found. Looking if flighting is enabled...."

            $diagnosticsHeaderContents = ($result.Headers["x-ms-diagnostics"]).Split(";")

            Foreach ($item in $diagnosticsHeaderContents) {
                If ($item -like "*flight*") {
                    Write-Host $item -ForegroundColor Yellow
                }
            }
        } Else {
            Write-Host "No x-ms-diagnostics header found. Looks like modern auth is active" -ForegroundColor Green
        }
    }
}

## Finally

So next time you quickly want to know, if a Office 365 tenant has enabled modern authentication or not, you can check this setting without any credentials. The only thing you need to know is one of the configured domains that is used.