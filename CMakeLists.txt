# Copyright 2012-2015 Free Software Foundation, Inc.
#
# This file is part of SoapyOsmo support modules
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
# Project setup
########################################################################
cmake_minimum_required(VERSION 2.8.7)
project(SoapyOsmo CXX C)
enable_testing()

set(GR_OSMOSDR_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/gr-osmosdr)

list(APPEND CMAKE_MODULE_PATH ${GR_OSMOSDR_SOURCE_DIR}/cmake/Modules)

#select the release build type by default to get optimization flags
if(NOT CMAKE_BUILD_TYPE)
   set(CMAKE_BUILD_TYPE "Release")
   message(STATUS "Build type not specified: defaulting to release.")
endif(NOT CMAKE_BUILD_TYPE)
set(CMAKE_BUILD_TYPE ${CMAKE_BUILD_TYPE} CACHE STRING "")

# Set the version information here
set(VERSION "0.2.4")
set(SOVERSION 0)

find_package(SoapySDR NO_MODULE REQUIRED)

########################################################################
# Compiler specific setup
########################################################################

IF(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|AMD64|x86")
 SET(USE_SIMD "SSE2" CACHE STRING "Use SIMD instructions")
ELSE()
 SET(USE_SIMD "no" CACHE STRING "Use SIMD instructions")
ENDIF()
 SET(USE_SIMD_VALUES "no" "SSE2" "AVX")
 SET_PROPERTY(CACHE USE_SIMD PROPERTY STRINGS ${USE_SIMD_VALUES})
 LIST(FIND USE_SIMD_VALUES ${USE_SIMD} USE_SIMD_INDEX)
 IF(${USE_SIMD_INDEX} EQUAL -1)
   message(FATAL_ERROR "Option ${USE_SIMD} not supported, valid entries are ${USE_SIMD_VALUES}")
 ENDIF()

IF(CMAKE_CXX_COMPILER MATCHES ".*clang")
    SET(CMAKE_COMPILER_IS_CLANGXX 1)
ENDIF()

IF(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_COMPILER_IS_CLANGXX)
    ADD_DEFINITIONS(-Wall)
    ADD_DEFINITIONS(-Wextra)
    ADD_DEFINITIONS(-Wno-unused-parameter)
    ADD_DEFINITIONS(-Wsign-compare)
    #ADD_DEFINITIONS(-Wconversion)
    #ADD_DEFINITIONS(-pedantic)
    #ADD_DEFINITIONS(-ansi)
    IF(NOT WIN32)
        #only export symbols that are declared to be part of the api (non dll platforms)
        ADD_DEFINITIONS(-fvisibility=hidden)
        ADD_DEFINITIONS(-fvisibility-inlines-hidden)
    ENDIF(NOT WIN32)
    IF(USE_SIMD MATCHES SSE2)
        ADD_DEFINITIONS(-msse2)
        ADD_DEFINITIONS(-DUSE_SSE2)
    ENDIF()
    IF(USE_SIMD MATCHES AVX)
        ADD_DEFINITIONS(-march=native)
        ADD_DEFINITIONS(-DUSE_AVX)
    ENDIF()
ELSEIF(MSVC)
    IF(USE_SIMD MATCHES SSE2)
        ADD_DEFINITIONS(/arch:SSE2)
        ADD_DEFINITIONS(-DUSE_SSE2)
    ENDIF()
    IF(USE_SIMD MATCHES AVX)
        ADD_DEFINITIONS(/arch:AVX)
        ADD_DEFINITIONS(-DUSE_AVX)
    ENDIF()
ENDIF()

########################################################################
# Setup boost
########################################################################
MESSAGE(STATUS "Configuring Boost C++ Libraries...")

# Although not required on my system, some users have linking issues without
SET(BOOST_REQUIRED_COMPONENTS
    thread
    system
    chrono
)

if(UNIX AND NOT BOOST_ROOT AND EXISTS "/usr/lib64")
    list(APPEND BOOST_LIBRARYDIR "/usr/lib64") #fedora 64-bit fix
endif(UNIX AND NOT BOOST_ROOT AND EXISTS "/usr/lib64")

set(Boost_ADDITIONAL_VERSIONS
    "1.35.0" "1.35" "1.36.0" "1.36" "1.37.0" "1.37" "1.38.0" "1.38" "1.39.0" "1.39"
    "1.40.0" "1.40" "1.41.0" "1.41" "1.42.0" "1.42" "1.43.0" "1.43" "1.44.0" "1.44"
    "1.45.0" "1.45" "1.46.0" "1.46" "1.47.0" "1.47" "1.48.0" "1.48" "1.49.0" "1.49"
    "1.50.0" "1.50" "1.51.0" "1.51" "1.52.0" "1.52" "1.53.0" "1.53" "1.54.0" "1.54"
    "1.55.0" "1.55" "1.56.0" "1.56" "1.57.0" "1.57" "1.58.0" "1.58" "1.59.0" "1.59"
    "1.60.0" "1.60" "1.61.0" "1.61" "1.62.0" "1.62" "1.63.0" "1.63" "1.64.0" "1.64"
    "1.65.0" "1.65" "1.66.0" "1.66" "1.67.0" "1.67" "1.68.0" "1.68" "1.69.0" "1.69"
)

find_package(Boost COMPONENTS ${BOOST_REQUIRED_COMPONENTS})

if(NOT Boost_FOUND)
    message(WARNING "Boost required to build -- ignoring osmosdr support")
    return()
endif()

ADD_DEFINITIONS(-DBOOST_ALL_DYN_LINK)
link_directories(${Boost_LIBRARY_DIRS})

message(STATUS "Boost include directories: ${Boost_INCLUDE_DIRS}")
message(STATUS "Boost library directories: ${Boost_LIBRARY_DIRS}")
message(STATUS "Boost libraries: ${Boost_LIBRARIES}")

########################################################################
# Setup defines for high resolution timing
########################################################################
MESSAGE(STATUS "")
MESSAGE(STATUS "Configuring high resolution timing...")
INCLUDE(CheckCXXSourceCompiles)

SET(CMAKE_REQUIRED_LIBRARIES -lrt)
CHECK_CXX_SOURCE_COMPILES("
    #include <ctime>
    int main(){
        timespec ts;
        return clock_gettime(CLOCK_MONOTONIC, &ts);
    }
    " HAVE_CLOCK_GETTIME
)
UNSET(CMAKE_REQUIRED_LIBRARIES)

INCLUDE(CheckCXXSourceCompiles)
CHECK_CXX_SOURCE_COMPILES("
    #include <mach/mach_time.h>
    int main(){
        mach_timebase_info_data_t info;
        mach_timebase_info(&info);
        mach_absolute_time();
        return 0;
    }
    " HAVE_MACH_ABSOLUTE_TIME
)

CHECK_CXX_SOURCE_COMPILES("
    #include <Windows.h>
    int main(){
        LARGE_INTEGER value;
        QueryPerformanceCounter(&value);
        QueryPerformanceFrequency(&value);
        return 0;
    }
    " HAVE_QUERY_PERFORMANCE_COUNTER
)

macro(GR_OSMOSDR_APPEND_LIBS lib)
    list(APPEND osmo_libraries ${lib})
endmacro(GR_OSMOSDR_APPEND_LIBS)

IF(HAVE_CLOCK_GETTIME)
    MESSAGE(STATUS "  High resolution timing supported through clock_gettime.")
    SET(TIME_SPEC_DEFS HAVE_CLOCK_GETTIME)
    GR_OSMOSDR_APPEND_LIBS("-lrt")
ELSEIF(HAVE_MACH_ABSOLUTE_TIME)
    MESSAGE(STATUS "  High resolution timing supported through mach_absolute_time.")
    SET(TIME_SPEC_DEFS HAVE_MACH_ABSOLUTE_TIME)
ELSEIF(HAVE_QUERY_PERFORMANCE_COUNTER)
    MESSAGE(STATUS "  High resolution timing supported through QueryPerformanceCounter.")
    SET(TIME_SPEC_DEFS HAVE_QUERY_PERFORMANCE_COUNTER)
ELSE()
    MESSAGE(STATUS "  High resolution timing supported through microsec_clock.")
    SET(TIME_SPEC_DEFS HAVE_MICROSEC_CLOCK)
ENDIF()

SET_SOURCE_FILES_PROPERTIES(
    ${GR_OSMOSDR_SOURCE_DIR}/lib/time_spec.cc
    PROPERTIES COMPILE_DEFINITIONS "${TIME_SPEC_DEFS}"
)

########################################################################
# osmosdr library
########################################################################
set(osmosdr_srcs
    #${GR_OSMOSDR_SOURCE_DIR}/lib/source_impl.cc
    #${GR_OSMOSDR_SOURCE_DIR}/lib/sink_impl.cc
    ${GR_OSMOSDR_SOURCE_DIR}/lib/ranges.cc
    ${GR_OSMOSDR_SOURCE_DIR}/lib/time_spec.cc
    #${GR_OSMOSDR_SOURCE_DIR}/lib/device.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/compat/lib/io_signature.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/compat/lib/sync_block.cc
)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/compat/include)
include_directories(${GR_OSMOSDR_SOURCE_DIR}/include)
include_directories(${GR_OSMOSDR_SOURCE_DIR}/lib)
include_directories(${Boost_INCLUDE_DIRS})

