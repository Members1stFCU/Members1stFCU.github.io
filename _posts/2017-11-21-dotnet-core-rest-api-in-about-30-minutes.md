---
title: .NET Core REST API in About 30 Minutes
author: Evan Kaiser
tags:
  - .NET Core
  - REST API
  - Programming Guides
---

Today I'm going to go over how to create a .NET Core 2.0 REST application. The result will be a very basic Model-Controller application with an In-Memory database that allows you to perform a few operations on a simple Airplane entity. It will use the new simplified VS Code IDE and the .NET Core CLI (Commandline Interface) in order to allow you to get a new app up and running in about 30 minutes!

If you have not set up your environment yet, please take a look at the previous article in this series, [Setting up .NET Core](../setting-up-dotnet-core).

***
This is an ongoing series on .NET Core.

1. [Setting up .NET Core](../setting-up-dotnet-core)
2. .NET Core REST API in About 30 Minutes
3. [.NET Core Request/Response](../dotnet-core-request-response)

## Setup

### VS Code and Directories

Now that you have the tools, it's time to set up your new project. First, open VS Code and press ```Ctrl+` ``` to open the built-in terminal. On Linux/MacOS, you'll find that it says "bash" or whatever your default scripting shell is in the top-right window selector. In Windows 10 it will say "powershell".

**NOTE**: for the purposes of this tutorial, when discussing commands/paths, I'll be using Windows 10 as the default OS.

Using the terminal navigate to where you want to put this sample project. I like to use `%USERPROFILE%\Source\Repos`, because this is where Visual Studio defaults git repositories. To create this directory, use the following commands:

```
cd $env:UserProfile
mkdir Source\Repos
```

Whether it was already there or newly created, navigate to the `Repos` directory and create a new directory for this project:

```
cd $env:UserProfile\Source\Repos
mkdir AirDemo
```

Now, in VS Code, click on the folders icon (top-left icon in left sidebar). Click the `Open Folder` button and select the new `AirDemo` folder. You will now notice that when you open the terminal, you will be automatically placed in that folder.

***
### Create Your First Project

