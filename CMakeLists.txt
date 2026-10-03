cmake_minimum_required(VERSION 2.8.12...3.20)

project(enet)

option(ENET_NO_INSTALL "Disable installation of headers and libraries" OFF)
option(ENET_BUILD_TESTS "Build opt-in send admission regression tests" OFF)

# The "configure" step.
include(CheckFunctionExists)
include(CheckStructHasMember)
include(CheckTypeSize)
check_function_exists("fcntl" HAS_FCNTL)
check_function_exists("ioctl" HAS_IOCTL)
check_function_exists("poll" HAS_POLL)
check_function_exists("getaddrinfo" HAS_GETADDRINFO)
check_function_exists("getnameinfo" HAS_GETNAMEINFO)
check_function_exists("gethostbyname_r" HAS_GETHOSTBYNAME_R)
check_function_exists("gethostbyaddr_r" HAS_GETHOSTBYADDR_R)
check_function_exists("inet_pton" HAS_INET_PTON)
check_function_exists("inet_ntop" HAS_INET_NTOP)
check_struct_has_member("struct msghdr" "msg_flags" "sys/types.h;sys/socket.h" HAS_MSGHDR_FLAGS)
set(CMAKE_EXTRA_INCLUDE_FILES "sys/types.h" "sys/socket.h")
check_type_size("socklen_t" HAS_SOCKLEN_T BUILTIN_TYPES_ONLY)
unset(CMAKE_EXTRA_INCLUDE_FILES)
set(CMAKE_EXTRA_INCLUDE_FILES "winsock2.h;qos2.h")
check_type_size("QOS_FLOWID" HAS_QOS_FLOWID BUILTIN_TYPES_ONLY)
check_type_size("PQOS_FLOWID" HAS_PQOS_FLOWID BUILTIN_TYPES_ONLY)
unset(CMAKE_EXTRA_INCLUDE_FILES)
if(MSVC)
	add_definitions(-W3)
else()
	add_definitions(-Wno-error)
endif()

if(HAS_FCNTL)
    add_definitions(-DHAS_FCNTL=1)
endif()
if(HAS_IOCTL)
    add_definitions(-DHAS_IOCTL=1)
endif()
if(HAS_POLL)
    add_definitions(-DHAS_POLL=1)
endif()
if(HAS_GETNAMEINFO)
    add_definitions(-DHAS_GETNAMEINFO=1)
endif()
if(HAS_GETADDRINFO)
    add_definitions(-DHAS_GETADDRINFO=1)
endif()
if(HAS_GETHOSTBYNAME_R)
    add_definitions(-DHAS_GETHOSTBYNAME_R=1)
endif()
if(HAS_GETHOSTBYADDR_R)
    add_definitions(-DHAS_GETHOSTBYADDR_R=1)
endif()
if(HAS_INET_PTON)
    add_definitions(-DHAS_INET_PTON=1)
endif()
if(HAS_INET_NTOP)
    add_definitions(-DHAS_INET_NTOP=1)
endif()
if(HAS_MSGHDR_FLAGS)
    add_definitions(-DHAS_MSGHDR_FLAGS=1)
endif()
if(HAS_SOCKLEN_T)
    add_definitions(-DHAS_SOCKLEN_T=1)
endif()
if(HAS_QOS_FLOWID)
    add_definitions(-DHAS_QOS_FLOWID=1)
endif()
if(HAS_PQOS_FLOWID)
    add_definitions(-DHAS_PQOS_FLOWID=1)
endif()

include_directories(${PROJECT_SOURCE_DIR}/include)

set(INCLUDE_FILES_PREFIX include/enet)
set(INCLUDE_FILES
    ${INCLUDE_FILES_PREFIX}/callbacks.h
    ${INCLUDE_FILES_PREFIX}/enet.h
    ${INCLUDE_FILES_PREFIX}/list.h
    ${INCLUDE_FILES_PREFIX}/protocol.h
    ${INCLUDE_FILES_PREFIX}/time.h
    ${INCLUDE_FILES_PREFIX}/types.h
    ${INCLUDE_FILES_PREFIX}/unix.h
    ${INCLUDE_FILES_PREFIX}/utility.h
    ${INCLUDE_FILES_PREFIX}/win32.h
)

set(SOURCE_FILES
    callbacks.c
    compress.c
    host.c
    list.c
    packet.c
    peer.c
    protocol.c
    unix.c
    win32.c)

source_group(include FILES ${INCLUDE_FILES})
source_group(source FILES ${SOURCE_FILES})

add_library(enet STATIC
    ${INCLUDE_FILES}
    ${SOURCE_FILES}
)
target_include_directories(enet SYSTEM PUBLIC include)

if (MINGW)
    target_link_libraries(enet winmm ws2_32)
endif()

if(ENET_BUILD_TESTS)
    enable_testing()
    find_package(GTest CONFIG QUIET)
    if(NOT TARGET GTest::gtest_main AND NOT TARGET gtest_main)
        if(NOT GTEST_SOURCE_DIR)
            message(FATAL_ERROR "Install GTest or specify GTEST_SOURCE_DIR to build ENet tests")
        endif()
        add_subdirectory("${GTEST_SOURCE_DIR}" "${CMAKE_CURRENT_BINARY_DIR}/googletest" EXCLUDE_FROM_ALL)
    endif()
    add_executable(enet-send-admission-tests tests/send_admission.cpp)
    target_compile_features(enet-send-admission-tests PRIVATE cxx_std_11)
    if(TARGET GTest::gtest_main)
        target_link_libraries(enet-send-admission-tests PRIVATE enet GTest::gtest_main)
    else()
        target_link_libraries(enet-send-admission-tests PRIVATE enet gtest_main)
    endif()
    add_test(NAME enet-send-admission-tests COMMAND enet-send-admission-tests)
endif()

if(NOT ENET_NO_INSTALL)
    install(TARGETS enet
        RUNTIME DESTINATION bin
        ARCHIVE DESTINATION lib/static
        LIBRARY DESTINATION lib)

    install(DIRECTORY include/
            DESTINATION include)
endif()
