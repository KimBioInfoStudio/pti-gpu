macro(SetRequiredCMakeVersion)
  set(REQUIRED_CMAKE_VERSION 2.8)
endmacro()

macro(SetCompilerFlags)
endmacro()

macro(SetBuildType)
  if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release")
  endif()
  message(STATUS "Build Type: ${CMAKE_BUILD_TYPE}")

  if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    message(STATUS "Bitness: 64 bits")
  else()
    message(FATAL_ERROR "32-bit mode is not supported")
  endif()
endmacro()

macro(CheckForIntelCompiler)
  if(WIN32)
    set(INTEL_COMPILER_NAME "icl.exe")
  else()
    set(INTEL_COMPILER_NAME "icpx")
  endif()
  get_filename_component(COMPILER_NAME ${CMAKE_CXX_COMPILER} NAME)
  if(COMPILER_NAME STREQUAL ${INTEL_COMPILER_NAME})
    message(STATUS "Intel(R) C++ Compiler is used")
  else()
    message(FATAL_ERROR
      "Intel(R) C++ Compiler is required. "
      "Use \"CXX=${INTEL_COMPILER_NAME} cmake ..\" command to configure the sample.")
  endif()
endmacro()

macro(CheckForDPCCompiler)
  if(WIN32)
    set(INTEL_COMPILER_NAME "dpcpp.exe")
  else()
    set(INTEL_COMPILER_NAME "dpcpp")
  endif()
  get_filename_component(COMPILER_NAME ${CMAKE_CXX_COMPILER} NAME)
  if(COMPILER_NAME STREQUAL ${INTEL_COMPILER_NAME})
    message(STATUS "Intel(R) oneAPI DPC++ Compiler is used")
  else()
    message(FATAL_ERROR
      "Intel(R) oneAPI DPC++ Compiler is required. "
      "Use \"CXX=${INTEL_COMPILER_NAME} cmake ..\" command to configure the sample.")
  endif()
endmacro()

macro(FindOpenCLLibrary TARGET)
  if(DEFINED ENV{LD_LIBRARY_PATH})
    string(REPLACE ":" ";" SEARCH_LIB_PATH $ENV{LD_LIBRARY_PATH})
  endif()

  find_library(L0_LIB_PATH
    NAMES OpenCL
    PATHS ${SEARCH_LIB_PATH})
  if(NOT L0_LIB_PATH)
    message(FATAL_ERROR
      "OpenCL library is not found. "
      "You may need to install OpenCL ICD Loader (e.g. by Khrono) to fix this issue.")
  else()
    message(STATUS
      "OpenCL library is found at ${L0_LIB_PATH}")
  endif()

  target_link_libraries(${TARGET}
    "${L0_LIB_PATH}")
endmacro()

macro(FindOpenCLHeaders TARGET)
  include(CheckIncludeFileCXX)
  CHECK_INCLUDE_FILE_CXX(CL/cl.h OPENCL_INC_FOUND)
  if (OPENCL_INC_FOUND)
    include(CheckTypeSize)
    set(CMAKE_EXTRA_INCLUDE_FILES "CL/cl.h")
    CHECK_TYPE_SIZE(cl_kernel_sub_group_info CL_KERNEL_SUB_GROUP_INFO_SIZE LANGUAGE CXX)
    set(CMAKE_EXTRA_INCLUDE_FILES)
  endif()
  if (NOT (OPENCL_INC_FOUND AND CL_KERNEL_SUB_GROUP_INFO_SIZE))
    set(OPENCL_INC_PATH "${CMAKE_BINARY_DIR}")

    add_custom_target(cl_headers ALL
                      DEPENDS "${OPENCL_INC_PATH}/CL/cl.h"
                              "${OPENCL_INC_PATH}/CL/cl_gl.h"
                              "${OPENCL_INC_PATH}/CL/cl_version.h"
                              "${OPENCL_INC_PATH}/CL/cl_platform.h")
    add_custom_command(OUTPUT "${OPENCL_INC_PATH}/CL/cl.h"
                              "${OPENCL_INC_PATH}/CL/cl_gl.h"
                              "${OPENCL_INC_PATH}/CL/cl_version.h"
                              "${OPENCL_INC_PATH}/CL/cl_platform.h"
                      COMMAND python "${PROJECT_SOURCE_DIR}/../build_utils/get_cl_headers.py" "${OPENCL_INC_PATH}" "${CMAKE_BINARY_DIR}")

    target_include_directories(${TARGET}
      PUBLIC "${OPENCL_INC_PATH}")
    add_dependencies(${TARGET}
      cl_headers)
  endif()
  target_compile_options(${TARGET}
    PRIVATE -DCL_TARGET_OPENCL_VERSION=210)
