MESSAGE(STATUS "PhysX Build Platform: " ${TARGET_BUILD_PLATFORM})
MESSAGE(STATUS "  CXX Compiler: " ${CMAKE_CXX_COMPILER})

# Avoid including NvidiaBuildOptions: {{{
option(NV_USE_STATIC_WINCRT "Use the statically linked windows CRT" ON)
mark_as_advanced(NV_USE_STATIC_WINCRT)
option(NV_USE_DEBUG_WINCRT "Use the debug version of the CRT" OFF)
mark_as_advanced(NV_USE_DEBUG_WINCRT)
set(NV_USE_GAMEWORKS_OUTPUT_DIRS ON)

include(GetCompilerAndPlatform)
if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(LIBPATH_SUFFIX "64")
    # Set default exe suffix. Unset on platforms that don't need it. Include underscore since it's optional
    set(EXE_SUFFIX "_64")
	set(RESOURCE_LIBPATH_SUFFIX "x64")
else()
    set(LIBPATH_SUFFIX "32")
    # Set default exe suffix. Unset on platforms that don't need it. Include underscore since it's optional
    set(EXE_SUFFIX "_32")
	set(RESOURCE_LIBPATH_SUFFIX "x86")
endif()
# Set the WINCRT_DEBUG and WINCRT_NDEBUG variables for use in project compile settings
# Really only relevant to windows
set(DISABLE_ITERATOR_DEBUGGING "/D \"_HAS_ITERATOR_DEBUGGING=0\" /D \"_ITERATOR_DEBUG_LEVEL=0\"")
set(DISABLE_ITERATOR_DEBUGGING_CUDA "-D_HAS_ITERATOR_DEBUGGING=0 -D_ITERATOR_DEBUG_LEVEL=0")
set(CRT_DEBUG_FLAG "/D \"_DEBUG\"")
set(CRT_NDEBUG_FLAG "/D \"NDEBUG\"")
# Need a different format for CUDA
set(CUDA_DEBUG_FLAG "-DNDEBUG ${DISABLE_ITERATOR_DEBUGGING_CUDA}")
set(CUDA_NDEBUG_FLAG "-DNDEBUG")
IF(NV_USE_STATIC_WINCRT)
    SET(WINCRT_NDEBUG "/MT ${DISABLE_ITERATOR_DEBUGGING} ${CRT_NDEBUG_FLAG}" CACHE INTERNAL "Windows CRT build setting")
    SET(CUDA_CRT_COMPILE_OPTIONS_NDEBUG "/MT")

    IF (NV_USE_DEBUG_WINCRT)
        SET(CUDA_DEBUG_FLAG "-D_DEBUG")
        SET(WINCRT_DEBUG "/MTd ${CRT_DEBUG_FLAG}" CACHE INTERNAL "Windows CRT build setting")
        SET(CUDA_CRT_COMPILE_OPTIONS_DEBUG "/MTd")
    ELSE()
        SET(WINCRT_DEBUG "/MT ${DISABLE_ITERATOR_DEBUGGING} ${CRT_NDEBUG_FLAG}" CACHE INTERNAL "Windows CRT build setting")
        SET(CUDA_CRT_COMPILE_OPTIONS_DEBUG "/MT")
    ENDIF()
ELSE()
    SET(WINCRT_NDEBUG "/MD ${DISABLE_ITERATOR_DEBUGGING} ${CRT_NDEBUG_FLAG}")
    SET(CUDA_CRT_COMPILE_OPTIONS_NDEBUG "/MD")

    IF(NV_USE_DEBUG_WINCRT)
        SET(CUDA_DEBUG_FLAG "-D_DEBUG")
        SET(WINCRT_DEBUG "/MDd ${CRT_DEBUG_FLAG}" CACHE INTERNAL "Windows CRT build setting")
        SET(CUDA_CRT_COMPILE_OPTIONS_DEBUG "/MDd")
    ELSE()
        SET(WINCRT_DEBUG "/MD ${DISABLE_ITERATOR_DEBUGGING} ${CRT_NDEBUG_FLAG}" CACHE INTERNAL "Windows CRT build setting")
        SET(CUDA_CRT_COMPILE_OPTIONS_DEBUG "/MD")
    ENDIF()