To create a new .NET Core project, we'll use the CLI, which stands for "Command Line Interface". First, let's create a project for the [Domain Model](https://en.wikipedia.org/wiki/Domain_model).

```
dotnet new classlib -n AirDemo.Domain
```

***
:bulb: *Tip*

You may notice that the first time you have C# code in the project and/or you open a C# file, it will ask you to install the C# VS Code extension. You will definitely want to do this. If it did not ask you to, you can install it through the command line:

```
code --install-extension ms-vscode.csharp
```
***

You'll note the new project in your Explorer window has appeared with the `.csproj` and a file called `Class1.cs`. Examine these two files. `Class1.cs` is your typical C# class that gets created in Visual Studio when you add a new project. However, note that the `.csproj` is greatly simplified compared to the project files created by VS2015 and earlier. This is a vast improvement that reflects how .NET Core focuses primarily on code above configuration allowing for better streamlined, modular, and testable code.

Also, note that the setting `TargetFramework` is set to `netstandard2.0`. This is to allow any framework that supports the .NET Standard library to use your library. You can find the list of supported frameworks [here](https://docs.microsoft.com/en-us/dotnet/standard/net-standard).

Since, for the purposes of this tutorial, I've decided to use Entity Framework Core as my ORM, we will need a reference to the library. You can use the CLI to add nuget packages, like so:

```
dotnet add .\AirDemo.Domain\ package Microsoft.EntityFrameworkCore
```

Take a look at the `.csproj` and note that it included your nuget package reference directly in this file. This is another change from VS2015 and earlier versions that included a package.json containing this info. This nicer, consolidated approach comes with the added benefit that packages will be carried along when projects that reference them are referenced by other ones. For example, when we ultimately create the Web API endpoint project in this tutorial, we will not need to explicitly add EF Core since we will already have a reference to the Domain project.

## Start Coding!

Now we have to start building up our Domain Model. For this tutorial, I've chosen to use an Airplane as our first entity, because it is a good example of a real-world thing that can contain various properties and behaviors, like the following:

* string ModelNumber, i.e. "Boeing 747"
* string SerialNumber, i.e. "123456789"
* string CurrentAirportCode, i.e. "BWI"
* int SeatCount
* decimal WeightInKilos
* DateTimeOffset? LastTakeoffTime
* DateTimeOffset? EstimatedLandingTime
* Fly(TimeSpan estimatedTripTime);
* Land(string airportCode);

Let's start off by creating our Airplane. We'll worry about how we're going to persist it to the database later. Add a new file to the Domain project that looks like this:

*Airplane.cs*
```c#
using System;

namespace AirDemo.Domain
{
    public class Airplane
    {
        public virtual Guid AirplaneId { get; private set; }
        
        public virtual string ModelNumber { get; set; }
        
        public virtual string SerialNumber { get; set; }
        
        public virtual string CurrentAirportCode { get; private set; }
        
        public virtual int SeatCount { get; set; }
        
        public virtual decimal WeightInKilos { get; set; }
        
        public virtual DateTimeOffset? LastTakeoffTime { get; private set; }
        
        public virtual DateTimeOffset? EstimatedLandingTime { get; private set; }

        public virtual void Fly(TimeSpan estimatedTripTime)
        {
            this.CurrentAirportCode = null;
            this.LastTakeoffTime = DateTimeOffset.Now;
            this.EstimatedLandingTime = DateTimeOffset.Now.Add(estimatedTripTime);
        }

        public virtual void Land(string airportCode)
        {
            this.CurrentAirportCode = airportCode;
            this.LastTakeoffTime = null;
            this.EstimatedLandingTime = null;
        }
    }
}
```

The properties that are determined by behavior (i.e. flying and landing) are set as private so that the entity itself maintains its state based on behavior. This is a way to use Code First entities to our advantage to build Object-Oriented applications.

One more thing we might want to do is decide which properties are immutable. Since certain fields never change on an airplane, i.e. Model Number, Serial Number, Weight, and Seat Count, there is no particular reason why we should allow those attributes to be set externally. Instead, we can have them be passed in on the constructor so that a calling service must set them when creating the entity. We can then make the empty constructor private to avoid creating it without the appropriate data (EF will use reflection magic to create the entity to look like what is in the record so it does not need a public empty constructor).

***
:bulb: *Tip*

Setting your properties and methods to `virtual` makes it easy to mock/override things in the future. However it is not strictly necessary. If you want to have better control of what a subclass can do with the properties, you may wish to eschew or be more selective with your use of this keyword.

***

Modify the rest of the properties to be private and add a constructor like so:

*Airplane.cs*
```c#
using System;

namespace AirDemo.Domain
{
    public class Airplane
    {
        private Airplane()
        {
        }

        public Airplane(string modelNumber, string serialNumber, int seatCount, decimal weightInKilos)
        {
            this.AirplaneId = Guid.NewGuid();
            this.ModelNumber = modelNumber;
            this.SerialNumber = serialNumber;
            this.SeatCount = seatCount;
            this.WeightInKilos = weightInKilos;
        }

        public virtual Guid AirplaneId { get; private set; }
        
        public virtual string ModelNumber { get; private set; }
        
        public virtual string SerialNumber { get; private set; }
        
        public virtual string CurrentAirportCode { get; private set; }
        
        public virtual int SeatCount { get; private set; }
        
        public virtual decimal WeightInKilos { get; private set; }
        
        public virtual DateTimeOffset? LastTakeoffTime { get; private set; }
        
        public virtual DateTimeOffset? EstimatedLandingTime { get; private set; }

        public virtual void Fly(TimeSpan estimatedTripTime)
        {
            this.CurrentAirportCode = null;
            this.LastTakeoffTime = DateTimeOffset.Now;
            this.EstimatedLandingTime = DateTimeOffset.Now.Add(estimatedTripTime);
        }

        public virtual void Land(string airportCode)
        {
            this.CurrentAirportCode = airportCode;
            this.LastTakeoffTime = null;
            this.EstimatedLandingTime = null;
        }
    }
}
```

We now have an almost immutable entity that is only modified by two particular methods, such that the entity is in control of the state of all its own properties.

***
### Persisting the Airplane

It's great to be able to create an entity in memory, but we need to be able to persist our entity to a database. For now, we're going to use a mock In-Memory database provided by EF Core, such that we don't need to get into Migrations or creating the actual DB yet.

In order to persist and read entities, we need to create a Database Context, which allows us to take in-memory entities and write them to the DB, or query the set of entities in a certain "set" (i.e. Table).

The Context is very easy to create, it is simply a class that inherits from DbContext and contains a public property representing the set of entities.

*AirplaneContext.cs*
```c#
using Microsoft.EntityFrameworkCore;

namespace AirDemo.Domain
{
    public class AirplaneContext : DbContext
    {
        public AirplaneContext(DbContextOptions options) : base(options)
        {
        }

        public virtual DbSet<Airplane> Airplanes { get; private set; }
    }
}
```

All we had to do was inherit from DbContext, supply the constructor that accepts options, and then create a property that returns a `DbSet` typed to the entity you wish to return. By default, EF expects the property to be the pluralization of the entity (which is also what the table name generated by Migrations would be called). Note that just like with the entity properties, the `DbSet` may also have a private setter, which essentially means only EF is responsible for setting it.

Our Domain is now ready to perform [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operations on the Airplane!

## Create the API Project

In order to interact with our Domain, we have to provide our REST API layer to perform the HTTP calls with a JSON data type. Create another project that will contain our API endpoints (and reference the Domain), using the following commands.

```
dotnet new webapi -n AirDemo.Api
dotnet add .\AirDemo.Api\ reference .\AirDemo.Domain\AirDemo.Domain.csproj
```

Let's take a look at a few files before moving on to coding...

***
### AirDemo.Api.csproj

Notice how this project file is a little more involved than the Domain one. One thing it does have in common is that it also has a `TargetFramework`, however it is set to `netcoreapp2.0` instead of `netstandard2.0` like the Domain project was. This is due to the fact that this is not a standard library, but the actual app itself. We need to create it as a true .NET Core API in order to have the tooling run it.

A few other items to take note of:
* `<Folder Include="wwwroot\" />` - This is used for projects where you are intending to serve MVC Razor or static files and could be ignored/removed for a pure WebAPI.
* `<PackageReference Include="Microsoft.AspNetCore.All" Version="2.0.0" />` - A reference to the Nuget package for ASP .NET was automatically added so that we could reference the ASP .NET libraries and so it could configure the Program/Startup. In 2.0 they introduced a nice new `.All` package that includes all the different ways to host .NET Core. This could theoretically be streamlined by including only the packages needed (such as: Kestrel, Http.Sys, etc), but for now, we'll just leave it to make life easier.
* `<DotNetCliToolReference Include="Microsoft.VisualStudio.Web.CodeGeneration.Tools" Version="2.0.0" />` - This adds a tooling reference for Visual Studio 2017 Code Generation, which we won't take advantage of currently since we're using VS Code. Eventually, we will want to add a similar tooling entry for Entity Framework Migrations.

***
### Program.cs

The entry point for all .NET Core apps has been standardized to the `Program::Main` method, which is nice, because it allows customization of the web host down to the nitty gritty level, if necessary. The code in this method will build the default web host and run it as a service. In Development, it will run using the [Kestrel web server](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel?tabs=aspnetcore2x) on port 5000.

Try running the app for the first time, using the following command (you can also `cd` into the API directory and simply type `dotnet run`):
```
dotnet run --project .\AirDemo.Api
```

You'll notice the text in the console indicating that your app is running and listening on `Http://localhost:5000`. If this is truly the first time you've run a .NET Core application, you'll probably also notice that it says: `Hosting environment: Production`.

We definitely don't want to be running using `Production` settings, so let's fix that. Temporarily set the `ASPNETCORE_ENVIRONMENT` variable, like so:
```
$env:ASPNETCORE_ENVIRONMENT="Development"
```

***
:bulb: *Tip*

You should also change this in your Environment Variable settings so that you don't have to remember to change it every time. In Windows 10, search for "Environment Variables" and click "Edit environment variables for your account". Then under "User variables", click "New..." and add a new entry with the `Name=ASPNETCORE_ENVIRONMENT` and the `Value=Development`.

***

### Startup.cs

Here is where all the application configuration magic happens.

The startup constructor injects the configuration automatically. We'll see in a moment where the configuration can be set up, but just remember that this is where it comes in. You may also optionally inject some other things that are already setup in the IoC container, such as `IHostingEnvironment`, then save it to another public property on the class to be used later (for example, in `ConfigureServices`).

The IoC Container can be setup using the `ConfigureServices` method. We'll see in a bit how we have to use this to configure the DB context so that it can be injected into our app.

The `Configure` method is responsible for setting up the application [Middleware](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware?tabs=aspnetcore2x). You can use built-in middleware (such as CORS provided by the `Microsoft.AspNetCore.Cors` and `Micrsofot.AspNetCore.All` nuget packages), and you may also write your own custom middleware, which I will address in a later post.

***
### appsettings.json and appsettings.Development.json

This explains why we set our ASPNETCORE_ENVIRONMENT to Development. The `appsettings.json` is the base configuration file that the app will read from for all environments. The other file, `appsettings.Development.json`, will be merged in when the environment variable is set to Development. Similarly, when on a server where that environment variable is "Test", it would merge the `appsettings.json` and `appsettings.Test.json` files.

Note that the merge will give precedent to the environment-specific file. Therefore if a key exists in both places, the environment file will win. It will also allow you to partially replace certain keys within configuration sections (which are objects within the JSON). For example, taking into account the following two files:

*appsettings.json*
```json
{
    "Service": {
        "ApiKey": "123456",
        "Timeout": 10
    }
}
```

*appsettings.Production.json*
```json
{
    "Service": {
        "ApiKey": "999999"
    }
}
```

The resulting configuration would be:
```json
{
    "Service": {
        "ApiKey": "999999",
        "Timeout": 10
    }
}
```

In other words, it does not "overwrite" the Service section, it will merge it and only overwrite the keys specified explicitly.

## Controllers

Controllers are the entry point for the app defining all the API routes to perform queries and transactional requests. Let's look at the Controller that they have provided as an example, `ValuesController.cs`. Open this file and note a few things about it:

1) The `[Route("api/[controller]")]` attribute on the class defines how the route from the root of this API should look. The `[controller]` placeholder in the route is substituted with the name of the class without the word "Controller" (not case sensitive). Therefore, the route to this controller would be `/api/values`.

2) The 5 standard API methods on a particular controller, are provided: GET (all), GET (by ID), POST (i.e. create), PUT (i.e. update), and DELETE.

3) Note how the methods take in plain POCO/simple types and return the same. Although this still allows for passing `json` back and forth by default, it is sometimes good to return `IActionResult` instead, in order to provide certain codes, for example 404 if not found in the `GET(int id)` or 201 to indicate "Created" for the `POST`.