endmacro()

macro(GetOpenCLTracingHeaders TARGET)
  set(OPENCL_TRACING_INC_PATH "${CMAKE_BINARY_DIR}")

  add_custom_target(cl_tracing_headers ALL
                    DEPENDS ${OPENCL_TRACING_INC_PATH}/CL/tracing_api.h
                            ${OPENCL_TRACING_INC_PATH}/CL/tracing_types.h)
  add_custom_command(OUTPUT ${OPENCL_TRACING_INC_PATH}/CL/tracing_api.h
                            ${OPENCL_TRACING_INC_PATH}/CL/tracing_types.h
                    COMMAND python "${PROJECT_SOURCE_DIR}/../build_utils/get_cl_tracing_headers.py" ${OPENCL_TRACING_INC_PATH} ${CMAKE_BINARY_DIR})

  target_include_directories(${TARGET}
    PUBLIC "${OPENCL_TRACING_INC_PATH}")
  add_dependencies(${TARGET}
    cl_tracing_headers)
endmacro()

macro(GetITT TARGET)
  set(ITT_INC_PATH "${CMAKE_BINARY_DIR}")

  add_custom_target(itt_headers ALL
                    DEPENDS ${ITT_INC_PATH}/ITT/disable_warnings.h
                            ${ITT_INC_PATH}/ITT/ittnotify_config.h
                            ${ITT_INC_PATH}/ITT/ittnotify_static.c
                            ${ITT_INC_PATH}/ITT/ittnotify_static.h
                            ${ITT_INC_PATH}/ITT/ittnotify_types.h
                            ${ITT_INC_PATH}/ITT/ittnotify.h
                            ${ITT_INC_PATH}/ITT/legacy/ittnotify.h)
  add_custom_command(OUTPUT ${ITT_INC_PATH}/ITT/disable_warnings.h
                            ${ITT_INC_PATH}/ITT/ittnotify_config.h
                            ${ITT_INC_PATH}/ITT/ittnotify_static.c
                            ${ITT_INC_PATH}/ITT/ittnotify_static.h
                            ${ITT_INC_PATH}/ITT/ittnotify_types.h
                            ${ITT_INC_PATH}/ITT/ittnotify.h
                            ${ITT_INC_PATH}/ITT/legacy/ittnotify.h
                    COMMAND python "${PROJECT_SOURCE_DIR}/../build_utils/get_itt.py" ${ITT_INC_PATH} ${CMAKE_BINARY_DIR})
  
  target_include_directories(${TARGET}
    PUBLIC "${ITT_INC_PATH}")
  add_dependencies(${TARGET}
    itt_headers)
endmacro()

macro(FindIGALibrary TARGET)
  if(WIN32)
    foreach(VALUE IN ITEMS $ENV{PATH})
      if(VALUE MATCHES "[S|s]ystem32*$")
        set(WIN_SYS_PATH ${VALUE})
        break()
      endif()
    endforeach()
    file(GLOB_RECURSE IGA_DLL_PATH "${WIN_SYS_PATH}/iga64.dll")
    if(NOT IGA_DLL_PATH)
      message(FATAL_ERROR
        "IGA is not found. "
        "You may need to install Intel(R) Processor Graphics Driver to fix this issue.")
    else()
      message(STATUS
        "IGA is found at ${IGA_DLL_PATH}")
    endif()

    find_library(IGA_LIB_PATH
      NAMES iga64
      PATHS ${SEARCH_LIB_PATH})
    if(NOT IGA_LIB_PATH)
      message(FATAL_ERROR
        "IGA library is not found. "
        "You may need to generate *.lib file from *.dll and provide path to it to cmake.")
    else()
      message(STATUS
        "IGA library is found at ${IGA_LIB_PATH}")
    endif()
  else()
    if(DEFINED ENV{LD_LIBRARY_PATH})
      string(REPLACE ":" ";" SEARCH_LIB_PATH $ENV{LD_LIBRARY_PATH})
    endif()
    find_library(IGA_LIB_PATH
      NAMES iga64
      PATHS ${SEARCH_LIB_PATH})
    if(NOT IGA_LIB_PATH)
      message(FATAL_ERROR
        "IGA library is not found. "
        "You may need to install Intel(R) Graphics Compute Runtime for oneAPI Level Zero and OpenCL(TM) Driver to fix this issue.")
    else()
      message(STATUS
        "IGA library is found at ${IGA_LIB_PATH}")
    endif()
  endif()

  target_link_libraries(${TARGET}
    "${IGA_LIB_PATH}")
