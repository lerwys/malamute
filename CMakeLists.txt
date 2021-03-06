################################################################################
#  THIS FILE IS 100% GENERATED BY ZPROJECT; DO NOT EDIT EXCEPT EXPERIMENTALLY  #
#  Please refer to the README for information about making permanent changes.  #
################################################################################

########################################################################
# Project setup
########################################################################
cmake_minimum_required(VERSION 2.8)
project(malamute)
enable_language(C)
enable_testing()

set(SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}")

########################################################################
# options
########################################################################
if (EXISTS "${SOURCE_DIR}/.git")
    OPTION (ENABLE_DRAFTS "Build and install draft classes and methods" ON)
else ()
    OPTION (ENABLE_DRAFTS "Build and install draft classes and methods" OFF)
endif ()

IF (ENABLE_DRAFTS)
    ADD_DEFINITIONS (-DMLM_BUILD_DRAFT_API)
ENDIF (ENABLE_DRAFTS)

########################################################################
# platform.h
########################################################################
include(CheckIncludeFile)
CHECK_INCLUDE_FILE("linux/wireless.h" HAVE_LINUX_WIRELESS_H)
CHECK_INCLUDE_FILE("net/if_media.h" HAVE_NET_IF_MEDIA_H)

include(CheckFunctionExists)
CHECK_FUNCTION_EXISTS("getifaddrs" HAVE_GETIFADDRS)
CHECK_FUNCTION_EXISTS("freeifaddrs" HAVE_FREEIFADDRS)

include(CheckIncludeFiles)
check_include_files("sys/socket.h;net/if.h" HAVE_NET_IF_H)
if (NOT HAVE_NET_IF_H)
    CHECK_INCLUDE_FILE("net/if.h" HAVE_NET_IF_H)
endif()

