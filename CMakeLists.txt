cmake_minimum_required(VERSION 3.14 FATAL_ERROR)

set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

if(POLICY CMP0092)
    cmake_policy(SET CMP0092 NEW) # Don't add -W3 warning level by default.
endif()


file(READ "${CMAKE_CURRENT_SOURCE_DIR}/version.txt" VERSION_FILE_CONTENTS)
string(STRIP "${VERSION_FILE_CONTENTS}" VERSION_FILE_CONTENTS)
set(POLYHOOK_VERSION "${VERSION_FILE_CONTENTS}" CACHE STRING "Set version name")
set(POLYHOOK_PACKAGE "polyhook" CACHE STRING "Set package name")
string(REPLACE "v" "" POLYHOOK_VERSION "${POLYHOOK_VERSION}")
string(REGEX REPLACE "[.+-]" ";" POLYHOOK_VERSION_LIST ${POLYHOOK_VERSION})
list(GET POLYHOOK_VERSION_LIST 0 POLYHOOK_VERSION_MAJOR)
list(GET POLYHOOK_VERSION_LIST 1 POLYHOOK_VERSION_MINOR)
list(GET POLYHOOK_VERSION_LIST 2 POLYHOOK_VERSION_PATCH)

project(polyhook 
		VERSION "${POLYHOOK_VERSION_MAJOR}.${POLYHOOK_VERSION_MINOR}.${POLYHOOK_VERSION_PATCH}"
		DESCRIPTION "PolyHook Plugin" 
		HOMEPAGE_URL "https://github.com/untrustedmodders/polyhook" 
		LANGUAGES CXX
)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_MSVC_DEBUG_INFORMATION_FORMAT "$<$<CONFIG:Debug,RelWithDebInfo>:EditAndContinue>")

if(NOT CMAKE_BUILD_TYPE MATCHES "Debug|Devel|MinSizeRel|RelWithDebInfo|Release")
    message(STATUS "CMAKE_BUILD_TYPE not set, defaulting to Debug.")
    set(CMAKE_BUILD_TYPE Debug)
endif()

if(UNIX AND NOT APPLE)
    set(LINUX TRUE)
endif()

#
# Format
#
include(CompatFormat)

#
# Polyhook & DynLibUtils
#
include(FetchPolyhook)
include(FetchDynlibUtils)

#
# Plugin
#
file(GLOB_RECURSE PLUGIN_SOURCES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "src/*.cpp")

add_library(${PROJECT_NAME} SHARED ${PLUGIN_SOURCES})
target_include_directories(${PROJECT_NAME} PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})

target_link_libraries(${PROJECT_NAME} PRIVATE PolyHook_2 cpp-memory_utils)
target_include_directories(${PROJECT_NAME} PRIVATE ${PolyHook_2_SOURCE_DIR} ${dynlibutils_SOURCE_DIR}/include)

if(MSVC)
    target_compile_options(asmjit PUBLIC /wd5054)
elseif(MINGW)
    target_compile_options(asmjit PUBLIC -Wno-deprecated-enum-enum-conversion)
else()
    target_compile_options(asmjit PUBLIC -Wno-deprecated-anon-enum-enum-conversion -Wno-deprecated-enum-enum-conversion)
endif()

if(MSVC)
    #target_compile_options(${PROJECT_NAME} PRIVATE /W4 /WX)
else()
    #target_compile_options(${PROJECT_NAME} PRIVATE -Wextra -Wshadow -Wconversion -Wpedantic -Werror)
endif()

include(GenerateExportHeader)
generate_export_header(${PROJECT_NAME} EXPORT_MACRO_NAME PLUGIN_API EXPORT_FILE_NAME ${CMAKE_BINARY_DIR}/exports/plugin_export.h)
target_include_directories(${PROJECT_NAME} PRIVATE ${CMAKE_BINARY_DIR}/exports)

if(APPLE)
    target_link_libraries(${PROJECT_NAME} PRIVATE "-Wl,-exported_symbols_list,${CMAKE_CURRENT_SOURCE_DIR}/sym/exported_symbols.lds")
elseif(UNIX)
    target_link_libraries(${PROJECT_NAME} PRIVATE "-Wl,--version-script,${CMAKE_CURRENT_SOURCE_DIR}/sym/version_script.lds")
endif()

if(LINUX)
    target_link_libraries(${PROJECT_NAME} PRIVATE -static-libstdc++ -static-libgcc)
endif()

target_compile_definitions(${PROJECT_NAME} PRIVATE
        PLUGIFY_FORMAT_SUPPORT=$<BOOL:${COMPILER_SUPPORTS_FORMAT}> 
        PLUGIFY_IS_DEBUG=$<STREQUAL:${CMAKE_BUILD_TYPE},Debug>
        PLUGIFY_IS_RELEASE=$<STREQUAL:${CMAKE_BUILD_TYPE},Release>
        PLH_SOURCEHOOK=$<BOOL:${WIN32}>  # Enable sourcehook fix (for vhooks only)
)
if(NOT COMPILER_SUPPORTS_FORMAT)
    target_link_libraries(${PROJECT_NAME} PRIVATE fmt::fmt-header-only)
endif()

configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/polyhook.pplugin.in
    ${CMAKE_CURRENT_BINARY_DIR}/polyhook.pplugin
)