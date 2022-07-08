---
layout: single
title:  "Azure App Service and IIS based reverse proxy (Part 2)"
date:   2022-07-08 12:00:00 +0200
categories: update azure devops web
author: Nilsas Firantas
teaser: "/assets/images/app_service_proxy_teaser.png"
---

## Azure App Service and IIS based reverse proxy - Part 2

### General

This is a second and final part in making an IIS based reverse proxy in Azure.
If you've missed the first part please refer to [this article]({% post_url 2022-03-28-azure-app-service-and-iis-reverse-proxy %}) to setup your infrastructure.

Once we have the App Service built we need a `web.config` to tell IIS under the hood what to do and `applicationHost.xdt` to tell it which additional features to enable.
Let me explain that.

### Web configuration

Most of you who worked with IIS and ASP.NET will be familiar with `web.config` an XML tag based configuration for your web app.
IIS being a full featured web server has a lot to offer and it's configuration can be manipulated by editing `web.config`.
For IIS to read you `web.config` it should be present in your application root directory (Yes, there are cases when this is not entirely true but I'm not discussing those cases).

Meat and potatoes are here, this will tell the `rewrite` module to change the request URL to the one provided in the configuration.

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <rewrite>
      <rules>
        <rule name="Probe" stopProcessing="true">
          <match url="(.*)" />
          <action type="Rewrite" url="devoops.lt/{R:1}" appendQueryString="true" logRewrittenUrl="false" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
```

For example if I'd deploy this configuration on `contoso.com` and you would try to access `contoso.com` you would get redirected to `devoops.lt` which is the whole idea.
The `{R:1}` means that anything you pass as URL param will get passed to the proxied web address as well. For example:
Accessing `contoso.com/my/little/pony` will translate directly to `devoops.lt/my/little/pony` this is great for handling many scenarios.
But of course you can drop it and just redirect all requests straight to the one in the configuration.

Rule name may be whatever you like mine is `Probe` just for this example.
Obviously you may have as many rules as you like (at least I haven't seen any limitations, do let me know if you found any).
Different rules may target different URLs using different match patterns, in case you haven't yet figured that out, Yes that is Regex.
I will not go over configuration in depth on this article, but if you are struggling to message and we'll try to figure it out.

So upload `web.config` restart app service and voila right? Right?

Not so fast, we still need to enable `rewrite` module as by default it is not enabled in App Service.
They probably didn't intend that some people would just go in and try to bend stuff that way.

### Application Host configuration

It seems that this is Microsoft's well guarded secret as documentation on how to use it is scarce.
But when you think of App Service as an old school WIndows Server hosting tons of web apps on it's IIS web server it all makes sense.
You just have to adapt some of the knowledge to the Cloud, so what am I going on about? This:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration xmlns:xdt="http://schemas.microsoft.com/XML-Document-Transform">
  <system.webServer>
    <proxy xdt:Transform="InsertIfMissing" enabled="true" preserveHostHeader="false" reverseRewriteHostInResponseHeaders="false" />
  </system.webServer>
</configuration>
```

Now for `applicationHost.xdt` to work it has to be deployed one layer above the application root, meaning we'll have to do some magic.
I'm not exactly a fan of doing things manually, but just incase it is fine with you,
you can just browse the FTP of your APP Service and place `web.config` in the root folder of your app and `applicationHost.xdt` in the directory above it.

I wanted to do this automatically as it's much safer and reliable, also whenever I need to change the config or add something new to the proxy
I can follow a simple model of submitting my changes to Git and CI pipeline making my changes to the proxy without me having to find URLs, Credentials, Files etc.

Enough ranting, first we need to login to Azure to pull App Service publish profile.
I'm doing this from the same pipeline that runs Terraform so I already have a reference to my credential variables.

This is how my job in `azure-pipelines.yml` looks like:

```yaml
- job: deploy
  dependsOn: apply
  variables:
    app_service_id: $[ dependencies.apply.outputs['outputs.app_service_id'] ]
  steps:
    - task: PowerShell@2
      displayName: 'Deploy Proxy'
      inputs:
        targetType: filePath
        filePath: $(Build.Repository.LocalPath)/proxy/DeployProxyConfig.ps1
        arguments: -AppServiceID $(app_service_id)
        pwsh: true
        workingDirectory: $(Build.Repository.LocalPath)/proxy
      env:
        ClientID: $(ClientID)
        ClientSecret: $(ClientSecret)
        TenantID: $(TenantID)
```

Nothing too crazy here, the somewhat more advanced feature used here is referencing variables from other job named `apply` which executes `terraform apply`.
To set `terraform output` as pipeline variables you can use Azure Pipelines task also named `Terraform Outputs` 
or just grab a the output value from `terraform` by using `terraform output app_service_id` (obviously you need to define an output with that name in Terraform to use it.)
and set it as Pipeline variable by using special Azure Pipelines syntax 

```powershell
Write-Host ##vso[task.setvariable variable=app_service_id;isOutput=true]The_Value_You_Grabbed
```

Let me know if there's a need for a tutorial here how to pass variables within the Pipeline.

Okay now that we have all variables, we need some PowerShell magic to get us going.

Again nothing too crazy, Let's grab the App Service. Please don't forget to write your PowerShell properly so that the next person can read and understand it as well :) Thanks.

