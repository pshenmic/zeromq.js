cmake_minimum_required(VERSION 3.16)

# Add project_options v0.25.0 https://github.com/aminya/project_options Change
# the version in the following URL to update the package (watch the releases of
# the repository for future updates)
include(FetchContent)
FetchContent_Declare(
  _project_options
  URL https://github.com/aminya/project_options/archive/refs/tags/v0.25.0.zip)
FetchContent_MakeAvailable(_project_options)
include(${_project_options_SOURCE_DIR}/Index.cmake)

# MacOS flags that should be set prior to any project calls
if(DEFINED APPLE AND APPLE)
  set(CMAKE_SHARED_LINKER_FLAGS
      "${CMAKE_SHARED_LINKER_FLAGS} -undefined dynamic_lookup")
  set(CMAKE_OSX_DEPLOYMENT_TARGET
      "10.15"
      CACHE STRING "Minimum OS X deployment version")
  add_definitions(-DMAC_OS_X_VERSION_MIN_REQUIRED=101500)
endif()

run_vcpkg(VCPKG_URL "https://github.com/microsoft/vcpkg.git" VCPKG_REV
          "927006b24c3a28dfd8aa0ec5f8ce43098480a7f1" ENABLE_VCPKG_UPDATE)

# Name of the project (will be the name of the plugin)
project(addon LANGUAGES C CXX)

project_options(ENABLE_CACHE)

add_library(
  addon SHARED
  "src/context.cc"
  "src/incoming_msg.cc"
  "src/module.cc"
  "src/observer.cc"
  "src/outgoing_msg.cc"
  "src/proxy.cc"
  "src/socket.cc")

target_link_libraries(addon project_options project_warnings)

# Dependencies
set(ZMQ_VERSION "4.3.5")
FetchContent_Declare(
  _zeromq
  URL "https://github.com/aminya/libzmq/archive/360e26116f62beb8bc8eb47d0be791e2e3a8fdad.tar.gz"
)
FetchContent_GetProperties(_zeromq)
if(NOT _zeromq_POPULATED)
  FetchContent_Populate(_zeromq)

  find_package(unofficial-sodium CONFIG REQUIRED)
  set(SODIUM_FOUND TRUE)
  set(SODIUM_DIR "${unofficial-sodium_DIR}")
  set(SODIUM_INCLUDE_DIR
      "${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/include/")
  set(SODIUM_INCLUDE_DIRS "${sodium_INCLUDE_DIR}")
  set(SODIUM_LIBRARY_DEBUG
      "${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/lib/libsodium${CMAKE_STATIC_LIBRARY_SUFFIX}"
  )
  set(SODIUM_LIBRARY_RELEASE
      "${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/debug/lib/libsodium${CMAKE_STATIC_LIBRARY_SUFFIX}"
  )
  set(SODIUM_STATIC "${SODIUM_LIBRARY_RELEASE}")
  set(SODIUM_DLL_DEBUG
      "${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/debug/bin/libsodium${CMAKE_SHARED_LIBRARY_SUFFIX}"
  )
  set(SODIUM_DLL_RELEASE
      "${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/bin/libsodium${CMAKE_SHARED_LIBRARY_SUFFIX}"
  )
  set(SODIUM_DIR "${unofficial-sodium_DIR}")

  execute_process(
    COMMAND
      ${CMAKE_EXECUTABLE}
      # paths
      -S "${_zeromq_SOURCE_DIR}" -B "${_zeromq_BINARY_DIR}"
      -DCMAKE_INSTALL_PREFIX="${CMAKE_CURRENT_BINARY_DIR}/libzmq"
      -DCMAKE_INSTALL_LIBDIR=lib
      # options
      -DBUILD_STATIC=ON -DBUILD_TESTS=OFF -DBUILD_SHARED=OFF -DWITH_DOCS=OFF
      # dependencies
      -Dsodium_FOUND="${sodium_FOUND}" -Dsodium_DIR="${sodium_DIR}"
      -Dsodium_INCLUDE_DIR="${sodium_INCLUDE_DIR}"
      -Dsodium_LIBRARY_DEBUG="${sodium_LIBRARY_DEBUG}"
      -Dsodium_LIBRARY_RELEASE="${sodium_LIBRARY_RELEASE}"
      -Dsodium_DLL_DEBUG="${sodium_DLL_DEBUG}"
      -Dsodium_DLL_RELEASE="${sodium_DLL_RELEASE}"
      # use same cmake settings as the current
      -G ${CMAKE_GENERATOR} -DCMAKE_MAKE_PROGRAM=${CMAKE_MAKE_PROGRAM}
      -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
      -DCMAKE_CXX_STANDARD=${CMAKE_CXX_STANDARD}
      -DCMAKE_C_STANDARD=${CMAKE_C_STANDARD}
      -DCMAKE_PREFIX_PATH=${CMAKE_PREFIX_PATH}
    COMMAND
      ${CMAKE_EXECUTABLE} --build "${_zeromq_BINARY_DIR}" --config
      ${CMAKE_BUILD_TYPE} --target install
      # exec options
      COMMAND_ERROR_IS_FATAL ANY)

endif()

find_library(zeromq zeromq)
target_include_directories(addon PRIVATE libzmq/include)

# Node specific
target_include_directories(addon PRIVATE ${CMAKE_JS_INC})
target_link_libraries(addon ${CMAKE_JS_LIB})

target_compile_definitions(addon PUBLIC V8_COMPRESS_POINTERS)
target_compile_definitions(addon PUBLIC V8_31BIT_SMIS_ON_64BIT_ARCH)
target_compile_definitions(addon PUBLIC V8_REVERSE_JSARGS)
target_compile_definitions(addon PUBLIC BUILDING_NODE_EXTENSION)
target_compile_definitions(addon PUBLIC NAPI_CPP_EXCEPTIONS)

# if(WIN32) target_compile_definitions(addon PUBLIC "NOMINMAX")
# target_compile_definitions(addon PUBLIC "NOGDI")
# target_compile_definitions(addon PUBLIC "WIN32_LEAN_AND_MEAN") endif()

# Use `.node` for the library without any "lib" prefix
set_target_properties(addon PROPERTIES PREFIX "" SUFFIX ".node")

# Windows workaround
if(WIN32)
  set_property(TARGET addon PROPERTY LINK_FLAGS "-Xlinker /DELAYLOAD:NODE.EXE")
  target_link_libraries(addon "ShLwApi.lib" "delayimp.lib")
endif()
