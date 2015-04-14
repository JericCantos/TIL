---
title: Setting up Jenkins to build .NET Projects from Git Lab.
tags: [Jenkins, MSBuild, GitLab]
---

Jenkins is currently being used by open source teams in my organization as their Continuous Integration solution. Coming from a Microsoft-centric team, we rely more on MSBuild/TFS Build Server and Visual Studio Release Management to do this for us. I wondered, though, if we could use Jenkins for MS projects instead, which would allow our organization to maybe focus on just one tool and cater to a wide range of projects. My goal was to build a pipeline from Visual Studio going into GitLab for source control and then moving into Jenkins installed on Windows Server 2012 for continuous integration, scheduled builds, and all that jazz. Here's my adventure. [TL;DR](#tldr)

#Installing Jenkins

I've been using Visual Studio and GitLab for a while now so my development machine is already properly configured to use both. I just needed to install and configure Jenkins and I'd be all set. So I downloaded it from the [Jenkins site] (https://jenkins-ci.org/), which thankfully provides a native Windows Package. It offered a no-fuss installation, and within Minutes, Jenkins was up and running. So far so good.

#Installing Jenkins Plugins

Next, I needed to teach Jenkins how to build Microsoft .NET solutions. A quick Google search led me to this article - [Jenkins for .net in 5 Minutes](http://justinramel.com/2013/01/15/5-minute-setup/) which informed me that there's already an MSBuild plugin available for Jenkins. Well, it turns out that it doesn't take just five minutes (at least not for me).

##Plugin Woes

The guide mentions that I should be able to simply search for the MSBuild plugin through the Jenkins UI via Jenkins -> Manage Jenkins -> Manage Plugins -> Available. Lo and behold, my Available Plugins page is empty.

![Not what I was seeing](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/availableplugins.jpg)
**Not what I was seeing.**

That's weird. How am I going to get the MSBuild plugin now? Well, somehow through more Google consultations I found a [directory of all Jenkins plugins](https://updates.jenkins-ci.org/download/plugins/) with a direct download link. Great! I quickly downloaded the latest version of the MSBuild plugin to my local machine, and then installed it through Jenkins -> Manage Jenkins -> Manage Plugins -> Advanced -> Upload Plugin.

Next on the list is Git and GitLab. Through various sources like [this one](http://doc.gitlab.com/ee/integration/jenkins.html) I found out that I require the Git, Git Client, and GitLab Hook plugins. I downloaded these and installed them like I did with MSBuild, and then configured them like so (Jenkins -> Manage Jenkins -> Configure System).

![MSBuild Config](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/msbuildconfig.jpg)
![Git Config](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/gitconfig.jpg)

#First Job Setup 

I was supposedly all set to create my continuous integration job. Here are the pertinent details I entered then:

![Project Details](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/projectbasics.jpg)
![Project Git Details](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/projectgit.jpg)
![Project Build Triggers](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/projectbuildtriggers.jpg)
![Project MSBuild Configuration](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/projectmsbuild.jpg)

Hitting "Build" rewarded me with the following output.

```
Started by user anonymous
Building in workspace C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace
Cloning the remote Git repository
Cloning repository git@XXXXXXX.XXXX.XXXXX:XXXX/continuousdeliverypoc.git
 > C:\Program Files (x86)\Git\cmd\git.exe init C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace # timeout=10
Fetching upstream changes from git@XXXXXXX.XXXX.XXXXX:XXXX/continuousdeliverypoc.git
 > C:\Program Files (x86)\Git\cmd\git.exe --version # timeout=10
 > C:\Program Files (x86)\Git\cmd\git.exe -c core.askpass=true fetch --tags --progress git@XXXXXXX.XXXX.XXXXX:XXXX/continuousdeliverypoc.git +refs/heads/*:refs/remotes/origin/*
ERROR: Timeout after 10 minutes
ERROR: Error cloning remote repo 'origin'
ERROR: Error cloning remote repo 'origin'
Finished: FAILURE
```

Very nice. This problem stumped me for quite a while and Google wasn't quite as helpful. I tried extending the timeout, to changing my machine's proxy. I was able to ping the GitLab server and I was able to fetch, pull, and push changes via GitExtensions but nothing I did allowed Jenkins to do the same.

#Seeing the Light

##Jenkins Behind Proxy
Around this time I completely removed Jenkins and did a reinstall, thinking I may have messed something up when managing the plugins. Around this time, I also got in touch with Jason Saba who had prior experience with Jenkins, but not specifically with Git. He pointed me in the direction of this [article](https://wiki.jenkins-ci.org/display/JENKINS/JenkinsBehindProxy) regarding Jenkins behind a proxy. The article talks about how to set the Jenkins proxy through setting some Java system properties, which was a problem for me because I did not actually install Java on my development machine, and it wasn't available (_read: I was too lazy to set it up_) in my PATH variable. Luckily, it seems I wasn't the only one requesting for the proxy to be maintainable through the Jenkins UI. It can be found in Jenkins -> Manage Jenkins -> Manage Plugins -> Advanced.

![Proxy Config](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/proxyconfig.jpg)
**Be sure to hit the "Check now" button at the bottom of this page.**

This worked like a charm, and switching to the "Available" tab I was able to search for and install all the necessary plugins I used during my first installation. Unfortunately though, my builds still met with the exact same error message. At least I knew it was no longer just a proxy issue.

##Jenkins SSH
My search led me to investigate SSH. If Jenkins wasn't using my machine's proxy by default, then maybe it wasn't using the SSH Key I setup to validate my machine with GitLab. True enough, it seems that Jenkins runs as a different user in Windows (System by default), and so is not able to make use of the key tagged against the currently logged-on user account. There were many guides that detailed how to get around this, like creating a Jenkins user, generating an SSH key for it, and the like, but for the purposes of my POC I didn't want the hassle of maintaining another user account. There must be a quicker way to do this. 

My persistence in laziness paid off and I was able to find the directory that Jenkins will use to locate its SSH keys. Here's how I set things up:

1. Go to C:\Windows\SysWOW64\config\systemprofile
2. Create a .ssh directory
3. Open the .ssh directory for the local user account where your previously generated key resides (e.g. C:\Users\Username\.ssh)
4. Copy the contents of C:\Users\Username\.ssh into C:\Windows\SysWOW64\config\systemprofile\.ssh

![SSH Directory](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/sshdirectory.jpg)

I immediately asked Jenkins to build my project and eagerly awaited the output.

```
Started by user anonymous
Building in workspace C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace
 > C:\Program Files (x86)\Git\cmd\git.exe rev-parse --is-inside-work-tree # timeout=10
Fetching changes from the remote Git repository
 > C:\Program Files (x86)\Git\cmd\git.exe config remote.origin.url git@XXXXXXX.XXXX.XXXXX:XXXX/continuousdeliverypoc.git # timeout=10
Fetching upstream changes from git@XXXXXXX.XXXX.XXXXX:XXXX/continuousdeliverypoc.git
 > C:\Program Files (x86)\Git\cmd\git.exe --version # timeout=10
 > C:\Program Files (x86)\Git\cmd\git.exe -c core.askpass=true fetch --tags --progress git@XXXXXXX.XXXX.XXXXX:XXXX/continuousdeliverypoc.git +refs/heads/*:refs/remotes/origin/* # timeout=20
 > C:\Program Files (x86)\Git\cmd\git.exe rev-parse "refs/remotes/origin/master^{commit}" # timeout=10
 > C:\Program Files (x86)\Git\cmd\git.exe rev-parse "refs/remotes/origin/origin/master^{commit}" # timeout=10
Checking out Revision 098d8e8b560ad5bf6a602b4e2bc577b41a52927f (refs/remotes/origin/master)
 > C:\Program Files (x86)\Git\cmd\git.exe config core.sparsecheckout # timeout=10
 > C:\Program Files (x86)\Git\cmd\git.exe checkout -f 098d8e8b560ad5bf6a602b4e2bc577b41a52927f
First time build. Skipping changelog.
Path To MSBuild.exe: C:\Windows\Microsoft.NET\Framework\v4.0.30319\MSBuild.exe
Executing the command cmd.exe /C " C:\Windows\Microsoft.NET\Framework\v4.0.30319\MSBuild.exe CDServices.sln " && exit %%ERRORLEVEL%% from C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace
[workspace] $ cmd.exe /C " C:\Windows\Microsoft.NET\Framework\v4.0.30319\MSBuild.exe CDServices.sln " && exit %%ERRORLEVEL%%
Microsoft (R) Build Engine version 4.0.30319.33440
[Microsoft .NET Framework, version 4.0.30319.33440]
Copyright (C) Microsoft Corporation. All rights reserved.
...
```

Progress! No timeout error, and the files were actually downloaded into my Jenkins workspace.

```
...
Building the projects in this solution one at a time. To enable parallel build, please add the "/m" switch.
Build started 4/10/2015 10:56:45 AM.
Project "C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace\CDServices.sln" on node 1 (default targets).
ValidateSolutionConfiguration:
  Building solution configuration "Debug|Any CPU".
Project "C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace\CDServices.sln" (1) is building "C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace\CDServices\CDServices.csproj" (2) on node 1 (default targets).
C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace\CDServices\CDServices.csproj(259,3): error MSB4019: The imported project "C:\Program Files (x86)\MSBuild\Microsoft\VisualStudio\v11.0\WebApplications\Microsoft.WebApplication.targets" was not found. Confirm that the path in the <Import> declaration is correct, and that the file exists on disk.
Done Building Project "C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace\CDServices\CDServices.csproj" (default targets) -- FAILED.
Done Building Project "C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace\CDServices.sln" (default targets) -- FAILED.

Build FAILED.

"C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace\CDServices.sln" (default target) (1) ->
"C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace\CDServices\CDServices.csproj" (default target) (2) ->
  C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace\CDServices\CDServices.csproj(259,3): error MSB4019: The imported project "C:\Program Files (x86)\MSBuild\Microsoft\VisualStudio\v11.0\WebApplications\Microsoft.WebApplication.targets" was not found. Confirm that the path in the <Import> declaration is correct, and that the file exists on disk.

    0 Warning(s)
    1 Error(s)

Time Elapsed 00:00:01.57
Build step 'Build a Visual Studio project or solution using MSBuild' marked build as failure
Finished: FAILURE
```

Welp. Not as much progress as I had hoped. 

##Reconfiguring MSBuild

My trip down the rabbit hole that is Google led me to this [article](http://stackoverflow.com/questions/20472177/msb4019-from-vs2012-solution-when-building-on-jenkins) from Stack Overflow. It essentially just states that you should add 
```
/p:VisualStudioVersion=12.0
```
as a command line argument to your MSBuild build step.

![MSBuild Reconfig](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/msbuildreconfig.jpg)

Progress once more, but not yet success. I was met with about five pages of these:

```
          Considered "bin\Microsoft.Owin.Host.SystemWeb.exe", but it didn't exist.
  Primary reference "Microsoft.Owin.Security".
C:\Windows\Microsoft.NET\Framework\v4.0.30319\Microsoft.Common.targets(1605,5): warning MSB3245: Could not resolve this reference. Could not locate the assembly "Microsoft.Owin.Security". Check to make sure the assembly exists on disk. If this reference is required by your code, you may get compilation errors. [C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace\CDServices\CDServices.csproj]
          For SearchPath "{HintPathFromItem}".
          Considered "..\packages\Microsoft.Owin.Security.2.0.0\lib\net45\Microsoft.Owin.Security.dll", but it didn't exist.
          For SearchPath "{TargetFrameworkDirectory}".
          Considered "C:\Program Files (x86)\Reference Assemblies\Microsoft\Framework\.NETFramework\v4.5\Microsoft.Owin.Security.winmd", but it didn't exist.
          Considered "C:\Program Files (x86)\Reference Assemblies\Microsoft\Framework\.NETFramework\v4.5\Microsoft.Owin.Security.dll", but it didn't exist.
          Considered "C:\Program Files (x86)\Reference Assemblies\Microsoft\Framework\.NETFramework\v4.5\Microsoft.Owin.Security.exe", but it didn't exist.
          Considered "C:\Program Files (x86)\Reference Assemblies\Microsoft\Framework\.NETFramework\v4.5\Facades\Microsoft.Owin.Security.winmd", but it didn't exist.
          Considered "C:\Program Files (x86)\Reference Assemblies\Microsoft\Framework\.NETFramework\v4.5\Facades\Microsoft.Owin.Security.dll", but it didn't exist.
          Considered "C:\Program Files (x86)\Reference Assemblies\Microsoft\Framework\.NETFramework\v4.5\Facades\Microsoft.Owin.Security.exe", but it didn't exist.
          For SearchPath "{Registry:Software\Microsoft\.NETFramework,v4.5,AssemblyFoldersEx}".
          Considered AssemblyFoldersEx locations.
          For SearchPath "{AssemblyFolders}".
          Considered "C:\Program Files (x86)\Microsoft SQL Server\110\SDK\Assemblies\Microsoft.Owin.Security.winmd", but it didn't exist.
          Considered "C:\Program Files (x86)\Microsoft SQL Server\110\SDK\Assemblies\Microsoft.Owin.Security.dll", but it didn't exist.
          Considered "C:\Program Files (x86)\Microsoft SQL Server\110\SDK\Assemblies\Microsoft.Owin.Security.exe", but it didn't exist.
          Considered "C:\Program Files\IIS\Microsoft Web Deploy V3\Microsoft.Owin.Security.winmd", but it didn't exist.
          Considered "C:\Program Files\IIS\Microsoft Web Deploy V3\Microsoft.Owin.Security.dll", but it didn't exist.
          Considered "C:\Program Files\IIS\Microsoft Web Deploy V3\Microsoft.Owin.Security.exe", but it didn't exist.
          Considered "C:\Program Files (x86)\Microsoft SQL Server\110\SDK\Assemblies\Microsoft.Owin.Security.winmd", but it didn't exist.
          Considered "C:\Program Files (x86)\Microsoft SQL Server\110\SDK\Assemblies\Microsoft.Owin.Security.dll", but it didn't exist.
          Considered "C:\Program Files (x86)\Microsoft SQL Server\110\SDK\Assemblies\Microsoft.Owin.Security.exe", but it didn't exist.
          Considered "C:\Program Files (x86)\Reference Assemblies\Microsoft\Framework\v3.5\Microsoft.Owin.Security.winmd", but it didn't exist.
          Considered "C:\Program Files (x86)\Reference Assemblies\Microsoft\Framework\v3.5\Microsoft.Owin.Security.dll", but it didn't exist.
          Considered "C:\Program Files (x86)\Reference Assemblies\Microsoft\Framework\v3.5\Microsoft.Owin.Security.exe", but it didn't exist.
          Considered "C:\Program Files (x86)\Reference Assemblies\Microsoft\Framework\v3.0\Microsoft.Owin.Security.winmd", but it didn't exist.
          Considered "C:\Program Files (x86)\Reference Assemblies\Microsoft\Framework\v3.0\Microsoft.Owin.Security.dll", but it didn't exist.
          Considered "C:\Program Files (x86)\Reference Assemblies\Microsoft\Framework\v3.0\Microsoft.Owin.Security.exe", but it didn't exist.
          Considered "C:\Program Files (x86)\Microsoft.NET\ADOMD.NET\110\Microsoft.Owin.Security.winmd", but it didn't exist.
          Considered "C:\Program Files (x86)\Microsoft.NET\ADOMD.NET\110\Microsoft.Owin.Security.dll", but it didn't exist.
          Considered "C:\Program Files (x86)\Microsoft.NET\ADOMD.NET\110\Microsoft.Owin.Security.exe", but it didn't exist.
          For SearchPath "{GAC}".
          Considered "Microsoft.Owin.Security", which was not found in the GAC.
          For SearchPath "{RawFileName}".
          Considered treating "Microsoft.Owin.Security" as a file name, but it didn't exist.
          For SearchPath "bin\".
          Considered "bin\Microsoft.Owin.Security.winmd", but it didn't exist.
          Considered "bin\Microsoft.Owin.Security.dll", but it didn't exist.
          Considered "bin\Microsoft.Owin.Security.exe", but it didn't exist.
...
```

##Locating Project References

It seems Jenkins wasn't able to locate the reference DLLs used in the project. I've never had to deal with this before, as Visual Studio/NuGet which automagically handled these for the most part when I get a solution from source control. At my wit's end, I decided to include the "packages" directory into my Git project. This directory contains a copy of all the DLL files that the project references. The contents of this directory were all included in the default .gitignore file for Windows and .NET applications so they were excluded from the list of artefacts tracked by Git. I'm not really sure if this is the best and only approach, but it worked.

```
...
Copying file from "C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace\packages\Microsoft.Owin.Security.Cookies.2.0.0\lib\net45\Microsoft.Owin.Security.Cookies.xml" to "bin\Microsoft.Owin.Security.Cookies.xml".
  Copying file from "C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace\packages\Microsoft.Owin.Security.OAuth.2.0.0\lib\net45\Microsoft.Owin.Security.OAuth.xml" to "bin\Microsoft.Owin.Security.OAuth.xml".
  Copying file from "C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace\packages\Microsoft.Owin.Security.Google.2.0.0\lib\net45\Microsoft.Owin.Security.Google.xml" to "bin\Microsoft.Owin.Security.Google.xml".
  Copying file from "C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace\packages\Microsoft.Owin.Security.Twitter.2.0.0\lib\net45\Microsoft.Owin.Security.Twitter.xml" to "bin\Microsoft.Owin.Security.Twitter.xml".
  Copying file from "C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace\packages\Microsoft.Owin.Security.MicrosoftAccount.2.0.0\lib\net45\Microsoft.Owin.Security.MicrosoftAccount.xml" to "bin\Microsoft.Owin.Security.MicrosoftAccount.xml".
  Copying file from "C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace\packages\Microsoft.AspNet.WebApi.Owin.5.0.0\lib\net45\System.Web.Http.Owin.xml" to "bin\System.Web.Http.Owin.xml".
_CopyAppConfigFile:
  Copying file from "Web.config" to "bin\CDServices.dll.config".
CopyFilesToOutputDirectory:
  Copying file from "obj\Debug\CDServices.dll" to "bin\CDServices.dll".
  CDServices -> C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace\CDServices\bin\CDServices.dll
  Copying file from "obj\Debug\CDServices.pdb" to "bin\CDServices.pdb".
Done Building Project "C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace\CDServices\CDServices.csproj" (default targets).
Done Building Project "C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace\CDServices.sln" (default targets).

Build succeeded.
    0 Warning(s)
    0 Error(s)

Time Elapsed 00:00:02.63
Finished: SUCCESS
```

That's it, concept proven! Congratulations to us!

#<a name="tldr"></a>TL;DR

1. Install Visual Studio (not in scope of this article).
2. Set up Git (not in scope of this article).
3. Generate and SSH Key for your machine and link it to your GitLab account (not in scope of this article).
4. Download and install [Jenkins](https://jenkins-ci.org/).
5. Configure the Jenkins Proxy via Jenkins -> Manage Jenkins -> Manage Plugins -> Advanced. Make sure to hit "Check now" at the bottom of the page.
6. Download and install the following plugins from Jenkins -> Manage Jenkins -> Manage Plugins -> Available
  *MSBuild
  *Git
  *GitClient
  *GitLab Hook
7. Configure Git via Jenkins -> Manage Jenkins -> Configure System.
![Git Config](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/gitconfig.jpg)
8. Configure MSBuild via Jenkins -> Manage Jenkins -> Configure System.
![MSBuild Config](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/msbuildconfig.jpg)
9. Create a Jenkins Job.
10. Configure Job details.
![Project Details](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/projectbasics.jpg)
![Project Git Details](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/projectgit.jpg)
![Project Build Triggers](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/projectbuildtriggers.jpg)
![MSBuild Reconfig](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/msbuildreconfig.jpg)
11. Copy your generated ssh key (normally located in C:\Users\Username\.ssh) into C:\Windows\SysWOW64\config\systemprofile\.ssh.
12. Ensure that the "packages" directory of your solution and all of its contents are explicitly added into your Git repository (override .gitignore by explicitly adding these files through git add).
13. Run your build job.
14. Profit.
