# Copyright: Universidad Carlos III de Madrid (C) 2017;
# Authors: Raul de Santos Rico & Juan Gonzalez Victores
# CopyPolicy: Released under the terms of the GNU GPL v2.0.

cmake_minimum_required(VERSION 2.8.11 FATAL_ERROR)

# Safety check against missing dummy library for system-available COLOR_DEBUG.
# Remove if cmake_minimum_required() above is set to 3.0 or later.
if(POLICY CMP0046)
    cmake_policy(SET CMP0046 NEW)
endif()

project(ROBOTICSLAB_SPEECH)

# Force building shared libraries and YARP DL modules.
set(BUILD_SHARED_LIBS TRUE CACHE INTERNAL "hide this!")

# Add suffix for debug libraries.
if(MSVC)
    message(STATUS "Running on windows")
    set(CMAKE_DEBUG_POSTFIX "d")
endif()

# Let the user specify a configuration (only single-config generators).
if(NOT CMAKE_CONFIGURATION_TYPES)
    # Possible values.
    set(_configurations Debug Release MinSizeRel RelWithDebInfo)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS ${_configurations})

    foreach(_conf ${_configurations})
        set(_conf_string "${_conf_string} ${_conf}")
    endforeach()

    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY HELPSTRING
                 "Choose the type of build, options are:${_conf_string}")

    if(NOT CMAKE_BUILD_TYPE)
        # Encourage the user to specify build type.
        message(STATUS "Setting build type to 'Release' as none was specified.")
        set_property(CACHE CMAKE_BUILD_TYPE PROPERTY VALUE Release)
    endif()
endif()

# Pick up our cmake modules.
list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)

# Bootstrap YCM. Stick to v0.2.2 because of:
# https://github.com/robotology/ycm/issues/118
# ...and to keep YCM itself compatible with cmake_minimum_required().
set(YCM_TAG v0.2.2)
include(YCMBootstrap)

# Standard installation directories.
include(GNUInstallDirs)

# Control where libraries and executables are placed during the build.
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_BINDIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})

# Enabling coverage.
option(ENABLE_coverage "Choose if you want to enable coverage collection" OFF)

if(ENABLE_coverage)
    # List supported compilers.
    if(CMAKE_COMPILER_IS_GNUCXX)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-arcs -ftest-coverage")
    endif()
endif()

# Superbuild phase - include color-debug.
include(FindOrBuildPackage)
find_or_build_package(COLOR_DEBUG)

# find_or_build_package() doesn't bring into scope COLOR_DEBUG_INCLUDE_DIRS
# nor other config variables. If a system copy is found, we have to call the
# find_package() command and proceed as usual.
if(USE_SYSTEM_COLOR_DEBUG)
    find_package(COLOR_DEBUG REQUIRED)
    include_directories(${COLOR_DEBUG_INCLUDE_DIRS})
endif()

# Store common include directories in global scope.
set_property(GLOBAL PROPERTY ROBOTICSLAB_SPEECH_INCLUDE_DIRS)

# Store IDL files in global scope.
set_property(GLOBAL PROPERTY ROBOTICSLAB_SPEECH_IDL_INCLUDES)
set_property(GLOBAL PROPERTY ROBOTICSLAB_SPEECH_IDL_SOURCES)

# Store list of exported targets in global scope.
set_property(GLOBAL PROPERTY ROBOTICSLAB_SPEECH_TARGETS)

# Add main contents.
add_subdirectory(cmake)
add_subdirectory(share)
add_subdirectory(libraries)
add_subdirectory(programs)

# Store the package in the user registry.
export(PACKAGE ROBOTICSLAB_SPEECH)

# Retrieve global properties.
get_property(_common_includes GLOBAL PROPERTY ROBOTICSLAB_SPEECH_INCLUDE_DIRS)
get_property(_idl_includes GLOBAL PROPERTY ROBOTICSLAB_SPEECH_IDL_INCLUDES)
get_property(_idl_sources GLOBAL PROPERTY ROBOTICSLAB_SPEECH_IDL_SOURCES)
get_property(_exported_targets GLOBAL PROPERTY ROBOTICSLAB_SPEECH_TARGETS)

# CMake installation path.
if(WIN32)
    set(_cmake_destination cmake)
else()
    set(_cmake_destination ${CMAKE_INSTALL_LIBDIR}/cmake/ROBOTICSLAB_SPEECH)
endif()

# Create and install config files.
include(CMakePackageConfigHelpers)

