project(game.libretro.stella)

cmake_minimum_required(VERSION 2.6)

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR})

enable_language(CXX)

find_package(kodi REQUIRED)

include(ExternalProject)
externalproject_add(stella
                    GIT_REPOSITORY https://github.com/libretro/stella-libretro
                    GIT_TAG d74c6bb
                    PREFIX stella
                    CONFIGURE_COMMAND ""
                    INSTALL_COMMAND ""
                    BUILD_COMMAND make
                    BUILD_IN_SOURCE 1)

set(STELLA_CUSTOM_BINARY ${PROJECT_BINARY_DIR}/stella/src/stella/stella_libretro${CMAKE_SHARED_LIBRARY_SUFFIX}
                         game.libretro.stella${CMAKE_SHARED_LIBRARY_SUFFIX} stella)

build_addon(game.libretro.stella STELLA DEPLIBS)
add_dependencies(game.libretro.stella stella)

include(CPack)
