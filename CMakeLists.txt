# Copyright 2010-2015 Free Software Foundation, Inc.
#
# This file is part of GNU Radio
#
# GNU Radio is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 3, or (at your option)
# any later version.
#
# GNU Radio is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with GNU Radio; see the file COPYING.  If not, write to
# the Free Software Foundation, Inc., 51 Franklin Street,
# Boston, MA 02110-1301, USA.

########################################################################
if(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
    message(FATAL_ERROR "Prevented in-tree build. This is bad practice.")
endif(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})

########################################################################
# Project setup
########################################################################
cmake_minimum_required(VERSION 2.6)
project(GrRuntime CXX C)
enable_testing()

#make sure our local CMake Modules path comes first
list(INSERT CMAKE_MODULE_PATH 0 ${CMAKE_SOURCE_DIR}/cmake/Modules)

include(GrBuildTypes)

#select the release build type by default to get optimization flags
if(NOT CMAKE_BUILD_TYPE)
   set(CMAKE_BUILD_TYPE "Release")
   message(STATUS "Build type not specified: defaulting to release.")
endif(NOT CMAKE_BUILD_TYPE)
GR_CHECK_BUILD_TYPE(${CMAKE_BUILD_TYPE})
set(CMAKE_BUILD_TYPE ${CMAKE_BUILD_TYPE} CACHE STRING "")
message(STATUS "Build type set to ${CMAKE_BUILD_TYPE}.")

# Set the version information here
set(VERSION_INFO_MAJOR_VERSION 3)
set(VERSION_INFO_API_COMPAT    7)
set(VERSION_INFO_MINOR_VERSION 9)
set(VERSION_INFO_MAINT_VERSION 1)
include(GrVersion) #setup version info

