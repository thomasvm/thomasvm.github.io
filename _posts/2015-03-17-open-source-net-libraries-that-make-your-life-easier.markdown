---
layout: post
title: "Open Source .net libraries that make your life easier"
date: 2015-03-12 20:21
comments: true
categories: [ ".net", "oss", "open source" ]
---

Even though people sometimes claim that open source .net is not in a healthy state and that
the community too often relies on what Microsoft is providing, there _do_ exist some awesome open
source libraries out there. Over the course of the last couple of years, we've started several - 
often small scale - new projects at work and I've come to rely on some libraries that solve some 
specific problems, and they solve them well. This post is not about packages that are in 
[NuGet's top 100][1] list, but about lesser known libraries that deserve some extra attention. 

Are there some things they have in common? Yes! 

1. Most of them have excellent **documentation**, time 
   and time again this turns out to be extremely important for any OSS project, especially go get
   starters on board. 
2. These libraries are [not frameworks][12] in the sense that they don't force you into any 
   application structure or require you to inherit from class X or Y. They are just nice libraries
   that you can _plug_ into your projects and they do what they promise.

So let's get into it... 

## Hangfire

[Hangfire][2] is a library that allows you to quickly and beautifully add background processing
to your ASP.net application. It's extremely simple to setup and scheduling background work is as
straightforward as it can be. 

``` csharp
BackgroundJob.Enqueue(() => Console.WriteLine("Simple!")); 
```

There are several things I love about Hangfire: 

* The built-in dashboard is beautiful, clear and gives a good view of what is happening and -
  maybe more important - what is going wrong. 
 
  {% img http://hangfire.io/img/dashboard.png %} 

* There are no requirements on the method that is to be executed in the background. No class to
  inherit from, no attributes have to added to it, it just needs to be a public method. This 
  allows you to structure your background jobs as you see fit. 

* While It Just Works when you add it to an ASP.net application, you're not restricted to it
  You can schedule and process the jobs basically anywhere: Windows Services, Console Applications, 
  Azure Web works, .... If you include this in your application, you know that you will be able 
  to take your code and split it into different components if needed. And Hangfire will still 
  be able to do the processing. 

* It was once featured on Scott Hanselman's [blog][3]. It probably helped a lot to highlight the
  existence of Hangfire, but [some people][4] also quickly noted its shortcomings. It had a dependancy
  on `System.Web` even though it was architected as an OWIN component, it depended on Common.Logging
  and used SQL Server by default. All of these were true, but it should be noted that Hangfire's author
  Sergey Odinokov has been quick to address the issues. By now, Hangfire is already in a much better
  shape, so I have the impression that Sergey knows very well what he is doing.

## Postal

Shortly after the Razor View Engine came out for ASP.net, [Andrew Davey][5] had the excellent idea to
start using it to do Email templating: [Postal][6] was born. The idea is simple, your `Views` folder gets
an extra subdirectory called `Emails` where you put in all your Razor Email templates and everything Razor
has to offer comes for free: Layouts, strongly-typed Models if you want, Url-helpers, partials, etc. Sending 
an e-mail is as simple as

``` csharp
dynamic email = new Email("Example"); // Uses Views\Emails\Example.cshtml
email.To = "webninja@example.com";
email.FunnyLink = DB.GetRandomLolcatLink();
email.Send();
```

You basically create a `Email` object and treat it as a `dynamic`, you chuck some properties onto it, which you can
then inside your Email template.

```
To: @ViewBag.To
From: lolcats@website.com
Subject: Important Message

Hello,
You wanted important web links right?
Check out this: @ViewBag.FunnyLink

<3
```

Of course you can use strongly-typed Models if you want to. 

It respects `Web.config`'s smtp settings and hence works perfectly with services like [Mandrill][7] or [Postmark][8], has
some nice features like image inlining, and it is built to integrate perfectly any MVC version. Good stuff!

## Formo

Whether you're in a console app, a web application or a windows service, reading from your `app.config`'s and `web.config`'s
application settings is never a fun thing to do, especially if you want to read `DateTime` or `int` values. 

[Formo][9] solves this problem very elegantly by simply exposing the settings as a `dynamic` object. This makes the code to
read the settings much easier on the eye.

``` csharp
dynamic config = new Formo.Configuration();

// This simply reads the string
string apiEndPoint = config.ApiEndPoint;

// This converts the value in to an int, no 
// need to do any conversion in your code
int timeout = config.ApiTimeout<int>();

// This uses a default value of 5 if the
// configuration files contains no values
int retries = config.ApiRetries<int>(5);

```

If you don't like `dynamic`, you can even go a step further and bind the application settings to a simple POCO class.

``` csharp
public class Settings
{
    public string ApiEndpoint { get; set; }

    public int ApiTimeout { get; set; }
}

dynamic configuration = new FormoConfiguration();
Settings settings = configuration.Bind<Settings>();
```

Even though Formo's scope is very limited and even though it seems that reading configuration files will [change drastically][10]
with aspnet5, I find it a very useful library which for me has reduced the friction of reading config files perfectly. 

## CsvHelper

When dealing with data from legacy systems, then CSV files are still commonly used as _the_ way to export data. In the past 
I have used several different ways of dealing with these files: copy/paste from codeproject (yeah, I know), hand written 
CSV reader for specific format, etc. 

Until I bumped into [CsvHelper][11], that is. The feature that I love the most, is how you can define a `CsvClassMap` to populate
a certain POCO class. You can then use a `CsvReader` to convert each row into an instance of that POCO class. That way, you don't
have to think about all the nitty gritty details that can occur when you need to read CSV files. These class maps provide a lot
of configuration options and I have yet to encounter a situation that CsvHelper cannot handle. 

## TopShelf

Writing a Windows Service is a lot harder than it should be. The logic itself is the easy part, but writing and configuring the part
that creates the service on the system is just a mess. If you follow the Visual Studio Way of doing things, this mess involves

* Creating a `ServiceBase` class, this contains the business logic of th Service
* Adding a `ProjectInstaller` to the project. Here we go into drag-and-drop territory, with `.Designer.cs` files and toolboxes and
  magically generated `InitializeComponent` methods.
* This `ProjectInstaller` should contain an instance of a `ServiceProcessInstaller`. This tells the installer under which account
  the service should run.
* The `ProjectInstaller` also needs to contain one or more `ServiceInstaller` instances which tell the ...
* ...

You get the gist, it is overly complex and involves too much automatically generated code and magic strings. Compare this to the beauty
of a [TopShelf][13] service.

* First you create a console app and add TopShelf through NuGet
* Then, you create _a class_ that holds your service logic. The only important thing is that the class must have some kind of `Start`
  and `Stop` method
* and finally the main method of the `Program.cs` should look something like this

``` csharp
public static void Main()
{
    HostFactory.Run(c =>
    {
        c.Service<BookSyncService>(s =>
        {
           s.WhenStarted(tc => tc.Start());
           s.WhenStopped(tc => tc.Stop());
        });;
    
        c.UseNLog();
    
        c.SetServiceName("BookSync");
        c.SetDisplayName("Book Sync Service");
        c.SetDescription("Sync books");
    });
}
```

The result is a simple .exe file that you can use to install, uninstall, start and stop your Windows Service. To install it you
simply go to the command-line and execute

    .\yourservice.exe install

and the service will be configured in the Windows Service registry. Other operations are very similar and you can find 
more information in the [full reference][14].

## Conclusion

At first sight none of these libraries do something spectacular. They do however make your work lighter by simply taking care
of some very mundane problems like background processing, reading CSV files or creating a Windows Service. And this is where 
there value comes from: they allow you to focus on the real challenges of your application. And as cheesy as it may be, that's
a beautiful goal.

[1]: http://www.nuget.org/stats/packages 
[2]: http://hangfire.io/ 
[3]: http://www.hanselman.com/blog/HowToRunBackgroundTasksInASPNET.aspx 
[4]: https://twitter.com/randompunter/status/504510526345740288 
[5]: https://twitter.com/andrewdavey 
[6]: http://aboutcode.net/postal/ 
[7]: http://mandrill.com/
[8]: https://postmarkapp.com/ 
[9]: https://github.com/ChrisMissal/Formo  
[10]: http://whereslou.com/2014/05/23/asp-net-vnext-moving-parts-iconfiguration/ 
[11]: http://joshclose.github.io/CsvHelper/
[12]: http://tomasp.net/blog/2015/library-frameworks/
[13]: http://topshelf-project.com/ 
[14]: http://docs.topshelf-project.com/en/latest/overview/commandline.html 
