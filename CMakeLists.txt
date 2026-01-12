cmake_minimum_required(VERSION 3.25)

project(slick-object-pool
    VERSION 0.1.3
    LANGUAGES CXX
)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Add header-only library
add_library(slick-object-pool INTERFACE)
add_library(slick::object-pool ALIAS slick-object-pool)
# For backward compatibility
add_library(slick_object_pool ALIAS slick-object-pool)
add_library(slick::slick_object_pool ALIAS slick-object-pool)

target_include_directories(slick-object-pool INTERFACE 
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)

set_target_properties(slick-object-pool PROPERTIES EXPORT_NAME object-pool)

if(UNIX AND NOT APPLE)
    target_link_libraries(slick-object-pool INTERFACE rt atomic)
endif()

option(BUILD_SLICK_OBJECTPOOL_TESTS "Build tests" ON)
if(BUILD_SLICK_OBJECTPOOL_TESTS)
    enable_testing()
    if (WIN32)
        add_subdirectory(tests)
    else()
        add_subdirectory(tests EXCLUDE_FROM_ALL)
    endif()
endif()

# Installation rules
install(TARGETS slick-object-pool
    EXPORT slick-object-pool-targets
    INCLUDES DESTINATION include
)

install(DIRECTORY include/ DESTINATION include)

# Install CMake config files for find_package support
include(CMakePackageConfigHelpers)

write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/slick-object-pool-config-version.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/slick-object-pool-config.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/slick-object-pool-config.cmake"
    INSTALL_DESTINATION lib/cmake/slick-object-pool
)

install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/slick-object-pool-config.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/slick-object-pool-config-version.cmake"
    DESTINATION lib/cmake/slick-object-pool
)

install(EXPORT slick-object-pool-targets
    FILE slick-object-pool-targets.cmake
    NAMESPACE slick::
    DESTINATION lib/cmake/slick-object-pool
)

message(STATUS "slick-object-pool: ${PROJECT_VERSION}")