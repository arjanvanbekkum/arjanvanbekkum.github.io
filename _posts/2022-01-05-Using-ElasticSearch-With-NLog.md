---
layout: post
title: "Using Elastic Search with NLog"
date: 2022-01-05
summary: "Setting up centralized logging using Elastic Search with NLog in .NET"
minute: 5
tags: [dotnet]
---

I recently worked on a project that uses `NLog` to log all application logging to a file. That in itself is fine, but if you have multiple instances running on different servers, it might be hard to get the log files. To read the log file, you need access to the server (or at least some need access). Access to a server is usually restricted to a few, so getting the files could be challenging. 

## NLog Setup

When using NLog, you need to configure a `Target`; that target needs to be wrapped. The `TargetWrapper` needs to be assigned to a `LoggingRule`. Your code should look something like the code below. For simplicity, I removed some of the properties, but you can set all the properties you need. 

Let's take a quick look at the `FileTarget` configuration, here we set the `Layout`, the `FileName`, and some other properties to create the files. The `SimpleLayout` we set here is just the `message` we send in when logging. 

```csharp
var wrappedTarget = new FileTarget("FileTarget")
{
    Layout = new SimpleLayout("${message}"),
    FileName = Path.Combine("C:\MyFolder", "InstallationFolder", "logfile.txt"),
    CreateDirs = true,
    Encoding = Encoding.UTF8
};
```
Next, we need to set the `TargetWrapper`; this wrapper the `FileTarget` and makes it async. NLog will use a different thread to write to files and not the logging thread. The `AsyncTargetWrapperOverflowAction` is set to `Block` to prevent buffer overflow when writing log files. The other options might cause memory issues, so be careful there. 

```csharp
var target = new AsyncTargetWrapper("FileTargetAsync", wrappedTarget)
{
    OverflowAction = AsyncTargetWrapperOverflowAction.Block
};
```

The last step is to add the `AsyncTargetWrapper` to a `LoggingRule`. We can set the `LogLevel` and the `Pattern`.

```csharp
var loggingRule = new LoggingRule("FileLogger")
{
    LoggerNamePattern = "*",
    Targets =
    {
        loggingConfiguration.FindTargetByName("FileTargetAsync")
    }
};
loggingRule.EnableLoggingForLevels(NLog.LogLevel.Trace, NLog.LogLevel.Fatal);
loggingConfiguration.LoggingRules.Add(loggingRule);
```

So we set up the `FileTarget` and can log to a file. If you want to change to centralized logging easily, you can extend the `NLog` targets; we decided to use `ElasticSearch`.

## ElasticSearch

To extend the NLog configuration, there is a `NuGet` package available called `NLog.Targets.ElasticSearch`; the recent version when writing this blog is 7.6.0. 

When you install the package, you can easily add a new target. You can almost repeat what you did when adding the `FileTarget`. Create a new `ElasticSearchTarget` and assign the properties you need for logging to `ElasticSearch`

```csharp
var elasticSearchTarget = new ElasticSearchTarget()
{
    Layout = new SimpleLayout("${message}"),
    Name = "ElasticTarget",
    Uri = "https://your_elastic_search_url",
    Index = new SimpleLayout("MyIndex"),
}
```

You will probably need something to authenticate when sending the logging to Elastic. We use an `ApiKey` and an `ApiKeyId` where the `ApiKey` is the secret value, and the `ApiKeyId` is clientId.

Next, we need to wrap this `ElasticSearchTarget`into an `AsyncTargetWrapper`. 

```csharp
var target = new AsyncTargetWrapper("ElasticTargetAsync", elasticSearchTarget)
{
    OverflowAction = AsyncTargetWrapperOverflowAction.Block,
};
```

And the last step is again to add this wrapper to a logging rule. You do not have to add two different LogRules. You can connect both `AsyncTargetWrappers` into the same `loggingRule` just like we do here. 

