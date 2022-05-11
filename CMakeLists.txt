################################################################################
# Project:  Lib PyQt
# Purpose:  CMake build scripts
################################################################################
# Copyright (C) 2022, NextGIS <info@nextgis.com>
#
# Permission is hereby granted, free of charge, to any person obtaining a
# copy of this software and associated documentation files (the "Software"),
# to deal in the Software without restriction, including without limitation
# the rights to use, copy, modify, merge, publish, distribute, sublicense,
# and/or sell copies of the Software, and to permit persons to whom the
# Software is furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included
# in all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
# OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
# THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
# FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
# DEALINGS IN THE SOFTWARE.
################################################################################
cmake_minimum_required(VERSION 2.8.12 FATAL_ERROR)

project(pyqt5_sip)

if(NOT DEFINED PACKAGE_VENDOR)
    set(PACKAGE_VENDOR NextGIS)
endif()

if(NOT DEFINED PACKAGE_BUGREPORT)
    set(PACKAGE_BUGREPORT info@nextgis.com)
endif()

# some init settings
set(CMAKE_COLOR_MAKEFILE ON)
# set path to additional CMake modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

if(OSX_FRAMEWORK AND (BUILD_SHARED_LIBS OR BUILD_STATIC_LIBS))
  message(FATAL_ERROR "Only OSX_FRAMEWORK key or any or both BUILD_SHARED_LIBS
                       and BUILD_STATIC_LIBS keys are permitted")
endif()

if(OSX_FRAMEWORK)
    set(CMAKE_MACOSX_RPATH ON)
endif()

set(INSTALL_CMAKECONF_DIR CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)

find_package(Python3 COMPONENTS Interpreter Development REQUIRED)

if(${Python3_FOUND})
    set(PYTHON_EXECUTABLE ${Python3_EXECUTABLE})
    set(PYTHON_LIBRARY ${Python3_LIBRARIES})
    set(PYTHON_VERSION_MAJOR ${Python3_VERSION_MAJOR})
    set(PYTHON_VERSION_MINOR ${Python3_VERSION_MINOR})
endif()

file(DOWNLOAD "https://pypi.python.org/pypi/${PROJECT_NAME}/json" "${CMAKE_CURRENT_BINARY_DIR}/meta.json")

include(util)
get_compiler_version(COMPILER)
# get version and download url
execute_process(COMMAND ${PYTHON_EXECUTABLE}
    ${CMAKE_SOURCE_DIR}/cmake/pypi_process.py ${PLUGIN} ${CMAKE_CURRENT_BINARY_DIR}/meta.json ${COMPILER} ${LATEST_AVAILABLE_VERSION}
    OUTPUT_VARIABLE PACKAGE_DETAILS
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

list(GET PACKAGE_DETAILS 0 DOWNLOAD_URL)
list(GET PACKAGE_DETAILS 1 VERSION)
list(GET PACKAGE_DETAILS 2 FULL_VERSION)
list(GET PACKAGE_DETAILS 3 PROJECT_CPACK_FILENAME)

report_version(${PROJECT_NAME} ${VERSION})

# Download tarbal
set(OUT_DIR_NAME ${PROJECT_NAME}-${VERSION})
set(OUT_NAME "${OUT_DIR_NAME}.tar.gz")

if(NOT EXISTS "${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}")
    status_message("Download ${DOWNLOAD_URL}")
    file(DOWNLOAD "${DOWNLOAD_URL}" "${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}"
        SHOW_PROGRESS
    )
endif()

# Extract archive
if(NOT EXISTS ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME})
    status_message("Extract ${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}")
    execute_process(COMMAND ${CMAKE_COMMAND} -E tar xzf ${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME})
endif()

set(BUILD_DIR "build")

add_custom_target(build ALL
    COMMAND ${PYTHON_EXECUTABLE} setup.py build --build-lib=${BUILD_DIR}
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}
    VERBATIM
)

set(PACKAGE_NAME ${PROJECT_NAME})
string(TOUPPER ${PACKAGE_NAME} PACKAGE_UPPER_NAME)

include(GNUInstallDirs)

if(OSX_FRAMEWORK)
    set(INSTALL_PYTHON_DIR "Library/Python/${PYTHON_VERSION_MAJOR}/site-packages/")
elseif(WIN32)
    set(INSTALL_PYTHON_DIR "${CMAKE_INSTALL_LIBDIR}/Python${PYTHON_VERSION_MAJOR}/site-packages/")
else()
    set(INSTALL_PYTHON_DIR "lib/python${PYTHON_VERSION_MAJOR}/dist-packages/")
endif()

configure_file(cmake/PackageConfig.cmake.in
    ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake @ONLY)

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL )
    install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}/${BUILD_DIR}/PyQt5/
        DESTINATION "${INSTALL_PYTHON_DIR}/PyQt5" USE_SOURCE_PERMISSIONS)
    
    install(FILES ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake
      DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)
endif()

# Archiving ====================================================================
set(PACKAGE_NAME ${PROJECT_NAME})

set(CPACK_PACKAGE_NAME "${PACKAGE_NAME}")
set(CPACK_PACKAGE_VENDOR "${PACKAGE_VENDOR}")
set(CPACK_PACKAGE_VERSION "${VERSION}")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${PACKAGE_NAME} Installation")
set(CPACK_PACKAGE_RELOCATABLE TRUE)
set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)
set(CPACK_GENERATOR "ZIP")
set(CPACK_MONOLITHIC_INSTALL ON)
set(CPACK_STRIP_FILES TRUE)

# Get cpack zip archive name
get_cpack_filename(${VERSION} PROJECT_CPACK_FILENAME)
set(CPACK_PACKAGE_FILE_NAME ${PROJECT_CPACK_FILENAME})

include(CPack)
