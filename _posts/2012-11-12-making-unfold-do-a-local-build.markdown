---
layout: post
title: "Making unfold build your application locally"
date: 2012-11-12 12:47
comments: true
categories: [ "deployment", "asp.net", "unfold" ]
---
By default [Unfold](https://github.com/thomasvm/unfold) performs almost all operations on 
the target machine of your deployment. This means that the checkout from source control, the
building of the project and the creation of the _release_ folder for your application all
happen on the remote machine.

### The pros and cons of remote builds

This approach has quite some benefits: 

* you're sure that all **dependencies** are satisfied on the remote machine. Sometimes a local
  build can trick you into thinking that your application runs just fine, but once you start
  running on a remote machine it appears that assemblies are missing.

* Only the source control **differences** need to be fetched over the network in order for the target
  machine to contain the last version of your code. That way we don't have to send the entire
  application across the wire each time we perform a new deployment.

* Because there's less data to be sent over the network, deployments can happen faster.

Of course, there are also scenarios where performing the build on the remote machine simply isn't
possible or wanted. 

* _Does the deployment server have access to source control?_
  Because unfold takes your code from source control, your deployment target **needs access to
  your source control** server. In case of Git for example, you need to set the ssh keys. It
  also might be that your source control server sits behind a firewall and is simply not 
  accessible from the deployment machine.

* _Do you depend on SDKs?_
  Microsoft sometimes makes it [particularly](http://blog.hinshelwood.com/solution-getting-silverlight-to-build-on-team-foundation-build-services-2010/) 
  [hard](http://blog.gasparnagy.com/2011/03/building-windows-phone-7-applications.html) to 
  build an application on a machine that does not have Visual Studio installed. Especially when **depending on
  SDKs** you can run into trouble. It's crazy, for sure, but unfortunately it is a problem that we need 
  to handle for our deployment scenarios

* _Do you want the deployment server to access source control?_:
  Maybe you just don't want to give your deployment server access to source control, for whatever
  reason. No problem, simply switch to local builds. 
  
### Switching to local build
  
Luckily, making Unfold perform the _checkout + build_ on your local machine is very easy. As
can be expected, the configuration happens through some `Set-Config` calls. **Two** to be precise

``` posh
Set-Config localbuild $true
Set-Config localbuildpath "c:\path\to\a\local\checkout"
``` 

That was easy.  

When these two configuration variables are available, Unfold will take the following approach
when deploying:

* The source is fetched from source control on your local machine, in the `localbuildpath` to
  exact. 
* The build is executed on the local machine
* Next, we create the `release` on the local machine as well. 
* Finally the output of the `release` task put into a zipfile, and that zipfile is copied 
  over the PowerShell Remoting Session, towards the remote machine where it is extracted
  inside the `basePath` configuration variable. 
* From there on, the normal deployment behaviour can take over again, as all we need to do
  now is setup the remote machine to use the new _release_ we created.

## Conclusion  

Unfold makes it easy for you to configure where the _build_ and _release_ of your application 
needs to happen, all that's needed are two configuration variables. 

Whether or not you use the localbuild scenario depends on the following considerations

* Does my deployment machine have access to source control?
* Do I need build dependencies that are not installable/available on the deployment target?
* You just don't _want_ to setup your deployment server to access source control.

Once you've take the decision, the switch is easy.
