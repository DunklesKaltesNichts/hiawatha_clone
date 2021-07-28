#
# CMake build system design considerations:
#
# - Include directories:
#   + Do not define include directories globally using the include_directories
#     command but rather at the target level using the
#     target_include_directories command. That way, it is easier to guarantee
#     that targets are built using the proper list of include directories.
#   + Use the PUBLIC and PRIVATE keywords to specifiy the scope of include
#     directories. That way, a target linking to a library (using the
#     target_link_librairies command) inherits from the library PUBLIC include
#     directories and not from the PRIVATE ones.
# - MBEDTLS_TARGET_PREFIX: CMake targets are designed to be alterable by calling
#   CMake in order to avoid target name clashes, via the use of
#   MBEDTLS_TARGET_PREFIX. The value of this variable is prefixed to the
#   mbedtls, mbedx509, mbedcrypto and apidoc targets.
#

# We specify a minimum requirement of 3.10.2, but for now use 3.5.1 here
# until our infrastructure catches up.
cmake_minimum_required(VERSION 3.5.1)

include(CMakePackageConfigHelpers)

# https://cmake.org/cmake/help/latest/policy/CMP0011.html
# Setting this policy is required in CMake >= 3.18.0, otherwise a warning is generated. The OLD
# policy setting is deprecated, and will be removed in future versions.
cmake_policy(SET CMP0011 NEW)
# https://cmake.org/cmake/help/latest/policy/CMP0012.html
# Setting the CMP0012 policy to NEW is required for FindPython3 to work with CMake 3.18.2
# (there is a bug in this particular version), otherwise, setting the CMP0012 policy is required
# for CMake versions >= 3.18.3 otherwise a deprecated warning is generated. The OLD policy setting
# is deprecated and will be removed in future versions.
cmake_policy(SET CMP0012 NEW)

if(TEST_CPP)
    project("mbed TLS" C CXX)
else()
    project("mbed TLS" C)
endif()

# Set the project root directory.
set(MBEDTLS_DIR ${CMAKE_CURRENT_SOURCE_DIR})

option(UNSAFE_BUILD "Allow unsafe builds. These builds ARE NOT SECURE." OFF)
option(MBEDTLS_FATAL_WARNINGS "Compiler warnings treated as errors" ON)

string(REGEX MATCH "Clang" CMAKE_COMPILER_IS_CLANG "${CMAKE_C_COMPILER_ID}")
string(REGEX MATCH "GNU" CMAKE_COMPILER_IS_GNU "${CMAKE_C_COMPILER_ID}")
string(REGEX MATCH "IAR" CMAKE_COMPILER_IS_IAR "${CMAKE_C_COMPILER_ID}")
string(REGEX MATCH "MSVC" CMAKE_COMPILER_IS_MSVC "${CMAKE_C_COMPILER_ID}")

# Warning string - created as a list for compatibility with CMake 2.8
set(CTR_DRBG_128_BIT_KEY_WARN_L1 "****  WARNING!  MBEDTLS_CTR_DRBG_USE_128_BIT_KEY defined!\n")
set(CTR_DRBG_128_BIT_KEY_WARN_L2 "****  Using 128-bit keys for CTR_DRBG limits the security of generated\n")
set(CTR_DRBG_128_BIT_KEY_WARN_L3 "****  keys and operations that use random values generated to 128-bit security\n")

set(CTR_DRBG_128_BIT_KEY_WARNING "${WARNING_BORDER}"
                         "${CTR_DRBG_128_BIT_KEY_WARN_L1}"
                         "${CTR_DRBG_128_BIT_KEY_WARN_L2}"
                         "${CTR_DRBG_128_BIT_KEY_WARN_L3}"
                         "${WARNING_BORDER}")

