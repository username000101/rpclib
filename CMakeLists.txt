cmake_minimum_required(VERSION 3.5.1)
project(rpc VERSION 2.3.0)

set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake")

include(policies)
include(msvc_support)
include(coverage)
include(check_warning_flag)

#
# Options
#
option(RPCLIB_BUILD_TESTS 
  "Build unit RPCLIB_BUILD_TESTS." 
  OFF)
option(RPCLIB_GENERATE_COMPDB 
  "Generate compilation database. Useful for YCM." 
  OFF)
option(RPCLIB_BUILD_EXAMPLES 
  "Build examples." 
  OFF)
option(RPCLIB_ENABLE_LOGGING 
  "ALlow logging in the library for debug purposes."
  OFF)
option(RPCLIB_ENABLE_COVERAGE 
  "Generate coverage information" 
  OFF)
option(RPCLIB_MSVC_STATIC_RUNTIME 
  "MSVC only: build with /MT instead of /MD" 
  OFF)

#
# Other configuration values
#
set(RPCLIB_DEFAULT_PORT 8080 
  CACHE STRING "Default port used for running tests and examples")
set(RPCLIB_DEFAULT_BUFFER_SIZE "1024 << 10" 
  CACHE STRING "Default buffer size")
set(RPCLIB_CXX_STANDARD 11 CACHE STRING 
  "C++ version used to build rpclib (Currently: Only 11 and 14 supported)")

