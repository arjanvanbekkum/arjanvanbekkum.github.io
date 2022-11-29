---
layout: post
title: "GitHub Codespaces port forwarding connection refused "
date: 2022-11-29
summary: "When you start a .NET 6 webapi project in GitHub Codespaces the port forwarding does not work. The connection is refused by localhost"
minute: 5
tags: [dotnet, github]
---

GitHub recently annouced Codespaces is available for everyone. If you do not know what Codespaces is, check it [here](https://github.com/features/codespaces). Basically it is Visual Studio Code running on a VM in Azure availble from your webbrowser.

I wanted to play around with Codespaces a little more, so I decided to create a new .NET webapi project. As I have always been a Visual Studio userfor .NET development, so using Codespaces for .NET was a new expiriance for me. 

## .NET webapi project
Let's first create a new .NET project, to do this, you can run the following command
```cli
 dotnet new webapi --language "C#"
```
This will create a new .NET webapi project with some basic items like a controller and support for swagger.

You can run this project by executing the following command. 

```cli
 dotnet run
```
This will start the project in Code Spaces and you can use a browser on your local machine to access the webapi. The idea behind it is you can use it just like you do on your local development machine.

Code Spaces creates port forwarding to your local machine so you can interact with the webapi.

## Port Forwarding Issue
When you run your project you will see a pop-up in your code spaces telling you you can browse your website on the http port. But if you click the `Open in browser` button in the popup you will not be able to open the site.

The issue here is that the default .NET webapi project does portforward to HTTPS. So you will be redirected to the https port and that is not supported within Code Spaces. 

The easiest way to fix it is to just remove the following line. 

```csharp
 app.UseHttpsRedirection();
```
This will stop the default https forwarding and you are able the use the http port again. It might not be the solution that is most secure when running this webapi in a production environment. 

So, what if we do not want to remove the line, what other options do we have. When you use the `dotnet run` command to start your project, the terminal will show something like this
```
 Now listening on: https://localhost:7198
 Now listening on: http://localhost:5000
``` 
You can click on this urls to open the website, but as we already know the `localhost:5000` does not work. if you click on the https url you will see the website appear. 

There is another option to access the http port if you want. On the `Ports` tab you can `Copy the local address`, this will copy the `xxx.5000.preview.app.github.dev`. You can use this address in your browser address bar. If you paste it like this it will still use the forward to https and the website will still not appear.

But if you use this address with for example `/swagger` behind the url, it will show the swagger page. The same goes for the other controllers like the default `/WeatherForecast` as long as you do not use the root.
