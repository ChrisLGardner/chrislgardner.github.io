---
layout: post
title:  "[Scriptblock] and ConvertTo-Json: a match made in recursive hell"
categories: powershell
date:   2019-02-17 13:00:00 +0000
excerpt_separator: <!--more-->
---

I've been working on [JeaDsc](https://github.com/ChrisLGardner/JeaDsc) off and on for a few months to improve on the original project and make it available in the PowerShell Gallery. The biggest bug it's currently got is that it doesn't compare an existing configuration against a new one very well, especially for complex configurations. For a DSC resource this is a huge problem and something I've been wanting to fix got a little while. As part of fixing it I came across this wonderful problem with ConvertTo-Json.

<!--more-->

Before I get into the details of the problem and how I worked around it let's look at JEA a little.

## What is JEA?

JEA stands for Just Enough Administration and is a PowerShell feature that allows you to restrict what a user can do when they make use of PowerShell Remoting into another machine. There's some documentation on [docs.com](https://docs.microsoft.com/en-us/powershell/jea/overview) about it.

With the ever growing use of PowerShell to manage systems, especially at scale, it's important to consider what permissions a user needs to perform their tasks and only grant them that much access. Level 1 Helpdesk users don't need Domain Admin permissions if their main work is resetting passwords and basic troubleshooting, but depending on your AD structure you might not be able to easily grant them the permissions they need, this is where JEA comes in. You can define a Role Capabilities file that states a user can only run `Set-ADAccountPassword` and that the identity they can specify must be a certain format, that which matches your normal staff users. We then take this file and add it to a Session Configuration file, which specifies who can connect to a PowerShell remoting session and if they do then what permissions they get and a bunch of other useful things like transaction logging etc.

With this we can securely control who can access which servers and what they can do on those servers, and all of this can be controlled through PowerShell so we can deploy this using DSC. The big benefit of using DSC to handle this is that we can version control these configuration files to see what permissions were granted and when and by who. We can then feed this through a release pipeline to allow us to deploy these changes automatically to our pull server, first to a test environment and then to production. I'll have another blog post or two on this process in future as I'm currently working on a pipeline for this.

## Where does ConvertTo-Json come into this?