# Create a symbolic link from ${base_name} in the binary directory
# to the corresponding path in the source directory.
function(link_to_source base_name)
    # Get OS dependent path to use in `execute_process`
    if (CMAKE_HOST_WIN32)
        #mklink is an internal command of cmd.exe it can only work with \
        string(REPLACE "/" "\\" link "${CMAKE_CURRENT_BINARY_DIR}/${base_name}")
        string(REPLACE "/" "\\" target "${CMAKE_CURRENT_SOURCE_DIR}/${base_name}")
    else()
        set(link "${CMAKE_CURRENT_BINARY_DIR}/${base_name}")
        set(target "${CMAKE_CURRENT_SOURCE_DIR}/${base_name}")
    endif()

    if (NOT EXISTS ${link})
        if (CMAKE_HOST_UNIX)
            set(command ln -s ${target} ${link})
        else()
            if (IS_DIRECTORY ${target})
                set(command cmd.exe /c mklink /j ${link} ${target})
            else()
                set(command cmd.exe /c mklink /h ${link} ${target})
            endif()
        endif()

        execute_process(COMMAND ${command}
            RESULT_VARIABLE result
            ERROR_VARIABLE output)

        if (NOT ${result} EQUAL 0)
            message(FATAL_ERROR "Could not create symbolic link for: ${target} --> ${output}")
        endif()
    endif()
endfunction(link_to_source)

string(REGEX MATCH "Clang" CMAKE_COMPILER_IS_CLANG "${CMAKE_C_COMPILER_ID}")

include(CheckCCompilerFlag)

if(CMAKE_COMPILER_IS_GNU)
    # some warnings we want are not available with old GCC versions
    # note: starting with CMake 2.8 we could use CMAKE_C_COMPILER_VERSION
    execute_process(COMMAND ${CMAKE_C_COMPILER} -dumpversion
                    OUTPUT_VARIABLE GCC_VERSION)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Wwrite-strings")
    if (GCC_VERSION VERSION_GREATER 3.0 OR GCC_VERSION VERSION_EQUAL 3.0)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wformat=2 -Wno-format-nonliteral")
    endif()
    if (GCC_VERSION VERSION_GREATER 4.3 OR GCC_VERSION VERSION_EQUAL 4.3)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wvla")
    endif()
    if (GCC_VERSION VERSION_GREATER 4.5 OR GCC_VERSION VERSION_EQUAL 4.5)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wlogical-op")
    endif()
    if (GCC_VERSION VERSION_GREATER 4.8 OR GCC_VERSION VERSION_EQUAL 4.8)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wshadow")
    endif()
    if (GCC_VERSION VERSION_GREATER 5.0)
        CHECK_C_COMPILER_FLAG("-Wformat-signedness" C_COMPILER_SUPPORTS_WFORMAT_SIGNEDNESS)
        if(C_COMPILER_SUPPORTS_WFORMAT_SIGNEDNESS)
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wformat-signedness")
        endif()
    endif()
    if (GCC_VERSION VERSION_GREATER 7.0 OR GCC_VERSION VERSION_EQUAL 7.0)
      set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wformat-overflow=2 -Wformat-truncation")
    endif()
    set(CMAKE_C_FLAGS_RELEASE     "-O2")
    set(CMAKE_C_FLAGS_DEBUG       "-O0 -g3")
    set(CMAKE_C_FLAGS_COVERAGE    "-O0 -g3 --coverage")
    set(CMAKE_C_FLAGS_ASAN        "-fsanitize=address -fno-common -fsanitize=undefined -fno-sanitize-recover=all -O3")
    set(CMAKE_C_FLAGS_ASANDBG     "-fsanitize=address -fno-common -fsanitize=undefined -fno-sanitize-recover=all -O1 -g3 -fno-omit-frame-pointer -fno-optimize-sibling-calls")
    set(CMAKE_C_FLAGS_CHECK       "-Os")
    set(CMAKE_C_FLAGS_CHECKFULL   "${CMAKE_C_FLAGS_CHECK} -Wcast-qual")
endif(CMAKE_COMPILER_IS_GNU)

