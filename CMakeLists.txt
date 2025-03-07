#------------------------------------------------------------------------------#
# Copyright 2021 Kitware, Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#------------------------------------------------------------------------------#

cmake_minimum_required(VERSION 3.1)
project(Legion)

#------------------------------------------------------------------------------#
# Some boilerplate to setup nice output directories
#------------------------------------------------------------------------------#

#for multilib distros
include(GNUInstallDirs)

list(INSERT CMAKE_MODULE_PATH 0 "${Legion_SOURCE_DIR}/cmake")
if(NOT CMAKE_ARCHIVE_OUTPUT_DIRECTORY)
  set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${Legion_BINARY_DIR}/lib)
endif()
if(NOT CMAKE_LIBRARY_OUTPUT_DIRECTORY)
  set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${Legion_BINARY_DIR}/lib)
endif()
if(NOT CMAKE_RUNTIME_OUTPUT_DIRECTORY)
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${Legion_BINARY_DIR}/bin)
endif()

#library api version, bump from time to time
set(SOVERSION 1)

set(INSTALL_SUFFIX "" CACHE STRING "Suffix to add to installed libraries and binaries")

#------------------------------------------------------------------------------#
# Version information
#------------------------------------------------------------------------------#

# the Legion/Realm version string is set by the first of these that works:
# 1) a defined value for the cmake Legion_VERSION variable
# 2) the output of `git describe`, if successful
# 3) the contents of 'VERSION' (at the root of the source tree), if available
# 4) "unknown", if all else fails

if(NOT DEFINED Legion_VERSION)
  find_package(Git)
  if(GIT_FOUND)
    execute_process(COMMAND ${GIT_EXECUTABLE} describe --dirty --match legion*
      WORKING_DIRECTORY ${Legion_SOURCE_DIR}
      RESULT_VARIABLE _git_result
      OUTPUT_VARIABLE Legion_VERSION
      ERROR_QUIET
      OUTPUT_STRIP_TRAILING_WHITESPACE)
  else()
    set(_git_result -1)
  endif()
  if(_git_result EQUAL 0)
    message(STATUS "Version string from git: ${Legion_VERSION}")
  else()
    if(EXISTS ${Legion_SOURCE_DIR}/VERSION)
      file(READ ${Legion_SOURCE_DIR}/VERSION _file_contents)
      string(STRIP "${_file_contents}" Legion_VERSION)
      unset(_file_contents)
      message(STATUS "Version string from VERSION file: ${Legion_VERSION}")
    else()
      set(Legion_VERSION "unknown")
      message(WARNING "Could not determine version string - using 'unknown'")
    endif()
  endif()
  unset(_git_result)
else()
  # it's just a "cache variable" right now, so turn into a normal variable
  set(Legion_VERSION "$CACHE{Legion_VERSION}")
endif()
set(Legion_VERSION "${Legion_VERSION}" CACHE STRING "string to use for Legion/Realm version")
# these are the actual variables that are written into the headers
set(LEGION_VERSION "${Legion_VERSION}")
set(REALM_VERSION "${Legion_VERSION}")

#------------------------------------------------------------------------------#
# Build type and shared vs static library
#------------------------------------------------------------------------------#
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CXX_FLAGS)
  set(CMAKE_BUILD_TYPE  Debug CACHE STRING "Choose the type of build" FORCE)
endif(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CXX_FLAGS)

if (CMAKE_BUILD_TYPE AND ( CMAKE_BUILD_TYPE STREQUAL "Debug"))
  # define variable for realm_defines.h
  set(DEBUG_REALM ON)
  # define variable for legion_defines.h
  set(DEBUG_LEGION ON)
endif()

set(BUILD_SHARED_LIBS OFF   CACHE BOOL   "Whether or not to build shared libraries instead of static")

#------------------------------------------------------------------------------#
# Architecture
#------------------------------------------------------------------------------#
if(BUILD_MARCH AND BUILD_MCPU)
  message(FATAL_ERROR "BUILD_MARCH and BUILD_MCPU are incompatible")
endif()

# Try -march first. On platforms that don't support it, GCC will issue
# a hard error, so we'll know not to use it.
if(BUILD_MARCH)
  set(INTERNAL_BUILD_MARCH ${BUILD_MARCH})
else()
  set(INTERNAL_BUILD_MARCH "native")
endif()

include(CheckCXXCompilerFlag)
check_cxx_compiler_flag("-march=${INTERNAL_BUILD_MARCH}" COMPILER_SUPPORTS_MARCH)
if(COMPILER_SUPPORTS_MARCH)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=${INTERNAL_BUILD_MARCH}")
elseif(BUILD_MARCH)
  message(FATAL_ERROR "The flag -march=${INTERNAL_BUILD_MARCH} is not supported by the compiler")
else()
  unset(INTERNAL_BUILD_MARCH)
endif()

