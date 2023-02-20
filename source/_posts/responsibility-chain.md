---
title: Taming Complexity with Responsibility
tags:
  - C#
  - Gang Of Four
  - Design Patterns
  - Programming
categories:
  - Programming
  - Design Patterns
top_img: cover.jpg
cover: cover.jpg
---

## It all starts with a simple task

Imagine, one quiet morning, your boss comes to you and says, "Hey, our web shop is growing and we will be having more than one delivery provider now. Can you implement something that would select the best provider after a client pays for a delivery?".
After some back and forth about the criteria on how a delivery company should be selected - mostly by package size, weight and delivery company area, you set out to write the code. How hard can it be? Just write a few ``if`` statements, and that's it, right?

Some time later, you might end with a following solution. Rather simple but short and concise. No need for over-engineering and [YAGNI](https://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it) is the right way to go, right?

```cs
//note: IPoint and IGeometry are NetTopologySuite classes and depict geospatial data
public record OrderDelivery(IPoint Address, Vector3 Size, float Weight)
{
}

public interface IDeliveryProvider
{
   float MaxWeight { get; }
   Vector3 MaxSize { get; }
   IGeometry DeliveryArea { get; } //WKT polygon...
}

public class DeliveryProviderSelector
{
  private readonly IDeliveryProvider _providerA;
  private readonly IDeliveryProvider _providerB;

  //constructor and initialization
  
  public void bool TryChoose(OrderDelivery delivery, out IDeliveryProvider provider)
  {
    provider = null;

    // this provider doesn't have limits on delivery area and it's cheaper so it's first
    if(_providerA.MaxWeight >= delivery.Weight && _provider.MaxSize >= delivery.Size)
    {
      provider = _providerA;
      return true;
    }

    // _providerA won't take the package so make sure _providerB will
    if(_providerB.DeliveryArea.Contains(delivery.Address)) 
    {
      provider = _providerB;
      return true;
    }

    return false;
  }
}
```

All good and well, but due to an obvious downside of ``_providerA`` making far away deliveries only for small and lightweight packages and ``_providerB`` limiting the area it agrees to deliver the packages, a contract is made with another delivery provider who agrees to deliver heavier packages further but requires extra payment due to package damange insurance. Easy-peasy, you think, simply add another ``if`` statement and that's it. Next task!

A month later, another provider is added. Then another. And another. Each provider has unique requirements and conditions for accepting packages, such as different areas of responsibility or types of package contents. Unsurprisingly, the selection logic grows increasingly complex and harder to maintain.
What should we do about it?

## Responsibility trumps Chaos

This kind of problem is not something new and fortunately, there is a solution: *"Chain of Responsibility"* pattern. The idea behind the pattern is simple. Instead of having one giant mess of code with various conditional statements and interconnected methods, we divide the possible handlers of the problem into small discrete parts of code. Thus, each handler is responsible for a specific task or condition, and it either handles the request or passes it on to the next handler in the chain. The chain continues until the request is handled or until the end of the chain is reached, at which point the request is considered unhandled.
This would allow us to better adhere to the *open-close principle* if business logic changes and simplify the extension of the logic.  

In our case, we could apply this pattern by first adding a method ``CanHandleDelivery`` to ``IDeliveryProvider``. The method would *encapsulate* all the relevant conditions for a specific delivery provider and return true whether the provider would accept the delivery or not.

```cs
public interface IDeliveryProvider
{
   float MaxWeight { get; }
   Vector3 MaxSize { get; }
   IGeometry DeliveryArea { get; } //WKT polygon...

   bool CanHandleDelivery(OrderDelivery delivery, CustomerInfo context);
}
```

Then, we would simply iterate over all providers and choose the first that would accept the delivery.

```cs
public class DeliveryProviderSelector
{
  private readonly CustomerInfo _customer;
  private readonly IEnumerable<IDeliveryProvider> _providers;
  //constructor and initialization
  
  public void bool TryChoose(OrderDelivery delivery, out IDeliveryProvider chosenProvider)
  {
    chosenProvider = null;

    foreach(var provider in providers) {
      if(provider.CanHandleDelivery(delivery, _customer)) {
        chosenProvider = provider; 
        return true;
      }
    }

    return false;
  }
}
```

In addition to adhering to the open-closed principle, such approach promotes loose coupling and separation of concerns by dividing the problem into smaller, discrete parts of code. As a result, changes made to one handler are unlikely to affect the other handlers in the chain, making the code more maintainable and extensible.

As we have seen, proper design pattern usage can simplify your life, if the use-case is correctly recognized. That's it, for now. Now go write some code :)