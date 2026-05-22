cmake_minimum_required(VERSION 3.16)

if(POLICY CMP0091)
    cmake_policy(SET CMP0091 NEW)
endif()

project(IniParser LANGUAGES CXX)

include(GNUInstallDirs)

add_library(IniParser STATIC
    IniParser/ParseHelper.cpp
    IniParser/ParseIni.cpp
    IniParser/strtod.cpp
)

target_include_directories(IniParser
    PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/IniParser>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/IniParser>
)

target_compile_definitions(IniParser
    PUBLIC
        UNICODE
        _UNICODE
)

if(MSVC)
    set_property(TARGET IniParser PROPERTY MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
    target_compile_options(IniParser PRIVATE /W3)
else()
    target_compile_options(IniParser PRIVATE -Wall -Wextra -Wno-unused-parameter)
endif()

target_compile_features(IniParser PUBLIC cxx_std_11)

set_target_properties(IniParser PROPERTIES
    OUTPUT_NAME IniParser
)

install(TARGETS IniParser
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

install(FILES
    IniParser/ParseHelper.h
    IniParser/ParseIni.h
    IniParser/strtod.h
    IniParser/utf8conv.h
    IniParser/utf8except.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/IniParser
)
