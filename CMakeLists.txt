cmake_minimum_required(VERSION 3.16)

# include/LibLsp/lsp/Markup/string_ref.h:1015 required _stricmp, which is non-standard.
#   https://stackoverflow.com/a/6312370
if(CMAKE_CXX_COMPILER STREQUAL "GNU")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=gnu++17")
endif()

if(POLICY CMP0025)
    # detect Apple's Clang
    cmake_policy(SET CMP0025 NEW)
endif()
if(POLICY CMP0054)
    cmake_policy(SET CMP0054 NEW)
endif()

add_compile_options(-Wno-maybe-uninitialized -Wno-unknown-warning-option -Wno-enum-constexpr-conversion -Wno-c++11-narrowing)

if(WIN32 AND CROSS_COMPILING_ON_LINUX)
  # https://stackoverflow.com/questions/5411282/cross-compiling-boost-for-windows-on-linux
  # https://stackoverflow.com/a/60238549
  # https://stackoverflow.com/a/11907834
  # https://github.com/jw3/example-cmake-boost-cross
  # https://gitlab.com/lkkmpn/boost-mingw
  # https://gitlab.com/lkkmpn/boost-mingw/-/blob/main/.gitlab-ci.yml?ref_type=heads#L73 <-
  add_custom_target(boost
    test -d boost ||
    (cp -r ${CMAKE_CURRENT_SOURCE_DIR}/boost_1_78_0 boost &&
    cd boost &&
    # Скрипты b2 (системы сборки boost), видимо, ожидают, что компилятор mingw называется
    #   тоже g++... И оттуда определяет, что за toolchain, mingw ли. Чтож, дадим mingw.
    #   Ппц.. Не смотря на все инструкции с toolset=gcc-mingw, это просто не работает..
    #   Даже это написано в оригинальной документации, под эту же версию.
    #   https://www.boost.org/doc/libs/1_78_0/tools/build/doc/html/index.html#bbv2.tasks.crosscompile
    #   Почитай исходный код, называется. :(
    ln -s /usr/bin/x86_64-w64-mingw32-g++ g++ &&
    ./bootstrap.sh &&
    export PATH=$$(pwd):"$$PATH" &&
    ./b2
      --with-chrono
      --with-date_time
      --with-filesystem
      --with-program_options
      --with-system
      --with-thread
      target-os=windows
      address-model=64
      variant=release
      link=static
      runtime-link=static)
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
  )
else()
  add_custom_target(boost
    test -d boost ||
    (cp -r ${CMAKE_CURRENT_SOURCE_DIR}/boost_1_78_0 boost &&
    cd boost &&
    ./bootstrap.sh &&
    ./b2
      --with-chrono
      --with-date_time
      --with-filesystem
      --with-program_options
      --with-system
      --with-thread
      link=static
      runtime-link=static)
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
  )
endif()

# Fix too many sections error.
#   https://stackoverflow.com/a/62281145
#   https://stackoverflow.com/a/64287929
if(WIN32)
  add_compile_options(-m64 -Wa,-mbig-obj)
  add_link_options(-m64 -Wa,-mbig-obj)
endif()

message("CMAKE_BUILD_TYPE = ${CMAKE_BUILD_TYPE}")
if(NOT "$ENV{BUILD_SANITIZERS}" STREQUAL "1")
    set(BUILD_SHARED_LIBS OFF) # So that it is portable across linuxes.
    if(WIN32 AND MSVC)
        set(CMAKE_FIND_LIBRARY_SUFFIXES .lib)
    else()
        set(CMAKE_FIND_LIBRARY_SUFFIXES .a)
        add_compile_options(-static)
        add_link_options(-static)
    endif()
else()
    add_compile_options(-fsanitize=address,undefined)
    add_link_options(-fsanitize=address,undefined)
endif()


set(LIB_MAJOR_VERSION "1")
set(LIB_MINOR_VERSION "0")
set(LIB_PATCH_VERSION "0")
set(LIB_VERSION_STRING "${LIB_MAJOR_VERSION}.${LIB_MINOR_VERSION}.${LIB_PATCH_VERSION}")

# Without this, paths are not relative in the sources list
cmake_policy(SET CMP0076 NEW)
project(lspcpp VERSION "${LIB_VERSION_STRING}" LANGUAGES CXX C)

SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
SET(GOOGLETEST_VERSION "0.00")

# compile in RelWithDebInfo  mode by default
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "RelWithDebInfo" CACHE STRING "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif()

###########################################################
# Options
###########################################################
function (option_if_not_defined name description default)
    if(NOT DEFINED ${name})
        option(${name} ${description} ${default})
    endif()
