cmake_minimum_required(VERSION 3.16)

if(POLICY CMP0091)
    cmake_policy(SET CMP0091 NEW)
endif()

project(IniParser LANGUAGES CXX)

include(CMakePackageConfigHelpers)
include(GNUInstallDirs)

add_library(IniParser STATIC
    IniParser/ParseHelper.cpp
    IniParser/ParseIni.cpp
    IniParser/strtod.cpp
)

target_include_directories(IniParser
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/IniParser>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

target_compile_definitions(IniParser
    PUBLIC
        UNICODE
        _UNICODE
)

if(MSVC)
    set_property(TARGET IniParser PROPERTY MSVC_RUNTIME_LIBRARY "MultiThreadedDLL$<$<CONFIG:Debug>:Debug>")
    target_compile_options(IniParser PRIVATE /W3)
else()
    target_compile_options(IniParser PRIVATE -Wall -Wextra -Wno-unused-parameter)
endif()

target_compile_features(IniParser PUBLIC cxx_std_11)

set_target_properties(IniParser PROPERTIES
    OUTPUT_NAME IniParser
    EXPORT_NAME IniParser
)

write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/IniParserConfigVersion.cmake"
    VERSION 1.0.0
    COMPATIBILITY SameMajorVersion
)

install(TARGETS IniParser
    EXPORT IniParserTargets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

install(EXPORT IniParserTargets
    NAMESPACE IniParser::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/IniParser
)

install(FILES
    IniParser/ParseHelper.h
    IniParser/ParseIni.h
    IniParser/strtod.h
    IniParser/utf8conv.h
    IniParser/utf8except.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/IniParser
)

install(FILES
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/IniParserConfig.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/IniParserConfigVersion.cmake"
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/IniParser
)


