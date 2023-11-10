cmake_minimum_required(VERSION 3.27)

project(dropwatch)

# Settings and Checking build binay in the root of source dir IS NOT ALLOWED
if("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_BINARY_DIR}")
  message(
    FATAL_ERROR
      "FATAL: In-source builds are not allowed.
           You should create a separate directory for build files.
           ")
endif()

# Generating a compilation database: compile_commands.json
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

option(CMAKE_BUILD_TYPE "Release")
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Release")
endif()

message(STATUS "CMAKE Version: ${CMAKE_VERSION}")
message(STATUS "System name: ${CMAKE_SYSTEM_NAME}")
message(STATUS "Host System name: ${CMAKE_HOST_SYSTEM_NAME}")
message(STATUS "System version: ${CMAKE_SYSTEM_VERSION}")
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")
message(STATUS "C compiler: ${CMAKE_C_COMPILER}")
message(STATUS "C++ compiler: ${CMAKE_CXX_COMPILER}")

# Set default compiler flags
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${CMAKE_NO_WARN_FLAGS}")

# Set build specific compiler flags
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -O0 -g")
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O3 -g")
set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO} -g -O3")

# dump if set it to true
set(CMAKE_VERBOSE_MAKEFILE false)

# dependency
if(NOT CMAKE_CROSSCOMPILING)
  # mismatch with libnl3
  include_directories(/usr/include/libnl3)
else()
  include(FetchContent)
  message(STATUS "CMAKE_CROSSCOMPILING: ${CMAKE_C_COMPILER}")
  # libpcap
  FetchContent_Declare(
    libpcap
    GIT_REPOSITORY https://github.com/the-tcpdump-group/libpcap
    GIT_TAG libpcap-1.9.1)
  set(DISABLE_DBUS
      ON
      CACHE BOOL "" FORCE)
  set(DISABLE_RDMA
      ON
      CACHE BOOL "" FORCE)
  set(DISABLE_TC
      ON
      CACHE BOOL "" FORCE)
  set(DISABLE_DAG
      ON
      CACHE BOOL "" FORCE)
  set(DISABLE_DPDK
      ON
      CACHE BOOL "" FORCE)
  set(DISABLE_SNF
      ON
      CACHE BOOL "" FORCE)
  set(DISABLE_AIRPCAP
      ON
      CACHE BOOL "" FORCE)
  include_directories(${libpcap_SOURCE_DIR}/)
  FetchContent_MakeAvailable(libpcap)
  # readline
  #[[
  message(CHECK_START "Fetching readline")
  set(readline_patch
      ${CMAKE_CURRENT_SOURCE_DIR}/patches/0001-build-add-cmake-patch.patch)
  FetchContent_Declare(
    readline
    GIT_REPOSITORY https://git.savannah.gnu.org/git/readline.git
    GIT_TAG readline-8.2
    PATCH_COMMAND git apply --ignore-whitespace ${readline_patch}
    UPDATE_DISCONNECTED 1)
  FetchContent_MakeAvailable(readline)
  include_directories(${readline_SOURCE_DIR}/)
  message(CHECK_PASS "Fetched")
  ]]
endif()

# usage: CHECK_INCLUDE_FILES (<header> <RESULT_VARIABLE> )
include(CheckIncludeFiles)
include(CheckFunctionExists)
include(CheckLibraryExists)

check_library_exists(pcap pcap_lookupdev "" HAVE_LIBPCAP)
check_function_exists(getopt_long HAVE_GETOPT_LONG)
check_include_files(bfd.h HAVE_BFD_H)
check_include_files(inttypes.h HAVE_INTTYPES_H)
check_include_files(stdint.h HAVE_STDINT_H)
check_include_files(stdio.h HAVE_STDIO_H)
check_include_files(stdio.h HAVE_STDLIB_H)
check_include_files(stdlib.h HAVE_STDIO_H)
check_include_files(strings.h HAVE_STRINGS_H)
check_include_files(sys/stat.h HAVE_SYS_STAT_H)
check_include_files(sys/types.h HAVE_SYS_TYPES_H)
check_include_files(unistd.h HAVE_UNISTD_H)

configure_file("${PROJECT_SOURCE_DIR}/config.h.in"
               "${PROJECT_SOURCE_DIR}/config.h")
# dropwatch
include_directories(${PROJECT_SOURCE_DIR}/src)

add_executable(
  dropwatch
  ${PROJECT_SOURCE_DIR}/src/main.c ${PROJECT_SOURCE_DIR}/src/lookup.c
  ${PROJECT_SOURCE_DIR}/src/lookup_bfd.c ${PROJECT_SOURCE_DIR}/src/lookup_kas.c)
# target_include_directories(dropwatch PUBLIC ${PROJECT_SOURCE_DIR})
target_link_libraries(dropwatch nl-genl-3 nl-3 pcap readline bfd)

get_property(
  dirs
  DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  PROPERTY INCLUDE_DIRECTORIES)
foreach(dir ${dirs})
  message(STATUS "dropwatch include path: '${dir}'")
endforeach()

add_executable(
  dwdump
  ${PROJECT_SOURCE_DIR}/src/dwdump.c ${PROJECT_SOURCE_DIR}/src/lookup.c
  ${PROJECT_SOURCE_DIR}/src/lookup_bfd.c ${PROJECT_SOURCE_DIR}/src/lookup_kas.c)
target_link_libraries(dwdump nl-genl-3 nl-3 pcap readline bfd)
install(TARGETS dropwatch dwdump DESTINATION ${CMAKE_INSTALL_PREFIX}/bin)