if(RPCLIB_GENERATE_COMPDB)
  set(CMAKE_EXPORT_COMPILE_COMMANDS "ON") # for YCM
  add_custom_command(PROJECT_NAME ${PROJECT_NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E
    copy ${CMAKE_BINARY_DIR}/compile_commands.json ${CMAKE_BINARY_DIR}/../compile_commands.json)
endif()

if(NOT ${RPCLIB_CXX_STANDARD} EQUAL 14 AND 
   NOT ${RPCLIB_CXX_STANDARD} EQUAL 11)
  message(fatal_error "Unsupported C++ standard: ${RPCLIB_CXX_STANDARD}")
endif()

set(CMAKE_CXX_STANDARD ${RPCLIB_CXX_STANDARD})

#
# Compile & install the library
#
if (WIN32)
  set(RPCLIB_OS_DEF "RPCLIB_WIN32")
elseif (LINUX)
  set(RPCLIB_OS_DEF "RPCLIB_LINUX")
elseif (APPLE)
  set(RPCLIB_OS_DEF "RPCLIB_MAC")
endif()

set(RPCLIB_DEPENDENCIES "${CMAKE_CURRENT_LIST_DIR}/dependencies")

configure_file(
  "${PROJECT_SOURCE_DIR}/include/rpc/version.h.in"
  "${PROJECT_SOURCE_DIR}/include/rpc/version.h")

configure_file(
  "${PROJECT_SOURCE_DIR}/include/rpc/config.h.in"
  "${PROJECT_SOURCE_DIR}/include/rpc/config.h")

file(GLOB_RECURSE RPCLIB_HEADERS
  include/rpc/*.h
  include/msgpack/*.hpp)
file(GLOB_RECURSE DEP_HEADERS
  ${RPCLIB_DEPENDENCIES}/include/*.h
  ${RPCLIB_DEP_LIBRARIDEPENDENCIES}/include/*.hpp)

set(DEP_SOURCES
  ${RPCLIB_DEPENDENCIES}/src/format.cc
  ${RPCLIB_DEPENDENCIES}/src/posix.cc)

add_library(${PROJECT_NAME}
  lib/rpc/dispatcher.cc
  lib/rpc/server.cc
  lib/rpc/client.cc
  lib/rpc/this_handler.cc
  lib/rpc/this_session.cc
  lib/rpc/this_server.cc
  lib/rpc/rpc_error.cc
  lib/rpc/detail/server_session.cc
  lib/rpc/detail/response.cc
  lib/rpc/detail/client_error.cc
  lib/rpc/nonstd/optional.cc
  ${DEP_SOURCES}
  ${DEP_HEADERS}
  ${RPCLIB_HEADERS})

# Perform steps and checks required for MSVC support
rpclib_msvc_support()

set(RPCLIB_BUILD_FLAGS "") # reset flags

if(RPCLIB_ENABLE_COVERAGE)
  enable_coverage(${PROJECT_NAME})
endif()

if ("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
  # clang is the compiler used for developing mainly, so
  # this is where I set the highest warning level
  list(APPEND RPCLIB_BUILD_FLAGS
    -Wall -pedantic -Weverything -Wno-c++98-compat
    -Wno-c++98-compat-pedantic -Wno-padded -Wno-missing-prototypes
    -Wno-undef)

    set(UNUSED_LAMBDA_CAPTURE_WARN_SUPPORTED)
    check_warning_flag("unused-lambda-capture" UNUSED_LAMBDA_CAPTURE_WARN_SUPPORTED)
    if(${UNUSED_LAMBDA_CAPTURE_WARN_SUPPORTED})
      list(APPEND RPCLIB_BUILD_FLAGS -Wno-unused-lambda-capture)
    endif()

    check_warning_flag("zero-as-null-pointer-constant" ZERO_AS_NULL_POINTER_CONSTANT_WARN_SUPPORTED)
    set(ZERO_AS_NULL_POINTER_CONSTANT_WARN_SUPPORTED)
    if(${ZERO_AS_NULL_POINTER_CONSTANT_WARN_SUPPORTED})
      list(APPEND RPCLIB_BUILD_FLAGS -Wno-zero-as-null-pointer-constant)
    endif()
endif()

if (RPCLIB_EXTRA_BUILD_FLAGS)
  list(APPEND RPCLIB_BUILD_FLAGS ${RPCLIB_EXTRA_BUILD_FLAGS})
endif()

target_compile_definitions(${PROJECT_NAME} 
  PRIVATE
    "${RPCLIB_COMPILE_DEFINITIONS}"
    "${RPCLIB_ARCH_DEF}"
    "ASIO_STANDALONE"
    "RPCLIB_ASIO=clmdep_asio"
    "RPCLIB_FMT=clmdep_fmt"
  PUBLIC
    "${RPCLIB_OS_DEF}"
    "RPCLIB_MSGPACK=clmdep_msgpack"
  )

if(RPCLIB_ENABLE_LOGGING)
  target_compile_definitions(${PROJECT_NAME} PRIVATE "RPCLIB_ENABLE_LOGGING")
endif()

if(RPCLIB_BUILD_FLAGS)
  target_compile_options(${PROJECT_NAME} PRIVATE ${RPCLIB_BUILD_FLAGS})
endif()

if(RPCLIB_COMPILE_DEFINITIONS)
  set_target_properties(${PROJECT_NAME}
    PROPERTIES
    COMPILE_DEFINITIONS "${RPCLIB_COMPILE_DEFINITIONS}")
endif()

target_link_libraries(${PROJECT_NAME} ${RPCLIB_DEP_LIBRARIES})
if (WIN32)
  target_link_libraries(${PROJECT_NAME} wsock32 ws2_32)
endif()
target_include_directories(
  ${PROJECT_NAME} PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>
  )
target_include_directories(
  ${PROJECT_NAME} SYSTEM
  PRIVATE ${RPCLIB_DEPENDENCIES}/include
  )

install(TARGETS ${PROJECT_NAME} DESTINATION lib EXPORT rpclibTargets)
install(DIRECTORY include/
  DESTINATION include
  FILES_MATCHING
  PATTERN "*.h"
  PATTERN "*.hpp"
  PATTERN "*.inl"
  PATTERN "*.in" EXCLUDE)

#
# Unit tests
#
if(RPCLIB_BUILD_TESTS)
  add_subdirectory(tests)
endif()

#
# Example programs
#
if(RPCLIB_BUILD_EXAMPLES)
  find_package(Threads)
  set(RPCLIB_ROOT_DIR "${PROJECT_SOURCE_DIR}")
  set(RPCLIB_PROJECT_NAME "${CMAKE_PROJECT_NAME}")
  set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake/localbuild")
  set(ENV{RPCLIB_DEFAULT_PORT} "${RPCLIB_DEFAULT_PORT}")
  add_subdirectory(examples/echo)
  add_subdirectory(examples/mandelbrot)
  add_subdirectory(examples/calculator)
endif()

#
# Cmake Package
#
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/cmake/rpclibConfigVersion.cmake"
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY AnyNewerVersion
  )

set(CONFIG_PACKAGE_LOCATION lib/cmake/rpclib)
set(INCLUDE_INSTALL_DIR include/ )

configure_package_config_file(cmake/rpclibConfig.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/cmake/rpclibConfig.cmake
  INSTALL_DESTINATION ${CONFIG_PACKAGE_LOCATION}
  PATH_VARS INCLUDE_INSTALL_DIR
  )

export(EXPORT rpclibTargets
  FILE "${CMAKE_CURRENT_BINARY_DIR}/cmake/rpclibTargets.cmake"
  NAMESPACE rpclib::
  )

install(EXPORT rpclibTargets
  FILE rpclibTargets.cmake
  NAMESPACE rpclib::
  DESTINATION ${CONFIG_PACKAGE_LOCATION}
  )
install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/cmake/rpclibConfig.cmake
  ${CMAKE_CURRENT_BINARY_DIR}/cmake/rpclibConfigVersion.cmake
  DESTINATION ${CONFIG_PACKAGE_LOCATION}
  )

#
# Pkg-config
#
if(NOT MSVC)  # Don't install pkg-config files when building with MSVC
  # Variables for pkg-config files
  set(prefix "${CMAKE_INSTALL_PREFIX}")
  set(exec_prefix "")
  set(libdir "${CMAKE_INSTALL_PREFIX}/lib")
  set(includedir "${CMAKE_INSTALL_PREFIX}/include")
  set(rpclib_version ${PROJECT_VERSION})
  get_target_property(rpclib_cflags ${PROJECT_NAME} COMPILE_OPTIONS)
  string(REPLACE ";" " " rpclib_cflags "${rpclib_cflags}") # Convert list to string

  configure_file(rpclib.pc.in "${CMAKE_CURRENT_BINARY_DIR}/rpclib.pc" @ONLY)
  install(FILES "${CMAKE_CURRENT_BINARY_DIR}/rpclib.pc" DESTINATION "${libdir}/pkgconfig")
endif()

