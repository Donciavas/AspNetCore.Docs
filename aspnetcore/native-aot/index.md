---
title: ASP.NET Core support for .NET native AOT
author: mitchdenny
description: Learn about ASP.NET Core support for .NET native AOT
monikerRange: '>= aspnetcore-8.0'
ms.author: midenn
ms.custom: mvc
ms.date: 04/10/2023
uid: native-aot/index
---
# ASP.NET Core support for .NET native AOT.

:::moniker range=">= aspnetcore-8.0"

In .NET 8.0 the ASP.NET Core team is introducing support for .NET native AOT. Not all ASP.NET Core features are compatible with native AOT at this time. This article shows how to get started with native AOT support in ASP.NET Core, pros and cons of using .NET native AOT, and what limitations exist. For more information on .NET native in general please refer to (TODO: insert native AOT link).

## Why use .NET native AOT with ASP.NET Core?

Using the .NET native AOT deployment model provides the following benefits:

* **Minimize disk footprint**; when publishing using native AOT a single executable is produced containing just the code from external dependencies that is used to support the program. Reduced executable size can lead to smaller container images (in containerized deployment scenarios) which can reduce deployment time.
* **Reduced startup time**; native AOT applications can show reduced start-up times which means the application is ready to service requests quicker. This can also help during deployment where container orchestrators need manage transition from one version of the application to another.
* **Reduce memory demand**; native AOT applications can have reduced memory demands depending on the nature of the work being performed by the application. This reduced memory consumption can lead to greater deployment density and improved scalability.

## When to avoid using .NET native AOT with ASP.NET Core?

Not all features in ASP.NET Core are currently compatible with .NET native AOT. The following table summarizes ASP.NET Core feature compatability with .NET native AOT:

| Feature                       | Fully Supported | Partially Supported | Not Supported |
| ----------------------------- | --------------- | ------------------- | ------------- |
| ASP.NET Core Minimal APIs     |                 | Yes[^1]             |               |
| ASP.NET Core MVC              |                 |                     | No            |
| ASP.NET Core Blazor           |                 |                     | No[^2]        |
| ... TODO                      |                 |                     |               |

In addition to API compatability, deploying ASP.NET Core applications can still be more efficient in some scenarios. For example when a container image is created for your application that is based on the ASP.NET Core base images, an additional layer is created containing just your binaries which will often be smaller than the native AOT binary. If the nodes that host your container images already contain a cached copy of the base layers of the container image the time to download the additiona layer can be quite small. The benefits of .NET native AOT deployment of ASP.NET Core applications will depend heavily on the exact details of your deployment strategy.

## Getting started with .NET native AOT deployment in ASP.NET Core

To help developers get started deploying with .NET native AOT in ASP.NET Core, we have created a variant of the API template which includes customizations to remove unsupported components from the application. Use the ```dotnet new``` command to create a new ASP.NET Core API application that is configured to work with .NET native AOT:

```
PS> dotnet new api -aot -n MyFirstAotWebApi && cd MyFirstAotWebApi
The template "ASP.NET Core API" was created successfully.

Processing post-creation actions...
Restoring C:\Code\Demos\MyFirstAotWebApi\MyFirstAotWebApi.csproj:
  Determining projects to restore...
  Restored C:\Code\Demos\MyFirstAotWebApi\MyFirstAotWebApi.csproj (in 302 ms).
Restore succeeded.
```

Before we look more closely at the code in the template, let's make sure that it can but published using .NET native AOT correctly by using the following command (note the versions of .NET 8.0+ that you are using may vary from the output shown below):

```
PS> dotnet publish /p:PublishAot=true
MSBuild version 17.6.0-preview-23171-02+b84faa7d0 for .NET
  Determining projects to restore...
  Restored C:\Code\Demos\MyFirstAotWebApi\MyFirstAotWebApi.csproj (in 241 ms).
C:\Code\dotnet\aspnetcore\.dotnet\sdk\8.0.100-preview.4.23176.5\Sdks\Microsoft.NET.Sdk\targets\Microsoft.NET.RuntimeIde
ntifierInference.targets(287,5): message NETSDK1057: You are using a preview version of .NET. See: https://aka.ms/dotne
t-support-policy [C:\Code\Demos\MyFirstAotWebApi\MyFirstAotWebApi.csproj]
  MyFirstAotWebApi -> C:\Code\Demos\MyFirstAotWebApi\bin\Release\net8.0\win-x64\MyFirstAotWebApi.dll
  Generating native code
  MyFirstAotWebApi -> C:\Code\Demos\MyFirstAotWebApi\bin\Release\net8.0\win-x64\publish\
```

Once the publish command finishes executing review the contents of output directory:

```
PS> dir bin\Release\net8.0\win-x64\publish

    Directory: C:\Code\Demos\MyFirstAotWebApi\bin\Release\net8.0\win-x64\publish

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a---          30/03/2023  1:41 PM        9480704 MyFirstAotWebApi.exe
-a---          30/03/2023  1:41 PM       43044864 MyFirstAotWebApi.pdb
```

At time of publishing the API template with AOT enabled produces a binary of about 9.4MB on Windows, although size may vary depending on the exact build of .NET 8.0 you are using as we are constantly making improvements and adjustments. This executable could be moved to a machine without the .NET Core runtime installed and it would execute.

```
PS> .\bin\Release\net8.0\win-x64\publish\MyFirstAotWebApi.exe
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5000
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Production
info: Microsoft.Hosting.Lifetime[0]
      Content root path: C:\Code\Demos\MyFirstAotWebApi
```

## Specific coding requirements

The ```Program.cs``` source file contains some important changes to make publishing to .NET native AOT error free:

```csharp
using System.Text.Json.Serialization;
using MyFirstAotWebApi;

var builder = WebApplication.CreateSlimBuilder(args);
builder.Logging.AddConsole();

builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.AddContext<AppJsonSerializerContext>();
});

var app = builder.Build();

var sampleTodos = TodoGenerator.GenerateTodos().ToArray();

var todosApi = app.MapGroup("/todos");
todosApi.MapGet("/", () => sampleTodos);
todosApi.MapGet("/{id}", (int id) =>
    sampleTodos.FirstOrDefault(a => a.Id == id) is { } todo
        ? Results.Ok(todo)
        : Results.NotFound());

app.Run();

[JsonSerializable(typeof(Todo[]))]
internal partial class AppJsonSerializerContext : JsonSerializerContext
{

}
```

A significant difference is that `Microsoft.AspNetCore.Builder.WebApplication.CreateBuilderSlim` is used to create the web application builder.  The `CreateBuilderSlim` method initializes the <xref:Microsoft.AspNetCore.Builder.WebApplicationBuilder> with incompatible ASP.NET Core features that are disabled.
<!-- Update the preceding with the following when the .NET 8 API is published :
<xref:Microsoft.AspNetCore.Builder.WebApplication.CreateBuilderSlim%2A>
-->

Because .NET native AOT is unable to use reflection at runtime we also need to use the source generator to generate the JSON serializer code to support the custom types that are being returned from the minimal API routes.

TODO: WIP

[^1]: TODO: List out support for specific minimal API operations.
[^2]: TODO: Add mention that Blazor web assembly can still call native AOT compiled backends.

:::moniker-end