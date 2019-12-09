---
title: Adventures of a C# dev in C++ land - dependency injection
date: 2019-12-07 01:15:48
tags:
  - C++
  - IOC/DI
  - C++ Templates
  - Programming
categories:
  - Programming
author: Michael Yarichuk  
top_img: top.jpg
cover: /2019/12/07/cpp-di-metaprogramming/cover.jpg
---
I stumbled upon [Boost.DI](https://boost-experimental.github.io/di/index.html) by accident and was instantly intrigued: for a developer used to C#, dependency injection during compilation time sounds crazy.

> Boost.DI uses C++ [template metaprogramming](https://en.wikipedia.org/wiki/Template_metaprogramming) to implement its functionality. If you are not familiar with it, take a look [here](https://www.fluentcpp.com/2017/06/02/write-template-metaprogramming-expressively/).

### Dependency Injection
Resolves dependencies during compilation sounds great as an optimization. The biggest problem with DI libraries in C# is performance.  
Just take a look at [IOC/DI library benchmarks](https://github.com/danielpalme/IocPerformance#basic-features)! 
> In case you are not familiar with DI, a good place to start reading *what* it is and *why* it is needed is those [slides from a conference talk](https://boost-experimental.github.io/di/cppcon-2018/#/). 

Everything seems great in "hello world"-ish sample code I found at Boost.DI repository, but I wondered if actually using it would be as easy as it seems. Probably because its been a while since I wrote C++, I quickly ran into an issue. I came up with the following test code that *should* have worked.
```c++ 
#include "di.hpp"
#include <string>
#include <iostream>

using namespace std;
namespace di = boost::di;

class igreeter
{
public:
    virtual void greet() = 0;
};

class greeter : igreeter
{
private:
    std::string message_;

public:
    greeter(std::string message)
        : message_(std::move(message))
    {
    }
    virtual void greet() override { std::cout << message_ << std::endl; }
};

class greeting_provider
{
private:
    igreeter& greeter_;
public:
    greeting_provider(igreeter& greeter)
        : greeter_(greeter)
    {
    }
    void greet() const { greeter_.greet(); }
};

int main()
{
   //define how to resolve dependencies
    const auto injector = make_injector
                          (
                              di::bind<igreeter>.to<greeter>(),
                              di::bind<greeting_provider>().to<greeting_provider>(),
                              di::bind<std::string>().to("Hello World!")
                          );
    //resolve all dependencies and construct an object
    const auto provider = injector.create<greeting_provider>();
    provider.greet();
    
    return 0;
}

```
It *should* have worked, but it failed with a cryptic error.
![Compilation Error](wtf.jpg)

After fiddling around with code, I solved it. Apparently, because of the arcane rules of C++ and even more arcane rules of template magick, so I needed to change the declaration of ``greeter`` and add 'public' to inheritance declaration.
```c++
class greeter : public igreeter //made the inheritance 'public' *facepalm*
{
private:
    std::string message_;

public:
    greeter(std::string message)
        : message_(std::move(message))
    {
    }
    virtual void greet() override { std::cout << message_ << std::endl; }
};
```
*It actually worked!*
![that feeling when your code finally works](rage.jpg)
  
Now, the only thing left in this case is to make ``message`` a named parameter, so not every class that has a ``std::string`` in its constructor will receive the value from the binding.  
A minor change was needed to ``greeter`` declaration:
```c++
auto msg = [] {}; //this is parameter 'name'

class greeter : public igreeter
{
private:
    std::string message_;

public:
    //specify that the first parameter has the name 'msg'
    BOOST_DI_INJECT_TRAITS((named = msg) std::string);
    greeter(std::string message)
        : message_(std::move(message))
    {
    }
    virtual void greet() override { std::cout << message_ << std::endl; }
};
```

And then, a minor change to dependency registration code:
```c++
const auto injector = 
  make_injector
  (
    di::bind<igreeter>.to<greeter>(),
    di::bind<greeting_provider>().to<greeting_provider>(),
    
    //this is a *named* parameter now
    di::bind<std::string>().named(msg).to("Hello World!")
  );
```


I continued to play around with the library, using it feels a bit awkward to me, but what do I know? I am not a C++ developer and it is likely I felt that way because I am used to the way C# IOC/DI libraries work.   
Considering the fact that C++ has no reflection, I never thought a library with such rich functionality is even possible. (and I am very happy to be wrong!)  
> If you want to see *how* rich is the functionality, take a look at [Boost.DI examples](https://boost-experimental.github.io/di/examples.html)

And if any C++ developer reads this, do tell me if I wrote anything wrong here :)