```csharp
var loggingRuleElastic = new LoggingRule("Logger")
{
    LoggerNamePattern = "*",
    Targets =
        {
            loggingConfiguration.FindTargetByName("ElasticTargetAsync"),
            loggingConfiguration.FindTargetByName("FileTargetAsync")
        }
};
loggingRuleElastic.EnableLoggingForLevels(NLog.LogLevel.Debug, NLog.LogLevel.Fatal);
loggingConfiguration.LoggingRules.Add(loggingRuleElastic);
}
```

This is enough to send the logging to `ElasticSearch`, but you want to add some more information. You can add a so-called `LayoutRenderer`.

## LayoutRenderer

In a `LayoutRenderer`, you can add information you want to add to the logging, for example, version information or assembly info. You are not limited to these two; you can add everything you need. We will create a simple layout to add the assembly name and assembly version.

First, create a class and inherit this class from the `LayoutRender`. The name you want to add to this layout goes into the `LayoutRenderer` as a parameter; in this case, `assembly` as Next, you need to override the `Append()` method. You can extend the `LogEventInfo` by adding new properties in this method.

```csharp
[LayoutRenderer("assembly")]
public class AssemblyLayoutRender : LayoutRenderer
{
    protected override void Append(StringBuilder builder, LogEventInfo logEvent)
    {
        var assembly = Assembly.GetExecutingAssembly();

        if (assembly.Location == null)
        {
            return;
        }

        string version = FileVersionInfo.GetVersionInfo(assembly.Location).ProductVersion;
        string name = assembly.GetName().Name;

        logEvent.Properties.Add("assemblyVersion", version);
        logEvent.Properties.Add("assemblyName", name);
    }
}
```

After creating the new `LayoutRenderer,` we need to register this layout. If you do not register this layout, you cannot use it when logging messages. You need to register the payout before you want to use it. Registration of a new layout is just one line. 

```csharp
ConfigurationItemFactory.Default.LayoutRenderers.RegisterDefinition("assembly", typeof(AssemblyLayoutRender));
```

The only thing we need to do now adds the new layout to `ElasticSearchTarget`. This target has a list of `Fields`; you can use these fields to extend your logging message. We created a new layout, but plenty of out-of-the-box layouts are available. In this case, we will also add a message template. Using structured logging will add your logging template to the logging message. We will add both our assembly layout and the message template.

```csharp
Fields = new List<Field>
{
    new Field()
    {
        Name = "assembly",
        Layout = new SimpleLayout("${assembly}")
    },
    new Field()
    {
        Name = "messagetemplate",
        Layout = new SimpleLayout("${message:raw=true}")
    }
}
```
This should work in most cases, but on Windows 11, you might run into some issues.
                
## Windows 11 and Tls 1.3 issue

If you use Windows 11, you might get this exception message

```csharp
The SSL connection could not be established; see inner exception.
---> System.Security.Authentication.AuthenticationException: Authentication failed because the remote party sent a TLS alert: 'HandshakeFailure'.
```

The issue is caused by Windows 11 in combination with tls 1.3. A workaround is available in the latest version of the `ElasticSearch.NET` package. Unfortunately, it would help to change something to the `ElasticSearchTarget`. 

First, we cloned the ElasticSearchTarget repository from <a href ="https://github.com/markmcdowell/NLog.Targets.ElasticSearch">GitHub</a>. Next, instead of using the package, implement the code inside the cloned repository in your code. 

Now we have to update the `ElasticSearch.NET` package to the latest version; in this case, version `7.15.2` In this version, there is a fix available to work around the tls1.3 issue in combination with Windows 11. 

You must now change the code inside the cloned repository to activate this workaround. Locate the `ElasticSearchTarget.cs` file inside of the repository. Inside of this class, an `ElasticLowLevelClient` is created. You need to add this line above where the `ElasticLowLevelClient` is created.

```csharp
config.UnsafeDisableTls13();

_client = new ElasticLowLevelClient(config);
``` 

Now you should use the `ElasticSearchTarget` on your Windows 11 machine.
