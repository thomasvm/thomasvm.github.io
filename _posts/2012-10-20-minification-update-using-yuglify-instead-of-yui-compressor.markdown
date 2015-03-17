---
layout: post
title: "Minification update: using yuglify"
date: 2012-10-20 12:40
comments: true
categories: ["deployment", "YUI", "yuglify", "unfold" ]
---
In the [previous post](http://thomasvm.github.com/blog/2012/10/11/unfold-task-hooks/) we 
described how to use YUI Compressor to minify your CSS and JavaScript when deploying 
your application using [Unfold](https://github.com/thomasvm/unfold). The next day, 
[Andrew Wooldridge](https://twitter.com/triptych) pointed me to a blog post called 
[State of YUI Compressor](http://www.yuiblog.com/blog/2012/10/16/state-of-yui-compressor/).

> YUI Compressor will be going through a deprecation process as the YUI team moves 
> to using a custom-wrapped UglifyJS for JavaScript compression and a more fully 
> updated and maintained version of CSSMin for CSS compression. This will allow 
> YUI to move beyond the limitations of the current YUI Compressor as well as take 
> advantage of the current state of the art in compression. 
> 
> The new compression utilities are wrapped into a single node package called yuglify 
> and new issues can be filed against this project on GitHub. 

So, let's convert the `minify` task we wrote to use [yuglify](https://github.com/yui/yuglify) 
instead of the soon to be deprecated YUI Compressor. 

## The steps

1. We added the `yui-compressor.jar` to a tools folder in our scm. We can now remove that, we 
   will depend on yuglify being available on the `PATH`
2. yuglify is a node.js package, so to install it on our system we need to install
   [node.js](http://nodejs.org/). Note that it needs to be available on the target machine of 
   our deployment because that's where the minification will happen. (unless you're using the
   local build option, more on that later)
3. Open a command prompt and install yuglify through npm

    npm install -g yuglify
    
4. Now we can alter the task to use yuglify:

``` posh
task minify {
    # Invoke-Script, so this gets executed on the target
    Invoke-Script {
        # Define a function, that can minify a folder that's under web
        function Minify($folder, $type) {
            Write-Host "Minifying folder $folder" -Fore Green

            If(-not $type) {
                $type = $folder
            }
    
            # get all files inside the web folder and minify them
            Foreach($file in (Get-ChildItem "$($config.releasepath)\web\$folder")) {
                Write-Host "Minifying $file"
                # Wrap inside psake exec function so the build 
                # fails if this operation fails
                Exec {
                    # get the file's content and pass it to yuglify
                    Get-Content $file.FullName `
                    | yuglify --terminal --type $type `
                              --output $file.FullName
                }
            }
        }

        # Now call the function for each folder we want to minimize
        Minify "js"
        Minify "css"
    }
}

Set-AfterTask release minify
```

## That's it
We're now ready for the future, using yuglify instead of the deprecated YUI Compressor.
