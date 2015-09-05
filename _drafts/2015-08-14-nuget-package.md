---
layout: post
title: "NuGet Primer"
categories: articles
date: 2015-08-14-T23:27:00-05
comments: true
---

# Package Management Very Briefly

Large software project's depend on several other components (called *dependencies*); those may be developed by other in-house teams or may come from external sources. Users of the dependency must make sure that they have the right dependency (package version, framework version, cpu architecture, etc.) Each time a dependency has updates the project must be tested for integration: to check if a bug is indeed fixed, if a new feature is really in effect, if the update introduces any side effects, etc.

For small projects with few dependency and only a handful of developer's working on it, manual management works well. As project gets larger and more complex, it's dependency also grows, so does the complexity of their management. Lack of structured package management may cause tremendous frustration and waste of valuable development hours. Moreover these effort deosn't contribute anything to the product's excellence. An automated package management is the only rescue in these situations.

I found the following benefits of an automated package management system:

- **Ease of distribution**. Automated package management systesm distribute packages through a common package repository; this way publishers know where to publish and consumers know where to look for updates.
- **Improved portability**. Solutions can be distributed without worrying about dependencies (whether the user platform is x86 or x64 etc.)
- **Preservation of integrity**. As long as consumers pick up packages from the common repository---rather than some other random source---they know they have the right thing.
- **Enhanced communication**. Publishers don't need to inform consumers: for updates just check the repository. Also consumers risk less missing an update by checking for updates routinely.
- **Leverages further automation**. Automated package management enables adhering to further automations like continuous integration.

# What is NuGet

Most open software platforms took package management seriously. Their dependencey management tools have been around for about a decade: like Java world has *Maven* since 2002.

