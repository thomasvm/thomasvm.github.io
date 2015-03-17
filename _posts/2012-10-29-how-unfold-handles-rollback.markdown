---
layout: post
title: "How Unfold handles rollback"
date: 2012-10-29 12:53
comments: true
categories: ['asp.net', 'deployment', 'unfold', 'rollback']
---
Rollback should be an important aspect of any deployment solution, simply because we tend to mess things up. There are a lot of 
things that can go wrong, and sometimes you only realize something is broken when the code is already running. Yes, yes, I know,
my tests should have full coverage, I should perform user testing on different environments, and my code should be peer reviewed.
But even then things can and will go wrong, there's no escaping it.

And when that happens, you find yourself saying: "If only we could revert to the previous version, until we fixed this problem".
Well, Unfold is here to help.

### Listing the possible versions
As we explained before Unfold keeps old releases around for some time. By default, it keeps the last 5 releases. 
Before you execute a rollback it's best to check exactly what releases are available. To do so, simply issue the following 
command in a PowerShell:

    .\unfold.ps1 listremoteversions -on dev

The `listremoteversions` task does nothing special, it just checks which _releases_ are available on the deployment target and
prints them in PowerShell the console. The output looks like this

{% img /images/blog/listremoteversions.png %} 

These releases are what is available on the deployment target. The first part of the folder name is the timestamp of deployment,
the second part is the short hash of the revision is what deployed from. The number that are in front of the folder names 
(01, 02, ...) is what we'll use when executing a rollback.

### Executing the rollback
Now that you've picked the version you want to rollback to, you can perform the rollback. This command for example will rollback
to the third release - on the dev environment:

    .\unfold.ps1 rollback -on dev -properties @{to=3}

In short, this will execute the rollback task on the dev environment. The rollback task however, needs a parameter. 
It needs to know which version to rollback to. For parameters we depend on the properties system of psake, for full documentation
check [this page](https://github.com/psake/psake/wiki/How-can-I-override-a-property-defined-in-my-psake-script%3F).

### How rollback works
In a [previous post](http://thomasvm.github.com/blog/2012/10/10/the-unfold-tasks/) we have explained the default tasks that together 
compose a deployment. Basically there are two major _parts_ to this. The first part is getting the last version of the code 
from scm, building and releasing it. The second part deals with setting up IIS and configuring the application. 

In this rollback scenario however we don't need to perform the first part. Also, because our system uses psake tasks we can
**simply re-use those tasks** in the rollback flow. As a consequence any **task hooks** that are attached to - let's say - 
the `setupiis` task, will also be executed during rollback, making sure that also your own customizations to the deployment flow
will be applied when performing a rollback.

## Conclusion
You have now seen how Unfold handles rollback. Rollback is made easy because of two reasons:

* we keep older versions around. Because we do so, there's always something ready to rollback to.
* because we use psake and have split the deployment into several smaller tasks, we can re-use a lot of logic and skip the
  parts we don't need.

Rollback is now a matter of seconds and very trivial to perform. So go on, _mess things up_. 

