---
date: "2025-12-24T23:19:09+05:30"
draft: false
title: "Factory"
cascade:
  type: docs
---

In the [Builder Pattern](../builder), we focused on constructing complex objects piece by piece. It works great when an object requires multiple steps or configurations. However, not every situation demands that level of control.

Imagine you just want to create an object based on a type—no step-by-step construction, no complex setup. Just a simple decision: *give me the right object*.

How do we design our code so that object creation remains flexible, maintainable, and scalable? This is exactly the problem the Factory Pattern is designed to solve.

However, there’s also a subtle issue that becomes apparent as applications grow.
Directly creating objects using constructors often leads to scattered conditional logic:

```csharp
if (type == "truck")
    transport = new Truck();
else if (type == "ship")
    transport = new Ship();

```
At first, this might seem manageable. But over time, it leads to code that is tightly coupled to concrete classes. Every time a new type is added, existing 
code must be modified—sometimes in multiple places. As pointed out by [Refactoring Guru](https://refactoring.guru/design-patterns/factory-method), this results in code that is harder to maintain, harder to extend, 
and filled with repetitive conditionals.

This is where the real shift in thinking happens.

## Why the Factory Pattern Exists

> [!IMPORTANT]
> The Factory Pattern exists to separate object creation from usage, reducing coupling and making systems easier to extend. Instead of asking *"Which class should I instantiate?", we move toward "How can I delegate this responsibility?"*

Now, the responsibility of deciding what to create is moved away from the client code and into specialized classes. This keeps our code clean, reduces coupling, and makes it much easier to introduce new types without modifying existing logic.

In the following sections, we’ll explore how this works in practice and how the Factory Pattern helps us write more maintainable code. 

## Basic design of the factory pattern

<iframe frameborder="0" style="width:100%;height:700px;" src="https://viewer.diagrams.net/?tags=%7B%7D&lightbox=1&highlight=000000&edit=_blank&layers=1&nav=1&title=BasicFactory.drawio&dark=1#Uhttps%3A%2F%2Fdrive.google.com%2Fuc%3Fid%3D1GH3pRIbOyTlAL8iiOm24OD2D_XpFtclJ%26export%3Ddownload"></iframe>

The Factory Method pattern revolves around two main roles: the ***`Product`*** and the ***`Creator`***.

The Product is the thing we want to create. Instead of working with concrete classes directly, we 
define a common abstraction—usually an interface like `IProduct`. All concrete implementations (`ConcreteProductA`, `ConcreteProductB`, etc.) 
follow this contract. This allows the rest of the code to stay unaware of the exact implementation it’s working with.

The `Creator` is where things get interesting. It defines a method—typically called something like `createProduct()`—that returns an IProduct. 
But the key point is: it doesn’t actually decide which product to create.

That responsibility is delegated to subclasses.

Each concrete creator (`ConcreteCreatorA`, `ConcreteCreatorB`, …) overrides the factory method and returns a specific product. 
One returns `ConcreteProductA`, another returns `ConcreteProductB`, and so on.

From the client’s perspective, this simplifies everything:

```csharp
var product = creator.createProduct();
product.doStuff();
```

No conditionals. No knowledge of concrete classes. Just a clean interaction with an abstraction.

This is where the Factory Method really pays off. The decision of what to create is no longer scattered across the codebase—it’s 
centralized and encapsulated. If a new product type is introduced, you don’t go back and modify existing logic. You just add a new creator.

In real-world code, this helps avoid the kind of if-else or switch blocks that tend to grow over time and become hard to maintain. Instead, 
object creation becomes predictable, isolated, and easy to extend.

In short, the Factory Method gives you a clean way to delegate object creation while keeping your core logic simple and decoupled.

## Factory Implementation without Dependency Injection

Now let’s connect this abstract idea to a real implementation.

<iframe frameborder="0" style="width:100%;height:500px;" src="https://viewer.diagrams.net/?tags=%7B%7D&lightbox=1&highlight=000000&edit=_blank&layers=1&nav=1&title=BasicFactory.drawio&dark=1#Uhttps%3A%2F%2Fdrive.google.com%2Fuc%3Fid%3D1Ye5gCuKKhx9JbLfaWF_OBkN1P6D7HXOT%26export%3Ddownload"></iframe>

In the diagram above, we move from the generic Product / Creator terminology to something more concrete: a payment system. While the names have changed, the underlying structure remains exactly the same.

### Mapping the Theory to Code

If we translate the basic Factory Method design into this example, the roles become clearer:
- ***IPayment*** → Product
- ***ApplePayPayment, GooglePayPayment, PayPalPayment*** → Concrete Products
- ***PaymentProvider*** → Creator
- ***ApplePayPaymentProvider, GooglePayPaymentProvider, PayPalPaymentProvider*** → Concrete Creators
- ***CheckoutService*** → Client

The structure follows the same principle we discussed earlier:

The client (*CheckoutService*) does not directly instantiate payment objects. Instead, it delegates this responsibility to a factory (PaymentProvider).

### The Product: `IPayment`

The `ÌPayment` interface represents the Product in the Factory Method pattern. 

```csharp {filename="IPayment.cs"}
public interface IPayment
{
    void ProcessPayment(decimal amount);
}
````

Instead of working with concrete classes like `PayPalPayment` or `ApplePayPayment`, the system depends on this abstraction. Each concrete implementation - `PayPalPayment`, `ApplePayPayment` and `GooglePayPayment` provides its own behavior for processing a payment, while still following the same contract. This allows the rest of the system to remain completely unaware of which specific payment method is being used.


```csharp {filename="PayPalPayment.cs"}
public class PayPalPayment : IPayment
{
    public void ProcessPayment(decimal amount)
    {
        Console.WriteLine($"Processing PayPal Payment for Amount: {amount}");
    }
}
```

```csharp {filename="ApplePayPayment.cs"}
public class ApplePayPayment : IPayment
{
    public void ProcessPayment(decimal amount)
    {
        Console.WriteLine($"Processing ApplePay Payment for Amount: {amount}");
    }
}
```

```csharp {filename="GooglePayPayment.cs"}
public class GooglePayPayment : IPayment
{
    public void ProcessPayment(decimal amount)
    {
        Console.WriteLine($"Processing GooglePay Payment for Amount: {amount}");
    }
}
```
### The Creator: `PaymentProvider`

The `PaymentProvider` acts as the Creator. 

```csharp {filename="PaymentProvider.cs"}
public abstract class PaymentProvider
{
    public abstract IPayment CreatePayment();

    public void ExecutePayment(decimal amount)
    {
        IPayment payment = CreatePayment();
        payment.ProcessPayment(amount);
    }
}
```

This is where the Factory Method lives.

- `CreatePayment()` is the factory method
- It returns an `IPayment`, but does not specify the concrete payment type

What’s interesting here is the `ExecutePayment` method.

Instead of the client creating a payment and calling ProcessPayment, the creator itself handles the full workflow:

1. It creates the product using `CreatePayment()`
2. It immediately uses it via `ProcessPayment(amount)`

In other words, the creator doesn’t just create the product—it also invokes its behavior. This keeps the client even simpler and encapsulates more logic inside the creator.

Each concrete providers `PayPalPaymentProvider`, `ApplePayPaymentProvider`, `GooglePayPaymentProvider` decide which product to instantiate. Each one overrides `CreatePayment` and returns its corresponding product.


```csharp {filename="PayPalPaymentProvider.cs"}
public class PayPalPaymentProvider : PaymentProvider
{
    public override IPayment CreatePayment()
    {
        return new PayPalPayment();
    }
}
```

```csharp {filename="ApplePayPaymentProvider.cs"}
public class ApplePayPaymentProvider : PaymentProvider
{
    public override IPayment CreatePayment()
    {
        return new ApplePayPayment();
    }
}
```

```csharp {filename="GooglePayPaymentProvider.cs"}
public class GooglePayPaymentProvider : PaymentProvider
{
    public override IPayment CreatePayment()
    {
        return new GooglePayPayment();
    }
}
```

### The Client: `CheckoutService`

The `CheckoutService`  represents the client code. It neither instantiates any payment object, nor calls `ProcessPayment` directly. Instead it determines the correct provider and delegates the execution to it. This keep shte client focussed on the orchestration rather than implementation details.

```csharp {filename="ICheckoutService.cs"}
public interface ICheckoutService
{
    void Checkout(string paymentType, decimal amount);
    PaymentProvider GetPaymentProvider(string paymentType);
}
```

```csharp {filename="CheckoutService.cs"}
public class CheckoutService : ICheckoutService
{
    public void Checkout(string paymentType, decimal amount)
    {
        PaymentProvider provider = GetPaymentProvider(paymentType);
        provider.ExecutePayment(amount);
    }

    // Extracted for testing so it can be mocked.
    public PaymentProvider GetPaymentProvider(string paymentType)
    {
        return paymentType switch
        {
            "payPal" => new PayPalPaymentProvider(),
            "googlePay" => new GooglePayPaymentProvider(),
            "applePay" => new ApplePayPaymentProvider(),
            _ => throw new ArgumentException("invalid payment type"),
        };
    }
}
```
```csharp {filename="Program.cs"}
CheckoutService checkoutService = new();
checkoutService.Checkout("payPal", 1000);
checkoutService.Checkout("applePay", 2000);
checkoutService.Checkout("googlePay", 3000);
```


### Selecting the Correct Provider

There is one part of the implementation that deserves a closer look: how the correct `PaymentProvider` is selected.

```csharp
public PaymentProvider GetPaymentProvider(string paymentType)
{
    return paymentType switch
    {
        "payPal" => new PayPalPaymentProvider(),
        "googlePay" => new GooglePayPaymentProvider(),
        "applePay" => new ApplePayPaymentProvider(),
        _ => throw new ArgumentException("invalid payment type"),
    };
}
```

This `switch` expression determines which concrete creator should be used. At first glance, this might look similar to the problem we originally tried to solve—conditional logic based on a type. However, there is an important difference:

{{< callout type="important" >}} 
The conditional logic is no longer responsible for creating the product, but only for selecting the factory.
{{< /callout >}}

This is already a significant improvement.

- The creation of IPayment implementations is fully delegated
- The CheckoutService does not know anything about concrete payment classes
- Each provider encapsulates its own creation logic

The `CheckoutService` is still responsible for choosing the correct provider. which is a small delimitiation. As the number of payment methods grows, this `switch` statement will also grow. While manageable in small examples, it introduces a form of coupling that we might want to eliminate.

### Looking Ahead

A cleaner approach would be to delegate this responsibility to a separate component, often called a resolver or factory of factories. This would allow the `CheckoutService` to focus purely on orchestration, without needing to know how providers are selected.

In the next implementation, we’ll take this idea a step further by introducing Dependency Injection, where:

- The correct provider is resolved automatically
- The switch statement disappears entirely
- The system becomes even more flexible and extensible

## Factory Implementation with Dependency Injection

In the previous implementation, the `CheckoutService` was still responsible for selecting the correct `PaymentProvider` using a `switch` statement. While this worked, it meant that part of the decision-making logic still lived inside the client. With Dependency Injection, this responsibility is now fully delegated. Below is a diagram which shows the exact example we discussed before but using Dependency Injection.

<iframe frameborder="0" style="width:100%;height:660px;" src="https://viewer.diagrams.net/?tags=%7B%7D&lightbox=1&highlight=000000&edit=_blank&layers=1&nav=1&title=BasicFactory.drawio&dark=1#Uhttps%3A%2F%2Fdrive.google.com%2Fuc%3Fid%3D1Vsad57cMnrq_CFv5ey4Et3MOOqBYVojJ%26export%3Ddownload"></iframe>

### Making Providers Self-Describing

The first change is subtle but important. Each `PaymentProvider` now exposes a `PaymentType`:

```csharp {filename="PaymentProvider.cs"}
public abstract class PaymentProvider
{
    public abstract string PaymentType { get; }
    public abstract IPayment CreatePayment();

    public void ExecutePayment(decimal amount)
    {
        IPayment payment = CreatePayment();
        payment.ProcessPayment(amount);
    }
}
```

Each concrete provider defines its own type:

```csharp {filename="GooglePayPaymentProvider.cs"}
public class GooglePayPaymentProvider : PaymentProvider
{
    public override string PaymentType => "GooglePay";

    public override IPayment CreatePayment()
    {
        return new GooglePayPayment();
    }
}
```

```csharp {filename="PayPalPaymentProvider.cs"}
public class PayPalPaymentProvider : PaymentProvider
{
    public override string PaymentType => "PayPal";

    public override IPayment CreatePayment()
    {
        return new PayPalPayment();
    }
}
```

```csharp {filename="ApplePayPaymentProvider.cs"}
public class ApplePayPaymentProvider : PaymentProvider
{
    public override string PaymentType => "ApplePay";

    public override IPayment CreatePayment()
    {
        return new ApplePayPayment();
    }
}
```

This small addition allows each provider to describe what it supports. And that’s exactly what enables the next step.

### Resolving the Provider

Instead of using a `switch`, we introduce a resolver:

```csharp {filename="PaymentCreatorFactory.cs"}
public class PaymentCreatorFactory(IEnumerable<PaymentProvider> creators)
{
    public PaymentProvider Get(string paymentType)
    {
        PaymentProvider? creator = creators.FirstOrDefault(x =>
            x.PaymentType.Equals(paymentType, StringComparison.OrdinalIgnoreCase)
        );

        return creator ?? throw new ArgumentException($"Unknown payment type: {paymentType}");
    }
}
```

This class acts as a resolver. Instead of hardcoding the decision, we now query a collection of available providers.

- All `PaymentProvider` implementations are injected as a collection
- The factory searches for the one matching the requested `paymentType`
- The correct provider is returned dynamically

### Simplifying the Client

With this in place, the `CheckoutService` becomes much simpler:

```csharp {filename="CheckoutService.cs"}
public class CheckoutService(PaymentCreatorFactory paymentCreatorFactory) : ICheckoutService
{
    public void Checkout(string paymentType, decimal amount)
    {
        PaymentProvider provider = paymentCreatorFactory.Get(paymentType);
        provider.ExecutePayment(amount);
    }
}
```
There is no conditional logic left.

- No `switch`
- No `ìf-else`
- No knowledge of concrete providers

The client now focuses purely on orchestrating the flow.

### Wiring It All Together

The Dependency Injection container takes care of assembling everything:

```csharp {filename="Program.cs"}
ServiceCollection services = new();

services.AddTransient<IPayment, PayPalPayment>();
services.AddTransient<IPayment, ApplePayPayment>();
services.AddTransient<IPayment, GooglePayPayment>();

services.AddTransient<PaymentProvider, PayPalPaymentProvider>();
services.AddTransient<PaymentProvider, GooglePayPaymentProvider>();
services.AddTransient<PaymentProvider, ApplePayPaymentProvider>();

services.AddScoped<PaymentCreatorFactory>();
services.AddScoped<ICheckoutService, CheckoutService>();
```

All providers are registered, and the container automatically injects them as `IEnumerable<PaymentProvider>`. This is what makes the resolver work without any manual wiring.

You can find the codes in the following GitHub repository.

## See On Github Repository

{{< cards cols="1" >}}
{{< card link="https://github.com/vicky-sh/FactoryPattern" title="Factory Pattern" icon="github" >}}
{{< /cards >}}

## Open in an Online Code Editor

{{< cards cols="1" >}}
{{< card link="https://github.dev/vicky-sh/FactoryPattern" title="Factory Pattern" icon="github" >}}
{{< /cards >}}