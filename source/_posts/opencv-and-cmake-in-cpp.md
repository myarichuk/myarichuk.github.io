---
title: OpenCV + Visual Studio + CMake = Adventure time
date: 2019-12-25T21:57:38.448Z
tags:
  - C++
  - CMake
  - OpenCV
categories:
  - Programming
  - C++
author: Michael Yarichuk
top_img: https://upload.wikimedia.org/wikipedia/commons/1/13/Cmake.svg
cover: /2019/12/17/cryptic-cmake-errors/cover.png
---
In theory, configuring a popular library like OpenCV in CMake should be easy. In practice, sadly, it is anything BUT easy. Why? Let me show you...
> Note that I am not very knowledgeable in CMake, so it is possible I am doing something stupid. If so, it is even worse - as far as I understand the intent behind CMake, it should make developers lives easier, not harder.

This post is documentation of a frustrating journey of performing what *should* have been a trivial task - make dependency compile as part of a project. 
If you'd like to skip and take a look at working code, you can take a look at [this repository](https://github.com/myarichuk/OpenCV.CMake).

## Attempt 1
According to online tutorials and CMake documentation, a good way (best practice?) to download OpenCV and configure it as a part of the project is to use ``FetchContent``. Alright, I thought to myself and wrote the following:
```cmake
cmake_minimum_required (VERSION 3.14)
project ("CMake Reference Samples")

include(FetchContent)
FetchContent_Declare(  
	opencv  
	GIT_REPOSITORY https://github.com/opencv/opencv.git
	GIT_TAG        master
)

FetchContent_MakeAvailable(opencv)

add_subdirectory ("OpenCV.CMake")
```

And then, in the project subfolder that is supposed to use OpenCV, I wrote the following:
```cmake
cmake_minimum_required (VERSION 3.11)

find_package(OpenCV CONFIG REQUIRED)

add_executable (OpenCV.CMake "OpenCV.CMake.cpp" "OpenCV.CMake.h")

add_dependencies(OpenCV.CMake opencv)
target_include_directories(OpenCV.CMake PRIVATE ${OpenCV_INCLUDE_DIRS})
target_link_libraries(OpenCV.CMake PRIVATE ${OpenCV_LIBS})
```
  
And then, generation of CMake build scripts... fails. With lots of errors that look like this:
```csv
CMake Error in out/build/x64-Debug/_deps/opencv-src/modules/core/CMakeLists.txt:
  Target "opencv_core" INTERFACE_INCLUDE_DIRECTORIES property contains path:

    "E:/projects/OpenCV.CMake/out/build/x64-Debug"

  which is prefixed in the build directory.				
```

## Attempt 1.5
I thought to myself, I must've done something wrong. First thing that I did, I added ``set(FETCHCONTENT_QUIET off)`` before calling ``Fetch_Content()`` - in theory, that would allow to see the exact steps it took to download and configure OpenCV. Well, I did see the steps.
```cmake
1> [CMake] -- Populating opencv
1> [CMake] -- Configuring done
1> [CMake] -- Generating done
1> [CMake] -- Build files have been written to: E:/projects/OpenCV.CMake/out/build/x64-Debug/external/opencv-subbuild
1> [CMake] [0/6] Performing update step for 'opencv-populate'
1> [CMake] From https://github.com/opencv/opencv
1> [CMake]    d76f245..7b12cbd  master     -> origin/master
1> [CMake]    9572895..7b28d5b  3.4        -> origin/3.4
1> [CMake] First, rewinding head to replay your work on top of it...
1> [CMake] Fast-forwarded master to origin/master.
1> [CMake] [2/6] No configure step for 'opencv-populate'
1> [CMake] [3/6] No build step for 'opencv-populate'
1> [CMake] [4/6] No install step for 'opencv-populate'
1> [CMake] [5/6] No test step for 'opencv-populate'
1> [CMake] [6/6] Completed 'opencv-populate'
```

That is nice, but it didn't solve the issue, the problem apparently was in configure stage of CMake because I saw the same errors as above. 
Reading on the internet and trying various stuff didn't help much. For example, one solution offered by forums was to set target properties of the target like this:
```cmake
set_target_properties(opencv PROPERTIES
  INTERFACE_INCLUDE_DIRECTORIES "${OpenCV_INCLUDE_DIRS}"
  INTERFACE_LINK_LIBRARIES "${OpenCV_LIBS}")
```

This looked like it *should* work, but after running ``FetchContent_MakeAvailable()`` on **opencv**, it apparently wasn't recognized as target (which seemed like it *should* have recognized).