# Try -mcpu. We do this second because it is deprecated on x86, but
# GCC won't issue a hard error, so we can't tell if it worked or not.
if (NOT INTERNAL_BUILD_MARCH)
  if(BUILD_MCPU)
    set(INTERNAL_BUILD_MCPU ${BUILD_MCPU})
  else()
    set(INTERNAL_BUILD_MCPU "native")
  endif()

  check_cxx_compiler_flag("-mcpu=${INTERNAL_BUILD_MCPU}" COMPILER_SUPPORTS_MCPU)
  if(COMPILER_SUPPORTS_MCPU)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mcpu=${INTERNAL_BUILD_MCPU}")
  elseif(BUILD_MCPU)
    message(FATAL_ERROR "The flag -mcpu=${INTERNAL_BUILD_MCPU} is not supported by the compiler")
  else()
    unset(INTERNAL_BUILD_MCPU)
  endif()
endif()

# Add flags for Power architectures
check_cxx_compiler_flag("-maltivec -Werror" COMPILER_SUPPORTS_MALTIVEC)
if(COMPILER_SUPPORTS_MALTIVEC)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -maltivec")
endif()
check_cxx_compiler_flag("-mabi=altivec -Werror" COMPILER_SUPPORTS_MABI_ALTIVEC)
if(COMPILER_SUPPORTS_MABI_ALTIVEC)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mabi=altivec")
endif()
check_cxx_compiler_flag("-mvsx -Werror" COMPILER_SUPPORTS_MVSX)
if(COMPILER_SUPPORTS_MVSX)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mvsx")
endif()

# detect use of the cxx_defcheck wrapper and enable build checks
check_cxx_compiler_flag("--defcheck __test__" COMPILER_SUPPORTS_DEFCHECK)

# task registration across nodes often relies on being able to map function
#  pointers back to symbol names, so ask cmake to export symbols in binaries
set(CMAKE_ENABLE_EXPORTS ON)

#------------------------------------------------------------------------------#
# Optimizations
#------------------------------------------------------------------------------#

# For now we want the optimization flags to match on both normal make and cmake
# builds so we override the cmake defaults here for release, this changes 
# -O3 to -O2 and removes -DNDEBUG
set(CMAKE_CXX_FLAGS_RELEASE "-O2")
set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-O2 -g")

# add build types for thread/ub sanitizer goodness
set(CMAKE_CXX_FLAGS_TSAN
  "${CMAKE_CXX_FLAGS_DEBUG} -fsanitize=thread -DTSAN_ENABLED"
  CACHE STRING "additional C++ compiler flags for thread sanitizer" FORCE)
set(CMAKE_EXE_LINKER_FLAGS_TSAN
  "${CMAKE_EXE_LINKER_FLAGS_DEBUG} -fsanitize=thread"
  CACHE STRING "additional linker flags for thread sanitizer" FORCE)
set(CMAKE_CXX_FLAGS_UBSAN
  "${CMAKE_CXX_FLAGS_DEBUG} -fsanitize=undefined"
  CACHE STRING "additional C++ compiler flags for UB sanitizer" FORCE)
set(CMAKE_EXE_LINKER_FLAGS_UBSAN
  "${CMAKE_EXE_LINKER_FLAGS_DEBUG} -fsanitize=undefined"
  CACHE STRING "additional linker flags for UB sanitizer" FORCE)
set(CMAKE_CXX_FLAGS_ASAN
  "${CMAKE_CXX_FLAGS_DEBUG} -fsanitize=address -DASAN_ENABLED"
  CACHE STRING "additional C++ compiler flags for address sanitizer" FORCE)
set(CMAKE_EXE_LINKER_FLAGS_ASAN
  "${CMAKE_EXE_LINKER_FLAGS_DEBUG} -fsanitize=address"
  CACHE STRING "additional linker flags for address sanitizer" FORCE)

# if requested (and we know how to) make any warnings during the build fatal
option(Legion_BUILD_WARN_AS_ERROR "Consider warnings during build of Legion to be fatal errors" OFF)
if(Legion_BUILD_WARN_AS_ERROR)
  check_cxx_compiler_flag("-Wall -Werror" COMPILER_SUPPORTS_WALL_WERROR)
  if(COMPILER_SUPPORTS_WALL_WERROR)
    set(CXX_BUILD_WARNING_FLAGS -Wall -Werror)
  else()
    message(FATAL_ERROR "cannot determine how to convert warnings to errors in C++ compiler")
  endif()
else()
  set(CXX_BUILD_WARNING_FLAGS "")
endif()

#------------------------------------------------------------------------------#
# Minimum log level
#------------------------------------------------------------------------------#
set(Legion_OUTPUT_LEVEL "DEBUG" CACHE STRING "Compile time logging level")
set_property(CACHE Legion_OUTPUT_LEVEL PROPERTY STRINGS SPEW DEBUG INFO PRINT WARNING ERROR FATAL NONE)

# define variable for realm_defines.h
set(COMPILE_TIME_MIN_LEVEL LEVEL_${Legion_OUTPUT_LEVEL})