# Set exported variables (build tree).
set(ROBOTICSLAB_SPEECH_INCLUDE_DIR "${_common_includes}")
set(ROBOTICSLAB_SPEECH_IDL_INCLUDE "${_idl_includes}")
set(ROBOTICSLAB_SPEECH_IDL_SOURCE "${_idl_sources}")
set(ROBOTICSLAB_SPEECH_MODULE_DIR ${CMAKE_SOURCE_DIR}/cmake)

# <pkg>Config.cmake (build tree).
configure_package_config_file(${CMAKE_SOURCE_DIR}/cmake/template/ROBOTICSLAB_SPEECHConfig.cmake.in
                              ${CMAKE_BINARY_DIR}/ROBOTICSLAB_SPEECHConfig.cmake
                              INSTALL_DESTINATION ${CMAKE_BINARY_DIR}
                              INSTALL_PREFIX ${CMAKE_BINARY_DIR}
                              PATH_VARS ROBOTICSLAB_SPEECH_INCLUDE_DIR
                                        ROBOTICSLAB_SPEECH_IDL_INCLUDE
                                        ROBOTICSLAB_SPEECH_IDL_SOURCE
                                        ROBOTICSLAB_SPEECH_MODULE_DIR
                              NO_CHECK_REQUIRED_COMPONENTS_MACRO)

# Set exported variables (install tree).
set(ROBOTICSLAB_SPEECH_INCLUDE_DIR ${CMAKE_INSTALL_INCLUDEDIR})
set(ROBOTICSLAB_SPEECH_IDL_INCLUDE)
set(ROBOTICSLAB_SPEECH_IDL_SOURCE)

if(DEFINED ROBOTICSLAB-SPEECH_DATA_INSTALL_DIR)
    find_package(YARP QUIET) # retrieve YARP version string

    set(ROBOTICSLAB_SPEECH_IDL_INCLUDE ${ROBOTICSLAB-SPEECH_DATA_INSTALL_DIR}/idls)

    foreach(_file ${_idl_sources})
        get_filename_component(_filename ${_file} NAME)

        if(NOT YARP_VERSION_SHORT VERSION_LESS 2.3.70.1)
            list(APPEND ROBOTICSLAB_SPEECH_IDL_SOURCE ${ROBOTICSLAB-SPEECH_DATA_INSTALL_DIR_FULL}/idls/${_filename})
        else()
            list(APPEND ROBOTICSLAB_SPEECH_IDL_SOURCE ${CMAKE_INSTALL_PREFIX}/${ROBOTICSLAB-SPEECH_DATA_INSTALL_DIR}/idls/${_filename})
        endif()
    endforeach()
endif()

set(ROBOTICSLAB_SPEECH_MODULE_DIR ${CMAKE_INSTALL_DATADIR}/ROBOTICSLAB_SPEECH/cmake)

# <pkg>Config.cmake (install tree).
configure_package_config_file(${CMAKE_SOURCE_DIR}/cmake/template/ROBOTICSLAB_SPEECHConfig.cmake.in
                              ${CMAKE_BINARY_DIR}/ROBOTICSLAB_SPEECHConfig.cmake.install
                              INSTALL_DESTINATION ${_cmake_destination}
                              PATH_VARS ROBOTICSLAB_SPEECH_INCLUDE_DIR
                                        ROBOTICSLAB_SPEECH_IDL_INCLUDE
                                        ROBOTICSLAB_SPEECH_IDL_SOURCE
                                        ROBOTICSLAB_SPEECH_MODULE_DIR
                              NO_CHECK_REQUIRED_COMPONENTS_MACRO)

# Install <pkg>Config.cmake.
install(FILES ${CMAKE_BINARY_DIR}/ROBOTICSLAB_SPEECHConfig.cmake.install
        RENAME ROBOTICSLAB_SPEECHConfig.cmake
        DESTINATION ${_cmake_destination})

# Export library targets if enabled.
# https://github.com/roboticslab-uc3m/project-generator/issues/19
if(_exported_targets)
    # <pkg>Targets.cmake (build tree).
    # In CMake 3.0 or later: export(EXPORT ROBOTICSLAB_SPEECH...)
    export(TARGETS ${_exported_targets}
           NAMESPACE ROBOTICSLAB::
           FILE ROBOTICSLAB_SPEECHTargets.cmake)

    # <pkg>Targets.cmake (install tree).
    install(EXPORT ROBOTICSLAB_SPEECH
            DESTINATION ${_cmake_destination}
            NAMESPACE ROBOTICSLAB::
            FILE ROBOTICSLAB_SPEECHTargets.cmake)
endif()

# Configure and create uninstall target.
include(AddUninstallTarget)