As part of JeaDsc we need to test the existing configuration to see if we need to apply the new configuration to it. As both Role Capabilities and Session Configurations are stored as hashtables it's a little complex to compare them, we can't just do a simple `$ExistingConfiguration -eq $NewConfiguration` as they are different objects even if the keys and values are the same. So my first implementation for Role Capabilities was to make use of a existing function to [compare hashtables for pester](https://github.com/stuartleeks/PesterMatchHashtable) but this runs into a bit of problem when the hashtables can be as complex as these, including arrays and hashtables within the values of some of the keys. This wasn't initially a problem until you start making use of the FunctionDefinitions feature and have to compare scriptblocks, as raised [here by Raimund Andree](https://github.com/ChrisLGardner/JeaDsc/issues/19).

So I looked at how Session Configurations were handling this as that had existing before I started working on this and I found this method:

{% highlight PowerShell %}
 hidden [bool] ComplexObjectsEqual($object1, $object2) {
    $object1ordered = [System.Collections.Specialized.OrderedDictionary]@{}
    $object1.Keys | Sort-Object -Descending | ForEach-Object {$object1ordered.Insert(0, $_, $object1["$_"])}

    $object2ordered = [System.Collections.Specialized.OrderedDictionary]@{}
    $object2.Keys | Sort-Object -Descending | ForEach-Object {$object2ordered.Insert(0, $_, $object2["$_"])}

    $json1 = ConvertTo-Json -InputObject $object1ordered -Depth 100
    $json2 = ConvertTo-Json -InputObject $object2ordered -Depth 100

    if ($json1 -ne $json2) {
        Write-Verbose "object1: $json1"
        Write-Verbose "object2: $json2"
    }

    return ($json1 -eq $json2)
}
{% endhighlight %}

Ignoring the badly named variables, this is a pretty simple but elegant solution. Converting the objects to ordered dictionaries and then to JSON should ensure that the comparison is accurate. This method works very well for Session Configuration files but how well will it work with Role Capabilities? I think we can guess the answer to this by the fact I'm writing a blog post about it.

## Scriptblocks and their properties

Within a JEA Role Capability file ScriptBlocks are used for the FunctionDefinitions and allow administrators to define custom functions that are available within a JEA session to further control what a user can or can't do. The problem from my perspective comes when I try to convert those to JSON as the default output to the console isn't the only property of the object, much like many other object types in PowerShell.

Let's take a look at an example:

{% highlight PowerShell %}
PS> $ScriptBlock = { Get-Command }

PS> $scriptblock
 Get-Command

PS> $scriptblock.ToString()
 Get-Command

PS> $ScriptBlock | Get-Member -MemberType Properties
   TypeName: System.Management.Automation.ScriptBlock

Name            MemberType Definition
----            ---------- ----------
Ast             Property   System.Management.Automation.Language.Ast Ast {get;}
Attributes      Property   System.Collections.Generic.List[System.Attribute] Attributes {get;}
DebuggerHidden  Property   bool DebuggerHidden {get;set;}
File            Property   string File {get;}
Id              Property   guid Id {get;}
IsConfiguration Property   bool IsConfiguration {get;set;}
IsFilter        Property   bool IsFilter {get;set;}
Module          Property   psmoduleinfo Module {get;}
StartPosition   Property   System.Management.Automation.PSToken StartPosition {get;}
{% endhighlight %}

So we can see here that the default output for a `[ScriptBlock]` type is just calling the ToString() method on it but it has a few other properties on it that `ConvertTo-Json` will try to use instead, since it doesn't care about methods, and many of these properties also have properties of their own. This normally wouldn't be a problem as `ConvertTo-Json` has a limit of how far down the tree it'll go, by specifying the `-Depth` parameter we can control this and it defaults to 2. Other than not really returning what we need from this I also discovered that after a certain depth there are some references to further up the chain and we get into a bit of a recursive lookup problem before PowerShell either crashes or throws a Stackoverflow exception. [Someone else had found this too](https://github.com/PowerShell/PowerShell/issues/7091) and helpfully logged it on GitHub but with no solution in sight and if there was it would only be available in PowerShell 6+ I needed a workaround.

## The Solution/Workaround

Because I don't want all of those properties that come with a ScriptBlock and only really need the ToString() output I decided the simplest soluiton was to just replace any ScriptBlocks with their ToString() output and ended up with this:

{% highlight PowerShell %}
if ($ReferenceObjectordered.FunctionDefinitions) {
    foreach ($FunctionDefinition in $ReferenceObjectordered.FunctionDefinitions) {
        $FunctionDefinition.ScriptBlock = $FunctionDefinition.ScriptBlock.Ast.ToString().Replace(' ', '')
    }
}
{% endhighlight %}

I'm calling the AST version of ToString() to ensure I get the braces in as well, this will make it a bit easier for people debugging if their new configuration doesn't match the existing one when they'd expect it to, since it'll match exactly what is in the file. I'm also removing all the whitespace to handle any odd spacing that might come in to play as we don't care about the code actually running and it won't have any impact on the Set method.

The [full function](https://github.com/ChrisLGardner/JeaDsc/blob/master/JeaDsc.psm1#L105) has been renamed to something a bit more focused and I've updated the variable names to be more meaningful. This will be rolled into the Session Configuration resource as well so that I only have to maintain a single version of the code. I also wrote a [whole bunch of tests](https://github.com/ChrisLGardner/JeaDsc/blob/master/Tests/Unit/Compare-JeaConfiguration.Tests.ps1) for the different scenarios I could think of where I'd want to use this function and this way I can ensure it still works whenever I make changes to it, which should hopefully be very rare now.

Arguably this function should really be called `Test-JeaConfiguration` since one output is a bool and it should output `$true` when they match but as I'm actually doing a comparison it made more sense to call it `Compare-JeaConfiguration`. I should probably fix the output to be `$true` when they do match just to remain consistent with the `$false` output but other `Compare-*` functions don't necessarily output anything when they match.
