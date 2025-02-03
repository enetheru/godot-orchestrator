## This file is part of the Godot Orchestrator project.
##
## Copyright (c) 2023-present Crater Crash Studios LLC and its contributors.
##
## Licensed under the Apache License, Version 2.0 (the "License");
## you may not use this file except in compliance with the License.
## You may obtain a copy of the License at
##
##		http://www.apache.org/licenses/LICENSE-2.0
##
## Unless required by applicable law or agreed to in writing, software
## distributed under the License is distributed on an "AS IS" BASIS,
## WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
## See the License for the specific language governing permissions and
## limitations under the License.
##
CMAKE_MINIMUM_REQUIRED(VERSION 3.28 FATAL_ERROR)

#[================================[ Options ]================================]

set( WANT_GODOT OFF CACHE BOOL "Whether to fetch and update godot")

#[==============================[ Dependencies ]==============================]
include( FetchContent )

# Python
find_package(Python3 3.4 REQUIRED) # pathlib should be present

# Godot-cpp
set( GODOTCPP_GIT_URL "https://github.com/enetheru/godot-cpp.git" CACHE STRING "The git url of godot-cpp" )
set( GODOTCPP_GIT_TAG "master" CACHE STRING "The git branch of godot-cpp to fetch" )
set( GODOT_BUILD_PROFILE "${CMAKE_SOURCE_DIR}/extern/godot-cpp-profile.json" )
FetchContent_Declare( godot-cpp
        GIT_REPOSITORY ${GODOTCPP_GIT_URL}
        GIT_TAG "${GODOTCPP_GIT_TAG}"
        GIT_PROGRESS ON
        SYSTEM              # requires cmake 3.25
        EXCLUDE_FROM_ALL    # Requires cmake 3.28
        SOURCE_DIR "${CMAKE_SOURCE_DIR}/extern/godot-cpp"
)
FetchContent_MakeAvailable( godot-cpp )

if( WANT_GODOT )
    set( GODOT_GIT_URL "https://github.com/godotengine/godot.git" CACHE STRING "The git url of godot" )
    set( GODOT_GIT_TAG "master" CACHE STRING "The git branch of godot-cpp to fetch" )
    FetchContent_Declare( godot
            GIT_REPOSITORY ${GODOT_GIT_URL}
            GIT_TAG "${GODOT_GIT_TAG}"
            GIT_PROGRESS ON
            GIT_SHALLOW ON
            SYSTEM
            EXCLUDE_FROM_ALL
            SOURCE_DIR "${CMAKE_SOURCE_DIR}/extern/godot"
    )
    FetchContent_MakeAvailable( godot )
endif()

SET(GDEXTENSION_LIB_NAME orchestrator)
SET(GDEXTENSION_LIB_PATH "${CMAKE_CURRENT_SOURCE_DIR}/project/addons/orchestrator")

# Configurable options
OPTION(AUTOFORMAT_SRC_ON_CONFIGURE "If enabled, clang-format will be used to format all sources in /src during configuration" OFF)

# Set basic CMAKE properties
SET(CMAKE_CXX_STANDARD 20)
SET(CMAKE_CXX_EXTENSIONS ON)
SET(CMAKE_CXX_STANDARD_REQUIRED ON)
SET(CMAKE_COLOR_DIAGNOSTICS ON)
SET(CMAKE_MESSAGE_LOG_LEVEL STATUS)

# Get the current repository Git commit hash
# This is used when creating the data from VERSION to show the commit hash in the about box
FIND_PACKAGE(Git QUIET)
IF (GIT_FOUND)
    EXECUTE_PROCESS(
            COMMAND ${GIT_EXECUTABLE} rev-parse HEAD
            WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
            OUTPUT_VARIABLE GIT_COMMIT_HASH
            OUTPUT_STRIP_TRAILING_WHITESPACE)

    EXECUTE_PROCESS(
            COMMAND ${GIT_EXECUTABLE} rev-parse HEAD
            WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/extern/godot-cpp"
            OUTPUT_VARIABLE GODOT_CPP_GIT_COMMIT_HASH
            OUTPUT_STRIP_TRAILING_WHITESPACE)
ELSE ()
    # In this case the about box will simply show an unknown git commit
    SET(GIT_COMMIT_HASH "<Unknown>")
    SET(GODOT_CPP_GIT_COMMIT_HASH "<Unknown>")
ENDIF ()

# Setup the CMAKE_MODULE_PATH
LIST(APPEND CMAKE_MODULE_PATH
        "${CMAKE_CURRENT_SOURCE_DIR}/cmake/")

# Include and execute various CMAKE modules
INCLUDE(generate-authors)
INCLUDE(generate-license)
INCLUDE(generate-version)
INCLUDE(generate-donors)
INCLUDE(godot-extension-db-generator)

