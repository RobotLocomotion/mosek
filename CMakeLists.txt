cmake_minimum_required(VERSION 3.5)
project(mosek VERSION 7.1.0.45 LANGUAGES C)

if(CMAKE_VERSION VERSION_LESS 3.7)
  list(INSERT CMAKE_MODULE_PATH 0
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules/3.7"
  )
endif()

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE Release CACHE STRING
    "Choose the type of build; options are Debug Release RelWithDebInfo MinSizeRel"
    FORCE
  )
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY
    STRINGS
      Debug
      Release
      RelWithDebInfo
      MinSizeRel
  )
endif()

include(CMakePackageConfigHelpers)
include(CTest)
include(GNUInstallDirs)

if(NOT MOSEK_PLATFORM)
  if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    if(CMAKE_SIZEOF_VOID_P EQUAL 4)
      message(FATAL_ERROR "MOSEK is NOT available for the osx32x86 platform")
    else()
      set(MOSEK_PLATFORM osx64x86)
    endif()
  elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    if(CMAKE_SIZEOF_VOID_P EQUAL 4)
      set(MOSEK_PLATFORM linux32x86)
    else()
      set(MOSEK_PLATFORM linux64x86)
    endif()
  elseif(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    if(CMAKE_SIZEOF_VOID_P EQUAL 4)
      set(MOSEK_PLATFORM win32x86)
    else()
      set(MOSEK_PLATFORM win64x86)
    endif()
  else()
    message(FATAL_ERROR "MOSEK is NOT available for this platform")
  endif()
endif()

set(MOSEK_PLATFORM "${MOSEK_PLATFORM}" CACHE STRING
  "Choose the MOSEK platform; options are linux32x86 linux64x86 osx64x86 win32x86 win64x86"
)
set_property(CACHE MOSEK_PLATFORM PROPERTY
  STRINGS
    linux32x86
    linux64x86
    osx64x86
    win32x86
    win64x86
)

message(STATUS "MOSEK platform is ${MOSEK_PLATFORM}")

if(MOSEK_PLATFORM STREQUAL "osx64x86")
  set(MOSEK_ARCHIVE_SUFFIX .tar.bz2)
  set(MOSEK_ARCHIVE_EXPECTED_SHA256
    26c5bc0be667c92d1a5f81d2a1f7694de1fb0a3e9e9064c17f98e425db0a3c64
  )
elseif(MOSEK_PLATFORM MATCHES "^linux")
  set(MOSEK_ARCHIVE_SUFFIX .tar.bz2)

  if(MOSEK_PLATFORM STREQUAL "linux32x86")
    set(MOSEK_ARCHIVE_EXPECTED_SHA256
      78082ecf35166a5b43010b7de922a3e61814720669765062638919fba119de1b
    )
  else()
    set(MOSEK_ARCHIVE_EXPECTED_SHA256
      9b2bfcba7bcdd24b7e87ecdcccc11222302ced7b3d2a2af7090bdf625ab7cfae
    )
  endif()
elseif(MOSEK_PLATFORM MATCHES "^win")
  set(MOSEK_ARCHIVE_SUFFIX .zip)

  if(MOSEK_PLATFORM STREQUAL "win32x86")
    set(MOSEK_ARCHIVE_EXPECTED_SHA256
      b4752e09b042f713cd9a0061669a2a40643466bc1443d7c9f164cb482a790732
    )
  else()
    set(MOSEK_ARCHIVE_EXPECTED_SHA256
      89f3e4973d96cbee62b7bd08dde5b405e5742c6756697c11658b9106aecd97d9
    )
  endif()
else()
  message(FATAL_ERROR "MOSEK is NOT available for this platform")
endif()

set(MOSEK_ARCHIVE
  "${CMAKE_CURRENT_BINARY_DIR}/mosektools${MOSEK_PLATFORM}${MOSEK_ARCHIVE_SUFFIX}"
)

if(EXISTS "${MOSEK_ARCHIVE}")
  file(SHA256 "${MOSEK_ARCHIVE}" MOSEK_ARCHIVE_ACTUAL_SHA256)

  if(NOT MOSEK_ARCHIVE_ACTUAL_SHA256 STREQUAL MOSEK_ARCHIVE_EXPECTED_SHA256)
    file(REMOVE "${MOSEK_ARCHIVE}")
  endif()
endif()

if(NOT EXISTS "${MOSEK_ARCHIVE}")
  set(MOSEK_ARCHIVE_URL
    "http://download.mosek.com/stable/${PROJECT_VERSION_MAJOR}/mosektools${MOSEK_PLATFORM}${MOSEK_ARCHIVE_SUFFIX}"
  )

  message(STATUS "Downloading the MOSEK archive from ${MOSEK_ARCHIVE_URL}")

  file(DOWNLOAD "${MOSEK_ARCHIVE_URL}" "${MOSEK_ARCHIVE}"
    SHOW_PROGRESS
    STATUS MOSEK_DOWNLOAD_STATUS
    TIMEOUT 300
  )

  list(GET MOSEK_DOWNLOAD_STATUS 0 MOSEK_DOWNLOAD_RESULT)

  if(NOT MOSEK_DOWNLOAD_RESULT EQUAL 0)
    message(FATAL_ERROR "Could NOT extract MOSEK from archive")
  endif()

  file(SHA256 "${MOSEK_ARCHIVE}" MOSEK_ARCHIVE_ACTUAL_SHA256)

  if(NOT MOSEK_ARCHIVE_ACTUAL_SHA256 STREQUAL MOSEK_ARCHIVE_EXPECTED_SHA256)
    message(FATAL_ERROR "The MOSEK archive SHA256 hash does NOT match")
  endif()
endif()

set(MOSEK_ARCHIVE_DIR "${CMAKE_CURRENT_BINARY_DIR}/mosek")

if(EXISTS "${MOSEK_ARCHIVE_DIR}")
  file(TIMESTAMP "${MOSEK_ARCHIVE}" MOSEK_ARCHIVE_TIMESTAMP %s)
  file(TIMESTAMP "${MOSEK_ARCHIVE_DIR}" MOSEK_ARCHIVE_DIR_TIMESTAMP %s)

  if(MOSEK_ARCHIVE_TIMESTAMP GREATER MOSEK_ARCHIVE_DIR_TIMESTAMP)
    file(REMOVE_RECURSE "${MOSEK_ARCHIVE_DIR}")
  endif()
endif()

set(MOSEK_PLATFORM_DIR
  "${MOSEK_ARCHIVE_DIR}/${PROJECT_VERSION_MAJOR}/tools/platform/${MOSEK_PLATFORM}"
)

if(NOT EXISTS "${MOSEK_ARCHIVE_DIR}")
  message(STATUS
    "Extracting MOSEK from the archive mosektools${MOSEK_PLATFORM}${MOSEK_ARCHIVE_SUFFIX}"
  )

  execute_process(COMMAND "${CMAKE_COMMAND}" -E tar xf "${MOSEK_ARCHIVE}"
    WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
    TIMEOUT 300
    RESULT_VARIABLE MOSEK_EXTRACT_RESULT_VARIABLE
  )

  if(NOT MOSEK_EXTRACT_RESULT_VARIABLE EQUAL 0)
    message(FATAL_ERROR "Could NOT extract MOSEK from the archive")
  endif()

  if(NOT EXISTS "${MOSEK_PLATFORM_DIR}")
    message(FATAL_ERROR
      "Could NOT extract MOSEK platform directory from the archive"
    )
  endif()
endif()

if(NOT EXISTS "$ENV{HOME}/mosek/mosek.lic")
  message(STATUS
    "Could NOT find MOSEK license file. Please install your MOSEK license to \$HOME/mosek/mosek.lic."
  )
endif()

if(MOSEK_PLATFORM MATCHES "^win")
  set(IOMP5_LIBRARY libiomp5md)
else()
  set(IOMP5_LIBRARY iomp5)
endif()

set(IOMP5_LIBRARY_FILE
  "${CMAKE_SHARED_LIBRARY_PREFIX}${IOMP5_LIBRARY}${CMAKE_SHARED_LIBRARY_SUFFIX}"
)

if(MOSEK_PLATFORM MATCHES "32x86$")
  set(MOSEK_LIBRARY mosek)
  set(MOSEKGLB_LIBRARY mosekglb)
else()
  set(MOSEK_LIBRARY mosek64)
  set(MOSEKGLB_LIBRARY mosekglb64)
endif()

set(MOSEK_LIBRARY_DIR ${CMAKE_INSTALL_LIBDIR})

function(mosek_change_dylib_id_name name)
  execute_process(
    COMMAND
      "${CMAKE_INSTALL_NAME_TOOL}" -id "${CMAKE_INSTALL_PREFIX}/${MOSEK_LIBRARY_DIR}/${name}" "${MOSEK_PLATFORM_DIR}/bin/${name}"
      RESULT_VARIABLE MOSEK_INSTALL_NAME_TOOL_RESULT_VARIABLE
  )

  if(NOT MOSEK_INSTALL_NAME_TOOL_RESULT_VARIABLE EQUAL 0)
    message(FATAL_ERROR
      "Could NOT change ${name} dynamic shared library identification name"
    )
  endif()
endfunction()

set(MOSEK_ARCHIVE_FILES)
set(MOSEK_LIBRARY_FILES)
set(MOSEK_RUNTIME_FILES)

if(MOSEK_PLATFORM MATCHES "^win")
  set(MOSEK_RUNTIME_FILES
    "${MOSEK_PLATFORM_DIR}/bin/${IOMP5_LIBRARY_FILE}"
  )

  if(MOSEK_PLATFORM STREQUAL "win32x86")
    set(MOSEK_ARCHIVE_FILES
      "${MOSEK_PLATFORM_DIR}/bin/${CMAKE_STATIC_LIBRARY_PREFIX}${MOSEK_LIBRARY}_${PROJECT_VERSION_MAJOR}_${PROJECT_VERSION_MINOR}${CMAKE_STATIC_LIBRARY_SUFFIX}"
    )

    list(APPEND MOSEK_RUNTIME_FILES
      "${MOSEK_PLATFORM_DIR}/bin/${CMAKE_SHARED_LIBRARY_PREFIX}${MOSEK_LIBRARY}_${PROJECT_VERSION_MAJOR}_${PROJECT_VERSION_MINOR}${CMAKE_SHARED_LIBRARY_SUFFIX}"
      "${MOSEK_PLATFORM_DIR}/bin/${CMAKE_SHARED_LIBRARY_PREFIX}${MOSEKGLB_LIBRARY}_${PROJECT_VERSION_MAJOR}_${PROJECT_VERSION_MINOR}${CMAKE_SHARED_LIBRARY_SUFFIX}"
    )
  else()
    set(MOSEK_ARCHIVE_FILES
      "${MOSEK_PLATFORM_DIR}/bin/${CMAKE_STATIC_LIBRARY_PREFIX}${MOSEK_LIBRARY}_${PROJECT_VERSION_MAJOR}_${PROJECT_VERSION_MINOR}${CMAKE_STATIC_LIBRARY_SUFFIX}"
    )

    list(APPEND MOSEK_RUNTIME_FILES
      "${MOSEK_PLATFORM_DIR}/bin/${CMAKE_SHARED_LIBRARY_PREFIX}${MOSEK_LIBRARY}_${PROJECT_VERSION_MAJOR}_${PROJECT_VERSION_MINOR}${CMAKE_SHARED_LIBRARY_SUFFIX}"
      "${MOSEK_PLATFORM_DIR}/bin/${CMAKE_SHARED_LIBRARY_PREFIX}${MOSEKGLB_LIBRARY}_${PROJECT_VERSION_MAJOR}_${PROJECT_VERSION_MINOR}${CMAKE_SHARED_LIBRARY_SUFFIX}"
    )
  endif()
else()
  set(MOSEK_LIBRARY_FILES
    "${MOSEK_PLATFORM_DIR}/bin/${IOMP5_LIBRARY_FILE}"
    "${MOSEK_PLATFORM_DIR}/bin/${CMAKE_SHARED_LIBRARY_PREFIX}${MOSEK_LIBRARY}${CMAKE_SHARED_LIBRARY_SUFFIX}"
  )

  if(MOSEK_PLATFORM STREQUAL "osx64x86")
    set(MOSEK_LIBRARY_FILE
      "${CMAKE_SHARED_LIBRARY_PREFIX}${MOSEK_LIBRARY}.${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}${CMAKE_SHARED_LIBRARY_SUFFIX}"
    )
    set(MOSEKGLB_LIBRARY_FILE
      "${CMAKE_SHARED_LIBRARY_PREFIX}${MOSEKGLB_LIBRARY}.${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}${CMAKE_SHARED_LIBRARY_SUFFIX}"
    )


    mosek_change_dylib_id_name(${IOMP5_LIBRARY_FILE})
    mosek_change_dylib_id_name(${MOSEK_LIBRARY_FILE})
    mosek_change_dylib_id_name(${MOSEKGLB_LIBRARY_FILE})

  else()
    set(MOSEK_LIBRARY_FILE
      "${CMAKE_SHARED_LIBRARY_PREFIX}${MOSEK_LIBRARY}${CMAKE_SHARED_LIBRARY_SUFFIX}.${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}"
    )
    set(MOSEKGLB_LIBRARY_FILE
      "${CMAKE_SHARED_LIBRARY_PREFIX}${MOSEKGLB_LIBRARY}${CMAKE_SHARED_LIBRARY_SUFFIX}.${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}"
    )
  endif()

  list(APPEND MOSEK_LIBRARY_FILES
    "${MOSEK_PLATFORM_DIR}/bin/${MOSEK_LIBRARY_FILE}"
    "${MOSEK_PLATFORM_DIR}/bin/${MOSEKGLB_LIBRARY_FILE}"
  )
endif()

if(MOSEK_PLATFORM MATCHES "^win")
  set(LIBM_LIBRARY)
else()
  find_library(LIBM_LIBRARY NAMES m)

  if(NOT LIBM_LIBRARY)
    message(FATAL_ERROR "LibM NOT found")
  endif()
endif()

set(MOSEK_INCLUDE_DIR ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME})

