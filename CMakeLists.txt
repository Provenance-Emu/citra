# CMake 3.12 required for 20 to be a valid value for CXX_STANDARD
cmake_minimum_required(VERSION 3.15)

# Don't override the warning flags in MSVC:
cmake_policy(SET CMP0092 NEW)
# Enforce new LTO setting
cmake_policy(SET CMP0069 NEW)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/CMakeModules")
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/externals/cmake-modules")
include(DownloadExternals)
include(CMakeDependentOption)

project(citra LANGUAGES C CXX ASM)

if (APPLE)
    enable_language(OBJC)
endif()

option(ENABLE_LTO "Enable link time optimization" OFF)

option(ENABLE_SDL2 "Enable the SDL2 frontend" ON)
option(USE_SYSTEM_SDL2 "Use the system SDL2 lib (instead of the bundled one)" OFF)

# Set bundled qt as dependent options.
option(ENABLE_QT "Enable the Qt frontend" ON)
option(ENABLE_QT_TRANSLATION "Enable translations for the Qt frontend" OFF)
CMAKE_DEPENDENT_OPTION(CITRA_USE_BUNDLED_QT "Download bundled Qt binaries" ON "ENABLE_QT;MSVC" OFF)

option(ENABLE_WEB_SERVICE "Enable web services (telemetry, etc.)" ON)

option(ENABLE_CUBEB "Enables the cubeb audio backend" ON)

option(ENABLE_FFMPEG_AUDIO_DECODER "Enable FFmpeg audio (AAC) decoder" OFF)
option(ENABLE_FFMPEG_VIDEO_DUMPER "Enable FFmpeg video dumper" OFF)

if (ENABLE_FFMPEG_AUDIO_DECODER OR ENABLE_FFMPEG_VIDEO_DUMPER)
    set(ENABLE_FFMPEG ON)
endif()

CMAKE_DEPENDENT_OPTION(CITRA_USE_BUNDLED_FFMPEG "Download bundled FFmpeg binaries" ON "ENABLE_FFMPEG;MSVC" OFF)

option(USE_DISCORD_PRESENCE "Enables Discord Rich Presence" OFF)

option(CITRA_USE_PRECOMPILED_HEADERS "Use precompiled headers" ON)

CMAKE_DEPENDENT_OPTION(ENABLE_MF "Use Media Foundation decoder (preferred over FFmpeg)" ON "WIN32" OFF)

CMAKE_DEPENDENT_OPTION(COMPILE_WITH_DWARF "Add DWARF debugging information" ON "MINGW" OFF)

option(USE_SYSTEM_BOOST "Use the system Boost libs (instead of the bundled ones)" OFF)

CMAKE_DEPENDENT_OPTION(ENABLE_FDK "Use FDK AAC decoder" OFF "NOT ENABLE_FFMPEG_AUDIO_DECODER;NOT ENABLE_MF" OFF)

if (CITRA_USE_PRECOMPILED_HEADERS)
    if (MSVC AND CCACHE)
        # buildcache does not properly cache PCH files, leading to compilation errors.
        # See https://github.com/mbitsnbites/buildcache/discussions/230
        message(WARNING "Buildcache does not properly support Precompiled Headers. Disabling PCH")
        set(CITRA_USE_PRECOMPILED_HEADERS OFF)
    endif()
    if(APPLE)
        message(WARNING "Precompiled Headers currently do not work on Apple. Disabling PCH")
        set(CITRA_USE_PRECOMPILED_HEADERS OFF)
    endif()
endif()
if (CITRA_USE_PRECOMPILED_HEADERS)
    message(STATUS "Using Precompiled Headers.")
    set(CMAKE_PCH_INSTANTIATE_TEMPLATES ON)
endif()

if(NOT EXISTS ${PROJECT_SOURCE_DIR}/.git/hooks/pre-commit)
    message(STATUS "Copying pre-commit hook")
    file(COPY hooks/pre-commit
        DESTINATION ${PROJECT_SOURCE_DIR}/.git/hooks)
endif()

