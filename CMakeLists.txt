project(game.libretro.stella)

cmake_minimum_required(VERSION 2.6)

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR})

enable_language(CXX)

find_package(kodi REQUIRED)
find_package(stella REQUIRED)

set(DEPLIBS stella)

if(CMAKE_BUILD_TYPE MATCHES Debug)
  set(LIBRETRO_DEBUG DEBUG=1)
endif()

set(STELLA_CUSTOM_BINARY ${STELLA_LIB}
                         game.libretro.stella${CMAKE_SHARED_LIBRARY_SUFFIX} stella)

build_addon(game.libretro.stella STELLA DEPLIBS)