endmacro()

macro(GetIGAHeaders TARGET)
  set(IGA_INC_PATH "${CMAKE_BINARY_DIR}")

  add_custom_target(iga_headers ALL
                    DEPENDS ${IGA_INC_PATH}/IGA/iga.h
                            ${IGA_INC_PATH}/IGA/iga_types_ext.hpp
                            ${IGA_INC_PATH}/IGA/iga_types_swsb.hpp
                            ${IGA_INC_PATH}/IGA/iga_bxml_ops.hpp
                            ${IGA_INC_PATH}/IGA/iga_bxml_enums.hpp
                            ${IGA_INC_PATH}/IGA/kv.h
                            ${IGA_INC_PATH}/IGA/kv.hpp)
  add_custom_command(OUTPUT ${IGA_INC_PATH}/IGA/iga.h
                            ${IGA_INC_PATH}/IGA/iga_types_ext.hpp
                            ${IGA_INC_PATH}/IGA/iga_types_swsb.hpp
                            ${IGA_INC_PATH}/IGA/iga_bxml_ops.hpp
                            ${IGA_INC_PATH}/IGA/iga_bxml_enums.hpp
                            ${IGA_INC_PATH}/IGA/kv.h
                            ${IGA_INC_PATH}/IGA/kv.hpp
                    COMMAND python "${PROJECT_SOURCE_DIR}/../build_utils/get_iga_headers.py" ${IGA_INC_PATH} ${CMAKE_BINARY_DIR})

  target_include_directories(${TARGET}
    PUBLIC "${IGA_INC_PATH}")
  add_dependencies(${TARGET}
    iga_headers)
endmacro()

macro(GetIGCHeaders TARGET)
  set(IGC_INC_PATH "${CMAKE_BINARY_DIR}")

  add_custom_target(igc_headers ALL
                    DEPENDS ${IGC_INC_PATH}/IGC/program_debug_data.h
                            ${IGC_INC_PATH}/IGC/patch_list.h)
  add_custom_command(OUTPUT ${IGC_INC_PATH}/IGC/program_debug_data.h
                            ${IGC_INC_PATH}/IGC/patch_list.h
                    COMMAND python "${PROJECT_SOURCE_DIR}/../build_utils/get_igc_headers.py" ${IGC_INC_PATH} ${CMAKE_BINARY_DIR})

  target_include_directories(${TARGET}
    PUBLIC "${IGC_INC_PATH}")
  add_dependencies(${TARGET}
    igc_headers)
endmacro()

macro(CheckForMDLibrary)
  if(WIN32)
    foreach(VALUE IN ITEMS $ENV{PATH})
      if(VALUE MATCHES "[S|s]ystem32*$")
        set(WIN_SYS_PATH ${VALUE})
        break()
      endif()
    endforeach()
    file(GLOB_RECURSE MD_LIB_PATH "${WIN_SYS_PATH}/igdmd64.dll")
    if(NOT MD_LIB_PATH)
      message(WARNING
        "Intel(R) Metrics Discovery Application Programming Interface is not found. "
        "You may need to install Intel(R) Processor Graphics Driver to fix this issue.")
    else()
      message(STATUS
        "Intel(R) Metrics Discovery Application Programming Interface is found at ${MD_LIB_PATH}")
    endif()
  else()
    if(DEFINED ENV{LD_LIBRARY_PATH})
      string(REPLACE ":" ";" SEARCH_LIB_PATH $ENV{LD_LIBRARY_PATH})
    endif()
    find_library(MD_LIB_PATH
      NAMES md
      PATHS ${SEARCH_LIB_PATH})
    if(NOT MD_LIB_PATH)
      message(WARNING
        "Intel(R) Metrics Discovery Application Programming Interface library is not found. "
        "You may need to install it to fix this issue.")
    else()
      message(STATUS
        "Intel(R) Metrics Discovery Application Programming Interface library is found at ${MD_LIB_PATH}")
    endif()
  endif()
  
endmacro()