4) Attributes for the method, i.e. `HttpGet` optionally take a route parameter. This is appended to the controller route to form the full route. For example, when specifying `{id}`, the full route would be `/api/values/{id}` where `{id}` is the ID of the record to work with.

***
### Airplane Controller

Let's use this as an example to start creating our Airplane API.

First we need to create the controller. While we're at it, let's also inject the `AirplaneContext`, since we know we're going to need to interact with the Domain.

*AirplanesController.cs*
```c#
using AirDemo.Domain;
using Microsoft.AspNetCore.Mvc;

namespace AirDemo.Api.Controllers
{
    [Route("api/[controller]")]
    public class AirplanesController : Controller
    {
        private readonly AirplaneContext _context;

        public AirplanesController(AirplaneContext context)
        {
            _context = context;
        }
    }
}
```

Now we need at least one method to call. For now, let's just add the `GET` with no parameters to get all records. Since we have no way of creating these records yet, it will simply return an empty array, but will at least allow us to try out what we have so far. Add the following method to `AirplanesController.cs`:

```c#
// GET api/airplanes
[HttpGet]
public IEnumerable<Airplane> Get()
{
    return _context.Airplanes;
}
```

Let's try running the app and getting the results. Call `dotnet run` and just navigate to `http://localhost:5000/api/airplanes`.

