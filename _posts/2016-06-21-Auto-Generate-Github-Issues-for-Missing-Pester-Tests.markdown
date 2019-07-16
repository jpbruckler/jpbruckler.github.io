---
layout: single
title:  "Auto Generate Github Issues for Missing Pester Tests"
date:   2016-06-21 11:20:00
tags: [PowerShell, Github, REST]
---

I've finally gotten on the [Pester](https://github.com/pester/Pester)
bandwagon (you should too, coincidentally), and have started writing pester tests
for module functions. I even wrote a Pester test for just a regular old script.
Once you get used to the workflow, it does make development so much easier and
less error prone.

If you don't know what Pester is, and you're using PowerShell, you've probably
been under a rock, but here's some useful links anyway:

  * [https://github.com/pester/Pester/wiki/Pester](https://github.com/pester/Pester/wiki/Pester)
  * [https://blogs.technet.microsoft.com/heyscriptingguy/2015/12/16/unit-testing-powershell-code-with-pester/](https://blogs.technet.microsoft.com/heyscriptingguy/2015/12/16/unit-testing-powershell-code-with-pester/)
  * [http://www.powershellmagazine.com/2014/03/12/get-started-with-pester-powershell-unit-testing-framework/](http://www.powershellmagazine.com/2014/03/12/get-started-with-pester-powershell-unit-testing-framework/)
  * [https://www.simple-talk.com/sysadmin/powershell/practical-powershell-unit-testing-getting-started/](https://www.simple-talk.com/sysadmin/powershell/practical-powershell-unit-testing-getting-started/)

But I'm not here to preach about how awesome Pester is, and why you should use it.
I'm here to talk about generating issues for missing Pester tests.

The complete script lives in a [gist](https://gist.github.com/jpbruckler/91897130b501686720798f4223fbb4b2)
(embedded for your edification in its entirety below). It uses [Github's REST API]
to create issues for the given repo.

## Assumptions

I wrote this as a quick and dirty way to create issues. It works for the way that
I write modules, which is that I keep each function in it's own file, and tend
to group files in folders based on functionality.

The original script was used to interact with Github Enterprise, it's been modified
to work with Github.com. If you'd like to use it for a Github Enterprise repo,
you'll need to change `$BaseURI` to `https://[hostname]/api/v3` where hostname is
the hostname of your Github Enterprise server.

## Setup

There are 2 things you're going to need to run this script:

  1. A Personal Access Token
  2. An issues label called 'pester'

A personal access token (aka API key) can be generated from your Github profile
[settings page](https://github.com/settings/tokens). When you generate your token,
you'll be able to copy it to a clipboard. Be sure to save this in a file
somewhere safe (or more preferably in a password management program of some kind).
This is the last time that you'll be able to get your key. If you lose it, you'll
have to generate a new one, and revoke the old one.

The 'pester' label is optional, but if you don't want to set it up, remove or
comment out the line:

```
labels    = @('pester')
```

## Script Explanation

The first step is authentication to Github. In this script I'm using Basic
Authentication<sup>[1]</sup> which requires a username and a Personal Access
Token which can be created from your settings page. I'm going to call it an API
key for short. The username and API key is base-64 encoded and passed to Github
via an HTTP header. The snippet below shows how this is created.

For more explanation of what's going on here, [Trevor Sullivan](https://twitter.com/pcgeek86) does a great job
in his Channel9 video about [Automating Github REST API using PowerShell](https://channel9.msdn.com/Blogs/trevor-powershell/Automating-the-GitHub-REST-API-Using-PowerShell).

{% highlight powershell %}
$AuthToken = '{0}:{1}' -f $Username, $ApiKey
$AuthToken = [System.Convert]::ToBase64String([char[]] $AuthToken)
$Headers   = @{
                Authorization = 'Basic {0}' -f $AuthToken
              }
{% endhighlight %}

With that done, I next generate a list of files that are missing tests. This is
done with the `Get-FilesWithNoTests` function.

{% highlight powershell %}
function Get-FilesWithNoTests
{
    <#
    .SYNOPSIS
        Returns file names for .ps1 files that do not have a corresponding
        .tests.ps1 file.
    .DESCRIPTION
        For a given path, will find all .ps1 files that do not have a .tests.ps1
        file in a test directory.
    #>
    param(
        [Parameter(Mandatory)]
        [string] $ScanPath,

        [string] $TestsDirectoryName = 'Tests'
    )

    process {
        if (!(Test-Path $ScanPath -ErrorAction SilentlyContinue)) {
            throw ('Module path not found at: {0}' -f $ScanPath)
        }

        $TestsPath = (Join-Path $ScanPath $TestsDirectoryName)
        if (!(Test-Path $TestsPath -ErrorAction SilentlyContinue)) {
            throw ('Test directory not found at: {0}' -f $TestsPath)
        }

        $Files = Get-ChildItem -Path $ScanPath -Include .ps1 -Recurse -File |
                    Where-Object {
                        $DirLeaf = Split-Path $PSItem.Directory -Leaf
                        $DirLeaf -notmatch 'Tests'
                    } | Select-Object Name

        $Tests = Get-Childitem -Path $TestsPath -Recurse -Name -File
        foreach ($File in $Files) {
            $TestFileName = $File.Name -replace '.ps1','.tests.ps1'
            if ($TestFileName -notin $Tests) {
                Write-Output $File.Name
            }
        }
    }
}
{% endhighlight %}

This function uses `Get-ChildItem` to find all the files in a given path, recursively,
including only .ps1 files. The output of `Get-ChildItem` is assigned to the
`$Files` variable.

It then uses `Get-ChildItem` again to find all the files in the `Tests` directory,
assigning those file names to the `$Tests` variable.

Finally a simple `foreach` loop is used to loop through all the .ps1 files.

{% highlight powershell %}
foreach ($File in $Files) {
    $TestFileName = $File.Name -replace '.ps1','.tests.ps1'
    if ($TestFileName -notin $Tests) {
        Write-Output $File.Name
    }
}
{% endhighlight %}

The `foreach` loop first creates a new string representing the name of the matching
test file. It does this by replacing `.ps1` with `.tests.ps1` in the filename.
That string is then used along with the -notin comparison operator. If the generated
filename is not in the `$Tests` array, then the script outputs the name of the
file.

This output is assigned to the `$MissingTests` variable.

{% highlight powershell %}
$MissingTests = Get-FilesWithNoTests -ScanPath $ScanPath -TestsDirectoryName $TestsDirectoryName
{% endhighlight %}

In the next bit, the script loops through the filenames in `$MissingTests`, and
creates issues using `Invoke-RestMethod`.

{% highlight powershell %}
foreach ($MissingTest in $MissingTests) {
    $Body = @{
                title     = 'Missing Pester test for {0}' -f ($MissingTest -replace '.ps1','')
                body      = 'No corresponding pester test file for file: {0}' -f $MissingTest
                labels    = @('pester')
             }
    $JSON = $Body | ConvertTo-JSON
    $Response = Invoke-RestMethod -Method POST -Headers $Headers -Body $JSON -Uri $Uri
    Write-Output $Response | Select-Object id,number,title,milestone
}
{% endhighlight %}

The Github API expects to receive JSON in the body of the REST request, so the
first thing that's done in the loop is to create a hashtable<sup>[2]</sup> then
convert that to JSON with the aptly named `ConvertTo-JSON` cmdlet.

Then the `Invoke-RestMethod` cmdlet is used to actually create the issue. Because
Github's API sepcification for creating issues calls for using a PUT request, the
method parameter is set to PUT.

The HTTP headers created at the beginning of the script are also passed to the
`Invoke-RestMethod` cmdlet, as is the JSON created from the `$Body` variable, and
the `$Uri` - the address the REST method will POST to. The output of `Invoke-RestMethod`
is saved in the `$Response` variable, and then the id, number, and title are output
to the console.

When it's done, you'll have an issue that looks like this:
![Pester issue](/assets/PesterIssue.png)

## The Whole Enchilada

As promised, the full script:

{% gist 91897130b501686720798f4223fbb4b2 %}


[1]:https://developer.github.com/v3/auth/#basic-authentication
[2]:https://technet.microsoft.com/en-us/library/hh847780.aspx
[Github's REST API]:https://developer.github.com/v3/
