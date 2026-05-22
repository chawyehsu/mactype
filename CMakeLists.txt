cmake_minimum_required(VERSION 3.20)

if(POLICY CMP0091)
    cmake_policy(SET CMP0091 NEW)
endif()

project(gdipp LANGUAGES C CXX RC)

include(GNUInstallDirs)

if(NOT WIN32)
    message(FATAL_ERROR "gdipp only supports Windows builds.")
endif()

option(GDIPP_BUILD_MACLOADER "Build the macloader executable" ON)
option(GDIPP_USE_DETOURS "Build gdipp with the Detours hook backend" ON)
option(GDIPP_ENABLE_TRACE "Enable TRACE instrumentation in gdipp" OFF)
option(GDIPP_ENABLE_INFINALITY "Enable Infinality-specific code paths" OFF)

if(NOT GDIPP_USE_DETOURS)
    message(FATAL_ERROR "The initial CMake port currently supports only the Detours backend. Set GDIPP_USE_DETOURS=ON.")
endif()

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(GDIPP_CORE_OUTPUT_NAME MacType64.Core)
    set(GDIPP_MACLOADER_OUTPUT_NAME MacLoader64)
    set(GDIPP_DETOURS_NAMES detours64 detours)
    set(GDIPP_WOW64EXT_REQUIRED OFF)
else()
    set(GDIPP_CORE_OUTPUT_NAME MacType.Core)
    set(GDIPP_MACLOADER_OUTPUT_NAME MacLoader)
    set(GDIPP_DETOURS_NAMES detours detours64)
    set(GDIPP_WOW64EXT_REQUIRED ON)
endif()

find_package(Freetype REQUIRED)
find_package(IniParser CONFIG REQUIRED)
find_package(nlohmann_json CONFIG REQUIRED)

find_path(DETOURS_INCLUDE_DIR
    NAMES detours.h
    PATH_SUFFIXES include
)
find_library(DETOURS_LIBRARY
    NAMES ${GDIPP_DETOURS_NAMES}
    PATH_SUFFIXES lib
)

if(NOT DETOURS_INCLUDE_DIR OR NOT DETOURS_LIBRARY)
    message(FATAL_ERROR "Detours not found. Run pixi build or use a pixi shell so CMAKE_PREFIX_PATH points at the pixi-installed detours package.")
endif()

if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    get_filename_component(GDIPP_DETOURS_LIBRARY_NAME "${DETOURS_LIBRARY}" NAME)
    if(NOT GDIPP_DETOURS_LIBRARY_NAME STREQUAL "detours64.lib")
        configure_file(
            "${DETOURS_LIBRARY}"
            "${CMAKE_CURRENT_BINARY_DIR}/detours64.lib"
            COPYONLY
        )
    endif()
endif()

if(GDIPP_WOW64EXT_REQUIRED)
    find_library(WOW64EXT_LIBRARY
        NAMES wow64ext wow64ext_dbg
        PATHS
            ${CMAKE_CURRENT_SOURCE_DIR}
            ${CMAKE_CURRENT_SOURCE_DIR}/lib
            $ENV{WOW64EXT_PATH}
            $ENV{WOW64EXT_DIR}
        PATH_SUFFIXES lib
    )
    if(NOT WOW64EXT_LIBRARY)
        message(FATAL_ERROR "wow64ext.lib is required for 32-bit builds. Put it in lib/ or make it discoverable via CMAKE_PREFIX_PATH.")
    endif()
endif()

set(GDIPP_CORE_SOURCES
    cache.cpp
    common.cpp
    directwrite.cpp
    dll.cpp
    dynCodeHelper.cpp
    EventLogging.cpp
    expfunc.cpp
    expfunc.def
    ft.cpp
    ft2vert.c
    fteng.cpp
    ftref.c
    gdiPlusFlat2.cpp
    gdidll.rc
    hash_list.cpp
    hook.cpp
    hookCounter.cpp
    misc.cpp
    override.cpp
    ownedcs.cpp
    settings.cpp
)

add_library(gdipp_core SHARED ${GDIPP_CORE_SOURCES})
add_library(gdipp::core ALIAS gdipp_core)

target_compile_definitions(gdipp_core PRIVATE
    UNICODE
    _UNICODE
    GDIPP_CMAKE_BUILD
    GDIPP_EXPORTS
    _GDIPP_DLL
    _USRDLL
    USE_DETOURS
)

if(GDIPP_ENABLE_TRACE)
    target_compile_definitions(gdipp_core PRIVATE TRACE)
endif()

if(GDIPP_ENABLE_INFINALITY)
    target_compile_definitions(gdipp_core PRIVATE INFINALITY)
endif()

target_include_directories(gdipp_core PRIVATE
    ${DETOURS_INCLUDE_DIR}
)

if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    target_link_directories(gdipp_core PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
endif()

target_link_libraries(gdipp_core PRIVATE
    IniParser::IniParser
    Freetype::Freetype
    nlohmann_json::nlohmann_json
    ${DETOURS_LIBRARY}
    dwrite
    shlwapi
    usp10
    user32
    version
)

if(CMAKE_SIZEOF_VOID_P EQUAL 4)
    target_link_libraries(gdipp_core PRIVATE ${WOW64EXT_LIBRARY})
endif()

if(MSVC)
    target_compile_options(gdipp_core PRIVATE /utf-8 /W3 /Zc:threadSafeInit-)
endif()

set_target_properties(gdipp_core PROPERTIES
    OUTPUT_NAME ${GDIPP_CORE_OUTPUT_NAME}
)

if(GDIPP_BUILD_MACLOADER)
    add_executable(macloader WIN32
        run.cpp
        macloader.rc
    )

    target_compile_definitions(macloader PRIVATE
        UNICODE
        _UNICODE
        GDIPP_CMAKE_BUILD
        USE_DETOURS
        _GDIPP_EXE
        _CRT_SECURE_NO_DEPRECATE=1
        WINVER=0x0500
        _WIN32_WINNT=0x0500
        _WIN32_IE=0x0601
        WIN32_LEAN_AND_MEAN=1
    )

    target_include_directories(macloader PRIVATE
        ${DETOURS_INCLUDE_DIR}
    )

    if(CMAKE_SIZEOF_VOID_P EQUAL 8)
        target_link_directories(macloader PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
    endif()

    target_link_libraries(macloader PRIVATE
        ${DETOURS_LIBRARY}
        kernel32
        ole32
        shell32
        shlwapi
        user32
    )

    if(MSVC)
        target_compile_options(macloader PRIVATE /utf-8 /W3)
    endif()

    set_target_properties(macloader PROPERTIES
        OUTPUT_NAME ${GDIPP_MACLOADER_OUTPUT_NAME}
    )
endif()

install(TARGETS gdipp_core
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_BINDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

if(GDIPP_BUILD_MACLOADER)
    install(TARGETS macloader
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    )
endif()