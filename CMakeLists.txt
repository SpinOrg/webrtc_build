cmake_minimum_required(VERSION 3.18)
# project name
project(webrtc VERSION 86)

####### External dependencies
include(FetchContent)
set(FETCHCONTENT_QUIET FALSE)

set(EP_GCLIENT "GCLIENT")
set(CMAKE_BUILD_TYPE Release)
FetchContent_Declare(
    "GCLIENT"
    GIT_REPOSITORY https://chromium.googlesource.com/chromium/tools/depot_tools.git
    # GIT_TAG llvmorg-12-init
    GIT_SHALLOW ON
    BUILD_ALWAYS OFF
    # SOURCE_SUBDIR llvm
    GIT_PROGRESS TRUE
    USES_TERMINAL_DOWNLOAD TRUE
)
FetchContent_MakeAvailable(${EP_GCLIENT})
FetchContent_GetProperties(${EP_GCLIENT})

message(STATUS "DEPOT_TOOLS_DIR: ${gclient_SOURCE_DIR}")
set(DEPOT_TOOLS_DIR "${gclient_SOURCE_DIR}" CACHE STRING "" FORCE)
LIST(APPEND CMAKE_SYSTEM_PROGRAM_PATH ${gclient_SOURCE_DIR})
set(WEBRTC_DIR "${gclient_SOURCE_DIR}/../webrtc" CACHE STRING "" FORCE)
file(MAKE_DIRECTORY ${WEBRTC_DIR})

set(ENV{PATH} "${gclient_SOURCE_DIR}:$ENV{PATH}") # add gclient to path

set (fetch_cmd "fetch") # ${gclient_SOURCE_DIR}/fetch
set (fetch_arg1 "--nohooks")
set (fetch_arg2 "webrtc") 
execute_process(COMMAND ${fetch_cmd} ${fetch_arg1} ${fetch_arg2}
        WORKING_DIRECTORY ${WEBRTC_DIR}
        RESULT_VARIABLE __fetch_exit_code
        ERROR_VARIABLE __error)
if(__fetch_exit_code AND NOT "${__fetch_exit_code}" STREQUAL "0")
message(STATUS "Error: ${__fetch_exit_code}")
message(STATUS "Err: ${_error}")
# message(FATAL_ERROR "Fatal error executing: fetch --nohooks webrtc")
endif()

set(git "git")
set(checkout "checkout")
set(b "-b")
set(bname "webrtc86")
set(rname "refs/remotes/branch-heads/4240")
execute_process(COMMAND ${git} ${checkout} ${b} ${bname} ${rname}
        WORKING_DIRECTORY "${WEBRTC_DIR}/src"
        RESULT_VARIABLE __fetch_exit_code
        ERROR_VARIABLE __error)
if(__fetch_exit_code AND NOT "${__fetch_exit_code}" STREQUAL "0")
message(STATUS "Error: ${__fetch_exit_code}")
message(STATUS "Err: ${_error}")
# message(FATAL_ERROR "git checkout failed")
endif()

set (gclient_cmd "gclient")
set (gclient_arg1 "sync")
set (gclient_arg2 "-D")
execute_process(COMMAND ${gclient_cmd} ${gclient_arg1} ${gclient_arg2}
        WORKING_DIRECTORY ${WEBRTC_DIR}
        RESULT_VARIABLE __fetch_exit_code)
if(__fetch_exit_code AND NOT "${__fetch_exit_code}" STREQUAL "0")
message(STATUS "${__fetch_exit_code}")
message(FATAL_ERROR "Fatal error executing: gclient sync")
endif()
message(STATUS "WEBRTC_ROOT_DIR: ${WEBRTC_DIR}/src") # ${gclient_SOURCE_DIR}/../webrtc-checkout/src
set(WEBRTC_ROOT_DIR "${WEBRTC_DIR}/src" CACHE STRING "" FORCE)
set(webrtc_CMAKE_DIR "${PROJECT_SOURCE_DIR}/cmake/")

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${PROJECT_SOURCE_DIR}/cmake/")
message(STATUS "Searching FindWebRTC.cmake in: ${webrtc_CMAKE_DIR}")
find_package(WebRTC REQUIRED CONFIG
             PATHS ${webrtc_CMAKE_DIR}
             NO_DEFAULT_PATH)

if(WEBRTC_FOUND)
    message(STATUS "Found WebRTC")
    message(STATUS "Using FindWebRTC.cmake in: ${webrtc_CMAKE_DIR}")
endif()

include(Mergestaticlibs)

