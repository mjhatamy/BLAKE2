cmake_minimum_required(VERSION 3.10)  # Minimum CMake version
project(blake2 VERSION 1.0 LANGUAGES C)

# Enable SSE2 and NEON flags based on platform
include(CheckCCompilerFlag)

set(BLAKE2s_SOURCES )

message(STATUS "${CMAKE_SYSTEM_PROCESSOR} Processor detected")
# Set compiler flags for optimizations and SSE2/NEON
if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64" OR CMAKE_SYSTEM_PROCESSOR MATCHES "AMD64" OR CMAKE_SYSTEM_PROCESSOR MATCHES "i686")
    if(MSVC)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /arch:SSE2")  # For SSE2 on x86 with MSVC
    else()
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -msse2 -Wall -Wextra -std=c89 -pedantic -Wno-long-long")  # For SSE2 on x86 with GCC/MinGW and additional flags
    endif()

    set(SOURCE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/sse")
    list(APPEND BLAKE2s_SOURCES ${SOURCE_PATH}/blake2s.c ${SOURCE_PATH}/blake2.h ${SOURCE_PATH}/blake2s-load-sse41.h)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "arm")
    if(MSVC)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mfpu=neon")  # For NEON on ARM
    else()
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mfpu=neon -Wall -Wextra -std=c89 -pedantic -Wno-long-long")  # For NEON on ARM and additional flags
    endif()

    set(SOURCE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/neon")
    list(APPEND BLAKE2s_SOURCES ${SOURCE_PATH}/blake2s.c ${SOURCE_PATH}/blake2s-load-sse2.h)
endif()

# Add optimization flags
if(MSVC)
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /O2")
    set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} /Od")
else()
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -O3")
    set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -O0")
endif()

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/testvectors)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/testvectors)

# Create a static library
add_library(${PROJECT_NAME}s STATIC ${BLAKE2s_SOURCES})

# Set the public header files
set_target_properties(${PROJECT_NAME}s PROPERTIES PUBLIC_HEADER "${SOURCE_PATH}/blake2.h")

# Installation rules
include(GNUInstallDirs)
install(TARGETS ${PROJECT_NAME}s
        EXPORT ${PROJECT_NAME}sTargets
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}
)

# Generate and install export file
install(EXPORT ${PROJECT_NAME}sTargets
        FILE ${PROJECT_NAME}sTargets.cmake
        NAMESPACE ${PROJECT_NAME}::
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)

# Generate the config file
include(CMakePackageConfigHelpers)
configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/Config.cmake.in
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
        INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)

# Generate the version file
write_basic_package_version_file(
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY SameMajorVersion
)

# Install the config and version files
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