install(FILES "${MOSEK_PLATFORM_DIR}/h/mosek.h"
  DESTINATION ${MOSEK_INCLUDE_DIR}
)

set(MOSEK_DIR ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME})

configure_package_config_file(
  mosek-c-targets.cmake.in mosek-c-targets-install.cmake
  INSTALL_DESTINATION ${MOSEK_DIR}
  PATH_VARS MOSEK_INCLUDE_DIR MOSEK_LIBRARY_DIR
  NO_CHECK_REQUIRED_COMPONENTS_MACRO
  NO_SET_AND_CHECK_MACRO
)

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/mosek-c-targets-install.cmake"
  DESTINATION ${MOSEK_DIR}
  RENAME mosek-c-targets.cmake
)

find_package(PkgConfig MODULE QUIET)

if(PkgConfig_FOUND)
  find_package(Threads MODULE REQUIRED)

  configure_file(mosek.pc.in mosek.pc @ONLY)

  install(FILES "${CMAKE_CURRENT_BINARY_DIR}/mosek.pc"
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
endif()

option(WITH_JAVA "Install the MOSEK Java bindings" OFF)
option(WITH_MATLAB "Install the MOSEK MATLAB bindings" OFF)

if(WITH_JAVA)
  find_package(Java 1.6 MODULE REQUIRED)
endif()

if(WITH_MATLAB)
  if(MOSEK_PLATFORM STREQUAL "osx64x86")
    set(MOSEK_MINIMUM_MATLAB_RELEASE r2012a)
    set(MOSEK_MINIMUM_MATLAB_VERSION 7.14)
  else()
    set(MOSEK_MINIMUM_MATLAB_RELEASE r2009b)
    set(MOSEK_MINIMUM_MATLAB_VERSION 7.9)
  endif()

  set(MATLAB_ADDITIONAL_VERSIONS "R2018a=9.4")
  find_package(Matlab ${MOSEK_MINIMUM_MATLAB_VERSION} MODULE REQUIRED
    COMPONENTS MAIN_PROGRAM
  )
endif()

if(Java_FOUND OR Matlab_FOUND)
  set(MOSEK_JAVA_DIR ${CMAKE_INSTALL_DATADIR}/java)

  if(Java_FOUND)
    set(MOSEK_JAR mosek.jar)
    set(MOSEK_JAR_FILES "${MOSEK_PLATFORM_DIR}/bin/${MOSEK_JAR}")

    configure_package_config_file(
      mosek-java-targets.cmake.in mosek-java-targets-install.cmake
      INSTALL_DESTINATION ${MOSEK_DIR}
      PATH_VARS MOSEK_JAVA_DIR
      NO_CHECK_REQUIRED_COMPONENTS_MACRO
      NO_SET_AND_CHECK_MACRO
    )

    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/mosek-java-targets-install.cmake"
      DESTINATION ${MOSEK_DIR}
      RENAME mosek-java-targets.cmake
    )

    set(INCLUDE_MOSEK_JAVA_TARGETS
      "include(\"\${CMAKE_CURRENT_LIST_DIR}/mosek-java-targets.cmake\")"
    )
  endif()

  if(Matlab_FOUND)
    if(MOSEK_PLATFORM STREQUAL "linux32x86")
      if(Matlab_VERSION VERSION_LESS 7.14)
        set(MOSEK_MATLAB_RELEASE r2009b)
        set(MOSEK_MATLAB_VERSION 7.9)
      else()
        set(MOSEK_MATLAB_RELEASE r2012a)
        set(MOSEK_MATLAB_VERSION 7.14)
      endif()
    elseif(MOSEK_PLATFORM STREQUAL "osx64x86")
      if(Matlab_VERSION VERSION_LESS 8.1)
        set(MOSEK_MATLAB_RELEASE r2012a)
        set(MOSEK_MATLAB_VERSION 7.14)
      else()
        set(MOSEK_MATLAB_RELEASE r2013a)
        set(MOSEK_MATLAB_VERSION 8.1)
      endif()
    else()
      if(Matlab_VERSION VERSION_LESS 7.14)
        set(MOSEK_MATLAB_RELEASE r2009b)
        set(MOSEK_MATLAB_VERSION 7.9)
      elseif(Matlab_VERSION VERSION_LESS 8.1)
        set(MOSEK_MATLAB_RELEASE r2012a)
        set(MOSEK_MATLAB_VERSION 7.14)
      else()
        set(MOSEK_MATLAB_RELEASE r2013a)
        set(MOSEK_MATLAB_VERSION 8.1)
      endif()
    endif()

    message(STATUS
      "MOSEK MATLAB closest version is ${MOSEK_MATLAB_VERSION} (${MOSEK_MATLAB_RELEASE})"
    )

    set(MOSEKMATLAB_JAR mosekmatlab.jar)
    list(APPEND MOSEK_JAR_FILES "${MOSEK_PLATFORM_DIR}/bin/${MOSEKMATLAB_JAR}")

    configure_file(addpath_mosek.m.in addpath_mosek.m @ONLY)
    configure_file(rmpath_mosek.m.in rmpath_mosek.m @ONLY)

    install(FILES
      "${CMAKE_CURRENT_BINARY_DIR}/addpath_mosek.m"
      "${CMAKE_CURRENT_BINARY_DIR}/rmpath_mosek.m"
      DESTINATION matlab
    )
  endif()

  set(MOSEKJAVA_LIBRARY mosekjava)

  if(MOSEK_PLATFORM STREQUAL "osx64x86")
    set(MOSEK_JNI_LIBRARY_SUFFIX .jnilib)
  else()
    set(MOSEK_JNI_LIBRARY_SUFFIX ${CMAKE_SHARED_LIBRARY_SUFFIX})
  endif()

  set(MOSEKJAVA_LIBRARY_FILE
    "${CMAKE_SHARED_LIBRARY_PREFIX}${MOSEKJAVA_LIBRARY}${PROJECT_VERSION_MAJOR}_${PROJECT_VERSION_MINOR}${MOSEK_JNI_LIBRARY_SUFFIX}"
  )

  if(MOSEK_PLATFORM MATCHES "^win")
    list(APPEND MOSEK_RUNTIME_FILES
      "${MOSEK_PLATFORM_DIR}/bin/${MOSEKJAVA_LIBRARY_FILE}"
    )
  else()
    list(APPEND MOSEK_LIBRARY_FILES
      "${MOSEK_PLATFORM_DIR}/bin/${MOSEKJAVA_LIBRARY_FILE}"
    )

    if(MOSEK_PLATFORM STREQUAL "osx64x86")
      mosek_change_dylib_id_name(${MOSEKJAVA_LIBRARY_FILE})
    endif()
  endif()

  install(FILES ${MOSEK_JAR_FILES} DESTINATION ${MOSEK_JAVA_DIR})
