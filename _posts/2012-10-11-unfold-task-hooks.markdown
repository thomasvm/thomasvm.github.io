---
layout: post
title: "Unfold task hooks: minification example"
date: 2012-10-11 15:49
comments: true
published: true
categories: 
---
Up until now we've mostly discussed the [inner workings](thomasvm.github.com/blog/2012/10/02/introducing-unfold/) of 
[Unfold](https://github.com/thomasvm/unfold) and which [default tasks](thomasvm.github.com/blog/2012/10/10/the-unfold-tasks/) 
come out-of-the-box. This works for standard applications, but most of the time you'll want more. 

For the biggest part you can simply rely on Unfold itself: it will get your application from source control, build it, setup
IIS, etc. But let's say your site has a bunch of CSS and JavaScript files that you want to minimize when you're deploying to 
production. I know there are libraries [out](https://github.com/jetheredge/SquishIt) [there](http://getcassette.net/) 
that do the work for you, but in this case you just want to minimize your files beforehand using YUI Compressor.

## A word on hooks

Unfold comes with support for _task hooks_. This concept allows you to _attach_ tasks you create onto the default task flow. In
our previous post we explained [the several tasks](http://getcassette.net/) that compose the deployment operation: setup, 
updatecode, build, release, setupapppool, ...

If you want to extend the operations that happen during a given task, then what you have to do is pretty simple

1. define a psake task that contains these extra operations
2. indicate that you want to execute the task before or after another task. This is done using the `Set-BeforeTask` and
   `Set-AfterTask` functions.

## Back to the minification 

Now that you know how to extend the deployment, you can start adding minification to it. 

1. We'll create a psake task called `minify` that will perform the minification
2. We're going to attach it to the `release` task. As described in the previous post the `release` task moves your compiled
   application into a separate folder that we later use for pointing IIS at. 
   Minifying the js and css that's inside is a natural extension to it.

So let's start. Add this to your `deploy.ps1` file:

``` posh
# 1. create the task
task minify {
    # minification goes here
}

# 2. attach it execute after the release task
Set-AfterTask release minify
```

There is also a `Set-BeforeTask` which performs the opposite operation: it mark a task to be executed _before_ another task 
kicks off.

## Performing the minification

**UPDATE 2012/10/20** 
YUI Compressor will be 
[deprecated](http://www.yuiblog.com/blog/2012/10/16/state-of-yui-compressor/), the following blog post describes [how to use yuglify](http://thomasvm.github.com/blog/2012/10/20/minification-update-using-yuglify-instead-of-yui-compressor/) instead of YUI Compressor.

To perform the minification we're using [YUI Compressor](http://developer.yahoo.com/yui/compressor/). To use it, simply download
the latest version from their GitHub [downloads](https://github.com/yui/yuicompressor/downloads) page for example. Extract it
and add the final `jar` file somewhere to your source repository for example under `tools\yui\yuicompressor-2.4.2.jar`. 
By doing so, we're sure that it is available when we're performing the minification because we always release based on what's in
source control. 

Now that the tooling is in place, we can write the minify task. Here goes:

``` posh
task minify {
    # Invoke-Script, so this gets executed on the target
    Invoke-Script {
        # Define a function, that can minify a folder that's under web
        function Minify($folder) {
            Write-Host "Minifying folder $folder" -Fore Green
    
            # get all files inside the web folder and minify them
            Foreach($file in (Get-ChildItem "$($config.releasepath)\web\$folder")) {
                Write-Host "Minifying $file"
                # Wrap inside psake exec function so the build 
                # fails if this operation fails
                Exec {
                    # the code folder will contain the current code checkout, and 
                    # that's where the jar will be. We use it to compress the
                    # found file
                    java -jar ".\code\tools\yui\yuicompressor-2.4.2.jar" `
                         --charset utf8 `
                         -o "$($file.FullName)" `
                         "$($file.FullName)"
                }
            }
        }

        # Now call the function for each folder we want to minimize
        Minify "js"
        Minify "css"
    }
```

## Only applying it on production
As the task is defined now, it will always perform the minification, but as we said before, we only want this on _production_.
That way we can still easily debug problems on our dev environment for example. What we're going to do is simple: we'll 
change the configuration. Move to the configuration part of the `deploy.ps1` file and add the following

``` posh
# Normally we don't minimize
Set-Config minimize $false

# On production however, we enable it
Set-Environment production {
    ... other configuration settings ...

    Set-Config minimize $true
}
```

Now that these settings are available we can extend the `minify` task just a little bit more. If the `$config.minimize` setting
is set to `$false`, we simply abort the task.

``` posh
task minify {
    If(-not $config.minimize) {
        return
    }

    Invoke-Script {
     ... rest of task
```

And there you go, the minification is only applied on production.

## Conclusion
After this task has executed, all css and javascript files within your web folder will be minimized. Just what we need. By 
hooking the `minify` task up with the `release` task we can extend the standard flow and tailor it to perform completely
custom deployment operations. We've also explained how you can define custom configuration options and use them inside your
tasks.