# Use ccache for android if available
# =======================================================================
if (NOT $ENV{NDK_CCACHE} EQUAL "")
    set(CCACHE_EXE $ENV{NDK_CCACHE})
    set(CMAKE_C_COMPILER_LAUNCHER ${CCACHE_EXE})
    set(CMAKE_CXX_COMPILER_LAUNCHER ${CCACHE_EXE})
endif()

# Check for LTO support
# =======================================================================
if (ENABLE_LTO)
    include(CheckIPOSupported)
    check_ipo_supported(RESULT supported OUTPUT error_msg)

    if (supported)
        message(STATUS "LTO enabled")
    else()
        message(STATUS "LTO enabled but is unavailable, disabling: ${error_msg}")
        set(ENABLE_LTO OFF)
    endif()
else()
    message(STATUS "LTO disabled")
endif()

# Sanity check : Check that all submodules are present
# =======================================================================

function(check_submodules_present)
    file(READ "${PROJECT_SOURCE_DIR}/.gitmodules" gitmodules)
    string(REGEX MATCHALL "path *= *[^ \t\r\n]*" gitmodules ${gitmodules})
    foreach(module ${gitmodules})
        string(REGEX REPLACE "path *= *" "" module ${module})
        if (NOT EXISTS "${PROJECT_SOURCE_DIR}/${module}/.git")
            message(SEND_ERROR "Git submodule ${module} not found."
                    "Please run: git submodule update --init --recursive")
        endif()
    endforeach()
endfunction()
check_submodules_present()

configure_file(${PROJECT_SOURCE_DIR}/dist/compatibility_list/compatibility_list.qrc
               ${PROJECT_BINARY_DIR}/dist/compatibility_list/compatibility_list.qrc
               COPYONLY)
if (ENABLE_COMPATIBILITY_LIST_DOWNLOAD AND NOT EXISTS ${PROJECT_BINARY_DIR}/dist/compatibility_list/compatibility_list.json)
    message(STATUS "Downloading compatibility list for citra...")
    file(DOWNLOAD
        https://api.citra-emu.org/gamedb/
        "${PROJECT_BINARY_DIR}/dist/compatibility_list/compatibility_list.json" SHOW_PROGRESS)
endif()
if (EXISTS ${PROJECT_SOURCE_DIR}/dist/compatibility_list/compatibility_list.json)
    file(COPY "${PROJECT_SOURCE_DIR}/dist/compatibility_list/compatibility_list.json"
        DESTINATION "${PROJECT_BINARY_DIR}/dist/compatibility_list/")
endif()
if (NOT EXISTS ${PROJECT_BINARY_DIR}/dist/compatibility_list/compatibility_list.json)
    file(WRITE ${PROJECT_BINARY_DIR}/dist/compatibility_list/compatibility_list.json "")
endif()

# Detect current compilation architecture and create standard definitions
# =======================================================================

include(CheckSymbolExists)
function(detect_architecture symbol arch)
    if (NOT DEFINED ARCHITECTURE)
        set(CMAKE_REQUIRED_QUIET 1)
        check_symbol_exists("${symbol}" "" ARCHITECTURE_${arch})
        unset(CMAKE_REQUIRED_QUIET)

        # The output variable needs to be unique across invocations otherwise
        # CMake's crazy scope rules will keep it defined
        if (ARCHITECTURE_${arch})
            set(ARCHITECTURE "${arch}" PARENT_SCOPE)
        endif()
    endif()
endfunction()

if (NOT ENABLE_GENERIC)
    if (CMAKE_OSX_ARCHITECTURES)
        set(ARCHITECTURE "${CMAKE_OSX_ARCHITECTURES}")
    elseif (MSVC)
        detect_architecture("_M_AMD64" x86_64)
        detect_architecture("_M_IX86" x86)
        detect_architecture("_M_ARM" arm)
        detect_architecture("_M_ARM64" arm64)
    else()
        detect_architecture("__x86_64__" x86_64)
        detect_architecture("__i386__" x86)
        detect_architecture("__arm__" arm)
        detect_architecture("__aarch64__" arm64)
    endif()