add_library(SoapyOsmoSDR SHARED ${osmosdr_srcs})
target_link_libraries(SoapyOsmoSDR ${Boost_LIBRARIES} ${osmo_libraries})
set_target_properties(SoapyOsmoSDR PROPERTIES DEFINE_SYMBOL "gnuradio_osmosdr_EXPORTS")
set_target_properties(SoapyOsmoSDR PROPERTIES VERSION ${VERSION})
set_target_properties(SoapyOsmoSDR PROPERTIES SOVERSION ${SOVERSION})

install(TARGETS SoapyOsmoSDR
    LIBRARY DESTINATION lib${LIB_SUFFIX} # .so file
    ARCHIVE DESTINATION lib${LIB_SUFFIX} # .lib file
    RUNTIME DESTINATION bin              # .dll file
)

########################################################################
# Find build dependencies
########################################################################
INCLUDE(FindPkgConfig)
find_package(LibOsmoSDR)
find_package(LibRTLSDR)
find_package(LibMiriSDR)
find_package(LibHackRF)
find_package(LibAIRSPY)
find_package(LibbladeRF)
find_package(LibFreeSRP)

########################################################################
# module builder
########################################################################
function(GrOsmoSDRModule)

    include(CMakeParseArguments)
    CMAKE_PARSE_ARGUMENTS(OSMO_MODULE "" "TARGET" "SOURCES;SOURCE_DIR;LIBRARIES" ${ARGN})

    list(APPEND OSMO_MODULE_LIBRARIES SoapyOsmoSDR)
    list(APPEND OSMO_MODULE_LIBRARIES ${Boost_LIBRARIES})

    if (OSMO_MODULE_SOURCE_DIR)
        include_directories(${OSMO_MODULE_SOURCE_DIR})
        file(GLOB glob_srcs "${OSMO_MODULE_SOURCE_DIR}/*.*")
        list(APPEND OSMO_MODULE_SOURCES ${glob_srcs})
    endif()

    set(HAS_SINK 0)
    set(HAS_SOURCE 0)
    unset(LOCAL_INCLUDES)
    file(GLOB glob_hdrs "${OSMO_MODULE_SOURCE_DIR}/*.h")
    foreach(hdr ${glob_hdrs})
        set(LOCAL_INCLUDES "#include \"${hdr}\"\n${LOCAL_INCLUDES}")
        string(FIND "${hdr}" "source" source_pos)
        string(FIND "${hdr}" "sink" sink_pos)
        if ("${source_pos}" STREQUAL "-1")
        else()
            set(HAS_SOURCE 1)
        endif()
        if ("${sink_pos}" STREQUAL "-1")
        else()
            set(HAS_SINK 1)
        endif()
    endforeach(hdr)

    set(TARGET ${OSMO_MODULE_TARGET})
    configure_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/GrOsmoSDRRegister.in.cpp
        ${CMAKE_CURRENT_BINARY_DIR}/register_${OSMO_MODULE_TARGET}.cc
    @ONLY)
    list(APPEND OSMO_MODULE_SOURCES ${CMAKE_CURRENT_BINARY_DIR}/register_${OSMO_MODULE_TARGET}.cc)

    include(SoapySDRUtil)
    include_directories(${CMAKE_CURRENT_SOURCE_DIR})
    SOAPY_SDR_MODULE_UTIL(
        TARGET ${OSMO_MODULE_TARGET}Support
        SOURCES ${OSMO_MODULE_SOURCES}
        LIBRARIES ${OSMO_MODULE_LIBRARIES}
    )