.NET world, however, joined the race much later. Their solution to the dependency problem is *NuGet*: a package/library manager for .NET languages, originally developed by *Outercurve Foundation* and now owned by *.NET Foundation*(?). Since its introduction in 2010 it went popular rapidly: at present [Nuget.org](http://www.nuget.org), the official .NET Foundation public NuGet package feed, hosts over 40,000 packages. These days NuGet is ubiquitous and it is the de facto standard in .NET world.

# How NuGet Works

In this section, I will give brief overview of NuGet's working method and the concepts involved; this will be sufficient to make it work in very simple situations. In later sections I will cover installing NuGet, publishing a sample C# project as a NuGet package, and consuming that package.

NuGet is, however, highly configurable and supports many advanced features that lets you handle complex environments. I will leave references for further study wherever relevant.

#### NuGet Package File

NuGet bundles a project's build artifacts into a `.nupkg` archive for distribution. It includes project output (dll or exe depending on project output), auxiliary files (configurations, resources, etc.), other dependencies, etc. A new .nupkg file is created each time a new version of a project is published. Get a .nupkg file and open it with WinRAR and explore what's inside for better insight.

#### NuGet Package Schema

There are exceptions but usually NuGet organizes package contents in the following file schema:

    package
    |--lib/
    |--content/

For short, the contents inside `lib` are added to the target project's reference and contents of the `content` folder is copied to the target project's root path.

#### NuGet Project Manifest File

Each package published by NuGet is coupled with a manifest file (an XML configuration file.) In this file user specifies what to copy and how etc. Here is how a nuspec file looks like on its creation:

{% highlight xml linenos %}
<?xml version="1.0"?>
<package >
  <metadata>
    <id>$id$</id>
    <version>$version$</version>
    <title>$title$</title>
    <authors>$author$</authors>
    <owners>$author$</owners>
    <licenseUrl>http://LICENSE_URL_HERE_OR_DELETE_THIS_LINE</licenseUrl>
    <projectUrl>http://PROJECT_URL_HERE_OR_DELETE_THIS_LINE</projectUrl>
    <iconUrl>http://ICON_URL_HERE_OR_DELETE_THIS_LINE</iconUrl>
    <requireLicenseAcceptance>false</requireLicenseAcceptance>
    <description>$description$</description>
    <releaseNotes>Summary of changes made in this release of the package.</releaseNotes>
    <copyright>Copyright 2015</copyright>
    <tags>Tag1 Tag2</tags>
  </metadata>
</package>
{% endhighlight %}

Only *id*, *version*, *title*, and *author* in metadata section are mandatory.

For exahustive coverage see official [Nuspec reference](https://docs.nuget.org/Create/NuSpec-Reference).

#### NuGet Tools

A NuGet ecosystem consists of the following two key components:

1. **NuGet client.** Mainly a command line utility though other variants are avilable. It coordinates both package publication and consumption.
2. **NuGet feed.** Provider of NuGet packages; can be a network shared directory or an http server.

#### NuGet Commands

The following commands are sufficent for basic NuGet operations. For the exhaustive list see [official command line reference](https://docs.nuget.org/consume/command-line-reference).

- [**Spec.**](https://docs.nuget.org/Consume/Command-Line-Reference#spec-command) Creates .nuspec file.
- [**Pack.**](https://docs.nuget.org/Consume/Command-Line-Reference#pack-command) Creates a new package from project manifest file.
- [**Install.**](https://docs.nuget.org/Consume/Command-Line-Reference#install-command) Installs a package.
- [**Update.**](https://docs.nuget.org/Consume/Command-Line-Reference#update-command) Updates an already installed package.
- [**Restore.**](https://docs.nuget.org/Consume/Command-Line-Reference#restore-command) Downloads a missing package from repo.

# Setting Up NuGet

#### Install NuGet Client

NuGet client comes in two variants: as command line utility, called nuget.exe, and as Visual Studio extension. Installing both are straightforward and I recommend having both.

Download the NuGet command line utility from its [home page](https://nuget.org/nuget.exe) (it is only a single .exe file), copy it to your desired location, and add the path to environment variable.

NuGet Package Manager extension comes pre-installed in Visual Studio 2012 or later. In earlier versions, install it through *Tools > Extensions Manager*.

#### Setup NuGet Feed

Setup a network path with write permission to package authors and read permission to package users (for this article I will use `\\farhan-lenovo\nuget-feed`); this will be your NuGet feed for now.

# Authoring NuGet packages

Create a new C# project named *Miscellaneous* and set the output type to class library. Add whatever classes and functionalities you want.

Open Command Prompt or Powershell and switch to your project directory.

1. Create the project manifest file, Miscellaneous.nuspec, with this command:

        nuget spec Miscellaneous

2. Fill in the mandatory fields, *AssemblyDescription* and *AssemblyCompany*, in the project's AssemblyInfo.cs file. 

3. Update assembly version.

4. Build the project.

5. Build package with the following command:

        nuget pack Miscellaneous.csproj

    The package file, named Miscellaneous.version.nupkg, is created.

6. Move package to feed location.

        move Miscellaneous.version.nupkg \\farhan-lenovo\nuget-feed

Publishing first release of the project done! Assuming you don't change meta information of the package, to publish every next version, performing steps 3 to 6 will do.

# Using NuGet Packages

You can consume NuGet packages through both the command line utility and the Visual Studio extension. The Visual Studio extension is, however, easy to use and manages packages more conveniently.

#### Configuring Package Source

First step to using NuGet packages is to configure your package sources. By default only one, nuget.org, feed is available in the Visual Studio extension. To add your own feed, do the following steps.

1. Open NuGet Package Manager configuration dialog box in Visual Studio (Goto *Tools* > *NuGet Package Manager* > *Package Manager Settings* > *Package Sources*.)
2. Click the **+** sign near the top-right corner.
3. Put a name for your source in the *Name* field.
4. Put the shared folder path in *Source* field, click OK.

You are done!

![config-src]({{site.url}}/images/config-src.png)

#### Installation

Now let's install the package we published earlier, called Miscellaneous, in some other project. Open the target project in Visual Studio, go to *Tools > NuGet Package Manager > Manage NuGet Packages for Solution...* The package browser window will appear; in the Online tab find and select the source we configured earlier. All packages available in the current feed is listed here. The package Miscellaneous should be visible here. Click the install button beside it.

![install-package]({{site.url}}/images/install-package.png)

If your package installation is successful, you will see these changes in your project: a library called Miscellaneous.dll is added to the project references, a new folder called *packages* is created under your solution directory, and a file called *packages.config* is included in your project. The 'packages' folder contains the package you just installed and all further packages you install in any other project under the same solution; the 'packages.config' file contains information of the packages the project depends on in XML format.

#### Updating

Suppose you have made some changes to the package Miscellaneous and want to publish a new version. Make the changes you wish and follow steps 3 to 6 in section *Authoring NuGet Packages* to have the new version published.

To get the update, open package browser window just the way we did it during installation, select the 'Updates' tab, select 'Test Package Source.' The package 'Miscellaneous' should appear with an 'Update' button beside it. Click the 'Update' button to install the updated package.

![update-package]({{site.url}}/images/update-package.png)

#### Uninstalling

Open package browser window, select 'Installed Packages' tab. The package 'Miscellaneous' should appear here with an 'Uninstall' button beside it. Click that Uninstall button. Package is uninstalled; the reference to 'Miscellaneous.dll' as well as the folder 'Miscellaneous' in packages folder under solution directory disappears if uninstallation is successful.

![uninstall-package]({{site.url}}/images/uninstall-package.png)

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

