---
layout: post
title:  "Git bisect and PowerShell"
categories: powershell
date:   2019-07-17 21:00:00 +0000
excerpt_separator: <!--more-->
---

Inevitably when writing code there will be bugs, it's just part of being human and writing code and especially true when the code gets complex. So when it occurs we need to track it down and fix it, which can be easy, but we often want to track down where the bug was introduced so we can figure out what caused it (especially for the more difficult to pinpoint bugs). As we're all using source control this becomes easier with git and it's extremely powerful bisect tool.

<!--more-->

## What is git bisect?

To quote the [official documentation](https://git-scm.com/docs/git-bisect) "git-bisect - Use binary search to find the commit that introduced a bug", which to the average person doesn't mean a lot. The simple description is it takes two points you specify and splits them down the middle, picks a side and splits it again, doing this until it finds where the bad commit is. The way it figures out which side to pick can be done in two ways, either by you manually telling it "good" or "bad" or it can automatically figure that out for you if you give it something to run. It's this last part we'll be looking at in more detail here but we'll look at the manual approach first.

## git bisect in action

First we'll need a git repository to work with, [here's one I prepared earlier](https://github.com/ChrisLGardner/gitbisect) that includes a function that has been changed over the course of a number of commits. This repository has a bit of history to it, looking at the log with `git log --oneline` we can see that it started out quite simple and extra functionality has been added over time, we've also got some less than useful commit messages that we should probably speak to the dev about and try to get them writing better messages in future but that's a different problem. The function in example.ps1 is pretty simple and just does some simple data gathering, either from a local computer or a remote one, nothing too special and it's been working for a while.

{% highlight PowerShell %}
function Get-Data {
    param (
        $ServerName = $Env:COMPUTERNAME,
        $Username = 'test'
    )
    if ($ServerName -eq $Env:COMPUTERNAME) {
        $ComputerInfo = Get-ComputerInfo
        [PSCustomObject]@{
            Name = $Env:COMPUTERNAME
            OS = $ComputerInfo.OSName
            IPAddress = (Get-NetIPAddress -AddressFamily IPv4 -InterfaceAlias 'Ethernet').IPAddress
            Domain = $ComputerInfo.CsDomain
            Model = $ComputerInfo.Model
            Manufacturer = $ComputerInfo.Manufacturer
        }
    }
    else {
        Invoke-Command -ComputerName $ComputerName -ScriptBlock {
            $ComputerInfo = Get-ComputerInfo
            [PSCustomObject]@{
                Name = $Env:COMPUTERNAME
                OS = $ComputerInfo.OSName
                IPAddress = (Get-NetIPAddress -AddressFamily IPv4 -InterfaceAlias 'Ethernet').IPAddress
                Domain = $ComputerInfo.CsDomain
                Model = $ComputerInfo.Model
                Manufacturer = $ComputerInfo.Manufacturer
            }
        }
    }
}

{% endhighlight %}

We've been informed by a colleague that it's stopped querying remote machines at some point. Looking at the current version of the script it's pretty obvious what the problem is, the parameter being passed in is $ServerName but the Invoke-Command is trying to connect to $ComputerName, that'll not work out well for us but it's an easy fix to revert everything to ComptuerName since that's the convention in PowerShell (we can add an alias for ServerName if necessary). But now we need to know when this bug was introduced, in our case it's an easy fix but in many it won't be so obvious and it's useful to know what else might be impacted by this change if it was made at the same time as others. This is where `git bisect` comes in.

Let's look at the manual way first, we'll need to start a bisect session:
{% highlight PowerShell %}
PS > git bisect start
{% endhighlight %}

Then we need to tell it which commit is bad and which is good, we'll start with bad because we know what that is:
{% highlight PowerShell %}
PS > git bisect bad
{% endhighlight %}
In this case we can also use `git bisect bad HEAD` to point at the current commit or give it part of the sha for a specific commit if we've already fixed the issue on this branch.

Next up we need to tell git which commit we know is good, this can be either a specific commit sha or it can be relative to the current HEAD position:
{% highlight PowerShell %}
PS > git bisect good 6b98

Bisecting: 2 revisions left to test after this (roughly 1 step)
[e3eb782ce10f59c9bf10626ecf7c00f3b89e5344] fix other thing
{% endhighlight %}
In our case we know that the first commit that added remote computer support was good so we'll use the first few characters of the sha for that. You can provide as many characters as you want from the sha of that commit but I find 4 is usually enough on most code bases, if you've got one with a lot of history then you might need to use 5 or more.

As we can see from the output git has already picked a commit somewhere close to the middle of the distance between the commits we've specified. From here we can test the code and see if we're still seeing the error. As we know that we are then we can tell git that this is a bad commit and to look for another commit to try somewhere between here and the known good commit.
{% highlight PowerShell %}
PS > git bisect bad

Bisecting: 0 revisions left to test after this (roughly 0 steps)
[e4886e8a59c1a921d149b7948eca0954d658b050] fix parameter name
{% endhighlight %}
Now git has picked another commit but looking at the function we can see it's also bad in this one. Git tries to predict how many more commits it's got to check before it finds the issue, and given the short amount of history we've got available it predicts it's got 0 steps left to take.

So we'll tell git that this commit is also bad:
{% highlight PowerShell %}
PS > git bisect bad

e4886e8a59c1a921d149b7948eca0954d658b050 is the first bad commit
commit e4886e8a59c1a921d149b7948eca0954d658b050
Author: Chris Gardner <chris.gardner@section8games.co.uk>
Date:   Wed Jul 17 20:16:40 2019 +0100

    fix parameter name

 example.ps1 | 4 ++--
 1 file changed, 2 insertions(+), 2 deletions(-)
{% endhighlight %}
Based on the the fact that the next commit it could check is our known good commit git has decided that this must be the first bad commit and where our problem first occurred. So we can see what changed and who did it, in this case some person called Chris Gardner is going to get a nice bug to fix since he introduced it.

When we're done with this bisect we simple run `git bisect reset` to get out of it and back to the branch we started on.

In this case it was a pretty short history to look through and the code base wasn't very complex, but if we imagine scaling this up to an older code base with 10s or 100s of commits (or 1000s) and a lot of files then it can become quite time consuming to try to narrow down where the bad commit is. Luckily the folks who wrote bisect thought of this ahead of time and gave us a way to automate this.

## git bisect run

From the [documentation](https://git-scm.com/docs/git-bisect#_bisect_run) we can see that `git bisect run` expects a command to run and its arguments, and if it returns an exit code between 1 and 127 (but not 125) then it'll assume the result was equivalent to `git bisect bad` and try moving on to the next commit. So how do we leverage this with PowerShell?

We'll start with the same example repository and try running our simple test file against it. This test file can be as simple or as complex as we need it to be, as long as it fails in at least one of the ways that we see with the bug, ideally it'll become part of our testing suite for this code (or start it if we don't have any yet) so that we can prevent it occurring again (or catch it before it gets to production).
{% highlight PowerShell %}
Describe "Testing Get-Data function" {
    BeforeAll {
        Mock -CommandName Invoke-Command -MockWith {}
        Mock -CommandName Get-ComputerInfo -MockWith {}
    }

    It "Should call Invoke-Command with the computer name" {
        .\example.ps1 "NotARealComputer"

        Assert-MockCalled -CommandName Invoke-Command -ParameterFilter {
            $ComputerName -eq 'NotARealComputer'
        }
    }
}
{% endhighlight %}

So with our example.tests.ps1 file we can start the git bisect again. **Note** before you run this you'll want to do `git reset --soft HEAD~1` to ensure the test files aren't part of the history of the repository and therefore disappear as soon as git bisect checks out an earlier commit.
{% highlight PowerShell %}
PS > git bisect start HEAD 6b98
Bisecting: 2 revisions left to test after this (roughly 1 step)
[e3eb782ce10f59c9bf10626ecf7c00f3b89e5344] fix other thing
PS > git bisect run .\example.tests.ps1
running .\example.tests.ps1
C:/Program Files/Git/mingw64/libexec/git-core\git-bisect: line 247: .\example.tests.ps1: No such file or directory
{% endhighlight %}

Now we see a problem here, while we'd like to just run our test file and let Pester do what it should we can see that it's not actually going to work that way. Because git isn't a PowerShell tool it doesn't know how to handle .ps1 files even if we run it from within a PowerShell prompt. So the workaround is to of course just run PowerShell.exe and pass it the file we want to run.

{% highlight PowerShell %}
PS > git bisect run powershell -file .\example.tests.ps1
running powershell.exe -file .\example.tests.ps1

Describing Testing Get-Data function
  [-] Should call Invoke-Command with the computer name 99ms
    Expected Invoke-Command to be called at least 1 times but was called 0 times
    10:         Assert-MockCalled -CommandName Invoke-Command -ParameterFilter {
    at <ScriptBlock>, C:\code\gitbisect\example.tests.ps1: line 10
Bisecting: 0 revisions left to test after this (roughly 0 steps)
[d77bd49d42ab8012a5593dd222fb52003e26d2f2] make local the same as remote

Describing Testing Get-Data function
  [-] Should call Invoke-Command with the computer name 94ms
    Expected Invoke-Command to be called at least 1 times but was called 0 times
    10:         Assert-MockCalled -CommandName Invoke-Command -ParameterFilter {
    at <ScriptBlock>, C:\code\gitbisect\example.tests.ps1: line 10
a9192eae044133433e24c25d29e4a350862d1d02 is the first bad commit
commit a9192eae044133433e24c25d29e4a350862d1d02
Author: Chris Gardner <chris.gardner@section8games.co.uk>
Date:   Wed Jul 17 20:31:48 2019 +0100

    Add support for usernames

 example.ps1 | 3 ++-
 1 file changed, 2 insertions(+), 1 deletion(-)
bisect run success
{% endhighlight %}

So now we can get our code running and the tests correctly failing, but the commit it's flagging as the bad commit is actually in the wrong direction. What's causing this? It's due to the fact we're running the test file directly and it's not giving back an exit code,so therefore git assumes it's a 0 and the commit it's checked is good.

The solution to this problem is one of two things, we can write a quick one line script to call `Invoke-Pester -Script .\Example.tests.ps1 -EnableExit` (as seen in runtests.ps1) or we can just use `PowerShell.exe -command Invoke-Pester -Script .\Example.tests.ps1 -EnableExit`, either will work just as well. EnableExit on Invoke-Pester will cause it to return an exit code equal to the number of failing tests, hopefully this'll be a low number (less than 127) but if it doesn't you can instead use the script approach and capture the output of Invoke-Pester (using `-PassThru`) and return your own error code if it fails. This approach is very useful when you've got a large test suite that you need to run against the code as part of a bisect to ensure no other bugs have appeared (that you don't already know about), for most cases though you want to craft a small number of tests to check just this bug and use those rather than whatever full suite you have. 

{% highlight PowerShell %}
PS > git bisect run PowerShell.exe -command Invoke-Pester -Script .\Example.tests.ps1 -EnableExit
running PowerShell.exe -command Invoke-Pester -Script .\Example.tests.ps1 -EnableExit
Executing all tests in '.\Example.tests.ps1'

Executing script .\Example.tests.ps1

  Describing Testing Get-Data function
    [-] Should call Invoke-Command with the computer name 88ms
      Expected Invoke-Command to be called at least 1 times but was called 0 times
      10:         Assert-MockCalled -CommandName Invoke-Command -ParameterFilter {
      at <ScriptBlock>, C:\code\gitbisect\example.tests.ps1: line 10
Tests completed in 1s
Tests Passed: 0, Failed: 1, Skipped: 0, Pending: 0, Inconclusive: 0 
Bisecting: 0 revisions left to test after this (roughly 0 steps)
[e4886e8a59c1a921d149b7948eca0954d658b050] fix parameter name
running PowerShell.exe -command Invoke-Pester -Script .\Example.tests.ps1 -EnableExit
Executing all tests in '.\Example.tests.ps1'

Executing script .\Example.tests.ps1

  Describing Testing Get-Data function
    [-] Should call Invoke-Command with the computer name 91ms
      Expected Invoke-Command to be called at least 1 times but was called 0 times
      10:         Assert-MockCalled -CommandName Invoke-Command -ParameterFilter {
      at <ScriptBlock>, C:\code\gitbisect\example.tests.ps1: line 10
Tests completed in 940ms
Tests Passed: 0, Failed: 1, Skipped: 0, Pending: 0, Inconclusive: 0 
e4886e8a59c1a921d149b7948eca0954d658b050 is the first bad commit
commit e4886e8a59c1a921d149b7948eca0954d658b050
Author: Chris Gardner <chris.gardner@section8games.co.uk>
Date:   Wed Jul 17 20:16:40 2019 +0100

    fix parameter name

 example.ps1 | 4 ++--
 1 file changed, 2 insertions(+), 2 deletions(-)
bisect run success
{% endhighlight %}

And now we can see it actually finds the correct commit that introduced the issue. In this case it took about as long either way, but that was due to the simplicity of the code and the small number of commits, on larger codebases or more complex code the automated way will almost always be better (and helps your overall test coverage).

## Conclusion

So we've seen here how to use `git bisect` to find the exact commit a bug appeared in, using our knowledge of Pester if necessary to automate it, and from there we can take whatever action we need to beyond just fixing the bug. If this was a larger commit with a few files changed we could see if the same or similar issues had been introduced in those but not detected in production use yet. The documentation for `git bisect` is very good (as is most of the git documentation) and includes a few other things you can do with it. Hopefully this will help with any future debugging you need to do, narrowing down the root cause of a bug can help to fix it where just studying the code in its current form might not be proving as useful, especially if there has been refactoring going on at the same time.