file(WRITE "${SOURCE_DIR}/src/platform.h.in" "
#cmakedefine HAVE_LINUX_WIRELESS_H
#cmakedefine HAVE_NET_IF_H
#cmakedefine HAVE_NET_IF_MEDIA_H
#cmakedefine HAVE_GETIFADDRS
#cmakedefine HAVE_FREEIFADDRS
")

configure_file("${SOURCE_DIR}/src/platform.h.in" "${SOURCE_DIR}/src/platform.h")

#The MSVC C compiler is too out of date,
#so the sources have to be compiled as c++
if (MSVC)
    enable_language(CXX)
    file(GLOB sources "${SOURCE_DIR}/src/*.c")
    set_source_files_properties(
        ${sources}
        PROPERTIES LANGUAGE CXX
    )
    set(MORE_LIBRARIES ws2_32 Rpcrt4 Iphlpapi)
endif()

# required libraries for mingw
if (MINGW)
    set(MORE_LIBRARIES -lws2_32 -lrpcrt4 -liphlpapi)
endif()

# required libraries for cygwin
if (CYGWIN)
    set(MORE_LIBRARIES -luuid)
endif()

list(APPEND CMAKE_MODULE_PATH "${SOURCE_DIR}")
set(OPTIONAL_LIBRARIES)

########################################################################
# LIBZMQ dependency
########################################################################
find_package(libzmq REQUIRED)
IF (LIBZMQ_FOUND)
    include_directories(${LIBZMQ_INCLUDE_DIRS})
    list(APPEND MORE_LIBRARIES ${LIBZMQ_LIBRARIES})
ELSE (LIBZMQ_FOUND)
    message( FATAL_ERROR "libzmq not found." )
ENDIF (LIBZMQ_FOUND)

########################################################################
# UUID dependency
########################################################################
find_package(uuid)
IF (UUID_FOUND)
    include_directories(${UUID_INCLUDE_DIRS})
    list(APPEND MORE_LIBRARIES ${UUID_LIBRARIES})
    add_definitions(-DHAVE_UUID)
    list(APPEND OPTIONAL_LIBRARIES ${UUID_LIBRARIES})
ENDIF (UUID_FOUND)

########################################################################
# CZMQ dependency
########################################################################
find_package(czmq REQUIRED)
IF (CZMQ_FOUND)
    include_directories(${CZMQ_INCLUDE_DIRS})
    list(APPEND MORE_LIBRARIES ${CZMQ_LIBRARIES})
ELSE (CZMQ_FOUND)
    message( FATAL_ERROR "czmq not found." )
ENDIF (CZMQ_FOUND)

########################################################################
# includes
########################################################################
set (mlm_headers
    include/mlm_library.h
    include/malamute.h
    src/mlm_msg.h
    src/mlm_stream_simple.h
    src/mlm_mailbox_simple.h
    src/mlm_server_engine.inc
    src/mlm_client_engine.inc
    src/mlm_client_custom.xml
)

IF (ENABLE_DRAFTS)
    list(APPEND mlm_headers
        include/mlm_proto.h
        include/mlm_server.h
        include/mlm_client.h
    )
ENDIF (ENABLE_DRAFTS)

source_group ("Header Files" FILES ${mlm_headers})
install(FILES ${mlm_headers} DESTINATION include)

########################################################################
# library
########################################################################
include_directories("${SOURCE_DIR}/src" "${SOURCE_DIR}/include")
set (mlm_sources
    src/mlm_msg.c
    src/mlm_stream_simple.c
    src/mlm_mailbox_simple.c
)

IF (ENABLE_DRAFTS)
    list (APPEND mlm_sources
        src/mlm_proto.c
        src/mlm_server.c
        src/mlm_client.c
    )
ENDIF (ENABLE_DRAFTS)

source_group("Source Files" FILES ${mlm_sources})
if (NOT DEFINED BUILD_SHARED_LIBS)
    SET(BUILD_SHARED_LIBS ON)
endif()
add_library(mlm ${mlm_sources})
set_target_properties(mlm
    PROPERTIES DEFINE_SYMBOL "LIBMLM_EXPORTS"
)
set_target_properties(mlm
    PROPERTIES LIBRARY_OUTPUT_DIRECTORY "${SOURCE_DIR}/src"
)
target_link_libraries(mlm
    ${ZEROMQ_LIBRARIES} ${MORE_LIBRARIES}
)

install(TARGETS mlm
    LIBRARY DESTINATION "lib${LIB_SUFFIX}" # .so file
    ARCHIVE DESTINATION "lib${LIB_SUFFIX}" # .lib file
    RUNTIME DESTINATION bin              # .dll file
)

########################################################################
# pkgconfig
########################################################################
set (VERSION "1.1.0")
set (prefix "${CMAKE_INSTALL_PREFIX}")
set (exec_prefix "\${prefix}")
set (libdir "\${prefix}/lib${LIB_SUFFIX}")
set (includedir "\${prefix}/include")
IF (ENABLE_DRAFTS)
    set (pkg_config_defines "-DMLM_BUILD_DRAFT_API=1")
ELSE (ENABLE_DRAFTS)
    set (pkg_config_defines "")
ENDIF (ENABLE_DRAFTS)
configure_file(
    "${SOURCE_DIR}/src/libmlm.pc.in"
    "${SOURCE_DIR}/src/libmlm.pc"
@ONLY)

install(
    FILES "${SOURCE_DIR}/src/libmlm.pc"
    DESTINATION "lib${LIB_SUFFIX}/pkgconfig"
)

########################################################################
# executables
########################################################################
add_executable(
    malamute
    "${SOURCE_DIR}/src/malamute.c"
)
target_link_libraries(
    malamute
    mlm
    ${LIBZMQ_LIBRARIES}
    ${CZMQ_LIBRARIES}
    ${OPTIONAL_LIBRARIES}
)
set_target_properties(
    malamute
    PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${SOURCE_DIR}/src"
)
add_executable(
    mshell
    "${SOURCE_DIR}/src/mshell.c"
)
target_link_libraries(
    mshell
    mlm
    ${LIBZMQ_LIBRARIES}
    ${CZMQ_LIBRARIES}
    ${OPTIONAL_LIBRARIES}
)
set_target_properties(
    mshell
    PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${SOURCE_DIR}/src"
)
add_executable(
    mlm_tutorial
    "${SOURCE_DIR}/src/mlm_tutorial.c"
)
target_link_libraries(
    mlm_tutorial
    mlm
    ${LIBZMQ_LIBRARIES}
    ${CZMQ_LIBRARIES}
    ${OPTIONAL_LIBRARIES}
)
set_target_properties(
    mlm_tutorial
    PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${SOURCE_DIR}/src"
)
add_executable(
    mlm_perftest
    "${SOURCE_DIR}/src/mlm_perftest.c"
)
target_link_libraries(
    mlm_perftest
    mlm
    ${LIBZMQ_LIBRARIES}
    ${CZMQ_LIBRARIES}
    ${OPTIONAL_LIBRARIES}
)
set_target_properties(
    mlm_perftest
    PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${SOURCE_DIR}/src"
)
add_executable(
    mlm_selftest
    "${SOURCE_DIR}/src/mlm_selftest.c"
)
target_link_libraries(
    mlm_selftest
    mlm
    ${LIBZMQ_LIBRARIES}
    ${CZMQ_LIBRARIES}
    ${OPTIONAL_LIBRARIES}
)
set_target_properties(
    mlm_selftest
    PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${SOURCE_DIR}/src"
)

########################################################################
# tests
########################################################################
set(CLASSTEST_TIMEOUT 5 CACHE STRING "Timeout of the selftest of a class")
set(TOTAL_TIMEOUT 20 CACHE STRING "Timout of the total testsuite")

set(TEST_CLASSES
    mlm_msg
    mlm_stream_simple
    mlm_mailbox_simple
)

IF (ENABLE_DRAFTS)
    list (APPEND TEST_CLASSES
    mlm_proto
    mlm_server
    mlm_client
    )
ENDIF (ENABLE_DRAFTS)

foreach(TEST_CLASS ${TEST_CLASSES})
    add_test(
        NAME ${TEST_CLASS}
        COMMAND mlm_selftest --continue --verbose --test ${TEST_CLASS}
    )
    set_tests_properties(
        ${TEST_CLASS}
        PROPERTIES TIMEOUT ${CLASSTEST_TIMEOUT}
    )
endforeach(TEST_CLASS)


########################################################################
# summary
########################################################################
message (STATUS "*** Version number: ${VERSION}")
message (STATUS "*** Install target: ${CMAKE_INSTALL_PREFIX}")

################################################################################
#  THIS FILE IS 100% GENERATED BY ZPROJECT; DO NOT EDIT EXCEPT EXPERIMENTALLY  #
#  Please refer to the README for information about making permanent changes.  #
################################################################################
