# Running ML.NET models on scalable ASP.NET Core WebAPIs or web apps

This posts explains how to optimize your code when running an ML.NET model on an ASP.NET Core WebAPI service, but the code is very similar when running it on an ASP.NET Core MVC or Razor web app.

## Basic background on scalable and multithreaded ASP.NET Core services and apps

ASP.NET Core services and apps are multithreaded applications so they can serve many HTTP requests at the same time.

.NET Core provides a managed thread pool that is managed by the system. As a developer, you don’t need to deal with the thread management. 

As shown in the simplified image below, and ASP.NET Core WebAPI or web app accepts many Http requests which will be handled by that thread pool.

![alt text](images/Simplified-threading-in-ASPNETCore-app.png "Multi-threads in ASP.NET Core apps")

The specifics of the architecture are not exactly the same if you deploy your application into *IIS* or selfhosted by using *Kestrel*, but those differences are not important in this case.

The bottom line here is that when writing code for a multithreaded application such as ASP.NET Core services or apps **your code and objects used in your code need to be *thread-safe* if the same object is going to be shared across multiple threads while updating data in-memory**. The reason is because if you update data within the same shared object from multiple threads, those multiple threads can access to the same address space at the same time. So, they can write to the exact same memory location at the same time causing data corruption and application mal-function.

A code is called *thread-safe* if it is being called from multiple threads concurrently without the breaking of functionalities. 

### This is usually not a problem in regular ASP.NET Core apps

Usually, most of the code and objects you use in an ASP.NET Core WebAPI or app, for instance, objects you instanciate in a controller, are not shared across Http requests because the most common pattern is to use objects you create for each Http requests scope, either with `new` or with [transient lifetime](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-2.2#service-lifetimes) when using dependency injection in .NET Core, as shown in the following image. Not a problem here:

![alt text](images/Transient-Objects-in-ASPNETCore-app.png "Multi-threads in ASP.NET Core apps")

The problem can happen if you use shared objects across threads, for example by using [static member variables](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/static-classes-and-static-class-members#static-members) or [singleton lifetime](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-2.2#service-lifetimes) objects in Dependency Injection, as explained later on in this post.

# C# code needed to run an ML.NET model and predict

As you can check out on may ML.NET getting started samples at the [ML.NET Samples GitHub repo](https://github.com/dotnet/machinelearning-samples), the basic code you need to load an already trained and serialized ML.NET model and predict with it (usually called 'ML model scoring code'), is the following:

```cs
// (*Expensive*) Load ML model from serialized .zip file 
ITransformer mlModel;
using (var stream = new FileStream(modelFilePath, FileMode.Open, FileAccess.Read, FileShare.Read))
{
    mlModel = mlContext.Model.Load(stream);
}

// Create sample data to do a single prediction with it 
SampleObservation sampleData = CreateMySingleDataSample();

// (*Expensive*) Create Prediction Engine
var predictionEngine = mlModel.CreatePredictionEngine<SampleObservation, SamplePrediction>(mlContext);

// Try a single prediction
SamplePrediction predictionResult = predictionEngine.Predict(sampleData);
```

This code looks very atraightforward and simple to use. If you just copy that code and run it on any application (console, desktop, web, etc.) it'll work okay. 

However, the lines of code marked with `(*Expensive*)` in the comments are object instantiations that are significantly 'expensive', meaning that it takes significant time to execute such as 1 or 2 seconds per each call and that can vary depending on the size of the ML model.

## You need to improve your ML.NET model scoring code for scalable apps

As you would initially think, improving that execution for an application that will run multiple predictions could be as simple as 'caching' the ML model (ITransformer) object and the Prediction Engine object, so those objects are instantiaoned just once and shared across the upcoming requests, right?

Well, that is partially true, but there are important caveats here and this is the reason why I created this post, precisely. ;)


# Sharing the ML model (ITransformer) across HTTP requests in ASP.NET Core 

Therefore, the first optimization you could do would be to cache in memory the ML model object that you loaded from the .zip file, so it can be re-used across many Http requests.

Since the **ML model (ITransformer) object is *thread-safe***, this is not an issue and can be done very easily.

The recommended way to share the ITransfomer object across Http requests is to register it as [singleton lifetime](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-2.2#service-lifetimes) object in your IoC container for Dependency Injection usage, as shown in the code below:

```cs
Startup.cs

public void ConfigureServices(IServiceCollection services)
{
    //..Other code..

    // Register Types in IoC container for DI

    //MLContext created as singleton for the whole ASP.NET Core app
    services.AddSingleton<MLContext>();

    //ML Model (ITransformed) created as singleton for the whole ASP.NET Core app. Loads from .zip file here.
    services.AddSingleton<ITransformer,
                            TransformerChain<ITransformer>> ((ctx) =>
                        {
                            MLContext mlContext = ctx.GetRequiredService<MLContext>();
                            string modelFilePathName = Configuration["MLModel:MLModelFilePath"];

                            ITransformer mlModel;
                            using (var fileStream = File.OpenRead(modelFilePathName))
                                mlModel = mlContext.Model.Load(fileStream);

                            return (TransformerChain<ITransformer>) mlModel;
                        });
    //..Other code..
}
```

Okay, that would be ready for you so you can inject and use the ML model (ITransformer) object on any object, for instance, by injecting it into any controller's constructor, like in the following code:

```cs
MyController.cs

[Route("api/[controller]")]
[ApiController]
public class MyController : ControllerBase
{
    private readonly MLContext _mlContext;
    private readonly ITransformer _mlModel;
    
    public MyController(MLContext mlContext, ITransformer mlModel)
    {
        // Get the injected objects
        _mlContext = mlContext;
        _mlModel = mlModel;
    }

    //...Other code using the ML model (ITransformer) object...
}

```

If later on, you create a `PredictionEngine` object whenever you need to call `predictionEngine.Predict()`, either by explicitely creating it with `new` or registering it as `Transient` lifetime, that would be safe in ASP.NET Core.

This is approximately the approach taken by this ML.NET tutorial named [How-To: Serve Machine Learning Model Through ASP.NET Core Web API](https://docs.microsoft.com/en-us/dotnet/machine-learning/how-to-guides/serve-model-web-api-ml-net)

However, you can do a lot better because with that approach it won't be optimized since whenever you get an Http request you'll be creating a new PredictionEngine object which, as previously mentioned, is a pretty 'expensive' operation for a scalable application if you want to have an optimized performance. 

For achieving the best performance in your application when predicting you will need, somehow, to cache the prediction engine object. But as mentioned, there are important caveats and problems here to solve.


# The problem when running/scoring an ML.NET model in multi-threaded applications

The problem running/scoring an ML.NET model in multi-threaded applications comes when you want to do single predictions with the PredictionEngine object and you want to cache it (i.e. as Singleton or static) so it is being reused by multiple Http requests (therefore it would be accessed by multiple threads)  because **the Prediction Engine is not thread-safe** ([ML.NET issue, Nov 2018](https://github.com/dotnet/machinelearning/issues/1718)).

Here's a diagram showing the important ML.NET classes you need to use:

![alt text](images/MLNET-classes-dependencies-for-scoring.png "ML.NET classes dependencies for scoring code with prediction engine")

If you register the Prediction Engine object as Singleton, you will get into trouble because it is not thread-safe.

# The solution: Use Object Pooling for the Prediction Engine 

TBD

# References

[Managed threading best practices](https://docs.microsoft.com/en-us/dotnet/standard/threading/managed-threading-best-practices) 