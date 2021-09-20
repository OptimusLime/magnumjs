cmake_minimum_required(VERSION 3.9)
cmake_policy(SET CMP0042 NEW)
# project(MyApplication)
project (magnum-js)


# Very first thing to do. Either node_modules exists within our directory
# or it exists two directories up and we are being built as a dependency (most likely)
get_filename_component(_fullpath "node_modules" REALPATH)
get_filename_component(_fullbackpath "../../node_modules" REALPATH)
IF(EXISTS ${_fullpath})
  set(BASE_NODE_DIR ${_fullpath})
elseif(EXISTS ${_fullbackpath})
  # we are a dependency for another project inside of a node_modules folder
  set(BASE_NODE_DIR ${_fullbackpath})
endif()
message(STATUS "Setting Node Depends Dir: ${BASE_NODE_DIR}")

set(MAGNUMJS_MAJOR_VERSION 0)
set(MAGNUMJS_MINOR_VERSION 1)
set(MAGNUMJS_PATCH_VERSION 0)
set(MAGNUMJS_VERSION
  ${MAGNUMJS_MAJOR_VERSION}.${MAGNUMJS_MINOR_VERSION}.${MAGNUMJS_PATCH_VERSION})

  # Offer the user the choice of overriding the installation directories
set(INSTALL_LIB_DIR lib CACHE PATH "Installation directory for libraries")
set(INSTALL_BIN_DIR bin CACHE PATH "Installation directory for executables")
set(INSTALL_INCLUDE_DIR include CACHE PATH
  "Installation directory for header files")

if(WIN32 AND NOT CYGWIN)
  set(DEF_INSTALL_CMAKE_DIR cmake)
else()
  set(DEF_INSTALL_CMAKE_DIR lib/cmake/MinMagApi)
endif()
set(INSTALL_CMAKE_DIR ${DEF_INSTALL_CMAKE_DIR} CACHE PATH
  "Installation directory for CMake files")

# Make relative paths absolute (needed later on)
foreach(p LIB BIN INCLUDE CMAKE)
set(var INSTALL_${p}_DIR)
if(NOT IS_ABSOLUTE "${${var}}")
  set(${var} "${CMAKE_INSTALL_PREFIX}/${${var}}")
endif()
endforeach()

set (CMAKE_CXX_STANDARD 14)
include_directories(${CMAKE_JS_INC})

# FetchContent added in CMake 3.11, downloads during the configure step
include(FetchContent)
# FetchContent_MakeAvailable was not added until CMake 3.14; use our shim
if(${CMAKE_VERSION} VERSION_LESS 3.14)
    include(cmake/add_FetchContent_MakeAvailable.cmake)
endif()

add_library(${PROJECT_NAME} SHARED) 
set_target_properties(${PROJECT_NAME} PROPERTIES PREFIX "" SUFFIX ".node")
# set(CMAKE_INSTALL_PREFIX ${CMAKE_SOURCE_DIR}/bin)

message(STATUS "ATTEMPT FIND CORRADE: ${CMAKE_SOURCE_DIR}/node_modules/corradejs/corrade/share/cmake/Corrade")
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${BASE_NODE_DIR}/corradejs/corrade/share/cmake/Corrade)
set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH} ${BASE_NODE_DIR}/corradejs/corrade)
# set(_CORRADE_MODULE_DIR ${BASE_NODE_DIR}/corradejs/corrade/share)
find_package(Corrade REQUIRED)

message(STATUS "Corrade Incl: ${CORRADE_INCLUDE_DIR}")
set(CORRADE_INCLUDE_DIR ${BASE_NODE_DIR}/corradejs/corrade/include ${CORRADE_INCLUDE_DIR})

# add our dependencies
FetchContent_Declare(
  magnum
  GIT_REPOSITORY https://github.com/mosra/magnum.git
  GIT_TAG dbec10dbee8cc3c2b560c6e7de9166c2b6d2f002
)

FetchContent_GetProperties(magnum)
if(NOT magnum_POPULATED)
    message(STATUS "Fetching MAGNUM v${MAGNUM_VERSION}")

    FetchContent_Populate(magnum)

    # Add Magnum as a subproject, enable Sdl2Application
    set(WITH_SDL2APPLICATION TRUE CACHE BOOL "" FORCE)
    # set(WITH_TGAIMPORTER TRUE CACHE BOOL "" FORCE)
    set(WITH_ANYIMAGEIMPORTER TRUE CACHE BOOL "" FORCE)
    set(WITH_TRADE TRUE CACHE BOOL "" FORCE)
    set(WITH_GL TRUE CACHE BOOL "" FORCE)
    set(WITH_PRIMITIVES TRUE CACHE BOOL "" FORCE)
    set(WITH_SHADERS TRUE CACHE BOOL "" FORCE)

    set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${magnum_SOURCE_DIR}/modules)
    add_subdirectory(${magnum_SOURCE_DIR} ${magnum_BINARY_DIR})

    set(MAGNUM_INCLUDE_DIR 
        ${magnum_SOURCE_DIR}/src
        ${magnum_BUILD_DIR}/src
        )

    set(MAGNUM_LIBRARIES 
        Magnum::Sdl2Application
        Magnum::GL
        Magnum::Magnum
        Magnum::Trade
        Magnum::MeshTools
        Magnum::Shaders
        Magnum::SceneGraph
        Magnum::Primitives
        )

endif()

target_compile_definitions(
        ${PROJECT_NAME}
        PRIVATE
            # ENTT_ID_TYPE=${ENTT_ID_TYPE}
            # _ENABLE_EXTENDED_ALIGNED_STORAGE
            NOMINMAX
            # ENTT_API_EXPORT
    )

file(GLOB SRC_MAIN
    "src/lib/*.cpp")

target_sources(${PROJECT_NAME} PRIVATE
    ${SRC_MAIN}
    ${EASYLOGGING}
    ${EMPTY_RESOURCES}
    ${CMAKE_JS_SRC}
    )

target_link_libraries(${PROJECT_NAME} PRIVATE
    ${CMAKE_JS_LIB} 
    ${CORRADE_LIBRARIES}
) 

# Include Node-API wrappers
execute_process(COMMAND node -p "require('node-addon-api').include"
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE NODE_ADDON_API_DIR
        )

string(REPLACE "\n" "" NODE_ADDON_API_DIR ${NODE_ADDON_API_DIR})
string(REPLACE "\"" "" NODE_ADDON_API_DIR ${NODE_ADDON_API_DIR})
string(REGEX REPLACE "[\r\n\"]" "" NODE_ADDON_API_DIR ${NODE_ADDON_API_DIR})

target_include_directories(${PROJECT_NAME} PRIVATE 
    ./
    src
    ../external/cr
    ${CORRADE_INCLUDE_DIR}
    ${MAGNUM_INCLUDE_DIR}
    ${NODE_ADDON_API_DIR})

# define NPI_VERSION
add_definitions(-DNAPI_VERSION=7)
