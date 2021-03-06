cmake_minimum_required(VERSION 2.8)
set(CMAKE_LEGACY_CYGWIN_WIN32 0)

set(CMAKE_ECLIPSE_VERSION 4.4)

project("Thunderbolt")

#define modules path
list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake/Modules)
list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)

SET(EXECUTABLE_OUTPUT_PATH ${PROJECT_BINARY_DIR}/bin)

set(DRIVER_SHARED_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../../shared"
    CACHE PATH "The path to the shared code with driver")

set(SERVICE_SHARED_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../Shared"
    CACHE PATH "The path to the shared code with windows")

set(NETLINK_SOCKETS_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../../External/netLink"
    CACHE PATH "The path to the netlink sockets c++ library")

set(BOOST_INLCUDE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../../External/boost"
    CACHE PATH "The path to the boost headers")

enable_testing()

#include modules
include(poject_macros)
include(gmock)
find_package(Libnl REQUIRED)
find_package(LibDBus-c++-1 REQUIRED)
find_package(DBus REQUIRED)

if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU" OR
    "${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
    set(warnings "-Wall -Wextra")
    set(cpp11_flags "-std=c++11")
elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
    set(warnings "/W4 /WX /EHsc")
    set(cpp11_flags "")
endif()

if (NOT CONFIGURED_ONCE)
    set(CMAKE_CXX_FLAGS "${warnings} ${cpp11_flags}"
        CACHE STRING "Flags used by the compiler during all build types." FORCE)
    set(CMAKE_C_FLAGS   "${warnings}"
        CACHE STRING "Flags used by the compiler during all build types." FORCE)
endif()

#-------------------------= main application =--------------------------

include_directories(${CMAKE_CURRENT_BINARY_DIR}
                    ${SERVICE_SHARED_DIR}/include
                    ${DRIVER_SHARED_DIR}
                    ${CMAKE_CURRENT_SOURCE_DIR}/include
                    ${BOOST_INLCUDE_DIR}
                    ${LIBNL3_INCLUDE_DIRS}
                    ${LIBDBUS-C++-1_INCLUDE_DIRS})

#add here all subdirectories that need to be compiled
aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR}/src LIB_SRC_FILES)
aux_source_directory(${SERVICE_SHARED_DIR}/src SERVICE_SHARED_SRC_FILES)
aux_source_directory(${SERVICE_SHARED_DIR}/src/XDomainProperties SERVICE_SHARED_XDOMAIN_SRC_FILES)
aux_source_directory(${NETLINK_SOCKETS_DIR}/src NETLINK_SOCKETS_SRC_FILES)
add_library(lib OBJECT ${LIB_SRC_FILES}
                       ${SERVICE_SHARED_SRC_FILES}
                       ${NETLINK_SOCKETS_SRC_FILES}
                       ${SERVICE_SHARED_XDOMAIN_SRC_FILES})

add_executable(thunderboltd thunderbolt.cpp $<TARGET_OBJECTS:lib>)
add_subdirectory(tests)
target_link_libraries(thunderboltd
                      ${LIBDBUS-C++-1_LIBRARIES}
                      ${LIBNL3_LIBRARIES}
                      -lstdc++
                      -lpthread)

#Install section
set(CMAKE_INSTALL_PREFIX "/usr")
if(NOT DEFINED CMAKE_INSTALL_LIBDIR)
    set(CMAKE_INSTALL_LIBDIR lib64)
endif()

set(DBUS_CONFIG_DIR "/etc/dbus-1/system.d/")
set(CONFIG_DIR ${CMAKE_CURRENT_SOURCE_DIR}/config)
set(UDEV_RULES_DIR "/usr/lib/udev/rules.d")
set(DAEMON_INSTALL_DIR "/usr/lib/thunderbolt")

find_package(PkgConfig)

