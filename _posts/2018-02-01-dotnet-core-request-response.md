---
title: .NET Core Request/Response
author: Evan Kaiser
tags:
  - .NET Core
  - REST API
  - Programming Guides
---

This post will go over how to improve request/response patterns by adding a Service Layer that includes [Data transfer objects (DTOs)](https://en.wikipedia.org/wiki/Data_transfer_object) and the ability to return more detailed info about what happened in the request through the `IActionResult` class. 

In the last post of this series, we built a basic .NET Core REST API in the simplest way possible - by having the controllers interact directly with our Entity Framework Domain project and receive/respond entities directly. While this approach allowed us to create a quick API, it also can cause some problems and pitfalls, such as:
* The `GET` endpoints pass back Entity Framework entities, which may not be what we need to give the client. Oftentimes, we'll need to expand on the response with data from additional entities, descriptions, aggregations, etc. **_Solution_: Add DTOs**
* When an item is not found using the `GET (one)` endpoint, it returns status `204 No Content`, rather than the ideal `404 Not Found`. **_Solution_: Update API endpoints to return `IActionResult`**
* The `POST` endpoint allows the client to create a new entity with whatever data it can put on the properties. This is not ideal for a situation where we want to use Object Oriented techniques to allow the Entity to control its own status/state based on a set of limited inputs. **_Solution_: Service Layer**

Using the techniques in this post, we should be able to address all these issues and provide a more functional API.

***
This is an ongoing series on .NET Core.

1. [Setting up .NET Core](../setting-up-dotnet-core)
2. [.NET Core REST API in About 30 Minutes](../dotnet-core-rest-api-in-about-30-minutes)
3. .NET Core Request/Response
4. [Validation in .NET Core REST API](../validation-in-dotnet-core-rest-api)
5. [.NET Core Messages, Exception, and Logging](../dotnet-core-messages-exception-and-logging)

## The Code

You may retrieve the code from [https://github.com/members1stfcu/blog-dotnetcore-airdemo](https://github.com/members1stfcu/blog-dotnetcore-airdemo) in the state we left it at the end of the previous post, by using the following `git` commands (skip `git clone` if you've already grabbed it once):

```
git clone https://github.com/members1stfcu/blog-dotnetcore-airdemo.git
cd blog-dotnetcore-airdemo
git checkout -f blog/part2
```

### Postman

You may also import the postman collection for this demo (in the state it should be for this post) by clicking the `Import` button and selecting the `AirDemo.postman_collection.json` file at the root of the project.

***
## Service Layer

First let's start by adding a layer in-between the API and the Domain called the "Service Layer". As discussed above, this layer provides a set of DTOs and routines to populate and read from these request/response models in order to provide additional information beyond the simple entity properties, as well as a way to control how the Domain is used when performing data manipulation.

```
dotnet new classlib -n AirDemo.Service
```

Let's first change that `Class1.cs` that was generated into the following interface for our service:

*IAirplaneService.cs*
```c#
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using AirDemo.Domain;
namespace AirDemo.Service
{
    public interface IAirplaneService
    {
        Task<IEnumerable<Airplane>> GetAirplanes();
        Task<AirplaneResponse> GetAirplane(string sn);
        Task RegisterNewAirplane(Airplane plane);
        Task FlyAirplane(string sn, TimeSpan estimatedTripTime);
        Task LandAirplane(string sn, string airportCode);
        Task DeleteAirplane(string sn);
    }
}
```

Now, you'll notice the missing references to `AirDemo.Domain` and the `Airplane` class, so we'll have to add the reference to the Domain (and potentially close/reopen VS Code):
```
dotnet add .\AirDemo.Service\AirDemo.Service.csproj reference .\AirDemo.Domain\AirDemo.Domain.csproj
dotnet restore .\AirDemo.Service
```

***
### DTOs

Taking a look at the interface, you might think to yourself: *Did we really gain anything by adding this interface? We're essentially moving the bad behavior from the Controller (API Layer) to the Service Layer.* 

Let's fix that by creating Request/Response DTOs for each set of things being passed into or out of a method. These Data-Transfer Objects are used to provide the slimmest possible way of conveying the necessary data between client application and API server, by requesting only what is absolutely needed, and providing a response with only what is asked for.

Create a `Models` folder within the `AirDemo.Service` project and add the following classes:

*Models/AirplaneResponse.cs*
```c#
using System;
namespace AirDemo.Service.Models
{
    public class AirplaneResponse
    {
        public string ModelNumber { get; set; }
        public string SerialNumber { get; set; }
        public string CurrentAirportCode { get; set; }
        public int SeatCount { get; set; }
        public decimal WeightInKilos { get; set; }
        public DateTimeOffset? LastTakeoffTime { get; set; }
        public DateTimeOffset? EstimatedLandingTime { get; set; }
    }
}
```

*Models/AirplaneAddRequest.cs*
```c#
namespace AirDemo.Service.Models
{
    public class AirplaneAddRequest
    {
        public string ModelNumber { get; set; }
        public string SerialNumber { get; set; }
        public int? SeatCount { get; set; }
        public decimal? WeightInKilos { get; set; }
    }
}
```

*Models/AirplaneFlyRequest.cs*
```c#
using System;
namespace AirDemo.Service.Models
{
    public class AirplaneFlyRequest
    {
        public TimeSpan? EstimatedTripTime { get; set; }
    }
}
```

*Models/AirplaneLandRequest.cs*
```c#
namespace AirDemo.Service.Models
{
    public class AirplaneLandRequest
    {
        public string AirportCode { get; set; }
    }
}
```

Now, we can modify our interface to look like this:

*IAirplaneService.cs*
```c#
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using AirDemo.Service.Models;
namespace AirDemo.Service
{
    public interface IAirplaneService
    {
        Task<IEnumerable<AirplaneResponse>> GetAirplanes();
        Task<AirplaneResponse> GetAirplane(string serialNumber);
        Task RegisterNewAirplane(AirplaneAddRequest addRequest);
        Task FlyAirplane(string serialNumber, AirplaneFlyRequest flyRequest);
        Task LandAirplane(string serialNumber, AirplaneLandRequest landRequest);
        Task DeleteAirplane(string serialNumber);
    }
}
```

You'll notice now that we're not passing in and out any entities. This approach also has the added benefit of being able to add additional parameters to a request without modifying the method signature on your interface. For example, say we needed one additional bit of information for a request for an airplane to fly: the intended destination. On the previous interface declaration, we would have to modify the method signature to add the additional parameter and thus modify the interface itself. This is a violation of the [Open-Closed principle of SOLID development](https://scotch.io/bar-talk/s-o-l-i-d-the-first-five-principles-of-object-oriented-design#toc-open-closed-principle), and therefore should be avoided. Given this new interface, when an additional request parameter is needed it is not a parameter, but rather just another property on the request model.

A downside of this approach is then the fact that we no longer have a way of strictly requiring that bit of data, since a particular instance of the request model could leave that property null. However, we can fix that problem by simply using data annotations, like so:

```c#
[Required]
public string ModelNumber { get; set; }
```

Go ahead and add the `[Required]` annotation to every property on every `Request` class (which will require `using System.ComponentModel.DataAnnotations`). We will go over later how to check `ModelState` to validate these annotations and return a `400 Bad Request` error if they are invalid.

***
### Implementing the Service class

To call the Service layer from the API, we'll have to create an implementation of `IAirplaneService`. Add the following class to the `AirDemo.Service` project (we'll fill in the implementations of the methods in steps):

*AirplaneService.cs*
```c#
using System.Collections.Generic;
using System.Threading.Tasks;
using AirDemo.Domain;
using AirDemo.Service.Models;

namespace AirDemo.Service
{
    public class AirplaneService : IAirplaneService
    {
        private readonly AirplaneContext _context;

        public AirplaneService(AirplaneContext context)
        {
            _context = context;
        }
    }
}
```

***
:thought_balloon: *Thought*

Previously, we had injected the `AirplaneContext` into the Controller, but now we're going to inject it into the service and inject the service into the controller. The .NET Core IoC will automatically rearrange how these various classes are initialized (as long as they are all registered). Think about how we would need to create a reference to the service on the Controller without an IoC. It might look something like this (assuming we have created a factory to deliver a static instance of `DbContextOptions`):

```c#
    private readonly IAirplaneService _service;
    public AirplanesController()
    {
        var options = MyDbFactory.GetDbContextOptions();
        var context = new AirplaneContext(options);
        _service = new AirplaneService(context);
    }
```

It's not incredibly complex in this scenario, but as you continue adding services, the need for additional initializations increases (sometimes exponentially, depending on how many contexts, providers, or other services they require). The nice thing about the IoC is that it handles those initializations and dependency chains for you, however it is always good to keep in mind what that initialization might entail when architecting your application. If you can't easily imagine how to perform the dependency injection manually, you probably have too many dependencies somewhere, and the design needs to be reimagined.

***

Now, let's start implementing the methods that we defined on the interface. Recall that we had the following methods to implement:
* `Task<IEnumerable<AirplaneResponse>> GetAirplanes()`
* `Task RegisterNewAirplane(AirplaneAddRequest addRequest)`
* `Task FlyAirplane(AirplaneFlyRequest flyRequest)`
* `Task LandAirplane(AirplaneLandRequest landRequest)`

#### GetAirplanes Method

The Get method is pretty straightforward. We simply have to query the items and return the results as DTOs instead of entities. In a real system we'd probably want to have a request object with paging, sorting, and filtering because we wouldn't want to return a list of thousands of airplane entities in one go. However, for the purposes of this example, we'll keep it simple. Add this method to the AirplaneService class:

```c#
        public async Task<IEnumerable<AirplaneResponse>> GetAirplanes()
        {
            return await _context.Airplanes.ToAsyncEnumerable()
                .Select(a => new AirplaneResponse
                {
                    ModelNumber = a.ModelNumber,
                    SerialNumber = a.SerialNumber,
                    CurrentAirportCode = a.CurrentAirportCode,
                    SeatCount = a.SeatCount,
                    WeightInKilos = a.WeightInKilos,
                    LastTakeoffTime = a.LastTakeoffTime,
                    EstimatedLandingTime = a.EstimatedLandingTime
                }).ToList();
        }
```

Usually you're not going to have such a straightforward copy of properties from entity to DTO in a real application, but this does seem very redundant. You might ask if there's a way to automatically map properties from entity to DTO, and the answer is "yes", with the help of [AutoMapper](https://github.com/AutoMapper/AutoMapper). AutoMapper is one of those great tools that has the potential to be abused severely, and therefore it takes some thought regarding how you want to use it. Personally, I think the best way to avoid abuse is to only ever use a particular DTO once for a method. It is not good to assume that everytime you query for an airplane that you will want a particular mapping. Therefore if we only use `AirplaneResponse` in this `GetAirplanes` method, we are safe to create a mapping for it. You will still reduce useless redundant code through the use of AutoMapper's automatic property copy that uses the type and name of the property in each object to assume the data should be copied. Therefore using the same name when appropriate is advantageous.

Let's pull in AutoMapper using the following command:

```
dotnet add .\AirDemo.Service\AirDemo.Service.csproj package AutoMapper
dotnet restore
```

Next, add the following class to the `AirDemo.Service` project:

*AirplaneServiceMappingProfile.cs*
```c#
using AirDemo.Domain;
using AirDemo.Service.Models;
using AutoMapper;

namespace AirDemo.Service
{
    public class AirplaneServiceMappingProfile : Profile
    {
        public AirplaneServiceMappingProfile()
        {
            this.CreateMap<Airplane, AirplaneResponse>();
        }
    }
}
```

In the `AirDemo.Api` project, we need to register the mapping within the `ConfigureServices` method of `Startup.cs` (above `services.AddMvc()`):

```c#
    // AutoMapper
    var config = new MapperConfiguration(cfg =>
    {
        cfg.AddProfile<AirplaneServiceMappingProfile>();
    });
    services.AddSingleton<MapperConfiguration>(config);
```

Finally, provide the configuration to the `AirplaneService` by injecting it into the constructor and utilize the configured mapping (Note that this requires the manual addition of `using AutoMapper.QueryableExtensions`):

*AirplaneService.cs*
```c#
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using AirDemo.Domain;
using AirDemo.Service.Models;
using AutoMapper;
using AutoMapper.QueryableExtensions;

namespace AirDemo.Service
{
    public class AirplaneService : IAirplaneService
    {
        private readonly AirplaneContext _context;
        private readonly MapperConfiguration _map;

        public AirplaneService(
            AirplaneContext context,
            MapperConfiguration map)
        {
            _context = context;
            _map = map;
        }

        public async Task<IEnumerable<AirplaneResponse>> GetAirplanes()
        {
            return await _context.Airplanes
                .ProjectTo<AirplaneResponse>(_map)
                .ToAsyncEnumerable() // This is where the query breaks into Linq to Objects (i.e. the SQL query formed by the previous 2 lines is executed here)
                .ToList();
        }
    }
}
```

***
:bulb: *Tip*

In my opinion, it's probably best to use AutoMapper sparingly, and only in the cases where all or a majority of the properties are a straightforward copy from one value to the other. It tends to get messy quickly with more complex projections from multiple entities joined by navigation properties, aggregations, type conversions, and/or string manipulations.

***
#### GetAirplane Method

Now we'll also need a method for getting a singular instance of an Airplane. For now, we'll re-use the mapping by creating the method like so:

```c#
    public async Task<AirplaneResponse> GetAirplane(string serialNumber)
    {
        return await _context.Airplanes
            .Where(x => x.SerialNumber == serialNumber)
            .ProjectTo<AirplaneResponse>(_map)
            .ToAsyncEnumerable()
            .SingleOrDefault();
    }
```

***
:bulb: *Tip*

You're typically going to need to create a separate response object for querying a single thing (i.e. for an edit/view page) versus a collection (i.e. for a list page or search grid). In my experience, a response for a singular value is going to eventually need aggregates and child entities that you simply cannot afford, performance-wise, to return for every result in a grid (nor would that generally make sense to show in a grid or list page without polluting the screen).

***
#### RegisterNewAirplane Method

If you look back on the Controller method for the `POST` endpoint, you'll see that we were able to add the entity right into the context. This was undesirable because it skirted a lot of what the Domain entity might do when initialized, such as maintaining proper state and validation. So now instead of being handed an entity, we're handed an `AirplaneAddRequest` meaning we have to create the entity first.

Add the following method to the `AirplaneService` to accomplish registering the new airplane and having it be persisted to the database:

```c#
    public async Task RegisterNewAirplane(AirplaneAddRequest addRequest)
    {
        var airplane = new Airplane(addRequest.ModelNumber, addRequest.SerialNumber, addRequest.SeatCount ?? 0, addRequest.WeightInKilos ?? 0);
        _context.Add(airplane);
        await _context.SaveChangesAsync();
    }
```

We're now using the Service Layer to manipulate objects like `airplane` in an OOP way. When code begins to get more complex, and involves multiple entities, this will be advantageous when the entities begin interacting with each other. Instead of having to decide what properties to set on each entity, the service can focus on calling behaviors, such as in this hypothetical request:

```c#
IRepairable airplane = new Airplane(...);
IRepairer technician = new AirplaneMaintenanceUser(...);
technician.Repair(airplane, serviceRequest.DateServiced, serviceRequest.PartRepaired);
```

Just from reviewing the method, anyone, regardless of proficiency in code, should be able to tell that this method causes the Airplane repair log to be updated with the work done by the specified technician, including the date it occurred and what part was repaired. It even allows you to work with interfaces that are specific to the task at hand, which helps adhere to the [Single-responsibility principle of SOLID development](https://scotch.io/bar-talk/s-o-l-i-d-the-first-five-principles-of-object-oriented-design#toc-single-responsibility-principle).

***
#### Finishing the Service

The remaining methods for `Fly`, `Land`, and `Delete`, are almost identical to what we had previously put into the Controller:

```c#
    public async Task FlyAirplane(string serialNumber, AirplaneFlyRequest flyRequest)
    {
        var airplane = _context.Airplanes.Where(x => x.SerialNumber == serialNumber).FirstOrDefault();
        if (airplane != null)
        {
            airplane.Fly(flyRequest.EstimatedTripTime ?? new TimeSpan(0, 0, 0));
        }

        await _context.SaveChangesAsync();
    }

    public async Task LandAirplane(string serialNumber, AirplaneLandRequest landRequest)
    {
        var airplane = _context.Airplanes.Where(x => x.SerialNumber == serialNumber).FirstOrDefault();
        if (airplane != null)
        {
            airplane.Land(landRequest.AirportCode);
        }

        await _context.SaveChangesAsync();
    }

    public async Task DeleteAirplane(string serialNumber)
    {
        var airplane = _context.Airplanes.Where(x => x.SerialNumber == serialNumber).FirstOrDefault();
        if (airplane != null)
        {
            _context.Airplanes.Remove(airplane);
        }

        await _context.SaveChangesAsync();
    }
```

***
## Connecting to the Service Layer

In order to use the new service layer, we'll have to connect to it from the API. Run the following command to add the reference:

```
dotnet add .\AirDemo.Api\AirDemo.Api.csproj reference .\AirDemo.Service\AirDemo.Service.csproj
dotnet restore .\AirDemo.Api
```

Now we can swap out the context that was previously in the `AirplanesController` for the new service we created and call the corresponding methods like this:

*AirplanesController.cs*
```c#
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using AirDemo.Service;
using AirDemo.Service.Models;
using Microsoft.AspNetCore.Mvc;

namespace AirDemo.Api.Controllers
{
    [Route("api/[controller]")]
    public class AirplanesController : Controller
    {
        private readonly IAirplaneService _service;

        public AirplanesController(IAirplaneService service)
        {
            _service = service;
        }

        // GET api/airplanes
        [HttpGet]
        public async Task<IEnumerable<AirplaneResponse>> Get()
        {
            return await _service.GetAirplanes();
        }

        // GET api/airplanes/12345
        [HttpGet("{sn}")]
        public async Task<AirplaneResponse> Get(string sn)
        {
            return await _service.GetAirplane(sn);
        }

        // POST api/airplanes
        [HttpPost]
        public async Task Post([FromBody]AirplaneAddRequest request)
        {
            await _service.RegisterNewAirplane(request);
        }

        // PUT api/airplanes/12345/fly
        [HttpPut("{sn}/fly")]
        public async Task Fly(string sn, [FromBody]AirplaneFlyRequest request)
        {
            await _service.FlyAirplane(sn, request);
        }

        // PUT api/airplanes/12345/land
        [HttpPut("{sn}/land")]
        public async Task Land(string sn, [FromBody]AirplaneLandRequest request)
        {
            await _service.LandAirplane(sn, request);
        }

        // DELETE api/airplanes/12345
        [HttpDelete("{sn}")]
        public async Task Delete(string sn)
        {
            await _service.DeleteAirplane(sn);
        }
    }
}
```

Now we should be able to run our code and test it, right? Nope, we're not quite done. Remember we added a new interface that needs to be injected into our Controller. Therefore, we need to register it with the IoC (I'll put it right above the previously added AutoMapper code):

```c#
services.AddScoped<IAirplaneService, AirplaneService>();
```

***
:warning: *Warning*

If you were to instead add the service using `AddSingleton`, you would likely cause the DbContext to no longer be scoped to a particular request. Going back to the example of creating the service using plain Dependency Injection, the DbContext is passed into the constructor when the service is created, and it uses that instance of the context for the life of the service itself. Therefore, if you were to cause the service to live on between multiple requests, that same context would also be shared between those requests.

Luckily, .NET Core appears to throw an exception indicating that this is not possible, but it is still good to understand why it is not.
***

**Now** we should be able to run the API and test out our changes:

```
cd AirDemo.Api
dotnet run
```

Again, by going through the requests, you should be able to add, fly, land, and delete the airplane. After deleting it, you will notice that the `Get Single Airplane` request has again returned a status of `204 No Content`.

***
## Action Results

To correctly return the status code of `404` for an item not found, modify the `Get` method that returns a single result to return an `IActionResult` and check for whether a result was retrieved, returning the appropriate status, like so:

```c#
    [HttpGet("{sn}")]
    public async Task<IActionResult> Get(string sn)
    {
        var plane = await _service.GetAirplane(sn);
        if (plane == null)
        {
            return NotFound();
        }
        else
        {
            return Ok(plane);
        }
    }
```

There are other ways to use the `IActionResult` to provide a more fully-functional API. For example, on the `POST` method, you may return a status of `201 Created` when the item is successfully created or a `400 Bad Request` if it is not, indicating that something went amiss. The `Created` response might also include a `Location` header indicating where you might retrieve the created resource (1st parameter), as well as a request body containing the created item (2nd parameter).

```c#
    [HttpPost]
    public async Task<IActionResult> Post([FromBody]AirplaneAddRequest request)
    {
        await _service.RegisterNewAirplane(request);
        var plane = await _service.GetAirplane(request.SerialNumber);
        if (plane == null)
        {
            return BadRequest();
        }
        else
        {
            return Created($"api/airplanes/{request.SerialNumber}", plane);
        }
    }
```

***
:bulb: *Tip*

Using Postman, try out the modified `POST` method and see the resulting body being returned. To see the `Location` header, click "Headers". You should see the exact URL (minus the domain root) that is needed to retrieve the resource again using a `GET` request. This is mostly useful when you are generating an ID on the backend, but is helpful in any regard and follows proper RESTful patterns.

***

Also mentioned previously was the validation of the `[Required]` data annotations. For validating those annotations, add the following to the beginning of all of your `POST` and `PUT` methods:

```c#
    if (!this.ModelState.IsValid)
    {
        return BadRequest(this.ModelState);
    }
```

Now use Postman to try out the request, leaving out a field, to see what happens when this validation is triggered.

While I won't go into all the ways that `IActionResult` may be used, you can look at the finished code by checking out the next part of the series:

```
git checkout -f blog/part3
```

***
## Wrap Up

We are well on our way to providing a fully functional REST API for tracking airplanes and their locations! What we're lacking now is some basic validation, and a way to return any messages generated by the validation back to the client. In the next post, I'll demonstrate two different ways to track success/failure and messages: a `Result DTO` and a `Message Provider`.
