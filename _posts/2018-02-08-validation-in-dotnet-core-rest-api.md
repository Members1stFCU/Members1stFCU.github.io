---
title: Validation in .NET Core REST API
author: Evan Kaiser
tags:
  - .NET Core
  - REST API
  - Programming Guides
  - Validation Handling
  - Result DTO
---

Now that we have a functional API that can be used for tracking airplanes at different airports and their current flight routes, we will want to start putting in some basic validation, such as avoiding duplicate serial numbers or a negative trip time. This validation will need to be tracked and reported back to the client in some fashion as well.

In this post, I'll go over two different patterns to collect and report back such validation errors: a Result DTO and a Message Provider. They both have their own pros and cons, but could even be used together for different purposes.

***
This is an ongoing series on .NET Core.

1. [Setting up .NET Core](../setting-up-dotnet-core)
2. [.NET Core REST API in About 30 Minutes](../dotnet-core-rest-api-in-about-30-minutes)
3. [.NET Core Request/Response](../dotnet-core-request-response)
4. Validation in .NET Core REST API
5. [.NET Core Messages, Exception, and Logging](../dotnet-core-messages-exception-and-logging)

## The Code

You may retrieve the code from [https://github.com/members1stfcu/blog-dotnetcore-airdemo](https://github.com/members1stfcu/blog-dotnetcore-airdemo) in the state we left it at the end of the previous post, by using the following `git` commands (skip `git clone` if you've already grabbed it once):

```
git clone https://github.com/members1stfcu/blog-dotnetcore-airdemo.git
cd blog-dotnetcore-airdemo
git checkout -f blog/part3
```

### Postman

You may also import the postman collection for this demo (in the state it should be for this post) by clicking the `Import` button and selecting the `AirDemo.postman_collection.json` file at the root of the project.

***
## Adding the Validation

The first thing we need to do is to decide on a bit of validation to add and then decide where to put the code to check for it. One simple bit of validation we know we are going to need is to check for a duplicate Serial Number when registering a new `Airplane`. Now the question is: where to put the validation? One obvious place to put it is at the beginning of the `AirplaneService::RegisterNewAirplane` method. This is typically where such validation would've gone in traditionally layered applications with a "Business Layer" abstraction overtop of a "Data Layer". However, following more modern patterns, the [Onion Architecture](http://jeffreypalermo.com/blog/the-onion-architecture-part-1/) and [Domain Driven Design](https://en.wikipedia.org/wiki/Domain-driven_design) teach us to treat the Domain (i.e. the Airplane entity) as an object in charge of its own state and behavior, and the Service as the layer that simply provides a mechanism for interacting with those Domain objects. Therefore, if we let the Airplane handle its own state, regarding whether or not it is allowed to be registered, all the Service needs to know is that it is trying to register the airplane and then, if that action succeeds, to initiate the persistence layer to save those changes.

So given that, where do we put the validation? Well the next obvious location might be the constructor of Airplane. However, it's typically not good practice to do a lot of processing in a constructor. I like to have a method name that clearly defines what is happening verbally, such as `RegisterNewAirplane`. But since we don't actually have an entity to work with yet, one thing we can do is put a static method on the entity that creates the object and hands it back to the service. This approach allows the entity to create itself (giving the routine access to all the private/protected properties and methods it needs to create itself, as well). Try modifying the object as follows to add this method:

*AirDemo.Domain/Airplane.cs*
```c#
using System;

namespace AirDemo.Domain
{
    public class Airplane
    {
        private Airplane()
        {
            this.AirplaneId = Guid.NewGuid();
        }

        public static Airplane RegisterNewAirplane(string modelNumber, string serialNumber, int seatCount, decimal weightInKilos)
        {
            return new Airplane
            {
                ModelNumber = modelNumber,
                SerialNumber = serialNumber,
                SeatCount = seatCount,
                WeightInKilos = weightInKilos
            };
        }
        
        // Existing additional properties and methods
    }
}
```

Note that we were still able to set all the properties with private setters, but outside of the instance of the class itself, because it was a static method contained within that class. I also moved the setting of `AirplaneId` to the private default constructor, since we always want to set the ID when the class is created, regardless of what other data is set.

Now, if we modify that method to pass in the context, we can both check for the existence of other planes with the given serial number, and even move the adding of the entity to the context to this method, which relieves the Service method from doing the work to decide whether the context should contain the entity.

*AirDemo.Domain/Airplane.cs*
```c#
using System;
using System.Linq;

namespace AirDemo.Domain
{
    public class Airplane
    {
        private Airplane()
        {
            this.AirplaneId = Guid.NewGuid();
        }

        public static Airplane RegisterNewAirplane(AirplaneContext context, string modelNumber, string serialNumber, int seatCount, decimal weightInKilos)
        {
            var plane = new Airplane
            {
                ModelNumber = modelNumber,
                SerialNumber = serialNumber,
                SeatCount = seatCount,
                WeightInKilos = weightInKilos
            };

            if (plane.IsDuplicate(context))
            {
                return null;
            }
            else
            {
                context.Airplanes.Add(plane);
                return plane;
            }
        }
        
        // Existing additional properties and methods

        private bool IsDuplicate(AirplaneContext context)
        {
            return context.Airplanes.Any(x => x.AirplaneId != this.AirplaneId && x.SerialNumber == this.SerialNumber);
        }
    }
}
```

For now, we'll return a `null` to indicate that the registration failed. Later on, we'll make this more robust by adding a validation message. Let's modify the service method to change how the registration is called.

*AirDemo.Service/AirplaneService.cs*
```c#
        public async Task RegisterNewAirplane(AirplaneAddRequest addRequest)
        {
            var airplane = Airplane.RegisterNewAirplane(_context, addRequest.ModelNumber, addRequest.SerialNumber, addRequest.SeatCount ?? 0, addRequest.WeightInKilos ?? 0);
            if (airplane != null)
            {
                await _context.SaveChangesAsync();
            }
        }
```

Go ahead and try this out by calling `dotnet run` while in the `AirDemo.Api` project, and adding an airplane using the `Add Airplane` endpoint in Postman. Then initiate the same POST a 2nd time. It looks like it succeeded. Why did that occur? If we go look at `AirplanesController::Post`, it may become clear. The call to `_service.RegisterNewAirplane(request)` is a "fire and forget" type of method that calls the service method but doesn't process any result. It then queries the Airplane by Serial Number, which finds the initially registered airplane. 

If you execute the `Get All Airplanes` endpoint, you will see that it did not indeed create a duplicate, and therefore the validation failed. However, it didn't report back that failure to the controller, thus giving a false indication of success. This highlights the necessity for some sort of `Result DTO` to manage what happened in a POST, PUT, or DELETE action.

***
## Result DTO

The following `Result DTO` addresses a few different needs that have been mentioned above and that might also come up as you're working on adding validation to your domain.

1. **It allows you to track success and failure.** The `Success` property contains whether or not the Result of performing the validation was a success or not. This may also be changed over the course of the validation by the `Reevaluate` method that takes in new information from additional validations. For example, the above validation might have succeeded (i.e. the Serial # was indeed different), but we had an additional validation - say to check for a Serial # to meet a certain regular expression, i.e. between 5 and 10 alphanumeric characters - that then failed, we would not want the `Success` property to remain true.
2. **It provides a way to log a text message indicating the reason for a failure using the `Error` property.** Additionally, if multiple error messages are added due to multiple validation failures, it aggregates them into a semi-colon separated list. For more granular control over this practice of collecting error messages, the Message Provider explained further on in this post might be more ideal, but this provides a basic mechanism for doing so.
3. **It provides a way to embed a particular object into the result to be retrieved by the caller.** The `DataResult` child class includes the `Data` property, which lets you store the result of some routine. This allows you to still return something back from a method, while at the same time leaving the possibility open for returning a failure with a corresponding message, without using the `ref` or `out` keywords that tend to make method signatures messy.

Before we add the Result DTO to our `AirDemo`, we'll want to add another class library. Since all 3 of the projects we have so far are specific to the application being built, we do not want to mix in the Result DTO, which is a type of common "plumbing" class that would be able to be reused in many different applications. Typically, you'd want to start building it into a common `nuget` feed that would be reusable and consumable by many projects, but for the purposes of this demo, we'll just make it a separate library called "My.Feed".

Add the common library and include it in all 3 projects:

```
dotnet new classlib -n My.Feed
dotnet add .\AirDemo.Api\AirDemo.Api.csproj reference .\My.Feed\My.Feed.csproj
dotnet add .\AirDemo.Domain\AirDemo.Domain.csproj reference .\My.Feed\My.Feed.csproj
dotnet add .\AirDemo.Service\AirDemo.Service.csproj reference .\My.Feed\My.Feed.csproj
```

Now, let's create a folder called "Services" under that new project and add the following Result DTO:

*My.Feed/Services/Result.cs*
```c#
using System;

namespace My.Feed.Services
{
    public class Result
    {
        public Result(bool success)
        {
            this.Success = success;
        }

        public Result(bool success, string error)
            : this(success)
        {
            this.Error = error;
        }

        public bool Success { get; private set; }

        public string Error { get; private set; }

        public static implicit operator bool(Result result)
        {
            return result.Success;
        }

        public void Reevaluate<T>(ref Result updated, T additionalConsideration)
            where T : Result
        {
            if (!additionalConsideration)
            {
                if (this)
                {
                    updated = additionalConsideration;
                    return;
                }
                else
                {
                    updated.Error = updated ? additionalConsideration.Error : string.Join("; ", updated.Error, additionalConsideration.Error);
                }
            }

            updated = this;
        }
    }

    public class FailResult : Result
    {
        public FailResult()
            : base(false)
        {
        }

        public FailResult(string error)
            : base(false, error)
        {
        }
    }

    public class SuccessResult : Result
    {
        public SuccessResult()
            : base(true)
        {
        }

        public static DataResult<T> WithData<T>(T data)
        {
            return new DataResult<T>(data);
        }
    }

    public class DataResult<T> : SuccessResult
    {
        public DataResult(T obj)
        {
            this.Data = obj;
        }

        public T Data { get; private set; }
    }
}
```

***
### Utilizing the Result DTO

Now that we have the Result to use, let's see how we can use it within our code. First, let's modify the `Airplane` class and the `AirplaneService::RegisterNewAirplane` and `AirplanesController::Post` methods to use it.

*Airplane.cs*
```c#
using System;
using System.Linq;
using My.Feed.Services;

namespace AirDemo.Domain
{
    public class Airplane
    {
        private Airplane()
        {
            this.AirplaneId = Guid.NewGuid();
        }

        public static Result RegisterNewAirplane(AirplaneContext context, string modelNumber, string serialNumber, int seatCount, decimal weightInKilos)
        {
            var plane = new Airplane
            {
                ModelNumber = modelNumber,
                SerialNumber = serialNumber,
                SeatCount = seatCount,
                WeightInKilos = weightInKilos
            };

            var result = plane.IsDuplicate(context);
            if (result)
            {
                context.Airplanes.Add(plane);
            }

            return result;
        }
        
        // Existing additional properties and methods

        private Result IsDuplicate(AirplaneContext context)
        {
            if (context.Airplanes.Any(x => x.AirplaneId != this.AirplaneId && x.SerialNumber == this.SerialNumber))
            {
                return new FailResult($"The Serial # {this.SerialNumber} is already taken.");
            }
            
            return new SuccessResult();
        }
    }
}
```

*IAirplaneService.cs* - Update the return value of method signature
```c#
Task<Result> RegisterNewAirplane(AirplaneAddRequest addRequest);
```

*AirplaneService.cs* - Update the return value of method signature and modify it to return the result
```c#
public async Task<Result> RegisterNewAirplane(AirplaneAddRequest addRequest)
{
    var result = Airplane.RegisterNewAirplane(_context, addRequest.ModelNumber, addRequest.SerialNumber, addRequest.SeatCount ?? 0, addRequest.WeightInKilos ?? 0);

    // Check if action was successful
    if (result)
    {
        await _context.SaveChangesAsync();
    }

    return result;
}
```

*AirplanesController.cs*
```c#
// POST api/airplanes
[HttpPost]
public async Task<IActionResult> Post([FromBody]AirplaneAddRequest request)
{
    if (!this.ModelState.IsValid)
    {
        return BadRequest(this.ModelState);
    }

    var result = await _service.RegisterNewAirplane(request);
    if (result)
    {
        // Return 201 with the Location header indicating how to retrieve the resource, and the state of the resource after it was created
        var plane = await _service.GetAirplane(request.SerialNumber);
        return Created($"api/airplanes/{request.SerialNumber}", plane);
    }
    else
    {
        // There was some reason the resource does not exist in the database, so something was wrong with the request, return 400
        return BadRequest(result.Error);
    }
}
```

Note that in the controller, we can now return the error message as part of the `400 BadRequest` response, since we have passed that validation error that was added in the Domain all the way back to the controller. We've also modified the method to check for a successful result before querying for the airplane. Now let's run the application with this modification and execute the same double `POST` call from Postman that we did previously.

Now, on the 2nd request, you should get the status `400 Bad Request` with the response body containing the message `The Serial # 12345 is already taken.` This is a much more useful response than thinking that the airplane was successfully added a 2nd time!

***
### Using DataResult to pass back the entity added/updated

One more thing you might want to do is to also pass back the entity or response model containing the thing that was added or updated with modified data. This is useful for situations where you might want to pass the response object back in the `Created` response without requiring the Controller to re-query the entity. For this particular example, we're simply going to move that responsibility to the Service, but at least in that case, the service standardizes that work for all callers (i.e. if the service method is also called from a job or different API).

Let's modify the Service method to return the `DataResult<T>`, which in this case will be a `DataResult<AirplaneResponse>`.

*AirplaneService.cs*
```c#
public async Task<Result> RegisterNewAirplane(AirplaneAddRequest addRequest)
{
    var result = Airplane.RegisterNewAirplane(_context, addRequest.ModelNumber, addRequest.SerialNumber, addRequest.SeatCount ?? 0, addRequest.WeightInKilos ?? 0);

    // Check if action was successful
    if (result)
    {
        await _context.SaveChangesAsync();
        var updatedPlane = await this.GetAirplane(addRequest.SerialNumber);

        // Pass back a DataResult with the inner Data being the AirplaneResponse queried above
        return SuccessResult.WithData(updatedPlane);
    }

    return result;
}
```

Note that we don't change the return type to a `Task<DataResult<AirplaneResponse>>`. If we tried that, we would not be able to pass back a `FailResult` when a failure occurs. Therefore, it's a good idea when processing the response to check whether the service actually delivered the expected result using either the `is` or `as` keyword, as seen below:

*AirplaneController.cs*
```c#
// POST api/airplanes
[HttpPost]
public async Task<IActionResult> Post([FromBody]AirplaneAddRequest request)
{
    if (!this.ModelState.IsValid)
    {
        return BadRequest(this.ModelState);
    }

    var result = await _service.RegisterNewAirplane(request);
    if (result)
    {
        // Return 201 with the Location header indicating how to retrieve the resource, and the state of the resource after it was created
        return Created($"api/airplanes/{request.SerialNumber}", (result as DataResult<AirplaneResponse>)?.Data);
    }
    else
    {
        // There was some reason the resource does not exist in the database, so something was wrong with the request, return 400
        return BadRequest(result.Error);
    }
}
```

***
:thought_balloon: *Thought*

A little more explanation might be needed on the code above. While it was boiled down to one line, the processing of the `Result` could be rewritten like this:

```c#
if (result)
{
    // Setup a variable to put the response in, which might be null if the Result is not of the expected type
    AirplaneResponse plane = null;

    // Check if the result is a DataResult with the Data being of type AirplaneResponse
    if (result is DataResult<AirplaneResponse>)
    {
        // If the result is the expected type, cast it to that type
        var planeResult = (DataResult<AirplaneResponse>)result;

        // Now get the inner Data property representing the data the result is holding
        plane = planeResult.Data;
    }
    return Created($"api/airplanes/{request.SerialNumber}", plane);
}
```

However, it's cleaner to boil that down to one line where the 2nd parameter of Created is passed: `(result as DataResult<AirplaneResponse>)?.Data`. Note that the `as` keyword is similar to `is`, except that instead of returning a Boolean indicating whether the type matches, it returns the casted object or `null` if the cast fails. The `?` after that prevents an Object Reference error if the cast does fail, therefore returning the default of `AirplaneResponse` (also `null`) when `.Data` is requested. So essentially, the response will still either be the expected object or `null`.

***

### Modify Fly, Land, and Delete to also use the Result DTO

*Airplane.cs* - For now, let's modify the return types, but simply return `SuccessResult`, since we don't have any validation... yet.
```c#
public virtual Result Fly(TimeSpan estimatedTripTime)
{
    this.CurrentAirportCode = null;
    this.LastTakeoffTime = DateTimeOffset.Now;
    this.EstimatedLandingTime = DateTimeOffset.Now.Add(estimatedTripTime);
    return new SuccessResult();
}

public virtual Result Land(string airportCode)
{
    this.CurrentAirportCode = airportCode;
    this.LastTakeoffTime = null;
    this.EstimatedLandingTime = null;
    return new SuccessResult();
}
```

*IAirplaneService.cs* - Update the return value of method signatures from `bool` to `Result`
```c#
Task<Result> FlyAirplane(string serialNumber, AirplaneFlyRequest flyRequest);
Task<Result> LandAirplane(string serialNumber, AirplaneLandRequest landRequest);
Task<Result> DeleteAirplane(string serialNumber);
```

*AirplaneService.cs* - Update the return value of method signatures and modify the methods accordingly
```c#
public async Task<Result> FlyAirplane(string serialNumber, AirplaneFlyRequest flyRequest)
{
    var airplane = _context.Airplanes.Where(x => x.SerialNumber == serialNumber).FirstOrDefault();
    if (airplane == null)
    {
        return new FailResult($"Airplane with Serial # {serialNumber} not found.");
    }

    var result = airplane.Fly(flyRequest.EstimatedTripTime ?? new TimeSpan(0, 0, 0));
    if (result)
    {
        await _context.SaveChangesAsync();
    }

    return result;
}

public async Task<Result> LandAirplane(string serialNumber, AirplaneLandRequest landRequest)
{
    var airplane = _context.Airplanes.Where(x => x.SerialNumber == serialNumber).FirstOrDefault();
    if (airplane == null)
    {
        return new FailResult($"Airplane with Serial # {serialNumber} not found.");
    }

    var result = airplane.Land(landRequest.AirportCode);
    if (result)
    {
        await _context.SaveChangesAsync();
    }

    return result;
}

public async Task<Result> DeleteAirplane(string serialNumber)
{
    var airplane = _context.Airplanes.Where(x => x.SerialNumber == serialNumber).FirstOrDefault();
    if (airplane == null)
    {
        return new FailResult($"Airplane with Serial # {serialNumber} not found.");
    }

    _context.Airplanes.Remove(airplane);
    await _context.SaveChangesAsync();
    return new SuccessResult();
}
```

***
:bulb: *Tip*

For now, we can leave the `AirplanesController` as is, since we are not returning a `DataResult` and the signatures were changed from `bool` to `Result`, which implicitly evaluates to `bool`.

***

## Wrap Up

The `Result DTO` is a quick and easy way to pass information back from a service method, but it is not comprehensive enough to address additional necessities that come along with developing an application, such as:

* Multiple messages
* Other types of message, such as Success, Info, and Warning
* Extending the implementation of message handling, such as including logging or using a resource file to address language support

These additional requirements speak to the need of a provider that might be injected into the application and could be implemented in a variety of different ways.

The next post will go over addressing this with an `IMessageProvider`, along with some additional notes on Exceptions and Logging.