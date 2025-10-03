# Using the same minimum as the godot-cpp project
cmake_minimum_required(VERSION 3.17)

# Silence unused variable warning when specified from toolchain
if(CMAKE_C_COMPILER)
endif()

set(LIBNAME "EXTENSION-NAME" CACHE STRING "The name of the library")
set(GODOT_PROJECT_DIR "demo" CACHE STRING "The directory of a Godot project folder")

# Make sure all the dependencies are satisfied
find_package(Python3 3.4 REQUIRED)
find_program(GIT git REQUIRED)

# Ensure godot-cpp submodule has been updated
if(NOT EXISTS "${CMAKE_CURRENT_LIST_DIR}/godot-cpp/src")
    message(NOTICE "godot-cpp bindings source not found")
    message(NOTICE "initializing/updating the godot-cpp submodule...")

    # update the c++ bindings submodule to populate it with the necessary source for the library
    execute_process(
        COMMAND git submodule update --init godot-cpp
        WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
        COMMAND_ERROR_IS_FATAL ANY
    )
endif()
add_subdirectory(godot-cpp SYSTEM)

# Add godot-cpp's module path and include the exported functions.
# This is made available for documentation generation
set(CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH};${godot-cpp_SOURCE_DIR}/cmake")
include(GodotCPPModule)

# The godot-cpp target has some of useful properties attached that can be retrieved like so.
get_target_property(GODOTCPP_SUFFIX godot::cpp GODOTCPP_SUFFIX)
get_target_property(GODOTCPP_PLATFORM godot::cpp GODOTCPP_PLATFORM)

# Now we can specify our own project which will inherit any global cmake properties or variables that have been defined.
project(godot-cpp-template
    VERSION 1.0
    DESCRIPTION "This repository serves as a quickstart template for GDExtension development with Godot 4.0+."
    HOMEPAGE_URL "https://github.com/enetheru/godot-cpp-template/tree/main"
    LANGUAGES CXX
)

add_library(${LIBNAME} SHARED)

target_sources(${LIBNAME}
    PRIVATE
    src/register_types.cpp
    src/register_types.h
    src/example_class.cpp
    src/example_class.h
)

# Fetch a list of the xml files to use for documentation and add to our target
file(GLOB_RECURSE DOC_XML LIST_DIRECTORIES NO CONFIGURE_DEPENDS "${PROJECT_SOURCE_DIR}/doc_classes/*.xml")

# conditionally add doc data to compile output
if(DOC_XML)
    if(GODOTCPP_TARGET MATCHES "editor|template_debug")
        target_doc_sources(${LIBNAME} ${DOC_XML})
    endif()
endif()

target_link_libraries(${LIBNAME} PRIVATE godot-cpp)

# Require at least C++17 for this target
set_property(TARGET ${LIBNAME} PROPERTY CXX_STANDARD 17)

set_target_properties(${LIBNAME}
    PROPERTIES
    # The generator expression here prevents msvc from adding a Debug or Release subdir.
    RUNTIME_OUTPUT_DIRECTORY "$<1:${PROJECT_SOURCE_DIR}/bin/${GODOTCPP_PLATFORM}>"

    PREFIX ""
    OUTPUT_NAME "${LIBNAME}${GODOTCPP_SUFFIX}"
)

set(GODOT_PROJECT_BINARY_DIR "${PROJECT_SOURCE_DIR}/${GODOT_PROJECT_DIR}/bin/${GODOTCPP_PLATFORM}")

add_custom_command(TARGET ${LIBNAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy "$<TARGET_FILE:${LIBNAME}>" "${GODOT_PROJECT_BINARY_DIR}/$<TARGET_FILE_NAME:${LIBNAME}>"
)

# Attempt to read Godot executable path from the project's metadata, but allow manual override via cache
set(GODOT_EXECUTABLE "" CACHE FILEPATH "Path to the Godot executable")
if(NOT GODOT_EXECUTABLE)
    # Try to read from: <GODOT_PROJECT_DIR>/.godot/editor/project_metadata.cfg
    set(_GODOT_PROJECT_METADATA "${PROJECT_SOURCE_DIR}/${GODOT_PROJECT_DIR}/.godot/editor/project_metadata.cfg")
    if(EXISTS "${_GODOT_PROJECT_METADATA}")
        file(READ "${_GODOT_PROJECT_METADATA}" _GODOT_PROJECT_METADATA_CONTENTS)
        # Capture: executable_path="/path/to/Godot"
        string(REGEX MATCH "executable_path=\"([^\"]+)\"" _GODOT_EXEC_MATCH "${_GODOT_PROJECT_METADATA_CONTENTS}")
        if(CMAKE_MATCH_1)
            set(GODOT_EXECUTABLE "${CMAKE_MATCH_1}" CACHE FILEPATH "Path to the Godot executable" FORCE)
            message(STATUS "Godot executable detected from project metadata: ${GODOT_EXECUTABLE}")
        else()
            message(WARNING "Could not find 'executable_path' in ${_GODOT_PROJECT_METADATA}. You can set GODOT_EXECUTABLE manually in CMake cache.")
        endif()
    else()
        message(WARNING "Godot project metadata not found at ${_GODOT_PROJECT_METADATA}. You can set GODOT_EXECUTABLE manually in CMake cache.")
    endif()
endif()

add_custom_target(run-godot
        COMMAND ${GODOT_EXECUTABLE} --path ${PROJECT_SOURCE_DIR}/${GODOT_PROJECT_DIR}
        DEPENDS ${LIBNAME}
        WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/${GODOT_PROJECT_DIR}
        USES_TERMINAL
)

message(GODOT_EXECUTABLE = ${GODOT_EXECUTABLE})