endfunction(GrOsmoSDRModule)

function(GR_INCLUDE_SUBDIRECTORY target)

    unset(my_incs)
    unset(my_libs)
    string(TOUPPER "${target}" libname)
    macro(add_if_exists name)
        if (${name}_INCLUDE_DIRS)
            list(APPEND my_incs ${${name}_INCLUDE_DIRS})
        endif()
        if (${name}_LIBRARIES)
            list(APPEND my_libs ${${name}_LIBRARIES})
        endif()
    endmacro(add_if_exists)
    add_if_exists(${libname})
    add_if_exists(LIB${libname})
    add_if_exists(LIB${libname}SDR)

    message(STATUS "GrOsmoSDRModule(${target})")
    message(STATUS "    includes ${my_incs}")
    message(STATUS "    libraries ${my_libs}")
    include_directories(${my_incs})
    GrOsmoSDRModule(
        TARGET ${target}
        SOURCE_DIR ${GR_OSMOSDR_SOURCE_DIR}/lib/${target}
        LIBRARIES ${my_libs}
    )
endfunction(GR_INCLUDE_SUBDIRECTORY)

include(GrComponent)

########################################################################
# Setup OsmoSDR component
########################################################################
GR_REGISTER_COMPONENT("sysmocom OsmoSDR" ENABLE_OSMOSDR LIBOSMOSDR_FOUND)
if(ENABLE_OSMOSDR)
GR_INCLUDE_SUBDIRECTORY(osmosdr)
endif(ENABLE_OSMOSDR)

