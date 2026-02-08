---
date: "2025-12-24T18:35:17+05:30"
draft: false
title: "Builder"
cascade:
  type: docs
---

Creating objects is one of the most common tasks in software development. In simple cases, a constructor is more than enough. However, as applications grow, object creation often becomes complex, error-prone, and hard to read.
This is where the Builder Pattern comes into play.
The Builder Pattern helps us construct complex objects step by step, while keeping the creation logic readable, maintainable, and flexible.

## The Problem: Complex Object Construction

Consider an object with many optional parameters:

```csharp
var user = new User(
    "vicky",
    "vicky@example.com",
    null,
    true,
    false,
    DateTime.Now
);
```

**At first glance, it’s not clear:**

- What each parameter represents
- Which values are optional
- Whether the order is correct

**As more parameters are added, developers often introduce:**

- Constructor overloading
- Optional parameters
- Long parameter lists

All of these solutions work—but they don’t scale well.

## The Intent of the Builder Pattern

> [!IMPORTANT]
> “Separate the construction of a complex object from its representation so that the same construction process can create different representations.”

**In simpler terms:**

- The object is built incrementally
- Construction logic is separated from business logic
- The final object is only exposed once it is complete
- The Builder Pattern focuses on how an object is created, not just what it is.

## When Should You Use the Builder Pattern?

The Builder Pattern is especially useful when:

- An object has many optional properties
- Construction must happen in specific steps
- The same object can be built in different variations
- You want to avoid constructor telescoping
- You want more readable and expressive object creation

## Fluent and Non-Fluent Builder Patterns

The Builder Pattern is often associated with fluent APIs and method chaining. However, fluent builders are only one variation of the pattern. In practice, there are two commonly used styles: non-fluent (classic) and fluent (modern) builders.
Both follow the same core idea—constructing a complex object step by step—but they differ in how the steps are expressed.

## Non-Fluent Builder (Classic GoF Style)

The non-fluent builder is the original form described in the Gang of Four design patterns book. In this style, builder methods typically return void, and each method represents a distinct construction step.

### Characteristics

- Builder methods return void
- Construction happens through explicit steps
- Order of method calls often matters

### Example Usage

```csharp
builder.BuildHeader();
builder.BuildBody();
builder.BuildFooter();
```

Here, each method modifies the internal state of the builder. Once all steps are completed, the final object is retrieved, usually through a separate method such as `GetResult()`.

### When to Use

- When the construction process is complex
- When build steps must be enforced or ordered
- When the same construction process should create different representations
- A real-world example of this style is ASP.NET Core’s `IApplicationBuilder`, where the HTTP request pipeline is built step by step using `UseXyz()` methods.

### Implementation (Example 1)

<iframe frameborder="0" style="width:100%;height:430px;" src="https://viewer.diagrams.net/?tags=%7B%7D&lightbox=1&highlight=000000&edit=_blank&layers=1&nav=1&title=NonFluentBuilder1.drawio&dark=1#Uhttps%3A%2F%2Fdrive.google.com%2Fuc%3Fid%3D1xMhmGLegMb5frQgfU3sgDuHU2B-xvKY5%26export%3Ddownload"></iframe>

```csharp {filename="DeploymentPipeline.cs"}
public class DeploymentPipeline
{
    public bool ConfigurationValidated { get; set; }
    public bool ArtifactsBuilt { get; set; }
    public bool TestsExecuted { get; set; }
    public bool Deployed { get; set; }
    public bool Verified { get; set; }

    public override string ToString()
    {
        return $"{nameof(ConfigurationValidated)}={ConfigurationValidated}, "
            + $"{nameof(ArtifactsBuilt)}={ArtifactsBuilt}, "
            + $"{nameof(TestsExecuted)}={TestsExecuted}, "
            + $"{nameof(Deployed)}={Deployed}, "
            + $"{nameof(Verified)}={Verified}";
    }
}
```

```csharp {filename="IDeploymentPipelineBuilder.cs"}
public interface IDeploymentPipelineBuilder
{
    void ValidateConfiguration();
    void BuildArtifacts();
    void RunTests();
    void Deploy();
    void Verify();
    DeploymentPipeline GetResult();
}
```

```csharp {filename="SocialMediaPostsServiceTests.cs"}
public class ProductionDeploymentBuilder : IDeploymentPipelineBuilder
{
    private readonly DeploymentPipeline _pipeline = new();

    public void ValidateConfiguration()
    {
        // Setting ConfigurationValidated to true for simulation purposes
        _pipeline.ConfigurationValidated = true;
    }

    public void BuildArtifacts()
    {
        if (!_pipeline.ConfigurationValidated)
            throw new InvalidOperationException("Configuration must be validated first.");

        _pipeline.ArtifactsBuilt = true;
    }

    public void RunTests()
    {
        if (!_pipeline.ArtifactsBuilt)
            throw new InvalidOperationException("Artifacts must be built before tests.");

        _pipeline.TestsExecuted = true;
    }

    public void Deploy()
    {
        if (!_pipeline.TestsExecuted)
            throw new InvalidOperationException("Tests must pass before deployment.");

        _pipeline.Deployed = true;
    }

    public void Verify()
    {
        if (!_pipeline.Deployed)
            throw new InvalidOperationException("Deployment must complete before verification.");

        _pipeline.Verified = true;
    }

    public DeploymentPipeline GetResult()
    {
        return _pipeline;
    }
}
```

