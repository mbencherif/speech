# Copyright: (C) 2017 Universidad Carlos III de Madrid
# Author: Juan G. Victores & Raul de Santos Rico
# CopyPolicy: Released under the terms of the LGPLv2.1 or later, see LGPL.TXT

yarp_prepare_plugin(Espeak
                    CATEGORY device
                    TYPE roboticslab::Espeak
                    INCLUDE Espeak.hpp
                    DEFAULT ON)

if(NOT SKIP_Espeak)

find_package(YARP REQUIRED)
find_package(Espeak REQUIRED)

include(YarpIDL)

yarp_idl_to_dir(speech.thrift ${CMAKE_CURRENT_BINARY_DIR} idl_sources idl_headers idl_include_paths)

get_property(_common_dirs GLOBAL PROPERTY ROBOTICSLAB_VISION_INCLUDE_DIRS)

include_directories(${CMAKE_CURRENT_SOURCE_DIR} # yarp plugin builder needs this
                    ${ESPEAK_INCLUDE_DIRS}
                    ${idl_include_paths}
                    ${_common_dirs})

yarp_add_plugin(Espeak Espeak.hpp
                       Espeak.cpp
                       ${idl_sources}
                       ${idl_headers})

add_dependencies(Espeak COLOR_DEBUG)

target_link_libraries(Espeak YARP::YARP_OS
                             YARP::YARP_dev
                             YARP::YARP_sig
                             ${ESPEAK_LIBRARIES})

yarp_install(TARGETS Espeak
             COMPONENT runtime
             LIBRARY DESTINATION ${YARP_DYNAMIC_PLUGINS_INSTALL_DIR}
             ARCHIVE DESTINATION ${YARP_STATIC_PLUGINS_INSTALL_DIR})

yarp_install(FILES Espeak.ini
             COMPONENT runtime
             DESTINATION ${YARP_PLUGIN_MANIFESTS_INSTALL_DIR})

# Already included in /share?
if(NOT DEFINED ROBOTICSLAB-SPEECH_DATA_INSTALL_DIR)
    include(YarpInstallationHelpers)
    yarp_configure_external_installation(roboticslab-speech)
endif()

yarp_install(FILES ${thrift_files}
                   ${idl_sources}
                   ${idl_headers} 
             DESTINATION ${ROBOTICSLAB-SPEECH_DATA_INSTALL_DIR}/idls)

set_property(GLOBAL APPEND PROPERTY ROBOTICSLAB_SPEECH_IDL_INCLUDES ${idl_include_paths})
set_property(GLOBAL APPEND PROPERTY ROBOTICSLAB_SPEECH_IDL_SOURCES ${idl_sources} ${idl_headers})

endif()
