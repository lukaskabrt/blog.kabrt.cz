Title: Serverless ASP.NET Core MVC
Published: 11/11/2018
Tags: [ASP.NET Core, Azure Functions]
---

I am working on a small side-project, that uses ASP.NET Core API. This application will be used only occasionally so it would really benefit from using the serverless approach. Unfortunately Azure Functions doesn't provide support for running ASP.NET applications, instead, you can use `HttpTrigger`. [Azure Function documentation](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-http-webhook) says:

> The HTTP trigger lets you invoke a function with an HTTP request. You can use an HTTP trigger to build serverless APIs and respond to webhooks.

Nice, but I want something different. I would like to take an existing ASP NET Core API application and run it in the serverless matter. Quick Google search showed just few results:
* [MSDN - Running an ASP.NET Core Pipeline using Azure Functions](https://social.msdn.microsoft.com/Forums/en-US/c3e9f882-541f-48a2-a856-956ad3383f67/running-an-aspnet-core-pipeline-using-azure-functions?forum=AzureFunctions)
* [Stack Overflow - Running an ASP.NET Core Pipeline using Azure Functions](https://stackoverflow.com/questions/48446132/running-an-asp-net-core-pipeline-using-azure-functions)
* [ASP.NET Core MVC/Web API support](https://feedback.azure.com/forums/355860-azure-functions/suggestions/33134257-asp-net-core-mvc-web-api-support)

It looks like somebody managed to accomplish it, several people think it's a bad idea and at least 30 people think Azure Functions should support ASP.NET Core API applications ... but no clear example of how to do it.

Basically, I had 3 options:
* rewrite the application to use `HttpTrigger`
* use [AWS Lambda](https://aws.amazon.com/blogs/developer/serverless-asp-net-core-2-0-applications/), that natively supports ASP.NET Core applications. (It's quite sad ... Azure doesn't allow you to run ASP.NET Core, but ASW does)
* find a way to host the ASP.NET Core application in Azure Function

The first option didn't look very attractive. I also didn't want to move to AWS, because I am using other Azure resources, so I decided to give the third option a try.

## HttpTrigger and Azure Functions

Azure Functions is built on top of the WebJobs SDK. Thankfully both projects are opensource ([WebJobs SDK](https://github.com/Azure/azure-webjobs-sdk) [Azure Functions Host](https://github.com/Azure/azure-functions-host)), so we can have a look at what is going on inside the Azure Function. `HttpTrigger` is defined in yet another library [WebJobs SDK Extensions](https://github.com/Azure/azure-webjobs-sdk-extensions).

## Implementation

After some browsing thru the code, the most straightforward approach seems to reuse the `HttpTrigger` as a proxy between Azure Functions Host and the ASP.NET Core application.

### Routing

The easiest part as it is supported by default. We want our proxy function to trigger for all HTTP verbs and for all URLs.

```csharp
[FunctionName("Proxy")]
public static async Task<IActionResult> Run(
    [HttpTrigger AuthorizationLevel.Anonymous, Route = "{*all}")]HttpRequest req,
    ILogger log) {
    ...
}
```

We don't define any HTTP verb constraints in the `HttpTrigger` and use the 'catch-all' pattern for the `Route` parameter.

Also, we want to remove the default `api` prefix for URL. The prefix can be configured in the `host.json` file.

```json
{
  "version": "2.0",
  "extensions": {
    "http": {
      "routePrefix": ""
    }
  }
}
```

### ASP.NET Core application hosting

In order to use ASP.NET Core application, we have to be able to instantiate it inside our function. I took inspiration from the `TestServer` [implementation](https://github.com/aspnet/Hosting/tree/master/src/Microsoft.AspNetCore.TestHost) in the ASP.NET Core repository and created a minimal implementation of the `IServer` [interface](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.hosting.server.iserver?view=aspnetcore-2.1).

```csharp
public class InternalServer : IServer {
    private bool _disposed = false;
    private IWebHost _host;

    public IHttpApplication<HostingApplication.Context> Application;

    public InternalServer(IWebHostBuilder builder)
        : this(builder, new FeatureCollection()) {
    }

    public InternalServer(IWebHostBuilder builder, IFeatureCollection featureCollection) {
        var host = builder.UseServer(this).Build();
        host.StartAsync().GetAwaiter().GetResult();
        _host = host;
    }

    public IFeatureCollection Features { get; }

    public void Dispose() {
        if (!_disposed) {
            _disposed = true;
            _host.Dispose();
        }
    }

    public Task StartAsync<TContext>(IHttpApplication<TContext> application, CancellationToken cancellationToken) {
        this.Application = (IHttpApplication<HostingApplication.Context>)application;

        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken cancellationToken) {
        return Task.CompletedTask;
    }
}

```

We use the single instance of the `InternalServer` that processes all incoming requests. The instance gets initialized with `Api.Startup` class from the ASP.NET Core API project.

```csharp
public static InternalServer Instance { get; set; }

static InternalServer() {
    var builder = new WebHostBuilder()
        .UseStartup<Api.Startup>();
    Instance = new InternalServer(builder);
}
```

It is a very similar code to the standard ASP.NET Core project initialization.

```csharp
public class Program {
    public static void Main(string[] args) {
        BuildWebHost(args).Run();
    }
    
    public static IWebHost BuildWebHost(string[] args) =>
        WebHost.CreateDefaultBuilder(args)
            .UseStartup<Startup>()
            .Build();
    }
```

### Proxy 

This was the most challenging part of the implementation. The documentation for HttpBinding says: 

> To send an HTTP response, use the language-standard response patterns. In C# or C# script, make the function return type `HttpResponseMessage` or `Task<HttpResponseMessage>`. In C#, a return value attribute isn't required.

If we look at the source code of the binding 

```csharp
IActionResult actionResult = result as IActionResult;
if (actionResult == null) {
    if (result is Stream) {
        FunctionBinding.ConvertStreamToValue((Stream)result, DataType.String, ref result);
        actionResult = CreateResult(request, result);
    } else if (result is JObject) {
        actionResult = CreateResult(request, result);
    } else {
        var objectResult = new ObjectResult(result);

        if (result is System.Net.Http.HttpResponseMessage) {
            objectResult.Formatters.Add(new HttpResponseMessageOutputFormatter());
        }

        actionResult = objectResult;
    }
}
```

we see the function can return an `IActionResult`, a `Stream` or a `JObject` with a serialized response or a `HttpResponseMessage`. None of these options fits seamlessly to our use case, because the ASP.NET application we are hosting in the `InternalServer` exposes only the `ProcessRequestAsync` method.

```csharp
Task ProcessRequestAsync(HostingApplication.Context context)
```

This method runs ASP.NET pipeline against the `context.HttpContext` and writes output to the `context.HttpContext.Response`. The `IHttpApplication` uses an `IActionResult` internally but it gets executed and its result is written to the `context.HttpContext.Response` when we call the `ProcessRequestAsync` method.

I tried to write a `CopyResponseActionResult`, that executes the request in the ASP.NET application, inspects the application response and copies it to the function response. This approach worked for simple status responses but didn't work for responses with a body because `HttpContext.Response.Stream` can't be read. But what if we don't execute the ASP.NET application directly in the triggered function ... introducing `ProxyActionResult`.

```csharp
public class ProxyActionResult : IActionResult {
    public async Task ExecuteResultAsync(ActionContext context) {
        await InternalServer.Instance.Application.ProcessRequestAsync(new HostingApplication.Context() { HttpContext = context.HttpContext });
    }
}
```

When Azure Functions Host executes the `ProxyActionResult`, it passes the `context` parameter to the function with the `HttpContext.Request` populated from the actual HTTP request and uses `HttpContext.Response` to send HTTP response to the client. We take `HttpContext` from the Azure Functions Host and pass it in the `HostingApplicationContext` to the ASP.NET application. This way, the application reads directly from the functions `HttpContext.Request` and writes response directly to the functions `HttpContext.Response`.

In the function, we simply return a new instance of the `ProxyActionResult` class.

```csharp
[FunctionName("Proxy")]
public static async Task<IActionResult> Run([HttpTrigger(AuthorizationLevel.Anonymous, Route = "{*all}")]HttpRequest req, ILogger log) {
    return new ProxyActionResult();
}
```

It might seem odd to return the action result without any parameters, but as described higher, the current request context is passed to the `ProxyActionResult` by the Azure Functions Host.

<figure class="figure w-100 text-center">
    <a href="/content/2018-11-serverless-asp-net/01-running.png" data-toggle="lightbox">
        <img src="/content/2018-11-serverless-asp-net/01-running_t.png" class="figure-img img-fluid" alt="Running ASP.NET on Azure Function">
    </a>  
    <figcaption class="figure-caption text-center">An ASP.NET Core API running on the Azure Functions.</figcaption>
</figure>

There might be some hidden caveats, especially in cases where ASP.NET Core application uses some properties from the `HttpContext` that aren't initialized by the Azure Functions Host. Another small disadvantage is startup time of the application, every now and then, when the platform spins up a new instance of our function application a user can experience few seconds long delay, because the internal application is initializing. According to my testing, the application stays active for 10-15 minutes after the last request, so the whole idea seems viable ... at least for my use case.

The complete sample project can be dowmloaded from the [GitHub repository](https://github.com/lukaskabrt/blog-MvcOnAzureFunctions).