cmake_minimum_required(VERSION 3.25)

# Note: We check CMAKE_BUILD_TYPE STREQUAL "Debug" and fallback to optimized build
# This matches gyp's 'default_configuration': 'Release' behavior

find_package(cmake-bare REQUIRED PATHS node_modules/cmake-bare)
find_package(cmake-napi REQUIRED PATHS node_modules/cmake-napi)

project(sqlcipher C CXX)

bare_target(target)

# Note: OpenSSL not needed - using Signal's custom crypto provider from Rust extension

# Skip bare module for now - focus on Node.js build first
# TODO: Research the correct plain C N-API approach for bare compatibility
# add_bare_module(signalapp__sqlcipher_bare)

# Determine Rust target triple based on platform and architecture
if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  if(CMAKE_SYSTEM_PROCESSOR STREQUAL "arm64")
    set(RUST_TARGET "aarch64-apple-darwin")
  else()
    set(RUST_TARGET "x86_64-apple-darwin")
  endif()
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
  if(CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "arm64")
    set(RUST_TARGET "aarch64-unknown-linux-gnu")
  else()
    set(RUST_TARGET "x86_64-unknown-linux-gnu")
  endif()
elseif(CMAKE_SYSTEM_NAME STREQUAL "Windows")
  if(CMAKE_SYSTEM_PROCESSOR STREQUAL "ARM64")
    set(RUST_TARGET "aarch64-pc-windows-msvc")
  else()
    set(RUST_TARGET "x86_64-pc-windows-msvc")
  endif()
elseif(CMAKE_SYSTEM_NAME STREQUAL "Android")
  if(CMAKE_ANDROID_ARCH_ABI STREQUAL "arm64-v8a")
    set(RUST_TARGET "aarch64-linux-android")
  elseif(CMAKE_ANDROID_ARCH_ABI STREQUAL "x86_64")
    set(RUST_TARGET "x86_64-linux-android")
  elseif(CMAKE_ANDROID_ARCH_ABI STREQUAL "armeabi-v7a")
    set(RUST_TARGET "armv7-linux-androideabi")
  else()
    message(FATAL_ERROR "Unsupported Android architecture: ${CMAKE_ANDROID_ARCH_ABI}")
  endif()
elseif(CMAKE_SYSTEM_NAME STREQUAL "iOS")
  if(CMAKE_OSX_ARCHITECTURES STREQUAL "arm64")
    set(RUST_TARGET "aarch64-apple-ios")
  elseif(CMAKE_OSX_ARCHITECTURES STREQUAL "x86_64")
    set(RUST_TARGET "x86_64-apple-ios")
  else()
    message(FATAL_ERROR "Unsupported iOS architecture: ${CMAKE_OSX_ARCHITECTURES}")
  endif()
else()
  message(FATAL_ERROR "Unsupported platform: ${CMAKE_SYSTEM_NAME}")
endif()

# Set library prefix and extension based on platform
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
  set(RUST_LIB_PREFIX "")
  set(RUST_LIB_EXT "lib")
else()
  set(RUST_LIB_PREFIX "lib")
  set(RUST_LIB_EXT "a")
endif()

# Build Signal Rust extension
set(EXTENSION_LIB_PATH ${CMAKE_CURRENT_SOURCE_DIR}/deps/extension/target/${RUST_TARGET}/release/${RUST_LIB_PREFIX}signal_sqlcipher_extension.${RUST_LIB_EXT})
set(EXTENSION_HEADER_PATH ${CMAKE_CURRENT_SOURCE_DIR}/deps/extension/target/signal-tokenizer.h)

add_custom_command(
  OUTPUT ${EXTENSION_LIB_PATH} ${EXTENSION_HEADER_PATH}
  COMMAND ${CMAKE_COMMAND} -E env
    MACOSX_DEPLOYMENT_TARGET=11.0
    $<$<NOT:$<PLATFORM_ID:Windows>>:CFLAGS=-Wa,--noexecstack>
    $<$<OR:$<STREQUAL:${RUST_TARGET},aarch64-apple-darwin>,$<STREQUAL:${RUST_TARGET},aarch64-unknown-linux-gnu>>:RUSTFLAGS=--cfg\ aes_armv8>
    node cargo-wrap.js target/${RUST_TARGET}
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/deps/extension
  COMMENT "Building Signal Rust extension for target: ${RUST_TARGET}"
)

# Create custom target that depends on the Rust build
add_custom_target(build_signal_extension DEPENDS ${EXTENSION_LIB_PATH} ${EXTENSION_HEADER_PATH})

