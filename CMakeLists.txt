cmake_minimum_required(VERSION 3.26)
project(SimplyCoreAudio LANGUAGES Swift)

# Set some policies for better Swift support
if(POLICY CMP0157)
    cmake_policy(SET CMP0157 NEW)  # Swift compilation mode support
endif()

# Require Atomics target to be provided by the consuming project
if(NOT TARGET Atomics)
    message(FATAL_ERROR "SimplyCoreAudio requires the 'Atomics' target from swift-atomics. Please ensure swift-atomics is available before including SimplyCoreAudio.")
endif()

# Collect all Swift source files
file(GLOB_RECURSE SIMPLYCOREAUDIO_SOURCES
    "Sources/SimplyCoreAudio/*.swift"
)

# Create the main library
add_library(SimplyCoreAudio STATIC ${SIMPLYCOREAUDIO_SOURCES})

# Set Swift-specific properties
set_target_properties(SimplyCoreAudio PROPERTIES
    Swift_MODULE_NAME SimplyCoreAudio
    Swift_MODULE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/swift
)

# Link against the Atomics package and system frameworks
target_link_libraries(SimplyCoreAudio
    PRIVATE
        Atomics
)

# Add the SimplyCoreAudioC target for C compatibility layer
file(GLOB_RECURSE SIMPLYCOREAUDIOC_SOURCES
    "Sources/SimplyCoreAudioC/*.c"
    "Sources/SimplyCoreAudioC/*.h"
)

if(SIMPLYCOREAUDIOC_SOURCES)
    add_library(SimplyCoreAudioC STATIC ${SIMPLYCOREAUDIOC_SOURCES})
    
    # Set include directories for the C library using generator expressions
    target_include_directories(SimplyCoreAudioC
        PUBLIC
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/Sources/SimplyCoreAudioC>
            $<INSTALL_INTERFACE:include>
    )
    
    # Link SimplyCoreAudioC to SimplyCoreAudio
    target_link_libraries(SimplyCoreAudio
        PUBLIC
            SimplyCoreAudioC
    )
endif()

# Platform-specific linking (macOS frameworks)
if(APPLE)
    find_library(COREAUDIO_FRAMEWORK CoreAudio)
    find_library(AUDIOTOOLBOX_FRAMEWORK AudioToolbox)
    find_library(FOUNDATION_FRAMEWORK Foundation)
    
    if(COREAUDIO_FRAMEWORK AND AUDIOTOOLBOX_FRAMEWORK AND FOUNDATION_FRAMEWORK)
        target_link_libraries(SimplyCoreAudio
            PUBLIC
                ${COREAUDIO_FRAMEWORK}
                ${AUDIOTOOLBOX_FRAMEWORK}
                ${FOUNDATION_FRAMEWORK}
        )
    else()
        message(WARNING "Some required macOS frameworks were not found")
    endif()
endif()

# Export include directories
target_include_directories(SimplyCoreAudio
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/Sources>
        $<INSTALL_INTERFACE:include>
)

# Tests
file(GLOB_RECURSE SIMPLYCOREAUDIO_TEST_SOURCES
    "Tests/SimplyCoreAudioTests/*.swift"
)

if(SIMPLYCOREAUDIO_TEST_SOURCES)
    add_executable(SimplyCoreAudioTests ${SIMPLYCOREAUDIO_TEST_SOURCES})
    
    # Set Swift properties for tests
    set_target_properties(SimplyCoreAudioTests PROPERTIES
        Swift_MODULE_NAME SimplyCoreAudioTests
    )
    
    target_link_libraries(SimplyCoreAudioTests
        PRIVATE
            SimplyCoreAudio
    )
    
    # Enable testing
    enable_testing()
    add_test(NAME SimplyCoreAudioSuite COMMAND SimplyCoreAudioTests)
endif()

# Installation rules
install(TARGETS SimplyCoreAudio
    EXPORT SimplyCoreAudioTargets
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    RUNTIME DESTINATION bin
    INCLUDES DESTINATION include
)

if(TARGET SimplyCoreAudioC)
    install(TARGETS SimplyCoreAudioC
        EXPORT SimplyCoreAudioTargets
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib
        RUNTIME DESTINATION bin
        INCLUDES DESTINATION include
    )
    
    install(FILES Sources/SimplyCoreAudioC/SimplyCoreAudio.h
        DESTINATION include
    )
endif()

# Export the targets
install(EXPORT SimplyCoreAudioTargets
    FILE SimplyCoreAudioTargets.cmake
    NAMESPACE SimplyCoreAudio::
    DESTINATION lib/cmake/SimplyCoreAudio
)

# Create a config file
include(CMakePackageConfigHelpers)

# Create the config file
configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/SimplyCoreAudioConfig.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/SimplyCoreAudioConfig.cmake"
    INSTALL_DESTINATION lib/cmake/SimplyCoreAudio
)

write_basic_package_version_file(
    "SimplyCoreAudioConfigVersion.cmake"
    VERSION 1.0.0
    COMPATIBILITY SameMajorVersion
)

install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/SimplyCoreAudioConfig.cmake"
    "SimplyCoreAudioConfigVersion.cmake"
    DESTINATION lib/cmake/SimplyCoreAudio
)
