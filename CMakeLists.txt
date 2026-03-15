# Minimum version determined by:
# https://cmake.org/cmake/help/latest/command/add_library.html#index-0-prop_tgt:PUBLIC_HEADER
cmake_minimum_required(VERSION 3.15)

project(uSockets VERSION "0.8.8" LANGUAGES C HOMEPAGE_URL "https://github.com/uNetworking/uSockets")

include(CMakePackageConfigHelpers)
include(GNUInstallDirs)
set(CMAKE_INSTALL_DOCDIR "${CMAKE_INSTALL_DATAROOTDIR}/doc/${PROJECT_NAME}")

#----------------------------------------------------------------
# Define and validate build options
#----------------------------------------------------------------

option(USOCKETS_ENABLE_SSL "Build with SSL/TLS support using OpenSSL or BoringSSL" OFF)
option(USOCKETS_ENABLE_QUIC "Build with QUIC support using LiteSpeed QUIC" OFF)

if(APPLE)
  option(USOCKETS_USE_GCD "Use libdispatch as the event-loop back end (macOS only)" OFF)
  set(default_is_libuv OFF)
elseif("${CMAKE_SYSTEM_NAME}" STREQUAL "Linux")
  option(
    USOCKETS_USE_IO_URING
    "Use io_uring as the event-loop and networking back end (Linux only)"
    OFF
  )
  set(default_is_libuv OFF)
else()
  set(default_is_libuv ON)
endif()
option(USOCKETS_USE_LIBUV "Use libuv as the event-loop back end" "${default_is_libuv}")
option(USOCKETS_USE_ASIO "Use Boost.Asio as the event-loop back end" OFF)

# Validate option values and determine the event-loop back end
if(
  (USOCKETS_USE_GCD AND (USOCKETS_USE_IO_URING OR USOCKETS_USE_LIBUV OR USOCKETS_USE_ASIO))
  OR (USOCKETS_USE_IO_URING AND (USOCKETS_USE_LIBUV OR USOCKETS_USE_ASIO))
  OR (USOCKETS_USE_LIBUV AND USOCKETS_USE_ASIO)
)
  set(msg "Only one event-loop back end may be enabled. Current settings:\n")
  if(DEFINED CACHE{USOCKETS_USE_GCD})
    list(APPEND msg "  USOCKETS_USE_GCD:      ${USOCKETS_USE_GCD}\n")
  endif()
  if(DEFINED CACHE{USOCKETS_USE_IO_URING})
    list(APPEND msg "  USOCKETS_USE_IO_URING: ${USOCKETS_USE_IO_URING}\n")
  endif()
  list(
    APPEND msg
    "  USOCKETS_USE_LIBUV:    ${USOCKETS_USE_LIBUV}\n"
    "  USOCKETS_USE_ASIO:     ${USOCKETS_USE_ASIO}"
  )
  message(FATAL_ERROR ${msg})
elseif(USOCKETS_USE_GCD)
  if(NOT APPLE)
    message(
      FATAL_ERROR
      "libdispatch cannot be used on this OS; USOCKETS_USE_GCD=${USOCKETS_USE_GCD} is valid\n"
      "only on macOS."
    )
  endif()
  set(eventloop_backend GCD)
elseif(USOCKETS_USE_IO_URING)
  if(NOT "${CMAKE_SYSTEM_NAME}" STREQUAL "Linux")
    message(
      FATAL_ERROR
      "io_uring cannot be used on this OS; USOCKETS_USE_IO_URING=${USOCKETS_USE_IO_URING} is valid\n"
      "only on Linux."
    )
  endif()
  set(eventloop_backend IO_URING)
elseif(USOCKETS_USE_ASIO)
  set(eventloop_backend ASIO)
elseif(USOCKETS_USE_LIBUV)
  set(eventloop_backend LIBUV)
else() # Use platform-specific defaults
  if("${CMAKE_SYSTEM_NAME}" STREQUAL "Linux")
    set(eventloop_backend EPOLL)
  elseif(UNIX) # macOS and FreeBSD
    set(eventloop_backend KQUEUE)
  else() # Windows and others
    message(FATAL_ERROR "This OS must use libuv (default) or Boost.Asio.")
  endif()
endif()

#----------------------------------------------------------------
# Find dependencies
#----------------------------------------------------------------

