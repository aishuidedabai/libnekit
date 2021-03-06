cmake_minimum_required(VERSION 3.0.0)

if (CMAKE_VERSION VERSION_LESS "3.7.1")
  message(WARNING "It is highly recommended to used the latest version of CMake.")
  message(WARNING "To support Android, you should use at least 3.7.1")
endif()

project(libnekit C CXX)

set(PLATFORM "mac" CACHE STRING "Name of the platform.")

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(CheckCXXCompilerFlag)

check_cxx_compiler_flag(-Wall HAS_WALL)
if(HAS_WALL)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall")
endif()

check_cxx_compiler_flag(-Wextra HAS_WEXTRA)
if(HAS_WEXTRA)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wextra")
endif()

check_cxx_compiler_flag(-pedantic HAS_PEDANTIC)
if(HAS_PEDANTIC)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pedantic")
endif()

check_cxx_compiler_flag(-Wno-pragma-once-outside-header HAS_PRAGMA_ONCE_WARNING)
if(HAS_PRAGMA_ONCE_WARNING)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-pragma-once-outside-header")
endif()

option(COVERAGE "Generate coverage report" OFF)
if(COVERAGE)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -O0 --coverage")
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} --coverage")
endif()

include_directories("include")
include_directories("modules")
include_directories(SYSTEM "deps/${PLATFORM}/include")

set(CMAKE_FIND_ROOT_PATH "${PROJECT_SOURCE_DIR}/deps/${PLATFORM}")

if(APPLE AND NOT IOS)
  execute_process(COMMAND scripts/find_include_path.sh WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} OUTPUT_VARIABLE INCLUDE_DIRS)
  string(REPLACE "\n" ";" INCLUDE_DIRS ${INCLUDE_DIRS})
  include_directories(AFTER SYSTEM ${INCLUDE_DIRS})
  message(STATUS "Added more include dirs for apple clang")
endif()

if(WIN32)
  add_definitions(-DELPP_WINSOCK2 -DWIN32_LEAN_AND_MEAN)
endif()

add_definitions(-DBOOST_ASIO_DISABLE_HANDLER_TYPE_REQUIREMENTS)

set(SOURCES
  src/transport/tcp_socket.cc
  src/transport/tcp_listener.cc
  src/transport/tunnel.cc
  src/transport/tcp_connector.cc
  src/transport/error_code.cc
  src/utils/boost_error.cc
  src/utils/system_resolver.cc
  src/utils/error.cc
  src/utils/logger.cc
  src/utils/cancelable.cc
  src/utils/maxmind.cc
  src/utils/subnet.cc
  src/utils/country_iso_code.cc
  src/utils/http_header_parser.cc
  src/init.cc
  src/proxy_manager.cc
  src/rule/rule_manager.cc
  src/rule/all_rule.cc
  src/rule/dns_fail_rule.cc
  src/rule/geo_rule.cc
  src/rule/domain_rule.cc
  src/rule/domain_regex_rule.cc
  src/rule/subnet_rule.cc
  src/instance.cc
  src/crypto/random.cc
  src/crypto/key_generator.cc
  src/crypto/stream_cipher_interface.cc
  src/utils/buffer.cc
  src/utils/endpoint.cc
  src/data_flow/socks5_server_data_flow.cc
  )

set(SOURCES ${SOURCES}
  modules/CxxUrl/url.cc
  modules/picohttpparser/picohttpparser.c)

if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/contrib" AND IS_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/contrib" AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/contrib/CMakeLists.txt")
  add_subdirectory("${CMAKE_CURRENT_SOURCE_DIR}/contrib")
endif()

cmake_policy(SET CMP0063 NEW)
add_library(nekit ${SOURCES})
set_target_properties(nekit PROPERTIES CXX_VISIBILITY_PRESET hidden VISIBILITY_INLINES_HIDDEN YES)

find_library(BOOST boost
  NO_SYSTEM_ENVIROMENT_PATH)
if(NOT BOOST)
  message(FATAL_ERROR "Boost not found. Build dependency first.")
endif()
set(LIBS ${LIBS} ${BOOST})

find_library(OPENSSL crypto NO_SYSTEM_ENVIROMENT_PATH)
if(NOT OPENSSL)
  message(FATAL_ERROR "OpenSSL not found. Build dependency first.")
endif()
set(LIBS ${LIBS} ${OPENSSL})

find_library(SODIUM sodium NO_SYSTEM_ENVIROMENT_PATH)
if(NOT SODIUM)
  message(FATAL_ERROR "libsodium not found. Build dependency first.")
endif()
set(LIBS ${LIBS} ${SODIUM})

find_library(MAXMINDDB maxminddb NO_SYSTEM_ENVIROMENT_PATH)
if(NOT MAXMINDDB)
  message(FATAL_ERROR "maxminddb not found. Build dependency first.")
endif()
set(LIBS ${LIBS} ${MAXMINDDB})

target_link_libraries(nekit ${LIBS})

if (NOT IOS AND NOT ANDROID)
  enable_testing()
  add_subdirectory(test)
endif()

if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/app" AND IS_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/app" AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/app/CMakeLists.txt")
  add_subdirectory("${CMAKE_CURRENT_SOURCE_DIR}/app")
endif()
