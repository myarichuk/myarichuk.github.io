---
title: From Inheritance Hell to Component Heaven, the ECS Pattern
date: 2023-11-09 23:00:00
tags:
  - C#
  - Entity Component System
  - Design Patterns
  - Programming
  - Architecture
categories:
  - Programming
  - Design Patterns
---

## All you need is love and Object-Oriented, right? Right?

Object-Oriented programming is one of the most widely used programming paradigms. It's flexible, powerful, and has proven its worth over the years. However, as with any tool, there are situations where it might not be the best fit. In some cases, using an Object-Oriented approach can result in code that's hard to maintain and overly complex.

Let's say we're developing a game and we want to add magic weapons. It should be trivial to create a hierarchy of base classes for melee and ranged weapons, isn't it? So we might create classes that look like this:

```cs
public abstract class Armor
{
  public int ArmorRating { get; set; }
}

public abstract class Weapon
{
    public int Damage { get; set; }
}

public class MeleeWeapon : Weapon
{
    public int SwingSpeed { get; set; }    
}

public class RangedWeapon : Weapon
{
    public int Range { get; set; }
    public int AmmoCapacity { get; set; }
}
```

But what if we want to allow archers to use their bows in melee combat? And contrary to fantasy cliche, bows can actually be used in melee combat. Don't believe me? Check this out. 

<p align="center">
<iframe width="350" height="197" src="https://www.youtube.com/embed/_rb28VGRWbU" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen>
</iframe>
</p>

Anyway, if we're lucky enough to be using C++, or maybe we're stuck with it, we might be tempted to use multiple inheritance to inherit the bow class from both ``RangedWeapon`` and ``MeleeWeapon``. But let's face it, most other object-oriented languages don't play so nicely, and it's no coincidence that multiple inheritance isn't available in many of them. It can cause so many issues that it's usually not worth using it in non-trivial projects. As a result, we would probably end up creating something like this:
 
```cs
public class MeleeAndRangedWeapon : Weapon
{
    public int SwingSpeed { get; set; }
    public int Range { get; set; }
    public int AmmoCapacity { get; set; }
}

public class Bow: MeleeAndRangedWeapon
{
}
```

As we've seen, deep class hierarchies can quickly become unwieldy and inflexible when it comes to implementing complex game logic. Consider the example of adding shield bashing to our game - should a shield inherit from the ``Weapon`` or some ``Armor`` class, or perhaps from a new hybrid class? This kind of decision can quickly propagate throughout the codebase, making it harder to maintain and evolve.

To overcome these limitations, we can turn to the Entity Component System (ECS) pattern. Unlike traditional OO approaches, ECS separates an object's properties and behaviors into distinct components, which can be combined in flexible ways to create new game entities. This makes it much easier to add or modify game behavior without affecting other parts of the code, while also improving performance by optimizing for cache locality and minimizing memory overhead. In the next section, we'll dive into the details of how ECS works and how you can use it to build more flexible and performant game systems.

## So, what is this Entity Component System?

Entity Component System (ECS) is a design pattern originally invented for structuring game engine architectures. Unlike the "traditional" Object-Oriented paradigm, ECS separates game objects into three primary elements: entities, components, and systems.

Entities are simply unique IDs that don't have inherent behavior or data - they serve as an "anchor" to represent game objects to which data is attached. Components are where the data is stored and represent specific aspects of a game object's behavior, like its position in space or health. Systems are responsible for performing actions on one or more components of an entity and typically represent one aspect of functionality, such as rendering or physics simulation.

## ECS and memory locality

Another cool thing about ECS is that it makes use of the magic of memory locality to improve performance. What's memory locality, you ask?  
Well, in ECS, the components are often stored as simple structs in arrays, and the systems that operate on them iterate over those arrays. This might not sound too exciting, but it actually makes a big difference in how fast the code runs. When a computer reads data from memory, it doesn't just grab one byte at a time - it grabs a whole chunk of nearby bytes, known as a cache line or cache prefetching. This means that if your data is stored in a way that's scattered around in memory, the computer has to keep jumping around from cache line to cache line, which is slow.