```powershell
[CmdletBinding()]
param (
    [Parameter(Mandatory)]
    [string]$AppServiceID # We passed this as an argument in the Pipeline snippet above
)

Begin {
    # Connect to Azure
    az login --service-principal -u "$env:ClientID" -p "$env:ClientSecret" --tenant "$env:TenantID"
    # Get Publish profile and parse it from JSON
    $profiles = (az webapp deployment list-publishing-profiles --ids $app_service_id) | ConvertFrom-Json
    # Extract connection information from publishing profile
    $username = ($profiles | Where-Object {$_.publishMethod -eq "FTP"}).userName
    $password = ($profiles | Where-Object {$_.publishMethod -eq "FTP"}).userPWD
    $url = ($profiles | Where-Object {$_.publishMethod -eq "FTP"}).publishUrl
}
```

I should note that while it is possible to use `Az` module in PowerShell,
here I used `az-cli` as it returns plain JSON instead of XML. 
Not to say that you can't parse `XML` with PowerShell but IMO it's just easier to go with `JSON`.
Together with that `az-cli` is easier to login to while using service principal I have a winner,
however this all can be implemented in pure PowerShell if needs be.

Okay now that we have FTP credentials we need to upload our files to it, some more PowerShell 💪

```powershell
function Upload-File {
    param (
      $uri,
      $username,
      $password,
      $filePath
    )
    $request = [System.Net.FtpWebRequest]([System.net.WebRequest]::Create($uri))
    $request.Method = [System.Net.WebRequestMethods+Ftp]::UploadFile
    $request.Credentials = New-Object System.Net.NetworkCredential($username,$password)
    # Enable SSL for FTPS. Should be $false if FTP.
    $request.EnableSsl = $true;
    # Write the file to the request object.
    $fileBytes = [System.IO.File]::ReadAllBytes($filePath)
    $request.ContentLength = $fileBytes.Length;
    $requestStream = $request.GetRequestStream()
    try {
        $requestStream.Write($fileBytes, 0, $fileBytes.Length)
    }
    finally {
        $requestStream.Dispose()
    }
    Write-Host "Uploading to $($uri.AbsoluteUri)"
    try {
        $response = [System.Net.FtpWebResponse]($request.GetResponse())
        Write-Host "Status: $($response.StatusDescription)"
    }
    finally {
        if ($null -ne $response) {
            $response.Close()
        }
    }
  }
```

The above function is a sort-of native FTP client.
Well at least without external libraries like WinSCP and such.
However it uses some `.NETCore` namespaces.
This does expand the list of requirements if you have Powershell installed on our machine it already has `.NETCore`.
As far as I know PS doesn't have anything more native in it's core to handle FTP connections.

We have the function that will do the heavy lifting, we have the credentials to use, and we have the configuration files we need, now to just to wrap it inside the script.

```powershell
Process {
    $webConfigFile = Get-Item -Path "./web.config" -ErrorAction Stop
    $applicationHostFile = Get-Item -Path "./applicationHost.xdt" -ErrorAction Stop

    $webConfigUri = New-Object System.Uri("$url/$($webConfigFile.Name)")
    $applicationHostUri = New-Object System.Uri("{0}/{1}" -f $url.Substring(0, $url.LastIndexOf("/")), $($applicationHostFile.Name))
    Upload-File -uri $webConfigUri -username $username -password $password -filePath $webConfigFile
    Upload-File -uri $applicationHostUri -username $username -password $password -filePath $applicationHostFile
}
```

The only interesting part of this snipped probably is how I get the URI for `applicationHost.xdt`.
So I know that variable `$url` which I get by parsing App Service Publish Profile leads me to the application root folder.
I've mentioned previously in this article that for `aplicationHost.xdt` to work it needs to be one directory above application root directory.
That is why I use PowerShell's `substring` function and shorten the URL to the last `/` slash.
That way I loose the last segment of URL and now have the needed directory URL.
Not particularly elegant, I'll update when I'll think of a better way.

If you did this not at the same time as `Terraform` code deployment, now you need to `Stop` and `Start` your App Service (`Restart` sometimes is not enough to pick up new files).
