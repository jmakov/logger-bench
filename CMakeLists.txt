cmake_minimum_required(VERSION 3.5.1)
include (ExternalProject)
include(ExternalData)

set (VERSION_MAJOR 1)
set (VERSION_MINOR 0)
set (VERSION_REV 0)
set (VERSION_FULL "${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_REV}")

project(logger_bench VERSION ${VERSION_FULL})

set(SOURCES ${CMAKE_SOURCE_DIR}/src)
set(LOGGER_INSTALL ${CMAKE_BINARY_DIR}/install)

option(MALC "mini async log C enabled" on)
option(MAL "mini async log enabled" off)
option(NANOLOG "NanoLog enabled" off)
option(SPDLOG "spdlog enabled" off)
option(GLOG "Google log enabled" off)
option(G3LOG "G3 log enabled" off)

#-- Own Binary -----------------------------------------------------------------

add_executable(
    logger-bench
    ${SOURCES}/main.cpp
    )

target_include_directories(
    logger-bench PRIVATE
    ${SOURCES}
    ${LOGGER_INSTALL}/include/
    )

#-- Build system specific things -----------------------------------------------

execute_process(
    COMMAND         ${CMAKE_CXX_COMPILER} -dumpmachine
    OUTPUT_VARIABLE ARCH_TRIPLET
    )
string (STRIP ${ARCH_TRIPLET} ARCH_TRIPLET)

set (CMAKE_CXX_STANDARD 11)

#-- MALC -----------------------------------------------------------------------

if (MALC)
    #PATCH_COMMAND shouldn't be necessary if CMake cloned with --recursive
    ExternalProject_Add(
        malc_artifacts
        GIT_REPOSITORY    https://github.com/RafaGago/mini-async-log-c.git
        GIT_TAG           ba721eb9915577a361205cc3bfbea5c04ec55247
        PATCH_COMMAND     git submodule update --init --recursive
        CONFIGURE_COMMAND
            meson malc_build --prefix=/ --buildtype=release -Dno_compressed_builtins=true -Dno_compressed_ptrs=true
        UPDATE_COMMAND    ""
        BUILD_IN_SOURCE   1
        BUILD_COMMAND     ninja -C malc_build
        INSTALL_COMMAND   DESTDIR=${LOGGER_INSTALL} ninja -C malc_build install
        )
    add_library (malc STATIC IMPORTED)
    set_property(
        TARGET                     malc
        PROPERTY IMPORTED_LOCATION ${LOGGER_INSTALL}/lib/${ARCH_TRIPLET}/libmalc.a
        )
    add_dependencies(malc malc_artifacts)

    add_library (bl_base STATIC IMPORTED)
    set_property(
        TARGET                     bl_base
        PROPERTY IMPORTED_LOCATION ${LOGGER_INSTALL}/lib/${ARCH_TRIPLET}/libbl-base.a
        )
    add_dependencies (bl_base malc_artifacts)

    add_library (bl_nonblock STATIC IMPORTED)
    set_property(
        TARGET                     bl_nonblock
        PROPERTY IMPORTED_LOCATION ${LOGGER_INSTALL}/lib/${ARCH_TRIPLET}/libbl-nonblock.a
        )
    add_dependencies (bl_nonblock malc_artifacts bl_nonblock bl_base)

    target_link_libraries (logger-bench malc)
    target_compile_definitions (logger-bench PRIVATE HAS_MALC=1)
    target_sources (logger-bench PRIVATE ${SOURCES}/malc.cpp)

endif (MALC)

#-- MAL ------------------------------------------------------------------------

if (MAL)
    ExternalProject_Add(
        mal_artifacts
        GIT_REPOSITORY https://github.com/RafaGago/mini-async-log.git
        GIT_TAG        d239c085e8af03180fbd1bf61181a3c7c96f8a51
        UPDATE_COMMAND ""
        INSTALL_DIR    ${LOGGER_INSTALL}
        CMAKE_ARGS     -DCMAKE_INSTALL_PREFIX=${LOGGER_INSTALL} -DBUILD_SHARED_LIBS=off
        )
    add_library (mal STATIC IMPORTED)
    set_property(
        TARGET                     mal
        PROPERTY IMPORTED_LOCATION ${LOGGER_INSTALL}/lib/libmini_async_log.a
        )
    add_dependencies (mal mal_artifacts)

    target_link_libraries (logger-bench mal)
    target_compile_definitions (logger-bench PRIVATE HAS_MAL=1)
    target_sources (logger-bench PRIVATE ${SOURCES}/mal.cpp)