```csharp {filename="IDeploymentDirector.cs"}
public interface IDeploymentDirector
{
    DeploymentPipeline ConstructFullPipeline();
}

```

```csharp {filename="DeploymentDirector.cs"}
public class DeploymentDirector(IDeploymentPipelineBuilder builder) : IDeploymentDirector
{
    public DeploymentPipeline ConstructFullPipeline()
    {
        builder.ValidateConfiguration();
        builder.BuildArtifacts();
        builder.RunTests();
        builder.Deploy();
        builder.Verify();
        return builder.GetResult();
    }
}
```

```csharp {filename="Program.cs"}
using Microsoft.Extensions.DependencyInjection;
using NonFluentBuilder;

var services = new ServiceCollection();
services.AddScoped<IDeploymentPipelineBuilder, ProductionDeploymentBuilder>();
services.AddScoped<IDeploymentDirector, DeploymentDirector>();

using var serviceProvider = services.BuildServiceProvider();
using var scope = serviceProvider.CreateScope();
var director = scope.ServiceProvider.GetRequiredService<IDeploymentDirector>();
var pipeline = director.ConstructFullPipeline();

Console.WriteLine(pipeline);
```

### Implementation (Example 2)

<iframe frameborder="0" style="width:100%;height:430px;" src="https://viewer.diagrams.net/?tags=%7B%7D&lightbox=1&highlight=000000&edit=_blank&layers=1&nav=1&title=NonFluent2.drawio&dark=1#Uhttps%3A%2F%2Fdrive.google.com%2Fuc%3Fid%3D1vTKQui3lM33jkzVi791a1Rro_uM_ZVh7%26export%3Ddownload"></iframe>

```csharp {filename="DatabaseConfiguration.cs"}
public class DatabaseConfiguration
{
    [Required(ErrorMessage = "Host is required.")]
    public string? Host { get; set; }

    [Range(1, 65535, ErrorMessage = "Port must be between 1 and 65535.")]
    public int Port { get; set; }

    [Required(ErrorMessage = "Database name is required.")]
    public string? Database { get; set; }

    public bool UseSsl { get; set; }

    [Range(0, int.MaxValue, ErrorMessage = "Timeout cannot be negative.")]
    public int TimeoutSeconds { get; set; }
    public override string ToString()
    {
        return $"{nameof(Host)}={(Host ?? "null")}, " +
               $"{nameof(Port)}={Port}, " +
               $"{nameof(Database)}={(Database ?? "null")}, " +
               $"{nameof(UseSsl)}={UseSsl}, " +
               $"{nameof(TimeoutSeconds)}={TimeoutSeconds}";
    }
}
```

```csharp {filename="IDatabaseConfigurationBuilder.cs"}
public interface IDatabaseConfigurationBuilder
{
    void SetHost(string host);
    void SetPort(int port);
    void SetDatabase(string database);
    void EnableSsl(bool enabled);
    void SetTimeout(int seconds);
    DatabaseConfiguration Build();
}

```

```csharp {filename="DatabaseConfigurationBuilder.cs"}
public class DatabaseConfigurationBuilder : IDatabaseConfigurationBuilder
{
    private readonly DatabaseConfiguration _configuration = new();

    public void SetHost(string host)
    {
        _configuration.Host = host;
    }

    public void SetPort(int port)
    {
        _configuration.Port = port;
    }

    public void SetDatabase(string database)
    {
        _configuration.Database = database;
    }

    public void EnableSsl(bool enabled)
    {
        _configuration.UseSsl = enabled;
    }

    public void SetTimeout(int seconds)
    {
        _configuration.TimeoutSeconds = seconds;
    }

    public DatabaseConfiguration Build()
    {
        var context = new ValidationContext(_configuration);
        var results = new List<ValidationResult>();

        if (Validator.TryValidateObject(_configuration, context, results, validateAllProperties: true))
            return _configuration;

        var messages = results.Select(r => r.ErrorMessage).Where(m => m != null);
        throw new InvalidOperationException(string.Join(" ", messages));
    }
}

```

```csharp {filename="Program.cs"}
using Microsoft.Extensions.DependencyInjection;
using NonFluentBuilder2;

var services = new ServiceCollection();
services.AddScoped<IDatabaseConfigurationBuilder, DatabaseConfigurationBuilder>();

using var serviceProvider = services.BuildServiceProvider();
using var scope = serviceProvider.CreateScope();

var builder = scope.ServiceProvider.GetRequiredService<IDatabaseConfigurationBuilder>();

builder.SetHost("db.myCompany.com");
builder.SetPort(5432);
builder.SetDatabase("Orders");
builder.EnableSsl(true);
builder.SetTimeout(30);

var databaseConfiguration = builder.Build();
Console.WriteLine(databaseConfiguration);

```

