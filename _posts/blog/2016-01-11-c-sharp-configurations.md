---
layout: post
title: "C# Application Configuration Basics"
modified:
categories: blog
excerpt: "Various methods to read-write .NET application configurations."
tags: [".net", "C#", "configuration", "appconfig"]
image:
feature:
share: true
date: 2016-01-11T02:05:00-04:00
comments: true
---

{% if page.description %}
    <meta name="description" content="{{page.description}}" />
{% endif %}

Configurations are integral to applications that has to deal with user preferences or configurable components. C# .NET offers two standard means of handling configurations that relieves developers from much of the hassle of implementing configuration handling themselves. Though the configuration handling features are extendable, the defaults are adequate for the basic needs.

# The *ConfigurationManager.AppSettings* Property

The first---and very straightforward---way to store and retrieve your configurations is to add a section called *appSettings* in your project's *App.config* file. By default C# projects don't have an App.config file; to add one, from Visual Studio add a new *Application Configuration File* item to the project. 
Here follows a sample App.config file with an appSettings sction with only one key called *Name*; you may add as many keys as you want in appSettings.

{% highlight xml linenos %}
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <appSettings>
    <add key="Name" value="John Doe" />
  </appSettings>
</configuration>
{% endhighlight %}

On build the App.config file renames to *AssemblyName.OutputType.config* and copies to the project output directory. On application startup, configurations in the appSettings section are read and cached as `NameValueCollection` in `Configuration.AppSettings` property. The following statement accesses the key *Name* from `AppSettings` property:

```csharp
var name = ConfigurationManager.AppSettings["Name"];
```

Note here that accessing a key that doesn't exist in config file returns null; no exception is thrown.

#### Storing New Configurations During Run Time

There are situations where you don't know some configuratios beforehand; you only get to know them on run time. Following code snippet demonstrates how you add new setting to App.config file in run time.

```csharp
var configFile = ConfigurationManager.OpenExeConfiguration(ConfigurationUserLevel.None);
configFile.AppSettings.Settings.Add("NewName", "Daniel Doe");
configFile.Save(ConfigurationSaveMode.Modified);
```

The changes take effect immediately in the assembly's config file once the `Save()` method is called.

#### Reloading Settings From File in Run Time

After making many changes to the configuration values in run time, you may sometimes need to invalidate the changes and re-read the values from configuration file. Just call the `RefreshSection()` method with parameter `appSettings` as the following statement does.

```csharp
ConfigurationManager.RefreshSection("appSettings");
```

#### Moving appSettings to a Separate File

Configuration file of applications that use too many configurable components, e.g. dependency injection framework, logger, etc., are often long. If your appSettings section is long, a good option to keep the App.config file cleaner is to put the appSettings in a separate file and putting that file's reference in App.config. Say your appSettings section is stored in file *appSettings.config*, then you may access the settings in that file in the following way:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <appSettings file="appSettings.config" />
</configuration>
```

The appSettings.config file must have `appSettings` as its root element as following:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<appSettings>
  <add key="Name" value="Mike Doe"/>
  <add key="Port" value="8080"/>
</appSettings>
```

It should be obvious from the App.config file examples that appSettings section doesn't store type information of values (not *strongly typed* in programming jargon), all values are stored and returned as strings: in case you want to use non string values, you have to do your own type conversion in program. This is, in fact, the biggest limitation of appSettings section.

# The *Settings.settings* Section

The other way of handling configurations is by `Settings` class: extension of `System.Configuration.ApplicationSettingsBase`. Here you add one or more `Settings` class to your project, add configurations as strongly typed static properties of that class, and provide default values for configurations in project App.config file. This approach addresses the strong typing limitation of appSettings and it is the recommended way these days.

Visual Studio has a settings class designer that does most of the settings operations. By default you don't have any settings class in your project; to add one, open project properties, go to Settings tab; the settings designer opens, add as many settings you want, select type for them, provide default values, and save. As result of your changes, a new element called Settings.settings is added in your project's Properties node and your App.config file changes to something like this:

{% highlight xml linenos %}
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <configSections>
    <sectionGroup name="applicationSettings" type="System.Configuration.ApplicationSettingsGroup, System, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089" >
      <section name="Experiments.Properties.Settings" type="System.Configuration.ClientSettingsSection, System, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089" requirePermission="false" />
    </sectionGroup>
  </configSections>
  <applicationSettings>
    <Experiments.Properties.Settings>
      <setting name="Name" serializeAs="String">
        <value>John Doe</value>
      </setting>
    </Experiments.Properties.Settings>
  </applicationSettings>
</configuration>
{% endhighlight %}

To access the setting `Name` do the following:

```csharp
var name = Properties.Settings.Default.Name;
```

Additionally you may also want to have a look at the `Settings.Designer.cs` file to get a feel how this works.

#### The *Application* and *User* Scopes

In settings designer you should have noticed that the settings keys has two choices for scope: *Application* and *User*. They differ in the type of property they result in Settings class.

- The *Application* scoped settings are get only properties; stores system level information that are not likely to change at run time. Default values for application scoped configurations reside in *applicationSettings* section in App.config file.
- The *User* scoped settings are get and set properties; intended to store user preferences that are likely to change at run time. User scoped configuratios have their default values in *userSettings* section in App.config file.

#### Updating and Saving User Scoped Application Settings in Run Time

You can update and save the user scoped settings during run time and retrieve them next time the application starts. The following snippet shows how to do that for the setting *Name*.

```csharp
Properties.Settings.Default.Name = "Jack Doe";
Properties.Settings.Default.Save();
```

The first time you build your application, you have the user scoped settings---unless specified otherwise---in a section of your app.config file. As you make any changes to settings in run time, your updated user settings are saved in a *user.config* file in your system's *%AppData%* folder instead of the config file in application's working directory.

#### Reloading Settings in Run Time

During run time if you feel that, ignoring runtime changes, all settings must be re-read from config file use the following approach:

```csharp
Properties.Settings.Default.Reload();
```

# Further Reading

MSDN covers configuration and settings in great detail---so detailed that hard for first timers to find the right place to get started! I found the following links suitable for myself:

1. [ConfigurationManager.AppSettings Property](https://msdn.microsoft.com/en-us/library/system.configuration.configurationmanager.appsettings(v=vs.110).aspx).
1. [Application Settings](https://msdn.microsoft.com/en-us/library/a65txexh(v=vs.100).aspx).
1. [Settings Page, Project Designer](https://msdn.microsoft.com/en-us/library/cftf714c(v=vs.100).aspx).
1. [Application Settings Architecture](https://msdn.microsoft.com/en-us/library/8eyb2ct1(v=vs.100).aspx).

# Updates

__January 28, 2016:__ Renamed section "*The* applicationSettings *and* userSettings *Sections*" to "*The* Application *and* User *Scopes*" and updated content within the section.

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
