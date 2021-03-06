#
# GCC 4.8 or higher compiler required.
#
#   Required Defines on cmake command line
#
#   1) Set location of ROCR header files (required)
#
#      ROCM_DIR="Root for RocM install"
#
#   2) Set ROCRTST_BLD_TYPE to either "Debug" or "Release".
#      If not set, the default value is "Debug" is bound.
#
#      ROCRTST_BLD_TYPE=Debug or ROCRTST_BLD_TYPE=Release
#
#   3) Set ROCRTST_BLD_BITS to either "32" or "64"
#      If not set, the default value of "64" is bound.
#
#       ROCRTST_BLD_BITS=32 or ROCRTST_BLD_BITS=64
#
#   Building rocminfo
#
#   1) Create build folder e.g. "rocminfo/build" - any name will do
#   2) Cd into build folder
#   3) Run cmake, passing in the above defines, as needed/required:
#       "cmake -DROCM_DIR=<path to rocm root> <other defines if needed> .."
#   4) Run "make"
#
#   Upon a successful build, the executable "rocminfo" will be in the
#   build directory.
#
# Currently support for Windows platform is not present
#

#
# Minimum version of cmake required
#
cmake_minimum_required(VERSION 3.5.0)

set(ROCMINFO_EXE "rocminfo")
set(PROJECT_NAME ${ROCMINFO_EXE})
project (${PROJECT_NAME})

include ( GNUInstallDirs )
if(WIN32)
  message("This sample is not supported on Windows platform")
  return()
endif()

## Set default module path if not already set
if(NOT DEFINED CMAKE_MODULE_PATH)
    set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake_modules/")
endif()
## Include common cmake modules
include(utils)

#
# Process input variables
#

find_package(hsa-runtime64 1.0 REQUIRED )

string(TOLOWER "${ROCRTST_BLD_TYPE}" tmp)
if("${tmp}" STREQUAL release)
  set(BUILD_TYPE "Release")
  set(ISDEBUG 0)
else()
  set(BUILD_TYPE "Debug")
  set(ISDEBUG 1)
endif()

# The following default version values should be updated as appropriate for
# ABI breaks (update MAJOR and MINOR), and ABI/API additions (update MINOR).
# Until ABI stabilizes VERSION_MAJOR will be 0. This should be over-ridden
# by git tags (through "git describe") when they are present.
set(PKG_VERSION_MAJOR 1)
set(PKG_VERSION_MINOR 0)
set(PKG_VERSION_PATCH 0)
set(PKG_VERSION_NUM_COMMIT 0)

################# Determine the library version #########################
## Setup the package version based on git tags.
set(PKG_VERSION_GIT_TAG_PREFIX "rocminfo_pkg_ver")

find_program (GIT NAMES git)

get_package_version_number("1.0.0" ${PKG_VERSION_GIT_TAG_PREFIX} GIT)
# VERSION_* variables should be set by get_version_from_tag
message("Package version: ${PKG_VERSION_STR}")

#
# Print out the build configuration being used:
#
#   Build Src directory
#   Build Binary directory
#   Build Type: Debug Vs Release, 32 Vs 64
#   Compiler Version, etc
#
message("")
message("Build Configuration:")
message("-----------BuildType: " ${BUILD_TYPE})
message("------------Compiler: " ${CMAKE_CXX_COMPILER})
message("-------------Version: " ${CMAKE_CXX_COMPILER_VERSION})
message("--------Proj Src Dir: " ${PROJECT_SOURCE_DIR})
message("--------Proj Bld Dir: " ${PROJECT_BINARY_DIR})
message("--------Proj Lib Dir: " ${PROJECT_BINARY_DIR}/lib)
message("--------Proj Exe Dir: " ${PROJECT_BINARY_DIR}/bin)
message("")


#
# Set the build type based on user input
#
set(CMAKE_BUILD_TYPE ${BUILD_TYPE})
#
# Flag to enable / disable verbose output.
#
SET( CMAKE_VERBOSE_MAKEFILE on )
#
# Compiler pre-processor definitions.
#
# Define MACRO "DEBUG" if build type is "Debug"
if(${BUILD_TYPE} STREQUAL "Debug")
add_definitions(-DDEBUG)
endif()

add_definitions(-D__linux__)
add_definitions(-DLITTLEENDIAN_CPU=1)

#
# Linux Compiler options
#
set(CMAKE_CXX_FLAGS "-std=c++11 ")

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fexceptions")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-rtti")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-math-errno")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-threadsafe-statics")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fmerge-all-constants")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fms-extensions")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -m64")

#
# Extend the compiler flags for 64-bit builds
#
if((${CMAKE_HOST_SYSTEM_PROCESSOR} STREQUAL "x86_64") OR (${CMAKE_HOST_SYSTEM_PROCESSOR} STREQUAL "AMD64"))
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -msse -msse2")
endif()

#
# Add compiler flags to include symbol information for debug builds
#
if(ISDEBUG)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -ggdb -O0")
endif()

###########################
# rocm_agent_enumerator
###########################

configure_file(rocm_agent_enumerator rocm_agent_enumerator COPYONLY)


###########################
# RocR Info
###########################
aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR} ROCMINFO_SOURCES)
add_executable(${ROCMINFO_EXE} ${ROCMINFO_SOURCES})
target_link_libraries(${ROCMINFO_EXE} hsa-runtime64::hsa-runtime64)

###########################
# Install directives
###########################
install (
   FILES ${CMAKE_CURRENT_BINARY_DIR}/${ROCMINFO_EXE}
   PERMISSIONS OWNER_READ OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
   DESTINATION ${CMAKE_INSTALL_BINDIR} )
install (
   FILES ${CMAKE_CURRENT_BINARY_DIR}/rocm_agent_enumerator
   PERMISSIONS OWNER_READ OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
   DESTINATION ${CMAKE_INSTALL_BINDIR} )

###########################
# Packaging directives
###########################
# set(CPACK_PACKAGE_NAME "${PROJECT_NAME}")
set(CPACK_PACKAGE_FILE_NAME "${PROJECT_NAME}-${PKG_VERSION_STR}")

if (NOT DEFINED CPACK_PACKAGE_VENDOR)
    set(CPACK_PACKAGE_VENDOR "AMD")
endif ()

if (NOT DEFINED CPACK_PACKAGE_DESCRIPTION_SUMMARY)
    set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Radeon Open Compute (ROCm) Runtime rocminfo tool")
endif ()


if (NOT DEFINED CPACK_PACKAGE_CONTACT)
    set(CPACK_PACKAGE_CONTACT "Advanced Micro Devices Inc.")
endif()

###########################
# Debian package specific variables
###########################
set(CPACK_DEBIAN_PACKAGE_HOMEPAGE ${CPACK_DEBIAN_PACKAGE_HOMEPAGE} CACHE STRING "https://github.com/RadeonOpenCompute/ROCm")

###########################
# RPM package specific variables
###########################
if ( DEFINED CPACK_PACKAGING_INSTALL_PREFIX )
    set ( CPACK_RPM_EXCLUDE_FROM_AUTO_FILELIST_ADDITION "${CPACK_PACKAGING_INSTALL_PREFIX} ${CPACK_PACKAGING_INSTALL_PREFIX}/${CMAKE_INSTALL_BINDIR}" )
endif ( )

###########################
# Include packaging
###########################
include ( CPack )
