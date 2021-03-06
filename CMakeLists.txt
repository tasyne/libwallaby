cmake_minimum_required(VERSION 2.8)

project(libwallaby)

option(no_wallaby "Not a wallaby controller" OFF)

if(no_wallaby)
  add_definitions(-DNOT_A_WALLABY)
endif()

list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")


set(INCLUDE ${libwallaby_SOURCE_DIR}/include)
set(SRC ${libwallaby_SOURCE_DIR}/src)
set(PIXELTOASTER ${SRC}/pixeltoaster)

include_directories(${INCLUDE})


file(GLOB SOURCES ${SRC}/*.cpp)
file(GLOB C_TESTS tests/*.c)
file(GLOB PIXELTOASTER_SOURCES ${PIXELTOASTER}/*.cpp)

# Cross-compile
# set(CMAKE_CXX_COMPILER arm-linux-gnueabihf-g++)
# set(CMAKE_C_COMPILER arm-linux-gnueabihf-gcc)


# C++11
# http://www.guyrutenberg.com/2014/01/05/enabling-c11-c0x-in-cmake/
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
if(COMPILER_SUPPORTS_CXX11)
	add_definitions(--std=c++11)
elseif(COMPILER_SUPPORTS_CXX0X)
	add_definitions(--std=c++0x)
else()
	message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
endif()


# REMOVE ME
include_directories(${SRC}/pixeltoaster)

#############################
#   PixelToaster settings   #
#############################

if(APPLE)
	file(GLOB OBJC_SOURCES ${PIXELTOASTER}/*.mm)
elseif(WIN32)
	add_definitions(-fno-check-new)
endif()



if(DEFINED ENV{OPENCV_INCLUDE_DIR})
  set(OPENCV_INCLUDE $ENV{OPENCV_INCLUDE_DIR})
elseif(WIN32)
  set(OPENCV_INCLUDE ${CMAKE_SOURCE_DIR}/../opencv/build/install/include)
endif()

if(DEFINED ENV{OPENCV_LIB_DIR})
  set(OPENCV_LIB $ENV{OPENCV_LIB_DIR})
elseif(WIN32)
  set(OPENCV_LIB ${CMAKE_SOURCE_DIR}/../opencv/build/install/x86/vc11/lib)
endif()

if(DEFINED ENV{ZBAR_DIR})
  set(ZBAR_DIR $ENV{ZBAR_DIR})
elseif(WIN32)
  set(ZBAR_DIR "C:/Program Files (x86)/ZBar")
endif()

find_package( OpenCV REQUIRED )
find_package(JPEG REQUIRED)
find_package (Threads)


include_directories(${CMAKE_INSTALL_PREFIX}/include)
include_directories(${OPENCV_INCLUDE})
include_directories(${JPEG_INCLUDE_DIR})
include_directories(${ZBAR_DIR}/include)

link_directories(${CMAKE_INSTALL_PREFIX}/lib)
link_directories(${OPENCV_LIB})
link_directories(${ZBAR_DIR}/lib)


# Library: libwallaby
add_library(wallaby SHARED
	${SOURCES} ${PIXELTOASTER_SOURCES} ${OBJC_SOURCES})
  
target_link_libraries(wallaby ${OpenCV_LIBS} ${JPEG_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT} zbar)

if(APPLE)
  find_library(OPENGL_LIBRARY OpenGL)
  find_library(COCOA_LIBRARY Cocoa)
  find_library(CORESERVICE_LIBRARY CoreServices)
  target_link_libraries(wallaby ${OPENGL_LIBRARY} ${COCOA_LIBRARY} ${CORESERVICE_LIBRARY})
endif()

if(UNIX AND NOT APPLE)
    target_link_libraries(wallaby rt X11)
endif()


# Executables: test programs
# TODO: use a list and foreach to make this more concise
set(TESTS )

foreach(C_TEST ${C_TESTS})
  get_filename_component(C_TEST_NAME ${C_TEST} NAME_WE)
  set(TEST_BIN_NAME ${C_TEST_NAME}_c)
  add_executable(${TEST_BIN_NAME} ${C_TEST})
  target_link_libraries(${TEST_BIN_NAME} wallaby)
  list(APPEND TESTS ${TEST_BIN_NAME})
endforeach()

install(TARGETS ${TESTS} DESTINATION bin/wallaby)
install(TARGETS wallaby DESTINATION lib)
install(DIRECTORY ${INCLUDE}/wallaby DESTINATION include)

