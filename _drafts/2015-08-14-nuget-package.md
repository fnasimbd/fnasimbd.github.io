---
layout: post
title: "NuGet Primer"
categories: articles
date: 2015-08-14-T23:27:00-05
comments: true
---

# Package Management Very Briefly

Every large software project depends on several other components (usually called *dependencies*); those may be developed by other in-house teams or may come from external sources. Each time a dependency has updates the project must be tested for integration: to check if a bug is indeed fixed, if a new feature is really in effect, if the update introduces any side effects, etc.

For small projects with few dependency and only a handful of developer's working on it, manual management works well. As project gets larger and more complex, it's dependency also grows, so does the complexity of their management. Lack of structured package management may cause tremendous frustration and waste of valuable development hours. An automated package management is the only rescue in these situations.

I found the following benefits of an automated package management system:

- **Ease of distribution**. Automated package management systesm distribute packages through a common package repository; this way publishers know where to publish and consumers know where to look for updates.
- **Improved portability**. Solutions can be distributed without worrying about dependencies.
- **Preservation of integratiy**. As long as consumers pick up packages from the common repository---rather than some other random source---they know they have the right thing.
- **Enhanced communication**. Publishers don't need to inform consumers: for updates just check the repository. Also consumers risk less missing an update by checking for updates routinely.
- **Leverages further automation**. Automated package management enables adhering to further automations like continuous integration.

# What is NuGet

Most open software platforms took package management seriously. Their dependencey management tools have been around for about a decade: like Java world has *Maven* since 2002.

.NET world, however, joined the race much later. Their solution to the dependency problem is *NuGet*: it is a package manager for .NET packages, originally developed by *Outercurve Foundation* and now owned by *.NET Foundation*(?). Since its introduction in 2010 it went popular rapidly: at present [Nuget.org](http://www.nuget.org), the official .NET Foundation public NuGet package feed, hosts over 40,000 packages. These days NuGet is ubiquitous and it is the de facto standard in .NET world.

# How NuGet Works

I will give brief overview of NuGet's working principle---the amount sufficient to make it work in simple situations. NuGet is highly configurable and supports many advanced features that gives you lot more flexibility in handling complex environments. I plan to cover those details in a future article.

#### NuGet Package File

NuGet packages a project's build artifacts into a `.nupkg` archive for distribution. It includes project output (class library or executable), auxiliary files (configurations), other dependencies, etc. Open a .nupkg file with WinRAR and explore what's inside for better understanding.

NuGet maintains versioning to facilitate tracking.

#### NuGet Project Manifest File

Each package published by NuGet is coupled with a manifest file: it is a configuration file with XML structre. In this file user specifies what to copy and how etc.

#### NuGet Package Schema

NuGet organizes package contents in the following file schema:

    package
    |--lib
    |--content

For short, the project output (class library or executable) is copied to `lib` and all contents inside the `content` folder is copied to the target project's root path.

#### NuGet Tools

It consists of the following two key components:

1. NuGet client.
2. NuGet feed.

#### NuGet Commands

The following commands are sufficent for basic NuGet operations. For the exhaustive list see [official command line reference](https://docs.nuget.org/consume/command-line-reference).

- **Pack.** Creates a new package.
- **Install.** Installs a package.
- **Update.** Updates an already installed package.
- **Restore.** Downloads a missing package from repo.

# Setting Up NuGet

NuGet client comes in two formats: as command line utility, called nuget.exe, and as Visual Studio extension. Installing both are straightforward; I recommend having both.

- Download the NuGet command line utility from its [home page](https://nuget.org/nuget.exe) (it is only a single .exe file), copy it to your desired location, and add the path to environment variable.
- NuGet Package Manager extension is pre-installed in Visual Studio 2012 or later. In earlier versions, install it through Extensions Manager.

Setup a network path with write permission to package authors and read permission to package users (for this article I will use `\\farhan-lenovo\nuget-feed`); this will be your NuGet feed for now.

# Authoring NuGet packages

1. Choose a project to publish. Create the project manifest `project-name.nuspec` file with this command:

        nuget spec project-name

2. Fill in the mandatory fields in the .nuspec file. Some fields will be picked up from project's assemby resource file.

3. Update assembly version. NuGet uses [semantic versioning](http://semver.org). For simplicity, just keep this in mind: "NuGet updates packages only if minor version or patch verstion is updated." See [NuGet Versioning Reference](http://docs.nuget.org/Create/Versioning) for details.

4. Build the project.

5. Build package with the followig command:

        pack project-name

6. Move package to feed location.

        move project-name.nupkg repo-location

Publishing done!

# Using NuGet Packages

#### Configuring Package Source

First step to using NuGet packages is to configure your package sources. By default nuget.org feed is available in Visual Studio. To add your own feed, do the following steps.

1. Open NuGet Package Manager configuration dialog box in Visual Studio (Goto *Tools* > *NuGet Package Manager* > *Package Manager Settings* > *Package Sources*.)
2. Click the **+** sign near the top-right corner.
3. Put a name for your source in the *Name* field.
4. Put the shared folder path in *Source* field, click OK.

You are done!

![config-src]({{site.url}}/images/config-src.png)

#### Installation

Open Package Manager Console, choose your package source from dropdown, and run

    Install-Package package-name

From Visual Studio, right click on solution or project and click *Manage NuGet Packages...*, choose your feed, find the package you want, and click install.

#### Updating

Open Package Manager Console, choose package source, and run

    Update-Package package-name

Find the package in *Installed Packages* list and click update.

#### Uninstalling

Open package manager console, and run

    Uninstall-Package package-name

# Routine Practices

For best outcome, form your team's consensus on the following practices---or in their modification or augmentation as you find convenient.

1. Create the manifest file for a project the moment it is created.
2. Follow stringent versioning for assemblies.
3. Write a publish script for each project that will build and publish the latest artifacts to NuGet repository. You may also consider hooking the publish script to your version control system.
4. Write an update script for each project that will update all of its dependencies.

# External Links

1. [NuGet Home](https://github.com/nuget/home)
2. [MyGet](http://www.myget.org/). Personal and enterprise NuGet hosting.
3. [NuGet Package Explorer](https://npe.codeplex.com/). GUI tool for NuGet package viewing and management.

{% if page.comments %}
<div id="disqus_thread"></div>
<script type="text/javascript">
    /* * * CONFIGURATION VARIABLES * * */
    var disqus_shortname = 'fnasim';
    
    /* * * DON'T EDIT BELOW THIS LINE * * */
    (function() {
        var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
        dsq.src = '//' + disqus_shortname + '.disqus.com/embed.js';
        (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
    })();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript" rel="nofollow">comments powered by Disqus.</a></noscript>

{% endif %}