endif (MAL)

#-- Nanolog --------------------------------------------------------------------

if (NANOLOG)
    set (NANOLOG_REPO ${CMAKE_BINARY_DIR}/nanolog-git)

    ExternalProject_Add(
        nanolog_clone
        GIT_REPOSITORY    https://github.com/Iyengar111/NanoLog.git
        GIT_TAG           40a53c36e0336af45f7664abeb939f220f78273e
        SOURCE_DIR        ${NANOLOG_REPO}
        UPDATE_COMMAND    ""
        CONFIGURE_COMMAND ""
        BUILD_COMMAND     ""
        INSTALL_COMMAND   ""
        )
    set_property (SOURCE ${NANOLOG_REPO}/NanoLog.cpp APPEND PROPERTY GENERATED 1)
    add_library (nanolog STATIC ${NANOLOG_REPO}/NanoLog.cpp)
    target_include_directories (nanolog PRIVATE ${NANOLOG_REPO})
    add_dependencies (nanolog nanolog_clone)

    target_link_libraries (logger-bench nanolog)
    target_compile_definitions (logger-bench PRIVATE HAS_NANOLOG=1)
    target_include_directories (logger-bench PRIVATE ${NANOLOG_REPO})
    target_sources (logger-bench PRIVATE ${SOURCES}/nanolog.cpp)

endif (NANOLOG)

#-- spdlog ---------------------------------------------------------------------

if (SPDLOG)
    set (SPDLOG_REPO ${CMAKE_BINARY_DIR}/spdlog-git)

    ExternalProject_Add(
        spdlog_clone
        GIT_REPOSITORY    https://github.com/gabime/spdlog.git
        GIT_TAG           e6b9fa577de852fd7b78a0a3d2837875e4425990
        SOURCE_DIR        ${SPDLOG_REPO}
        UPDATE_COMMAND    ""
        CONFIGURE_COMMAND ""
        BUILD_COMMAND     ""
        INSTALL_COMMAND   ""
        )
    add_dependencies (logger-bench spdlog_clone)
    target_compile_definitions (logger-bench PRIVATE HAS_SPDLOG=1)
    target_include_directories (logger-bench PRIVATE ${SPDLOG_REPO}/include)
    target_sources (logger-bench PRIVATE ${SOURCES}/spdlog.cpp)

endif (SPDLOG)

#-- glog -----------------------------------------------------------------------

if (GLOG)
    ExternalProject_Add(
        glog_artifacts
        GIT_REPOSITORY https://github.com/google/glog.git
        GIT_TAG        2063b387080c1e7adffd33ca07adff0eb346ff1a
        UPDATE_COMMAND ""
        INSTALL_DIR    ${LOGGER_INSTALL}
        CMAKE_ARGS     -DCMAKE_INSTALL_PREFIX=${LOGGER_INSTALL} -DBUILD_SHARED_LIBS=off
        )
    add_library (glog STATIC IMPORTED)
    set_property(
        TARGET                     glog
        PROPERTY IMPORTED_LOCATION ${LOGGER_INSTALL}/lib/libglog.a
        )
    add_dependencies (glog glog_artifacts)

    target_link_libraries (logger-bench glog)
    target_compile_definitions (logger-bench PRIVATE HAS_GLOG=1)
    target_sources (logger-bench PRIVATE ${SOURCES}/glog.cpp)

endif (GLOG)

#-- g3log ----------------------------------------------------------------------

if (G3LOG)
    ExternalProject_Add(
        g3log_artifacts
        GIT_REPOSITORY https://github.com/KjellKod/g3log.git
        GIT_TAG        0ddfd6dccc86818fa0e48c1e311c41d0120dec3b
        UPDATE_COMMAND ""
        INSTALL_DIR    ${LOGGER_INSTALL}
        CMAKE_ARGS     -DCMAKE_INSTALL_PREFIX=${LOGGER_INSTALL} -DCPACK_INSTALL_PREFIX=${LOGGER_INSTALL} -DG3_SHARED_LIB=off
        )
    add_library (g3log STATIC IMPORTED)
    set_property(
        TARGET                     g3log
        PROPERTY IMPORTED_LOCATION ${LOGGER_INSTALL}/lib/libg3logger.a
        )
    add_dependencies (g3log g3log_artifacts)

    target_link_libraries (logger-bench g3log)
    target_compile_definitions (logger-bench PRIVATE HAS_G3LOG=1)
    target_sources (logger-bench PRIVATE ${SOURCES}/g3log.cpp)

endif (G3LOG)