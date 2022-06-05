---
title: Convert csv data to yaml using Powershell
description: While migrating some photo gallery data from my old website to Hugo, I wanted to convert some exported csv data to yaml for using in Hugo content.
date: 2022-06-04
tags:
  - powershell
  - csv
  - yaml
layout: layouts/post.njk
---

While migrating some photo gallery data from my old website to Hugo, I wanted to convert some exported csv data to yaml for using in Hugo content.

I found this post helpful - [Hugo: Insert data into content with a shortcode](https://input.sh/hugo-data-into-content-with-a-shortcode/). But my use case was slightly different. I was creating a new layout while trying to reuse an existing `partial` template in a Hugo theme.

## Quick Script

I installed powershell-yaml module from PSGallery.

```powershell
Install-Module -Name powershell-yaml

Import-Module powershell-yaml
```

Import csv data. My data didn't have a header and had some fields I didn't care about.

```powershell
$header = 'setId', 'setTitle', 'setKey', 'setUrl', 'setLink', 'lastUpdated', 'lookup', 'photoId', 'photoPath', 'photoDate', 'photoTitle', 'photoDescription', 'isHidden'
$data = Import-Csv -Path .\redmond.csv -Header $header
```

Create a Hash Table and append only the relevant data from csv data.

```powershell
$pictures = @()
foreach ($row in $data) {$pictures += @{ 'title' = $row.photoTitle; 'description' = $row.photoDescription; 'image' = $row.photoPath; 'thumb' = $row.photoPath;}}
$yml = @{ 'title' = "Hawaii"; 'style' = "style1 medium lightbox onscroll-fade-in"; 'content' = "<em>Dec 2010. Redmond, WA, USA.</em>"; 'pictures' = $pictures}
```

Convert to Yaml and output to a file

```powershell
ConvertTo-Yaml $yml | Out-File hawaii.yml
```

## References

Some references I used for my setup

+ <https://www.powershellgallery.com/packages/powershell-yaml/0.4.2>
+ <https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/import-csv>
+ [Modify YML files with PowerShell](https://mjc.si/2020/10/08/modify-yml-files-with-powershell/)
+ [Working With Powershell Objects to Create Yaml](https://dev.to/sheldonhull/working-with-powershell-objects-to-create-yaml-2kp0)
+ <https://stackoverflow.com/questions/10655788/powershell-set-content-and-out-file-what-is-the-difference>
