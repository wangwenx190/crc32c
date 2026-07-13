cmake_minimum_required(VERSION 4.0)

project(crc32c
    VERSION 1.1.0
    DESCRIPTION "CRC32C implementation with hardware acceleration"
    LANGUAGES CXX
)

if(WIN32)
    enable_language(RC)
endif()
if(NOT DEFINED CMAKE_BUILD_TYPE AND NOT DEFINED CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE "Release")
endif()
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_VISIBILITY_INLINES_HIDDEN ON)
set(CMAKE_CXX_VISIBILITY_PRESET "hidden")
if(NOT "x${CMAKE_CXX_FLAGS}" STREQUAL "x")
    if(MSVC)
        string(REGEX REPLACE " [/-]W[01234] " " " CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS})
    else()
        string(REGEX REPLACE " -W(all)?(extra)? " " " CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS})
        string(REGEX REPLACE " -[W]?pedantic " " " CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS})
    endif()
endif()
string(APPEND CMAKE_CXX_FLAGS " -w ")

set(__top_level ON)
if("x${CMAKE_SOURCE_DIR}" STREQUAL "x${PROJECT_SOURCE_DIR}")
    message("CRC32C: Building as a top-level project.")
else()
    set(__top_level OFF)
    message("CRC32C: Building as a subproject.")
endif()

option(CRC32C_BUILD_SHARED "Build shared library edition of CRC32C." ON)
option(CRC32C_BUILD_STATIC "Build static library edition of CRC32C." ON)
option(CRC32C_INSTALL "Enable installation for CRC32C." OFF)
if(NOT CRC32C_BUILD_SHARED AND NOT CRC32C_BUILD_STATIC)
    message(SEND_ERROR "You MUST build either the shared library or the static library or both of them!")
endif()

if(APPLE AND CMAKE_OSX_ARCHITECTURES)
    string(TOLOWER "${CMAKE_OSX_ARCHITECTURES}" __architecture)
elseif(CMAKE_CXX_COMPILER_ARCHITECTURE_ID)
    string(TOLOWER "${CMAKE_CXX_COMPILER_ARCHITECTURE_ID}" __architecture)
else()
    string(TOLOWER "${CMAKE_SYSTEM_PROCESSOR}" __architecture)
endif()
set(__x64 OFF)
set(__arm64 OFF)
foreach(__arch IN LISTS __architecture)
    if(__arch MATCHES "^(amd64|x86_64|x64)$")
        set(__x64 ON)
    elseif(__arch MATCHES "^(aarch64|arm64|arm64e)$")
        set(__arm64 ON)
    endif()
endforeach()

function(add_kernel_flags source architecture)
    if(APPLE AND __architecture MATCHES ";")
        foreach(flag IN LISTS ARGN)
            set_property(SOURCE "${source}" APPEND PROPERTY COMPILE_OPTIONS
                "-Xarch_${architecture}" "${flag}")
        endforeach()
    else()
        set_property(SOURCE "${source}" APPEND PROPERTY COMPILE_OPTIONS ${ARGN})
    endif()
endfunction()

include(TestBigEndian)
test_big_endian(BYTE_ORDER_BIG_ENDIAN)
set(HAVE_BUILTIN_PREFETCH 0)
if(CMAKE_CXX_COMPILER_ID MATCHES "^(GNU|Clang|AppleClang)$" AND NOT MSVC)
    set(HAVE_BUILTIN_PREFETCH 1)
endif()
set(CRC32C_BUILD_SSE42 ${__x64})
set(CRC32C_BUILD_ARM64 OFF)
if(__x64 AND NOT MSVC)
    add_kernel_flags(src/crc32c_sse42.cc x86_64 -msse4.2)
endif()
if(__arm64 AND NOT MSVC)
    set(CRC32C_BUILD_ARM64 ON)
    add_kernel_flags(src/crc32c_arm64.cc arm64 -march=armv8-a+crc+crypto)
    if(CMAKE_SYSTEM_NAME MATCHES "^(Linux|Android)$")
        include(CheckCXXSymbolExists)
        check_cxx_symbol_exists(getauxval "sys/auxv.h" HAVE_STRONG_GETAUXVAL)
        if(NOT HAVE_STRONG_GETAUXVAL AND CMAKE_CXX_COMPILER_ID MATCHES "^(GNU|Clang|AppleClang)$")
            set(HAVE_WEAK_GETAUXVAL 1)
        endif()
    endif()
