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
I stumbled upon [Boost.DI](https://boost-experimental.github.io/di/index.html) by accident and was instantly intrigued: for a developer used to C# (like me!), dependency injection during compilation time sounds... crazy.

C++ [template metaprogramming](https://en.wikipedia.org/wiki/Template_metaprogramming) is not a new feature, but in case you are not familiar with it, in short, it is a language within language that executes at compile time. If you *ARE* familiar with it, you can skip the next section.

### Template Metaprogramming
Here is a "classic" example that calculates numbers from fibonacci sequence during compilation time:
```c++
#include <iostream>

//recursive template
template <int n>
struct factorial {
	enum { value = n * factorial<n - 1>::value };
};

//'specialization' of the above template
template <>
struct factorial<0> {
	enum { value = 1 };
};

int main()
{
  //the values are calculated during compilation and not at runtime
  std::cout << factorial<0>::value << std::endl; //prints 1
  std::cout << factorial<3>::value << std::endl; //prints 6
  return 0;
}
```
So, what happens here?  
Each time a ``factorial<T>::value`` is evaluated in code, the C++ compiler 'unrolls' the code and substitutes the ``value`` for numeric result. All this happens *during* compilation.
> There is a great introduction to C++ templates and template metaprogramming [here](https://www.fluentcpp.com/2017/06/02/write-template-metaprogramming-expressively/).

### Dependency Injection
Let's get back to the topic: with the help of template metaprogramming, Boost.DI resolves dependencies during compilation. This sounds great as an optimization, especially for a C# developer like me. IOC/DI libraries in C# are useful and easy to use but it has a price tag: performance.  
Just take a look at [IOC/DI library benchmarks](https://github.com/danielpalme/IocPerformance#basic-features)! 
> In case you are not familiar with DI, a good place to start reading *what* it is and *why* it is needed is those [slides from a conference talk](https://boost-experimental.github.io/di/cppcon-2018/#/). 

In comparison to C#, resolving dependencies during compilation sounds especially awesome. However, not all is well with such approach: what if you make some sort of mistake? For example, after looking at samples, I came up with the following (simple) code that *should* have worked.
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
It *should* have worked, but it fails with very cryptic errors, which highlight a big problem with C++ template metaprogramming: cryptic errors which are not really "user-friendly".
![BoostDI Compilation Error](wtf.jpg)

Apparently, because of the arcane rules of C++ and even more arcane rules of template magick, I needed to change the declaration of ``greeter`` and add 'public' to inheritance declaration.
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
![rage](rage.jpg)
  
Now, the only thing left for this particular example is to make that ``message`` a named parameter, so not every class that has a ``std::string`` in its constructor will receive the value binding.  
Only a minor change was needed to ``greeter`` declaration:
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
const auto injector = make_injector
                      (
                         di::bind<igreeter>.to<greeter>(),
                         di::bind<greeting_provider>().to<greeting_provider>(),

                         //this is a *named* parameter now
                         di::bind<std::string>().named(msg).to("Hello World!")
                      );
```


All in all, using the library feels a bit awkward to me, but what do I know? I am not a C++ developer and in any case I am used to C# IOC/DI libraries...   
Considering the fact that C++ has no reflection, I never thought a library with such rich functionality is even possible. (and I am very happy to be wrong!)  
> If you want to see *how* rich is the functionality, take a look at [Boost.DI examples](https://boost-experimental.github.io/di/examples.html)

And if any C++ developer reads this, do tell me if I wrote anything wrong here :)