
---
layout: post
title: Tensorflow, C++, TIFF - All in one
author: Mahmoud Lababidi
desc: Tensorflow's C++ support has been admitedly sparse. We present a tutorial on using a Tensorflow shared library (.so) in a C++ application along with converting a TIFF file into a Tensor, the usable data type in Tensorflow.
keywords: multispectral, tensorflow, c++, cpp, library, tiff, gdal
published: True
---

Tensorflow's C++ support has been admitedly sparse. We present a tutorial on using a Tensorflow shared library (.so) in a C++ application along with converting a TIFF file into a Tensor, the usable data type in Tensorflow.

## 1 Clone and set up Tensorflow
First, clone Tensorflow and going through the [setup](https://www.tensorflow.org/install/install_sources) to install the prerequisites and install Tensorflow from source. 
Go through all the steps if you want to use Tensorflow with python.
Building models usually needs the python library.
If you already have the python Tensorflow library installed, you could omit the python portion, but many times, the library installed may not have been [customized for your CPU](https://stackoverflow.com/questions/41293077/how-to-compile-tensorflow-with-sse4-2-and-avx-instructions).

## 2 Configure and Build the Library
After going through the setup, run `./configure` within the tensorflow directory.
After configuring, run the `bazel` command below to build the Tensorflow C++ shared library (.so).
There's much [discussion](https://github.com/bazelbuild/bazel/issues/1920) on getting `bazel` to produce a static library, but for now hold your horses and use the shared library.
```
./configure
bazel build  //tensorflow:libtensorflow_cc.so
```

## 3 Copy the library and header files
Then Copy the following include headers and dynamic shared library to `/usr/local/lib` and `/usr/local/include`:
```
mkdir /usr/local/include/tensorflow
cp -r bazel-genfiles/ /usr/local/include/tensorflow/
cp -r tensorflow /usr/local/include/tensorflow/
cp -r third_party /usr/local/include/tensorflow/
cp -r bazel-bin/libtensorflow_cc.so /usr/local/lib/
```
## 4 Write an application
The following example application is from a [Tensorflow-CMake](https://github.com/cjweeks/tensorflow-cmake) project [example](https://github.com/cjweeks/tensorflow-cmake/blob/master/examples/external-project/main.cc).
While the motivation of this project we are 100% behind, some of the implementation is a bit confusing.
We provide our own CMake files below, but let's crawl before we walk.
```c
#include "tensorflow/core/public/session.h"
#include "tensorflow/core/platform/env.h"

/**
 * Checks if the given status is ok.
 * If not, the status is printed and the
 * program is terminated.
 */
void checkStatus(const tensorflow::Status& status) {
  if (!status.ok()) {
    std::cout << status.ToString() << std::endl;
    exit(1);
  }
}

int main(int argc, char** argv) {
    namespace tf = tensorflow;

    tf::Session* session;
    tf::Status status = tf::NewSession(tf::SessionOptions(), &session);
    if (!status.ok()) {
        std::cout << status.ToString() << std::endl;
        exit(1);
    }
    tf::GraphDef graph_def;
    status = ReadBinaryProto(tf::Env::Default(), "graph.pb", &graph_def);
    checkStatus(status);

    status = session->Create(graph_def);
    checkStatus(status);

    tf::Tensor x(tf::DT_FLOAT, tf::TensorShape()), y(tf::DT_FLOAT, tf::TensorShape());
    x.scalar<float>()() = 23.0;
    y.scalar<float>()() = 19.0;

    std::vector<std::pair<tf::string, tf::Tensor>> input_tensors = {{"x", x}, {"y", y}};
    std::vector<tf::Tensor> output_tensors;

    status = session->Run(input_tensors, {"z"}, {}, &output_tensors);
    checkStatus(status);
    
    tf::Tensor output = output_tensors[0];
    std::cout << "Success: " << output.scalar<float>() << "!" << std::endl;
    session->Close();
    return 0;
}
```

## 4b Compile without CMake

Just to verify things work without the full machinery of CMake, try compiling with the following:
```
g++ -std=c++11 -o example \
    -I/usr/local/include/tf \
    -I/usr/local/include/eigen3 \
    -g -Wall -D_DEBUG -Wshadow -Wno-sign-compare -w  \
    -L/usr/local/lib/libtensorflow_cc \
    `pkg-config --cflags --libs protobuf`  \
    -ltensorflow_cc example.cpp
```
After compilation, 

## 5 Use CMake, it's good for you

```cmake
# Locates the tensorFlow library and include directories.

include(FindPackageHandleStandardArgs)
unset(TENSORFLOW_FOUND)

find_path(TENSORFLOW_INCLUDE_DIR
        NAMES
        tensorflow
        third_party
        HINTS
        /usr/local/include/tensorflow
        /usr/include/tensorflow)

find_library(TENSORFLOW_LIBRARY NAMES tensorflow_cc
        HINTS
        /usr/lib
        /usr/local/lib)

# set TensorFlow_FOUND
find_package_handle_standard_args(TensorFlow DEFAULT_MSG TENSORFLOW_INCLUDE_DIR TENSORFLOW_LIBRARY)

# set external variables for usage in CMakeLists.txt
if(TENSORFLOW_FOUND)
    set(TENSORFLOW_LIBRARIES ${TENSORFLOW_LIBRARY})
    set(TENSORFLOW_INCLUDE_DIRS ${TENSORFLOW_INCLUDE_DIR})
endif()

# hide locals from GUI
mark_as_advanced(TENSORFLOW_INCLUDE_DIR TENSORFLOW_LIBRARY)
```

And the following is our `CMakeLists.txt` file. 
We'll use it to build our application that reads a TIFF and converts it into a Tensor to perform prediction on it.

```cmake
cmake_minimum_required(VERSION 2.8)
project(segmentation)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/bin)
set(SOURCE_FILES segment.cpp)
set(EXECUTABLE segmentation)

# Add modules
list(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake/Modules")

find_package(TensorFlow REQUIRED)

find_package(Protobuf REQUIRED)

find_package(OpenCV REQUIRED)
include_directories( ${OpenCV_INCLUDE_DIRS} )

find_package(GDAL REQUIRED)
include_directories(${GDAL_INCLUDE_DIRS})
link_libraries(${GDAL_LIBRARIES})


find_package(TIFF REQUIRED)
include_directories(${TIFF_INCLUDE_DIRS})
link_libraries(${TIFF_LIBRARIES})


find_package( PkgConfig )
pkg_check_modules( EIGEN3 REQUIRED eigen3 )
include_directories( ${EIGEN3_INCLUDE_DIRS} )

# set variables for external dependencies
set(EXTERNAL_DIR "${PROJECT_SOURCE_DIR}/external" CACHE PATH "Location where external dependencies will installed")
set(DOWNLOAD_LOCATION "${EXTERNAL_DIR}/src" CACHE PATH "Location where external projects will be downloaded")
mark_as_advanced(EXTERNAL_DIR DOWNLOAD_LOCATION)



set(PROJECT_INCLUDE_DIRS ${TENSORFLOW_INCLUDE_DIRS} ${EXTERNAL_DIR}/include)
set(PROJECT_LIBRARIES ${TENSORFLOW_LIBRARIES} ${OpenCV_LIBS})

include_directories(${PROJECT_INCLUDE_DIRS})
add_executable(${EXECUTABLE} ${SOURCE_FILES})
target_link_libraries(${EXECUTABLE} ${PROJECT_LIBRARIES})```