#------------------------------------------------------------------------------#
# OpenMP
#------------------------------------------------------------------------------#
option(Legion_USE_OpenMP "Use OpenMP" OFF)
if(Legion_USE_OpenMP)
  option(Legion_OpenMP_GOMP_SUPPORT "Enable GOMP support for OpenMP" ON)
  option(Legion_OpenMP_KMP_SUPPORT "Enable KMP support for OpenMP" ON)

  # define variable for realm_defines.h
  set(REALM_USE_OPENMP ON)
  if(Legion_OpenMP_GOMP_SUPPORT)
    set(REALM_OPENMP_GOMP_SUPPORT ON)
  endif()
  if(Legion_OpenMP_KMP_SUPPORT)
    set(REALM_OPENMP_KMP_SUPPORT ON)
  endif()
endif()

#------------------------------------------------------------------------------#
# Python
#------------------------------------------------------------------------------#
option(Legion_USE_Python "Use Python" OFF)
if(Legion_USE_Python)
  set(Legion_Python_Version "" CACHE STRING "Minimum Python version")

  # Note: Legion now supports several versions, so no need to be so forceful
  # set(Python_ADDITIONAL_VERSIONS 2.7)
  find_package(PythonInterp ${Legion_Python_Version} REQUIRED) # Needed to invoke setup.py for install
  find_package(PythonLibs ${Legion_Python_Version} REQUIRED)

  # define variables for realm_defines.h
  set(REALM_USE_PYTHON ON)
  set(REALM_PYTHON_LIB "${PYTHON_LIBRARIES}")
  set(REALM_PYTHON_VERSION_MAJOR ${PYTHON_VERSION_MAJOR})

  string(REPLACE "." ";" PYTHON_VERSION_SPLIT ${PYTHONLIBS_VERSION_STRING})
  list(GET PYTHON_VERSION_SPLIT 0 PYTHON_VERSION_MAJOR)
endif()

set(BINDINGS_DEFAULT_MODULE "" CACHE STRING "module to load by default in Python bindings, if any")

#------------------------------------------------------------------------------#
# Kokkos configuration
#------------------------------------------------------------------------------#
option(Legion_USE_Kokkos "Enable support for the Kokkos runtime" OFF)
if(Legion_USE_Kokkos)
  # the kokkos interop stuff uses target_link_options, which doesn't show up
  #  until cmake 3.13
  cmake_minimum_required(VERSION 3.13)

  # cmake uses -DKokkos_DIR=.../lib/cmake to look in non-standard locations
  #
  # starting with Kokkos 3.3.0, including a cuda-enabled Kokkos will mess with
  #  the global compilation settings unless you ask it not to by requesting
  #  the 'separable_compilation' component
  find_package(Kokkos REQUIRED OPTIONAL_COMPONENTS separable_compilation)

  # in order to build using Kokkos' exported compile options, we need to use
  #  the same compiler - newer versions of Kokkos will tell us, but for older
  #  versions, we need it from the configuration or the environment
  if(Kokkos_CXX_COMPILER)
    set(KOKKOS_CXX_COMPILER ${Kokkos_CXX_COMPILER} CACHE STRING
                            "C++ compiler used by Kokkos")
  else()
    set(KOKKOS_CXX_COMPILER $ENV{KOKKOS_CXX_COMPILER} CACHE STRING
                            "C++ compiler used by Kokkos")
  endif()
  if(NOT KOKKOS_CXX_COMPILER)
    message(FATAL_ERROR "to build correctly with Kokkos, the exact compiler used in the Kokkos build (typically set via CXX=... or -DCMAKE_CXX_COMPILER=...) must be provided in KOKKOS_CXX_COMPILER (either on the command line or from the environment)")
  endif()

  set(REALM_USE_KOKKOS ON)
endif()

#------------------------------------------------------------------------------#
# CUDA configuration
#------------------------------------------------------------------------------#
option(Legion_USE_CUDA "Enable support for the CUDA runtime" OFF)
if(Legion_USE_CUDA)
  set(Legion_CUDA_ARCH "" CACHE STRING "Comma-separated list of CUDA architectures to build for (e.g. 60,70)")

  if(NOT BUILD_SHARED_LIBS)
    set(CUDA_USE_STATIC_CUDA_RUNTIME ON)
  else()
    set(CUDA_USE_STATIC_CUDA_RUNTIME OFF)
  endif()
  find_package(CUDA REQUIRED)
  install(FILES ${Legion_SOURCE_DIR}/cmake/FindCUDA.cmake
    DESTINATION ${CMAKE_INSTALL_DATADIR}/Legion/cmake
  )
  install(FILES ${Legion_SOURCE_DIR}/cmake/newcmake/FindCUDA.cmake
    DESTINATION ${CMAKE_INSTALL_DATADIR}/Legion/cmake/newcmake
  )

  # before cmake 3.13, the CUDAHOSTCXX env variable wasn't propagated to CUDA_HOST_COMPILER automatically
  if(CMAKE_VERSION VERSION_LESS "3.13" AND DEFINED ENV{CUDAHOSTCXX})
    set(CUDA_HOST_COMPILER $ENV{CUDAHOSTCXX} CACHE FILEPATH "Host side compiler used by NVCC" FORCE)
  endif()

  # define variable for legion_defines.h
  set(LEGION_USE_CUDA ON)

  option(Legion_GPU_REDUCTIONS "Use tasks to perform reductions on GPU" OFF)
  if(Legion_GPU_REDUCTIONS)
    set(LEGION_GPU_REDUCTIONS ON)
  endif()

  # define variable for realm_defines.h
  set(REALM_USE_CUDA ON)

  # default is to use hijack unless Kokkos interop is enabled
  if(Legion_USE_Kokkos)
    option(Legion_HIJACK_CUDART "Hijack and rewrite application calls into the CUDA runtime" OFF)
  else()
    option(Legion_HIJACK_CUDART "Hijack and rewrite application calls into the CUDA runtime" ON)
  endif()
  if(Legion_HIJACK_CUDART)
    set(REALM_USE_CUDART_HIJACK ON)
  endif()
