cmake_minimum_required(VERSION 3.26)
project(SimplyCoreAudio LANGUAGES Swift)

include(FetchContent)

# Set some policies for better Swift support
if(POLICY CMP0157)
    cmake_policy(SET CMP0157 NEW)  # Swift compilation mode support
endif()

# Only fetch swift-atomics if Atomics target doesn't already exist
if(NOT TARGET Atomics)
    FetchContent_Declare(
        swift-atomics
        GIT_REPOSITORY https://github.com/apple/swift-atomics.git
        GIT_TAG        1.3.0
    )
    
    # Make the content available but don't add to ALL target by default
    FetchContent_MakeAvailable(swift-atomics)
    
    # Verify that the Atomics target was created
    if(NOT TARGET Atomics)
        message(FATAL_ERROR "Failed to create Atomics target from swift-atomics")
    endif()
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
    PUBLIC
        Atomics
)

# Add the SimplyCoreAudioC target for C compatibility layer
file(GLOB_RECURSE SIMPLYCOREAUDIOC_SOURCES
    "Sources/SimplyCoreAudioC/*.c"
    "Sources/SimplyCoreAudioC/*.h"
)

if(SIMPLYCOREAUDIOC_SOURCES)
    add_library(SimplyCoreAudioC STATIC ${SIMPLYCOREAUDIOC_SOURCES})
    
    # Set include directories for the C library
    target_include_directories(SimplyCoreAudioC
        PUBLIC
            Sources/SimplyCoreAudioC
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

# Optional: Installation rules
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
write_basic_package_version_file(
    "SimplyCoreAudioConfigVersion.cmake"
    VERSION 1.0.0
    COMPATIBILITY SameMajorVersion
)

install(FILES
    "SimplyCoreAudioConfigVersion.cmake"
    DESTINATION lib/cmake/SimplyCoreAudio
)
