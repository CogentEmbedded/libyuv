# CMakeLists for libyuv
# Originally created for "roxlu build system" to compile libyuv on windows
# Run with -DTEST=ON to build unit tests

PROJECT ( YUV C CXX )	# "C" is required even for C++ projects
CMAKE_MINIMUM_REQUIRED( VERSION 3.16 )
OPTION( TEST "Built unit tests" OFF )
OPTION(WITH_TOOLS "Build tools" OFF)
OPTION(WITH_JPEG "Enable JPEG support" OFF)

INCLUDE(GNUInstallDirs)
INCLUDE(CMakePackageConfigHelpers)

SET ( ly_base_dir	${PROJECT_SOURCE_DIR} )
SET ( ly_src_dir	${ly_base_dir}/source )
SET ( ly_inc_dir	${ly_base_dir}/include )
SET ( ly_tst_dir	${ly_base_dir}/unit_test )
SET ( ly_lib_name	yuv )
SET ( ly_lib_filename	"lib${ly_lib_name}" )

FILE ( GLOB_RECURSE	ly_source_files ${ly_src_dir}/*.cc )
LIST ( SORT			ly_source_files )

FILE ( GLOB_RECURSE	ly_unittest_sources ${ly_tst_dir}/*.cc )
LIST ( SORT			ly_unittest_sources )

if(MSVC)
  ADD_DEFINITIONS ( -D_CRT_SECURE_NO_WARNINGS )
endif()

ADD_LIBRARY	(${ly_lib_name} ${ly_source_files} )
ADD_LIBRARY(Libyuv::Libyuv ALIAS ${ly_lib_name})

TARGET_INCLUDE_DIRECTORIES(${ly_lib_name}
  PUBLIC
    $<BUILD_INTERFACE:${ly_inc_dir}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

if (WITH_TOOLS)
  # this creates the conversion tool
  ADD_EXECUTABLE			( yuvconvert ${ly_base_dir}/util/yuvconvert.cc )
  TARGET_LINK_LIBRARIES	( yuvconvert ${ly_lib_name} )

  # this creates the yuvconstants tool
  ADD_EXECUTABLE      ( yuvconstants ${ly_base_dir}/util/yuvconstants.c )
  TARGET_LINK_LIBRARIES  ( yuvconstants ${ly_lib_name} )
endif()

if (WITH_JPEG)
  find_package ( JPEG )
  if (JPEG_FOUND)
    target_include_directories(${ly_lib_name} PRIVATE ${JPEG_INCLUDE_DIR} )
    target_link_libraries(${ly_lib_name} PRIVATE ${JPEG_LIBRARY} )
    target_compile_definitions(${ly_lib_name} PRIVATE HAVE_JPEG )
  endif()
endif()

if(TEST)
  if (NOT CMAKE_CROSSCOMPILING)
    find_library(GTEST_LIBRARY gtest)
    if(GTEST_LIBRARY STREQUAL "GTEST_LIBRARY-NOTFOUND")
      set(GTEST_SRC_DIR /usr/src/gtest CACHE STRING "Location of gtest sources")
      if(EXISTS ${GTEST_SRC_DIR}/src/gtest-all.cc)
        message(STATUS "building gtest from sources in ${GTEST_SRC_DIR}")
        set(gtest_sources ${GTEST_SRC_DIR}/src/gtest-all.cc)
        add_library(gtest STATIC ${gtest_sources})
        include_directories(${GTEST_SRC_DIR})
        include_directories(${GTEST_SRC_DIR}/include)
        set(GTEST_LIBRARY gtest)
      else()
        message(FATAL_ERROR "TEST is set but unable to find gtest library")
      endif()
    endif()
  else()
    add_subdirectory(third_party/googletest/src)
    set(GTEST_LIBRARY gtest)
  endif()

  add_executable(libyuv_unittest ${ly_unittest_sources})
  target_link_libraries(libyuv_unittest ${ly_lib_name} ${GTEST_LIBRARY})
  find_library(PTHREAD_LIBRARY pthread)
  if(NOT PTHREAD_LIBRARY STREQUAL "PTHREAD_LIBRARY-NOTFOUND")
    target_link_libraries(libyuv_unittest pthread)
  endif()
  if (JPEG_FOUND)
    target_link_libraries(libyuv_unittest ${JPEG_LIBRARY})
  endif()

  if(NACL AND NACL_LIBC STREQUAL "newlib")
    target_link_libraries(libyuv_unittest glibc-compat)
  endif()

  find_library(GFLAGS_LIBRARY gflags)
  if(NOT GFLAGS_LIBRARY STREQUAL "GFLAGS_LIBRARY-NOTFOUND")
    target_link_libraries(libyuv_unittest gflags)
    add_definitions(-DLIBYUV_USE_GFLAGS)
  endif()
endif()

set_target_properties(${ly_lib_name} PROPERTIES EXPORT_NAME Libyuv)
INSTALL ( TARGETS ${ly_lib_name}
  EXPORT ${ly_lib_name}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  COMPONENT devel
)

INSTALL (DIRECTORY ${PROJECT_SOURCE_DIR}/include/
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
  COMPONENT devel)

configure_package_config_file(cmake/${ly_lib_filename}Config.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/${ly_lib_filename}Config.cmake
  INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${ly_lib_filename}
    PATH_VARS
        CMAKE_INSTALL_INCLUDEDIR
        CMAKE_INSTALL_LIBDIR)

install(
  FILES ${CMAKE_CURRENT_BINARY_DIR}/${ly_lib_filename}Config.cmake
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${ly_lib_filename}
  COMPONENT devel)

install(
  EXPORT ${ly_lib_name}
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${ly_lib_filename}
  FILE ${ly_lib_filename}Targets.cmake
  NAMESPACE Libyuv::)

if (WITH_TOOLS)
  INSTALL ( PROGRAMS
    ${CMAKE_BINARY_DIR}/yuvconvert
    ${CMAKE_BINARY_DIR}/yuvconstants
    DESTINATION ${CMAKE_INSTALL_BINDIR} )
endif()

# create the .deb and .rpm packages using cpack
INCLUDE ( CM_linux_packages.cmake )