endif()

#------------------------------------------------------------------------------#
# HIP configuration
#------------------------------------------------------------------------------#
option(Legion_USE_HIP "Enable support for the HIP runtime on AMD/NVIDIA GPU" OFF)
set(Legion_HIP_TARGETS ROCM CUDA)
set(Legion_HIP_TARGET "ROCM" CACHE STRING "Select HIP target ${Legion_HIP_TARGETS}")
set_property(CACHE Legion_HIP_TARGET PROPERTY STRINGS ${Legion_HIP_TARGETS})
if(Legion_USE_HIP)
  if (Legion_HIP_TARGET STREQUAL "CUDA")
    set(Legion_CUDA_ARCH "" CACHE STRING "Comma-separated list of CUDA architectures to build for (e.g. 60,70)")

    if(NOT BUILD_SHARED_LIBS)
      set(CUDA_USE_STATIC_CUDA_RUNTIME ON)
    else()
      set(CUDA_USE_STATIC_CUDA_RUNTIME OFF)
    endif()
    find_package(CUDA REQUIRED)
    install(FILES ${Legion_SOURCE_DIR}/cmake/FindCUDA.cmake
      DESTINATION ${CMAKE_INSTALL_DATADIR}/Legion/cmake
    )
    install(FILES ${Legion_SOURCE_DIR}/cmake/newcmake/FindCUDA.cmake
      DESTINATION ${CMAKE_INSTALL_DATADIR}/Legion/cmake/newcmake
    )
    set(HIPCC_FLAGS "-D__HIP_PLATFORM_NVCC__")    
  endif()
  
  if (Legion_HIP_TARGET STREQUAL "ROCM")
    set(Legion_HIP_ARCH "" CACHE STRING "Comma-separated list of HIP architectures to build for (e.g. gfx906,gfx908)")

    install(FILES ${Legion_SOURCE_DIR}/cmake/FindHIP.cmake
      DESTINATION ${CMAKE_INSTALL_DATADIR}/Legion/cmake
    )
    set(HIPCC_FLAGS "-fno-strict-aliasing")
    if("${Legion_HIP_ARCH}" STREQUAL "")
      set(HIP_GENCODE "")
    else()
      set(HIP_GENCODE "--amdgpu-target=${Legion_HIP_ARCH}")
    endif()
  endif()
  
  # find the hip library
  find_package(HIP REQUIRED)

  # define variable for legion_defines.h
  set(LEGION_USE_HIP ON)

  option(Legion_GPU_REDUCTIONS "Support reductions natively on GPU" ON)
  if(Legion_GPU_REDUCTIONS)
    set(LEGION_GPU_REDUCTIONS ON)
  endif()

  # define variable for realm_defines.h
  set(REALM_USE_HIP ON)

  option(Legion_HIJACK_HIP "Enable HIJACK HIP" ON)
  if(Legion_HIJACK_HIP)
    set(REALM_USE_HIP_HIJACK ON)
  endif()

endif()


#------------------------------------------------------------------------------#
# network interface configuration
#------------------------------------------------------------------------------#
set(Legion_NETWORKS "" CACHE STRING "Network backend(s) to use")
separate_arguments(Legion_NETWORKS)

# compatibility mode - Legion_USE_GASNet==ON -> Legion_NETWORKS=gasnet1
option(Legion_USE_GASNet "Enable the distributed GASNet backend" OFF)
if(Legion_USE_GASNet)
  list(APPEND Legion_NETWORKS gasnet1)
endif()

