include("../build_utils/CMakeLists.txt")
SetRequiredCMakeVersion()
cmake_minimum_required(VERSION ${REQUIRED_CMAKE_VERSION})

project(PTI_Samples_L0_Intercept CXX)
SetCompilerFlags()
SetBuildType()

# Tool Library

add_library(zet_intercept SHARED "${PROJECT_SOURCE_DIR}/../loader/init.cc" tool.cc)
target_include_directories(zet_intercept
  PRIVATE "${PROJECT_SOURCE_DIR}"
  PRIVATE "${PROJECT_SOURCE_DIR}/../utils"
  PRIVATE "${PROJECT_SOURCE_DIR}/../ze_hot_functions"
  PRIVATE "${PROJECT_SOURCE_DIR}/../ze_hot_kernels")
if(CMAKE_INCLUDE_PATH)
  target_include_directories(zet_intercept
    PUBLIC "${CMAKE_INCLUDE_PATH}")
endif()

FindL0Library(zet_intercept)
FindL0Headers(zet_intercept)

FindL0HeadersPath(zet_intercept "${PROJECT_SOURCE_DIR}/../ze_hot_functions/gen_tracing_callbacks.py")

if(UNIX)
  FindDRMLibrary(zet_intercept)
  CheckDRMHeaders(zet_intercept)
endif()

# Loader

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DTOOL_NAME=zet_intercept")
add_executable(ze_intercept "${PROJECT_SOURCE_DIR}/../loader/loader.cc")
target_include_directories(ze_intercept
  PRIVATE "${PROJECT_SOURCE_DIR}/../utils")
if(UNIX)
  target_link_libraries(ze_intercept
    dl)
endif()