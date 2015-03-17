---
layout: post
title: "Introducing Unfold"
date: 2012-10-02 20:17
comments: true
categories: ["deployment", "asp.net", "capistrano", "unfold"]
---

Unfold is a **capistrano**-like deployment solution for **.net applications**. 
I created it because it didn't find any solution that gave me enough flexibility and at the same
time provided enough functionality out-of-the-box to deploy _standard_ applications. In Unix environments,
when it comes to deploying web applications, there's nothing that rivals [Capistrano](http://en.wikipedia.org/wiki/Capistrano).
So what I wanted was pretty simple: a capistrano for .net/windows. This led me to the following design decisions:

* **Powershell**: 

  we use _only_ Powershell. In terms of automating things, there's
  nothing more powerful than Powershell in the Windows world. You can execute any application, 
  there's a wealth of Modules that allow you to automate anything from installing a Windows Service,
  configuring IIS or creating folders

* **Psake**: 

  Capistrano is built on top of [Rake](http://docs.rubyrake.org/tutorial/index.html) and
  it's a great way of splitting a deployment process into smaller components. Makes your _flow_ easy
  to understand. I wanted that too, so I simply used [Psake](https://github.com/psake/psake).

* **Powershell Remoting**: 

  for access to remote machines, we depend on Powershell Remoting. It makes
  it very easy to execute scripts or ScriptBlocks on remote machines. For an easy explanation of 
  Powershell Remoting, check [here](http://www.howtogeek.com/117192/how-to-run-powershell-commands-on-remote-computers/). The
  unfold wiki contains an article on [how to setup Powershell remoting](https://github.com/thomasvm/unfold/wiki/Setting-up-Powershell-Remoting) 
  for your own target server.

* **Good defaults** with **good extensibilty**: 

  Unfold had to contain a default deployment flow that is *good enough* for simple
  applications, if you need customization, then you should be able to built upon that flow. 

The source code is on [GitHub](https://github.com/thomasvm/unfold).

## 1. Installation  
  
To install unfold, open a Powershell command-prompt and issue:

    cd ~\Documents\WindowsPowerShell\Modules
    git clone https://github.com/thomasvm/unfold.git 

To check whether the installation was successful, import the module.

    Import-Module unfold    

Great!

### Add unfold to your project

Now navigate towards a project folder where you want to specify your deployment recipe. For example, if your source code 
sits inside a `src` folder, then the parent folder is ideal. Once you're there do:

    Install-Unfold 

This will create a folder called `deployment`, containing two files 

* *deploy.ps1*:

  this file is the most important file, as it contains all your configuration and deployment steps. 
  It's the file that you customize to define your deployment.

* *unfold.ps1*:

  this file is for loading the Unfold module and passing the command-line parameters in to it. More on that later

## 2. The `deploy.ps1` file
Let's have a look at a slightly modified `deploy.ps1` file:

``` posh
# Configuration
Set-Config project "YourProjectName"

Set-Config scm git
Set-Config repository "git@github.com:yourusername/a-project.git"

# Environment to use when not specified
Set-Config default dev

# Specify a custom set of build files, don't forget
# to prepend with ".\code\" because that is the remote checkout folder
Set-Config msbuild = @('.\code\src\YourWebProject\YourWebProject.csproj')

Set-Environment dev {
    # machine to deploy to
    Set-Config machine "localhost"
    Set-Config basePath "c:\inetpub\wwwroot\dev\YourProject" 
}

Set-Environment staging {
    Set-Config machine "your.machine.name"
    Set-Config basePath "d:\sites\YourProjectName\staging"
}

# Import the default Unfold deployment flow
Import-DefaultTasks

# Custom task
task ping {
    Invoke-Script {
        Write-Host $env:ComputerName
        Write-Host $(pwd)
        Write-Host "On this machine we'll deploy to $($config.basePath)"
    }
}
# Set deploy as default task
task Default -depends "deploy"
```

Yep, that's all there is to it, this is a full deployment script for a standard asp.net application. To test whether this works, 
open a PowerShell inside the folder and issue:

    .\unfold.ps1 deploy -to dev

In other words: call the launcher script `unfold.ps1` and let it invoke the `deploy` task (that is imported 
through `Import-DefaultTasks`) and deploy towards the `dev` environment.

Of course, most of the time you'll want to extend this. That's for a later post, for now, 
we'll have a look at what's inside this file.

### 2.a The configuration part
The `deploy.ps1` file starts off with a set of calls to a function called **`Set-Config`**. These calls populate a **`$config`** variable that
is available throughout your deployment. Most of the variables in the example above are used by the standard Unfold deployment 
tasks, but you can add your own if you need to. There are no restrictions. You've probably also noticed that some calls are wrapped within a
**`Set-Environment`** function call. These configuration settings are considered to be environment-specific. In the above example, 
if you deploy using

    .\unfold.ps1 deploy -to dev

Then `$config.machine` will be _localhost_, and the `$config.basePath` will point to _c:\inetpub\wwwroot\dev\YourProject_. On 
the other hand, if the -to parameter is set to "staging"

    .\unfold.ps1 deploy -to staging

Then this will assign the values will be _your.machine.name_ and _d:\sites\YourProjectName\staging_.

### 2.b The tasks part
Now that the configuration part is done, we're ready to define some tasks. If your application doesn't require too much
special, it's usually enough to just import the default Unfold tasks. That's exactly what this line of code does

    Import-DefaultTasks

It imports a set of Psake tasks that together compose the deployment flow. A future blog post will explain which tasks 
there are and what their purpose is.

Next, we define a custom task, this can be used from anything like restarting windows services, clearing a cache folder, or checking
whether a process is running. The little task that we define here is to check whether we can connect to our deployment machine.

``` posh
# Custom task
task ping {
    Invoke-Script {
        Write-Host $env:ComputerName
        Write-Host $(pwd)
        Write-Host "On this machine we'll deploy to $($config.basePath)"
    }
}
```

This task writes out the computername, the current folder, and the basePath property of the `$config` global variable we composed
using our `Set-Config` calls. What is important  however, is that we do this inside a powershell scriptblock that we pass to a
function called **`Invoke-Script`**.

This `Invoke-Script` is a _very, very_ important part of Unfold because it allows you to write deployment operations that work on
both your local machine as on a remote machine.

### 2.c Invoke-Script
The `Invoke-Script` function has 1 important parameter, namely the scriptblock that you pass into it. With that scriptblock, 
`Invoke-Script` performs the following operation.

* It checks the `$config.machine` parameter, and based on its value it decides where to execute the scriptblock.
* If the value equals "localhost", then `Invoke-Script` simply changes the current directory to `$config.basePath` and
  executes the provided scriptblock.
* If however, the value of `$config.machine` is something else, like an IP address, a computername or a remote hostname, then
  it creates a new PS-Session through the [New-PSSession](http://technet.microsoft.com/en-us/library/hh849717.aspx) function 
  and uses that session to execute the scriptblock **on the remote machine**. 

So, `Invoke-Script` allows you to write very transparent deployment code, but where to execute it, is completely dependant on
what the current configuration is.

In the above example, if the deployment is issues like this: 

    .\unfold.ps1 ping -to dev

Then the ping task will be executed and will write out the computername of the local machine. However, if we ask Unfold to
execute the ping task on staging, it will print out the computername of *your.machine.name*.

    .\unfold.ps1 ping -to staging

You may ask the question: "where do you get the credentials from when opening a session on a remote machine?". The answer is 
very simple: from the *Windows Credential Manager*. Just open the Windows Credential Manager from your start menu and 
add a _generic principal_ to it. This means we don't have to store credentials in our deployment scripts. 
The name for the principal must reflect the name you've configured using:

``` posh
Set-Config machine "your.machine.name"
```

The result of all this is that `Invoke-Script` allows you to write deployment code that is easily portable between environments and it makes setting up
a new environment for your deployments just a matter of adding an extra `Set-Environment` call to your deployment script.

## 3.Conclusion
You've now seen how you can easily create an Unfold deployment for your project, how the configuration system works 
and how to define customs deployment tasks.

In future posts, we'll explain what the default deployment tasks are defined by Unfold and how to extend 
them with your own tasks. By doing so, you'll be able to complete customize your deployment and make it
completely automated and runnable from the command-line.