#------------------------------------------------------------------------------#
# GASNet configuration
#------------------------------------------------------------------------------#
if("${Legion_NETWORKS}" MATCHES ".*gasnet(1|ex).*")
  option(Legion_EMBED_GASNet "Embed a custom GASNet build in Realm library" OFF)
  if(Legion_EMBED_GASNet)
    if(NOT GASNet_CONDUIT)
      message(FATAL_ERROR "GASNet_CONDUIT must be set when using an embedded GASNet build")
    endif()
    # an embedded GASNet uses the stanfordlegion/gasnet helper repository for
    #  a build wrapper and curated-for-Legion GASNet configurations - it can
    #  be fetched from github directly (the default) or use a local (and
    #  perhaps modified) clone of the repository
    set(Legion_EMBED_GASNet_GITREPO
      "https://github.com/StanfordLegion/gasnet.git"
      CACHE STRING "URL for cloning StanfordLegion/gasnet repository")
    set(Legion_EMBED_GASNet_GITREF
      "master"
      CACHE STRING "Branch/tag/commit to use from StanfordLegion/gasnet repository")
    set(Legion_EMBED_GASNet_LOCALSRC "" CACHE STRING "Path to local clone of stanfordlegion/gasnet repository")
    set(Legion_EMBED_GASNet_VERSION "" CACHE STRING "Override GASNet version to build")
    set(Legion_EMBED_GASNet_CONFIGURE_ARGS "" CACHE STRING "Extra configuration arguments for GASNet")
  else()
    set(GASNet_THREADING par)
    set(GASNet_PREFERRED_CONDUITS aries gemini ibv)
    find_package(GASNet REQUIRED)
    if(NOT GASNet_THREADING STREQUAL "par")
      message(FATAL_ERROR "GASNet threading mode \"${GASNet_THREADING}\" is not currently supported by Legion")
    endif()
  endif()

  # define variable for realm_defines.h
  if("${Legion_NETWORKS}" MATCHES .*gasnetex.*)
    set(REALM_USE_GASNETEX ON)
  else()
    set(REALM_USE_GASNET1 ON)
  endif()
  string(TOUPPER ${GASNet_CONDUIT} CONDUIT)
  set(GASNET_CONDUIT_${CONDUIT} ON)
  # conduits other than udp support MPI interop
  if(NOT ${GASNet_CONDUIT} STREQUAL "udp")
    set(Legion_MPI_INTEROP ON)
  endif()
endif()

#------------------------------------------------------------------------------#
# MPI configuration
#------------------------------------------------------------------------------#
if("${Legion_NETWORKS}" MATCHES .*mpi.*)
  cmake_minimum_required(VERSION 3.9)
  find_package(MPI REQUIRED)
  # define variable for realm_defines.h
  set(REALM_USE_MPI ON)
  set(Legion_MPI_INTEROP ON)
endif()

#------------------------------------------------------------------------------#
# LLVM configuration
#------------------------------------------------------------------------------#
option(Legion_USE_LLVM "Use LLVM JIT operations" OFF)
option(Legion_ALLOW_MISSING_LLVM_LIBS "Allow LLVM libraries to be missing at runtime" OFF)
option(Legion_LINK_LLVM_LIBS "Link LLVM libraries into Realm" ON)
if(Legion_USE_LLVM)
  set(Legion_LLVM_COMPONENTS irreader jit mcjit x86)
  find_package(LLVM REQUIRED COMPONENTS ${Legion_LLVM_COMPONENTS})
  install(FILES ${Legion_SOURCE_DIR}/cmake/FindLLVM.cmake
    DESTINATION ${CMAKE_INSTALL_DATADIR}/Legion/cmake
  )

  # define variables for realm_defines.h
  set(REALM_USE_LLVM ON)
  if(Legion_ALLOW_MISSING_LLVM_LIBS)
    set(REALM_ALLOW_MISSING_LLVM_LIBS ON)
  endif()
  # just want REALM_LLVM_VERSION=<major><minor>
  string(REPLACE "." ";" LLVM_VERSION_SPLIT ${LLVM_VERSION})
  list(GET LLVM_VERSION_SPLIT 0 1 REALM_LLVM_VERSION_LIST)
  string(REPLACE ";" "" REALM_LLVM_VERSION "${REALM_LLVM_VERSION_LIST}")
endif()

#------------------------------------------------------------------------------#
# HDF5 configuration
#------------------------------------------------------------------------------#
option(Legion_USE_HDF5 "Enable support for HDF5" OFF)
if(Legion_USE_HDF5)
  find_package(HDF5 REQUIRED COMPONENTS C)

  # define variable for both legion_defines.h and realm_defines.h
  set(LEGION_USE_HDF5 ON)
  set(REALM_USE_HDF5 ON)
endif()

#------------------------------------------------------------------------------#
# libdl configuration
#------------------------------------------------------------------------------#
# NOTE: although the name of the variable is "libdl", this is dynamic linking
#  support on any system, even if it doesn't have a libdl
option(Legion_USE_LIBDL "Enable run-time support for dynamic shared objects" ON)
if(Legion_USE_LIBDL)
  # define variable for legion_defines.h
  set(LEGION_USE_LIBDL ON)
  # define variable for realm_defines.h
  set(REALM_USE_LIBDL ON)
endif()

#------------------------------------------------------------------------------#
# HWLOC configuration
#------------------------------------------------------------------------------#
option(Legion_USE_HWLOC "Use hwloc for topology awareness" OFF)
if(Legion_USE_HWLOC)
  find_package(HWLOC REQUIRED)
  install(FILES ${Legion_SOURCE_DIR}/cmake/FindHWLOC.cmake
    DESTINATION ${CMAKE_INSTALL_DATADIR}/Legion/cmake
  )
  set(REALM_USE_HWLOC ON)
