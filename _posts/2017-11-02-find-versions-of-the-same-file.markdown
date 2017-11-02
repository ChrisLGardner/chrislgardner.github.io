---
layout: post
title:  "Find all versions of the same file"
categories: powershell
date:   2017-11-02 18:00:00 +0100
excerpt_separator: <!--more-->
---

While on site with a customer recently we discovered that their VSTS build for producing their Selenium tests was producing around 800MB of artifacts, this seemed pretty high for something that should just be producing a bunch of test DLLs. So I turned to PowerShell to figure out what was actually being produced and where it was.
<!--more-->

First up was figuring out if there was any duplication of files, the folder structure looked pretty comprehensive so I guessed they had a lot of projects in Visual Studio and that suggested there might be a bit of overlap between dependencies.

{% highlight PowerShell %}
$TotalFiles = Get-ChildItem -Path .\ -Recurse -File | Group-Object -Property Name | Sort-Object -Property Count -Descending
{% endhighlight %}

This showed a pretty impressive list of files, lots of 1s and 2s of files named like areas of their application. However it also revealed the main source of the extra bloat; 89 copies of WebDriver.dll, almost as many copies of NewtonSoft.Json.dll and a few other similar common dlls along with the associated pdbs and more. How to deal with this though? Surely every project is using the same versions of those files? Everyone standardises to a single version of common libraries don't they? The answer was (in reverse order) "Of course they don't", "Of course they aren't" and "PowerShell of course".

So how can we use PowerShell to solve this? Currently we just have a long list of files and the number of each we have. Luckily PowerShell is helpful enough to provide version information on FileInfo objects (those returned by Get-ChildItem) so with a bit of creative pipelining we can get some more useful information about the versions in use.

{% highlight PowerShell %}
$TotalFiles | Where-Object {$_.Count -ge 5 -and $_.name -like *.dll} | Foreach-Object { new-variable $_.name -value (Get-ChildItem -Path .\ -filter $_.name -Recurse | Select-Object -ExpandProperty VersionInfo | Select-Object -Property ProductVersion -Unique)}
{% endhighlight %}

That's a hefty pipeline to take in all at once so lets break it down a bit:

{% highlight PowerShell %}
$TotalFiles | Where-Object {$_.Count -ge 5 -and $_.name -like *.dll}
{% endhighlight %}

First we get all the files that end in dll, because we can guarantee the common libraries are correctly versioned, and we also only want anything that appears more than 5 times, mostly this was an arbitrary amount I decided seemed like a reasonable cutoff but it could easily be lower or higher depending on the dataset we're working with.

{% highlight PowerShell %}
Foreach-Object { new-variable $_.name -value (....)}
{% endhighlight %}

Then we iterate over all of those we've found using Foreact-Object and create a new variable named the same as the file. We don't intend to ever directly call any of these uing $ notation but it stores the output in a convenient location that we can query pretty easily assuming our session doesn't have a huge number of variables for some reason.

{% highlight PowerShell %}
Get-ChildItem -Path .\ -filter $_.name -Recurse | Select-Object -ExpandProperty VersionInfo | Select-Object -Property ProductVersion -Unique
{% endhighlight %}

This is the real meat of the command, and some very inefficient meat it is as well but for my dataset of a few thousand files it wasn't too inefficient. First we get all the files named the same as the current item in the pipeline, NewtonSoft.Json.dll for example, we then expand it's VersionInfo property into a new object on the pipeline. Then we finish off by selecting just the ProductVersion of those items we've found and filtering it even further by just selecting the unique ones. All of this is stored in the variable named NewtonSoft.Json

So what do we do with this new information we've got sitting in our variables? We can query those variables and find any that have multiple versions.

{% highlight PowerShell %}
Get-Variable | Where-Object {$_.name -like '*.dll' -and $_.value -is 'System.Array'}
{% endhighlight %}

This will take all our variables in the session, find any that end in .dll and that are also Arrays. We're then presented with a (hopefully) short list of variables and their values similar to below.

{% highlight PowerShell %}
Name                    Value
----                    -----
NewtonSoft.Json.dll     @({ProductVerion=9.1.0},{ProductVersion=6.5.0})
{% endhighlight %}

So at a glance we can see the various versions in use. We could have used -ExpandProperty on our final Select-Object statement in the Foreach loop to drop the ProductVersion from this output, I find both to be easily readable and this approach means we can call ${NewtonSoft.Json.dll} and see the usual styled object output.

## Summary

So as we can see it's pretty easy to get all the versions of any number of files and then figure out how many different versions you actually have. A similar process could be applied to files without VersionInfo using Get-FileHash to compare them and detect uniqueness.

Managing dependency versions can be difficult, especially in larger solutions or solutions that have evolved over a longer length of time, hopefully you catch it early enough that standardising is a less time consuming experience. In the case of the customer they were in pretty good shape as they only had around 3 different versions of a few common libraries so fixing those depenencies should be a pretty easy fix. Once that's implemented they can clean up their build process to flatten out their folders and then only produce one copy of each and save themselves a lot of space in their build artifacts and a noticeable amount of time in their release pipelines for copying these artifacts around.