if(USOCKETS_ENABLE_SSL OR USOCKETS_USE_ASIO)
  enable_language(CXX)
endif()

if(USOCKETS_ENABLE_SSL)
  if(NOT BUILD_SHARED_LIBS)
    set(OPENSSL_USE_STATIC_LIBS ON)
  endif()
  find_package(OpenSSL 1.1 REQUIRED COMPONENTS SSL)
endif()

if(USOCKETS_ENABLE_QUIC)
  find_package(lsquic REQUIRED)
endif()

if(USOCKETS_USE_IO_URING)
  find_package(PkgConfig MODULE REQUIRED)
  pkg_check_modules(liburing REQUIRED IMPORTED_TARGET liburing)
elseif(USOCKETS_USE_ASIO)
  if(NOT BUILD_SHARED_LIBS)
    set(Boost_USE_STATIC_LIBS ON)
  endif()
  # Using v1.69 because since that version Boost.Asio is fully header-only and does not
  # require linking with other Boost libraries
  find_package(Boost 1.69 REQUIRED)
elseif(USOCKETS_USE_LIBUV)
  find_package(libuv REQUIRED)
endif()

#----------------------------------------------------------------
# Define build targets
#----------------------------------------------------------------

# Verbose name to reduce chances of conflicts when this project is built as part of
# a superbuild project
add_library(unetworking_usockets)
target_compile_features(unetworking_usockets PRIVATE c_std_11)
target_include_directories(
  unetworking_usockets
  PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>"
  # An `$<INSTALL_INTERFACE:...>` counterpart is not needed because calling the `install(TARGETS)`
  # command with the `INCLUDES DESTINATION` argument, as done later in this file, sets the
  # appropriate `INTERFACE_INCLUDE_DIRECTORIES` at install time.
)
target_sources(
  unetworking_usockets
  PRIVATE
    "src/libusockets.h"
    "src/bsd.c"
    "src/context.c"
    "src/loop.c"
    "src/socket.c"
    "src/udp.c"
    "src/internal/internal.h"
    "src/internal/loop_data.h"
    "src/internal/networking/bsd.h"
    # SSL
    "$<$<BOOL:${USOCKETS_ENABLE_SSL}>:src/crypto/openssl.c>"
    "$<$<BOOL:${USOCKETS_ENABLE_SSL}>:src/crypto/sni_tree.cpp>"
    # QUIC
    "$<$<BOOL:${USOCKETS_ENABLE_QUIC}>:src/quic.h>"
    "$<$<BOOL:${USOCKETS_ENABLE_QUIC}>:src/quic.c>"
    # direct syscalls on Unix (epoll/kqueue)
    "$<$<OR:$<STREQUAL:${eventloop_backend},EPOLL>,$<STREQUAL:${eventloop_backend},KQUEUE>>:src/internal/eventing/epoll_kqueue.h>"
    "$<$<OR:$<STREQUAL:${eventloop_backend},EPOLL>,$<STREQUAL:${eventloop_backend},KQUEUE>>:src/eventing/epoll_kqueue.c>"
    # libdispatch
    "$<$<STREQUAL:${eventloop_backend},GCD>:src/internal/eventing/gcd.h>"
    "$<$<STREQUAL:${eventloop_backend},GCD>:src/eventing/gcd.c>"
    # io_uring
    "$<$<STREQUAL:${eventloop_backend},IO_URING>:src/io_uring/internal.h>"
    "$<$<STREQUAL:${eventloop_backend},IO_URING>:src/io_uring/io_context.c>"
    "$<$<STREQUAL:${eventloop_backend},IO_URING>:src/io_uring/io_loop.c>"
    "$<$<STREQUAL:${eventloop_backend},IO_URING>:src/io_uring/io_socket.c>"
    # Boost.Asio
    "$<$<STREQUAL:${eventloop_backend},ASIO>:src/internal/eventing/asio.h>"
    "$<$<STREQUAL:${eventloop_backend},ASIO>:src/eventing/asio.cpp>"
    # libuv
    "$<$<STREQUAL:${eventloop_backend},LIBUV>:src/internal/eventing/libuv.h>"
    "$<$<STREQUAL:${eventloop_backend},LIBUV>:src/eventing/libuv.c>"
)
set(public_headers "src/libusockets.h")
if(USOCKETS_ENABLE_QUIC)
  list(APPEND public_headers "src/quic.h")