endif()
if (NOT DEFINED ARCHITECTURE)
    set(ARCHITECTURE "GENERIC")
endif()
message(STATUS "Target architecture: ${ARCHITECTURE}")


# Configure C++ standard
# ===========================

# boost asio's concept usage doesn't play nicely with some compilers yet.
add_definitions(-DBOOST_ASIO_DISABLE_CONCEPTS)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# set up output paths for executable binaries
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin/$<CONFIG>)


# System imported libraries
# ======================

# Prefer the -pthread flag on Linux.
set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)

if (ENABLE_QT)
    if (CITRA_USE_BUNDLED_QT)
        if (MSVC_VERSION GREATER_EQUAL 1920 AND "x86_64" IN_LIST ARCHITECTURE)
            set(QT_VER qt-5.15.7-msvc2019_64)
        else()
            message(FATAL_ERROR "No bundled Qt binaries for your toolchain. Disable CITRA_USE_BUNDLED_QT and provide your own.")
        endif()

        if (DEFINED QT_VER)
            download_bundled_external("qt/" ${QT_VER} QT_PREFIX)
        endif()

        set(QT_PREFIX_HINT HINTS "${QT_PREFIX}")
    else()
        # Passing an empty HINTS seems to cause default system paths to get ignored in CMake 2.8 so
        # make sure to not pass anything if we don't have one.
        set(QT_PREFIX_HINT)
    endif()

    find_package(Qt5 REQUIRED COMPONENTS Widgets Multimedia Concurrent ${QT_PREFIX_HINT})

    if (ENABLE_QT_TRANSLATION)
        find_package(Qt5 REQUIRED COMPONENTS LinguistTools ${QT_PREFIX_HINT})
    endif()
endif()

# Ensure libusb is properly configured (based on dolphin libusb include)
if(NOT APPLE)
    include(FindPkgConfig)
    find_package(LibUSB)
endif()
if (NOT LIBUSB_FOUND)
    add_subdirectory(externals/libusb)
    set(LIBUSB_INCLUDE_DIR "")
    set(LIBUSB_LIBRARIES usb)
endif()

if (ENABLE_FFMPEG)
    if (CITRA_USE_BUNDLED_FFMPEG)
        if ((MSVC_VERSION GREATER_EQUAL 1920 AND MSVC_VERSION LESS 1940) AND "x86_64" IN_LIST ARCHITECTURE)
            set(FFmpeg_VER "ffmpeg-4.1-win64")
        else()
            message(FATAL_ERROR "No bundled FFmpeg binaries for your toolchain. Disable CITRA_USE_BUNDLED_FFMPEG and provide your own.")
        endif()

        if (DEFINED FFmpeg_VER)
            download_bundled_external("ffmpeg/" ${FFmpeg_VER} FFmpeg_PREFIX)
            set(FFMPEG_DIR "${FFmpeg_PREFIX}")
        endif()
    endif()

    if (ENABLE_FFMPEG_VIDEO_DUMPER)
        find_package(FFmpeg REQUIRED COMPONENTS avcodec avformat avutil swscale swresample)
    else()
        find_package(FFmpeg REQUIRED COMPONENTS avcodec)
    endif()
    if ("${FFmpeg_avcodec_VERSION}" VERSION_LESS "57.48.101")
        message(FATAL_ERROR "Found version for libavcodec is too low. The required version is at least 57.48.101 (included in FFmpeg 3.1 and later).")
    endif()
endif()

if (ENABLE_FFMPEG_VIDEO_DUMPER)
    add_definitions(-DENABLE_FFMPEG_VIDEO_DUMPER)
endif()

if (ENABLE_FDK)
    find_library(FDK_AAC fdk-aac DOC "The path to fdk_aac library")
    if(FDK_AAC STREQUAL "FDK_AAC-NOTFOUND")
        message(FATAL_ERROR "fdk_aac library not found.")
    endif()
endif()
# Platform-specific library requirements
# ======================================

