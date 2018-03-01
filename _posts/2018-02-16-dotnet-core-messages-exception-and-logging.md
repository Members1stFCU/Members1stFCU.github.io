---
title: .NET Core Messages, Exception, and Logging
author: Evan Kaiser
tags:
  - .NET Core
  - REST API
  - Programming Guides
  - Message Provider
  - Exceptions
  - Logging
---

Previously, I went over adding validation to your code, along with a Result class, containing a quick and easy way to pass back information to the client. However, at the end, I also address some things that could be improved upon, such as:

* Better support for multiple messages
* Other types of message, such as Success, Info, and Warning
* Extending the implementation of message handling, such as including logging or using a resource file to address language support

The solution to all of these things is the `IMessageProvider` described in this post.

***
This is an ongoing series on .NET Core.

1. [Setting up .NET Core](../setting-up-dotnet-core)
2. [.NET Core REST API in About 30 Minutes](../dotnet-core-rest-api-in-about-30-minutes)
3. [.NET Core Request/Response](../dotnet-core-request-response)
4. [Validation in .NET Core REST API](../validation-in-dotnet-core-rest-api)
5. .NET Core Messages, Exception, and Logging

## The Code

You may retrieve the code from [https://github.com/members1stfcu/blog-dotnetcore-airdemo](https://github.com/members1stfcu/blog-dotnetcore-airdemo) in the state we left it at the end of the previous post, by using the following `git` commands (skip `git clone` if you've already grabbed it once):

```
git clone https://github.com/members1stfcu/blog-dotnetcore-airdemo.git
cd blog-dotnetcore-airdemo
git checkout -f blog/part4
```

### Postman

You may also import the postman collection for this demo (in the state it should be for this post) by clicking the `Import` button and selecting the `AirDemo.postman_collection.json` file at the root of the project.

***
## The MessageProvider

We'll start off by creating a message provider in the `My.Feed` project that was added in the last post. This provider has an interface that can be implemented in a variety of different ways, allowing for flexibility depending on your apps specific needs, but also provides a consistent response back to the client in the form of the `Messages` collection.

*My.Feed/Providers/Messages/IMessageProvider.cs*
```c#
using System;
using System.Collections.Generic;

namespace My.Feed.Providers.Messages
{
    public interface IMessageProvider
    {
        ICollection<Message> Messages { get; }
        bool HasError { get; }
        void AddMessage(Message message);
        void AddException(Exception exception);
        void Clear();
    }

    public class Message
    {
        public Message(MessageType type, string text)
        {
            this.Type = type;
            this.Text = text;
        }
        public MessageType Type { get; }
        public string Text { get; }
        public string TypeDescription
        {
            get
            {
                return this.Type.ToString();
            }
        }
    }

    public enum MessageType
    {
        Error,
        Warning,
        Info,
        Success
    }
}
```

This interface gives us a starting point to expand upon furhter while providing the basic implementation of what we need to track for the message: The text of the message and the type (i.e. the severity, indicating whether it is an error or not).

Let's provide a basic implementation to start, which will just collect the messages that occur throughout the request in order to pass back all that information to the client.

*My.Feed/Providers/Messages/MessageProvider.cs*
```c#
using System;
using System.Collections.Generic;
using System.Linq;

namespace My.Feed.Providers.Messages
{
    public class MessageProvider : IMessageProvider
    {
        private readonly List<Message> _messages;

        public MessageProvider()
        {
            _messages = new List<Message>();
        }

        public ICollection<Message> Messages => _messages;

        public bool HasError => Messages.Any(x => x.Type == MessageType.Error);

        public void AddException(Exception exception)
        {
            _messages.Add(new Message(MessageType.Error, exception.Message));
        }

        public void AddMessage(Message message)
        {
            _messages.Add(message);
        }

        public void Clear()
        {
            _messages.Clear();
        }
    }
}
```

Now let's modify our code to use the Message Provider and pass back the messages in the resopnse.

*Airplane.cs* - Pass in the `IMessageProvider` to the public action method and the private validation method and add the error to it as well as the result.
```c#
using System;
using System.Linq;
using My.Feed.Providers.Messages;
using My.Feed.Services;

namespace AirDemo.Domain
{
    public class Airplane
    {
        private Airplane()
        {
            this.AirplaneId = Guid.NewGuid();
        }

        public static Result RegisterNewAirplane(AirplaneContext context, IMessageProvider messageProvider, string modelNumber, string serialNumber, int seatCount, decimal weightInKilos)
        {
            var plane = new Airplane
            {
                ModelNumber = modelNumber,
                SerialNumber = serialNumber,
                SeatCount = seatCount,
                WeightInKilos = weightInKilos
            };

            var result = plane.IsDuplicate(context, messageProvider);
            if (result)
            {
                context.Airplanes.Add(plane);
            }

            return result;
        }
        
        // Existing additional properties and methods

        private Result IsDuplicate(AirplaneContext context, IMessageProvider messageProvider)
        {
            if (context.Airplanes.Any(x => x.AirplaneId != this.AirplaneId && x.SerialNumber == this.SerialNumber))
            {
                var error = $"The Serial # {this.SerialNumber} is already taken.";
                messageProvider.AddMessage(new Message(MessageType.Error, error));
                return new FailResult(error);
            }
            
            return new SuccessResult();
        }
    }
}
```

*AirplaneService.cs* - Inject the `IMessageProvider` into the service and then pass it into the `RegisterNewAirplane` call. Let's also add a success messages to all the methods that can be displayed to the client upon successful modifications.
```c#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using AirDemo.Domain;
using AirDemo.Service.Models;
using AutoMapper;
using AutoMapper.QueryableExtensions;
using My.Feed.Providers.Messages;
using My.Feed.Services;

namespace AirDemo.Service
{
    public class AirplaneService : IAirplaneService
    {
        private readonly AirplaneContext _context;
        private readonly IMessageProvider _messageProvider;
        private readonly MapperConfiguration _map;

        public AirplaneService(
            AirplaneContext context,
            IMessageProvider messageProvider,
            MapperConfiguration map)
        {
            _context = context;
            _messageProvider = messageProvider;
            _map = map;
        }

        public async Task<IEnumerable<AirplaneResponse>> GetAirplanes()
        {
            return await _context.Airplanes.ProjectTo<AirplaneResponse>(_map).ToAsyncEnumerable().ToList();
        }

        public async Task<AirplaneResponse> GetAirplane(string serialNumber)
        {
            return await _context.Airplanes
                .Where(x => x.SerialNumber == serialNumber)
                .ProjectTo<AirplaneResponse>(_map)
                .ToAsyncEnumerable()                
                .SingleOrDefault();
        }

        public async Task<Result> RegisterNewAirplane(AirplaneAddRequest addRequest)
        {
            var result = Airplane.RegisterNewAirplane(_context, _messageProvider, addRequest.ModelNumber, addRequest.SerialNumber, addRequest.SeatCount ?? 0, addRequest.WeightInKilos ?? 0);

            // Check if action was successful
            if (result)
            {
                await _context.SaveChangesAsync();
                var updatedPlane = await this.GetAirplane(addRequest.SerialNumber);

                _messageProvider.AddMessage(new Message(MessageType.Success, $"Airplane with Serial # {addRequest.SerialNumber} registered successfully."));

                // Pass back a DataResult with the inner Data being the AirplaneResponse queried above
                return SuccessResult.WithData(updatedPlane);
            }

            return result;
        }

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
                _messageProvider.AddMessage(new Message(MessageType.Success, $"Airplane with Serial # {serialNumber} began flight for {flyRequest.EstimatedTripTime} hours."));

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
                _messageProvider.AddMessage(new Message(MessageType.Success, $"Airplane with Serial # {serialNumber} landed at {landRequest.AirportCode}."));

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
            _messageProvider.AddMessage(new Message(MessageType.Success, $"Airplane with Serial # {serialNumber} successfully deleted."));
            await _context.SaveChangesAsync();
            return new SuccessResult();
        }
    }
}
```

*AirplanesController.cs*
```c#
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using AirDemo.Service;
using AirDemo.Service.Models;
using Microsoft.AspNetCore.Mvc;
using My.Feed.Providers.Messages;
using My.Feed.Services;

namespace AirDemo.Api.Controllers
{
    [Route("api/[controller]")]
    public class AirplanesController : Controller
    {
        private readonly IAirplaneService _service;
        private readonly IMessageProvider _messageProvider;

        public AirplanesController(
            IAirplaneService service,
            IMessageProvider messageProvider)
        {
            _service = service;
            _messageProvider = messageProvider;
        }
        
        // Existing additional methods

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
                return Created($"api/airplanes/{request.SerialNumber}", new
                {
                    Data = (result as DataResult<AirplaneResponse>)?.Data,
                    Messages = _messageProvider.Messages
                });
            }
            else
            {
                // There was some reason the resource does not exist in the database, so something was wrong with the request, return 400
                return BadRequest(_messageProvider.Messages);
            }
        }
        
        // Existing additional methods
    }
}
```

***
:bulb: *Tip*

Note how I was able to simply create a new anonymous type in order to continue passing back the `Data` (the `AirplaneResponse` of the registered plane) along with the `Messages` collection. In case of a failure, it simply passes back the collection of validation errors that were added to the message provider (which is currently just 1).

***

*Startup.cs* - Finally, don't forget to add the `IMessageProvider` to the IoC (scoped, of course, so we don't keep collecting all messages from all requests!)
```c#
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<AirplaneContext>(opt => opt.UseInMemoryDatabase("HelloWorld"));
    services.AddScoped<IAirplaneService, AirplaneService>();
    services.AddScoped<IMessageProvider, MessageProvider>();
    
    // Additional services configuration
}
```

