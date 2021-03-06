cmake_minimum_required(VERSION 2.8.8)
project(deepdetect)


set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)
list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)

include(ProcessorCount)
ProcessorCount(N)
include(ExternalProject)

set (deepdetect_VERSION_MAJOR 0)
set (deepdetect_VERSION_MINOR 1)

# options
OPTION(BUILD_TESTS "Should the tests be built")
 
# Get the current working branch
execute_process(
  COMMAND git rev-parse --abbrev-ref HEAD
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
  OUTPUT_VARIABLE GIT_BRANCH
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

# Get the latest abbreviated commit hash of the working branch
execute_process(
  COMMAND git log -1 --format=%H
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
  OUTPUT_VARIABLE GIT_COMMIT_HASH
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

set (CMAKE_CXX_FLAGS "-g -Wall -Wextra -fopenmp -fPIC -std=c++11 -O2 -DUSE_OPENCV -DUSE_LMDB")

# configure a header file to pass some of the CMake settings
# to the source code
configure_file (
  "${PROJECT_SOURCE_DIR}/dd_config.h.in"
  "${PROJECT_BINARY_DIR}/dd_config.h"
  )

configure_file(
  "${PROJECT_SOURCE_DIR}/src/githash.h.in"
  "${PROJECT_BINARY_DIR}/githash.h"
)

# dependency on Eigen for confusion matrix fast computation
find_package(Eigen3 REQUIRED)
include_directories("${EIGEN3_INCLUDE_DIR}")

# dependency on Boost
find_package(Boost 1.54 REQUIRED COMPONENTS filesystem thread system iostreams)

# optional packages
if (USE_CPU_ONLY)
else()
include(cmake/Cuda.cmake) # cuda + cudnn
endif()

# OpenCV
find_package(OpenCV REQUIRED)
set(OPENCV_VERSION ${OpenCV_VERSION_MAJOR})
include_directories(${OpenCV_INCLUDE_DIRS})
message(STATUS "OpenCV ${OPENCV_VERSION} (${OpenCV_VERSION}) found (${OpenCV_CONFIG_PATH})")

# customized Caffe as external project
if (CAFFE_INC_DIR AND CAFFE_LIB_DIR)
  # do nothing
else()
  message(STATUS "Configuring customized caffe")
  if (CUDA_FOUND)
    if (HAVE_CUDNN)
      ExternalProject_Add(
	caffe_dd
	PREFIX caffe_dd
	INSTALL_DIR ${CMAKE_BINARY_DIR}
	URL https://github.com/beniz/caffe/archive/master.tar.gz
	CONFIGURE_COMMAND ln -sf Makefile.config.gpu.cudnn Makefile.config && echo "OPENCV_VERSION:=${OPENCV_VERSION}" >> Makefile.config && make -j${N}
	INSTALL_COMMAND ""
	BUILD_IN_SOURCE 1
	)
    else()
ExternalProject_Add(
	caffe_dd
	PREFIX caffe_dd
	INSTALL_DIR ${CMAKE_BINARY_DIR}
	URL https://github.com/beniz/caffe/archive/master.tar.gz
	CONFIGURE_COMMAND ln -sf Makefile.config.gpu Makefile.config && echo "OPENCV_VERSION:=${OPENCV_VERSION}" >> Makefile.config && make -j${N}
	INSTALL_COMMAND ""
	BUILD_IN_SOURCE 1
	)
    endif()
  else()
    ExternalProject_Add(
      caffe_dd
      PREFIX caffe_dd
      INSTALL_DIR ${CMAKE_BINARY_DIR}
      URL https://github.com/beniz/caffe/archive/master.tar.gz
      CONFIGURE_COMMAND ln -sf Makefile.config.cpu Makefile.config && echo "OPENCV_VERSION:=${OPENCV_VERSION}" >> Makefile.config && make -j${N}
      INSTALL_COMMAND ""
      BUILD_IN_SOURCE 1
      )
  endif()
  if (CUDA_FOUND)
    set(CAFFE_INC_DIR ${CMAKE_BINARY_DIR}/caffe_dd/src/caffe_dd/include ${CMAKE_BINARY_DIR}/caffe_dd/src/caffe_dd/build/src ${CUDA_INCLUDE_DIRS})
    set(CAFFE_LIB_DIR ${CMAKE_BINARY_DIR}/caffe_dd/src/caffe_dd/build/lib ${CMAKE_BINARY_DIR}/caffe_dd/src/caffe_dd/build/src /usr/lib/x86_64-linux-gnu/hdf5/serial)
  else()
    set(CAFFE_INC_DIR ${CMAKE_BINARY_DIR}/caffe_dd/src/caffe_dd/include ${CMAKE_BINARY_DIR}/caffe_dd/src/caffe_dd/build/src /usr/include/hdf5/serial)
    set(CAFFE_LIB_DIR ${CMAKE_BINARY_DIR}/caffe_dd/src/caffe_dd/build/lib ${CMAKE_BINARY_DIR}/caffe_dd/src/caffe_dd/build/src /usr/lib/x86_64-linux-gnu/hdf5/serial)
  endif()
endif()

# XGBoost
message(STATUS "Configuring XGBoost")
if (USE_XGBOOST)
  add_definitions(-DUSE_XGBOOST)
  ExternalProject_Add(
    xgboost
    PREFIX xgboost
    INSTALL_DIR ${CMAKE_BINARY_DIR}
    DOWNLOAD_COMMAND git clone --recursive https://github.com/dmlc/xgboost.git
    UPDATE_COMMAND git submodule foreach git pull origin master
    CONFIGURE_COMMAND pwd && make
    INSTALL_COMMAND ""
    BUILD_IN_SOURCE 1
    )
  set(XGBOOST_INC_DIR ${CMAKE_BINARY_DIR}/xgboost/src/xgboost/include ${CMAKE_BINARY_DIR}/xgboost/src/xgboost/src ${CMAKE_BINARY_DIR}/xgboost/src/xgboost/rabit/include ${CMAKE_BINARY_DIR}/xgboost/src/xgboost/dmlc-core/include ${CMAKE_BINARY_DIR}/xgboost/src/xgboost/dmlc-core/src/)
  set(XGBOOST_LIB_DIR ${CMAKE_BINARY_DIR}/xgboost/src/xgboost/lib ${CMAKE_BINARY_DIR}/xgboost/src/xgboost/dmlc-core/ ${CMAKE_BINARY_DIR}/xgboost/src/xgboost/rabit/lib/)
endif()
  
# add the binary tree to the search path for include files
# so that we will find dd_config.h
include_directories("${PROJECT_BINARY_DIR}")
include_directories(${CAFFE_INC_DIR} ${XGBOOST_INC_DIR})

# main library, main & tests
include_directories ("${PROJECT_SOURCE_DIR}/src")
add_subdirectory (src) 
add_subdirectory(main)

# templates
file(COPY "templates" DESTINATION ".")

# examples
file(COPY "examples" DESTINATION ".")

# unit testing
if (BUILD_TESTS)
  enable_testing()
  add_subdirectory(tests)
endif()

# status
MESSAGE(STATUS "Build Tests          : ${BUILD_TESTS}")
