# ML.NET Model Lifecycle with Azure DevOps CI/CD pipelines 

![alt text](images/header-image.png "ML.NET logo plus other")

As a developer or software architect, you are focused on the application lifecycle – building, maintaining, and continuously updating the end-user business application, as illustrated in the simplified image below:

![alt text](images/dev-ops-workflow.png "Regular DevOps workflow")

But when you infuse AI (such as an ML.NET model) into your application, then your application lifecycle needs to be extended so it embraces the additional 'Machine Learning Model Lifecycle'.

# Extending Azure DevOps CI/CD pipelines with the ML Model lifecycle

When bringing ML models to production, you need to automate the process to track / version / audit / certify / re-use every asset in your ML model lifecycle along with the end-user application lifecycle.

Basically, you have to extend your *DevOps CI/CD pipelines* (in this case using **Azure DevOps**) to handle not just your end-user application but the ML model generation/traing, model testing/evaluation and automatic model deployment and as in the following workflow illustration:

![alt text](images/dev-ops-workflow-with-ml-model.png "Regular DevOps workflow")

In short, the ML model lifecycle process must be part of the application’s Continuous Integration (CI) and Continuous Delivery (CD) pipelines.

Let’s walk through the diagram above to understand how this integration between the ML model lifecycle and the app development lifecycle can be achieved.

For this common scenario, a starting assumption is that Git is used as your code repository, but it could be any other source code management platform. 

In the same way than the app developer makes changes in the application and pushes code to Git and that will trigger builds and applications tests in the CI pipeline and ultimately followed by the application deployment in the CD pipeline, the ML model needs a similar process. 

Any changes made in the training code or equally important changes in traing data, that will trigger the Azure DevOps CI/CD pipeline to compile the trainer app, train a new ML model, run unit tests validating the quality of that ML model, deploy the ML model file into the end-user application and finally followed by the application deployment in the CD pipeline.

--------------------------
--------------------------

If the dataset to be used to train the model is a small dataset file (i.e. less than 100Mb file), you could push that dataset file directly into Git and that will trigger the CI/CD pipelines.

If the dataset you need to use for training is significantly large and you have those datasets in a different infrastructure such as Azure Files, you could set other specific triggers to execute both model retraining and application deployment steps. 

--------------------------
--------------------------

You retain full control over the ML model training. You can continue to write and train models in your favorite  environment when developing or experimenting (data wrangling, feature extraction, and algorithm/trainer). Then, you get to decide when to refresh the data or change the training code by pushing into Git to trigger the 'official' training of the ML model to be deployed into the end-user application. 

At the same time, you can sleep comfortably knowing that any changes you commit will pass through the required unit, integration testing, and optional human approval steps (releases to production from Azure DevOps Release Manager) for the overall application.

--------------------------
--------------------------

