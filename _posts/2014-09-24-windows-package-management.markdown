---
layout: post
title:  "Windows Package Management"
date:   2014-09-23 18:14:00
categories: dotnet
---

## Why so difficult?

Right now almost all of my work is within the Microsoft realm, and so I spend quite a bit of time in Windows. That being said, I've spent quite a bit of time working on other platforms and in other operating systems. One of the most significant projects I've been involved in was a python application, hosted on a flavor of linux. One of the things that always impressed me with various linux distributions was the package management utilities that many come with (notably yum and apt-get). 

Use of these capabilities allows for very easy, standard setup of both a developer's machine and a production server alike. Whereas before I simply took the process of installing applications on Windows as a sunk cost, the mundane tasks of downloading an installer, walking through the GUI, etc. has become very grating now that I've transitioned back to developing in .NET.

I'm a pretty voracious blog reader, but within the .NET-space there are a few guys who I really try and pay attention to. [Scott Hanselman's blog] (http://www.hanselman.com/blog) is a great example of one I habitually read. A little while back I ran across [this post] (http://www.hanselman.com/blog/ScottHanselmans2014UltimateDeveloperAndPowerUsersToolListForWindows.aspx) he made detailing a list of tools for assisting developer productivity.

I'm usually very skeptical about posts like this and have a habit of just skimming them. Different people prefer different things when doing development and often gravitate toward a series of esoteric utilities that may or may not be useful to someone else. I can say that many of the utilities Scott mentioned have actually turned out to be very beneficial. I use NimbleText almost daily now for doing things such as building long IN clauses when I'm fielding requests for one-off analytical queries for our business guys.

The one that really jumped out at me, though, was a Nuget-themed package management utility called  [Chocolatey](https://chocolatey.org/). Could it be that someone finally managed to write a decent package management system for windows?

## Testing out Chocolatey

The name gave me pause but after perusing the site and playing around with it a bit, I'm reasonably impressed by Chocolatey. The number of supported packages is large and growing, and after the initial installation usage of the utility is about as easy as picking up yum or apt-get. One of the perks of working for a cloud hosting company (shameless Rackspace plug) is that it's fairly easy to spin up a fresh instance of a windows machine to play around with.


```powershell
>> choco install package_name
>> choco install package_name -version versionnumber
>> choco search  keyword
```

![Chocolatey goodness](/img/choco-install-git.png)

It's all fairly intuitive. Interestingly enough, after perusing the help and documentation a bit I soon learned that it doesn't just handle third party packages, it can also be used to enable windows features like IIS. This was starting to look more and more promising.

Unfortunately, enabling various windows capabilities and features isn't quite as intitive as installing basic packages. And it would be nice if someone could abstract things enough to keep that ease of use while combinding it with some of the more rich features that powershell scripting has to offer. My overall goal with Chocolatey - or really any package manager in general - is to be able to base-kick a developer machine or production machine. This can involve not only application installation and the enabling of windows features but often other commands not necessarily supported by Chocolatey (think: adding a domain, setting machine keys, etc.). Plus it would be nice if I could script up a series of chocolatey installations in tandem with powershell cmdlets. Sure, I could just create a quick powershell script, but we all know there's more to setting up a development environment than installing a few utilities. For instance, adding a machine to a domain requires a restart, and resuming script execution afterward can get tricky. 

## Enter Boxstarter

Fortunately, someone else already thought of that. [Boxstarter](http://boxstarter.org/) builds on the Chocolatey package management system and takes it one step further. It is a framework that is designed for the installation of a complete environment on a fresh OS. This can get complicated, often resulting in multiple machine reboots as things are installed and added to the registry. Luckily Boxstarter is reboot resilient as well. The system will intercept chocolatey install commands, checks for reboots, recycles the machine, logs the user back in and resumes installation. In theory at least. I wanted to try it out for myself.

My main goal of this is to prove out that we can use it to base-kick a developer machine and a production webserver using the same tool. Consequently, I ended up modifying one of their sample scripts to include a series of things a developer might want and a few features that could be necessary to configure a machine in prod.

```
Set-WindowsExplorerOptions -EnableShowHiddenFilesFoldersDrives -EnableShowProtectedOSFiles -EnableShowFileExtensions
Enable-RemoteDesktop

cinst fiddler4
cinst git
cinst git-credential-winstore
cinst ChocolateyGUI
cinst sysinternals
cinst python
cinst console-devel
cinst notepadplusplus
cinst GoogleChrome
cinst nodejs.install
cinst mysql.workbench
cinst putty
cinst NuGet.CommandLine
cinst virtualbox
cinst Wget
cinst curl
cinst poshgit


cinst Microsoft-Hyper-V-All -source windowsFeatures
cinst IIS-WebServerRole -source windowsfeatures
```

This script contains setting windows explorer settings, installing developer tools like git, fiddler, and sysinternals, language releases, and enables Hyper-V and IIS. Now all that's left is to run it.

![Start it up](/img/boxstarter-kickoff.png)

All I needed to do was pass in a credentials object (in case of reboot) and the path to the script.

![Running](/img/boxstarter-installing.png)

It turns out one of the packages I installed required a reboot. Let's see how Boxstarter handles it. Note that I'm passing in an absolute path in the arguments above. The first time I tried it I got a "not found" error when the script tried to resume after the machine recycled.

![Rebooted](/img/boxstarter-restart.png)

I logged back in the check the progress after the machine restarted. The powershell prompt pops right back up and it resumed and the script continued execution until success.


### Conclusions

I finished up liking what I saw. Even better, after a bit more research I learned that the `0.9.8.20` release of Chocolatey contains many of the features Boxstarter offers and adds several more. To my knowedge, the reboot capabilities of boxstarter still have quite a bit to offer and for the moment it is still a relevant utility to use. My hope is that soon my team can have a script put together that will base install a fresh developer machine with everything we need. This will save valuable time when trying to get new devs started or a developer's machine is replaced. A stretch goal would be to use the same technology for base installations of web and app servers in our development and production environments. All in all, I'd consider it a successful afternoon.
