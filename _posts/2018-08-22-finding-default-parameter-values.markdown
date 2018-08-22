---
layout: post
title:  "Finding Default Parameter Values with the AST"
categories: powershell
date:   2018-08-22 18:00:00 +0000
excerpt_separator: <!--more-->
---

As part of a big refactor on an internal module I've decided to add a big pile of Pester tests. The module was lacking them previously due to various reasons and this seemed like the perfect oppurtunity to add them.

With my Pester test suites one of the things I like to do is have a bunch of tests for parameters and their various attributes that I want to ensure are correctly set. Previously I'd focused on the easy things like aliases, mandatory-ness, pipeline input etc. but this time I wanted to check for default values since we make use of them in a few places.

<!--more-->

## Possible soltuions

When approaching this problem there were a few options that occured to me, I could use some regular expressions (regex) to handle it or I could delve into the PowerShell Abstract Syntax Tree (AST). Here's the sample function we'll be working with for this:

{% highlight PowerShell %}
Function Get-SomeData {
    [cmdletbinding()]
    param (
        [Parameter(Mandatory)]
        [string]$ParameterName = 'DefaultValue',

        [int]$HowMany
    )

    $ParameterName
}
{% endhighlight %}

### Regex

There's a running joke among developers that when you choose to use regex to solve a problem then you now have two problems. However in this case it seemend like quite a reasonable solution to implement as it was a very regular pattern that would never be repeated in a file.

So here's the regex I came up with, and a handy comment above it with what it all means (perhaps a little too verbose a description):

{% highlight PowerShell %}
# \[ - Matches a [ character
# \w+ - Matches any word character 1 or more times
# \] - Matches a ] character
# \$ - Matches a $ character
# ParameterName - Matches the string ParamterName
#  * - Matches 0 or more spaces
# = - Matches a = character
# ("DefaultValue"|'DefaultValue') - Matches the DefaultValue string in either quotes and captures in a group
# ,$ - Matches a comma at the end of a line

$regex = "\[\w+\]\`$ParameterName *= *(`"DefaultValue`"|'DefaultValue'),$"
{% endhighlight %}

This works pretty well and will happily find me any instances of that parameter in a file, there should only be 1 so the rest of my It block would look like this:

{% highlight PowerShell %}
It "Should have a default value of 'DefaultValue' for ParameterName parameter" {
    $regex = "^\[\w+\]\`$ParameterName *= *(`"DefaultValue`"|'DefaultValue'),$"
    $MatchStrings = $Sut | Select-String $Regex
    $MatchStrings.Count | Should -Be 1
}
{% endhighlight %}

The major problem I have with this approach is that I have to run it against the functions source file rather than the compiled psm1 file, like all my other tests. It's unlikely there has been any change in the short time between my script combining all the ps1 files into a single psm1 and the tests running but I still prefer to always test the compiled module.

It would still be possible to do this test against the compiled psm1 but I'd have to account for how many functions use that parameter with a default value, which either means updating all my effected tests each time I add a new function or maintaining a config file that I use to control the tests. The later is certainly an option but doesn't feel quite right for this particular testing problem.

### AST

