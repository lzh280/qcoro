cmake_minimum_required(VERSION 3.18.4)
set(qcoro_VERSION 0.12.0)
set(qcoro_SOVERSION 0)
project(qcoro LANGUAGES CXX VERSION ${qcoro_VERSION})

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

include(GNUInstallDirs)
include(CTest)
include(FeatureSummary)

#-----------------------------------------------------------#
# Options
#-----------------------------------------------------------#


# Allow QCORO_BUILD_TESTING to override BUILD_TESTING, but default to BUILD_TESTING if not set
if (DEFINED QCORO_BUILD_TESTING)
    set(BUILD_TESTING ${QCORO_BUILD_TESTING})
endif()

option(QCORO_BUILD_EXAMPLES "Build examples" ON)
add_feature_info(Examples QCORO_BUILD_EXAMPLES "Build examples")
option(QCORO_BUILD_TESTING "Build QCoro tests" ${BUILD_TESTING})
add_feature_info(Testing QCORO_BUILD_TESTING "Build QCoro tests")
option(QCORO_ENABLE_ASAN "Build with AddressSanitizer" OFF)
add_feature_info(Asan QCORO_ENABLE_ASAN "Build with AddressSanitizer")
option(QCORO_DISABLE_DEPRECATED_TASK_H "Disable deprecated task.h header" OFF)

if(WIN32 OR APPLE OR ANDROID)
    option(QCORO_WITH_QTDBUS "Build QtDBus support" OFF)
else()
    option(QCORO_WITH_QTDBUS "Build QtDBus support" ON)
endif()
add_feature_info(QtDBus QCORO_WITH_QTDBUS "Build QtDBus support")
option(QCORO_WITH_QTNETWORK "Build QtNetwork support" ON)
add_feature_info(QtNetwork QCORO_WITH_QTNETWORK "Build QtNetwork support")
option(QCORO_WITH_QTWEBSOCKETS "Build QtWebSockets support" ON)
add_feature_info(QtWebSockets QCORO_WITH_QTWEBSOCKETS "Build QtWebSockets support")
option(QCORO_WITH_QTQUICK "Build QtQuick support" ON)
add_feature_info(QtQuick QCORO_WITH_QTQUICK "Build QtQuick support")
option(QCORO_WITH_QML "Build QML integration features" ON)
add_feature_info(QtQml QCORO_WITH_QML "Build QML integration features")
option(QCORO_WITH_QTTEST "Build QtTest support" ON)
add_feature_info(QtTest QCORO_WITH_QTTEST "Build QtTest support")

#-----------------------------------------------------------#
# Dependencies
#-----------------------------------------------------------#

set(THREADS_PREFER_PTHREAD_FLAG TRUE)
find_package(Threads REQUIRED)

include(cmake/CheckAtomic.cmake)

set(REQUIRED_QT_COMPONENTS Core)
set(REQUIRED_QT5_COMPONENTS)
set(REQUIRED_QT6_COMPONENTS)
if (QCORO_WITH_QTDBUS)
    list(APPEND REQUIRED_QT_COMPONENTS DBus)
endif()
if (QCORO_WITH_QTNETWORK)
    list(APPEND REQUIRED_QT_COMPONENTS Network)
endif()
if (QCORO_WITH_QTWEBSOCKETS)
    list(APPEND REQUIRED_QT_COMPONENTS WebSockets)
endif()
if (QCORO_WITH_QTQUICK)
    list(APPEND REQUIRED_QT_COMPONENTS Gui Quick QuickPrivate)
endif()
if (QCORO_WITH_QML)
    list(APPEND REQUIRED_QT_COMPONENTS Qml)
    # Qt6 needs access to private API
    list(APPEND REQUIRED_QT6_COMPONENTS QmlPrivate)
endif()
if (QCORO_WITH_QTTEST)
    list(APPEND REQUIRED_QT_COMPONENTS Test)
endif()
if (QCORO_BUILD_EXAMPLES)
    list(APPEND REQUIRED_QT_COMPONENTS Widgets Concurrent)
endif()
if (BUILD_TESTING)
    list(APPEND REQUIRED_QT_COMPONENTS Test Concurrent)
endif()

set(MIN_REQUIRED_QT5_VERSION "5.12")
set(MIN_REQUIRED_QT6_VERSION "6.2.0")