endif()

#------------------------------------------------------------------------------#
# PAPI configuration
#------------------------------------------------------------------------------#
option(Legion_USE_PAPI "Use PAPI for performance measurements" OFF)
if(Legion_USE_PAPI)
  # no FindPAPI in cmake by default, so just find the lib and include path
  #  ourselves
  find_library(PAPI_LIBRARIES NAMES libpapi.so libpapi.a papi
    HINTS ${PAPI_ROOT}/lib $ENV{PAPI_ROOT}/lib ENV PAPI_LIB_DIR)
  find_path(PAPI_INCLUDE_DIRS NAMES papi.h
    HINTS ${PAPI_ROOT}/include $ENV{PAPI_ROOT}/include ENV PAPI_INC_DIR)
  if(PAPI_LIBRARIES AND PAPI_INCLUDE_DIRS)
    set(REALM_USE_PAPI ON)
  else()
    message(FATAL_ERROR
      " Could not find PAPI - set PAPI_{ROOT,INC_DIR,LIB_DIR} if needed -- got:\n"
      "   PAPI_LIBRARIES: ${PAPI_LIBRARIES}\n"
      "   PAPI_INCLUDE_DIRS: ${PAPI_INCLUDE_DIRS}")
  endif()
endif()

#------------------------------------------------------------------------------#
# zlib configuration
#------------------------------------------------------------------------------#
if(NOT DEFINED Legion_USE_ZLIB)
  # neither ON nor OFF at user's request, so use it if we can find it
  find_package(ZLIB)
  if(ZLIB_FOUND)
    set(Legion_USE_ZLIB ON)
  else()
    set(Legion_USE_ZLIB OFF)
  endif()
else()
  set(Legion_USE_ZLIB "$CACHE{Legion_USE_ZLIB}")
endif()
set(Legion_USE_ZLIB "${Legion_USE_ZLIB}" CACHE BOOL "Enable support for zlib")
if(Legion_USE_ZLIB)
  find_package(ZLIB REQUIRED)
  # define variable for legion_defines.h
  set(LEGION_USE_ZLIB ON)
endif()
                                                                                          
#------------------------------------------------------------------------------#
# Fortran configuration
#------------------------------------------------------------------------------#
option(Legion_USE_Fortran "Enable support for Fortran" OFF)
if(Legion_USE_Fortran)
  enable_language(Fortran)
  set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -cpp")
endif()

#------------------------------------------------------------------------------#
# Miscelaneous other options
#------------------------------------------------------------------------------#
set(Legion_MAX_DIM 3 CACHE STRING "Maximum number of dimensions")
set_property(CACHE Legion_MAX_DIM PROPERTY STRINGS 1 2 3 4 5 6 7 8 9)
mark_as_advanced(Legion_MAX_DIM)

# define variable for legion_defines.h
set(LEGION_MAX_DIM ${Legion_MAX_DIM})
set(REALM_MAX_DIM ${Legion_MAX_DIM})

set(Legion_MAX_FIELDS 512 CACHE STRING "Maximum number of fields allocated to a single field space")
set_property(CACHE Legion_MAX_FIELDS PROPERTY STRINGS 32 64 128 256 512 1024)
mark_as_advanced(Legion_MAX_FIELDS)

# define variable for legion_defines.h
set(LEGION_MAX_FIELDS ${Legion_MAX_FIELDS})

option(Legion_WARNINGS_FATAL "Make all runtime warnings fatal" OFF)
set(LEGION_WARNINGS_FATAL ${Legion_WARNINGS_FATAL})

option(Legion_SPY "Enable detailed logging for Legion Spy" OFF)
set(LEGION_SPY ${Legion_SPY})

option(Legion_BOUNDS_CHECKS "Enable bounds checking in Legion accessors" OFF)
set(LEGION_BOUNDS_CHECKS ${Legion_BOUNDS_CHECKS})

option(Legion_PRIVILEGE_CHECKS "Enable privilege checking in Legion accessors" OFF)
set(LEGION_PRIVILEGE_CHECKS ${Legion_PRIVILEGE_CHECKS})

# hiding internal symbols is much more complicated in Windows
if(NOT WIN32)
  option(REALM_LIMIT_SYMBOL_VISIBILITY "hide private symbols in realm" ON)
endif()

#------------------------------------------------------------------------------#
# Hacks for building only Realm
#------------------------------------------------------------------------------#
option(Legion_BUILD_REALM_ONLY "HACK: only build realm" OFF)
option(Legion_BUILD_REALM_TESTS "HACK: build realm-only tests" OFF)

#------------------------------------------------------------------------------#
# Runtime library targets
#------------------------------------------------------------------------------#
add_subdirectory(runtime)

#------------------------------------------------------------------------------#
# Tools
#------------------------------------------------------------------------------#
add_subdirectory(tools)

#------------------------------------------------------------------------------#
# Documentation
#------------------------------------------------------------------------------#
add_subdirectory(doc)