But when your data is stored in a nice, contiguous array like in ECS, the computer can grab a whole bunch of it in one fell swoop, without having to keep jumping around. This makes iterating over the data much, much faster, which can be a big advantage when you're dealing with large amounts of data.  
>How big is the effect of data locality? See the post here {% post_link data-locality %}

## Enough with hand-waving! Show me the code :)

We will be using the awesome [DefaultEcs](https://github.com/Doraku/DefaultEcs) library for illustrating how ECS pattern can be used.  

In the following code, we create an entity, attach a component to it and fetch it. Note that we can attach multiple *different* components to an entity and do it at runtime.

```cs
using System;
using DefaultEcs;

record struct Position(float X, float Y)
{
}

class Program
{
    static void Main(string[] args)
    {
        //create an entity registry
        var world = new World();

        //create an entity and add components to it
        var entity = world.CreateEntity();
        entity.Set(new Position { X = 10f, Y = 20f });

        //get the position component from the entity
        ref Position position = ref entity.Get<Position>();

        Console.WriteLine($"Position: ({position.X}, {position.Y})");
    }
}
```

Now, let's add some interaction to the mix. We will add another component, ``Velocity`` which would indicate whether the entity has any velocity and not just a static object with a location.
We will also add ``VelocitySystem`` which will implement the movement of the entity - not all object should remain static, otherwiseit would make a game really boring. Essentially, velocity system will iterate over all entities that have position and velocity components and move them to a new location, depending on how much time has passed.  
>Notice how in the ``VelocitySystem`` constructor we choose which components have to be attached to an entity for this system to work. Kind of a filter.
```cs
record struct Velocity(float X, float Y)
{
    public Vector2 Value => new(X, Y);
}

record struct Position(float X, float Y)
{
}

public class VelocitySystem : AEntitySetSystem<float>
{
    public VelocitySystem(World world, IParallelRunner runner)
        : base(world.GetEntities()
                    .With<Velocity>()
                    .With<Position>()
                    .AsSet(), runner)
    {
    }

    protected override void Update(float elapsedTime, in Entity entity)
    {
        ref Velocity velocity = ref entity.Get<Velocity>();
        ref Position position = ref entity.Get<Position>();

        Vector2 offset = velocity.Value * elapsedTime;

        position.X += offset.X;
        position.Y += offset.Y;
    }
}
```
  
Now, in order to make things happen, we simply run the system. Not very complex, isn't it?

```cs
//initialize the runner, typically this is done once
var runner = new DefaultParallelRunner(Environment.ProcessorCount);

//in a typical use-case there will be more than one system running
var system = new VelocitySystem(world, runner);

//each time the system is updated, the logic will run.
//note that the state doesn't have to be a primitive like we have here
var timePassed = 0.5f;
system.Update(timePassed);
```

## So, when should I use ECS?

* Complexity: The decision to use ECS should depend on the nature of the project's data and the expected business logic. It's possible for even a simple project to benefit from using ECS if it involves processing large amounts of similar data. Also, ECS can be a good fit for projects that are expected to grow in complexity over time or have frequently changing requirements.
* Performance: ECS is not a silver bullet, it's benefits will be nullified by the cost of implementing and maintaining the ECS architecture, especially in smaller projects. Also, consider that performance benefits of ECS may only be noticeable in specific use cases, and that other architecture patterns may be more appropriate and will yield better results in other situations.
* Code maintainability: In general, ECS will yield more modular and reusable code, it can improve maintainability in the long term. However, the maintainability can and will depend on the size and complexity of the project, how much experience developers have and the quality of the design.
* Compatibility with existing codebase: The decision to use ECS should be based on the specific needs of the project, rather than compatibility with an existing codebase. Obviously, switching to such a different architectural pattern can be costly, but it's not necessarily a reason to avoid using ECS at all. It may be worth considering whether ECS can be gradually integrated into the codebase, perhaps through a hybrid architecture that combines ECS with other patterns.

## In conclusion

So, as you can see, ECS pattern can make your life easier in some occasions, but remember, it is not a silver bullet, it is just a tool. And if you do use it, keep your components small and concise and keep your systems focused on doing one thing and doing it well. That's it and now go write some code! :)
