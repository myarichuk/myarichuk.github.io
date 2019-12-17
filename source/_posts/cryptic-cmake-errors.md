---
title: Yet Another (Cryptic) CMake Error
date: 2019-12-17T19:19:41.567Z
tags:
  - C++
  - CMake
  - WTF
categories:
  - Programming
  - WTF
author: Michael Yarichuk
top_img: https://upload.wikimedia.org/wikipedia/commons/1/13/Cmake.svg
cover: /2019/12/17/cryptic-cmake-errors/cover.png

---
I like CMake. I really do. But sometimes... it is frustrating.
The samples found online are understandable and easy to follow and everything compiles fine until you try something non-trivial. 
Like having an external project compile as part of your own so including and linking can be done easier.  
I have created a new CMake project, then added sources of [OpenCV](https://github.com/opencv/opencv) with as a git submodule.  
The resulting CMake file looked like this:

```cmake
cmake_minimum_required (VERSION 3.13)

project ("[some project name]")

add_subdirectory ("Libs/opencv") #location of OpenCV's source
add_subdirectory ("[project that should work with OpenCV]") #the project I want to add refernce to OpenCV
```

Now, seeing that OpenCV uses CMake, I assumed that it *should* work. And then CMake gods punished me for assuming too much - with the following error (actually more than one, there were 256 of them, but they look the same)
```log
1> [CMake] -- Found OpenCV: C:/[working folder]/build/x64-Debug (found version "4.2.0") 
1> [CMake] -- Configuring done
1> [CMake] CMake Error in Libs/opencv/modules/core/CMakeLists.txt:
1> [CMake]   Target "opencv_core" INTERFACE_INCLUDE_DIRECTORIES property contains path:
1> [CMake] 
1> [CMake]     "C:/[working folder]/build/x64-Debug"
1> [CMake] 
1> [CMake]   which is prefixed in the build directory.
```
> If you happen to know why does this error happen, do let me know. I'd really appreciate that!

This is as cryptic as it can be, worse yet, few people actually encountered this, and those who did made the error seem even more cryptic than it *should* be.
For example, [this](https://cmake.org/pipermail/cmake-developers/2013-March/018513.html) seems related, but it didn't help.  
And [this guy](https://cmake.org/pipermail/cmake/2016-June/063717.html) never received an answer to his valid question - why?!  
Now, I am not the first one to come up with such idea - [this guy](https://answers.opencv.org/question/217218/how-to-link-with-opencv-as-cmake-subdirectory/) thought of it first, but also, surprisingly never got an answer.   

After some investigation, I reached the conculsion that ``find_package`` in the other CMake subproject was to blame for this error - probably I missed something very obvious. The problem is - I have no idea *what* did I miss.  
This is how my subproject CMake looked like:
```cmake
cmake_minimum_required (VERSION 3.13)

add_executable (<app name> <source files>)

find_package(OpenCV REQUIRED) #this is why the error happened
include_directories(${OpenCV_INCLUDE_DIRS})

# ... the rest of CMake
```


Overall, after working for a few month with CMake (and being frustrated most of that time), I think it should require less arcane talents to properly handle CMake scripting. It is an excellent idea that more often than not works, but often requires crazy amount of time to figure out why things *don't work*.

Thus, the moral of this story can be summed up as:
![](meme.jpg)