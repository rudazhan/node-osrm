cmake_minimum_required(VERSION 3.1)
project(node-osrm C CXX)

# Explicitly set the build type to Release if no other type is specified
# on the command line.  Without this, cmake defaults to an unoptimized,
# non-debug build, which almost nobody wants.
if(NOT CMAKE_BUILD_TYPE)
  message(STATUS "No build type specified, defaulting to Release")
  set(CMAKE_BUILD_TYPE Release)
endif()

option(BUILD_LIBOSRM "Download and build own libsorm version" OFF)

set(OSRM_BINARIES "")
if(BUILD_LIBOSRM)
  set(OSRM_INSTALL_DIR "/tmp/osrm-backend")
  set(OSRM_DEPS_DIR "${CMAKE_CURRENT_SOURCE_DIR}/deps/osrm-backend-${CMAKE_BUILD_TYPE}")

  execute_process(COMMAND node -p "require('../package.json').osrm_release" OUTPUT_VARIABLE OSRM_VERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
  message(STATUS "Using osrm ${OSRM_VERSION}")
  message(STATUS "Downloading to ${OSRM_DEPS_DIR}")
  message(STATUS "Installing to ${OSRM_INSTALL_DIR}")

  execute_process(COMMAND git clone https://github.com/Project-OSRM/osrm-backend.git ${OSRM_DEPS_DIR})
  execute_process(COMMAND git checkout ${OSRM_VERSION} WORKING_DIRECTORY ${OSRM_DEPS_DIR})

  # These will be set by the subproject
  # LibOSRM_INCLUDE_DIRS
  # LibOSRM_LIBRARIES
  # LibOSRM_DEPENDENT_LIBRARIES
  # LibOSRM_CXXFLAGS
  set(ENABLE_MASON ON CACHE INTERNAL "Always use mason for building osrm-backend" FORCE)
  add_subdirectory(${OSRM_DEPS_DIR})
  include(${OSRM_DEPS_DIR}/third_party/mason/mason.cmake)

  # FIXME we can remove this double versioning after the merge
  set(MASON_TBB_VERSION "43_20150316")
  mason_use(tbb VERSION ${MASON_TBB_VERSION})

  # We need this because we depend on the shared libraries from TBB
  foreach(libpath ${MASON_PACKAGE_tbb_LIBRARY_DIRS})
    file(GLOB TBBGlob ${libpath}/*.*)
    foreach(filepath ${TBBGlob})
      get_filename_component(filename ${filepath} NAME)
      set(targetfile "${CMAKE_SOURCE_DIR}/lib/binding/${filename}")
      add_custom_command(OUTPUT ${targetfile}
                         COMMAND ${CMAKE_COMMAND} -E copy ${filepath} ${CMAKE_SOURCE_DIR}/lib/binding/
                         DEPENDS ${filepath})
      list(APPEND OSRM_BINARIES "${targetfile}")
    endforeach()
  endforeach()

  add_custom_command(OUTPUT ${CMAKE_SOURCE_DIR}/lib/binding/osrm-extract
                     COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:osrm-extract> ${CMAKE_SOURCE_DIR}/lib/binding/
                     DEPENDS osrm-extract)
  add_custom_command(OUTPUT ${CMAKE_SOURCE_DIR}/lib/binding/osrm-contract
                     COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:osrm-contract> ${CMAKE_SOURCE_DIR}/lib/binding/
                     DEPENDS osrm-contract)
  add_custom_command(OUTPUT ${CMAKE_SOURCE_DIR}/lib/binding/osrm-routed
                     COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:osrm-routed> ${CMAKE_SOURCE_DIR}/lib/binding/
                     DEPENDS osrm-routed)
  add_custom_command(OUTPUT ${CMAKE_SOURCE_DIR}/lib/binding/osrm-datastore
                     COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:osrm-datastore> ${CMAKE_SOURCE_DIR}/lib/binding/
                     DEPENDS osrm-datastore)
  list(APPEND OSRM_BINARIES "${CMAKE_SOURCE_DIR}/lib/binding/osrm-extract")
  list(APPEND OSRM_BINARIES "${CMAKE_SOURCE_DIR}/lib/binding/osrm-contract")
  list(APPEND OSRM_BINARIES "${CMAKE_SOURCE_DIR}/lib/binding/osrm-routed")
  list(APPEND OSRM_BINARIES "${CMAKE_SOURCE_DIR}/lib/binding/osrm-datastore")
else()
  # These will be set by the find script
  # LibOSRM_INCLUDE_DIRS
  # LibOSRM_LIBRARIES
  # LibOSRM_DEPENDENT_LIBRARIES
  # LibOSRM_CXXFLAGS
  find_package(LibOSRM REQUIRED)
endif()

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/node_modules/node-cmake")

set(NodeJS_CXX_STANDARD 14 CACHE INTERNAL "Use C++14" FORCE)
set(NodeJS_DOWNLOAD ON CACHE INTERNAL "Download node.js sources" FORCE)
set(NodeJS_USE_CLANG_STDLIB OFF CACHE BOOL "Don't use libc++ by default" FORCE)
find_package(NodeJS REQUIRED)
add_nodejs_module(node-osrm src/node_osrm.cpp)

include_directories(SYSTEM ${LibOSRM_INCLUDE_DIRS})
link_directories(${LibOSRM_LIBRARY_DIRS})
target_link_libraries(node-osrm ${LibOSRM_LIBRARIES} ${LibOSRM_DEPENDENT_LIBRARIES})
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${LibOSRM_CXXFLAGS}")

# Enforce proper rpath for osrm.node
if(APPLE)
  set(LINKER_FLAGS "-Wl,-rpath,@loader_path/")
elseif(UNIX)
  set(LINKER_FLAGS "'-Wl,-rpath,$ORIGIN' -Wl,-z,origin")
endif()
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${LINKER_FLAGS}")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${LINKER_FLAGS}")
set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} ${LINKER_FLAGS}")

add_custom_command(TARGET node-osrm POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:node-osrm> ${CMAKE_SOURCE_DIR}/lib/binding/node-osrm.node)
add_custom_target(copy_osrm_binaries ALL DEPENDS ${OSRM_BINARIES})