ENDIF()

# platform_bin_name is of type linux.clang, win.x86_64.vc140 etc.
GetPlatformBinName(PLATFORM_BIN_NAME ${LIBPATH_SUFFIX})
message(STATUS "PLATFORM_BIN_NAME: ${PLATFORM_BIN_NAME}")
# }}} End NvidiaBuildOptions replacement

set(physx_build_targets_file "${CMAKE_CURRENT_BINARY_DIR}/PhysXTargets.cmake")
set(install_cmake_dir "${PX_ROOT_LIB_DIR}/cmake/physx")

# Include the platform specific CMakeLists
message(STATUS "PLATFORM_CMAKELISTS: ${PLATFORM_CMAKELISTS}")
include(${PLATFORM_CMAKELISTS})

# generate PxPhysXConfig.h header that will contain PhysX configuration defines like PX_PHYSX_STATIC_LIB
# XXX(phcerdan) the template is actually empty? Maybe populated after? No harm having an empty header meanwhile.
SET(HEADER_GUARD_NAME "CONFIG")
SET(HEADER_CONTENT "")
configure_file(${CMAKEMODULES_PATH}/template/PxIncludeTemplate.h ${PHYSX_ROOT_DIR}/include/PxConfig.h)

### Move libPhysXGPU from source tree to build and install tree:
# libPhysXGPU libraries are in the repository (/bin) but the CUDA code is not open
# XXX(phcerdan) Nvdia does not provides GNU (gcc) GPU libaries, only clang.
# I guess both compilers are ABI compatible, but not explicitly tested in this codebase
# GCC in linux is not an option right now upstream

##  Copy the folder containing libPhysXGPU to the build and install folder for the current CONFIG
# Build tree
add_custom_command(TARGET PhysX POST_BUILD
  COMMAND "${CMAKE_COMMAND}" -E copy_directory
     ${PROJECT_SOURCE_DIR}/bin/${PLATFORM_BIN_NAME}/$<CONFIG>/
     "${CMAKE_CURRENT_BINARY_DIR}/"
  COMMENT "Copying libPhysXGPU from the appropiate source folder to the build directory.")
# Install tree
install(FILES ${PROJECT_SOURCE_DIR}/bin/${PLATFORM_BIN_NAME}/$<CONFIG>
    DESTINATION ${PHYSX_INSTALL_PREFIX}/bin/)

# Introduced based on upstream version: 4.1.1.27006925
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  ${CMAKE_CURRENT_BINARY_DIR}/PhysXConfigVersion.cmake
  VERSION ${PHYSX_VERSION}
  COMPATIBILITY SameMajorVersion)

# Build tree
# TODO Export every target to build tree in each cmake/<platform>/CMakeLists.txt file
set(PhysX_TARGETS_FILE ${physx_build_targets_file})
configure_package_config_file(
  ${PROJECT_CMAKE_FILES_ABSOLUTE_DIR}/PhysXConfig.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/PhysXConfig.cmake
  INSTALL_DESTINATION ${install_cmake_dir}
  PATH_VARS PhysX_TARGETS_FILE
  # NO_CHECK_REQUIRED_COMPONENTS_MACRO # PhysX does not provide components
  INSTALL_PREFIX ${CMAKE_CURRENT_BINARY_DIR}
  )

# Install tree
install (EXPORT PhysX
  NAMESPACE PhysX::
  FILE PhysXTargets.cmake
  DESTINATION ${install_cmake_dir} )

set(PhysX_TARGETS_FILE ${CMAKE_INSTALL_PREFIX}/${install_cmake_dir}/PhysXTargets.cmake)
configure_package_config_file(
  ${PROJECT_CMAKE_FILES_ABSOLUTE_DIR}/PhysXConfig.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/cmake/PhysXConfig.cmake
  INSTALL_DESTINATION ${install_cmake_dir}
  PATH_VARS PhysX_TARGETS_FILE
  # NO_CHECK_REQUIRED_COMPONENTS_MACRO # PhysX does not provide components
  )
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/cmake/PhysXConfig.cmake
              ${CMAKE_CURRENT_BINARY_DIR}/PhysXConfigVersion.cmake
        DESTINATION ${install_cmake_dir})