Depending on the end-user application architecture (monolithic vs. microservices) the deployment to be done can be different. In a monolithic architecture you might need to re-deploy the whole app (let's say a monolithic web app deployed into Azure App Service) while in a microservices app you could just deploy a single microservice/container running the ML model into the existing microservices app already deployed.

--------------------------
--------------------------

# The sample ML model and sample ASP.NET Core WebAPI service for this blog post

In order to make this blog post simple and mostly focus on DevOps CI/CD, the [ML.NET](https://dotnet.microsoft.com/apps/machinelearning-ai/ml-dotnet) model and sample app/service to be deployed must be kept simple while with enough implementation so it is useful.

But the goal is not to have a large explanation about the ML model implementation since there are other tutorials I point to for that purpose. 

## Sample ML Model: Sentiment Analysis model (Binary Classification) created with the ML.NET CLI

The sample ML model and even its related trainer console app were created with the ML.NET CLI. In fact, you could create the same ML Model and trainer console app by following this [ML.NET CLI tutorial](https://docs.microsoft.com/en-us/dotnet/machine-learning/tutorials/mlnet-cli?tabs=windows).

This is basically the code generated by the CLI in that tutorial and what we're going to use as sample ML model and trainer console app for the CI pipeline:

![VS solution generated by the CLI](images/generated-csharp-solution-detailed.png)

However, you don't need to generate/create that code following the mentioned tutorial since I'm also making it available in the GitHub repo related to this Blog Post, here:

- [Yelp reviews-sentiment dataset with added header in the file](https://github.com/CESARDELATORRE/MLNETSENTIMENTCICD/blob/master/MLModel.Train/Data/yelp_labelled.tsv)


- [CLI command used to generate model and code](https://github.com/CESARDELATORRE/MLNETSENTIMENTCICD/blob/master/MLModel.Train/CLI-Commands.txt) 

- [Source code of trainer console app and Sample ML model](https://github.com/CESARDELATORRE/MLNETSENTIMENTCICD/tree/master/MLModel.Train/SentimentModel)

## Sample ASP.NET Core WebAPI service

The end-user app where you want to deploy the ML model is in this case a very simple ASP.NET Core WebAPI service. The code is available in the same GitHub repo:

- [Source code of sample end-user ASP.NET Core WebAPI service](https://github.com/CESARDELATORRE/MLNETSENTIMENTCICD/tree/master/Scalable.WebAPI)

You can also explore a very similar WebAPI implementation running an ML.NET model in this 'how to' guide:

- [Deploy a model in an ASP.NET Core Web API](https://docs.microsoft.com/en-us/dotnet/machine-learning/how-to-guides/serve-model-web-api-ml-net).

Having the ML model, trainer console app and final app/service to deploy the model to, let's now drill down on the different CI/CD pipelines in Azure DevOps and learn multiple approaches you can implement.

# The CI pipeline in Azure DevOps including the ML.NET Model lifecycle

The CI (Continuous Integration) pipeline is pretty similar to regular application CI pipelines, but you need to add the following steps:

- Build/compile the ML model trainer app (Usually a console app)
- Run the process (console app) to train the ML.NET model and generate the serialized model (.zip file).
- Run model's tests (model quality validation)
- Deployment the model file into the actual end-user application code (project structure)

Once you run those tasks in the CI pipeline then you'd follow the typical application build tasks such as:

- Build/compile the end-user app (such as an ASP.NET Core web app or WebAPI service)
- Run app's unit tests and integration tests
- Generate and publish the final deployment artifact in Azure DevOps (or if using containers, generate a Docker image and publish it into a Docker Registry)

Here's a screenshot of a simplified approach of an Azure DevOps CI pipeline including all those steps.


**Azure DevOps CI pipeline (Visual tasks approach)**

![Azure DevOps CI pipeline (Visual tasks approach)](images/azure-devops-pipeline-visual-tasks.png)

I published this Azure DevOps publicly as READ-ONLY, so you can also see how it runs, here:

- [Public access (Read-only) of Azure DevOps CI pipeline (Visual tasks approach)](https://dev.azure.com/mlnetsamples/MLNETWebAPISample/_build?definitionId=9)
 

Note that for the "first look" above I'm showing a visual "task-based" CI pipeline because it is clearer and more visual for a first quick look, but in the upcoming detailed steps we switch to a YAML pipeline (**Azure-Pipelines.yaml**) because it offers better tracking with its related source code since the .YAML file is also checked in and pushed into the same Git repo used for the artifacts source code.

# The CD pipeline in Azure DevOps 

This is pretty much the same build pipeline but using YAML as a **Azure-Pipelines.yaml** file, which you can also see published at my GitHub repo here:

https://github.com/CESARDELATORRE/MLNETSENTIMENTCICD/blob/master/azure-pipelines.yml

```yaml
# Build/CI pipeline for ML.NET model, its trainer app and end-user WebAPI service

trigger:
- master

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'

steps:

- script: dotnet build MLModel.Train/SentimentModel/SentimentModel.ConsoleApp/SentimentModel.ConsoleApp.csproj --configuration $(buildConfiguration)
  displayName: 'Build Trainer Console App (dotnet build) $(buildConfiguration)'

- script: dotnet run --project MLModel.Train/SentimentModel/SentimentModel.ConsoleApp/SentimentModel.ConsoleApp.csproj --configuration $(buildConfiguration)
  displayName: 'Train ML model (dotnet run)'

- script: dotnet build MLModel.Train/UnitTests/UnitTests.csproj --configuration $(buildConfiguration)
  displayName: 'Build Test project for ML Model (dotnet build) $(buildConfiguration)'

- task: DotNetCoreCLI@2
  displayName: 'Run Unit Tests using trained ML model'
  inputs:
    command: test
    projects: '**/UnitTests.csproj'
    arguments: '--configuration $(buildConfiguration)'

- task: CopyFiles@2
  displayName: 'Copy ML model file from Trainer app to WebAPI app'
  inputs:
    SourceFolder: 'MLModel.Train/SentimentModel/SentimentModel.Model'
    Contents: 'MLModel.zip'
    TargetFolder: 'Scalable.WebAPI/ML'
    OverWrite: true

- script: dotnet build Scalable.WebAPI/Scalable.WebAPI.csproj --configuration $(buildConfiguration)
  displayName: 'Build WebAPI service (dotnet build) $(buildConfiguration)'

- task: DotNetCoreCLI@2
  displayName: 'Generate WebAPI binaries (dotnet publish)'
  inputs:
    command: publish
    publishWebProjects: false
    projects: Scalable.WebAPI/Scalable.WebAPI.csproj
    arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'
    modifyOutputPath: false

- task: PublishPipelineArtifact@0
  displayName: 'Publish WebAPI as pipeline artifact'
  inputs:
    artifactName: MLNETWebAPI
    targetPath: '$(Build.ArtifactStagingDirectory)'
```

Initially, all the .YAML code can look a bit overwhelming compared to the visual tasks, but once you know the tasks you are running, the .YAML approach is much more direct and in a single scan/read you see everything the pipeline is running. On the other hand, if using the visual tasks you'd need to enter on each task and review all its parameters sometimes in hidden/closed tabs, etc.  Really, once you are used to use it, you will prefer the .YAML approach not counting the best benefit which is that you are checking the YAML file in the same repo than its related source code! :)\ 

Let's drill down in the 
CD (Continuous Deployment) pipeline







# Get started with ML.NET 1.0!

![alt text](MLNET-Blog-Post-images/get-started-rocket.png "Get started icon")

This current blog post is providing significant technical details. For a simpler ML.NET 1.0 release announcement, check out the [official ML.NET 1.0 announcement Blog Post here](https://aka.ms/mlnet1-announcement-blog-post).

Get started with ML.NET by exploring the following resources:

  * **Get started** with <a href="https://www.microsoft.com/net/learn/apps/machine-learning-and-ai/ml-dotnet/get-started">ML.NET here</a>. 
  * **Tutorials** and resources at the [Microsoft Docs ML.NET Guide](https://docs.microsoft.com/en-us/dotnet/machine-learning/)
  * **Sample apps** using ML.NET at the [machinelearning-samples GitHub repo](https://github.com/dotnet/machinelearning-samples)


Thanks and happy coding with [ML.NET](http://dot.net/ml)!

*Cesar de la Torre*

*Principal Program Manager*

*.NET and ML.NET product group*

*Microsoft Corp.*

