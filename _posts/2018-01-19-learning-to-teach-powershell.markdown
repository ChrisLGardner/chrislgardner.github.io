---
layout: post
title:  "Learning to Teach PowerShell"
categories: powershell
date:   2018-01-19 18:00:00 +0000
excerpt_separator: <!--more-->
---

I spend a reasonable amount of time on the PowerShell Slack team ([click here for invites](http://slack.poshcode.org/)) and we regularly have people new to PowerShell dropping in and asking questions, it recently got me thinking about the way in which I (and the many others) help people with these problems but also how I teach people in general.

Warning: This is likely to be a somewhat rambling post.

<!--more-->

## Background ##

I've been working with PowerShell for around 4 years now, I followed probably the most common route of copying scripts from online source and hacking around them to make them work for me and then moved on to writing my own scripts and now I'm writing modules, doing unit tests, running them through CI pipelines and all those other cool things that make life with PowerShell easier. Throughout all this I've found that PowerShell can be difficult to learn but a few simple rules that the language follows makes it a lot easier, however teaching PowerShell can be considerably more difficult, so I thought I'd document some of the things I've learned while trying to teach PowerShell and some of the methods I use when helping with problems.

## Learning styles ##

Having spent a number of years working in schools I picked up a few things while fixing AV during teachers’ seminars and other situations, one of them is that there are a number different ways that people learn and figuring out the way a person learns best can be very important to helping them progress through school. I'd always assumed that most people in IT based roles learned best by doing, but I wanted to get some data on that so I [asked twitter](https://twitter.com/HalbaradKenafin/status/953328408091615232). The sample size isn't particularly big but the heavy slant of data towards a 'learn by doing' approach is the sign I was looking for.

## Applying learning styles ##

With this assumption in mind let’s look at some of the questions we regularly see in Slack, [Reddit](https://reddit.com/r/powershell) and other places, how I usually deal with them and why I do it that way.

### I'm new to PowerShell, what's the best book to read ###

This is probably the most common question (or something similar) that you'll see on Reddit and a few other places, it's less common on Slack but we still get the occasional one.

The answer to this is invariably [Learn PowerShell in a Month of Lunches](https://www.amazon.co.uk/Learn-Windows-PowerShell-Month-Lunches/dp/1617294160/) by Don Jones and Jeff Hicks. It's a great book and I've read it myself a while ago for some of the great hints and tips that even experienced PowerShell users can benefit from. More recently there is [PowerShell 101](https://leanpub.com/powershell101) from Mike F Robbins which has received a lot of praise and based on the content of his blog I'm sure it's a great book.

### I want to do X with PowerShell ###

This is the more common question we get in Slack, where X can be a huge range of things. This is generally my least favourite style of question and the one I respond to in the least helpful way (at least I think so).

My typical way of dealing with this sort of request is to suggest some of the cmdlets that might be applicable to their problem, possibly provide some suggestions or small snippets of code that might help. A lot of the time the person at the other end has little PowerShell knowledge but hasn’t expressed that and therefore I'm trying to give them suggestions in the right direction and hopefully they'll dive into a console or the ISE or VSCode and start playing around with it. More often they want, or need, a bit more and I'm happy to oblige if I can replicate at least part of the problem their trying to solve.

The problem then comes with those people who, for whatever reason, just can't quite grasp the necessary PowerShell for the task they are trying to complete. In those cases, I try to step back a bit and break down the problem they are trying to solve into more manageable chunks. Show them how to solve a little bit at a time, show how PowerShell does things and how to tie things together and then slowly build up.

### I've got a script for doing Y but it doesn't work ###

This is probably my favourite question to get, the person has put in some amount of effort to get a working solution and just needs a bit of help to get it finished or troubleshoot some issue. I often dislike troubleshooting my own code because I've likely been working on it for too long to notice anything obvious but other people's code can be interesting and querying some of their choices can help both of you understand things better.

The way I find best to help with these issues is to take a standard troubleshooting approach, start with the error and work from there. If there are no obvious reasons the error should be occurring, then you start with the smallest amount of code necessary and slowly add more until you hit the error. Debuggers like VSCode help a lot with this, as do Pester tests for the various code paths (especially the unhappy paths through the code), once you narrow down the problem it usually becomes reasonably easy to solve it.

The more difficult, and potentially interesting, variety of these problems comes when someone has a script which is working but not quite how they intended it, or their approach seems very inefficient. The interesting parts come when you ask around why they are doing things in certain ways, usually it's because they didn't know of alternatives (often with +='ing arrays) but there are also times when they have strange setups that require jumping through various hoops.

## What I've learned ##

The big thing I've learned/realised is that I'm a big fan of the "teach a man to fish" approach, I'll occasionally go with the "give a man a fish" approach when I have to but I'll usually try giving the man a few little bits of fish first and see if he can put them together with some other bits of fish he finds lying around. There are a few of the other regulars on Slack that I've picked up on their usual way to help people and have a reasonable idea of if I should help out as well.

## Conclusion ##

This blog post was partially inspired by [Don Jones' post last year](https://donjones.com/2017/10/19/become-the-master-or-go-away/) because it was becoming apparent to me that even though I'm not a master in every area of PowerShell, I know more than enough to be helping people more. It's what really pushed me to speak at more events and it's making me want to write more blog posts, though they'll likely be more technical where possible because I find them to be easier to write.