#pkg_check_modules(SYSTEMD QUIET "systemd")
#pkg_check_modules(DBUS QUIET "dbus-1")
pkg_check_modules(DBUS "dbus-1")
message(STATUS "Finding dbus")
#if (SYSTEMD_FOUND AND "${SYSTEMD_SERVICES_INSTALL_DIR}" STREQUAL "")
#    execute_process(COMMAND ${PKG_CONFIG_EXECUTABLE}
#                    --variable=systemdsystemunitdir systemd
#                    OUTPUT_VARIABLE SYSTEMD_SERVICES_INSTALL_DIR)
#    #remove space, tabs and newline and place output on DBUS_SERVICES_INSTALL_DIR variable
#    string(REGEX REPLACE "[ \t\n]+" "" SYSTEMD_SERVICES_INSTALL_DIR
#           "${SYSTEMD_SERVICES_INSTALL_DIR}")
#elseif (NOT SYSTEMD_FOUND AND SYSTEMD_SERVICES_INSTALL_DIR)
#    message(FATAL_ERROR
#            "Variable SYSTEMD_SERVICES_INSTALL_DIR is defined, but we can't \
#            find systemd using pkg-config")
#endif()

if (DBUS_FOUND AND "${DBUS_SERVICES_INSTALL_DIR}" STREQUAL "")
    execute_process(COMMAND ${PKG_CONFIG_EXECUTABLE}
                    --variable=system_bus_services_dir dbus-1
                    OUTPUT_VARIABLE DBUS_SERVICES_INSTALL_DIR)
    #remove space, tabs and newline and place output on DBUS_SERVICES_INSTALL_DIR variable
    string(REGEX REPLACE "[ \t\n]+" "" DBUS_SERVICES_INSTALL_DIR
           "${DBUS_SERVICES_INSTALL_DIR}")
elseif (NOT DBUS_FOUND AND DBUS_SERVICES_INSTALL_DIR)
    message(FATAL_ERROR
            "Variable DBUS_SERVICES_INSTALL_DIR is defined, but we can't find \
            dbus using pkg-config")
endif()

#if (SYSTEMD_FOUND)
#    message(STATUS "SYSTEMD_SERVICES_INSTALL_DIR: ${SYSTEMD_SERVICES_INSTALL_DIR}")
#endif()

if (DBUS_FOUND)
    message(STATUS "DBUS_SERVICES_INSTALL_DIR: ${DBUS_SERVICES_INSTALL_DIR}")
endif()


if(DBUS_FOUND)
	#install(FILES ${CONFIG_DIR}/thunderbolt.service DESTINATION ${SYSTEMD_SERVICES_INSTALL_DIR})
    install(FILES ${CONFIG_DIR}/com.Intel.Thunderbolt.service DESTINATION ${DBUS_SERVICES_INSTALL_DIR})
    install(FILES ${CONFIG_DIR}/thunderbolt.conf DESTINATION ${DBUS_CONFIG_DIR})
    install(FILES ${CONFIG_DIR}/10-thunderbolt.rules DESTINATION ${UDEV_RULES_DIR})
    install(TARGETS thunderboltd DESTINATION ${DAEMON_INSTALL_DIR})
    install(SCRIPT post_install_script.cmake)
else()
    message(WARNING
            "Can't install Thunderbolt daemon because dbus couldn't be found")
endif()

set(CPACK_PACKAGE_VERSION ${VERSION})
set(CPACK_GENERATOR "RPM")
set(CPACK_PACKAGE_NAME "Thunderbolt SW")
set(CPACK_PACKAGE_RELEASE 1)
set(CPACK_PACKAGE_CONTACT "Ofir Benizri")
set(CPACK_PACKAGE_VENDOR "Intel")
set(CPACK_PACKAGING_INSTALL_PREFIX ${CMAKE_INSTALL_PREFIX})
set(CPACK_PACKAGE_FILE_NAME
    "${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}-${CPACK_PACKAGE_RELEASE}.${CMAKE_SYSTEM_PROCESSOR}")
include(CPack)

set(CONFIGURED_ONCE TRUE CACHE INTERNAL
    "A flag showing that CMake has configured at least once.")