# Create imported library for the Rust extension
add_library(signal_extension STATIC IMPORTED)
set_target_properties(signal_extension PROPERTIES
  IMPORTED_LOCATION ${EXTENSION_LIB_PATH}
)
add_dependencies(signal_extension build_signal_extension)

# Create Node.js module
add_napi_module(signalapp__sqlcipher_node)

# Ensure Signal extension header is built before compiling C++ sources
add_dependencies(${signalapp__sqlcipher_node} build_signal_extension)

# SQLCipher source files
target_sources(
  ${signalapp__sqlcipher_node}
  PRIVATE
    src/addon.cc
    deps/sqlcipher/sqlite3.c
)

# Include directories
target_include_directories(
  ${signalapp__sqlcipher_node}
  PRIVATE
    src
    deps/sqlcipher
    "${CMAKE_CURRENT_SOURCE_DIR}/node_modules/node-addon-api"
    ${CMAKE_CURRENT_SOURCE_DIR}/deps/extension/target  # Signal extension header
)

# Link Signal extension (provides crypto via custom provider)
target_link_libraries(
  ${signalapp__sqlcipher_node}
  PRIVATE
    signal_extension
)

# SQLite/SQLCipher compiler definitions (from deps/sqlcipher/sqlcipher.gyp)
target_compile_definitions(
  ${signalapp__sqlcipher_node}
  PRIVATE
    SQLITE_LIKE_DOESNT_MATCH_BLOBS
    SQLITE_THREADSAFE=2
    SQLITE_USE_URI=0
    SQLITE_DEFAULT_MEMSTATUS=0
    SQLITE_OMIT_AUTOINIT
    SQLITE_OMIT_DEPRECATED
    SQLITE_OMIT_DESERIALIZE
    SQLITE_OMIT_GET_TABLE
    SQLITE_OMIT_TCL_VARIABLE
    SQLITE_OMIT_PROGRESS_CALLBACK
    SQLITE_OMIT_SHARED_CACHE
    SQLITE_OMIT_UTF16
    SQLITE_OMIT_COMPLETE
    SQLITE_OMIT_AUTHORIZATION
    SQLITE_OMIT_LOAD_EXTENSION
    SQLITE_TRACE_SIZE_LIMIT=32
    SQLITE_DEFAULT_CACHE_SIZE=-16000
    SQLITE_DEFAULT_FOREIGN_KEYS=1
    SQLITE_DEFAULT_WAL_SYNCHRONOUS=1
    SQLITE_DQS=0
    SQLITE_ENABLE_MATH_FUNCTIONS
    SQLITE_ENABLE_DESERIALIZE
    SQLITE_ENABLE_COLUMN_METADATA
    SQLITE_ENABLE_UPDATE_DELETE_LIMIT
    SQLITE_ENABLE_STAT4
    SQLITE_ENABLE_FTS5
    SQLITE_ENABLE_JSON1
    SQLITE_INTROSPECTION_PRAGMAS
    HAVE_STDINT_H=1
    HAVE_INT8_T=1
    HAVE_INT16_T=1
    HAVE_INT32_T=1
    HAVE_UINT8_T=1
    HAVE_UINT16_T=1
    HAVE_UINT32_T=1
    SQLITE_HAS_CODEC
    SQLITE_TEMP_STORE=2
    SQLITE_SECURE_DELETE
    SQLITE_EXTRA_INIT=sqlcipher_extra_init
    SQLITE_EXTRA_SHUTDOWN=sqlcipher_extra_shutdown
    SQLCIPHER_CRYPTO_CUSTOM=signal_crypto_provider_setup
    NAPI_VERSION=9
    NAPI_DISABLE_CPP_EXCEPTIONS
)

# Base compiler flags (from deps/sqlcipher/sqlcipher.gyp)
target_compile_options(
  ${signalapp__sqlcipher_node}
  PRIVATE
    $<$<COMPILE_LANGUAGE:C>:-std=c99>
    $<$<COMPILE_LANGUAGE:C>:-w>
)

# Build type specific configurations
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
  # Debug build settings (from common.gypi Debug configuration)
  target_compile_definitions(
    ${signalapp__sqlcipher_node}
    PRIVATE
      DEBUG
      _DEBUG
      SQLITE_DEBUG
      SQLITE_MEMDEBUG
      SQLITE_ENABLE_API_ARMOR
      SQLITE_ENABLE_STMT_SCANSTATUS
  )

  target_compile_options(
    ${signalapp__sqlcipher_node}
    PRIVATE
      $<$<COMPILE_LANGUAGE:C>:-O0>
      $<$<COMPILE_LANGUAGE:CXX>:-O0>
  )