Uh oh, you got an error: `InvalidOperationException: Unable to resolve service for type 'AirDemo.Domain.AirplaneContext' while attempting to activate 'AirDemo.Api.Controllers.AirplanesController'.`

Well, that was to be expected. Since we're attempting to inject the `AirplaneContext`, it needs to be part of the IoC Container, but we haven't set that up yet. Let's do that now.

***
### Add the Context to the IoC

As mentioned before, the `ConfigureServices` method in `Startup.cs` is where you add to the container. .NET Core has a few methods built in for adding to the IoC container. It is important to understand the distinctions of each:

* `services.AddTransient`
  * Registers the service as transient, meaning that any time it is resolved (i.e. when it is being injected into the constructor of a class), the IoC will instantiate a new instance of the service. Therefore, if you have 2 different classes using this service, they will have 2 different instances with their own versions of the properties.
* `services.AddScoped`
  * Registers the service for the life of an API request. So the first time the service is resolved after a request begins, it will create the instance, but any other time it is resolved after that until the request ends it will use that same instance.
* `services.AddSingleton`
  * Registers the service as a Singleton, i.e. when the app starts, the first time it is resolved it will create the instance and use that same instance for the life of the application process, meaning it will be shared by all requests.
* `services.AddDbContext`
  * This is a special method that registers the DB Context. It will use the Scoped lifetime, i.e. the same as if you called `AddScoped`. However, the point of this method is to provide additional configuration options for the context.

