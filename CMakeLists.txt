cmake_minimum_required(VERSION 3.0)
set(VS ValuStor)
project("${VS}" LANGUAGES CXX)
set("${VS}_VERSION_MAJOR" 1)
set("${VS}_VERSION_MINOR" 0)

#
# Configure header-only target
#
add_library("${VS}" INTERFACE
  ValuStor.hpp
  )

add_library(Sensaphone::"${VS}" ALIAS "${VS}")

target_compile_features("${VS}"
  INTERFACE
    cxx_variadic_templates
    cxx_lambdas
    cxx_nullptr
    cxx_rvalue_references
    cxx_explicit_conversions
    cxx_defaulted_functions
  )

target_include_directories("${VS}"
  INTERFACE
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>"
    "$<INSTALL_INTERFACE:include>"
  )

find_library(CASS_DRIVER cassandra)
if ("${CASS_DRIVER}" STREQUAL "")
  message(FATAL_ERROR "Cassandra C++ client library could not be found.")
endif()

target_link_libraries("${VS}"
  INTERFACE
    "${CASS_DRIVER}"
  )

#
# Configure installation
#
set(VS_CONFIG "${VS}Config")
include(GNUInstallDirs)

# install location for header file
install(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
  DESTINATION "include"
  FILES_MATCHING PATTERN "*.hpp"
  )

# make the target importable from the install directory
install(EXPORT "${VS_CONFIG}"
  FILE "${VS_CONFIG}.cmake"
  NAMESPACE Sensaphone::
  DESTINATION "share/${VS}/cmake"
  )

# make the target importable from the build directory
export(TARGETS "${VS}"
  FILE "${VS_CONFIG}.cmake"
  )