Try running the app again and do the test to register a new airplane and then attempt it a 2nd time. You should receive the appropriate `Data` and the success message in `Messages` the 1st time, and the validation error the 2nd time.

***
## Exceptions and Logging

One more thing we might want to address is what to do when an exception occurs in the API. Let's first add some code to manufacture an exception just to see what we get as the code currently stands.

*AirplaneService.cs* - Add the thrown exception at the beginning of the method.
```c#
public async Task<Result> RegisterNewAirplane(AirplaneAddRequest addRequest)
{
    //TODO: Remove after testing exceptions
    throw new Exception("Test Exception");

    //...Rest of code
```

Run the API and see what you get when you try to initiate the Add request in Postman. We get an ugly HTML response with the details of the exception, which you can at least preview in Postman by pressing the "Preview" button. This is fine for initial development of the API, but once we hook on a client consumer, we're going to expect every response to be delivered as `JSON`.

The easiest way to catch all exceptions and respond with `JSON` is to use middleware. Middleware is a term used by various implementations of REST API infrastructure to indicate a piece of code that gets executed in the course of every request. The nice thing about middleware in Core is the ability to inject anything from the IoC into the method. Note that scoped providers, like the `IMessageProvider`, should be injected into the `Invoke` method in order to retrieve the instance used in the request, whereas singleton providers (such as the Core `ILogger`) could be injected into the `constructor` or `Invoke`.