Now, knowing that, you've probably guessed we will be using `AddDbContext`. Add the following line above `services.AddMvc()` (it is typically best practice to leave `services.AddMvc()` and `app.UseMvc()` at the bottom of their respective methods).

```c#
services.AddDbContext<AirplaneContext>(opt => opt.UseInMemoryDatabase("AirDemo"));
```

***
:bulb: *Tip*

When you have a missing namespace reference in your code, you can click on the item (i.e. AirplaneContext in the previous line) and press Ctrl+. in VS Code to provide a list of options. In the above example, this should allow you to select: `using AirDemo.Domain;`.

***

You will have an unresolved reference to `UseInMemoryDatabase`. This is because even though AirDemo.Domain comes with EF (and thus the extension method `AddDbContext`), it does not contain every OLE DB Provider in that library. Therefore, to connect to a provider (i.e. In Memory, SQL Server, SQLite, etc), we need to indicate to the API how to do so. Since, for this demonstration, we're using In Memory, make sure your terminal is in the API project directory and call the following:

```
dotnet add package Microsoft.EntityFrameworkCore.InMemory
```

Now, using VS Code's missing reference suggestions, you should see an option for `using Microsoft.EntityFrameworkCore`. If you don't see this suggestion, try using `dotnet restore` and/or restarting VS Code. Sometimes it is slow to pick up the new package added and requires a restart.

Now try running the project and navigating to `http://localhost:5000/api/airplanes`. The error should be gone, and you should receive an empty array response, i.e. `[]`.

***
### More API Methods

Now that we have it running and know that it will work, let's try adding the remaining API methods!