if (APPLE)
    # Umbrella framework for everything GUI-related
    find_library(COCOA_LIBRARY Cocoa)
    find_library(AVFOUNDATION_LIBRARY AVFoundation)
    set(PLATFORM_LIBRARIES ${COCOA_LIBRARY} ${AVFOUNDATION_LIBRARY} ${IOKIT_LIBRARY} ${COREVIDEO_LIBRARY})
elseif (WIN32)
    # WSAPoll and SHGetKnownFolderPath (AppData/Roaming) didn't exist before WinNT 6.x (Vista)
    add_definitions(-D_WIN32_WINNT=0x0601 -DWINVER=0x0601)
    set(PLATFORM_LIBRARIES winmm ws2_32)
    if (MINGW)
        # PSAPI is the Process Status API
        set(PLATFORM_LIBRARIES ${PLATFORM_LIBRARIES} psapi imm32 version)
    endif()
elseif (CMAKE_SYSTEM_NAME MATCHES "^(Linux|kFreeBSD|GNU|SunOS)$")
    set(PLATFORM_LIBRARIES rt)
endif()

# Setup a custom clang-format target (if clang-format can be found) that will run
# against all the src files. This should be used before making a pull request.
# =======================================================================

set(CLANG_FORMAT_POSTFIX "-12")
find_program(CLANG_FORMAT
    NAMES clang-format${CLANG_FORMAT_POSTFIX}
          clang-format
    PATHS ${PROJECT_BINARY_DIR}/externals)
# if find_program doesn't find it, try to download from externals
if (NOT CLANG_FORMAT)
    if (WIN32)
        message(STATUS "Clang format not found! Downloading...")
        set(CLANG_FORMAT "${PROJECT_BINARY_DIR}/externals/clang-format${CLANG_FORMAT_POSTFIX}.exe")
        file(DOWNLOAD
            https://github.com/citra-emu/ext-windows-bin/raw/master/clang-format${CLANG_FORMAT_POSTFIX}.exe
            "${CLANG_FORMAT}" SHOW_PROGRESS
            STATUS DOWNLOAD_SUCCESS)
        if (NOT DOWNLOAD_SUCCESS EQUAL 0)
            message(WARNING "Could not download clang format! Disabling the clang format target")
            file(REMOVE ${CLANG_FORMAT})
            unset(CLANG_FORMAT)
        endif()
    else()
        message(WARNING "Clang format not found! Disabling the clang format target")
    endif()
endif()

if (CLANG_FORMAT)
    set(SRCS ${PROJECT_SOURCE_DIR}/src)
    set(CCOMMENT "Running clang format against all the .h and .cpp files in src/")
    if (WIN32)
        if(MINGW)
            add_custom_target(clang-format
                COMMAND find `cygpath -u ${SRCS}` -iname *.h -o -iname *.cpp -o -iname *.mm | xargs `cygpath -u ${CLANG_FORMAT}` -i
                COMMENT ${CCOMMENT})
        else()
            add_custom_target(clang-format
                COMMAND powershell.exe -Command "Get-ChildItem '${SRCS}/*' -Include *.cpp,*.h,*.mm -Recurse | Foreach {&'${CLANG_FORMAT}' -i $_.fullname}"
                COMMENT ${CCOMMENT})
    endif()
    else()
        add_custom_target(clang-format
            COMMAND find ${SRCS} -iname *.h -o -iname *.cpp -o -iname *.mm | xargs ${CLANG_FORMAT} -i
            COMMENT ${CCOMMENT})
    endif()
    unset(SRCS)
    unset(CCOMMENT)
endif()

# Include source code
# ===================

# This function should be passed a list of all files in a target. It will automatically generate
# file groups following the directory hierarchy, so that the layout of the files in IDEs matches the
# one in the filesystem.
function(create_target_directory_groups target_name)
    # Place any files that aren't in the source list in a separate group so that they don't get in
    # the way.
    source_group("Other Files" REGULAR_EXPRESSION ".")

    get_target_property(target_sources "${target_name}" SOURCES)

    foreach(file_name IN LISTS target_sources)
        get_filename_component(dir_name "${file_name}" PATH)
        # Group names use '\' as a separator even though the entire rest of CMake uses '/'...
        string(REPLACE "/" "\\" group_name "${dir_name}")
        source_group("${group_name}" FILES "${file_name}")
    endforeach()
