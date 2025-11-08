cmake_minimum_required(VERSION 3.25)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# check if inside some project (for pd-else)
if(CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
  project(pdlua)
endif()

set(PDCMAKE_FILE ${CMAKE_BINARY_DIR}/pd.cmake)
set(PDCMAKE_VERSION "v0.2.6")
if(NOT EXISTS "${PDCMAKE_FILE}")
  file(
    DOWNLOAD
    https://raw.githubusercontent.com/pure-data/pd.cmake/refs/tags/${PDCMAKE_VERSION}/pd.cmake
    ${PDCMAKE_FILE})
endif()
include(${PDCMAKE_FILE})

# check if not lua was added before (for pd4web)
if(NOT TARGET lua)
  add_library(lua STATIC "${CMAKE_CURRENT_SOURCE_DIR}/lua/onelua.c")
  target_compile_definitions(lua PUBLIC "-DMAKE_LIB")
  set_target_properties(lua PROPERTIES POSITION_INDEPENDENT_CODE ON)
endif()

if(NOT TARGET pdlua)
  pd_add_external(pdlua pdlua.c)
  target_link_libraries(pdlua PUBLIC lua)
  pd_add_datafile(pdlua pd.lua)
  pd_add_datafile(pdlua
                  "${CMAKE_CURRENT_SOURCE_DIR}/pdlua/tutorial/examples/pdx.lua")
  target_include_directories(pdlua PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/lua")
else()
  target_sources(pdlua PUBLIC "${LIB_DIR}/pdlua.c")
endif()