endif()

option(WITH_PYTHON "Install the MOSEK Python bindings" OFF)

if(WITH_PYTHON)
  find_package(PythonInterp 2.7 MODULE REQUIRED)

  message(STATUS
    "MOSEK Python major version is ${PYTHON_VERSION_MAJOR}"
  )

  set(MOSEK_PYTHON_DIR
    lib/python${PYTHON_VERSION_MAJOR}.${PYTHON_VERSION_MINOR}/site-packages
  )

  install(DIRECTORY
    "${MOSEK_PLATFORM_DIR}/python/${PYTHON_VERSION_MAJOR}/mosek"
    DESTINATION ${MOSEK_PYTHON_DIR}
  )
endif()

if(Java_FOUND OR Matlab_FOUND OR Python_FOUND)
  set(MOSEKSCOPT_LIBRARY mosekscopt)
  set(MOSEKSCOPT_LIBRARY_FILE
    "${CMAKE_SHARED_LIBRARY_PREFIX}${MOSEKSCOPT_LIBRARY}${PROJECT_VERSION_MAJOR}_${PROJECT_VERSION_MINOR}${CMAKE_SHARED_LIBRARY_SUFFIX}"
  )

  set(MOSEKXX_LIBRARY mosekxx)
  set(MOSEKXX_LIBRARY_FILE
    "${CMAKE_SHARED_LIBRARY_PREFIX}${MOSEKXX_LIBRARY}${PROJECT_VERSION_MAJOR}_${PROJECT_VERSION_MINOR}${CMAKE_SHARED_LIBRARY_SUFFIX}"
  )

  if(MOSEK_PLATFORM MATCHES "^win")
    list(APPEND MOSEK_ARCHIVE_FILES
      "${MOSEK_PLATFORM_DIR}/bin/${CMAKE_STATIC_LIBRARY_PREFIX}${MOSEKSCOPT_LIBRARY}${PROJECT_VERSION_MAJOR}_${PROJECT_VERSION_MINOR}${CMAKE_STATIC_LIBRARY_SUFFIX}"
    )

    list(APPEND MOSEK_RUNTIME_FILES
      "${MOSEK_PLATFORM_DIR}/bin/${MOSEKSCOPT_LIBRARY_FILE}"
      "${MOSEK_PLATFORM_DIR}/bin/${MOSEKXX_LIBRARY_FILE}"
    )
  else()
    list(APPEND MOSEK_LIBRARY_FILES
      "${MOSEK_PLATFORM_DIR}/bin/${MOSEKSCOPT_LIBRARY_FILE}"
      "${MOSEK_PLATFORM_DIR}/bin/${MOSEKXX_LIBRARY_FILE}"
    )

    if(MOSEK_PLATFORM STREQUAL "osx64x86")
      mosek_change_dylib_id_name(${MOSEKSCOPT_LIBRARY_FILE})
      mosek_change_dylib_id_name(${MOSEKXX_LIBRARY_FILE})
    endif()
  endif()
endif()

install(FILES ${MOSEK_ARCHIVE_FILES} ${MOSEK_LIBRARY_FILES}
  DESTINATION ${MOSEK_LIBRARY_DIR}
)

if(MOSEK_RUNTIME_FILES)
  install(PROGRAMS ${MOSEK_RUNTIME_FILES} DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

install(FILES
  "${CMAKE_CURRENT_BINARY_DIR}/mosek/${PROJECT_VERSION_MAJOR}/license.pdf"
  DESTINATION ${CMAKE_INSTALL_DOCDIR}
)

configure_package_config_file(mosek-config.cmake.in mosek-config.cmake
  INSTALL_DESTINATION ${MOSEK_DIR}
  PATH_VARS MOSEK_INCLUDE_DIR MOSEK_LIBRARY_DIR
  NO_CHECK_REQUIRED_COMPONENTS_MACRO
)

write_basic_package_version_file(mosek-config-version.cmake
  COMPATIBILITY SameMajorVersion
)

install(FILES
  "${CMAKE_CURRENT_BINARY_DIR}/mosek-config.cmake"
  "${CMAKE_CURRENT_BINARY_DIR}/mosek-config-version.cmake"
  DESTINATION ${MOSEK_DIR}
)

add_subdirectory(test)