# Append -O2 optimization flag for Debug builds
SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -O2")
SET(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -O2")

########################################################################
# Environment setup
########################################################################
IF(NOT DEFINED BOOST_ROOT)
    SET(BOOST_ROOT ${CMAKE_INSTALL_PREFIX})
ENDIF()

########################################################################
# Import executables from a native build (for cross compiling)
# http://www.vtk.org/Wiki/CMake_Cross_Compiling#Using_executables_in_the_build_created_during_the_build
########################################################################
if(IMPORT_EXECUTABLES)
    include(${IMPORT_EXECUTABLES})
endif(IMPORT_EXECUTABLES)

#set file that the native build will fill with exports
set(EXPORT_FILE ${CMAKE_BINARY_DIR}/ImportExecutables.cmake)
file(WRITE ${EXPORT_FILE}) #blank the file (subdirs will append)

########################################################################
# Compiler specific setup
########################################################################
include(GrMiscUtils) #compiler flag check

if(CMAKE_COMPILER_IS_GNUCXX AND NOT WIN32)
    #http://gcc.gnu.org/wiki/Visibility
    GR_ADD_CXX_COMPILER_FLAG_IF_AVAILABLE(-fvisibility=hidden HAVE_VISIBILITY_HIDDEN)
endif()

if(CMAKE_COMPILER_IS_GNUCXX)
    GR_ADD_CXX_COMPILER_FLAG_IF_AVAILABLE(-Wsign-compare HAVE_WARN_SIGN_COMPARE)
    GR_ADD_CXX_COMPILER_FLAG_IF_AVAILABLE(-Wall HAVE_WARN_ALL)
    GR_ADD_CXX_COMPILER_FLAG_IF_AVAILABLE(-Wno-uninitialized HAVE_WARN_NO_UNINITIALIZED)
endif(CMAKE_COMPILER_IS_GNUCXX)

if(MSVC)
    if ("${MSVC_VERSION}" VERSION_LESS "1800")
        include_directories(${CMAKE_SOURCE_DIR}/cmake/msvc) #missing headers
    else()
        include_directories(${CMAKE_SOURCE_DIR}/cmake/msvc2013) #missing headers
    endif()
    add_definitions(-D_WIN32_WINNT=0x0502) #Minimum version: "Windows Server 2003 with SP1, Windows XP with SP2"
    add_definitions(-DNOMINMAX) #disables stupidity and enables std::min and std::max
    add_definitions( #stop all kinds of compatibility warnings
        -D_SCL_SECURE_NO_WARNINGS
        -D_CRT_SECURE_NO_WARNINGS
        -D_CRT_SECURE_NO_DEPRECATE
        -D_CRT_NONSTDC_NO_DEPRECATE
    )
    add_definitions(-DHAVE_CONFIG_H)
    add_definitions(/MP) #build with multiple processors
    add_definitions(/bigobj) #allow for larger object files
endif(MSVC)

# Record Compiler Info for record
STRING(TOUPPER ${CMAKE_BUILD_TYPE} GRCBTU)
set(COMPILER_INFO "")
IF(MSVC)
    IF(MSVC90)   #Visual Studio 9
        SET(cmake_c_compiler_version "Microsoft Visual Studio 9.0")
        SET(cmake_cxx_compiler_version "Microsoft Visual Studio 9.0")
    ELSE(MSVC10) #Visual Studio 10
        SET(cmake_c_compiler_version "Microsoft Visual Studio 10.0")
        SET(cmake_cxx_compiler_version "Microsoft Visual Studio 10.0")
    ELSE(MSVC11) #Visual Studio 11
        SET(cmake_c_compiler_version "Microsoft Visual Studio 11.0")
        SET(cmake_cxx_compiler_version "Microsoft Visual Studio 11.0")
    ELSE(MSVC12) #Visual Studio 12
        SET(cmake_c_compiler_version "Microsoft Visual Studio 12.0")
        SET(cmake_cxx_compiler_version "Microsoft Visual Studio 12.0")
    ENDIF()
ELSE()
    execute_process(COMMAND ${CMAKE_C_COMPILER} --version
            OUTPUT_VARIABLE cmake_c_compiler_version)
    execute_process(COMMAND ${CMAKE_CXX_COMPILER} --version
            OUTPUT_VARIABLE cmake_cxx_compiler_version)
ENDIF(MSVC)
set(COMPILER_INFO "${CMAKE_C_COMPILER}:::${CMAKE_C_FLAGS_${GRCBTU}} ${CMAKE_C_FLAGS}\n${CMAKE_CXX_COMPILER}:::${CMAKE_CXX_FLAGS_${GRCBTU}} ${CMAKE_CXX_FLAGS}\n" )

# Convert to a C string to compile and display properly
string(STRIP "${cmake_c_compiler_version}" cmake_c_compiler_version)
string(STRIP "${cmake_cxx_compiler_version}" cmake_cxx_compiler_version)
string(STRIP ${COMPILER_INFO} COMPILER_INFO)
MESSAGE(STATUS "Compiler Version: ${cmake_c_compiler_version}")
MESSAGE(STATUS "Compiler Flags: ${COMPILER_INFO}")
string(REPLACE "\n" " \\n" cmake_c_compiler_version ${cmake_c_compiler_version})
string(REPLACE "\n" " \\n" cmake_cxx_compiler_version ${cmake_cxx_compiler_version})
string(REPLACE "\n" " \\n" COMPILER_INFO ${COMPILER_INFO})

########################################################################
# Install directories
########################################################################
include(GrPlatform) #define LIB_SUFFIX
set(GR_RUNTIME_DIR     bin CACHE PATH "Path to install all binaries")
set(GR_LIBRARY_DIR     lib${LIB_SUFFIX} CACHE PATH "Path to install libraries")
set(GR_INCLUDE_DIR     include CACHE PATH "Path to install header files")
set(GR_DATA_DIR        share CACHE PATH "Base location for data")
set(GR_PKG_DATA_DIR    ${GR_DATA_DIR}/${CMAKE_PROJECT_NAME} CACHE PATH "Path to install package data")
set(GR_DOC_DIR         ${GR_DATA_DIR}/doc CACHE PATH "Path to install documentation")
set(GR_PKG_DOC_DIR     ${GR_DOC_DIR}/${CMAKE_PROJECT_NAME}-${DOCVER} CACHE PATH "Path to install package docs")
set(GR_LIBEXEC_DIR     libexec CACHE PATH "Path to install libexec files")
set(GR_PKG_LIBEXEC_DIR ${GR_LIBEXEC_DIR}/${CMAKE_PROJECT_NAME} CACHE PATH "Path to install package libexec files")
set(GRC_BLOCKS_DIR     ${GR_PKG_DATA_DIR}/grc/blocks CACHE PATH "Path to install GRC blocks")
set(GR_THEMES_DIR      ${GR_PKG_DATA_DIR}/themes CACHE PATH "Path to install QTGUI themes")

# Set location of config/prefs files in /etc
# Special exception if prefix is /usr so we don't make a /usr/etc.
set(GR_CONF_DIR etc CACHE PATH "Path to install config files")
string(COMPARE EQUAL ${CMAKE_INSTALL_PREFIX} "/usr" isusr)
if(isusr)
  set(SYSCONFDIR "/${GR_CONF_DIR}" CACHE PATH "System configuration directory")
else(isusr)
  set(SYSCONFDIR "${CMAKE_INSTALL_PREFIX}/${GR_CONF_DIR}" CACHE PATH "System configuration directory" FORCE)
endif(isusr)

set(GR_PKG_CONF_DIR ${SYSCONFDIR}/${CMAKE_PROJECT_NAME}/conf.d CACHE PATH "Path to install package configs")
set(GR_PREFSDIR     ${SYSCONFDIR}/${CMAKE_PROJECT_NAME}/conf.d CACHE PATH "Path to install preference files")

OPTION(ENABLE_PERFORMANCE_COUNTERS "Enable block performance counters" ON)
if(ENABLE_PERFORMANCE_COUNTERS)
  message(STATUS "ADDING PERF COUNTERS")
  SET(GR_PERFORMANCE_COUNTERS True)
  add_definitions(-DGR_PERFORMANCE_COUNTERS)
else(ENABLE_PERFORMANCE_COUNTERS)
  SET(GR_PERFORMANCE_COUNTERS False)
  message(STATUS "NO PERF COUNTERS")
endif(ENABLE_PERFORMANCE_COUNTERS)

OPTION(ENABLE_STATIC_LIBS "Enable building of static libraries" OFF)
message(STATUS "Building Static Libraries: ${ENABLE_STATIC_LIBS}")

########################################################################
# Variables replaced when configuring the package config files
########################################################################
file(TO_NATIVE_PATH "${CMAKE_INSTALL_PREFIX}"           prefix)
file(TO_NATIVE_PATH "\${prefix}"                        exec_prefix)
file(TO_NATIVE_PATH "\${exec_prefix}/${GR_LIBRARY_DIR}" libdir)
file(TO_NATIVE_PATH "\${prefix}/${GR_INCLUDE_DIR}"      includedir)
file(TO_NATIVE_PATH "${SYSCONFDIR}"                     SYSCONFDIR)
file(TO_NATIVE_PATH "${GR_PREFSDIR}"                    GR_PREFSDIR)

########################################################################
# On Apple only, set install name and use rpath correctly, if not already set
########################################################################
if(APPLE)
    if(NOT CMAKE_INSTALL_NAME_DIR)
        set(CMAKE_INSTALL_NAME_DIR
            ${CMAKE_INSTALL_PREFIX}/${GR_LIBRARY_DIR} CACHE
            PATH "Library Install Name Destination Directory" FORCE)
    endif(NOT CMAKE_INSTALL_NAME_DIR)
    if(NOT CMAKE_INSTALL_RPATH)
        set(CMAKE_INSTALL_RPATH
            ${CMAKE_INSTALL_PREFIX}/${GR_LIBRARY_DIR} CACHE
            PATH "Library Install RPath" FORCE)
    endif(NOT CMAKE_INSTALL_RPATH)
    if(NOT CMAKE_BUILD_WITH_INSTALL_RPATH)
        set(CMAKE_BUILD_WITH_INSTALL_RPATH ON CACHE
            BOOL "Do Build Using Library Install RPath" FORCE)
    endif(NOT CMAKE_BUILD_WITH_INSTALL_RPATH)
endif(APPLE)

########################################################################
# Create uninstall target
########################################################################
configure_file(
    ${CMAKE_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake
@ONLY)

add_custom_target(uninstall
    ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake
)

########################################################################
# Setup Boost for global use (within this build)
########################################################################
include(GrBoost)

########################################################################
# Setup volk as a subproject
########################################################################
message(STATUS "")
message(STATUS "Configuring VOLK support...")

find_package(Volk REQUIRED)
set(ENABLE_VOLK TRUE)

# Handle gr_log enable/disable
GR_LOGGING()

########################################################################
# Add subdirectories (in order of deps)
########################################################################
include(GrComponent)
add_subdirectory(gnuradio-runtime)

########################################################################
# Print summary
########################################################################
GR_PRINT_COMPONENT_SUMMARY()
message(STATUS "Using install prefix: ${CMAKE_INSTALL_PREFIX}")
message(STATUS "Building for version: ${VERSION} / ${LIBVER}")

# Create a config.h with some definitions to export to other projects.
CONFIGURE_FILE(
  ${CMAKE_CURRENT_SOURCE_DIR}/config.h.in
  ${CMAKE_CURRENT_BINARY_DIR}/gnuradio-runtime/include/gnuradio/config.h
)

#Re-generate the constants file, now that we actually know which components will be enabled.
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/gnuradio-runtime/lib/constants.cc.in
    ${CMAKE_CURRENT_BINARY_DIR}/gnuradio-runtime/lib/constants.cc
    ESCAPE_QUOTES
@ONLY)

# Install config.h in include/gnuradio
#install(
    #FILES
    #${CMAKE_CURRENT_BINARY_DIR}/gnuradio-runtime/include/gnuradio/config.h
    #DESTINATION ${GR_INCLUDE_DIR}/gnuradio
    #COMPONENT "runtime_devel"
#)
