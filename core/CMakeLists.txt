cmake_minimum_required(VERSION 3.5)

project(core LANGUAGES CXX)

find_package(Python3 REQUIRED)
find_package(PythonLibs REQUIRED)

if(DEFINED ENV{CONDA_PREFIX})
    set(PYTHON_VERSION "3.8")
    set(Python3_EXECUTABLE "$ENV{CONDA_PREFIX}/bin/python")
    set(PYTHON_INCLUDE_DIRS "$ENV{CONDA_PREFIX}/include/python${PYTHON_VERSION}")
endif()

message(STATUS "python executeable " ${Python3_EXECUTABLE})
message(STATUS "python include path " ${PYTHON_INCLUDE_DIRS})

option(USE_CUDA "use cuda" OFF)

option(USE_MMAP "use mmap" OFF)

message(STATUS "USE_CUDA: ${USE_CUDA}")

set(CMAKE_BUILD_TYPE "Release")

file(READ "${CMAKE_CURRENT_LIST_DIR}/VERSION" CORE_VERSION)

message(STATUS "core " ${CORE_VERSION})

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread --std=c++17 -O2")
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DNOMINMAX -O2 /std:c++17 /arch:AVX /source-charset:utf-8")
else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread --std=c++17 -O2 -march=native")
endif()

message(STATUS "CMAKE_CXX_FLAGS" ${CMAKE_CXX_FLAGS})
set(FASTLLM_CXX_SOURCES src/fastllm.cpp src/device.cpp src/model.cpp src/executor.cpp
        src/devices/cpu/cpudevice.cpp src/devices/cpu/cpudevicebatch.cpp
        src/models/chatglm.cpp src/models/moss.cpp src/models/llama.cpp src/models/qwen.cpp src/models/basellm.cpp)

include_directories(include)
include_directories(include/utils)
include_directories(include/models)

if (USE_MMAP)
    add_compile_definitions(USE_MMAP)
endif()

if (USE_CUDA)
    set(CMAKE_CUDA_COMPILER /usr/local/cuda/bin/nvcc)
    enable_language(CUDA)
    add_compile_definitions(USE_CUDA)
    include_directories(include/devices/cuda)
    #message(${CMAKE_CUDA_IMPLICIT_LINK_DIRECTORIES})
    set(FASTLLM_CUDA_SOURCES src/devices/cuda/cudadevice.cpp src/devices/cuda/cudadevicebatch.cpp src/devices/cuda/fastllm-cuda.cu)
    set(FASTLLM_LINKED_LIBS ${FASTLLM_LINKED_LIBS} cublas)
    set(CMAKE_CUDA_ARCHITECTURES "75")
endif()

include(FetchContent)
set(FETCHCONTENT_QUIET off)
get_filename_component(fc_base "fc_base" REALPATH BASE_DIR "${CMAKE_BINARY_DIR}")
set(FETCHCONTENT_BASE_DIR ${fc_base})

message(STATUS "Fetching pybind11")
FetchContent_Declare(pybind11
  URL      https://github.com/pybind/pybind11/archive/refs/tags/v2.11.1.tar.gz
  URL_HASH      SHA256=d475978da0cdc2d43b73f30910786759d593a9d8ee05b1b6846d1eb16c6d2e0c
)
FetchContent_MakeAvailable(pybind11)

include_directories(${PYTHON_INCLUDE_DIRS})

file(GLOB FASTLLM_CXX_HEADERS include/**/*.h)
add_library(core MODULE src/pybinding.cpp ${FASTLLM_CXX_SOURCES} ${FASTLLM_CXX_HEADERS} ${FASTLLM_CUDA_SOURCES})
target_link_libraries(core PUBLIC pybind11::module ${FASTLLM_LINKED_LIBS})
pybind11_extension(core)
