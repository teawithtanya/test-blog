---
title: Setting up Hugo static site on Azure Storage.
description: My Hugo workspace setup.
date: 2022-02-20
tags: hugo
layout: layouts/post.njk
---

Following [Hugo guide](https://gohugo.io/getting-started/) to setup Hugo on my dev workstation.

## Install Hugo

+  Downloaded [version 0.92.2](https://github.com/gohugoio/hugo/releases) Windows 64-bit binary
   +  Needed the 'extended' version to build some features
+  Updated `PATH` using PowerShell

```powershell
$Env:PATH += ";C:\Hugo\bin"
```

## Create a new site with a custom theme

Create a new Hugo site in folder HugoSite

```powershell
hugo.exe new site HugoSite
```

Download a custom theme from github

```powershell
cd HugoSite
git init
git submodule add https://github.com/caressofsteel/hugo-story.git themes/hugo-story
# overwrites default data and config from theme
cp -r -force themes/hugo-story/exampleSite/data ./
cp themes/hugo-story/exampleSite/config.toml ./
```

Test local changes

```powershell
hugo server -D
```

Generate static web pages in folder 'public'

```powershell
hugo -D
```

Pushed all local changes to private Git repo 'HugoSite'

## Azure Static Web App vs Azure Storage

For minimal overhead, I like Azure Statis Web Apps. But for finer control over web content it may be better
to build the static website locally, do any local tweaks and then upload the static website. I documented
both approaches I tried here.

## Alternateive 1.1: Setup an Azure Static Web App

Used [Azure Static Web Apps tutorial](https://docs.microsoft.com/en-us/azure/static-web-apps/publish-hugo) to setup a Hugo app

+ Created a 'Static Web App' resource
+ Picked a resource group
+ Name: 'my-hugo-site'
+ Hosting Plan Type: Free
+ Deployment Source: GitHub
+ Repository: HugoSite
+ Branch: main
+ Build Presets: Hugo
+ Clicked Review+Create > Create

## Alternative 1.2: Setup a Custom Domain with Azure Static Web App

Once the static website was created, I setup a [custom domain](https://docs.microsoft.com/en-us/azure/static-web-apps/custom-domain-external) to point to it.

+ Added a CNAME record for blog.mysite.com to URL found in 'Overview' of Azure Static Web App.
+ Under Settings > Custom Domains, I clicked Add to add the custom domain 'blog.mysite.com'.
+ Click Next and Azure validated ownership of this custom domain.

## Alternative 2.1: Enable static web hosting in Azure Storage

Ensure latest PowerShell 7 or higher

```powershell
$PSVersionTable.PSVersion
```

Set a permissive execution policy for PowerShell

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

Ensure Azure Module is installed

```powershell
Install-Module -Name Az -Scope CurrentUser -Repository PSGallery -Force -AllowClobber
```

Verify installed Azure Module

```powershell
Get-InstalledModule -Name Az -AllVersions | select Name,Version
```

Sign in to Azure account

```powershell
Connect-AzAccount
```

List Azure subscriptions

```powershell
Get-AzSubscription
```

Set context to an active subscription

```powershell
$context = Get-AzSubscription -SubscriptionId $subscriptionId
Set-AzContext $context
```
List Resource Groups

```powershell
 Get-AzResourceGroup
```

Choose a resource group to use and get storage accounts associated with it

```powershell
Get-AzStorageAccount -ResourceGroupName $resourceGroupName
```

Choose a storage account to use and set context

```powershell
$storageAccount = Get-AzStorageAccount -ResourceGroupName $resourceGroupName -AccountName $storageAccountName
$ctx = $storageAccount.Context
```

Enable static web hosting

```powershell
Enable-AzStorageStaticWebsite -Context $ctx -IndexDocument "index.html"
```

## Alternative 2.2: Upload file to Storage Account

Upload all files modified recently by Hugo

```powershell
cd public
Get-ChildItem -File -Recurse | Where-Object {$_.LastWriteTime -gt (Get-Date).AddDays(-2)} | Set-AzStorageBlobContent -Container `$web -Context $ctx
```

This doesn't set the correct Content Type though. I used [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-windows) instead.

```powershell
cd public
az login
az storage blob upload-batch -s . -d '$web' --account-name $storageAccountName
```

Get public url of the static website

```powershell
Write-Output $storageAccount.PrimaryEndpoints.Web
```

## Alternative 2.3: Create Azure CDN with custom domain

Create CDN profile

```powershell
New-AzCdnProfile -ProfileName CdnBlog -ResourceGroupName $resourceGroupName -Sku Standard_Microsoft -Location "Central US"
```

Create CDN Endpoint

```powershell
New-AzCdnEndpoint -ProfileName CdnBlog -ResourceGroupName $resourceGroupName -Location "Central US" -EndpointName "myblogep" -OriginName "myblog" -OriginHostName "myblog.web.core.windows.net"
```

Get newly created endpoint

```powershell
 $endpoint = Get-AzCdnEndpoint -ProfileName CdnBlog -ResourceGroupName $resourceGroupName -EndpointName "myblogep"
```

Add a CNAME record to point myblog.example.com to point to the new endpoint hostname

Validate the custom domain mapping

```powershell
$result = Test-AzCdnCustomDomain -CdnEndpoint $endpoint -CustomDomainHostName "myblog.example.com"
```

Create custom domain on CDN Endpoint

```powershell
If($result.CustomDomainValidated){ New-AzCdnCustomDomain -CustomDomainName MyBlog -HostName "myblog.example.com" -CdnEndpoint $endpoint }
```

I enabled CDN Managed Custom Domain HTTPS via Azure Portal.

I had to update Origin settings by setting both HTTP and HTTPS to 443 (since the Storage static website only had HTTPS enabled I guess).
