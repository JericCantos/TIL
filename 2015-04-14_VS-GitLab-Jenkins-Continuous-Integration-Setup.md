Jenkins is currently being used by open source teams in my organization as their Continuous Integration solution. Coming from a Microsoft-centric team, we rely more on MSBuild/TFS Build Server and Visual Studio Release Management to do this for us. I wondered, though, if we could use Jenkins for MS projects instead, which would allow our organization to maybe focus on just one tool and cater to a wide range of projects. My goal was to build a pipeline from Visual Studio going into GitLab for source control and then moving into Jenkins for continuous integration, scheduled builds, and all that jazz. Here's my adventure.

#Installing Jenkins

I've been using Visual Studio and GitLab for a while now so my development machine is already properly configured to use both. I just needed to install and configure Jenkins and I'd be all set. So I downloaded it from the [Jenkins site] (https://jenkins-ci.org/), which thankfully provides a native Windows Package. It offered a no-fuss installation, and within Minutes, Jenkins was up and running. So far so good.

#Installing Jenkins Plugins

Next, I needed to teach Jenkins how to build Microsoft .NET solutions. A quick Google search led me to this article - [Jenkins for .net in 5 Minutes], (http://justinramel.com/2013/01/15/5-minute-setup/) which informed me that there's already an MSBuild plugin available for Jenkins. Well, it turns out that it doesn't take just five minutes (at least not for me).

##Plugin Woes

The guide mentions that I should be able to simply search for the MSBuild plugin through the Jenkins UI via Jenkins -> Manage Jenkins -> Manage Plugins -> Available. Lo and behold, my Available Plugins page is empty.

![Not what I was seeing](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/availableplugins.jpg)
**Not what I was seeing.**

That's weird. How am I going to get the MSBuild plugin now? Well, somehow through more Google consultations I found a [directory of all Jenkins plugins](https://updates.jenkins-ci.org/download/plugins/) with a direct download link. Great! I quickly downloaded the latest version of the MSBuild plugin to my local machine, and then installed it through Jenkins -> Manage Jenkins -> Manage Plugins -> Advanced -> Upload Plugin.

Next on the list is Git and GitLab. Through various sources like [this one](http://doc.gitlab.com/ee/integration/jenkins.html) I found out that I require the Git, Git Client, and GitLab Hook plugins. I downloaded these and installed them like I did with MSBuild, and then configured them like so (Jenkins -> Manage Jenkins -> Configure System).

![MSBuild Config](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/msbuildconfig.jpg)
![Git Config](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/gitconfig.jpg)

#First Job Setup 

I was supposedly all set to set up my continuous integration job. Here are the pertinent details I entered then:

![Project Details](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/projectbasics.jpg)
![Project Git Details](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/projectgit.jpg)
![Project Build Triggers](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/projectbuildtriggers.jpg)
![Project MSBuild Configuration](https://github.com/JericCantos/TIL/blob/master/Images/Jenkins-CI/projectmsbuild.jpg)

Hitting "Build" rewarded me with the following output.

```
Started by user anonymous
Building in workspace C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace
Cloning the remote Git repository
Cloning repository git@absadsivir03.phl.hp.com:cantosj/continuousdeliverypoc.git
 > C:\Program Files (x86)\Git\cmd\git.exe init C:\Jenkins\jobs\ContinuousDeliveryPOC\workspace # timeout=10
Fetching upstream changes from git@absadsivir03.phl.hp.com:cantosj/continuousdeliverypoc.git
 > C:\Program Files (x86)\Git\cmd\git.exe --version # timeout=10
 > C:\Program Files (x86)\Git\cmd\git.exe -c core.askpass=true fetch --tags --progress git@absadsivir03.phl.hp.com:cantosj/continuousdeliverypoc.git +refs/heads/*:refs/remotes/origin/*
ERROR: Timeout after 10 minutes
ERROR: Error cloning remote repo 'origin'
ERROR: Error cloning remote repo 'origin'
Finished: FAILURE
```

Very nice. This problem stumped me for quite a while and Google wasn't quite as helpful. I tried extending the timeout, to changing my machine's proxy. I was able to ping the GitLab server and I was able to fetch, pull, and push changes via GitExtensions but nothing I did allowed Jenkins to do the same.