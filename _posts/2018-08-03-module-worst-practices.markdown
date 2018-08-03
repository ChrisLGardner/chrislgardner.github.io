---
layout: post
title:  "Module Worst Practices"
categories: powershell
date:   2018-08-03 18:00:00 +0000
excerpt_separator: <!--more-->
---

It has been a while since my last post so lets get back on track with some "interesting" things I've discovered while working on another project for another blog post (or two). The project involves taking almost all the modules in the PowerShell Gallery and pulling all their help data into a graph database (Neo4j) and then doing some analytics on it. That's still in progress but I thought I'd blog about some of the worst practices I've seen while doing this work.

<!--more-->

There'll be more details about the process I followed to get all the data into the database but it followed roughly 3 steps:

- Import the module and add a node to the DB
- Get all the public commands in the module and add each as a node in the DB
- Get the help for each command and update the command node with some properties

Ignoring the various performance improvements I could have made in this process (more on those in the other post), there were a number of problems that jumped out from these steps. We'll start with the most annoying ones and work from there.

## Don't prompt for credentials on import

### The Problem

This covers either using Get-Credential or Read-Host and it goes beyond just normal credentials but any sort of values to access things, like API keys or access tokens or anything of that sort. This stops any sort of automation dead if you haven't previously run the import manually and set up these values, assuming they can be persisted to disk securely somewhere.

### The Solution

Provide functions for this, a simple `Connect-MyService -Credential` cmdlet is often enough as then that credential object creation can be automated. Other options include making it configurable using something like `Set-MyServiceConfiguration -Credential`, this is especially useful if you've got a number of other configurable settings and modules like [http://psframework.org/](PSFramework) can make this a lot easier to work with.

Store credentials of any sort securely can be another problem to deal with, DPAPI helps a lot in keeping them secured to just the user who created them on the machine they were created but that's often going to cause further issues when you run your scripts as dedicated service accounts. There are a few possible soltuions to this:

- [https://github.com/Jaykul/BetterCredentials](Better Credentials) stores credentials in the Windows Credential Store for easier retrieval
- [https://github.com/dlwyatt/ProtectedData](Protected Data) lets you encrypt pretty much anything with either a certificate or a password and supports all the way back to PSv2
- [https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.security/protect-cmsmessage?view=powershell-5.1](Protect-CmsMessage) and the other CmsMessage cmdlets perform a similar role to ProtectedData but only work on strings and only work on PSv4+

Before implementing any of these (or your own method) always consider your use case and the possible points of failure.

## Don't change the prompt or console without warning or a way to restore what was there

### The Problem

Some modules are specifically designed to make changes to the prompt or add extra functionality and they can be very powerful in what they offer beyond the basics offered by the console. The problem comes when they make those changes on import and then don't offer a way to reverse it on removal or are used as RequiredModules by other modules. Suddenly finding your prompt has changed from whatever you had before, even if you didn't intentionally set it up that way, can be a jarring user experience.

An equally annoying one is changing the colours of the text or background colour, I've also seen at least one instance of a module clearing the screen on import before dumping some module information to it. These were not good user experiences and no matter how good those modules were at what they did I was considerably less likely to be using them.

### The Solution

At a minimum take a backup of the current settings that you're about to modify and store that somewhere sensible for the user like $HOME, preferably in the form of a script they can run to revert the changes. Beyond that provide functions to enable and disable the functionality your module is providing, see Posh-Git as an example; it has a Write-VcsStatus function that you can place in your custom Prompt function and have it work.

You can potentially also add some behaviour to your module that deals with what happens when someone runs `Remove-Module MyModule`, the process for doing this is detailed in the Notes section of the help for [https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/remove-module?view=powershell-6#notes](Remove-Module).

## Don't Set-StrictMode and leave it there

### The Problem

Set-StrictMode is very helpful for ensuring you haven't made any obvious mistakes in your code, like using a variable before its been assigned to, but it comes with a problem that many people are apparently not aware of: Its effects apply to the whole current session and not just to your module or function. There are some arguments that could be made to say that StrictMode should be enabled almost all of the time but that should be a user choice and not a random module that I'm using as part of this script I'm writing.

### The Solution

Similar to the solution about modifying the prompt and console, make sure you `Set-StrictMode -Off` at the end of your module or function. It is not always obvious when it has been enabled and there isn't a builtin cmdlet to find out if it is enabled and to what level, thankfully Chris Dent has written [https://gist.github.com/indented-automation/9279592035ca952360ce9e33643ba932](this helpful function) to solve this problem.

There are other ways around needing to even enable StrictMode, the best option would be Pester tests for all your functions to help ensure you know that all of your code paths will work and that your function will correctly handle null inputs in places etc. If you're primarily worried about users not inputting values for some parameters then that's where the Mandatory parameter attribute comes in, or validation attributes like `[ValidateNotNullOrEmpty()]` and others.

## Don't throw an error and fail importing if non-PowerShell dependencies aren't there

### The Problem

Some modules rely on tools outside of PowerShell, we're writing PowerShell to automate almost anything and that often means we need third party applications installed on the machines the functions will run on. The solution some module authors have chosen is to check for it on import and throw an error when it is not there, others have chosen to prompt for an optional download of the relevant application. There is some sense in this, you likely can't make much use of the module without that application but it also doesn't account for situations such as an application which has license costs and restrictions in place that prevent it being run on a machine that is being used to write the scripts that make use of it.

### The Solution

The simplest solution is to simply write a warning on import if you don't detect the application or other dependency is there. This lets the user know it is needed but doesn't prevent them from exploring the module, its commands and, importantly, the help. You can extend this further to have it documented in the Readme for the module detailing these dependencies, this should also be present on the page on the PowerShell Gallery, and its Github page if it is open source.

## Ensure your module manifest has the correct attributes

### The Problem

A number of modules I looked at were missing the RootModule attribute, and didn't make use of NestedModules or the other attributes that also work for this. By missing out these attributes the module wasn't exporting any commands, which means when I run `Get-Module MyModule -ListAvailable` or `Get-Command -Module MyModule` I get no output for what commands are available. The other common issue I've seen was `FunctionsToExport = @()` which also results in no commands being exported, even if you've called `Export-ModuleMember -Function` at the end of your psm1 file. The main cause of this is when a module developer has been working with the psm1 file and only adds the psd1 file to publish it to the PowerShell Gallery and doesn't fully understand what all the attributes are for, they'll keep importing just the psm1 and it works fine for them.

### The Solution

This is a very simple one to fix, set the RootModule attribute of your psd1 to point at your psm1 or compiled dll. If you've got multiple psm1 files then you can make use of the NestedModules attribute, but I'd also set one of them as the RootModule to allow you to Pester test them correctly.

FunctionsToExport is a bit more interesting, it should only list the functions you actually want to present to users. Keeping this up to date as you add new functions can be a bit of a pain especially with multiple people working on the module, the solution is to update it dynamically as part of your process to publish it to the gallery. The [https://github.com/PoshCode/Configuration](Configuration) module has a very useful function for handling this called `Update-Metadata` and I'd highly recommend making use of it, it'll also allow you to update the version number as you publish new versions.

## Conclusion

These are some of the problems I've encountered as part of the larger project I'm working on, I'm sure I'll find more as I get closer to being finished with it so check back in future for more updates or other blog posts about them.

If you've encountered anything you think is a "worst practice", disagree with any of these or want alternate ways to solve some problem then feel free to tweet me [https://twitter.com/halbaradkenafin](@halbaradkenafin) or find me on the [https://j.mp/psslack](PowerShell Slack).