## Fluent Builder (Modern Variation)

The fluent builder is a modern variation that improves readability by returning the builder instance (this) from each method. This allows multiple build steps to be chained together in a single expression.

### Characteristics

- Builder methods return the builder itself
- Enables method chaining
- Emphasizes readability and developer experience

### Example Usage

```csharp
var report = new ReportBuilder()
    .WithHeader("Header")
    .WithBody("Body")
    .WithFooter("Footer")
    .Build();
```

This style is especially popular in public APIs, SDKs, and configuration-heavy code, where clarity and ease of use are important.

### When to Use

- When an object has many optional parameters
- When API usability and readability matter
- When developers interact directly with the builder

### Implementation

<iframe frameborder="0" style="width:100%;height:430px;" src="https://viewer.diagrams.net/?tags=%7B%7D&lightbox=1&highlight=000000&edit=_blank&layers=1&nav=1&title=FluentBuilder.drawio&dark=1#Uhttps%3A%2F%2Fdrive.google.com%2Fuc%3Fid%3D1Cat3NVqnQPNTrq2RyBy7VCNMZGwdHhtR%26export%3Ddownload"></iframe>

```csharp {filename="Car.cs"}
public class Car
{
    public string? Brand { get; set; }
    public string? Model { get; set; }
    public string? Transmission { get; set; }
    public bool HasSunroof { get; set; }
    public bool HasNavigation { get; set; }
    public bool HasParkingSensors { get; set; }
    public int NumberOfDoors { get; set; }

    public override string ToString()
    {
        return $"{Brand} {Model}, {Transmission}, "
            + $"Doors: {NumberOfDoors}, "
            + $"Sunroof: {HasSunroof}, "
            + $"Navigation: {HasNavigation}, "
            + $"Parking Sensors: {HasParkingSensors}";
    }
}

```

```csharp {filename="ICarBuilder.cs"}
public interface ICarBuilder
{
    ICarBuilder SetBrand(string brand);
    ICarBuilder SetModel(string model);
    ICarBuilder SetTransmission(string transmission);
    ICarBuilder AddSunroof();
    ICarBuilder AddNavigation();
    ICarBuilder AddParkingSensors();
    ICarBuilder SetNumberOfDoors(int doors);
    Car Build();
}
```

```csharp {filename="CarBuilder.cs"}
public class CarBuilder : ICarBuilder
{
    private Car _car = new();

    public ICarBuilder SetBrand(string brand)
    {
        _car.Brand = brand;
        return this;
    }

    public ICarBuilder SetModel(string model)
    {
        _car.Model = model;
        return this;
    }

    public ICarBuilder SetTransmission(string transmission)
    {
        _car.Transmission = transmission;
        return this;
    }

    public ICarBuilder AddSunroof()
    {
        _car.HasSunroof = true;
        return this;
    }

    public ICarBuilder AddNavigation()
    {
        _car.HasNavigation = true;
        return this;
    }

    public ICarBuilder AddParkingSensors()
    {
        _car.HasParkingSensors = true;
        return this;
    }

    public ICarBuilder SetNumberOfDoors(int doors)
    {
        _car.NumberOfDoors = doors;
        return this;
    }

    public Car Build()
    {
        var built = new Car
        {
            Brand = _car.Brand,
            Model = _car.Model,
            Transmission = _car.Transmission,
            HasSunroof = _car.HasSunroof,
            HasNavigation = _car.HasNavigation,
            HasParkingSensors = _car.HasParkingSensors,
            NumberOfDoors = _car.NumberOfDoors,
        };

        // reset builder for next car
        _car = new Car();
        return built;
    }
}

```

```csharp {filename="Program.cs"}
using FluentBuilder;
using Microsoft.Extensions.DependencyInjection;

var services = new ServiceCollection();
services.AddScoped<ICarBuilder, CarBuilder>();

using var serviceProvider = services.BuildServiceProvider();
using var scope = serviceProvider.CreateScope();
var carBuilder = scope.ServiceProvider.GetRequiredService<ICarBuilder>();

var car = carBuilder
    .SetBrand("BMW")
    .SetModel("X5")
    .SetTransmission("Automatic")
    .SetNumberOfDoors(4)
    .AddSunroof()
    .AddNavigation()
    .Build();

Console.WriteLine(car);

```

You can find the codes in the following GitHub repository.

#### See On Github Repository

{{< cards cols="1" >}}
{{< card link="https://github.com/vicky-sh/BuilderPattern.git" title="Builder Pattern" icon="github" >}}
{{< /cards >}}

#### Open in an Online Code Editor

{{< cards cols="1" >}}
{{< card link="https://github.dev/vicky-sh/BuilderPattern" title="Builder Pattern" icon="github" >}}
{{< /cards >}}
