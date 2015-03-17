---
layout: post
title: The Unfold deployment tasks
date: 2012-10-10 20:00
comments: true
published: true
categories: ["asp.net", deployment, unfold]
---

In the [previous post](http://thomasvm.github.com/blog/2012/10/02/introducing-unfold/) I explained how to install 
and use [Unfold](https://github.com/thomasvm/unfold), the PowerShell deployment solution for .net web applications. Several 
times I mentioned _the default unfold tasks_. That's right, Unfold comes with a set of Psake tasks that are the framework
for a typical deployment flow. In this blog post, we'll explain the different tasks and how they cooperate to perform the
deployment.

Let's start by listing all of the tasks that are available for your deployment. Simply

    .\unfold.ps1 -docs

This will print out a list of all tasks that you can invoke from the command-line. The `deploy` task is the most important one.
It performs all necessary operations to fully deploy your application. This is an explanation of the subtasks that is it invokes. 
These _tasks_ are simply defined using [psake](https://github.com/psake/psake). 

So if you execute

    .\unfold.ps1 deploy -to staging

Then these are the tasks that get executed. 

* **setup**:
  
  The setup tasks is pretty simple. It ensures that the `$config.basePath` value is available on the deployment target. Once it's
  there we can start all kinds of other operations like updating the source code or composing a "release". 
  
  It's also important to note that after this task has executed, `Invoke-Script` will automatically change the current 
  location to the `$config.basePath` value when executing a script block. As a result all operations within 
  an `Invoke-Script` block can use relative paths.

* **updatecode**:
 
  The purpose of updatecode is pretty obvious. It ensures that the `$config.basePath` folder contains a subfolder called `code`
  that contains the latest version of your code. This code if fetched from source control, not a local copy. If the folder is
  already there, we perform an update to the latest version. If not, then we simply perform an initial checkout.  

  You can configure where to get the code from using two configuration calls:

``` posh
Set-Config scm git
Set-Config repository "git@github.com:your-account/your-repository.git"
```  

  Currently we support two scm's: git and svn. We take pull requests for others. 

  It's important to note that by default this fetch from source control happens _on the deployment machine_. So you have to make
  sure that your deployment machine can access source control (ssh keys, firewall restrictions, etc). This also means your 
  code will also be built on the remote machine. If you don't want code on the remote machine, you can force unfold to
  perform all tasks up to release to be executed on the local machine. To do so, add this to your deploy recipe.

``` posh
Set-Config localbuild $true
Set-Config localbuildpath "c:\path\to\perform\local\build"
```  

* **build**:

  Unlike Ruby or PHP projects .net projects need to be built using msbuild, there's no way around it. By default, when there's no
  specific configuration, we look for a Web Application project inside the code repository. If we find one, that's what we
  build. However it is also possible to specify which projects or solution files that we need to build. To do so, simply
  specify a configuration variable containing a list of msbuild files to execute. Additionally, you can also specify the 
  build configuration we need to use, but it's not required.

``` posh
Set-Config msbuild @('.\code\path\to\your\Project.csproj')
Set-Config buildConfiguration "Release"
```  

  That's all we need to know.

* **release**:

  Once your code is built successfully, we can create what we call a _release_. A release is folder that contains the build 
  result for a given revision. It should be an application that can run successfully. We put those under the `$config.basePath`
  and the folder name for a typical _release_ looks like this:

    20120905_1218_a6b9e4c_YourProject

  It is composed of 
  1. the date at which the release was created
  1. the hour of creation
  1. a scm revision reference, for git (as shown here) this is the revision hash
  1. the name of your project

  This offers you a way to immediately see when a release was created, and from which point in your revision history. This can
  be of great help when problems are occuring in your application and you want to know from which revision the code was built.

  If your build msbuild project is a Web Application Project, then we copy the build output to a folder below this called `web`.
  If not, you'll have to implement a release operation yourself. We'll explain this in a future blog post.

  We copy it under a `web` folder, because it will allow you to also _release_ other build artefacts under different folders. 
  Maybe you have a set of Windows Services, or maybe you should also install a dedicated web service. If so, then these can be
  _released_ next to the web output.

  Once this task is completed, the name of the release is available on the `$config` object as `$config.releasepath`.

  After you have executed multiple deployments, the `$config.baseFolder` will look like this.
 
  ![layout]({{ site.url }}/images/blog/releaselayout.png)

  What you see are:
  1. Five releases, each contains a web subfolder, and possibly others
  2. a code folder, which contains the current scm checkout
  3. a current link, established by the finalize task

  We keep a couple of releases around, because this allows us to very quickly rollback in case something goes wrong. When that
  happens, we can simply re-point IIS to one of the previous releases, and we're all fine again. 

* **releasefinalize**:

  this is an unfold internal task, that will finalize the work that is done in the previous step.

* **setupapppool**:  

  As the name explains, this will create an IIS Application Pool for your application. We consider it best practice to give
  each application its own applcation pool. This task ensures that the application pool is setup. You can specify the name
  of the applicatin pool using `Set-Config apppool "yourapppool"`, if that's not set, we use the `iisname` (which is the 
  name of the website under IIS). If that's not set either, we use the `project` configuration variable.

  All operations on IIS operations are performed using the 
  [WebAdministration Module](http://technet.microsoft.com/en-us/library/ee909471.aspx)

* **uninstallcurrentrelease**:

  If the web folder of a release contains a file called `App_Offline.html`, we rename it to `App_Offline.htm`. This will disable 
  your web application until the new release is fully configured. If there is other stuff that needs to be done - like 
  stopping Windows Services, killing an exe - than this is the place where you will hook your custom operations onto. 
  More on this in a future post.
 
* **setupiis**:

  We're almost there. This task configures the IIS website for you, it uses the application pool we created a couple of steps 
  ago, and creates or updates a website to point to the _release_ we created. This task uses two configuration variables:

``` posh
Set-Config iisname "www.yoursite.com"
Set-Config bindings @(
                      @{protocol="http";bindingInformation="*:80:www.yoursite.com"},
                      @{protocol="http";bindingInformation="*:80:yoursite.com"}
                     )
```

  This IIS Name, is the name that's used inside IIS, the bindings configuration defines the IIS bindings for your website. For
  more information on defining bindings, simply check the WebAdministration documentation.

* **finalize**:

  This task wraps it all up. It creates a link called _current_ pointing to the latest release. That way you can quickly navigate
  to the currently deployed version, and it allows us to detect which version is the current one. It moves the `App_Offline.htm`
  out of the way again, so the newly created release becomes active.

  Finally, it purges old releases. There is the `$config.keep` option, which by default is set to 5, and it defines the number
  of releases you want to keep. If you want 10, simply add this to your deploy recipe

``` posh  
Set-Config keep 10
```

## Conclusion
You've now seen which steps Unfold performs in order to fully deploy your application. Most of them have options 
which you can simply configure using `Set-Config` calls.

It's important to note that this flow is completely extensible. We'll dedicate a future blog post to how you can create custom
tasks and hook them to the default flow or how you can override them. By doing so, you can perfectly tailor deployments for your
own needs.