macro(CheckForMetricsLibrary)
  if(WIN32)
    foreach(VALUE IN ITEMS $ENV{PATH})
      if(VALUE MATCHES "[S|s]ystem32*$")
        set(WIN_SYS_PATH ${VALUE})
        break()
      endif()
    endforeach()
    file(GLOB_RECURSE ML_LIB_PATH "${WIN_SYS_PATH}/igdml64.dll")
  else()
    if(DEFINED ENV{LD_LIBRARY_PATH})
      string(REPLACE ":" ";" SEARCH_LIB_PATH $ENV{LD_LIBRARY_PATH})
    endif()
    find_library(ML_LIB_PATH
      NAMES igdml64
      PATHS ${SEARCH_LIB_PATH})
  endif()
  
  if(NOT ML_LIB_PATH)
    message(WARNING
      "Metrics Library for Metrics Discovery API (Metrics Library for MD API) is not found. "
      "You may need to install Metrics Library for Metrics Discovery API (Metrics Library for MD API) to fix this issue.")
  else()
    message(STATUS
      "Metrics Library for Metrics Discovery API (Metrics Library for MD API) is found at ${ML_LIB_PATH}")
  endif()
endmacro()

macro(GetMDHeaders TARGET)
  set(MD_INC_PATH "${CMAKE_BINARY_DIR}")

  add_custom_target(md_headers ALL
                    DEPENDS ${MD_INC_PATH}/MD/metrics_discovery_api.h
                            ${MD_INC_PATH}/MD/metrics_discovery_internal_api.h)
  add_custom_command(OUTPUT ${MD_INC_PATH}/MD/metrics_discovery_api.h
                            ${MD_INC_PATH}/MD/metrics_discovery_internal_api.h
                    COMMAND python "${PROJECT_SOURCE_DIR}/../build_utils/get_md_headers.py" ${MD_INC_PATH} ${CMAKE_BINARY_DIR})

  target_include_directories(${TARGET}
    PUBLIC "${MD_INC_PATH}")
  add_dependencies(${TARGET}
    md_headers)
endmacro()

macro(FindGTPinLibrary TARGET)
  if(DEFINED GTPIN_PATH)
    find_library(GTPIN_LIB_PATH
      NAMES gtpin
      HINTS "${GTPIN_PATH}/Lib/intel64")
    if(GTPIN_LIB_PATH)
      message(STATUS "Graphics Technology Pin (GT Pin) library path: ${GTPIN_LIB_PATH}")
      target_link_libraries(${TARGET}
        ${GTPIN_LIB_PATH})
    else()
      message(FATAL_ERROR "Graphics Technology Pin (GT Pin) library was not found at ${GTPIN_PATH}/Lib/intel64")
    endif()
  else()
    if(UNIX)
      message(WARNING "Graphics Technology Pin (GT Pin) library path was not defined - it will be downloaded automatically on build")
      set(GTPIN_LIB_PATH "${CMAKE_BINARY_DIR}")

      add_custom_target(gtpin_libs ALL
                        DEPENDS ${GTPIN_LIB_PATH}/GTPIN/libgcc_s.so.1
                                ${GTPIN_LIB_PATH}/GTPIN/libged.so
                                ${GTPIN_LIB_PATH}/GTPIN/libgtpin.so
                                ${GTPIN_LIB_PATH}/GTPIN/libgtpin_core.so
                                ${GTPIN_LIB_PATH}/GTPIN/libiga_wrapper.so
                                ${GTPIN_LIB_PATH}/GTPIN/libstdc++.so.6)
      add_custom_command(OUTPUT ${GTPIN_LIB_PATH}/GTPIN/libgcc_s.so.1
                                ${GTPIN_LIB_PATH}/GTPIN/libged.so
                                ${GTPIN_LIB_PATH}/GTPIN/libgtpin.so
                                ${GTPIN_LIB_PATH}/GTPIN/libgtpin_core.so
                                ${GTPIN_LIB_PATH}/GTPIN/libiga_wrapper.so
                                ${GTPIN_LIB_PATH}/GTPIN/libstdc++.so.6
                        COMMAND python "${PROJECT_SOURCE_DIR}/../build_utils/get_gtpin_libs.py" ${GTPIN_LIB_PATH} ${CMAKE_BINARY_DIR})

      target_link_libraries(${TARGET}
        "${GTPIN_LIB_PATH}/GTPIN/libgtpin.so")
      add_dependencies(${TARGET}
        gtpin_libs)
    else()
      message(FATAL_ERROR "Graphics Technology Pin (GT Pin) path was not specified. Use -DGTPIN_PATH to set it")
    endif()
  endif()
endmacro()

