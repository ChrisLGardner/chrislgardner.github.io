---
layout: post
title:  "Transforming Measure-Object output"
categories: powershell
date:   2018-02-25 18:00:00 +0000
excerpt_separator: <!--more-->
---

A user on the PowerShell Slack ([available here](https://powershell.slack.com) and [invites here](https://slack.poshcode.org)) asked about getting specific information out of Measure-Object into a more usable PowerShell objects Their initial approach was to use Select-Object and calculatd properties to do this but I suggested a nicer way to handle it by using a short function and pipeline input.

<!--more-->

So the best place to start with anything like this is to get some example data and what they want the output to look like. The outputted data from Measure-Object looked like below.

{% highlight PowerShell %}
Average       Sum Maximum Minimum Property
 -------       --- ------- ------- --------
            123.45                 Interest
         123456.78                 Total
{% endhighlight %}

The way they wanted the data to look at the end was something like this:

{% highlight PowerShell %}
    Total Interest
    ----- --------
123456.78   123.45

{% endhighlight %}

To transform this data and flatten it out into a single object was quite easy. The initial attempt at doing this made use of a hashtable to build up the object and then cast it to a PsCustomObject.

{% highlight PowerShell %}
$hash = @{}
foreach ($row in $ComplexObject) {
    $Hash.add($Row.Property, $ComplexObject.Where({$_.Property -eq $row.property}).sum)
}
$RealObject = [pscustomobject]$Hash
{% endhighlight %}

This will take an object (in this cast `$ComplexObject`) and for each row in it turn that into a property of the new object (or a "column" from a visualisation help). This is a great start and achieved what we wanted but it's not particularly reusable.

Lets look at making this a bit more reusable and helpful for other output from Measure-Object that we might want to work with. First up we'll turn it into a function:

{% highlight PowerShell %}
function Convert-MeasureObject {
    [cmdletbinding()]
    param (
        [Object[]]$InputObject,

        [string]$Property
    )
    $Hash = @{}
    foreach ($row in $InputObject) {
        $Hash.add($Row.Property, $InputObject.Where({$_.Property -eq $row.property}).$Property)
    }
    [PsCustomObject]$Hash
}
{% endhighlight %}

So now we've got a function that we can use by calling it with our output from Measure-Object using `Convert-MeasureObject -InputObject $MeasureObjectOutput -Property 'Sum'`. But this still feels a little cludgy and extra lines and variable assignments we don't really need if we're already using the pipeline for the Measure-Object portion of our data gathering and collation.

So let's add some pipeline support and tidy things up a little:

{% highlight PowerShell %}
function Convert-MeasureObject {
    [cmdletbinding()]
    param (
        [Parameter(ValueFromPipeline)]
        [Object[]]$InputObject,

        [string]$Property
    )

    begin {
        $Hash = @{}
    }
    process {
        foreach ($row in $InputObject) {
            $Hash.add($Row.Property, $InputObject.Where({$_.Property -eq $row.property}).$Property)
        }
    }
    end {
        [PsCustomObject]$Hash
    }
}
{% endhighlight %}

Now we're levaraging the pipeline in a more useful way and can do wonderful things like `<someInput> | Measure-Object -Property 'ThatValue' -Sum | Convert-MeasureObject -Property Sum`.

It's still a little off from a best practice approach as ideally we'd be putting each item back onto the pipeline as they come through in our process block, but the entire point of the object is to do some transformation on pipeline input we kind of need to wait for it all to come through before sending anything back out.

All that's missing now is a bit of help text, some verbose logging to help people figure out what's going on, some unit tests and then we're good to go and can drop this in our module of choice.

The fully completed version of the script is available [here](https://github.com/ChrisLGardner/PowershellScripts/tree/master/ConvertMeasureObject) along with all the tests I will have written for it.

## Conclusion ##

Hopefully that's given a nice look at how to appraoch solving an initial problem and then developing it further into a more reusable and complete solution.
