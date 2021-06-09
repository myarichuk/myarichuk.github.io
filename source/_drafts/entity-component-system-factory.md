---
title: Entity Component System & complex objects
date: 2021-01-15T20:39:38+02:00
tags:
  - C#
  - Game Programming
  - Design Patterns
categories:
  - Programming
author: Michael Yarichuk
top_img: http://www.graymatterdeveloper.com/2020/02/17/data-locality/benchmarkdotnet.png
cover: /2020/04/09/fun-string-create/string.svg
---

I've been toying around with an idea to write a game for quite a while, but despite being experienced working on large systems, I found myself at an interesting position of not even knowing where to begin.  
  
As it turned out, game programming has some interesting design patterns that I haven't seen in more enterprise-ey development, such as Entity Component System.

The idea of this pattern is to provide great flexibility at changing object fields behavior at runtime by allowing dynamic changes in properties and functionality during the object's lifetime.

## That's great, but what does it mean in practice?

