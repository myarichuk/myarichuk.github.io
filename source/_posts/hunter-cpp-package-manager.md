---
title: Hunter, C++ package management made easy
date: 2020-01-01T14:35:18.677Z
tags:
  - C++
  - CMake
  - Package Management
  - OpenCV
categories:
  - Programming
  - C++
author: Michael Yarichuk
top_img: https://upload.wikimedia.org/wikipedia/commons/1/18/ISO_C%2B%2B_Logo.svg
cover: https://upload.wikimedia.org/wikipedia/commons/1/18/ISO_C%2B%2B_Logo.svg
---

In a previous post, I detailed involved and frustrating process I needed to go through to compile OpenCV as part of my application. You can see the post [in this link](/2019/12/25/opencv-and-cmake-in-cpp/).  
All the trouble I describe there could have been avoided by using [Hunter](https://github.com/cpp-pm/hunter) - a CMake based package manager for C++ projects. The idea is to have a sort-of repository of configuration and install scripts for different packages that would handle configuring 3rd-party dependencies (and their dependencies) for your project. Kinda like NuGet for .Net, but for C++. Just take a look at the [list of supported libraries](https://cpp-pm-hunter.readthedocs.io/en/latest/packages.html) - I think it is awesome!

## Show me the code!
It is really simple to use Hunter. First, you need to download *CMake* script from [here](https://raw.githubusercontent.com/hunter-packages/gate/master/cmake/HunterGate.cmake), include it *before* ``project()`` statement and then run ``HunterGate()`` initialization function of the package manager (The exact URL and SHA1 for the parameters can be taken from the [releases page](https://github.com/cpp-pm/hunter/releases) of Hunter). In this sample, I am going to install *OpenCV* and use it in a very simple test app.  
So, after all this, the root *CMakeLists.txt* should look similar to this:
```cmake
cmake_minimum_required (VERSION 3.8)

include("cmake/HunterGate.cmake")
HunterGate(
    URL "https://github.com/cpp-pm/hunter/archive/v0.23.240.tar.gz"
    SHA1 "ca19f3769e6c80cfdd19d8b12ba5102c27b074e0"
)

project ("HunterDemo")

hunter_add_package(OpenCV)
add_subdirectory ("PrintImageSizeUtil")
```

Then, the *CMakeLists.txt* in the subdirectory where we want to use *OpenCV* would look like this:
```cmake
cmake_minimum_required (VERSION 3.8)

find_package(OpenCV REQUIRED)
add_executable (PrintImageSizeUtil "PrintImageSizeUtil.cpp" "PrintImageSizeUtil.h")
target_link_libraries(PrintImageSizeUtil PRIVATE ${OpenCV_LIBS})
```

Well, thats it! It will take some time for Hunter to download and compile all the needed dependencies, but when CMake configuration stage completes, the test project will simply compile - the whole experience of using Hunter *almost* feels like using NuGet, it feels really effortless.
You can take a look at the source code of this demo at [this repository](https://github.com/myarichuk/hunterdemo).