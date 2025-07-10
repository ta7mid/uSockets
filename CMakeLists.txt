cmake_minimum_required(VERSION 3.14)
project(uSockets VERSION 0.8.8 LANGUAGES C)

#----------------------------------------------------------------
# Options and dependencies
#----------------------------------------------------------------

# Create the options

option(ENABLE_SSL "Enable SSL support" OFF)
option(ENABLE_QUIC "Enable QUIC support using LiteSpeed QUIC" OFF)

if(APPLE)
  option(USE_GCD "Use libdispatch as the event-loop back end (macOS only)" OFF)
elseif("${CMAKE_SYSTEM_NAME}" STREQUAL "Linux")
  option(USE_IO_URING "Use io_uring as the event-loop and networking back end (Linux only)" OFF)
else()
  # https://github.com/uNetworking/uSockets/blob/8606de6414a102c55bef8e8ef3391932d7e8df6a/src/libusockets.h#L339-L348
  set(libuv_is_default ON)
endif()

option(USE_LIBUV "Use libuv as the event-loop back end" $<BOOL:${libuv_is_default}>)
option(USE_ASIO "Use Boost.Asio as the event-loop back end" OFF)

# Validate options and make ependencies available

if((USE_GCD AND (USE_IO_URING OR USE_LIBUV OR USE_ASIO)) OR
    (USE_IO_URING AND (USE_LIBUV OR USE_ASIO)) OR
    (USE_LIBUV AND USE_ASIO))
  set(msg
    "Only one event-loop back end may be enabled.\n"
    "Selection:\n"
  )
  if(DEFINED CACHE{USE_GCD})
    list(APPEND msg
      "  USE_GCD:      $CACHE{USE_GCD}\n"
    )
  endif()
  if(DEFINED CACHE{USE_IO_URING})
    list(APPEND msg
      "  USE_IO_URING: $CACHE{USE_IO_URING}\n"
    )
  endif()
  list(APPEND msg
    "  USE_LIBUV:    $CACHE{USE_LIBUV}\n"
    "  USE_ASIO:     $CACHE{USE_ASIO}"
  )
  message(FATAL_ERROR ${msg})
  unset(msg)
endif()

if(ENABLE_SSL OR USE_ASIO)
  enable_language(CXX)
endif()

if(ENABLE_SSL)
  find_package(OpenSSL 1.1 REQUIRED)
endif()

if(ENABLE_QUIC)
  find_package(lsquic REQUIRED)
endif()

if(USE_GCD)
  if(NOT APPLE)
    unset(USE_GCD)
    message(FATAL_ERROR "USE_GCD is valid only on macOS.")
  endif()
  set(eventloop_backend "GCD")
elseif(USE_IO_URING)
  if(NOT "${CMAKE_SYSTEM_NAME}" STREQUAL "Linux")
    unset(USE_IO_URING)
    message(FATAL_ERROR "USE_IO_URING is valid only on Linux.")
  endif()
  find_package(liburing REQUIRED)
  set(eventloop_backend "IO_URING")
elseif(USE_ASIO)
  find_package(Boost 1.66 REQUIRED COMPONENTS asio)
  set(eventloop_backend "ASIO")
elseif(USE_LIBUV)
  find_package(libuv REQUIRED)
  set(eventloop_backend "LIBUV")
else() # use platform-specific defaults
  if("${CMAKE_SYSTEM_NAME}" STREQUAL "Linux")
    set(eventloop_backend "EPOLL")
  elseif(UNIX) # macOS and FreeBSD
    set(eventloop_backend "KQUEUE")
  else() # Windows and others
    message(FATAL_ERROR
      "One of libuv or Boost.Asio must be enabled on this platform; the default is libuv."
    )
  endif()
endif()

#----------------------------------------------------------------
# Targets and properties
#----------------------------------------------------------------

add_library(uSockets STATIC
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
  "src/crypto/openssl.c"
  "src/crypto/sni_tree.cpp"

  # QUIC
  "src/quic.h"
  "src/quic.c"

  # libdispatch
  "src/internal/eventing/gcd.h"
  "src/eventing/gcd.c"

  # io_uring
  "src/io_uring/internal.h"
  "src/io_uring/io_context.c"
  "src/io_uring/io_loop.c"
  "src/io_uring/io_socket.c"

  # Boost.Asio
  "src/internal/eventing/asio.h"
  "src/eventing/asio.cpp"

  # direct syscalls on Unix
  "src/internal/eventing/epoll_kqueue.h"
  "src/eventing/epoll_kqueue.c"

  # libuv
  "src/internal/eventing/libuv.h"
  "src/eventing/libuv.c"
)
set_target_properties(uSockets PROPERTIES "OUTPUT_NAME" "usockets")
target_include_directories(uSockets PRIVATE "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>")
# https://github.com/uNetworking/uSockets/blob/f0a52a0b8390d2c71c1dbefec3099b28d228f695/Makefile#L20
target_compile_features(uSockets PRIVATE c_std_11)

if(ENABLE_SSL)
  target_compile_definitions(uSockets PUBLIC "LIBUS_USE_OPENSSL")
  # https://github.com/uNetworking/uSockets/blob/0ebdde0601cc82349fc11a7c4bbb6dc5c9f28f42/Makefile#L55
  target_compile_features(uSockets PRIVATE cxx_std_17)
  target_link_libraries(uSockets PRIVATE OpenSSL::SSL OpenSSL::Crypto)
else()
  target_compile_definitions(uSockets PUBLIC "LIBUS_NO_SSL")
endif()

if(ENABLE_QUIC)
  target_compile_definitions(uSockets PUBLIC "LIBUS_USE_QUIC")
  target_link_libraries(uSockets PRIVATE lsquic::lsquic)
endif()

target_compile_definitions(uSockets PUBLIC "LIBUS_USE_${eventloop_backend}")
if("${eventloop_backend}" STREQUAL "IO_URING")
  target_link_libraries(uSockets PRIVATE liburing::liburing)
elseif("${eventloop_backend}" STREQUAL "ASIO")
  target_compile_features(uSockets PRIVATE cxx_std_14)
  target_link_libraries(uSockets PRIVATE Boost::asio)
elseif("${eventloop_backend}" STREQUAL "LIBUV")
  target_link_libraries(uSockets PRIVATE $<IF:$<TARGET_EXISTS:libuv::uv_a>,libuv::uv_a,libuv::uv>)
  if(WIN32)
    # https://github.com/uNetworking/uSockets/blob/8606de6414a102c55bef8e8ef3391932d7e8df6a/src/libusockets.h#L35
    target_link_libraries(uSockets PRIVATE ws2_32)
  endif()
endif()

#----------------------------------------------------------------
# Installation
#----------------------------------------------------------------

include(GNUInstallDirs)

install(FILES
  "src/libusockets.h"
  "$<$<BOOL:${ENABLE_QUIC}>:src/quic.h>"
  DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)
target_include_directories(uSockets
  INTERFACE "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>"
)

install(TARGETS uSockets EXPORT uSocketsConfig)

install(EXPORT uSocketsConfig
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/uSockets"
  NAMESPACE "uSockets::"
)