endfunction()

option_if_not_defined(USE_SYSTEM_RAPIDJSON "Use system RapidJSON instead of the git submodule if exists" OFF)
option_if_not_defined(LSPCPP_WARNINGS_AS_ERRORS "Treat warnings as errors" OFF)
option_if_not_defined(LSPCPP_BUILD_EXAMPLES "Build example applications" OFF)
option_if_not_defined(LSPCPP_BUILD_FUZZER "Build fuzzer" OFF)
option_if_not_defined(LSPCPP_BUILD_WEBSOCKETS "Build websocket server" ON)
option_if_not_defined(LSPCPP_ASAN "Build lsp with address sanitizer" OFF)
option_if_not_defined(LSPCPP_MSAN "Build lsp with memory sanitizer" OFF)
option_if_not_defined(LSPCPP_TSAN "Build lsp with thread sanitizer" OFF)
option_if_not_defined(LSPCPP_INSTALL "Create lsp install target" OFF)
option_if_not_defined(LSPCPP_SUPPORT_BOEHM_GC "\
Enable support for Boehm GC. GC must be specified LSPCPP_GC_DOWNLOADED_ROOT, if downloaded or by
find_package(BDWgc CONFIG REQUIRED) or pkg_config." OFF)
option_if_not_defined(LSPCPP_USE_CPP17 "Use C++17 for compilation. Setting this to off requires boost-optional." OFF)

set(LSPCPP_GC_DOWNLOADED_ROOT "" CACHE STRING "\
If using downloaded gc root from https://www.hboehm.info/gc/, extract its contents and set this setting to
where the tar.gz file is extracted to. This directory must be an absolute path.
If this setting is set, LspCpp will use downloaded GC regardless of whether GC from find_package or
pkg_config is available or not.
")
option_if_not_defined(LSPCPP_GC_STATIC "Compiling with static gc library. Only used if a custom GC root is given" OFF)
set(LSPCPP_WIN32_WINNT_VALUE "0x0A00" CACHE STRING
        "Value to specify for _WIN32_WINNT macro when compiling on windows. See
https://learn.microsoft.com/en-us/cpp/porting/modifying-winver-and-win32-winnt?view=msvc-170"
)

###########################################################
# Boehm GC
###########################################################

if (LSPCPP_SUPPORT_BOEHM_GC AND NOT LSPCPP_GC_DOWNLOADED_ROOT)
    message(STATUS "Attempting to find BDWgc by find_package")
    find_package(BDWgc CONFIG)
    if (NOT BDWgc_FOUND)
        set(GC_USE_PKGCONFIG 1)
        # fallback to pkg-config
        message(STATUS "CMake config for BDWgc not found; falling back to pkg-config")
        find_package(PkgConfig REQUIRED)
        pkg_check_modules(gc REQUIRED IMPORTED_TARGET bdw-gc)
    endif()
endif()

###########################################################
# Directories
###########################################################
function (set_if_not_defined name value)
    if(NOT DEFINED ${name})
        set(${name} ${value} PARENT_SCOPE)
    endif()
endfunction()

set(LSPCPP_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/include)
set_if_not_defined(LSPCPP_THIRD_PARTY_DIR ${CMAKE_CURRENT_SOURCE_DIR}/third_party)

macro(lspcpp_set_target_options_with_nuget_pkg target id version)
    if (CMAKE_GENERATOR MATCHES "Visual Studio.*")
        if(EXISTS ${CMAKE_BINARY_DIR}/packages/${id}.${version}/build/${id}.targets)
            target_link_libraries(${target} PRIVATE ${CMAKE_BINARY_DIR}/packages/${id}.${version}/build/${id}.targets)
        elseif(EXISTS ${CMAKE_BINARY_DIR}/packages/${id}.${version}/build/native/${id}.targets)
            target_link_libraries(${target} PRIVATE ${CMAKE_BINARY_DIR}/packages/${id}.${version}/build/native/${id}.targets)
        else()
            message(FATAL_ERROR "Can't find target of ${id}.${version}")
        endif()
    else()
        message(FATAL_ERROR "NUGET package only use in Visual Studio")
    endif()

endmacro()

macro(INSTALL_NUGET id version)
    if (CMAKE_GENERATOR MATCHES "Visual Studio.*")
        unset(nuget_cmd)
        list(APPEND nuget_cmd install ${id} -Prerelease -Version ${version} -OutputDirectory ${CMAKE_BINARY_DIR}/packages)
        message("excute nuget install:${nuget_cmd}")
        execute_process(COMMAND nuget ${nuget_cmd} ENCODING AUTO)
    else()
        message(FATAL_ERROR "INSTALL_NUGET only use in Visual Studio")
    endif()

endmacro()
###########################################################
# Functions
###########################################################
function(lspcpp_set_target_options target)

    set_property(TARGET ${target} PROPERTY CXX_STANDARD_REQUIRED ON)

    # Enable C++14/17 (Required)
    if (LSPCPP_USE_CPP17)
        set_property(TARGET ${target} PROPERTY CXX_STANDARD 17)
    else()
        set_property(TARGET ${target} PROPERTY CXX_STANDARD 14)
    endif()

    set_property(TARGET ${target} PROPERTY CXX_EXTENSIONS OFF)

    if (CMAKE_GENERATOR MATCHES "Visual Studio.*")
        lspcpp_set_target_options_with_nuget_pkg(${target} boost 1.76.0.0)
        lspcpp_set_target_options_with_nuget_pkg(${target} boost_chrono-vc141 1.76.0.0)
        lspcpp_set_target_options_with_nuget_pkg(${target} boost_date_time-vc141 1.76.0.0)
        lspcpp_set_target_options_with_nuget_pkg(${target} boost_filesystem-vc141 1.76.0.0)
        lspcpp_set_target_options_with_nuget_pkg(${target} boost_program_options-vc141 1.76.0.0)
        lspcpp_set_target_options_with_nuget_pkg(${target} boost_system-vc141 1.76.0.0)
        lspcpp_set_target_options_with_nuget_pkg(${target} boost_thread-vc141 1.76.0.0)
    endif()

    # Enable all warnings
    if(MSVC)
        target_compile_options(${target} PRIVATE "-W4")
    else()
        target_compile_options(${target} PRIVATE "-Wall")
    endif()

    # Disable specific, pedantic warnings
    if(MSVC)
        target_compile_options(${target} PRIVATE
                "-D_CRT_SECURE_NO_WARNINGS"

                # Warnings from nlohmann/json headers.
                "/wd4267" # 'argument': conversion from 'size_t' to 'int', possible loss of data
                "/bigobj" # for visual studio 2022 x64 or later.
                )
    endif()

    # Add define for JSON library in use
    set_target_properties(${target} PROPERTIES
            COMPILE_DEFINITIONS "LSPCPP_JSON_${LSPCPP_JSON_LIBRARY_UPPER}=1"
            )

    # Treat all warnings as errors
    if(LSPCPP_WARNINGS_AS_ERRORS)
        if(MSVC)
            target_compile_options(${target} PRIVATE "/WX")
        else()
            target_compile_options(${target} PRIVATE "-Werror")
        endif()
    endif(LSPCPP_WARNINGS_AS_ERRORS)

    if(LSPCPP_ASAN)
        target_compile_options(${target} PUBLIC "-fsanitize=address")
        target_link_libraries(${target} PUBLIC "-fsanitize=address")
    elseif(LSPCPP_MSAN)
        target_compile_options(${target} PUBLIC "-fsanitize=memory")
        target_link_libraries(${target} PUBLIC "-fsanitize=memory")
    elseif(LSPCPP_TSAN)
        target_compile_options(${target} PUBLIC "-fsanitize=thread")
        target_link_libraries(${target} PUBLIC "-fsanitize=thread")
    endif()

    # Error on undefined symbols
    # if(NOT MSVC)
    #     target_compile_options(${target} PRIVATE "-Wl,--no-undefined")
    # endif()

    if (LSPCPP_SUPPORT_BOEHM_GC)
        if (LSPCPP_GC_DOWNLOADED_ROOT)
            message(STATUS "Using manually downloaded GC")
            target_include_directories(${target} PUBLIC ${LSPCPP_GC_DOWNLOADED_ROOT}/include)

            if (LSPCPP_GC_STATIC)
                target_compile_definitions(${target} PUBLIC GC_NOT_DLL)
            endif()
        else()
            if (NOT GC_USE_PKGCONFIG)
                message(STATUS "Using cmake config for locating gc")
                target_link_libraries(${target} PUBLIC BDWgc::gc)
                get_target_property(GC_INCLUDE_DIRS_BASE BDWgc::gc INTERFACE_INCLUDE_DIRECTORIES)
            else()
                message(STATUS "Using pkg-config for locating gc")
                target_link_libraries(${target} PUBLIC PkgConfig::gc)
                get_target_property(GC_INCLUDE_DIRS_BASE PkgConfig::gc INTERFACE_INCLUDE_DIRECTORIES)
            endif()

            if (NOT GC_INCLUDE_DIRS_BASE STREQUAL "GC_INCLUDE_DIRS_BASE-NOTFOUND")
                list(TRANSFORM GC_INCLUDE_DIRS_BASE APPEND /gc)
                target_include_directories(${target} PUBLIC ${GC_INCLUDE_DIRS_BASE})
            else()
                message(WARNING "Cannot find GC include directories; compilation may not work")
            endif()
        endif()
        target_compile_definitions(${target} PUBLIC LSPCPP_USEGC)
    endif()

    if (WIN32)
        target_compile_definitions(${target} PRIVATE _WIN32_WINNT=${LSPCPP_WIN32_WINNT_VALUE})
    endif()
endfunction()


# Libraries

set(Uri_USE_STATIC_CRT ON)
set(Uri_BUILD_TESTS OFF)
add_subdirectory(third_party/uri)

###########################################################
# boost library
###########################################################
if (CMAKE_GENERATOR MATCHES "Visual Studio.*")
    INSTALL_NUGET(boost 1.76.0.0)
    INSTALL_NUGET(boost_chrono-vc141 1.76.0.0)
    INSTALL_NUGET(boost_date_time-vc141 1.76.0.0)
    INSTALL_NUGET(boost_filesystem-vc141 1.76.0.0)
    INSTALL_NUGET(boost_program_options-vc141 1.76.0.0)
    INSTALL_NUGET(boost_system-vc141 1.76.0.0)
    INSTALL_NUGET(boost_thread-vc141 1.76.0.0)
else()

    #find_package(Boost CONFIG COMPONENTS date_time  chrono filesystem system thread program_options)
    #if(NOT Boost_FOUND)
    #    if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    #        message(FATAL_ERROR "Can't find boost,lease build boost and install it or install boost with : brew install boost")
    #    elseif(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    #        message(FATAL_ERROR "Can't find boost,please build boost and install it. or install boost with : sudo apt-get install libboost-dev")
    #    else()
    #        message(FATAL_ERROR "Boost not found. Please ensure boost is available for CMake.")
    #    endif()
    #endif()
endif()


###########################################################
# JSON library
###########################################################
if(USE_SYSTEM_RAPIDJSON)
    set(RapidJSON_MIN_VERSION "1.1.0")
    find_package(RapidJSON ${RapidJSON_MIN_VERSION} QUIET)
    if(NOT DEFINED RapidJSON_INCLUDE_DIRS AND DEFINED RAPIDJSON_INCLUDE_DIRS)
        set(RapidJSON_INCLUDE_DIRS "${RAPIDJSON_INCLUDE_DIRS}")
    endif()
endif()
if(NOT RapidJSON_FOUND)
    if(EXISTS "${PROJECT_SOURCE_DIR}/third_party/rapidjson/include")
        message(STATUS "Using local RapidJSON")
        set(RapidJSON_INCLUDE_DIRS third_party/rapidjson/include)
    else()
        message(STATUS "Please initialize rapidJSON git submodule as currently installed version is to old:")
        message(STATUS "git submodule init && git submodule update")
        message(FATAL_ERROR "RapidJSON version is likely too old.")
    endif()
endif()


###########################################################
# Targets
###########################################################

# lsp
add_library(lspcpp STATIC)
add_dependencies(lspcpp boost)

### Includes
target_include_directories(lspcpp
        PUBLIC
            ${LSPCPP_INCLUDE_DIR}
            ${Boost_INCLUDE_DIRS}
            ${RapidJSON_INCLUDE_DIRS}
            ${Uri_SOURCE_DIR}/include
        )

target_include_directories(lspcpp PUBLIC ${CMAKE_CURRENT_BINARY_DIR}/boost)
target_link_directories(lspcpp PUBLIC ${CMAKE_CURRENT_BINARY_DIR}/boost/stage/lib)
target_link_libraries(lspcpp PUBLIC network-uri boost_filesystem)

set(LSPCPP_THIRD_PARTY_DIR_LIST
        ${LSPCPP_THIRD_PARTY_DIR}/utfcpp/source
        )

foreach(include_dir  ${LSPCPP_THIRD_PARTY_DIR_LIST})
    get_filename_component(include_dir_realpath ${include_dir} REALPATH)
    # Don't add as SYSTEM if they are in CMAKE_CXX_IMPLICIT_INCLUDE_DIRECTORIES.
    # It would reorder the system search paths and cause issues with libstdc++'s
    # use of #include_next.
    if(NOT "${include_dir_realpath}" IN_LIST CMAKE_CXX_IMPLICIT_INCLUDE_DIRECTORIES)
        target_include_directories(lspcpp SYSTEM PRIVATE ${include_dir})
    endif()
endforeach()

# Fix wrong case, if cross-compiling for windows
set(mingw_sysroot /usr/x86_64-w64-mingw32)
if(WIN32 AND EXISTS ${mingw_sysroot}/include/windows.h)
  add_custom_target(fix_header_names_casing
    [ -d fix_name_casing_for_headers ] || (
      mkdir fix_name_casing_for_headers &&
      cp ${mingw_sysroot}/include/windows.h fix_name_casing_for_headers/Windows.h
    )
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
  )
  target_include_directories(lspcpp PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/fix_name_casing_for_headers)
  add_dependencies(lspcpp fix_header_names_casing)
endif()

# Fix compilation with mingw. Undefined references to bcrypt functions.
if(WIN32 AND CROSS_COMPILING_ON_LINUX)
  target_link_libraries(lspcpp PRIVATE bcrypt)
endif()

### Sources
set(JSONRPC_LIST
        src/jsonrpc/Context.cpp
        src/jsonrpc/Endpoint.cpp
        src/jsonrpc/GCThreadContext.cpp
        src/jsonrpc/message.cpp
        src/jsonrpc/MessageJsonHandler.cpp
        src/jsonrpc/RemoteEndPoint.cpp
        src/jsonrpc/serializer.cpp
        src/jsonrpc/StreamMessageProducer.cpp
        src/jsonrpc/TcpServer.cpp
        src/jsonrpc/threaded_queue.cpp
)
set(LSPCPP_LIST
        src/lsp/initialize.cpp
        src/lsp/lsp.cpp
        src/lsp/lsp_diagnostic.cpp
        src/lsp/Markup.cpp
        src/lsp/ParentProcessWatcher.cpp
        src/lsp/ProtocolJsonHandler.cpp
        src/lsp/textDocument.cpp
        src/lsp/utils.cpp
        src/lsp/working_files.cpp
        )

if(LSPCPP_BUILD_WEBSOCKETS)
    set(JSONRPC_LIST
        ${JSONRPC_LIST}
        src/jsonrpc/WebSocketServer.cpp
    )
endif()

target_sources(lspcpp PRIVATE
        ${JSONRPC_LIST}
        ${LSPCPP_LIST})

### Compile options

lspcpp_set_target_options(lspcpp)

set_target_properties(lspcpp PROPERTIES POSITION_INDEPENDENT_CODE 1)

# install
if(LSPCPP_INSTALL)
    include(GNUInstallDirs)

    install(DIRECTORY ${LSPCPP_INCLUDE_DIR}/LibLsp
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
            USE_SOURCE_PERMISSIONS
            )

    install(TARGETS lspcpp
            EXPORT lspcpp-targets
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
            INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
            )

    install(EXPORT lspcpp-targets
            FILE lspcpp-config.cmake
            NAMESPACE lspcpp::
            DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/lspcpp
            )
endif()

# examples
if(LSPCPP_BUILD_EXAMPLES)

    ###########################################################
    # OS libraries
    ###########################################################
    if(CMAKE_SYSTEM_NAME MATCHES "Windows")
        set(LSPCPP_OS_LIBS WS2_32)
    elseif(CMAKE_SYSTEM_NAME MATCHES "Linux")
        set(LSPCPP_OS_LIBS pthread)
    elseif(CMAKE_SYSTEM_NAME MATCHES "Darwin")
        set(LSPCPP_OS_LIBS)
    endif()

    function(build_example target)
        add_executable(${target} "${CMAKE_CURRENT_SOURCE_DIR}/examples/${target}.cpp")
        target_include_directories(${target} PRIVATE ${Uri_SOURCE_DIR}/include)
        set_target_properties(${target} PROPERTIES
                FOLDER "Examples"
                )
        lspcpp_set_target_options(${target})
        target_link_libraries(${target}  PRIVATE lspcpp "${LSPCPP_OS_LIBS}")
    endfunction(build_example)

    set(EXAMPLES
            StdIOClientExample
            StdIOServerExample
            TcpServerExample
            WebsocketExample
            )

    foreach (example ${EXAMPLES})
        build_example(${example})
    endforeach()
endif()

# Add a distclean target to the Makefile
ADD_CUSTOM_TARGET(distclean COMMAND ${CMAKE_COMMAND} -P ${CMAKE_SOURCE_DIR}/distclean.cmake)