endif()

set_target_properties(
  unetworking_usockets
  PROPERTIES
    PUBLIC_HEADER "${public_headers}"
    OUTPUT_NAME usockets
    # Define interface compatibility requirements for consumers of the library
    COMPATIBLE_INTERFACE_BOOL "SSL_SUPPORT;QUIC_SUPPORT"
    COMPATIBLE_INTERFACE_STRING EVENT_LOOP_BACKEND
    INTERFACE_SSL_SUPPORT "${USOCKETS_ENABLE_SSL}"
    INTERFACE_QUIC_SUPPORT "${USOCKETS_ENABLE_QUIC}"
    INTERFACE_EVENT_LOOP_BACKEND "${eventloop_backend}"
)
if(BUILD_SHARED_LIBS)
  set_target_properties(
    unetworking_usockets
    PROPERTIES
      EXPORT_NAME usockets_shared
      SOVERSION "${PROJECT_VERSION_MAJOR}"
      VERSION "${PROJECT_VERSION}"
  )
else()
  set_target_properties(
    unetworking_usockets
    PROPERTIES EXPORT_NAME usockets_static POSITION_INDEPENDENT_CODE ON
  )
endif()

if(WIN32)
  target_link_libraries(unetworking_usockets PUBLIC ws2_32)
endif()

if(USOCKETS_ENABLE_SSL)
  target_compile_features(unetworking_usockets PRIVATE cxx_std_17)
  target_compile_definitions(unetworking_usockets PUBLIC "LIBUS_USE_OPENSSL")
  target_link_libraries(unetworking_usockets PRIVATE OpenSSL::SSL)
else()
  target_compile_definitions(unetworking_usockets PUBLIC "LIBUS_NO_SSL")
endif()

if(USOCKETS_ENABLE_QUIC)
  target_compile_definitions(unetworking_usockets PUBLIC "LIBUS_USE_QUIC")
  target_link_libraries(unetworking_usockets PRIVATE lsquic::lsquic)
endif()

target_compile_definitions(unetworking_usockets PUBLIC "LIBUS_USE_${eventloop_backend}")
if("${eventloop_backend}" STREQUAL "IO_URING")
  target_link_libraries(unetworking_usockets PRIVATE PkgConfig::liburing)
elseif("${eventloop_backend}" STREQUAL "ASIO")
  if(NOT USOCKETS_ENABLE_SSL)
    target_compile_features(unetworking_usockets PRIVATE cxx_std_14)
  endif()
  target_link_libraries(unetworking_usockets PRIVATE Boost::headers)
elseif("${eventloop_backend}" STREQUAL "LIBUV")
  if(BUILD_SHARED_LIBS)
    target_link_libraries(unetworking_usockets PRIVATE libuv::uv)
  else()
    target_link_libraries(unetworking_usockets PRIVATE libuv::uv_a)
  endif()
endif()

#----------------------------------------------------------------
# Set export and install rules
#----------------------------------------------------------------

set(package_name "${PROJECT_NAME}")
set(config_install_dir "${CMAKE_INSTALL_LIBDIR}/cmake/${package_name}")
set(exports_name_prefix "${package_name}Targets") # used also in the config package input file
set(exports_name "${exports_name_prefix}-$<IF:$<BOOL:BUILD_SHARED_LIBS>,shared,static>")

configure_file(
  "cmake/uSocketsConfig.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/${package_name}Config.cmake"
  @ONLY
)
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/${package_name}ConfigVersion.cmake"
  COMPATIBILITY AnyNewerVersion
)

install(
  TARGETS unetworking_usockets
  EXPORT "${exports_name}"
  INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
  PUBLIC_HEADER DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
  ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
  LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
  RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
)
install(
  EXPORT "${exports_name}"
  DESTINATION "${config_install_dir}"
  NAMESPACE "${package_name}::"
)
install(
  FILES
    "${CMAKE_CURRENT_BINARY_DIR}/${package_name}Config.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/${package_name}ConfigVersion.cmake"
  DESTINATION "${config_install_dir}"
)
install(FILES "misc/manual.md" "misc/layout.png" TYPE DOC)
install(DIRECTORY "examples" TYPE DOC)
