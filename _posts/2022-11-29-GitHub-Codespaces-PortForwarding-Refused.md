---
layout: post
title: "GitHub Codespaces port forwarding connection refused "
date: 2022-11-29
summary: "When you start a .NET 6 web API project in GitHub Codespaces, the port forwarding does not work. The connection is refused by localhost."
minute: 5
tags: [dotnet, github]
---

GitHub recently announced that Codespaces is available for everyone. If you need to know what Codespaces is, check it [out here](https://github.com/features/codespaces). It is Visual Studio Code running on a VM in Azure available from your web browser.

I wanted to play around with Codespaces, so I created a new .NET web API project. As I have always been a Visual Studio user for .NET development, using Codespaces for .NET was a unique experience. 

## .NET webapi project
Let's first create a new .NET project. To do this, you can run the following command
```CLI
 dotnet new webapi --language "C#"
```
This will create a new .NET web API project with essential items like a controller and support for swagger.

You can run this project by executing the following command. 

```CLI
 dotnet run
```
This will start the project in Codespaces, and you can use a browser on your local machine to access the web API. The idea is that you can use it just like you do on your local development machine.

Code Spaces creates port forwarding to your local machine so you can interact with the web API.

## Port Forwarding Issue
When you run your project, you will see a pop-up in your code spaces telling you you can browse your website on the HTTP port. But if you click the `Open in browser button in the popup, you will not be able to open the site.

The issue here is that the default .NET web API project does port forward to HTTPS. So you will be redirected to the HTTPS port, which is not supported within Code Spaces. 

The easiest way to fix it is to remove the following line. 

```csharp
 app.UseHttpsRedirection();
```
This will stop the default HTTPS forwarding, and you will be able the use the HTTP port again. There might be a better solution than this one that is most secure when running this web API in a production environment. 

So, what if we do not want to remove the line? What other options do we have? When you use the `dotnet run` command to start your project, the terminal will show something like this.
```
 Now listening on https://localhost:7198.
 Now listening on http://localhost:5000.
``` 
You can click on these URLs to open the website, but as we already know, the `localhost:5000` does not work. If you click on the HTTPS URL, you will see the website appear. 

There is another option to access the HTTP port if you want. On the `Ports` tab, you can `Copy the local address`; this will copy the `xxx.5000.preview.app.github.dev`. You can use this address in your browser address bar. If you paste it like this, it will still use the forward to HTTPS, and the website will still not appear.

But if you use this address with, for example, `/swagger` behind the URL, it will show the swagger page. The same goes for the other controllers like the default `/WeatherForecast` as long as you do not use the root.