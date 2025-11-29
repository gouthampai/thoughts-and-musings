---
title: "Part 1 - Get a .NET 10 Service Running in a Docker Container Locally"
published: 2025-12-01
draft: false
description: 'A step by step guide on how to get a .NET 10 service running in a container locally'
series: '.NET 10 on Azure Container Apps'
tags: ['dotnet', 'azure', 'containers']
---

At the time of writing, .NET 10 has just been recently released. I've been looking for an opportunity to learn more about the container offerings in Azure and .NET 10. So, let's do both and get something simple deploying to a Container App. After all, how hard can it be?

### Prerequisites
Before you start diving into the code, you will need some tools setup first to make the next steps go smoothly.
You'll need to install the .NET 10 SDK on your development machine from the Microsoft .NET downloads page [here](https://dotnet.microsoft.com/en-us/download/dotnet/10.0)

Since we will be using Azure and Github for this project, we will also require accounts on both services.

I would also suggest installing Docker Desktop or Rancher so you can run your api server container locally before deploying things to Azure.

I set up the required infrastructure in Azure using Terraform. If that scares you, you may choose to create the needed services manually through the Azure Portal. Further in this post, I will share the main.tf file I used to set things up, perhaps it will be helpful to you.

The last thing you may want is a tool like Postman or Insomnia to be able to make requests to your new API endpoints. If you are a badass who uses cURL to do that, you can skip installing such a tool on your machine.

### Generating a new .NET 10 API
Let's go ahead and create a new .NET project using the below command.
```shell
dotnet new webapiaot -n <name of your project>
```

This will create a new .NET 10 webapi project with AOT enabled. AOT is ahead of time compilation and provides us with some advantages that make sense when deploying to a container environment. Using AOT promises reduced disk footprint, quicker cold startup times and reduced memory usage. However, using it does have some drawbacks. Mainly, we are required to use minimal APIs as MVC is not yet supported for AOT compilation. More information on AOT compilation is available [here](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/native-aot?view=aspnetcore-10.0#the-web-api-native-aot-template) and [here](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/?tabs=windows%2Cnet8).

AOT aside, you should now have a folder with some generated .NET code in it. Let's take a look at what we have. We'll find a csproj file targeting .NET 10, a Program.cs file, and some json files specifying configuration. Inspecting the contents of Program.cs below, we can see some boilerplate for setting up the .NET API server, configuration of OpenApi, and two sample minimal apis endpoints.
```csharp title="Program.cs"
using System.Text.Json.Serialization;
using Microsoft.AspNetCore.Http.HttpResults;

var builder = WebApplication.CreateSlimBuilder(args);

builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonSerializerContext.Default);
});

// Learn more about configuring OpenAPI at https://aka.ms/aspnet/openapi
builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

Todo[] sampleTodos =
[
    new(1, "Walk the dog"),
    new(2, "Do the dishes", DateOnly.FromDateTime(DateTime.Now)),
    new(3, "Do the laundry", DateOnly.FromDateTime(DateTime.Now.AddDays(1))),
    new(4, "Clean the bathroom"),
    new(5, "Clean the car", DateOnly.FromDateTime(DateTime.Now.AddDays(2)))
];

var todosApi = app.MapGroup("/todos");
todosApi.MapGet("/", () => sampleTodos)
        .WithName("GetTodos");

todosApi.MapGet("/{id}", Results<Ok<Todo>, NotFound> (int id) =>
    sampleTodos.FirstOrDefault(a => a.Id == id) is { } todo
        ? TypedResults.Ok(todo)
        : TypedResults.NotFound())
    .WithName("GetTodoById");

app.Run();

public record Todo(int Id, string? Title, DateOnly? DueBy = null, bool IsComplete = false);

[JsonSerializable(typeof(Todo[]))]
internal partial class AppJsonSerializerContext : JsonSerializerContext
{

}
```

Let's check out the configured open api endpoint. Take a look at the generated launchsettings.json file and you should be able to tell what URL your API server will get served on locally. Mine was configured to `http://localhost:5144`. Yours might be different. Run `dotnet run` in the directory of your newly generated csproj file and go to `<your_startup_url>/openapi/v1.json`. We can now see an OpenAPI spec defined for the two minimal api endpoints we saw before in `Program.cs`. Using a tool like Postman, you can also make calls to `<your_startup_url>/todos` and you should get a response back with a list of todo items. If you don't, your dotnet development server might not be running. 

### Docker Time!
Add a new Dockerfile at the root of the repository where your new api folder is. Please note that Dockerfile has no extension. Mine ended up being placed just one folder up from where the new csproj file is.
```
|__Dockerfile
|__<new project folder>
|____newproject.csproj
```

Here is what your Dockerfile contents should look like
```docker
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build

# Build argument for target architecture
ARG TARGETARCH
ARG RUNTIME_ID=linux-${TARGETARCH}

# Install native build tools required for AOT
RUN apt-get update && apt-get install -y clang zlib1g-dev

WORKDIR /app

# Copy source code
COPY YOURPROJECT/ YOURPROJECT/
WORKDIR /app/YOURPROJECT

# Restore and publish in one step
RUN dotnet publish -c Release -o /app/publish -r ${RUNTIME_ID}

# Runtime stage - use runtime-deps for AOT
FROM mcr.microsoft.com/dotnet/runtime-deps:10.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .

# Set environment variables
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_ENVIRONMENT=Production

# Expose port
EXPOSE 8080

ENTRYPOINT ["./YOURPROJECT"]
```

At a high level, this is what the above Dockerfile instructs the build tool to do:
1. Using the dotnet 10 sdk image from microsoft as a basis, copy and build the .NET 10 project as a self-contained application
2. Using the slimmed down runtime-deps image, listen on port 8080 after running the built project code

While getting this running on my personal M2 MacBook Pro, I did run into a couple minor snags around the supported os/processor architectures that Docker thinks I want it to use. I needed to build the Dockerfile in such a way that it would be able to support both x64 and arm64 systems. To make this work, I had to add a RuntimeIdentifiers value to my csproj file as shown below and then run both `dotnet clean` and `dotnet build`. This generated the correct project.assets.json file. 

```xml
<PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <InvariantGlobalization>true</InvariantGlobalization>
    <PublishAot>true</PublishAot>
    <RuntimeIdentifiers>linux-x64;linux-arm64</RuntimeIdentifiers>
</PropertyGroup>
```

In the Dockerfile, I had to add `RUN apt-get update && apt-get install -y clang zlib1g-dev` to the Dockerfile so that the correct dependencies were present when trying to build the AOT .NET code using the .NET 10 sdk linux image. This is documented [here](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/?tabs=linux-ubuntu%2Cnet8#prerequisites).

To allow the Dockerfile to work for building images locally and in the Github Actions workflow, I also needed to add the following.

```docker
ARG TARGETARCH
ARG RUNTIME_ID=linux-${TARGETARCH}
```

I spent a fair amount of time troubleshooting this issue so I hope documenting it here helps others avoid wasting time trying to fix it.

### Build a Docker image
Run the below command to build your .NET 10 service into a Docker image. Note the required "." at the end of the command.
```shell
docker build -t net10api .
```
After a bit, the build process should have finished and your new image should be published locally. Let's go ahead and run it with the below command.

```shell
docker run -p 8080:8080 net10api:latest
```
You should see some logs scroll by saying that there is a .NET server listening on port 8080. Try to hit `http://localhost:8080/todos` in Postman and you should get back a list of todos if everything worked correctly. Congrats, you now have a running .NET 10 web api in a container!

In the next part, we will explore setting up Github Actions to deploy to a Container App. Stay tuned!


