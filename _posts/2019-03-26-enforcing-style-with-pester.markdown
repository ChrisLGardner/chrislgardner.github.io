---
layout: post
title:  "Enforcing Code Style using Pester"
categories: powershell
date:   2019-03-26 21:00:00 +0000
excerpt_separator: <!--more-->
---

As part of an ongoing effort to improve code quality and consistency across the company we decided to apply the same principles to PowerShell code as we would apply to our C# and other code, since code is code no matter what language it is written in or who maintains it. With this in mind a few of us sat down many months ago and figured out what our style should be using [the community style guide](https://github.com/PoshCode/PowerShellPracticeAndStyle) as a baseline and picking the things we'd like to apply. With these basic guidelines decided it was up to me to enforce these in some way, and Pester as my tool of choice.

<!--more-->

## Why Pester

When looking at the guidelines we'd set out to follow there were a few choices that could be made to handle this:

* Pester
* ScriptAnalyzer
* Custom script

I discarded the idea of a custom script from the start, I didn't really want to reinvent existing tooling especially when this will be running across some unknown number of files.

ScriptAnalyzer is pretty much built for doing this kind of work however to achieve it would be to write some custom rules, which at the time was something I didn't have the time or skills for. If I were to revisit this (and I probably will soon) then that's the route I'll take as I'm much more comfortable with the AST now.

Pester therefore is the last option left, it sort of bridges the gap between the two other options where I just have to write some simple code to check for each guideline I want to test and then if I find it I just throw an error and Pester handles the reporting for me. And because this is going to be run as part of a CI process then I can take those reports and publish them as test results and fail my builds if I want to.

## How do we actually test the guidlines?

This turns out to be both a really simple problem and a really difficult one at the same time, depending on which guideline we want to test. My basic process is quite simple and best illustrated with some psuedo code:

```
Find all the ps1 and psm1 files in the repository
For each file found:
    Check a guideline
        Log out a warning if it fails with the specific line number it's failing on and why
    Check next guideline, repeat for each guideline

Optionally, if any tests have failed then fail the build
```

Quite a simple approach but quite extensible too as we decide on new guidelines we want to apply as it's just a case of writing a new Pester test and adding it to the script that runs this. We converted this to a private Azure Pipelines task so we can easily add it to all the builds we have which include any PowerShell and have the guidelines controlled from a single central point.

## What does a guideline look like?

So we know the Why and the How but what does some of this code actually look like, here is an example we have for ensuring that opening braces aren't on a line on their own as we follow Stroustrup for various reasons.

{% highlight PowerShell %}
It 'Should have opening braces on the same line as the statement' {
    $OpeningBracesExist = $Contents | Where-Object { $_.Trim() -eq '{'}
    if ($OpeningBracesExist) {
        Write-Warning "Found the following opening braces on their own line:"
        foreach ($OpeningBrace in $OpeningBracesExist) {
            Write-Warning "Opening Brace on it's own line - $OpeningBrace"
        }
    }
    $OpeningBracesExist | Should -BeNullOrEmpty
}
{% endhighlight %}

Again quite a simple example, we're taking $Contents (which is the result of `Get-Content -Path File.ps1`) and checking if any lines contain only `{`. If we find any instances of `{` then we'll write some warnings and then fail the test. For most of our projects this isn't necessary as we've got VS Code settings in place to enforce the Stroustrup style, but these settings aren't in all of our projects yet and some people still like to use full Visual Studio or the PowerShell ISE.

Now lets look at a more complex example, in this case we're looking for instances where a developer has made us of one of the Format-* commands to pretty up their output before passing it back to the user. As a general rule a function should always return objects and let the user decide if they want to format them in a pretty way or output to csv or do whatever with them, formatting before the user gets them takes that choice away from them and doesn't actually return normal objects that they can make use of outside of the console.

{% highlight PowerShell %}
It 'Should not have Format-* within a function' {
    $InputScript = [Scriptblock]::Create((Get-Content -LiteralPath $File.Fullname -Raw))

    $FunctionPredicate = {
        param ($Ast)

        $Ast -Is [System.Management.Automation.Language.FunctionDefinitionAst]
    }

    $Functions = $InputScript.Ast.FindAll($FunctionPredicate,$true)

    Foreach ($Function in $Functions) {
        $CommandPredicate = {
            param ($ast)

            $Ast -Is [System.Management.Automation.Language.CommandAst] -and
            $ast.GetCommandName() -like 'Format-*'
        }

        $Results = $Function.FindAll($CommandPredicate,$True)
    }

    Foreach($result in $Results) {
        Write-Warning -Message "Found a line containing a Format cmdlet: $($Result.Parent)"
    }

    $Results.Count | Should Be 0
}
{% endhighlight %}

In this case we're making use of the AST to find all the fuctions defined in a specified script, then for each of those fuction we're finding all the commands which are named like `Format-*`. This is a more recenty addition to the guidelines after a colleague created a few functions outputting pretty data that was completely unusable outside of the console.

## What's next?

So now we've got a bunch of guidelines that we apply to every build containing PowerShell and using Pester to test them, we take the output of those test runs and publish them back to Azure DevOps. This way we can see on a build how many failed and what they were, and when we're reasonably confident that the majority of our scripts are following these rules we'll start enforcing build failures when even a single test fails.

The next step beyond this will be converting many of these to custom ScriptAnalyzer rules. We've started making more use of it and are now outputting the results from analysis into SonarQube, so incorporating our custom rules into this would be very useful for an extra level of reporting. Some of them convert reasonably easy, like the check for Format-*, since they already make use of the AST, others will prove to be more difficult such as the check for the opening brace not being on its own line. When I make some progress on that then I'll almost certianly blog about the process.