#------------------------------------------------------------------------------#
# configure header
#------------------------------------------------------------------------------#
# Checking for all defines in the CXX Flags

string(REPLACE " " ";" FLAGS_LIST ${CMAKE_CXX_FLAGS})
FOREACH(FLAG ${FLAGS_LIST})
  string (FIND ${FLAG} "-D" START_STR)
  if (${START_STR} EQUAL "0")
  string(REPLACE "-D" "" NEW_DEFINE ${FLAG})
  set (${NEW_DEFINE} ON)
  endif()
endforeach()


#------------------------------------------------------------------------------#
# Build-tree package generation
#------------------------------------------------------------------------------#
export(EXPORT LegionTargets
  NAMESPACE Legion::
  FILE ${Legion_BINARY_DIR}/LegionTargets.cmake
)
install(EXPORT LegionTargets
  NAMESPACE Legion::
  FILE LegionTargets.cmake
  DESTINATION ${CMAKE_INSTALL_DATADIR}/Legion/cmake
)

configure_file(
  cmake/LegionConfigCommon.cmake.in
  ${Legion_BINARY_DIR}/LegionConfigCommon.cmake
  @ONLY
)
install(FILES ${Legion_BINARY_DIR}/LegionConfigCommon.cmake
  DESTINATION ${CMAKE_INSTALL_DATADIR}/Legion/cmake
)

configure_file(
  cmake/LegionConfig-build.cmake.in
  ${Legion_BINARY_DIR}/LegionConfig.cmake
  @ONLY
)

install(FILES cmake/LegionConfig-install.cmake
  DESTINATION ${CMAKE_INSTALL_DATADIR}/Legion/cmake
  RENAME LegionConfig.cmake
)

#------------------------------------------------------------------------------#
# Install into the CMake pacakge registry if explicitly selected to do so
#------------------------------------------------------------------------------#
set(CMAKE_EXPORT_NO_PACKAGE_REGISTRY ON CACHE INTERNAL "Disable the export(PACKAGE) command.")
option(Legion_USE_PACKAGE_REGISTRY "Register the build tree with the CMake package registry" OFF)
mark_as_advanced(Legion_USE_PACKAGE_REGISTRY)
if(Legion_USE_PACKAGE_REGISTRY)
  set(CMAKE_EXPORT_NO_PACKAGE_REGISTRY OFF)
endif()
export(PACKAGE Legion)

#------------------------------------------------------------------------------#
# Examples and Applications
#------------------------------------------------------------------------------#
option(Legion_BUILD_ALL "Build all Legion bindings, examples, tutorials and tests" OFF)
option(Legion_BUILD_APPS "Build Legion sample applications" OFF)
option(Legion_BUILD_BINDINGS "Build Legion language bindings" OFF)
option(Legion_BUILD_EXAMPLES "Build Legion examples" OFF)
option(Legion_BUILD_TUTORIAL "Build Legion tutorial" OFF)
option(Legion_BUILD_TESTS "Build Legion tests" OFF)
option(Legion_ENABLE_TESTING "Build and copy testing stuff" OFF)
if(Legion_ENABLE_TESTING)
  enable_testing()
  # if Legion_TEST_LAUNCHER is set, split the string into a list
  separate_arguments(Legion_TEST_LAUNCHER)
  separate_arguments(Legion_TEST_ARGS)
  # if we are using kokkos and cuda, we must hardcode the use of 1 gpu right now
  if(Legion_USE_Kokkos AND Legion_USE_CUDA)
    list(APPEND Legion_TEST_ARGS -ll:gpu 1)
  endif()
  # similar restrictions for openmp
  if(Legion_USE_Kokkos AND Legion_USE_OpenMP)
    list(APPEND Legion_TEST_ARGS -ll:ocpu 1 -ll:onuma 0)
  endif()
