#/***************************************************************************
# *
# *  @license
# *  Copyright (C) Codeplay Software Limited
# *  Licensed under the Apache License, Version 2.0 (the "License");
# *  you may not use this file except in compliance with the License.
# *  You may obtain a copy of the License at
# *
# *      http://www.apache.org/licenses/LICENSE-2.0
# *
# *  For your convenience, a copy of the License has been included in this
# *  repository.
# *
# *  Unless required by applicable law or agreed to in writing, software
# *  distributed under the License is distributed on an "AS IS" BASIS,
# *  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# *  See the License for the specific language governing permissions and
# *  limitations under the License.
# *
# *  SYCL-BLAS: BLAS implementation using SYCL
# *
# *  @filename CMakeLists.txt
# *
# **************************************************************************/
cmake_minimum_required(VERSION 3.4.3)
project(sycl-blas VERSION 0.1.0 LANGUAGES CXX)

if(POLICY CMP0074)
  cmake_policy(SET CMP0074 NEW)
endif()

if(EXISTS ${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)
  include(${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)
  conan_basic_setup(TARGETS NO_OUTPUT_DIRS)
  # We copy the pkgconfig files here
  list(APPEND CMAKE_PREFIX_PATH ${CMAKE_BINARY_DIR})
endif()

set(BUILD_SHARED_LIBS ON CACHE BOOL "")
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_POSITION_INDEPENDENT_CODE ${BUILD_SHARED_LIBS})

if(DEFINED SYSTEM_BLAS_ROOT)
  message(DEPRECATION
    "SYSTEM_BLAS_ROOT is deprecated. Add the path to the reference BLAS to CMAKE_PREFIX_PATH instead")
  list(INSERT CMAKE_PREFIX_PATH 0 ${SYSTEM_BLAS_ROOT})
endif()

list(APPEND CMAKE_MODULE_PATH
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules
  ${CMAKE_CURRENT_SOURCE_DIR}/external/computecpp-sdk/cmake/Modules
)

# This turns the long file names that are generated into short versions using a hash
# This is required to build SYCL-BLAS on Windows and on WSL since Windows doesn't
# have good support for long file names
option(SYCLBLAS_USE_SHORT_NAMES "Whether to use short names for generated files" OFF)
if (WIN32)
  # On WIN32 we can force this option to be true, but when building on WSL this
  # must be set in the command line
  set(SYCLBLAS_USE_SHORT_NAMES ON CACHE BOOL "Enable short names for generated files in Windows" FORCE)
endif()

# Setup datatypes, workgroup sizes and other options.
# NB: This has to be included before CmakeFunctionHelper as it declares various options.
include(ConfigureSYCLBLAS)
include(SYCL)
find_package(PythonInterp REQUIRED)

if (MSVC)
  # The device compiler needs C++14 to parse the Windows headers
  set(CMAKE_CXX_STANDARD 14)
  set(BUILD_SHARED_LIBS FALSE CACHE BOOL
    "Force SYCL-BLAS to be built as a static library on Windows"
    FORCE
  )
endif()

# By default, tall and skinny Gemm is enabled (for better performance)
option(GEMM_TALL_SKINNY_SUPPORT "Whether to enable tall and skinny Gemm" ON)
# By default vectorization in gemm kernels is enabled as it imrpove the performance on all Devices.
option(GEMM_VECTORIZATION_SUPPORT "Whether to enable vectorization in Gemm kernels" ON)

add_definitions(-DCL_TARGET_OPENCL_VERSION=220)

set(CBLAS_INCLUDE ${CMAKE_CURRENT_SOURCE_DIR}/external/cblas/include)
set(COMPUTECPP_SDK_INCLUDE ${CMAKE_CURRENT_SOURCE_DIR}/external/computecpp-sdk/include)
set(SYCLBLAS_GENERATED_SRC ${CMAKE_CURRENT_BINARY_DIR}/generated_src)
set(SYCLBLAS_INCLUDE ${CMAKE_CURRENT_SOURCE_DIR}/include)
set(SYCLBLAS_COMMON_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/common/include)
set(SYCLBLAS_SRC ${CMAKE_CURRENT_SOURCE_DIR}/src)
set(SYCLBLAS_SRC_GENERATOR ${CMAKE_CURRENT_SOURCE_DIR}/python_generator)
list(APPEND THIRD_PARTIES_INCLUDE ${CBLAS_INCLUDE} ${SYCL_INCLUDE_DIRS} ${COMPUTECPP_SDK_INCLUDE})

if(IMGDNN_DIR)
  add_definitions(-DIMGDNN_LIBRARY)
  find_package(IMGDNN REQUIRED)
  list(APPEND THIRD_PARTIES_INCLUDE ${IMGDNN_INCLUDE_DIRS})
endif()



# CmakeFunctionHelper has to be included after any options that it depends on are declared.
# These include:
# * TARGET
# * GEMM_TALL_SKINNY_SUPPORT
# * GEMM_VECTORIZATION_SUPPORT
# * BLAS_DATA_TYPES
# * NAIVE_GEMM
include(CmakeFunctionHelper)

add_subdirectory(src)
build_library(sycl_blas)

if (WIN32)
  # On Windows, all symbols must be resolved at link time for DLLs.
  # The sycl_blas target is just a collection of other objects so
  # the linked libraries are not going to be propagated to this target.
  # This requires manual linking against SYCL on Windows.
  if(is_computecpp)
    target_link_libraries(sycl_blas PRIVATE ComputeCpp::ComputeCpp)
  elseif(is_dpcpp)
    target_link_libraries(sycl_blas PRIVATE DPCPP::DPCPP)
  endif()
endif()
if(is_computecpp)
  set_target_properties(sycl_blas PROPERTIES
    VERSION ${PROJECT_VERSION}
    INTERFACE_LINK_LIBRARIES ComputeCpp::ComputeCpp
    INTERFACE_INCLUDE_DIRECTORIES "${SYCLBLAS_INCLUDE};${COMPUTECPP_SDK_INCLUDE};${SYCL_INCLUDE_DIRS}"
  )
elseif(is_dpcpp)
  set_target_properties(sycl_blas PROPERTIES
    VERSION ${PROJECT_VERSION}
    INTERFACE_LINK_LIBRARIES DPCPP::DPCPP
    INTERFACE_INCLUDE_DIRECTORIES "${SYCLBLAS_INCLUDE};${COMPUTECPP_SDK_INCLUDE};${SYCL_INCLUDE_DIRS}"
  )
  add_sycl_to_target(TARGET sycl_blas SOURCES)
endif()

if(IMGDNN_DIR)
  target_link_libraries(sycl_blas PUBLIC IMGDNN::IMGDNN)
endif()

include(GNUInstallDirs)
install(TARGETS sycl_blas
  RUNTIME
    DESTINATION ${CMAKE_INSTALL_BINDIR}
    COMPONENT sycl_blas
  LIBRARY
    DESTINATION ${CMAKE_INSTALL_LIBDIR}
    COMPONENT sycl_blas
  ARCHIVE
    DESTINATION ${CMAKE_INSTALL_LIBDIR}
    COMPONENT sycl_blas
)
install(DIRECTORY ${SYCLBLAS_INCLUDE}/
  DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/sycl_blas"
  COMPONENT sycl_blas
  FILES_MATCHING PATTERN "*.h"
)

option(BLAS_ENABLE_TESTING "Whether to enable testing" ON)
option(ENABLE_EXPRESSION_TESTS "Whether to build expression tree fusion tests" OFF)

if(${BLAS_ENABLE_TESTING})
  enable_testing()
  add_subdirectory(test)
endif()

option(BLAS_ENABLE_BENCHMARK "Whether to enable benchmarking" ON)
option(BLAS_VERIFY_BENCHMARK "Whether to verify the results of benchmarks" ON)
option(BUILD_CLBLAST_BENCHMARKS "Whether to build clBLAST benchmarks" OFF)
option(BUILD_CLBLAS_BENCHMARKS "Whether to build clBLAS benchmarks" OFF)
option(BUILD_ACL_BENCHMARKS "Whether to build ARM Compute Library benchmarks" OFF)
option(BUILD_EXPRESSION_BENCHMARKS "Whether to build the expression benchmarks" OFF)
option(BLAS_BUILD_SAMPLES "Whether to build SYCL-BLAS samples" ON)

if(${BLAS_ENABLE_BENCHMARK})
  add_subdirectory(benchmark)
endif()

if (BLAS_BUILD_SAMPLES)
  add_subdirectory(samples)
endif()

option(BLAS_ENABLE_AUTO_TUNERS "Whether to enable building GEMM auto tuners" OFF)
if(${BLAS_ENABLE_AUTO_TUNERS})
  # Note that the auto tuners are very slow to compile, so we avoid adding
  # them to the ALL target.
  add_subdirectory(tools/auto_tuner EXCLUDE_FROM_ALL)
endif()
