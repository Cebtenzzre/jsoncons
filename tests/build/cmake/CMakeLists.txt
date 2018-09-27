#
# jsoncons tests CMake file
#

cmake_minimum_required (VERSION 3.4)
# load global config
include (../../../build/cmake/config.cmake)

################
# ARM SETTINGS #
################

OPTION(CROSS_COMPILE_ARM "cross compile for ARM targets" OFF)

# Note: to compile on ARM (or cross compile), you may need to add the following:
# -DTARGET_ARCH="armv8-a -mfpu=neon -mfloat-abi=softfp -target arm-linux-gnueabi"
set(TARGET_ARCH "native" CACHE STRING "Target architecture arguments")
set(ARM_ARCH_DIRECTORY "arm-linux-gnueabi" CACHE STRING "ARM arch header dir")
set(ARM_GCC_VER "4.7.3" CACHE STRING "ARM GCC header dir")

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID MATCHES "GNU" OR CMAKE_CXX_COMPILER_ID MATCHES "Intel")
    if (CROSS_COMPILE_ARM)
        # We're cross-compiling with clang++ on Travis, this is all pretty specific and just for testing
        set(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS)
        set(CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS)
        set(CMAKE_THREAD_LIBS_INIT)

        include_directories(/usr/${ARM_ARCH_DIRECTORY}/include/c++/${ARM_GCC_VER}/${ARM_ARCH_DIRECTORY}/)
        include_directories(/usr/${ARM_ARCH_DIRECTORY}/include/c++/${ARM_GCC_VER}/)
        include_directories(/usr/${ARM_ARCH_DIRECTORY}/include/)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=${TARGET_ARCH} -Wunused-parameter -Wextra -Wreorder -std=c++11")
    elseif(NOT WIN32)
        CHECK_CXX_COMPILER_FLAG("-std=c++11" HAS_CPP11_FLAG)
        if (ENABLE_XTL_COMPLEX)
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -fPIC -march=${TARGET_ARCH} -Wunused-parameter -Wextra -Wreorder -std=c++14")
        elseif (HAS_CPP11_FLAG)
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -fPIC -march=${TARGET_ARCH} -Wunused-parameter -Wextra -Wreorder -std=c++11")
        else()
            message(FATAL_ERROR "Unsupported compiler -- xsimd requires C++11 support!")
        endif()
    endif()
endif()

project (Tests CXX)
set(JSONCONS_PROJECT_DIR ${PROJECT_SOURCE_DIR}/../../..)
set(JSONCONS_TESTS_SOURCE_DIR ${JSONCONS_PROJECT_DIR}/tests/src)
set(JSONCONS_TESTS_DIR ${JSONCONS_PROJECT_DIR}/tests)
set(JSONCONS_INCLUDE_DIR ${JSONCONS_PROJECT_DIR}/include)
set(JSONCONS_THIRD_PARTY_INCLUDE_DIR ${JSONCONS_PROJECT_DIR}/third_party)

set(CTEST_OUTPUT_ON_FAILURE ON)

# load per-platform configuration
include (${JSONCONS_PROJECT_DIR}/build/cmake/${CMAKE_SYSTEM_NAME}.cmake)

set(CATCH_INCLUDE_DIR ${JSONCONS_THIRD_PARTY_INCLUDE_DIR}/catch)
add_library(Catch INTERFACE)
target_include_directories (Catch INTERFACE ${CATCH_INCLUDE_DIR})

file(GLOB_RECURSE JSONCONS_TESTS_SOURCES ${JSONCONS_TESTS_SOURCE_DIR}/*.cpp)
add_executable(jsoncons_tests ${JSONCONS_TESTS_SOURCES})
target_compile_definitions (jsoncons_tests PUBLIC)
target_compile_options(jsoncons_tests PUBLIC
   "$<$<CXX_COMPILER_ID:MSVC>:/EHsc;$<$<CONFIG:Release>:/Od>>"
)
target_include_directories (jsoncons_tests PUBLIC ${JSONCONS_INCLUDE_DIR}
                                           PUBLIC ${JSONCONS_THIRD_PARTY_INCLUDE_DIR})

target_link_libraries(jsoncons_tests Catch)

if (CXXSTD)
set_target_properties (jsoncons_tests PROPERTIES
   CXX_STANDARD ${CXXSTD}
   CXX_STANDARD_REQUIRED ON
   CXX_EXTENSIONS OFF
 )
else()
set_target_properties (jsoncons_tests PROPERTIES
   CXX_STANDARD 11
   CXX_STANDARD_REQUIRED ON
   CXX_EXTENSIONS OFF
 )
endif()

if (${CMAKE_SYSTEM_NAME} STREQUAL "Linux" AND ${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang")
  # special link option on Linux because llvm stl rely on GNU stl
  target_link_libraries (jsoncons_tests -Wl,-lstdc++ Catch)
endif()

enable_testing()
add_test(NAME jsoncons_tests WORKING_DIRECTORY ${JSONCONS_TESTS_DIR} COMMAND ${CMAKE_CURRENT_BINARY_DIR}/jsoncons_tests)

set(JSONCONS_TARGET jsoncons_tests)
# add_executable(${JSONCONS_TARGET} EXCLUDE_FROM_ALL ${XSIMD_TESTS} ${XSIMD_HEADERS})
# target_link_libraries(${JSONCONS_TARGET} xsimd ${GTEST_BOTH_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT})

if (CROSS_COMPILE_ARM)
    add_custom_target(xtest COMMAND qemu-arm -L /usr/arm-linux-gnueabi/ jsoncons_tests DEPENDS ${JSONCONS_TARGET})
else()
    add_custom_target(xtest COMMAND jsoncons_tests DEPENDS ${JSONCONS_TARGET})
endif()