endif()
configure_file(
    src/crc32c_config.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/include/crc32c/crc32c_config.h
)

set(__targets)
if(CRC32C_BUILD_SHARED)
    add_library(${PROJECT_NAME}_shared SHARED)
    add_library(${PROJECT_NAME}::${PROJECT_NAME}_shared ALIAS ${PROJECT_NAME}_shared)
    set_target_properties(${PROJECT_NAME}_shared PROPERTIES OUTPUT_NAME ${PROJECT_NAME})
    target_compile_definitions(${PROJECT_NAME}_shared PRIVATE CRC32C_EXPORTS)
    if(WIN32)
        target_sources(${PROJECT_NAME}_shared PRIVATE ${PROJECT_NAME}.rc)
    endif()
    list(APPEND __targets ${PROJECT_NAME}_shared)
endif()
if(CRC32C_BUILD_STATIC)
    add_library(${PROJECT_NAME}_static STATIC)
    add_library(${PROJECT_NAME}::${PROJECT_NAME}_static ALIAS ${PROJECT_NAME}_static)
    target_compile_definitions(${PROJECT_NAME}_static PUBLIC CRC32C_STATIC_DEFINE)
    list(APPEND __targets ${PROJECT_NAME}_static)
endif()
if(CRC32C_BUILD_SHARED AND NOT CRC32C_BUILD_STATIC)
    add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME}_shared)
endif()
if(CRC32C_BUILD_STATIC AND NOT CRC32C_BUILD_SHARED)
    add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME}_static)
endif()

include(GNUInstallDirs)
file(GLOB_RECURSE __sources LIST_DIRECTORIES false CONFIGURE_DEPENDS
    "./include/crc32c/*.h" "./src/*.h" "./src/*.cc"
)
foreach(__target IN LISTS __targets)
    target_sources(${__target} PRIVATE ${__sources})
    if(NOT MSVC AND CMAKE_CXX_COMPILER_ID MATCHES "^(GNU|Clang|AppleClang)$")
        # Match upstream's exception-free dispatch and kernels.
        target_compile_options(${__target} PRIVATE -fno-exceptions -fno-rtti)
    endif()
    target_include_directories(${__target}
        PUBLIC
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
            $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
        PRIVATE
            ${CMAKE_CURRENT_BINARY_DIR}/include
            ${CMAKE_CURRENT_SOURCE_DIR}/src
    )
    if(__top_level)
        if(WIN32)
            target_compile_definitions(${__target} PRIVATE
                UNICODE _UNICODE
                NOMINMAX
                WIN32_LEAN_AND_MEAN
            )
        endif()
        if(MSVC)
            target_compile_options(${__target} PRIVATE
                /utf-8 /permissive- /Zc:__cplusplus
                $<$<NOT:$<CONFIG:Debug>>:/Gw /Gy /Zc:inline>
            )
            target_link_options(${__target} PRIVATE
                $<$<NOT:$<CONFIG:Debug>>:/OPT:REF /OPT:ICF>
            )
        else()
            target_compile_options(${__target} PRIVATE
                $<$<NOT:$<CONFIG:Debug>>:-ffunction-sections -fdata-sections>
            )
            if(APPLE)
                target_link_options(${__target} PRIVATE
                    $<$<NOT:$<CONFIG:Debug>>:-Wl,-dead_strip>
                )
            else()
                target_link_options(${__target} PRIVATE
                    $<$<NOT:$<CONFIG:Debug>>:-Wl,--gc-sections>
                )
            endif()
        endif()
    endif()
    set_target_properties(${__target} PROPERTIES VERSION ${PROJECT_VERSION})
    get_target_property(__type ${__target} TYPE)
    if(NOT "x${__type}" STREQUAL "xSTATIC_LIBRARY")
        set_target_properties(${__target} PROPERTIES SOVERSION ${PROJECT_VERSION_MAJOR})
    endif()
    if(CRC32C_INSTALL)
        install(TARGETS ${__target}
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        )
    endif()
endforeach()

if(CRC32C_INSTALL)
    install(
        FILES include/crc32c/crc32c.h
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/crc32c
    )
endif()