#### GET (all) Revisited

  First, let's change the existing `GET` to an async method. The `async/await` pattern allows us to offload DB access and other possibly long-running methods to free up the ASP worker process. Obviously this will not be the case with this simple app, but it doesn't hurt to get used to always using `async/await`!

 ```c#
  // GET api/airplanes
  [HttpGet]
  public async Task<IEnumerable<Airplane>> Get()
  {
      return await _context.Airplanes.ToAsyncEnumerable().ToList();
  }
  ```

  Let's go over the changes that make this work:
  * `async`: Marks the method as asynchronous to offload it to a separate thread that is not the ASP worker process.
  * `await`: Tells the thread to wait on the other async method being called (specifically, `IAsyncEnumerable::ToList()`). Without this, the method would do nothing immediately and then go off to perform the query afterward.
  * `ToAsyncEnumerable()`: Modifies the DB call to tell it to perform the query asynchronously. Returns the `IAsyncEnumerable`, which is an EF Core construct that causes any method that materializes the results to be asynchronous.
  * `Task<IEnumerable<Airplane>>`: Returns a Task allowing the calling method (the guts of ASP .NET Core, in this case) to await the results.

***
#### POST

  Now, before we can do much more, we need to be able to create a record. Add the following to your class:

  ```c#
  // POST api/airplanes
  [HttpPost]
  public async Task Post([FromBody]Airplane request)
  {
      _context.Add(request);
      await _context.SaveChangesAsync();
  }
  ```

  This method will take the `json` representing an Airplane entity from the body of the request, add it to the context, and then save the changes of the added entity to the context, thus adding it to the In-Memory database. Try it out, by opening Postman, and doing the following:
  * Open a new request by clicking the + icon if you already have a tab open, or simply use the existing tab
  * Use the dropdown where it says "GET" and change it to "POST".
  * To the right of the dropdown, enter the URL: `http://localhost:5000/api/airplanes`
  * Below that, click `Body`, select `raw`, and then click where it says `Text` and instead select `JSON (application/json)`
  * In the text area for the request body, enter the following:
  ```json
  {
      "ModelNumber": "AZ-999",
      "SerialNumber": "12345",
      "CurrentAirportCode": "BWI",
      "SeatCount": 232,
      "WeightInKilos": 1234567
  }
  ```
  * Click "Send"

  Above the bottom panel, to the far right, you should see: `Status: 200 OK`, along with the request time. If you do not see 200, there was an error. Check the console log to see if you can fix the error. If it was added correctly, you should be able to open up a new Postman tab, leave it as a `GET`, and enter the URL: `http://localhost:5000/api/airplanes` to see the results containing the Airplane you just added.

  ***
  :question: *Question*

  You'll probably be thinking, "Didn't we specifically make the empty constructor and all the properties of Airplane private, so calling code would have to use the constructor? How did we create it without doing that?". Well yes, normally we wouldn't be asking for an actual EF entity in the request, or returning it back to the client. Usually we would use Request/Response "ViewModels" for the API endpoint layer. 
  
  However, for the sake of simplicity in the initial setup, we'll just use the entity. The .NET Core default JSON serializer allows private properties, so it will work for now. In future posts I will go over using a Service layer, containing "ViewModels", to prevent the client from being able to control the entity's state.
  ***