# Generation steps
GENERATE_AUTHORS()
GENERATE_LICENSE()
GENERATE_VERSION()
GENERATE_DONORS()
GENERATE_GODOT_EXTENSION_DB()

# Configure project
PROJECT("${GDEXTENSION_LIB_NAME}" LANGUAGES C CXX VERSION ${RESOLVED_VERSION})

# Generate library resource
IF (WIN32)
    SET(win32_product_name "${RESOLVED_VERSION_NAME}")
    SET(win32_file_version "${PROJECT_VERSION_MAJOR},${PROJECT_VERSION_MINOR},${PROJECT_VERSION_PATCH},0")
    SET(win32_product_version "${PROJECT_VERSION_MAJOR},${PROJECT_VERSION_MINOR},${PROJECT_VERSION_PATCH},0")
    SET(win32_file_description "Godot ${win32_product_name} Plug-in")
    CONFIGURE_FILE(cmake/templates/windows.rc.in ${CMAKE_CURRENT_BINARY_DIR}/_generated/version.rc @ONLY)
ENDIF ()

# MacOS universal binary support
IF (APPLE)
    SET(CMAKE_OSX_ARCHITECTURES "x86_64;arm64" CACHE STRING "Build architectures for OSX" FORCE)
ENDIF ()

# Configure godot-cpp - a statically linked library to this project.
INCLUDE(godot-dev-configuration)

# Generate Doc Data
file( GLOB_RECURSE DOC_XML LIST_DIRECTORIES NO CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/doc_classes/*.xml" )
SET(DOC_DATA_CPP_FILE "${CMAKE_BINARY_DIR}/_generated/doc_data.cpp")
generate_doc_source( "${DOC_DATA_CPP_FILE}" "${DOC_XML}" )

# Library sources
FILE(GLOB_RECURSE gdext_sources
        CONFIGURE_DEPENDS
        "${CMAKE_CURRENT_SOURCE_DIR}/src/*.[hc]"
        "${CMAKE_CURRENT_SOURCE_DIR}/src/*.[hc]pp"
        # Include windows version resource, if exists
        "${CMAKE_CURRENT_BINARY_DIR}/_generated/version.rc")

# GDExtension library
ADD_LIBRARY(${PROJECT_NAME} SHARED
        ${gdext_sources}
        # Includes the generated doc data from /doc_classes
        "${DOC_DATA_CPP_FILE}")

find_program(ccache_exe ccache)
IF (ccache_exe)
    SET(CMAKE_VS_GLOBALS "TrackFileAccess=false" "UseMultiToolTask=true" "DebugInformationFormat=OldStyle")
ENDIF ()

# Add the special "_generated" directory to the include list
# This is where the generator CMAKE module steps creates automated files
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR}/_generated)

# Include directories for GDExtension library
TARGET_INCLUDE_DIRECTORIES(${PROJECT_NAME} PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/src")

IF (AUTOFORMAT_SRC_ON_CONFIGURE MATCHES ON)
    include(clang-format)
ENDIF ()

# Dependency linking
TARGET_LINK_LIBRARIES(${PROJECT_NAME} PUBLIC godot-cpp.editor)

### Get useful properties of the library
get_target_property( GODOT_PLATFORM godot-cpp::editor GODOT_PLATFORM   )
get_target_property( GODOT_ARCH     godot-cpp::editor GODOT_ARCH       )

SET_TARGET_PROPERTIES(${PROJECT_NAME}
        PROPERTIES

        POSITION_INDEPENDENT_CODE ON
        CMAKE_EXPORT_COMPILE_OPTIMIZATION_RELEASE ON
        CMAKE_INTERPROCEDURAL_OPTIMIZATION_RELEASE ON

        ARCHIVE_OUTPUT_DIRECTORY "$<1:${GDEXTENSION_LIB_PATH}>"
        LIBRARY_OUTPUT_DIRECTORY "$<1:${GDEXTENSION_LIB_PATH}>"
        RUNTIME_OUTPUT_DIRECTORY "$<1:${GDEXTENSION_LIB_PATH}>"
        CMAKE_PDB_OUTPUT_DIRECTORY "$<1:${GDEXTENSION_LIB_PATH}>"
        CMAKE_COMPILE_PDB_OUTPUT_DIRECTORY "$<1:${GDEXTENSION_LIB_PATH}>"

        PREFIX ""
        OUTPUT_NAME "${PROJECT_NAME}.${GODOT_PLATFORM}.${GODOT_ARCH}.$<LOWER_CASE:$<CONFIG>>")

INCLUDE(cmake-utils)

PRINT_PROJECT_VARIABLES()
