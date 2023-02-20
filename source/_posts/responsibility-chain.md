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
---

## It all starts with a simple task

Imagine, one quiet morning, your boss comes to you and says, "Hey, our web shop is growing and we will be having more than one delivery provider now. Can you implement something that would select the best provider after a client pays for a delivery?".
After some back and forth about the criteria on how a delivery company should be selected - mostly by package size, weight and delivery company area, you set out to write the code. How hard can it be? Just write a few ``if`` statements, and that's it, right?

Some time later, you might end with a following solution. Rather simple but short and cincise. No need for over-engineering and [YAGNI](https://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it) is the right way to go, right?

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
    if(_providerA.MaxWeight >= delivery.Weight)
    {
      provider = _providerA;
      return true;
    }

    return false;
  }
}
```
