cmake_minimum_required(VERSION 3.1)
project(game.libretro.stella)

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR})

find_package(Kodi REQUIRED)
find_package(libretro-stella REQUIRED)

set(DEPLIBS stella)

if(CMAKE_BUILD_TYPE MATCHES Debug)
  set(LIBRETRO_DEBUG DEBUG=1)
endif()

set(STELLA_CUSTOM_BINARY ${STELLA_LIB}
                         ${PROJECT_NAME}${CMAKE_SHARED_LIBRARY_SUFFIX})

build_addon(${PROJECT_NAME} STELLA DEPLIBS)