if(CMAKE_COMPILER_IS_CLANG)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Wwrite-strings -Wpointer-arith -Wimplicit-fallthrough -Wshadow -Wvla -Wformat=2 -Wno-format-nonliteral")
    set(CMAKE_C_FLAGS_RELEASE     "-O2")
    set(CMAKE_C_FLAGS_DEBUG       "-O0 -g3")
    set(CMAKE_C_FLAGS_COVERAGE    "-O0 -g3 --coverage")
    set(CMAKE_C_FLAGS_ASAN        "-fsanitize=address -fno-common -fsanitize=undefined -fno-sanitize-recover=all -O3")
    set(CMAKE_C_FLAGS_ASANDBG     "-fsanitize=address -fno-common -fsanitize=undefined -fno-sanitize-recover=all -O1 -g3 -fno-omit-frame-pointer -fno-optimize-sibling-calls")
    set(CMAKE_C_FLAGS_MEMSAN      "-fsanitize=memory -O3")
    set(CMAKE_C_FLAGS_MEMSANDBG   "-fsanitize=memory -O1 -g3 -fno-omit-frame-pointer -fno-optimize-sibling-calls -fsanitize-memory-track-origins=2")
    set(CMAKE_C_FLAGS_CHECK       "-Os")
endif(CMAKE_COMPILER_IS_CLANG)

if(CMAKE_COMPILER_IS_IAR)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} --warn_about_c_style_casts --warnings_are_errors -Ohz")
endif(CMAKE_COMPILER_IS_IAR)

if(CMAKE_COMPILER_IS_MSVC)
    # Strictest warnings
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /W3")
endif(CMAKE_COMPILER_IS_MSVC)

if(MBEDTLS_FATAL_WARNINGS)
    if(CMAKE_COMPILER_IS_MSVC)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /WX")
    endif(CMAKE_COMPILER_IS_MSVC)

    if(CMAKE_COMPILER_IS_CLANG OR CMAKE_COMPILER_IS_GNU)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Werror")
        if(UNSAFE_BUILD)
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-error=cpp")
            set(CMAKE_C_FLAGS_ASAN "${CMAKE_C_FLAGS_ASAN} -Wno-error=cpp")
            set(CMAKE_C_FLAGS_ASANDBG "${CMAKE_C_FLAGS_ASANDBG} -Wno-error=cpp")
        endif(UNSAFE_BUILD)
    endif(CMAKE_COMPILER_IS_CLANG OR CMAKE_COMPILER_IS_GNU)
endif(MBEDTLS_FATAL_WARNINGS)

if(CMAKE_BUILD_TYPE STREQUAL "Coverage")
    if(CMAKE_COMPILER_IS_GNU OR CMAKE_COMPILER_IS_CLANG)
        set(CMAKE_SHARED_LINKER_FLAGS "--coverage")
    endif(CMAKE_COMPILER_IS_GNU OR CMAKE_COMPILER_IS_CLANG)
endif(CMAKE_BUILD_TYPE STREQUAL "Coverage")

if(LIB_INSTALL_DIR)
else()
    set(LIB_INSTALL_DIR lib)
endif()

add_subdirectory(include)

add_subdirectory(library)

configure_package_config_file(
    "cmake/MbedTLSConfig.cmake.in"
    "cmake/MbedTLSConfig.cmake"
        INSTALL_DESTINATION "cmake")

write_basic_package_version_file(
    "cmake/MbedTLSConfigVersion.cmake"
        COMPATIBILITY SameMajorVersion
        VERSION 3.0.0)

install(
    FILES "${CMAKE_CURRENT_BINARY_DIR}/cmake/MbedTLSConfig.cmake"
          "${CMAKE_CURRENT_BINARY_DIR}/cmake/MbedTLSConfigVersion.cmake"
    DESTINATION "cmake")

export(
    EXPORT MbedTLSTargets
    NAMESPACE MbedTLS::
    FILE "cmake/MbedTLSTargets.cmake")

install(
    EXPORT MbedTLSTargets
    NAMESPACE MbedTLS::
    DESTINATION "cmake"
    FILE "MbedTLSTargets.cmake")

if(CMAKE_VERSION VERSION_GREATER 3.14)
    # Do not export the package by default
    cmake_policy(SET CMP0090 NEW)

    # Make this package visible to the system
    export(PACKAGE MbedTLS)
endif()
