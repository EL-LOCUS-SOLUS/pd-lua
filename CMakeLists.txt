cmake_minimum_required(VERSION 3.15)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# check if inside some project (for pd-else)
if(CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
  project(pdlua C)
endif()

set(PDCMAKE_FILE ${CMAKE_BINARY_DIR}/pd.cmake)
set(PDCMAKE_VERSION "v0.2.8")
if(NOT EXISTS "${PDCMAKE_FILE}")
  file(
    DOWNLOAD
    https://raw.githubusercontent.com/pure-data/pd.cmake/refs/tags/${PDCMAKE_VERSION}/pd.cmake
    ${PDCMAKE_FILE})
endif()
include(${PDCMAKE_FILE})

execute_process(
  COMMAND git describe --tags
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  OUTPUT_VARIABLE PDLUA_VERSION
  OUTPUT_STRIP_TRAILING_WHITESPACE
  ERROR_QUIET)
if(NOT PDLUA_VERSION)
  set(PDLUA_VERSION "unknown")
endif()

set(LUAJIT_DIR "${CMAKE_CURRENT_SOURCE_DIR}/luas/luajit/src")
set(LUAJIT_LIB "${LUAJIT_DIR}/libluajit.a")

if(NOT TARGET luajit_build)
  find_program(MAKE_EXE NAMES gmake make mingw32-make REQUIRED)

  if(WIN32)
    add_custom_command(
      OUTPUT  ${LUAJIT_LIB}
      COMMAND ${MAKE_EXE} BUILDMODE=static
              XCFLAGS=-DLUAJIT_ENABLE_LUA52COMPAT
      WORKING_DIRECTORY ${LUAJIT_DIR}
      COMMENT "Building LuaJIT (static)")
  else()
    add_custom_command(
      OUTPUT  ${LUAJIT_LIB}
      COMMAND ${MAKE_EXE} BUILDMODE=static
              "CFLAGS=-fPIC"
              MACOSX_DEPLOYMENT_TARGET=10.7
              XCFLAGS=-DLUAJIT_ENABLE_LUA52COMPAT
      WORKING_DIRECTORY ${LUAJIT_DIR}
      COMMENT "Building LuaJIT (static)")
  endif()

  add_custom_target(luajit_build DEPENDS ${LUAJIT_LIB})

  add_library(luajit STATIC IMPORTED GLOBAL)
  set_target_properties(luajit PROPERTIES
    IMPORTED_LOCATION             ${LUAJIT_LIB}
    INTERFACE_INCLUDE_DIRECTORIES  ${LUAJIT_DIR})
  add_dependencies(luajit luajit_build)
endif()

if(NOT TARGET lua)
  pd_add_external(lua luas/lua.c)
  target_sources(lua PUBLIC luas/luajit.c)

  target_include_directories(lua PRIVATE ${LUAJIT_DIR})

  target_compile_definitions(lua PRIVATE
    MAKE_LIB
    ELSE
    PDLUA_VERSION="${PDLUA_VERSION}")

  if(APPLE)
    target_compile_definitions(lua PRIVATE LUA_USE_MACOSX)
  elseif(WIN32)
    target_compile_definitions(lua PRIVATE LUA_USE_WINDOWS)
  elseif(UNIX)
    target_compile_definitions(lua PRIVATE LUA_USE_LINUX)
  endif()

  if(DEFINED PD_MULTICHANNEL)
    target_compile_definitions(lua PRIVATE PD_MULTICHANNEL=${PD_MULTICHANNEL})
  endif()

  # stbi / nanosvg have unused functions — silence the warnings like the mk does
  if(NOT MSVC)
    target_compile_options(lua PRIVATE -Wno-unused-function)
  endif()

  target_link_libraries(lua PUBLIC luajit)
  pd_add_datafile(lua ${CMAKE_CURRENT_SOURCE_DIR}/pd.lua)
  pd_add_datafile(lua ${CMAKE_CURRENT_SOURCE_DIR}/lua.pd_lua)

  add_dependencies(lua luajit_build)
endif()
