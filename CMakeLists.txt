cmake_minimum_required(VERSION 3.9)
project(wintun VERSION 1.0.0)

add_library(${PROJECT_NAME} INTERFACE)

# To prevent name collision issues, we include the directory _just above_ the project
# directory. To access the include files, use:
# #include "wintun/api/wintun.h"
target_include_directories(${PROJECT_NAME} INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/..)
