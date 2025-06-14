cmake_minimum_required(VERSION 3.15)
project(SimplyCoreAudio LANGUAGES Swift)

include(FetchContent)
FetchContent_Declare(
    swift-atomics
    GIT_REPOSITORY https://github.com/apple/swift-atomics.git
    GIT_TAG        1.3.0
)
FetchContent_MakeAvailable(swift-atomics)

file(GLOB_RECURSE SIMPLYCOREAUDIO_SOURCES
    "Sources/SimplyCoreAudio/*.swift"
)

add_library(SimplyCoreAudio STATIC ${SIMPLYCOREAUDIO_SOURCES})

# Link against the Atomics package
# CoreAudio and related frameworks are expected to be provided by the host system
# when building on macOS.
target_link_libraries(SimplyCoreAudio
    PUBLIC
        Atomics
)

# Tests
file(GLOB_RECURSE SIMPLYCOREAUDIO_TEST_SOURCES
    "Tests/SimplyCoreAudioTests/*.swift"
)

if(SIMPLYCOREAUDIO_TEST_SOURCES)
    add_executable(SimplyCoreAudioTests ${SIMPLYCOREAUDIO_TEST_SOURCES})
    target_link_libraries(SimplyCoreAudioTests
        PRIVATE
            SimplyCoreAudio
    )
    enable_testing()
    add_test(NAME SimplyCoreAudioSuite COMMAND SimplyCoreAudioTests)
endif()