endif()
if(Legion_BUILD_ALL OR Legion_BUILD_APPS OR Legion_BUILD_BINDINGS OR Legion_BUILD_EXAMPLES OR Legion_BUILD_TUTORIAL OR Legion_BUILD_TESTS)
  # Make a namespaced alias so in-build examples can use it the same way as if
  # it were imported
  add_library(Legion::Legion ALIAS Legion)
  add_library(Legion::Realm ALIAS Realm)

  if(Legion_USE_CUDA)
    # CUDA has some issues propogating target interface properties so we
    # have to extract them from the target and manually add them to the nvcc
    # flags
    set(PROP $<TARGET_PROPERTY:Legion::Legion,INTERFACE_COMPILE_OPTIONS>)
    set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} ${PROP}")
    set(PROP $<TARGET_PROPERTY:Legion::Legion,INTERFACE_COMPILE_DEFINITIONS>)
    set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} $<$<BOOL:${PROP}>:-D$<JOIN:${PROP}, -D>>")
    set(PROP $<TARGET_PROPERTY:Legion::Legion,INTERFACE_INCLUDE_DIRECTORIES>)
    set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} $<$<BOOL:${PROP}>:-I$<JOIN:${PROP}, -I>>")
  endif()
  
  if(Legion_HIP_TARGET STREQUAL "CUDA")
    set(PROP $<TARGET_PROPERTY:Legion::Legion,INTERFACE_COMPILE_OPTIONS>)
    set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} ${PROP}")
    set(PROP $<TARGET_PROPERTY:Legion::Legion,INTERFACE_COMPILE_DEFINITIONS>)
    set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} $<$<BOOL:${PROP}>:-D$<JOIN:${PROP}, -D>>")
    set(PROP $<TARGET_PROPERTY:Legion::Legion,INTERFACE_INCLUDE_DIRECTORIES>)
    set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} $<$<BOOL:${PROP}>:-I$<JOIN:${PROP}, -I>>")
    set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} ${HIPCC_FLAGS}")
  endif()

  if(Legion_HIP_TARGET STREQUAL "ROCM")
    set(HIP_HIPCC_FLAGS "${HIP_HIPCC_FLAGS} ${HIPCC_FLAGS} ${HIP_GENCODE}")
  endif()

  if(Legion_BUILD_ALL OR Legion_BUILD_APPS)
    add_subdirectory(apps)
  endif()
  if(Legion_BUILD_ALL OR Legion_BUILD_BINDINGS)
    add_subdirectory(bindings)
  endif()
  if(Legion_BUILD_ALL OR Legion_BUILD_EXAMPLES)
    add_subdirectory(examples)
  endif()
  if(Legion_BUILD_ALL OR Legion_BUILD_TUTORIAL)
    add_subdirectory(tutorial)
  endif()
  if(Legion_BUILD_ALL OR Legion_BUILD_TESTS)
    add_subdirectory(test)
  endif()
elseif(Legion_BUILD_REALM_TESTS)
  # temporary hackery to just enable the realm tests
  add_library(Legion::Realm ALIAS Realm)
  add_subdirectory(test/realm)
endif()

#------------------------------------------------------------------------------#
# Complex support
#------------------------------------------------------------------------------#

if(Legion_BUILD_ALL OR Legion_BUILD_BINDINGS)
  option(Legion_REDOP_COMPLEX "Use reduction operators for complex types" ON)
else()
  option(Legion_REDOP_COMPLEX "Use reduction operators for complex types" OFF)
endif()
if((Legion_BUILD_ALL OR Legion_BUILD_BINDINGS) AND NOT Legion_REDOP_COMPLEX)
  message(FATAL_ERROR "Bindings require Legion_REDOP_COMPLEX to be set")
endif()
if(Legion_REDOP_COMPLEX)
  # define variable for legion_defines.h
  set(LEGION_REDOP_COMPLEX ON)
endif()
mark_as_advanced(Legion_REDOP_COMPLEX)
if(Legion_REDOP_HALF)
  # define variable for legion_defines.h
  set(LEGION_REDOP_HALF ON)
endif()
mark_as_advanced(Legion_REDOP_HALF)

#------------------------------------------------------------------------------#
# Configure definition headers
#------------------------------------------------------------------------------#

configure_file(${PROJECT_SOURCE_DIR}/cmake/legion_defines.h.in
  ${PROJECT_BINARY_DIR}/runtime/legion_defines.h @ONLY)
install(FILES ${PROJECT_BINARY_DIR}/runtime/legion_defines.h
  DESTINATION  ${CMAKE_INSTALL_INCLUDEDIR})

configure_file(${PROJECT_SOURCE_DIR}/cmake/realm_defines.h.in
  ${PROJECT_BINARY_DIR}/runtime/realm_defines.h @ONLY)
install(FILES ${PROJECT_BINARY_DIR}/runtime/realm_defines.h
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

install(
  FILES
    ${PROJECT_SOURCE_DIR}/tools/legion_prof.py
    ${PROJECT_SOURCE_DIR}/tools/legion_spy.py
  DESTINATION
    ${CMAKE_INSTALL_BINDIR}
  PERMISSIONS
    OWNER_EXECUTE OWNER_WRITE OWNER_READ
    GROUP_EXECUTE GROUP_WRITE GROUP_READ
    WORLD_EXECUTE WORLD_READ)

install(
  FILES
    ${PROJECT_SOURCE_DIR}/tools/legion_serializer.py
    ${PROJECT_SOURCE_DIR}/tools/legion_prof_copy.html.template
  DESTINATION
    ${CMAKE_INSTALL_BINDIR}
  PERMISSIONS
    OWNER_WRITE OWNER_READ
    GROUP_WRITE GROUP_READ
    WORLD_READ)

install(
  DIRECTORY
    ${PROJECT_SOURCE_DIR}/tools/serializer_examples
    ${PROJECT_SOURCE_DIR}/tools/legion_prof_files
  DESTINATION
    ${CMAKE_INSTALL_BINDIR}
  USE_SOURCE_PERMISSIONS)

#------------------------------------------------------------------------------#
# vim: set tabstop=2 shiftwidth=2 expandtab :
#------------------------------------------------------------------------------#
