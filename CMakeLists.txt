cmake_minimum_required(VERSION 3.18)
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

project(pybind11_protobuf)

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

if(MSVC)
  set(CMAKE_CXX_STANDARD 20)
else()
  set(CMAKE_CXX_STANDARD 17)
endif()
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# ============================================================================
# Options

option(BUILD_SHARED_LIBS "Build shared libraries" ON)
option(BUILD_TESTS "Build tests." OFF)

# ============================================================================
# Find Python

find_package(Python COMPONENTS Interpreter Development.Module)

# ============================================================================
# Build dependencies

set(_absl_repository "https://github.com/abseil/abseil-cpp.git")
set(_absl_version 20230125)
set(_absl_tag 20230125.3)
find_package(absl ${_absl_version} QUIET)

set(_protobuf_repository "https://github.com/protocolbuffers/protobuf.git")
set(_protobuf_version 3.23.3)
set(_protobuf_tag v23.3)
find_package(Protobuf ${_protobuf_version} QUIET)

set(_pybind11_repository "https://github.com/pybind/pybind11.git")
set(_pybind11_version 2.11.1)
set(_pybind11_tag v2.11.1)
find_package(pybind11 ${_pybind11_version} QUIET)

add_subdirectory(cmake/dependencies dependencies)

# ============================================================================
# pybind11_proto_utils shared library
add_library(pybind11_proto_utils
  pybind11_protobuf/proto_utils.cc
  pybind11_protobuf/proto_utils.h)

target_link_libraries(
  pybind11_proto_utils
  PRIVATE absl::strings
  PUBLIC pybind11::pybind11
         protobuf::libprotobuf)

set_property(
  TARGET pybind11_proto_utils
  PROPERTY EXPORT_NAME proto_utils)

target_include_directories(
  pybind11_proto_utils
  PUBLIC $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}>
         $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)

# ============================================================================
# pybind11_native_proto_caster shared library
add_library(
  pybind11_native_proto_caster
  # bazel: pybind_library: native_proto_caster
  pybind11_protobuf/native_proto_caster.h
  # bazel: pybind_library: enum_type_caster
  pybind11_protobuf/enum_type_caster.h
  # bazel: pybind_library: proto_cast_util
  pybind11_protobuf/proto_cast_util.cc
  pybind11_protobuf/proto_cast_util.h
  pybind11_protobuf/proto_caster_impl.h
  # bazel: cc_library::check_unknown_fields
  pybind11_protobuf/check_unknown_fields.cc
  pybind11_protobuf/check_unknown_fields.h)

set_property(
  TARGET pybind11_native_proto_caster
  PROPERTY EXPORT_NAME native_proto_caster)

target_link_libraries(
  pybind11_native_proto_caster
  PRIVATE absl::flat_hash_map
          absl::flat_hash_set
          absl::hash
          absl::synchronization
  PUBLIC absl::optional
         absl::strings
         protobuf::libprotobuf
         pybind11::pybind11
         Python::Module)

target_include_directories(
  pybind11_native_proto_caster
  PUBLIC $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}>
         $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
         ${protobuf_SOURCE_DIR})

# ============================================================================
# pybind11_wrapped_proto_caster shared library
add_library(
  pybind11_wrapped_proto_caster
  # bazel: pybind_library: wrapped_proto_caster
  pybind11_protobuf/wrapped_proto_caster.h
  # bazel: pybind_library: proto_cast_util
  pybind11_protobuf/proto_cast_util.cc
  pybind11_protobuf/proto_cast_util.h
  pybind11_protobuf/proto_caster_impl.h
  # bazel: cc_library: check_unknown_fields
  pybind11_protobuf/check_unknown_fields.cc
  pybind11_protobuf/check_unknown_fields.h)

set_property(
  TARGET pybind11_wrapped_proto_caster
  PROPERTY EXPORT_NAME wrapped_proto_caster)

target_link_libraries(
  pybind11_wrapped_proto_caster
  PRIVATE absl::flat_hash_map
          absl::flat_hash_set
          absl::hash
          absl::synchronization
  PUBLIC absl::optional
         absl::statusor
         absl::strings
         protobuf::libprotobuf
         pybind11::pybind11
         Python::Module)

target_include_directories(
  pybind11_wrapped_proto_caster
  PUBLIC $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}>
         $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
         ${protobuf_SOURCE_DIR})

install(
  TARGETS pybind11_proto_utils
          pybind11_native_proto_caster
          pybind11_wrapped_proto_caster
  EXPORT pybind11_protobuf-targets)

install(
  DIRECTORY pybind11_protobuf
  DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
  FILES_MATCHING PATTERN "*.h")

install(
  EXPORT pybind11_protobuf-targets
  NAMESPACE pybind11_protobuf::
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/pybind11_protobuf")

configure_package_config_file(
  cmake/pybind11_protobuf-config.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/pybind11_protobuf-config.cmake"
  INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/pybind11_protobuf")

install(
  FILES "${CMAKE_CURRENT_BINARY_DIR}/pybind11_protobuf-config.cmake"
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/pybind11_protobuf")

# TODO set defines PYBIND11_PROTOBUF_ENABLE_PYPROTO_API see: bazel:
# pybind_library: proto_cast_util

# bazel equivs. checklist
#
# bazel: pybind_library: enum_type_caster - enum_type_caster.h
#
# bazel: pybind_library: native_proto_caster - native_proto_caster.h
#
# check_unknown_fields enum_type_caster proto_cast_util
#
# bazel: pybind_library: proto_cast_util - proto_cast_util.cc -
# proto_cast_util.h - proto_caster_impl.h
#
# check_unknown_fields
#
# bazel: pybind_library: wrapped_proto_caster - wrapped_proto_caster.h
#
# proto_cast_util
#