########################################################################
# Setup RTL component
########################################################################

#soapy osmo rtlsdr deprecated, use soapy RTL-SDR
#https://github.com/pothosware/SoapyRTLSDR/wiki
option(USE_OSMO_RTLSDR "Use rtlsdr through SoapyOsmo" OFF)

GR_REGISTER_COMPONENT("Osmocom RTLSDR" ENABLE_RTL LIBRTLSDR_FOUND;USE_OSMO_RTLSDR)
if(ENABLE_RTL)
GR_INCLUDE_SUBDIRECTORY(rtl)
endif(ENABLE_RTL)

########################################################################
# Setup MiriSDR component
########################################################################
GR_REGISTER_COMPONENT("Osmocom MiriSDR" ENABLE_MIRI LIBMIRISDR_FOUND)
if(ENABLE_MIRI)
GR_INCLUDE_SUBDIRECTORY(miri)
endif(ENABLE_MIRI)

########################################################################
# Setup HackRF component
########################################################################

#soapy osmo bladerf deprecated, use soapy hackrf
#https://github.com/pothosware/SoapyHackRF/wiki
option(USE_OSMO_HACKRF "Use hackrf through SoapyOsmo" OFF)

GR_REGISTER_COMPONENT("HackRF Jawbreaker" ENABLE_HACKRF LIBHACKRF_FOUND;USE_OSMO_HACKRF)
if(ENABLE_HACKRF)
GR_INCLUDE_SUBDIRECTORY(hackrf)
endif(ENABLE_HACKRF)

########################################################################
# Setup bladeRF component
########################################################################

#soapy osmo bladerf deprecated, use soapy bladerf
#https://github.com/pothosware/SoapyBladeRF/wiki
option(USE_OSMO_BLADERF "Use bladerf through SoapyOsmo" OFF)

GR_REGISTER_COMPONENT("nuand bladeRF" ENABLE_BLADERF LIBBLADERF_FOUND;USE_OSMO_BLADERF)
if(ENABLE_BLADERF)
GR_INCLUDE_SUBDIRECTORY(bladerf)
endif(ENABLE_BLADERF)

########################################################################
# Setup RFSPACE component
########################################################################

#https://github.com/pothosware/SoapyNetSDR/wiki
option(USE_OSMO_RFSPACE "Use netsdr through SoapyNetSDR" OFF)

GR_REGISTER_COMPONENT("RFSPACE Receivers" ENABLE_RFSPACE;USE_OSMO_RFSPACE)
if(ENABLE_RFSPACE)
GR_INCLUDE_SUBDIRECTORY(rfspace)
endif(ENABLE_RFSPACE)

########################################################################
# Setup AIRSPY component
########################################################################

#https://github.com/pothosware/SoapyAirspy/wiki
option(USE_OSMO_AIRSPY "Use airspy through SoapyAirspy" OFF)

GR_REGISTER_COMPONENT("AIRSPY Receiver" ENABLE_AIRSPY LIBAIRSPY_FOUND;USE_OSMO_AIRSPY)
if(ENABLE_AIRSPY)
GR_INCLUDE_SUBDIRECTORY(airspy)
endif(ENABLE_AIRSPY)

########################################################################
# Setup Red Pitaya component
########################################################################

#https://github.com/pothosware/SoapyRedPitaya/wiki
option(USE_OSMO_REDPITAYA "Use redpitaya through SoapyRedPitaya" OFF)

GR_REGISTER_COMPONENT("Red Pitaya SDR" ENABLE_REDPITAYA USE_OSMO_REDPITAYA)
if(ENABLE_REDPITAYA)
GR_INCLUDE_SUBDIRECTORY(redpitaya)
endif(ENABLE_REDPITAYA)

########################################################################
# Setup FreeSRP component
########################################################################
GR_REGISTER_COMPONENT("FreeSRP support" ENABLE_FREESRP LIBFREESRP_FOUND)
if(ENABLE_FREESRP)
GR_INCLUDE_SUBDIRECTORY(freesrp)
endif(ENABLE_FREESRP)

########################################################################
# Print Summary
########################################################################
GR_PRINT_COMPONENT_SUMMARY()
MESSAGE(STATUS "Building for version: ${VERSION}")
MESSAGE(STATUS "Using install prefix: ${CMAKE_INSTALL_PREFIX}")