include(cmake/QCoroFindQt.cmake)
# Find Qt. If USE_QT_VERSION is not set, it will try to look for Qt6 first
# and fallback to Qt5 otherwise.
qcoro_find_qt(
    QT_VERSION "${USE_QT_VERSION}"
    COMPONENTS "${REQUIRED_QT_COMPONENTS}"
    QT5_COMPONENTS "${REQUIRED_QT5_COMPONENTS}"
    QT6_COMPONENTS "${REQUIRED_QT6_COMPONENTS}"
    FOUND_VER_VAR QT_VERSION_MAJOR
)

configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/config.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/config.h
)

#-----------------------------------------------------------#
# Compiler Settings
#-----------------------------------------------------------#

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_AUTOMOC ON)

if (MSVC)
    if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
        # Explicitly enable exceptions support for clang-cl (it's only enabled by CMake when targeting the Windows-MSVC platform,
        # see https://github.com/danvratil/qcoro/issues/90 for details)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /EHsc")
    endif()
else()
    if (ANDROID)
        include(DetectAndroidNDKVersion)
        detectAndroidNDKVersion(NDK_VERSION)
        if ("${NDK_VERSION}" VERSION_LESS 26)
            # Android NDK < 26 ships mismatching versions of clang and libc++ - libc++ is older and doesn't have coroutine support
            # which forces us to use coroutines from std::experimental. But the clang itself is newer and emits an error about
            # std::experimental coroutines being deprecated in LLVM 14. This option is needed to suppress the error.
            add_compile_options(-Wno-error=deprecated-experimental-coroutine)
        endif()
    endif()
endif()

if (QCORO_ENABLE_ASAN)
    if (MSVC)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /INCREMENTAL:NO")
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /INCREMENTAL:NO")
        if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address")
        else()
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /fsanitize=address")
        endif()
    else()
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address -fno-omit-frame-pointer -fsanitize-recover=address")
    endif()
endif()

include(qcoro/QCoroMacros.cmake)
qcoro_enable_coroutines()

include(cmake/CodeCoverage.cmake)
add_code_coverage()
add_code_coverage_all_targets(EXCLUDE "${CMAKE_BINARY_DIR}" tests/utils/*)

#-----------------------------------------------------------#
# Definitions
#-----------------------------------------------------------#

# debug suffixes for qmake compatibility
if(WIN32)
    set(CMAKE_DEBUG_POSTFIX "d")
elseif(APPLE)
    set(CMAKE_DEBUG_POSTFIX "_debug")
else()
    set(CMAKE_DEBUG_POSTFIX "")
endif()

set(QCORO_TARGET_PREFIX "QCoro${QT_VERSION_MAJOR}")
set(QCORO_INSTALL_INCLUDEDIR "${CMAKE_INSTALL_INCLUDEDIR}/qcoro${QT_VERSION_MAJOR}")

#-----------------------------------------------------------#
# Sources
#-----------------------------------------------------------#

set(QCORO_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})
add_subdirectory(qcoro)
if (QCORO_BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()
if (QCORO_BUILD_TESTING)
    add_subdirectory(tests)
endif()

#-----------------------------------------------------------#
# Installation
#-----------------------------------------------------------#

include(CMakePackageConfigHelpers)

install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/config.h
    DESTINATION ${QCORO_INSTALL_INCLUDEDIR}/qcoro
    COMPONENT Devel
)

configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/QCoroConfig.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/QCoro${QT_VERSION_MAJOR}Config.cmake"
    INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/QCoro${QT_VERSION_MAJOR}"
    PATH_VARS CMAKE_INSTALL_INCLUDEDIR
)

write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/QCoro${QT_VERSION_MAJOR}ConfigVersion.cmake"
    VERSION ${qcoro_VERSION}
    COMPATIBILITY SameMajorVersion
)

install(
    FILES "${CMAKE_CURRENT_BINARY_DIR}/QCoro${QT_VERSION_MAJOR}Config.cmake"
          "${CMAKE_CURRENT_BINARY_DIR}/QCoro${QT_VERSION_MAJOR}ConfigVersion.cmake"
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/QCoro${QT_VERSION_MAJOR}"
    COMPONENT Devel
)

#-----------------------------------------------------------#
# Summary
#-----------------------------------------------------------#

feature_summary(FATAL_ON_MISSING_REQUIRED_PACKAGES WHAT ALL)