***
#### GET (one)

  Lets now see how the ability to get a single record is written. It is pretty straightforward.

  ```c#
  // GET api/airplanes/12345
  [HttpGet("{sn}")]
  public async Task<Airplane> Get(string sn)
  {
      return await _context.Airplanes.Where(x => x.SerialNumber == sn).ToAsyncEnumerable().FirstOrDefault();
  }
  ```

  Now, restart the app, perform the `POST` again (since the In-Memory database is wiped on the restart), and then use Postman to get: `http://localhost:5000/api/airplanes/12345`.

  Notice that instead of using the record's primary key (i.e. `AirplaneId`), we used the `SerialNumber` (passed in as `sn`) to identify the record to get. This is to avoid having to put the ugly 36-character GUID in the URL. This approach uses a unique property that is human readable to provide more user-friendly URLs. For example, if referencing a User, you might use their UserId. 
  
  For other cases, i.e. a blog post, you might need to generate a "slug" of the title. [Slugification](https://you.tools/slugify/) is the process of taking something that won't be friendly to a URL, i.e. "The Best Blog Post: Revisited" and turn it into something that is, i.e. "the-best-blog-post-revisited". We will go over how to generate a Slug in a later post.

***
#### PUT

  Now let's consider the `PUT` HTTP method. In a purely CRUD application, the PUT method simply updates every single property with what the client passed up. Sometimes this is fine. In the case of our application, however, we only have 2 ways that we want to modify the entity. It is not possible to retrieve it from the context, update all the properties, and save it. Therefore, we're going to have 2 different PUT methods called `Fly` and `Land`.

  ```c#
  // PUT api/airplanes/12345/fly
  [HttpPut("{sn}/fly")]
  public async Task Fly(string sn, [FromBody]TimeSpan estimatedTripTime)
  {
      var airplane = _context.Airplanes.Where(x => x.SerialNumber == sn).FirstOrDefault();
      if (airplane != null)
      {
          airplane.Fly(estimatedTripTime);
      }

      await _context.SaveChangesAsync();
  }

  // PUT api/airplanes/12345/land
  [HttpPut("{sn}/land")]
  public async Task Land(string sn, [FromBody]string airportCode)
  {
      var airplane = _context.Airplanes.Where(x => x.SerialNumber == sn).FirstOrDefault();
      if (airplane != null)
      {
          airplane.Land(airportCode);
      }

      await _context.SaveChangesAsync();
  }
  ```

  Let's try it out.
  1. Restart the app and perform the `POST` again
  2. Use Postman to make a `PUT` call to `http://localhost:5000/api/airplanes/12345/fly` with the request body simply containing the `json`: `"00:30:00"`.
  3. Now, if you do a `GET`, you should see that the `lastTakeoffTime` is set to the time you made the request and the `estimatedLandingTime` is set to 30 minutes later.
  4. Use Postman to make a `PUT` call to `http://localhost:5000/api/airplanes/12345/land` with the request body simply containing the `json`: `"JFK"`.
  5. Now, the times should both be `null` and the `currentAirportCode` is set to JFK! This demonstrates the advantages of using an Object Oriented approach. We've allowed the client to only care about one piece of data for each `PUT` request, but have modified up to 3 different properties correctly, rather than relying on the client to do that work!

***
#### DELETE

  Finally, the `DELETE` method is pretty straightforward:

  ```c#
  // DELETE api/airplanes/12345
  [HttpDelete("{sn}")]
  public async Task Delete(string sn)
  {
      var airplane = _context.Airplanes.Where(x => x.SerialNumber == sn).FirstOrDefault();
      if (airplane != null)
      {
          _context.Airplanes.Remove(airplane);
      }

      await _context.SaveChangesAsync();
  }
  ```

  Try it out by restarting the app, performing the `POST` (so you have something to delete), then have Postman hit the `DELETE` method of the URL: `http://localhost:5000/api/airplanes/12345`. Now if you perform a `GET` for that record, you should receive a `204 - No Content`.

  ***
  :thought_balloon: *Thought*

  Based on [good REST patterns](https://restpatterns.mindtouch.us/HTTP_Status_Codes), `204 - No Content` is really not want we want to see for the `GET` of a missing record. We really want to see `404 - Not Found`, but we'll have to modify it to return an `IActionResult`, which we will go over in a later post.

## Wrap Up

You now have a fully functional .NET Core REST API with 6 different API methods to interact with the Airplane entities. Now that we've gone over the basics, I will create some more posts later to address the following topics:

* Service Layer: ViewModels, `Result` class, and `IActionResults` for proper Status Codes
* Using a `MessageProvider` to track messages added by the code throughout the app, in order to provide more information to a client about why things failed
* Tracking Created/Updated fields using a `DateProvider`/`UserProvider`, and creating a reusable `GenericContext`.
* EF Migrations and persisting the context to a database
* Authentication using JSON Web Tokens and .NET Core Middleware
* Claims-based authorization using JSON Web Tokens
* Front-end integration and CORS