else()
  # Optimized build settings (from common.gypi Release configuration)
  target_compile_definitions(
    ${signalapp__sqlcipher_node}
    PRIVATE
      NDEBUG
  )
endif()

# Platform-specific configurations
if(WIN32)
  # Windows-specific settings (from common.gypi and sqlcipher.gyp)
  target_compile_definitions(
    ${signalapp__sqlcipher_node}
    PRIVATE
      WIN32
  )

  # MSVC runtime library settings (from common.gypi configurations)
  if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set_target_properties(${signalapp__sqlcipher_node} PROPERTIES
      MSVC_RUNTIME_LIBRARY "MultiThreadedDebug"
    )
    target_compile_definitions(
      ${signalapp__sqlcipher_node}
      PRIVATE
        SQLITE_WIN32_MALLOC_VALIDATE
    )
  else()
    set_target_properties(${signalapp__sqlcipher_node} PROPERTIES
      MSVC_RUNTIME_LIBRARY "MultiThreaded"
    )
  endif()

  # MSVC exception handling for Node.js module (from common.gypi VCCLCompilerTool)
  if(MSVC)
    target_compile_options(
      ${signalapp__sqlcipher_node}
      PRIVATE
        /EHsc  # Enable C++ exception handling (ExceptionHandling: 1)
    )
  endif()

elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
  # Linux-specific settings (from binding.gyp and common.gypi)
  target_link_options(
    ${signalapp__sqlcipher_node}
    PRIVATE
      -Wl,-Bsymbolic
      -Wl,--exclude-libs,ALL
  )

  if(NOT CMAKE_BUILD_TYPE STREQUAL "Debug")
    # LTO optimizations for GCC (from common.gypi Release + Linux conditions)
    target_compile_options(
      ${signalapp__sqlcipher_node}
      PRIVATE
        $<$<COMPILE_LANGUAGE:C>:-flto>
        $<$<COMPILE_LANGUAGE:CXX>:-flto>
        $<$<COMPILE_LANGUAGE:C>:-fuse-linker-plugin>
        $<$<COMPILE_LANGUAGE:CXX>:-fuse-linker-plugin>
        $<$<COMPILE_LANGUAGE:C>:-ffat-lto-objects>
        $<$<COMPILE_LANGUAGE:CXX>:-ffat-lto-objects>
    )
    target_link_options(
      ${signalapp__sqlcipher_node}
      PRIVATE
        -flto
        -fuse-linker-plugin
        -ffat-lto-objects
    )
  endif()

elseif(APPLE)
  # macOS-specific settings (from common.gypi xcode_settings)
  target_link_libraries(
    ${signalapp__sqlcipher_node}
    PRIVATE
      "-stdlib=libc++"
  )

  set_target_properties(${signalapp__sqlcipher_node} PROPERTIES
    MACOSX_DEPLOYMENT_TARGET "11.0"
  )

  # Always disable C++ exceptions on macOS (GCC_ENABLE_CPP_EXCEPTIONS: NO)
  target_compile_options(
    ${signalapp__sqlcipher_node}
    PRIVATE
      $<$<COMPILE_LANGUAGE:CXX>:-fno-exceptions>
  )

  if(NOT CMAKE_BUILD_TYPE STREQUAL "Debug")
    # Release optimizations (from common.gypi xcode_settings Release)
    target_compile_options(
      ${signalapp__sqlcipher_node}
      PRIVATE
        $<$<COMPILE_LANGUAGE:CXX>:-fvisibility=hidden>
        $<$<COMPILE_LANGUAGE:CXX>:-fvisibility-inlines-hidden>
    )

    # Enable LTO (LLVM_LTO: YES)
    set_target_properties(${signalapp__sqlcipher_node} PROPERTIES
      INTERPROCEDURAL_OPTIMIZATION TRUE
    )
  endif()
endif()

# Check for profiling builds (matching SQLCipher gyp profiling detection)
if(CMAKE_PROJECT_VERSION AND CMAKE_PROJECT_VERSION MATCHES "profiling")
  target_compile_definitions(
    ${signalapp__sqlcipher_node}
    PRIVATE
      SQLITE_ENABLE_STMT_SCANSTATUS
  )
endif()


# TODO: add optional dump_syms integration for release builds to generate .sym
# files for debugging/crash reporting.