macro(GetGTPinHeaders TARGET)
  if(DEFINED GTPIN_PATH)
    message(STATUS "Graphics Technology Pin (GT Pin) include path: ${GTPIN_PATH}/Include")
    target_include_directories(${TARGET}
      PUBLIC "${GTPIN_PATH}/Include"
      PUBLIC "${GTPIN_PATH}/Include/ged/intel64")
  else()
    if(UNIX)
      message(WARNING "Graphics Technology Pin (GT Pin) include path was not defined - it will be downloaded automatically on build")
      set(GTPIN_INC_PATH "${CMAKE_BINARY_DIR}")

      add_custom_target(gtpin_headers ALL
                        DEPENDS ${GTPIN_INC_PATH}/GTPIN/callbacks.h
                                ${GTPIN_INC_PATH}/GTPIN/client_knob.h
                                ${GTPIN_INC_PATH}/GTPIN/ged_ops.h
                                ${GTPIN_INC_PATH}/GTPIN/gtpin_deprecated_types.h
                                ${GTPIN_INC_PATH}/GTPIN/gtpin.h
                                ${GTPIN_INC_PATH}/GTPIN/gtpin.hpp
                                ${GTPIN_INC_PATH}/GTPIN/gtpintool_types.h
                                ${GTPIN_INC_PATH}/GTPIN/init.h
                                ${GTPIN_INC_PATH}/GTPIN/kernel.h
                                ${GTPIN_INC_PATH}/GTPIN/knob.h
                                ${GTPIN_INC_PATH}/GTPIN/knob_defs.h
                                ${GTPIN_INC_PATH}/GTPIN/ged/intel64/ged_basic_types.h
                                ${GTPIN_INC_PATH}/GTPIN/ged/intel64/ged_enumerations.h
                                ${GTPIN_INC_PATH}/GTPIN/ged/intel64/ged_enum_types.h
                                ${GTPIN_INC_PATH}/GTPIN/ged/intel64/ged.h
                                ${GTPIN_INC_PATH}/GTPIN/ged/intel64/ged_ins_field.h)
      add_custom_command(OUTPUT ${GTPIN_INC_PATH}/GTPIN/callbacks.h
                                ${GTPIN_INC_PATH}/GTPIN/client_knob.h
                                ${GTPIN_INC_PATH}/GTPIN/ged_ops.h
                                ${GTPIN_INC_PATH}/GTPIN/gtpin_deprecated_types.h
                                ${GTPIN_INC_PATH}/GTPIN/gtpin.h
                                ${GTPIN_INC_PATH}/GTPIN/gtpin.hpp
                                ${GTPIN_INC_PATH}/GTPIN/gtpintool_types.h
                                ${GTPIN_INC_PATH}/GTPIN/init.h
                                ${GTPIN_INC_PATH}/GTPIN/kernel.h
                                ${GTPIN_INC_PATH}/GTPIN/knob.h
                                ${GTPIN_INC_PATH}/GTPIN/knob_defs.h
                                ${GTPIN_INC_PATH}/GTPIN/ged/intel64/ged_basic_types.h
                                ${GTPIN_INC_PATH}/GTPIN/ged/intel64/ged_enumerations.h
                                ${GTPIN_INC_PATH}/GTPIN/ged/intel64/ged_enum_types.h
                                ${GTPIN_INC_PATH}/GTPIN/ged/intel64/ged.h
                                ${GTPIN_INC_PATH}/GTPIN/ged/intel64/ged_ins_field.h
                        COMMAND python "${PROJECT_SOURCE_DIR}/../build_utils/get_gtpin_headers.py" ${GTPIN_INC_PATH} ${CMAKE_BINARY_DIR})

      target_include_directories(${TARGET}
        PUBLIC "${GTPIN_INC_PATH}/GTPIN"
        PUBLIC "${GTPIN_INC_PATH}/GTPIN/ged/intel64")
      add_dependencies(${TARGET}
        gtpin_headers)
    else()
      message(FATAL_ERROR "Graphics Technology Pin (GT Pin) path was not specified. Use -DGTPIN_PATH to set it")
    endif()
  endif()
endmacro()

macro(FindL0Library TARGET)
  if(DEFINED ENV{LD_LIBRARY_PATH})
    string(REPLACE ":" ";" SEARCH_LIB_PATH $ENV{LD_LIBRARY_PATH})
  endif()

  find_library(L0_LIB_PATH
    NAMES ze_loader
    PATHS ${SEARCH_LIB_PATH})
  if(NOT L0_LIB_PATH)
    message(FATAL_ERROR
      "Level Zero loader is not found. "
      "You may need to install oneAPI Level Zero Driver to fix this issue.")
  else()
    message(STATUS
      "Level Zero loader is found at ${L0_LIB_PATH}")
  endif()

  target_link_libraries(${TARGET}
    "${L0_LIB_PATH}")
