# This file is part of Leela Zero.
# Copyright (C) 2017 Marco Calignano
# Copyright (C) 2017-2018 Gian-Carlo Pascutto and contributors
# Leela Zero is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.
# Leela Zero is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
# You should have received a copy of the GNU General Public License
# along with Leela Zero.  If not, see <http://www.gnu.org/licenses/>.

cmake_minimum_required(VERSION 3.1)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/Modules/")

project(leelaz)
#add_subdirectory(gtest EXCLUDE_FROM_ALL) # We don't want to install gtest, exclude it from `all`

# Required Packages
set(Boost_MIN_VERSION "1.58.0")
set(Boost_USE_MULTITHREADED ON)
find_package(Boost 1.58.0 REQUIRED program_options)
find_package(Threads REQUIRED)
find_package(ZLIB REQUIRED)
find_package(OpenCL REQUIRED)
# We need OpenBLAS for now, because we make some specific
# calls. Ideally we'd use OpenBLAS is possible and fall back to
# not doing those calls if it's not present.
if(NOT APPLE)
  set(BLA_VENDOR OpenBLAS)
endif()
find_package(BLAS REQUIRED)
find_path(BLAS_INCLUDE_DIRS openblas_config.h
  /usr/include
  /usr/local/include
  /usr/include/openblas
  /opt/OpenBLAS/include
  /usr/include/x86_64-linux-gnu
  $ENV{BLAS_HOME}/include)
find_package(Qt5Core)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED on)

# See if we can set optimization flags as expected.
if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
  set(GccSpecificFlags 1)
elseif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "AppleClang")
  set(GccSpecificFlags 1)
elseif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
  set(GccSpecificFlags 1)
elseif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Intel")
  set(GccSpecificFlags 0)
elseif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
  set(GccSpecificFlags 0)
endif()

if(NOT CMAKE_CONFIGURATION_TYPES AND NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE RELEASE)
endif(NOT CMAKE_CONFIGURATION_TYPES AND NOT CMAKE_BUILD_TYPE)

if(GccSpecificFlags)
  set(GCC_COMPILE_FLAGS "-Wall -Wextra -ffast-math -flto -march=native")
  set(GCC_DISABLED_WARNING_COMPILE_FLAGS "-Wno-ignored-attributes -Wno-maybe-uninitialized \
      -Wno-mismatched-tags")
  set(GCC_FLAGS "${GCC_COMPILE_FLAGS} ${GCC_DISABLED_WARNING_COMPILE_FLAGS}")
  set(CMAKE_CXX_FLAGS_DEBUG "${GCC_FLAGS} -g -Og")
  set(CMAKE_CXX_FLAGS_RELEASE "${GCC_FLAGS} -g -O3 -DNDEBUG")
  set(CMAKE_EXE_LINKER_FLAGS "-flto -g")
endif(GccSpecificFlags)

if(USE_CPU_ONLY)
  add_definitions(-DUSE_CPU_ONLY)
endif()
if(USE_HALF)
  add_definitions(-DUSE_HALF)
endif()

set(IncludePath "${CMAKE_CURRENT_SOURCE_DIR}/src")
set(SrcPath "${CMAKE_CURRENT_SOURCE_DIR}/src")

include_directories(${IncludePath})
include_directories(${Boost_INCLUDE_DIRS})
include_directories(${OpenCL_INCLUDE_DIRS})
include_directories(${ZLIB_INCLUDE_DIRS})

if((UNIX AND NOT APPLE) OR WIN32)
    include_directories(${BLAS_INCLUDE_DIRS})
endif()
if(APPLE)
    include_directories("/System/Library/Frameworks/Accelerate.framework/Versions/Current/Headers")
endif()

set(leelaz_MAIN "${SrcPath}/Leela.cpp")
file(GLOB leelaz_SRC "${SrcPath}/*.cpp")
list(REMOVE_ITEM leelaz_SRC ${leelaz_MAIN})

# Reuse for leelaz and gtest
add_library(objs OBJECT ${leelaz_SRC})

add_executable(leelaz $<TARGET_OBJECTS:objs> ${leelaz_MAIN})

target_link_libraries(leelaz ${Boost_LIBRARIES})
target_link_libraries(leelaz ${BLAS_LIBRARIES})
target_link_libraries(leelaz ${OpenCL_LIBRARIES})
target_link_libraries(leelaz ${ZLIB_LIBRARIES})
target_link_libraries(leelaz ${CMAKE_THREAD_LIBS_INIT})
install(TARGETS leelaz DESTINATION bin)

if(Qt5Core_FOUND)
    if(NOT Qt5Core_VERSION VERSION_LESS "5.3.0")
        add_subdirectory(autogtp)
        add_subdirectory(validation)
    else()
        message(WARNING "Qt ${Qt5Core_VERSION} is found but does not met required version 5.3.0, \
        build target for `autogtp` and `validation` is disabled.")
    endif()
else()
    message(WARNING "Qt is not found, build for `autogtp` and `validation` is disabled")
endif()

# Google Test below
#file(GLOB tests_SRC "${SrcPath}/tests/*.cpp")

#add_executable(tests ${tests_SRC} $<TARGET_OBJECTS:objs>)
#if(GccSpecificFlags)
#  target_compile_options(tests PRIVATE "-Wno-unused-variable")
#endif()

#target_link_libraries(tests ${Boost_LIBRARIES})
#target_link_libraries(tests ${BLAS_LIBRARIES})
#target_link_libraries(tests ${OpenCL_LIBRARIES})
#target_link_libraries(tests ${ZLIB_LIBRARIES})
#target_link_libraries(tests gtest_main ${CMAKE_THREAD_LIBS_INIT})
