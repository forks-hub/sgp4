cmake_minimum_required(VERSION 3.14)

project(SGP4
    VERSION 3.0.0
    DESCRIPTION "SGP4 satellite orbit propagation library"
    LANGUAGES CXX)

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

# ---------------------------------------------------------------------------
# Everything in this block sets project-global state that would corrupt the
# configuration of any project consuming libsgp4 via add_subdirectory() or
# FetchContent: the install prefix, the C++ standard, the compile-commands
# setting and the build type. Only apply it when this is the top-level project.
# As a subproject the consumer already decided all of these, and the library
# itself only needs C++17 on its own targets (set in libsgp4/CMakeLists.txt).
# ---------------------------------------------------------------------------
if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
    if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
        set(CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}/install" CACHE PATH "Install prefix" FORCE)
    endif()

    set(CMAKE_CXX_STANDARD 17)
    set(CMAKE_CXX_STANDARD_REQUIRED ON)
    set(CMAKE_CXX_EXTENSIONS OFF)
    set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

    if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
        set(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING "Build type" FORCE)
    endif()
endif()

# Examples and tests default ON only when libsgp4 is built on its own. As a
# subproject they default OFF: a consumer that only wants the propagation
# library should not silently gain four executables, a googletest download and
# a `format` target. Both options can still be forced ON explicitly.
if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
    set(_sgp4_default_build_examples ON)
    set(_sgp4_default_build_tests ON)
else()
    set(_sgp4_default_build_examples OFF)
    set(_sgp4_default_build_tests OFF)
endif()

option(BUILD_EXAMPLES "Build example programs (sattrack, runtest, passpredict)" ${_sgp4_default_build_examples})
option(BUILD_TESTS "Build unit tests" ${_sgp4_default_build_tests})

if(BUILD_TESTS)
    include(FetchContent)
    set(INSTALL_GTEST OFF CACHE BOOL "Enable installation of googletest" FORCE)
    # GoogleTest rewrites /MD to /MT in its own flags unless this is set, which would
    # link the static CRT into the tests and clash with the project's shared CRT.
    set(gtest_force_shared_crt ON CACHE BOOL "Build GoogleTest against the shared CRT" FORCE)
    FetchContent_Declare(
        googletest
        GIT_REPOSITORY https://github.com/google/googletest.git
        GIT_TAG v1.14.0)
    FetchContent_MakeAvailable(googletest)

    enable_testing()
    include(GoogleTest)
endif()

configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/SGP4-VER.TLE"
    "${CMAKE_CURRENT_BINARY_DIR}/SGP4-VER.TLE"
    COPYONLY)

add_subdirectory(libsgp4)

if(BUILD_EXAMPLES)
    add_subdirectory(sattrack)
    add_subdirectory(runtest)
    add_subdirectory(passpredict)
    add_subdirectory(csvprop)
endif()

if(BUILD_TESTS)
    add_subdirectory(tests)
endif()

# Developer conveniences. Only defined when libsgp4 is the top-level project:
# as a subproject these would add `format` / `format-check` targets into the
# consumer's build, where the names may already be taken, and they exist purely
# for working on libsgp4 itself.
if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
    find_program(CLANG_FORMAT_EXECUTABLE NAMES clang-format clang-format-21 clang-format-20 clang-format-19 clang-format-18)

    if(CLANG_FORMAT_EXECUTABLE)
        file(GLOB_RECURSE SGP4_FORMAT_SOURCES
            "${PROJECT_SOURCE_DIR}/libsgp4/*.h"
            "${PROJECT_SOURCE_DIR}/libsgp4/*.cpp"
            "${PROJECT_SOURCE_DIR}/tests/*.cpp"
            "${PROJECT_SOURCE_DIR}/sattrack/*.cpp"
            "${PROJECT_SOURCE_DIR}/runtest/*.cpp"
            "${PROJECT_SOURCE_DIR}/passpredict/*.cpp"
            "${PROJECT_SOURCE_DIR}/csvprop/*.cpp")

        add_custom_target(format
            COMMAND ${CLANG_FORMAT_EXECUTABLE} -i ${SGP4_FORMAT_SOURCES}
            WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
            COMMENT "Applying clang-format to all source files"
            USES_TERMINAL)

        add_custom_target(format-check
            COMMAND ${CLANG_FORMAT_EXECUTABLE} -i --dry-run --Werror ${SGP4_FORMAT_SOURCES}
            WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
            COMMENT "Checking clang-format compliance"
            USES_TERMINAL)
    endif()
endif()