The alternative approach to this is using the Abstract Syntax Tree and the various methods built into it. There are a number of great posts and books out there describing the AST much better than I can, including quite a good example on [Hey, Scripting Guy](https://blogs.technet.microsoft.com/heyscriptingguy/2012/09/26/learn-how-it-pros-can-use-the-powershell-ast/).

So let's dive into my solution to this problem. First we have to get our function and its AST representation, which `Get-Command` is able to provide quite easily. Let's see what properties we can retrieve using the ever useful `Get-Member`:

{% highlight PowerShell %}
Get-Command -Name Get-SomeData | Get-Member

  TypeName: System.Management.Automation.FunctionInfo

Name                MemberType     Definition
----                ----------     ----------
Equals              Method         bool Equals(System.Object obj)
GetHashCode         Method         int GetHashCode()
GetType             Method         type GetType()
ResolveParameter    Method         System.Management.Automation.ParameterMetadata ResolveParameter(string name)
ToString            Method         string ToString()
CmdletBinding       Property       bool CmdletBinding {get;}
CommandType         Property       System.Management.Automation.CommandTypes CommandType {get;}
DefaultParameterSet Property       string DefaultParameterSet {get;}
Definition          Property       string Definition {get;}
Description         Property       string Description {get;set;}
HelpFile            Property       string HelpFile {get;}
Module              Property       psmoduleinfo Module {get;}
ModuleName          Property       string ModuleName {get;}
Name                Property       string Name {get;}
Noun                Property       string Noun {get;}
Options             Property       System.Management.Automation.ScopedItemOptions Options {get;set;}
OutputType          Property       System.Collections.ObjectModel.ReadOnlyCollection[System.Management.Automation.PSTypeName] OutputType {get;}
Parameters          Property       System.Collections.Generic.Dictionary[string,System.Management.Automation.ParameterMetadata] Parameters {get;}
ParameterSets       Property       System.Collections.ObjectModel.ReadOnlyCollection[System.Management.Automation.CommandParameterSetInfo] Par...
RemotingCapability  Property       System.Management.Automation.RemotingCapability RemotingCapability {get;}
ScriptBlock         Property       scriptblock ScriptBlock {get;}
Source              Property       string Source {get;}
Verb                Property       string Verb {get;}
Version             Property       version Version {get;}
Visibility          Property       System.Management.Automation.SessionStateEntryVisibility Visibility {get;set;}
HelpUri             ScriptProperty System.Object HelpUri {get=$oldProgressPreference = $ProgressPreference...
{% endhighlight %}

That's a whole lot of properties but the one we're interested in is `ScriptBlock`. That will, unsurprisingly, return us a ScriptBlock object of the function and part of the scriptblock is the AST representation of it.

{% highlight PowerShell %}

(Get-Command -Name Get-SomeData).ScriptBlock.Ast | Get-Member

   TypeName: System.Management.Automation.Language.FunctionDefinitionAst

Name           MemberType Definition
----           ---------- ----------
Copy           Method     System.Management.Automation.Language.Ast Copy()
Equals         Method     bool Equals(System.Object obj)
Find           Method     System.Management.Automation.Language.Ast Find(System.Func[System.Management.Automation.Language.Ast,bool] predicate...
FindAll        Method     System.Collections.Generic.IEnumerable[System.Management.Automation.Language.Ast] FindAll(System.Func[System.Managem...
GetHashCode    Method     int GetHashCode()
GetHelpContent Method     System.Management.Automation.Language.CommentHelpInfo GetHelpContent(), System.Management.Automation.Language.Commen...
GetType        Method     type GetType()
SafeGetValue   Method     System.Object SafeGetValue()
ToString       Method     string ToString()
Visit          Method     System.Object Visit(System.Management.Automation.Language.ICustomAstVisitor astVisitor), void Visit(System.Managemen...
Body           Property   System.Management.Automation.Language.ScriptBlockAst Body {get;}
Extent         Property   System.Management.Automation.Language.IScriptExtent Extent {get;}
IsFilter       Property   bool IsFilter {get;}
IsWorkflow     Property   bool IsWorkflow {get;}
Name           Property   string Name {get;}
Parameters     Property   System.Collections.ObjectModel.ReadOnlyCollection[System.Management.Automation.Language.ParameterAst] Parameters {get;}
Parent         Property   System.Management.Automation.Language.Ast Parent {get;}
{% endhighlight %}

From here we want to make use of the FindAll method on the AST. This will let us find all AST elements which match a set of criteria we specify. In this case we want to find all of the AST elements which are of type ParameterAST and which have the name ParameterName.

{% highlight PowerShell %}
(Get-Command -Name $Sut).ScriptBlock.Ast.FindAll( {
    $args[0] -is [System.Management.Automation.Language.ParameterAst] -and
    $args[0].Name.VariablePath.UserPath -eq 'ParameterName'
}, $true)
{% endhighlight %}

Here we have a scriptblock in a similar format to those used in `Where-Object` and we make use of the `$args[0]` automatic variable, which is populated with each AST element one at a time. We can also declare a param block within this scriptblock if we wanted to use a more descriptive variable name.

Let's break down what this comparison is doing and look at it more closely:

{% highlight PowerShell %}
$args[0] -is [System.Management.Automation.Language.ParameterAst] -and
{% endhighlight %}

First we make sure the type of AST we're looking at is the type we care about, due to how `-and` comparisons work if this doesn't match then it'll continue on to the next AST element without even checking the other part of the comparison. To find the AST type you want to look at, and to generally explore the AST representation of a scriptblock, you can make use of `Show-AST` from the [ShowPSAst Module](https://www.powershellgallery.com/packages/ShowPSAst/1.0) or there are a few other AST modules available on the gallery as well.

{% highlight PowerShell %}
$args[0].Name.VariablePath.UserPath -eq 'ParameterName'
{% endhighlight %}

Next we compare the name of the parameter to what we want. This is buried a little deeper than just `$args[0].name` and took a little digging around in the resulting AST object that came back without this part of the query.

{% highlight PowerShell %}
}, $true)
{% endhighlight %}

This section is a little interesting as FindAll expects two arguements passed to it, a Predicate (the scriptblock) and a boolean. The Predicate is what we've just looked and it should just return true or false. The boolean is to tell FindAll if it should recurse through nested scriptblocks. In this case we'll want to do this so I've set it to `$true` but there are some cases where you won't want to do this and can therefore set it to `$false`.

So now, hopefully, we'll have an output from this containing the AST object for the parameter we're looking for. From here we just want to access the `Value` property of the `DefaultValue` property (as it's a nested object with some more details in it).

{% highlight PowerShell %}
(Get-Command -Name $Sut).ScriptBlock.Ast.FindAll( {
    $args[0] -is [System.Management.Automation.Language.ParameterAst] -and
    $args[0].Name.VariablePath.UserPath -eq 'ParameterName'
}, $true).DefaultValue

StringConstantType : SingleQuoted
Value              : DefaultValue
StaticType         : System.String
Extent             : 'DefaultValue'
Parent             : [Parameter(Mandatory)]
                     [string]$ParameterName = 'DefaultValue'
{% endhighlight %}

So our final test when all this is put together looks like the below. We could add another check in here too to ensure the parameter is only present once but that would probably be better suited a separate test.

{% highlight PowerShell %}
It "Should have a default value of 'DefaultValue' for ParameterName parameter" {
    (Get-Command -Name $Sut).ScriptBlock.Ast.FindAll( {
        $args[0] -is [System.Management.Automation.Language.ParameterAst] -and
        $args[0].Name.VariablePath.UserPath -eq 'ParameterName'
    }, $true).DefaultValue.Value | Should -Be 'DefaultValue'
}
{% endhighlight %}

The major benefit this has over the regex approach is that it'll run against the psm1 file as it's pulling the functions from Get-Command after the psm1 has been imported. It's also a lot more focused and less brittle than the regex approach, if a parameter isn't strongly typed then the regex will miss it. There are solutions to these problems but it will often feel like investing time and effort into a less flexible solution.

## Conclusion

The AST is really powerful but can also be pretty complicated when you're first looking at it, tools like Show-AST help a lot and there are a lot of really good blog posts out there about working with the AST. There are also a few really helpful people on the [PowerShell Slack](https://j.mp/psslack) who know a good deal about the AST. As we can see in this situation it has given us a much more flexible solution to the problem we were trying to solve and it accounts for a lot more possible situations.

Regex is an even more powerful and complicated beast but it can be pretty difficult to get your pattern correct for the various use cases you have. It's a tool I'll often employ first but it is not always the best solution, as can be seen here.
