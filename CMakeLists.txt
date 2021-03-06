cmake_minimum_required(VERSION 2.8.8)

if(APPLE)
  if(NOT CMAKE_OSX_DEPLOYMENT_TARGET)
    message(WARNING "Ensure that CMAKE_OSX_SYSROOT, CMAKE_OSX_DEPLOYMENT_TARGET are set correctly")
    set(CMAKE_OSX_ARCHITECTURES "x86_64" CACHE STRING "By default, build for 64-bit Leopard")
  endif()
endif()

project(TomVizSuperBuild)
set(SuperBuild_CMAKE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
set(SuperBuild_PROJECTS_DIR "${CMAKE_CURRENT_SOURCE_DIR}/projects")
set(SuperBuild_BINARY_DIR ${TomVizSuperBuild_BINARY_DIR})

include(CMakeDependentOption)

#-----------------------------------------------------------------------------
# Collect information about the build platform.
include(CheckTypeSize)
check_type_size(void* VOID_PTR_SIZE BUILTIN_TYPES_ONLY)
if(VOID_PTR_SIZE EQUAL 8)
  set(64bit_build TRUE)
else()
  set(64bit_build FALSE)
endif()

set(platform)
if(APPLE)
  set(platform "apple")
  option(APPLY_MOUNTAIN_LION_FIX "Apply Mountain Lion bundle fix" OFF)
  mark_as_advanced(APPLY_MOUNTAIN_LION_FIX)
elseif(UNIX)
  set(platform "unix")
elseif(WIN32)
  set(platform "win32")
endif()

# Set suffix to be used for generating archives. This ensures that the package
# files have decent names that we can directly upload to the website.
set(package_suffix "${CMAKE_SYSTEM_NAME}")
if(64bit_build)
  set(package_suffix "${package_suffix}-64bit")
else()
  set(package_suffix "${package_suffix}-32bit")
endif()

#-----------------------------------------------------------------------------
# Setup CMAKE_MODULE_PATH so that platform specific configurations are processed
# before the generic ones.
set(CMAKE_MODULE_PATH
  "${SuperBuild_PROJECTS_DIR}/${platform}"
  "${SuperBuild_PROJECTS_DIR}/common"
  "${SuperBuild_PROJECTS_DIR}"
  "${SuperBuild_CMAKE_DIR}"
  "${CMAKE_CURRENT_SOURCE_DIR}"
  ${CMAKE_MODULE_PATH})

#-----------------------------------------------------------------------------
# Setup some standard variables that control various locations and flags.
set(install_location "${CMAKE_CURRENT_BINARY_DIR}/install")
set(download_location "${CMAKE_CURRENT_BINARY_DIR}/downloads"
  CACHE PATH "Location where source tar-balls are (to be) downloaded.")
mark_as_advanced(download_location)

if(WIN32)
  set(ldflags)
  set(cppflags)
  set(cxxflags)
  set(ld_library_path)
else()
  # FIXME: currently, if any of these are empty, we have build issues on Unix.
  set(ldflags "$ENV{LDFLAGS} -L${install_location}/lib")
  set(cppflags "$ENV{CPPFLAGS} -fPIC")
  set(cxxflags "$ENV{CXXFLAGS} -fPIC")
  set(cflags "$ENV{CFLAGS} -fPIC")
  set(ld_library_path "${install_location}/lib:$ENV{LD_LIBRARY_PATH}")
endif()

if(APPLE)
  # --sysroot doesn't work with 10.5, 10.6 and -sysroot doesn't work with 10.9.
  if (CMAKE_OSX_DEPLOYMENT_TARGET AND CMAKE_OSX_DEPLOYMENT_TARGET VERSION_LESS "10.7")
    set (sysroot_flag "-sysroot")
  else ()
    set (sysroot_flag "--sysroot")
  endif()
  set (cflags "${cflags} -arch x86_64 -mmacosx-version-min=${CMAKE_OSX_DEPLOYMENT_TARGET} ${sysroot_flag}=${CMAKE_OSX_SYSROOT} ")
  set (cppflags "${cppflags} -arch x86_64 -mmacosx-version-min=${CMAKE_OSX_DEPLOYMENT_TARGET} ${sysroot_flag}=${CMAKE_OSX_SYSROOT} ")
  set (cxxflags "${cxxflags} -arch x86_64 -mmacosx-version-min=${CMAKE_OSX_DEPLOYMENT_TARGET} ${sysroot_flag}=${CMAKE_OSX_SYSROOT} ")
  unset (sysroot_flag)
endif()
set(prefix_path "${install_location}")

# merge in default/user-specified CMake flags.
set(ldflags "${CMAKE_SHARED_LINKER_FLAGS} ${ldflags}")
set(cppflags "${CMAKE_CXX_FLAGS} ${cppflags}")
set(cxxflags "${CMAKE_CXX_FLAGS} ${cxxflags}")
set(cflags "${CMAKE_C_FLAGS} ${cflags}")

string(STRIP "${ldflags}" ldflags)
string(STRIP "${cppflags}" cppflags)
string(STRIP "${cxxflags}" cxxflags)
string(STRIP "${cflags}"  cflags)
#-----------------------------------------------------------------------------
## Top-level options. These must be limited to a commonly used subset with
## default values being used for building TomViz binaries.
#option(PV_COMMAND_LINE_TOOLS_ONLY
#  "Turn this ON to build command lines tools alone." OFF)
if(WIN32)
  if((NOT "${CMAKE_GENERATOR}" MATCHES "^NMake.*$") OR
     (NOT "${CMAKE_GENERATOR}" MATCHES "^Visual Studio.*$"))
    # not use VS environment. We need to be pointed to nmake and devenv paths
    # since they are needed when building tools (qt, python, etc.)
    find_program(DEVENV_PATH devenv)
    find_program(NMAKE_PATH nmake)
    mark_as_advanced(DEVENV_PATH NMAKE_PATH)
  endif()
endif()

#-----------------------------------------------------------------------------
include(ParaViewModules)
include(versions)

#------------------------------------------------------------------------------
# enable testing support.
include(CTest)
configure_file(
  "${SuperBuild_CMAKE_DIR}/CTestCustom.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/CTestCustom.cmake" @ONLY)
#------------------------------------------------------------------------------

#-----------------------------------------------------------------------------
set(CM_PROJECTS_ALL)

# Include all optional projects. These are the ones user can pick and choose.
# Some of these allow using system libraries too.
include(qt)
include(python)
include(numpy)
if(NOT WIN32)
  include(fftw)
  include(pyfftw)
endif()
include(paraview)
include(tomviz)

process_dependencies()

if(tomviz_ENABLED)
  include(tomviz.bundle OPTIONAL)
endif()