endmacro()

macro(FindL0Headers TARGET)
  if(CMAKE_INCLUDE_PATH)
    set(CMAKE_REQUIRED_INCLUDES ${CMAKE_INCLUDE_PATH})
  endif()
  include(CheckIncludeFileCXX)
  check_include_file_cxx(level_zero/ze_api.h LO_INC_FOUND)
  if (NOT LO_INC_FOUND)
    message(FATAL_ERROR
      "Level Zero headers are not found. "
      "You may need to install oneAPI Level Zero Driver to fix this issue.")
  endif()
  set(CMAKE_REQUIRED_INCLUDES)
endmacro()

macro(FindL0HeadersPath TARGET L0_GEN_SCRIPT)
  find_path(L0_INC_PATH
    NAMES level_zero)
  if (NOT L0_INC_PATH)
    message(FATAL_ERROR
      "Level Zero headers path is not found.\n"
      "You may need to install oneAPI Level Zero Driver to fix this issue.")
  else()
    message(STATUS "Level Zero headers are found at ${L0_INC_PATH}")
  endif()

  set(L0_GEN_INC_PATH "${CMAKE_BINARY_DIR}")
  add_custom_target(ze_gen_headers ALL
                    DEPENDS ${L0_GEN_INC_PATH}/tracing.gen)
  add_custom_command(OUTPUT ${L0_GEN_INC_PATH}/tracing.gen
                     COMMAND ${CMAKE_COMMAND} -E env PYTHONPATH="${PROJECT_SOURCE_DIR}/../utils" python ${L0_GEN_SCRIPT} ${L0_GEN_INC_PATH} "${L0_INC_PATH}/level_zero")                     
  target_include_directories(${TARGET}
    PUBLIC "${L0_GEN_INC_PATH}")
  add_dependencies(${TARGET}
    ze_gen_headers)
endmacro()

macro(CheckForOMPTHeaders)
  include(CheckIncludeFileCXX)
  CHECK_INCLUDE_FILE_CXX(omp-tools.h OMPT_INC_FOUND)
  if(NOT OMPT_INC_FOUND)
    message(FATAL_ERROR
      "OMPT is not supported. Try to update your compiler.")
  endif()
endmacro()

macro(FindDRMLibrary TARGET)
  if(DEFINED ENV{LD_LIBRARY_PATH})
    string(REPLACE ":" ";" SEARCH_LIB_PATH $ENV{LD_LIBRARY_PATH})
  endif()

  find_library(DRM_LIB_PATH
    NAMES drm
    PATHS ${SEARCH_LIB_PATH})
  if(NOT DRM_LIB_PATH)
    message(FATAL_ERROR
      "DRM library is not found. You may need to install it to fix this issue.")
  else()
    message(STATUS
      "DRM library is found at ${DRM_LIB_PATH}")
  endif()

  target_link_libraries(${TARGET}
    "${DRM_LIB_PATH}")
endmacro()

macro(CheckDRMHeaders TARGET)
  include(CheckIncludeFileCXX)
  CHECK_INCLUDE_FILE_CXX(drm/i915_drm.h I915_DRM_INC_FOUND)
  if (NOT I915_DRM_INC_FOUND)
    message(FATAL_ERROR
      "DRM headers are not found. You may need to install DRM developer package to fix this issue.")
  endif()

  find_path(DRM_INC_PATH
    NAMES drm)
  if(NOT DRM_INC_PATH)
    message(FATAL_ERROR
    "DRM headers path is not found. You may need to install DRM developer package to fix this issue.")
  else()
    message(STATUS
      "DRM headers path is found at ${DRM_INC_PATH}/drm")
  endif()
  
  set(CMAKE_REQUIRED_INCLUDES " ${DRM_INC_PATH}/drm")
  CHECK_INCLUDE_FILE_CXX(xf86drm.h XF86DRM_INC_FOUND)
  unset(CMAKE_REQUIRED_INCLUDES)
  if(NOT XF86DRM_INC_FOUND)
    message(FATAL_ERROR
      "DRM headers are not found. You may need to install DRM developer package to fix this issue.")
  endif()

  target_include_directories(${TARGET}
    PUBLIC "${DRM_INC_PATH}/drm")
endmacro()