## Some investigation time!
I decided to dig depeer. According to CMake sources (found [here](https://github.com/Kitware/CMake/blob/966a9eece32f55fab479ea6997dea68e1c2d6212/Modules/FetchContent.cmake#L1038)), I am seeing the following implenentation of ``FetchContent_MakeAvailable``:
```cmake
macro(FetchContent_MakeAvailable)

  foreach(contentName IN ITEMS ${ARGV})
    string(TOLOWER ${contentName} contentNameLower)
    FetchContent_GetProperties(${contentName})
    if(NOT ${contentNameLower}_POPULATED)
      FetchContent_Populate(${contentName})

      # Only try to call add_subdirectory() if the populated content
      # can be treated that way. Protecting the call with the check
      # allows this function to be used for projects that just want
      # to ensure the content exists, such as to provide content at
      # a known location.
      if(EXISTS ${${contentNameLower}_SOURCE_DIR}/CMakeLists.txt)
        add_subdirectory(${${contentNameLower}_SOURCE_DIR}
                         ${${contentNameLower}_BINARY_DIR})
      endif()
    endif()
  endforeach()

endmacro()
```

Since it uses ``FetchContent_Populate()`` to do its job, I looked at its implementation as well. There I saw something interesting.
```cmake
function(__FetchContent_directPopulate contentName)

  # >> skipping some code for clarity's sake
  
  # Create and build a separate CMake project to carry out the population.
  configure_file("${CMAKE_CURRENT_FUNCTION_LIST_DIR}/FetchContent/CMakeLists.cmake.in"
                 "${ARG_SUBBUILD_DIR}/CMakeLists.txt")
  execute_process(
    COMMAND ${CMAKE_COMMAND} ${generatorOpts} .
    RESULT_VARIABLE result
    ${outputOptions}
    WORKING_DIRECTORY "${ARG_SUBBUILD_DIR}"
  )
  if(result)
    if(capturedOutput)
      message("${capturedOutput}")
    endif()
    message(FATAL_ERROR "CMake step for ${contentName} failed: ${result}")
  endif()
  execute_process(
    COMMAND ${CMAKE_COMMAND} --build .
    RESULT_VARIABLE result
    ${outputOptions}
    WORKING_DIRECTORY "${ARG_SUBBUILD_DIR}"
  )
  if(result)
    if(capturedOutput)
      message("${capturedOutput}")
    endif()
    message(FATAL_ERROR "Build step for ${contentName} failed: ${result}")
  endif()

endfunction()

```

Apparently, it dynamically creates another CMakeLists.txt and builds it as a child process of current CMake. And the template for the generated CMakeLists.txt looks like this:

```cmake
# Distributed under the OSI-approved BSD 3-Clause License.  See accompanying
# file Copyright.txt or https://cmake.org/licensing for details.

cmake_minimum_required(VERSION ${CMAKE_VERSION})

# We name the project and the target for the ExternalProject_Add() call
# to something that will highlight to the user what we are working on if
# something goes wrong and an error message is produced.

project(${contentName}-populate NONE)

include(ExternalProject)
ExternalProject_Add(${contentName}-populate
                    ${ARG_EXTRA}
                    SOURCE_DIR          "${ARG_SOURCE_DIR}"
                    BINARY_DIR          "${ARG_BINARY_DIR}"
                    CONFIGURE_COMMAND   ""
                    BUILD_COMMAND       ""
                    INSTALL_COMMAND     ""
                    TEST_COMMAND        ""
                    USES_TERMINAL_DOWNLOAD  YES
                    USES_TERMINAL_UPDATE    YES
)
```
This means that ``FetchContent_Populate()`` simply executes ExternalProject_Add but with ``CONFIGURE_COMMAND``, ``BUILD_COMMAND`` and ``INSTALL_COMMAND`` disabled (yes, I know this is in CMake documentation, but I find it easier to read code than documentation - it is less boring this way!)  
After looking at OpenCV CMake scripts, it seemed that those stages *should* be executed before adding OpenCV as subdirectory - its CMake seems to pull dependencies during build time, not configure time.

## Attempt 2
As a conclusion, I needed a way to run configure, build and install commands of ``ExternalProject_Add()`` at main CMake configuration step. [This Stackoverflow answer](https://stackoverflow.com/a/23570741/320103) seemed promising. I changed it a little to use ``GIT_REPOSITORY`` instead of ``URL`` command, so what I used looked like this:
```cmake
# This function is used to force a build on a dependant project at cmake configuration phase.
function (build_external_project target prefix url branch) #FOLLOWING ARGUMENTS are the CMAKE_ARGS of ExternalProject_Add

    set(trigger_build_dir ${CMAKE_BINARY_DIR}/force_${target})

    #mktemp dir in build tree
    file(MAKE_DIRECTORY ${trigger_build_dir} ${trigger_build_dir}/build)

    #generate false dependency project
    set(CMAKE_LIST_CONTENT "
        cmake_minimum_required(VERSION 3.0)
        project(ExternalProjectCustom)
        include(ExternalProject)
        ExternalProject_add(${target}
            PREFIX ${prefix}/${target}
            GIT_REPOSITORY  ${url}
            GIT_TAG ${branch}
            CMAKE_ARGS ${ARGN})

        add_custom_target(trigger_${target})
        add_dependencies(trigger_${target} ${target})
    ")

    file(WRITE ${trigger_build_dir}/CMakeLists.txt "${CMAKE_LIST_CONTENT}")

    execute_process(COMMAND ${CMAKE_COMMAND} ..
        WORKING_DIRECTORY ${trigger_build_dir}/build
        )
    execute_process(COMMAND ${CMAKE_COMMAND} --build .
        WORKING_DIRECTORY ${trigger_build_dir}/build
        )

endfunction()
```

Then in the root *CMakeLists.txt* I simply called this function
```cmake
include(cmake/build_external_project.cmake)
build_external_project(opencv ${CMAKE_CURRENT_BINARY_DIR}/external https://github.com/opencv/opencv.git master)
```
After some time, to my mild suprise, it succeeded!
```log
1> [CMake] -- Configuring done
1> [CMake] -- Generating done
1> [CMake] -- Build files have been written to: E:/projects/OpenCV.CMake/out/build/x64-Debug
1> [CMake] 
1> Extracted includes paths.
1> Extracted CMake variables.
1> Extracted source files and headers.
1> Extracted code model.
1> CMake generation finished.
```

The next step was to configure the subdirectory that will actually use OpenCV for compilation and linking. In the end, the subdirectory CMakeLists.txt looked like this:
```cmake
cmake_minimum_required (VERSION 3.14)

find_package(OpenCV REQUIRED)

add_executable (OpenCVCMake "OpenCV.CMake.cpp" "OpenCV.CMake.h")

target_include_directories(OpenCVCMake PRIVATE ${OpenCV_INCLUDE_DIRS})
target_link_libraries(OpenCVCMake PRIVATE ${OpenCV_LIBS})
```

I try to compile and... run into another issue
```log
Error		'external/opencv-build/lib/opencv_dnn420d.lib', needed by 'bin/OpenCVCMake.exe', missing and no known rule to make it
```

Then I looked again at CMake log and noticed this curious line:
```cmake
1> [CMake] -- Building for: Visual Studio 16 2019
```
Then I noticed that in the build folder there is lots of **vcxproj** files were being generated. Then I modified the function to invoke current CMake generator, in this case it is ``Ninja`` - the default generator for Visual Studio CMake projects.
I adjusted the configure step in ``build_external_project()`` to include the current CMake generator - notice the ``-G ${CMAKE_GENERATOR}`` addition.
```cmake
    execute_process(COMMAND ${CMAKE_COMMAND} -G ${CMAKE_GENERATOR} ..
        WORKING_DIRECTORY ${trigger_build_dir}/build
        )
```

And then I saw this:
```cmake
1> [CMake] CMake Error: Error: generator : Ninja
1> [CMake] Does not match the generator used previously: Visual Studio 16 2019
1> [CMake] Either remove the CMakeCache.txt file and CMakeFiles directory or choose a different binary directory.
```
After a bit of facepalm I deleted all possible temporary folders (which by now accumulted to 4GB!), and updated the root CMakeLists.txt, so the configure/build would take less time.

The resulting root CMakeLists.txt looked like this:
```cmake

cmake_minimum_required (VERSION 3.14)
project ("OpenCV CMake Reference Sample")

include(cmake/build_external_project.cmake)
build_external_project(
	opencv 
	${CMAKE_CURRENT_BINARY_DIR}/external 
	https://github.com/opencv/opencv.git 
	master
	-DBUILD_PERF_TESTS:BOOL=FALSE
    -DBUILD_DOCS:BOOL=FALSE
    -DBUILD_EXAMPLES:BOOL=FALSE
    -DBUILD_TESTS:BOOL=FALSE
    -DBUILD_SHARED_LIBS:BOOL=FALSE	
	  -DBUILD_WITH_DEBUG_INFO=OFF
    -DBUILD_PACKAGE:BOOL=OFF
	  -DCMAKE_BUILD_TYPE:STRING=Release
	  -DCMAKE_INSTALL_PREFIX:PATH=${CMAKE_CURRENT_BINARY_DIR}/opencv
	)

add_subdirectory ("OpenCV.CMake")

```

I re-generated CMake cache, and saw... another issue. 
```log
 [CMake] CMake Warning at out/build/x64-Debug/opencv/OpenCVConfig.cmake:176 (message):
1> [CMake]   Found OpenCV Windows Pack but it has no binaries compatible with your
1> [CMake]   configuration.
1> [CMake] 
1> [CMake]   You should manually point CMake variable OpenCV_DIR to your build of OpenCV
1> [CMake]   library.
1> [CMake] Call Stack (most recent call first):
1> [CMake]   OpenCV.CMake/CMakeLists.txt:3 (find_package)
1> [CMake] CMake Error at OpenCV.CMake/CMakeLists.txt:3 (find_package):
1> [CMake]   Found package configuration file:
1> [CMake] 
1> [CMake]     E:/projects/OpenCV.CMake/out/build/x64-Debug/opencv/OpenCVConfig.cmake
1> [CMake] 
1> [CMake]   but it set OpenCV_FOUND to FALSE so package "OpenCV" is considered to be
1> [CMake]   NOT FOUND.
```

After I specified CMake install prefix, in the 'opencv' directory of output, the compiled and *installed* OpenCV appeared. I was getting closer.
![](OpenCV_Install_Output.PNG)

The last thing (is it?) I needed to do is to update the subdirectory CMakeLists.txt and change the ``find_package()`` to ``find_library()``, so the resulting line looked like this:
```cmake
find_library(OpenCV CONFIG REQUIRED)
```
Apparently, it was not the last thing. The project compiled successfully, but there was no include folders - after some fiddling, apparently the ``find_library()`` doesn't set ``OpenCV_INCLUDE_DIRS`` and ``OpenCV_LIBS`` variables as it should. 

After looking through *OpenCVConfig.cmake* I saw a very interesting part:
```cmake
if(NOT DEFINED OpenCV_STATIC)
  # look for global setting
  if(NOT DEFINED BUILD_SHARED_LIBS OR BUILD_SHARED_LIBS)
    set(OpenCV_STATIC OFF)
  else()
    set(OpenCV_STATIC ON)
  endif()
endif()
```

Since I wanted to link statically, I set ``BUILD_SHARED_LIBS`` before "finding" OpenCV package.
```cmake
set(BUILD_SHARED_LIBS OFF)
find_package(OpenCV REQUIRED)
```

That finally got OpenCV found by the CMake
```log
1> [CMake] [5/5] Completed 'opencv'
1> [CMake] -- OpenCV ARCH: x64
1> [CMake] -- OpenCV RUNTIME: vc16
1> [CMake] -- OpenCV STATIC: ON
1> [CMake] -- Found OpenCV 4.2.0 in E:/projects/OpenCV.CMake/out/build/x64-Debug/opencv/x64/vc16/staticlib
1> [CMake] include => E:/projects/OpenCV.CMake/out/build/x64-Debug/opencv/include
1> [CMake] link => opencv_calib3d;opencv_core;opencv_dnn;opencv_features2d;opencv_flann;opencv_gapi;opencv_highgui;opencv_imgcodecs;opencv_imgproc;opencv_ml;opencv_objdetect;opencv_photo;opencv_stitching;opencv_video;opencv_videoio
1> [CMake] -- Configuring done
1> [CMake] -- Generating done
1> [CMake] -- Build files have been written to: E:/projects/OpenCV.CMake/out/build/x64-Debug
1> [CMake] 
1> Extracted includes paths.
1> Extracted CMake variables.
1> Extracted source files and headers.
1> Extracted code model.
1> CMake generation finished.
```

That was it! Finally, the project worked. I was able to compile and run very basic testing code:
```c++
#include <iostream>
#include <opencv2/imgcodecs.hpp>

using namespace std;

int main()
{
	const auto img = cv::imread("e:\\Capture.JPG");
	cout << "image size: " << img.cols << "x" << img.rows << endl;
	return 0;
}
```
You can download the full project from [its repository](https://github.com/myarichuk/OpenCV.CMake).