endfunction()

# Gets a UTC timstamp and sets the provided variable to it
function(get_timestamp _var)
    string(TIMESTAMP timestamp UTC)
    set(${_var} "${timestamp}" PARENT_SCOPE)
endfunction()

# Prevent boost from linking against libs when building
add_definitions(-DBOOST_ERROR_CODE_HEADER_ONLY
    -DBOOST_SYSTEM_NO_LIB
    -DBOOST_DATE_TIME_NO_LIB
    -DBOOST_REGEX_NO_LIB
)

# generate git/build information
include(GetGitRevisionDescription)
get_git_head_revision(GIT_REF_SPEC GIT_REV)
git_describe(GIT_DESC --always --long --dirty)
git_branch_name(GIT_BRANCH)
get_timestamp(BUILD_DATE)

if (NOT USE_SYSTEM_BOOST)
    add_definitions( -DBOOST_ALL_NO_LIB )
endif()

enable_testing()
add_subdirectory(externals)

# See externals/CMakeLists.txt
foreach(def ${CRYPTOPP_COMPILE_DEFINITIONS})
    add_definitions(-D${def})
endforeach()

# Boost
if (USE_SYSTEM_BOOST)
    find_package(Boost 1.70.0 COMPONENTS serialization REQUIRED)
else()
    add_library(Boost::boost ALIAS boost)
    add_library(Boost::serialization ALIAS boost_serialization)
endif()

# SDL2
if (ENABLE_SDL2 AND USE_SYSTEM_SDL2)
    find_package(SDL2 REQUIRED)
    add_library(SDL2 INTERFACE)
    target_link_libraries(SDL2 INTERFACE "${SDL2_LIBRARY}")
    target_include_directories(SDL2 INTERFACE "${SDL2_INCLUDE_DIR}")
    add_library(SDL2::SDL2 ALIAS SDL2)
endif()

add_subdirectory(src)
add_subdirectory(dist/installer)


# Set citra-qt project or citra project as default StartUp Project in Visual Studio depending on whether QT is enabled or not
if(ENABLE_QT)
    set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT citra-qt)
else()
    set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT citra)
endif()

# Installation instructions
# =========================

# Install freedesktop.org metadata files, following those specifications:
# http://standards.freedesktop.org/desktop-entry-spec/desktop-entry-spec-latest.html
# http://standards.freedesktop.org/icon-theme-spec/icon-theme-spec-latest.html
# http://standards.freedesktop.org/shared-mime-info-spec/shared-mime-info-spec-latest.html
if(ENABLE_QT AND UNIX AND NOT APPLE)
    install(FILES "${PROJECT_SOURCE_DIR}/dist/citra.desktop"
            DESTINATION "${CMAKE_INSTALL_PREFIX}/share/applications")
    install(FILES "${PROJECT_SOURCE_DIR}/dist/citra.svg"
            DESTINATION "${CMAKE_INSTALL_PREFIX}/share/icons/hicolor/scalable/apps")
    install(FILES "${PROJECT_SOURCE_DIR}/dist/citra.xml"
            DESTINATION "${CMAKE_INSTALL_PREFIX}/share/mime/packages")
endif()

if(UNIX)
    if(ENABLE_SDL2)
        install(FILES "${PROJECT_SOURCE_DIR}/dist/citra.6"
                DESTINATION "${CMAKE_INSTALL_PREFIX}/share/man/man6")
    endif()

    if (ENABLE_QT)
        install(FILES "${PROJECT_SOURCE_DIR}/dist/citra-qt.6"
                DESTINATION "${CMAKE_INSTALL_PREFIX}/share/man/man6")
    endif()
endif()