set(CMAKE_INSTALL_PREFIX "${PROJECT_BINARY_DIR}/dist" CACHE STRING "" FORCE)
add_library(dummylib dummy.cpp) 
if (MSVC)
    add_custom_command(TARGET dummylib PRE_BUILD
      COMMAND echo Building WebRTC library
      COMMAND set PATH=${DEPOT_TOOLS_DIR};%PATH%
      COMMAND cmd.exe /C gn gen ${WEBRTC_OUT_DIR} --args=\"is_debug=false is_component_build=false use_rtti=true use_custom_libcxx=false use_custom_libcxx_for_host=false target_cpu=\\\"x86\\\"\"
      COMMAND cmd.exe /C gn gen ${WEBRTC_OUT_DIR_DEBUG} --args=\"is_debug=true is_component_build=false use_rtti=true use_custom_libcxx=false use_custom_libcxx_for_host=false target_cpu=\\\"x86\\\"\"
      COMMAND ninja -C ${WEBRTC_OUT_DIR} 
      COMMAND ninja -C ${WEBRTC_OUT_DIR_DEBUG} 
      WORKING_DIRECTORY "${WEBRTC_ROOT_DIR}"
      COMMENT "Building WebRTC library (Once)"
    )
  elseif (APPLE)
    string(TOUPPER "${CMAKE_BUILD_TYPE}" _CMAKE_BUILD_TYPE)
    message(STATUS "Building WebRTC type: ${_CMAKE_BUILD_TYPE} at ${WEBRTC_OUT_DIR}")
    # skip this build using an external build of webrtc for now
    if (_CMAKE_BUILD_TYPE STREQUAL "DEBUG")
      add_custom_command(TARGET dummylib PRE_LINK
        COMMAND ${CMAKE_COMMAND} -E env \"PATH=${DEPOT_TOOLS_DIR}:$ENV{PATH}\" gn gen ${WEBRTC_OUT_DIR_DEBUG} --args=\"is_debug=true is_component_build=false use_rtti=true use_custom_libcxx=false use_custom_libcxx_for_host=false\"
        COMMAND ${CMAKE_COMMAND} -E env \"PATH=${DEPOT_TOOLS_DIR}:$ENV{PATH}\" ninja -C ${WEBRTC_OUT_DIR_DEBUG}
        WORKING_DIRECTORY "${WEBRTC_ROOT_DIR}"
        COMMENT "Building WebRTC library (Once)"
        )
    else()
      add_custom_command(TARGET dummylib PRE_LINK
        COMMAND ${CMAKE_COMMAND} -E env \"PATH=${DEPOT_TOOLS_DIR}:$ENV{PATH}\" gn gen ${WEBRTC_OUT_DIR} --args=\"is_debug=false is_component_build=false use_rtti=true use_custom_libcxx=false use_custom_libcxx_for_host=false\"
        COMMAND ${CMAKE_COMMAND} -E env \"PATH=${DEPOT_TOOLS_DIR}:$ENV{PATH}\" ninja -C ${WEBRTC_OUT_DIR}
        WORKING_DIRECTORY "${WEBRTC_ROOT_DIR}"
        COMMENT "Building WebRTC library (Once)"
        )
    endif()
  elseif(UNIX)
    string(TOUPPER "${CMAKE_BUILD_TYPE}" _CMAKE_BUILD_TYPE)
    if (_CMAKE_BUILD_TYPE STREQUAL "DEBUG")
      add_custom_command(TARGET dummylib PRE_LINK
        COMMAND export \"PATH=${DEPOT_TOOLS_DIR}:$ENV{PATH}\"
        COMMAND gn gen ${WEBRTC_OUT_DIR_DEBUG} --args=\"is_debug=true is_component_build=false use_rtti=true use_custom_libcxx=false use_custom_libcxx_for_host=false\"
        COMMAND ninja -C ${WEBRTC_OUT_DIR_DEBUG} 
        WORKING_DIRECTORY "${WEBRTC_ROOT_DIR}"
        COMMENT "Building WebRTC library (Once)"
        )
    else()
      add_custom_command(TARGET peerapi PRE_LINK
        COMMAND export \"PATH=${DEPOT_TOOLS_DIR}:$ENV{PATH}\"
        COMMAND gn gen ${WEBRTC_OUT_DIR} --args=\"is_debug=false is_component_build=false\"
        COMMAND ninja -C ${WEBRTC_OUT_DIR}
        WORKING_DIRECTORY "${WEBRTC_ROOT_DIR}"
        COMMENT "Building WebRTC library (Once)"
        )
    endif()

  endif(MSVC)