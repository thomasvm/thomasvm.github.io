---
layout: post
title: "Launch FluentMigrator from within Visual Studio"
date: 2012-12-10 22:16
comments: true
categories: [ "database", "migrations", ".net" ]
---

When it comes to creating database migrations for your .net application, then 
[FluentMigrator][fluentmigrator] is simply one of the best options around. It has an API 
that's very easy to discover and has support for lots of different database systems 
(SQL Server, sqlite, postgres, ...). As a result, re-creating a development database or
migrating your production database is only a matter of executing the migrations. Easy.

However, when you're creating new migrations, it often happens that you

* run your migrations
* check the result, see that you want change some minor things
* rollback the migration
* change your code
* run the migration again
* rollback the migration, e.g. to test your down operation
* ...

As you can see, you want to run the up and down parts of your migration several times. 
This article will describe **how to define an external tool in Visual Studio to quickly
migrate and rollback the migrations you are working on**. Thanks go out to 
[Frederick D'hont][frederickdhont] for giving me the initial idea.

## Setting up your migrations project

First we are going to modify our project in such a way that setting up our external
tool will become easier. This involves *4 steps*:

1. First, we install the [Fluent Migrator Tools][fluentmigrator-tools] nuget package
   inside our Migrations projeect. This will download and extract a `Migrate.exe` 
   executable. This executable can be useed to execute your migrations from the command
   line, exactly what we need

2. Add the `Migrate.exe` and the `FluentMigrator.Runner.dll` assemblies to your project by right-clicking on the project and selecting
   _Add an existing item_. Navigate to the `packages` folder of your solution and pick
   the `Migrate.exe` and the  `FluentMigrator.Runner.dll` that are somewhere inside the FluentMigrator.Tools folder. Make sure
   you select _Add as link_ and pick the `AnyCPU` version. 
   
   {% img /images/blog/fluentmigrator-add-migrate-fixed.png %}
   
   Alternatively, you can manually download or compile the exe and dll from source and add 
   them to your source control.

   _Note:_ In some cases, you might have already added the `FluentMigrator.Runner.dll` assembly 
   as an actual assembly reference, because it contains some functionality you need. In that 
   case, you don't need to add it again.

3. Open up the properties for the `Migrate.exe` and `FluentMigrator.Runner.dll` items in your Visual Studio project, and
   set it to _Copy to output directory_: _Copy always_

   {% img /images/blog/fluentmigrator-migrate-properties.png %}

5. Add the connection string for your development machine to the `app.config` of the
   migrations assembly. As the _name_ simply use your machine name. By doing so,
   FluentMigrator can automatically select the connection string for your machine. Also
   it allows your team members to put their connection string in the `app.config`
   file.

Once these steps are performed, the output directory of your project will now contain
the migrations assembly as well as an executable that can run them.

## Defining the external tool

Our project is now in the perfect shape to make an external tool consume it. Open
up the external tools manager and create a new tool. Give it the following values.

* **Title**: FluentMigrate
* **Command**: `$(BinDir)\Migrate.exe`
* **Arguments**: `--provider sqlserver2008 --a $(TargetName)$(TargetExt)`
* **Initial Directory**: `$(BinDir)`
* Use output window: click checkbox

   {% img /images/blog/fluentmigrator-custom-tool.png %}

You will probably receive a warning that the "Command is not a valid executable". This
is caused by the `$(BinDir)` variable that is used in the Command property, but don't
worry, this is not a problem. Simply click "No", indicating that you want to apply your
changes.

That's it, if you now build your migrations project and launch the external tool it will
run the current state. Additionally, you can define a second External Tool that executes 
a rollback. The settings for such a rollback tool are the same as above, except that 
FluentMigrator needs 1 extra parameer in the _arguments_: `--task=rollback`

## Runnig the external tool, and re-using it

Now that we have setup our tool, it can be run from any node in the solution explorer
that is related to the migrations project. Also if your open file is a migration, you 
can launch the external tool.

It's also important to note that **as long as migration projects are set up in the way that 
we described above, the external tool can be re-used**. So if you have multiple 
solutions that each have their migrations project, you can use the external tool.
tool can be re-used. 

### Bonus points: keyboard shortcuts

To further increase your productivity, you can assign keyboard shortcuts to each of them. 
Peresonally I use:

* `Ctrl-K, M`: for executing the pending migrations.
* `Ctrl-K, R`: for rolling back a migration.

There you go, running database migrations on your development database is as easy a issuing
a keyboard shortcut or navigating to a menu item. 

[fluentmigrator]: https://github.com/schambers/fluentmigrator
[fluentmigrator-tools]: http://nuget.org/packages/fluentmigrator.tools 
[frederickdhont]: https://twitter.com/frefre
