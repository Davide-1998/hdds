cmake_minimum_required(VERSION 3.14 FATAL_ERROR)
project(hdds
    VERSION 1.0.11
    DESCRIPTION "High-performance DDS"
    LANGUAGES C CXX
)

option(BUILD_SHARED_LIBS "Build static or shared library" OFF)
option(BUILD_HDDS_CXX "Enable building hdds_cxx library" ON)
option(HDDS_BUILD_INSTALL "Build install target" OFF)

# Set CMAKE_BUILD_TYPE if neither parent nor project set it or if we are using
# a multi-config generator
if(NOT CMAKE_BUILD_TYPE OR NOT GENERATOR_IS_MULTI_CONFIG)
  set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Build type for the library" FORCE)
endif()

# Ensure cargo is installed
set(CARGO_EXECUTABLE)
find_program(CARGO_EXECUTABLE NAMES cargo)
if("${CARGO_EXECUTABLE}" STREQUAL "CARGO_EXECUTABLE-NOTFOUND")
  message(FATAL_ERROR "Cargo not found. Look at: https://rust-lang.org/learn/get-started")
endif()

# Determine library extensions and suffixes for the current OS
if(WIN32)
  set(HDDS_SHARED_LIBRARY_EXTENSION "dll")
  set(HDDS_STATIC_LIBRARY_EXTENSION "lib")
  set(HDDS_LIBRARY_PREFIX "")
else()
  set(HDDS_SHARED_LIBRARY_EXTENSION "so")
  set(HDDS_STATIC_LIBRARY_EXTENSION "a")
  set(HDDS_LIBRARY_PREFIX "lib")
endif()
set(HDDS_C_LIBRARY_NAME ${HDDS_LIBRARY_PREFIX}hdds_c)

# Use the build type to know the folder in which the library will be built
if(${CMAKE_BUILD_TYPE} STREQUAL "Release" OR ${CMAKE_BUILD_TYPE} STREQUAL "Debug")
string(TOLOWER ${CMAKE_BUILD_TYPE} HDDS_BUILD_TYPE)
else()
    set(HDDS_BUILD_TYPE "release")
endif()

set(HDDS_CMAKE_LIBRARY_TYPE "STATIC")
if(BUILD_SHARED_LIBS)
  set(HDDS_CMAKE_LIBRARY_TYPE "SHARED")
endif()

# Set the path to the library binary file depending on the build type
set(HDDS_OUTPUT_DIR "${CMAKE_CURRENT_BINARY_DIR}/${HDDS_BUILD_TYPE}")
if(BUILD_SHARED_LIBS)
  set(HDDS_LIBRARY_OUTPUT ${HDDS_OUTPUT_DIR}/${HDDS_C_LIBRARY_NAME}.${HDDS_SHARED_LIBRARY_EXTENSION})
else()
  set(HDDS_LIBRARY_OUTPUT ${HDDS_OUTPUT_DIR}/${HDDS_C_LIBRARY_NAME}.${HDDS_STATIC_LIBRARY_EXTENSION})
endif()

# Set location of library headers
set(HDDS_C_INCLUDE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/sdk/c/include")
set(HDDS_C_HEADERS "${HDDS_C_INCLUDE_DIR}/hdds.h")

# Set build flag for cargo only if building in Release
if(${HDDS_BUILD_TYPE} STREQUAL "release")
    set(HDDS_CARGO_RELEASE_BUILD_FLAG "--${HDDS_BUILD_TYPE}")
endif()

# Add a custom command to build the library and a custom target that depends on the output of the command
# that will triger the cargo build task
add_custom_target(${PROJECT_NAME}_rust ALL
  ${CARGO_EXECUTABLE} "rustc" "--manifest-path" "${CMAKE_CURRENT_SOURCE_DIR}/Cargo.toml"
                      "--package" "hdds-c" "${HDDS_CARGO_RELEASE_BUILD_FLAG}"
                      "--crate-type" $<IF:$<BOOL:${BUILD_SHARED_LIBS}>,"cdylib","staticlib">
                      "--target-dir" "${CMAKE_CURRENT_BINARY_DIR}"
  DEPENDS ${HDDS_C_HEADERS} # CMake will re-run the command whenever the files change
)

# Generate the imported library for HDDS and link it to the custom target to ensure the custom command to be called
add_library(${PROJECT_NAME}_c ${HDDS_CMAKE_LIBRARY_TYPE} IMPORTED GLOBAL ${HDDS_C_HEADERS})
add_library(${PROJECT_NAME}::${PROJECT_NAME}_c ALIAS ${PROJECT_NAME}_c)

if (WIN32)
  # Link Windows libraries
  target_link_libraries(${PROJECT_NAME}_c INTERFACE ntdll kernel32 ws2_32 userenv iphlpapi bcrypt)
endif()

# Manually specify the dependency between HDDS target and the fake one to ensure compiler will not invert the order
add_dependencies(${PROJECT_NAME}_c ${PROJECT_NAME}_rust)

# Change imported library location to the built one
set(HDDS_LIBRARY_IMPORTED_LOCATION ${HDDS_LIBRARY_OUTPUT})

# Set imported target library location and implib.
# On Windows if the library is static then both properties
# must be the same .lib file. On Linux the 'IMPORTED_IMPLIB' will be ignored
set_target_properties(
  ${PROJECT_NAME}_c
  PROPERTIES IMPORTED_LOCATION ${HDDS_LIBRARY_IMPORTED_LOCATION}
             IMPORTED_IMPLIB ${HDDS_LIBRARY_OUTPUT}.${HDDS_STATIC_LIBRARY_EXTENSION}
)

target_include_directories(${PROJECT_NAME}_c INTERFACE 
  "${HDDS_C_INCLUDE_DIR}" # Allows inclusion as '#include <hdds.h>'
)

# Expose also hdds_cxx target
if(BUILD_HDDS_CXX)
  add_subdirectory("${CMAKE_CURRENT_SOURCE_DIR}/sdk/cxx")
endif()

if(HDDS_BUILD_INSTALL)
  include(CMakePackageConfigHelpers)

  get_filename_component(file ${HDDS_C_HEADERS} ABSOLUTE)
  install(FILES ${file} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

  # Configuration
  configure_package_config_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/${PROJECT_NAME}Config.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
  )
  write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
  )
  # Package
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
                ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
          DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
  )
  install(
    EXPORT ${PROJECT_NAME}Targets
    FILE ${PROJECT_NAME}Targets.cmake
    NAMESPACE ${PROJECT_NAME}::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
  )
  install(
    TARGETS ${PROJECT_NAME}
    EXPORT ${PROJECT_NAME}Targets
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  )

  #
  # CPack
  #
  set(CPACK_GENERATOR DEB RPM TXZ TGZ)
  set(CPACK_PACKAGE_VERSION ${PROJECT_VERSION})
  set(CPACK_PACKAGE_NAME ${PROJECT_NAME})
  set(CPACK_PACKAGE_DESCRIPTION_SUMMARY ${PROJECT_DESCRIPTION})
  set(CPACK_PACKAGE_FILE_NAME
      "${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}.${CMAKE_SYSTEM_PROCESSOR}"
  )
  set(CPACK_PACKAGE_RPM_DIR "${CMAKE_CURRENT_BINARY_DIR}/_CPack_Packages/Linux/RPM")
  include(CPack)
endif()