Let's add some middleware to our common nuget feed library and then use it within the app. First we have to install some dependencies:

```
cd My.Feed
dotnet add package Microsoft.AspNetCore.Http
dotnet add package Microsoft.AspNetCore.Hosting
dotnet add package Microsoft.Extensions.Logging
dotnet add package Newtonsoft.Json
```

*My.Feed/Middleware/ExceptionHandlingMiddleware.cs*
( Some of the code for this was originally from: https://www.devtrends.co.uk/blog/handling-errors-in-asp.net-core-web-api )
```c#
using System;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using My.Feed.Providers.Messages;
using Newtonsoft.Json;

namespace My.Feed.Middleware
{
    public class ExceptionHandlingMiddleware
    {
        private readonly RequestDelegate _next;
        private readonly ILogger<ExceptionHandlingMiddleware> _logger;
        private readonly IHostingEnvironment _env;
        
        public ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger, IHostingEnvironment env)
        {
            _next = next;
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
            _env = env ?? throw new ArgumentNullException(nameof(env));
        }

        public async Task Invoke(HttpContext context, IMessageProvider messageProvider)
        {
            try
            {
                await _next.Invoke(context);
            }
            catch (Exception ex)
            {
                if (_env.IsProduction())
                {
                    messageProvider.AddMessage(new Message(MessageType.Error, "Internal Server Error"));
                }
                else
                {
                    messageProvider.AddException(ex);
                }

                _logger.LogCritical(ex, ex.Message);

                if (!context.Response.HasStarted)
                {
                    context.Response.StatusCode = StatusCodes.Status500InternalServerError;
                    context.Response.ContentType = "application/json";
                    var json = JsonConvert.SerializeObject(messageProvider.Messages);
                    await context.Response.WriteAsync(json);
                }
            }
        }
    }   
}
```

*AirDemo.Api/Startup.cs* - Add the following line before `app.UseMvc()`
```c#
app.UseMiddleware<ExceptionHandlingMiddleware>();
```

Now let's run the API again and try out the add action. You should see the following JSON returned indicating the error and the corresponding message:
```json
[
    {
        "Type": 0,
        "Text": "Test Exception",
        "TypeDescription": "Error"
    }
]
```

Note how we use the `IHostingEnvironment` provider to tell us whether we're in Production or not. This allows us to show the full exception message to help in debugging exceptions in non-Production environments, but in Production, we swallow the error and return the generic "Internal Server Error" instead. Either way, we are logging the exception so that we may investigate the Stack Trace in any of the environments in the logs (as you can see has shown in the terminal where you have run the API).

For logging, I recommend extending the existing logger using [Serilog](https://github.com/serilog/serilog/wiki/Getting-Started) which provides ["Sinks"](https://github.com/serilog/serilog/wiki/Provided-Sinks) for various ways of logging, including Rolling File, which is useful for quickly researching exceptions or Slack to have your team notified immediately of a production exception!

Before we wrap up, let's not forget to remove the exception that was temporarily added at the beginning of `AirplaneService::RegisterNewAirplane`.

***
## Wrap Up

Now we have a way to properly validate the state of the Domain and send errors and other messages back to the client!

Speaking of the client, we're probably going to want to establish a way to define who is able to access this API (i.e. Authentication) and then track who made changes to the planes and when that occurred (i.e. Audit Trail). The first thing we'll need to do is provide a way to easily establish who is logged in and what the current date and time is (for the purpose of easing Unit Tests), so the next post will go over adding an `IUserProvider`, `IDateProvider`, and `IGenericContext`.