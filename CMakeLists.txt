cmake_minimum_required(VERSION 3.10)
project(blake2 VERSION 1.0 LANGUAGES C)

# Include necessary modules
include(CheckCCompilerFlag)
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

# Function to check and add compiler flags
function(add_compiler_flags target)
    if(MSVC)
        target_compile_options(${target} PRIVATE
                $<$<CONFIG:Release>:/O2>
                $<$<CONFIG:Debug>:/Od>
                /W4
        )
    else()
        target_compile_options(${target} PRIVATE
                $<$<CONFIG:Release>:-O3>
                $<$<CONFIG:Debug>:-O0>
                -Wall
                -Wextra
                -std=c89
                -pedantic
                -Wno-long-long
        )
    endif()
endfunction()

# Initialize source variables
set(BLAKE2s_SOURCES)
set(SOURCE_PATH "")

# Detect platform and set appropriate sources and flags
message(STATUS "Processor detected: ${CMAKE_SYSTEM_PROCESSOR}")

# Check for SSE2 support
if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|AMD64|i686)$")
    set(SOURCE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/sse")

    if(MSVC)
        # Check if SSE2 is supported
        check_c_compiler_flag("/arch:SSE2" COMPILER_SUPPORTS_SSE2)
        if(COMPILER_SUPPORTS_SSE2)
            add_compile_options("/arch:SSE2")
        else()
            message(FATAL_ERROR "SSE2 support is required but not available")
        endif()
    else()
        # Check if SSE2 is supported for GCC/Clang
        check_c_compiler_flag("-msse2" COMPILER_SUPPORTS_SSE2)
        if(COMPILER_SUPPORTS_SSE2)
            add_compile_options("-msse2")
        else()
            message(FATAL_ERROR "SSE2 support is required but not available")
        endif()
    endif()

    list(APPEND BLAKE2s_SOURCES
            ${SOURCE_PATH}/blake2s.c
            ${SOURCE_PATH}/blake2.h
            ${SOURCE_PATH}/blake2s-load-sse41.h
    )

    # Check for ARM NEON support
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^(arm|aarch64)")
    set(SOURCE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/neon")

    if(MSVC)
        check_c_compiler_flag("/arch:NEON" COMPILER_SUPPORTS_NEON)
        if(COMPILER_SUPPORTS_NEON)
            add_compile_options("/arch:NEON")
        else()
            message(FATAL_ERROR "NEON support is required but not available")
        endif()
    else()
        check_c_compiler_flag("-mfpu=neon" COMPILER_SUPPORTS_NEON)
        if(COMPILER_SUPPORTS_NEON)
            add_compile_options("-mfpu=neon")
        else()
            message(FATAL_ERROR "NEON support is required but not available")
        endif()
    endif()

    list(APPEND BLAKE2s_SOURCES
            ${SOURCE_PATH}/blake2s.c
            ${SOURCE_PATH}/blake2s-load-sse2.h
    )
else()
    message(FATAL_ERROR "Unsupported processor architecture: ${CMAKE_SYSTEM_PROCESSOR}")
endif()

# Verify that we have sources
if(NOT BLAKE2s_SOURCES)
    message(FATAL_ERROR "No source files were selected for the current platform")
endif()

# Add include directories
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/testvectors)

# Create the static library
add_library(${PROJECT_NAME}s STATIC ${BLAKE2s_SOURCES})
# Set include directories for the target
target_include_directories(${PROJECT_NAME}s
        PUBLIC
        $<BUILD_INTERFACE:${SOURCE_PATH}>
        $<INSTALL_INTERFACE:include>
)
# Add compiler flags
add_compiler_flags(${PROJECT_NAME}s)

# Set the public header
set_target_properties(${PROJECT_NAME}s PROPERTIES
        PUBLIC_HEADER "${SOURCE_PATH}/blake2.h"
)

# Make the target available for other projects in the build tree
add_library(blake2::blake2s ALIAS blake2s)

# Installation rules
install(TARGETS ${PROJECT_NAME}s
        EXPORT ${PROJECT_NAME}sTargets
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}
)

# Export targets
install(EXPORT ${PROJECT_NAME}sTargets
        FILE ${PROJECT_NAME}sTargets.cmake
        NAMESPACE ${PROJECT_NAME}::
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)

# Generate config file
configure_package_config_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/Config.cmake.in
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
        INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)

# Generate version file
write_basic_package_version_file(
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY SameMajorVersion
)

# Install config files
install(FILES
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)

# Export targets to build tree
export(EXPORT ${PROJECT_NAME}sTargets
        FILE "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}sTargets.cmake"
        NAMESPACE ${PROJECT_NAME}::
)