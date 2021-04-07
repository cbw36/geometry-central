cmake_minimum_required(VERSION 3.8.0)

project(geometry-central VERSION 0.1.0 LANGUAGES CXX)

### Policy settings
cmake_policy(SET CMP0054 NEW)   # don't implicitly dereference inside if()


### Process settings
option(BUILD_SHARED_LIBS "Build the shared library" TRUE)
if(BUILD_SHARED_LIBS)
    message("-- Building SHARED libraries")
else()
    message("-- Building STATIC libraries")
endif()


list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake") # look for stuff in the /cmake directory
include(UpdateCacheVariable)

# Work with non-standard homebrew installations
# (from ceres build system)
if (CMAKE_SYSTEM_NAME MATCHES "Darwin")
  find_program(HOMEBREW_EXECUTABLE brew)
  mark_as_advanced(FORCE HOMEBREW_EXECUTABLE)
  if (HOMEBREW_EXECUTABLE)
    # Detected a Homebrew install, query for its install prefix.
    execute_process(COMMAND ${HOMEBREW_EXECUTABLE} --prefix
      OUTPUT_VARIABLE HOMEBREW_INSTALL_PREFIX
      OUTPUT_STRIP_TRAILING_WHITESPACE)
    message(STATUS "Detected Homebrew with install prefix: "
      "${HOMEBREW_INSTALL_PREFIX}, adding to CMake search paths.")
    list(APPEND CMAKE_PREFIX_PATH "${HOMEBREW_INSTALL_PREFIX}")
  endif()
endif()


### Handle windows-specific fixes
if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
  add_definitions (-DNOMINMAX)  # don't use weird windows built-in min/max
  add_definitions (-D_USE_MATH_DEFINES)  # match unix behavior of constants in cmath
endif()

### Do anything needed for dependencies and bring their stuff in to scope
add_subdirectory(deps)

# copy variables set by deps upward
SET(GC_HAVE_SUITESPARSE ${GC_HAVE_SUITESPARSE} PARENT_SCOPE)

### Recurse to the source code
add_subdirectory(src)

### Install

# Mark cpp header files for installation
install(DIRECTORY include/geometrycentral DESTINATION include)
install(FILES deps/happly/happly.h DESTINATION include)

install(TARGETS ${PROJECT_NAME} eigen nanort nanoflann happly
        EXPORT ${PROJECT_NAME}-targets
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib)

install(EXPORT ${PROJECT_NAME}-targets NAMESPACE "${PROJECT_NAME}::" DESTINATION lib/cmake/${PROJECT_NAME})

# Create cmake config files
include(CMakePackageConfigHelpers)
configure_package_config_file(${CMAKE_CURRENT_LIST_DIR}/cmake/${PROJECT_NAME}-config.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake
  INSTALL_DESTINATION lib/cmake/${PROJECT_NAME}
  NO_CHECK_REQUIRED_COMPONENTS_MACRO)

write_basic_package_version_file(${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake
  VERSION ${PROJECT_VERSION} COMPATIBILITY ExactVersion)

install(FILES
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake"
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake"
  DESTINATION lib/cmake/${PROJECT_NAME})

export(EXPORT ${PROJECT_NAME}-targets NAMESPACE "${PROJECT_NAME}::" FILE ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-targets.cmake)

install(FILES package.xml DESTINATION share/${PROJECT_NAME})
