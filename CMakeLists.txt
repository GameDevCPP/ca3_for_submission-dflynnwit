cmake_minimum_required(VERSION 3.9)
project(DropPod)

set(CMAKE_CXX_STANDARD 14)  # normally use 17, however, SFML v 2.5.x may not compile
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(FetchContent)

# SFML
#set(SFML_STATIC_LIBRARIES TRUE)
FetchContent_Declare(
        sfml
        GIT_REPOSITORY https://github.com/SFML/SFML.git
        GIT_TAG        2.5.x  # note that you can use the latest 2.6.x
)
FetchContent_MakeAvailable(sfml)

# Box2D

FetchContent_Declare(box2d
        GIT_REPOSITORY     https://github.com/dooglz/Box2D.git
        GIT_TAG             v2.3.1)

# We need finer control over the build of box2d so we set the following variables
set(BOX2D_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
if(NOT box2d_POPULATED)
    FetchContent_Populate(box2d)
    add_subdirectory(${box2d_SOURCE_DIR}/Box2D ${box2d_BINARY_DIR})
endif()


# Main output directory
set(OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin/")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${OUTPUT_DIRECTORY})

# Resource copying targets to e.g. Debug/Release directory
add_custom_target(copy_resources ALL COMMAND ${CMAKE_COMMAND}
        -E copy_directory
        "${PROJECT_SOURCE_DIR}/res"
        ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/$<CONFIGURATION>/res
)
add_custom_target(copy_resources_rootdir ALL COMMAND ${CMAKE_COMMAND}
        -E copy_directory
        "${PROJECT_SOURCE_DIR}/res"
        ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/res
)

#message("Box2d is located at:" "${box2d_SOURCE_DIR}/Box2D" )
# Engine library
file(GLOB_RECURSE ENGINE_SOURCES engine/*.cpp engine/*.h)
add_library(lib_engine STATIC ${ENGINE_SOURCES})
target_include_directories(lib_engine PUBLIC "${CMAKE_SOURCE_DIR}/engine"  "${box2d_SOURCE_DIR}/Box2D" "{CMAKE_SOURCE_DIR}/lib_maths")
target_link_libraries(lib_engine PRIVATE sfml-graphics sfml-audio Box2D lib_ecm lib_maths)

# Maths lib
add_library(lib_maths INTERFACE)
target_include_directories(lib_maths INTERFACE "${CMAKE_SOURCE_DIR}/lib_maths")
target_link_libraries(lib_maths INTERFACE sfml-graphics sfml-system)

# Tile loader lib
file(GLOB_RECURSE TILE_LOADER_SOURCES lib_tile_level_loader/*.cpp lib_tile_level_loader/*.h)
add_library(lib_tile_level_loader STATIC ${TILE_LOADER_SOURCES})
target_include_directories(lib_tile_level_loader INTERFACE "${CMAKE_SOURCE_DIR}/lib_tile_level_loader")
target_link_libraries(lib_tile_level_loader PRIVATE lib_maths sfml-graphics)

# ECM lib
file(GLOB_RECURSE ECM_SOURCES lib_ecm/*.cpp lib_ecm/*.h)
add_library(lib_ecm STATIC ${ECM_SOURCES})
target_include_directories(lib_ecm INTERFACE "${CMAKE_SOURCE_DIR}/lib_ecm" "{CMAKE_SOURCE_DIR}/lib_maths")
target_link_libraries(lib_ecm PRIVATE lib_maths)

#DropPod executable
file(GLOB_RECURSE DropPod_SOURCES Drop-Pod/*.cpp Drop-Pod/*.h)
add_executable(DropPod ${DropPod_SOURCES})
target_link_libraries(DropPod PRIVATE lib_engine lib_ecm lib_maths sfml-graphics sfml-audio sfml-system sfml-window lib_tile_level_loader )
add_dependencies(DropPod copy_resources copy_resources_rootdir)


# if win32, copy dlls from _deps/sfml-build/lib
if(WIN32)
    add_custom_command(TARGET DropPod POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_directory
            "${CMAKE_BINARY_DIR}/_deps/sfml-build/lib"
            $<TARGET_FILE_DIR:DropPod>)
endif()


# Copy dlls to output directory
if (WIN32)
    add_custom_command(TARGET DropPod POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_directory
            "${CMAKE_BINARY_DIR}/_deps/sfml-build/lib"
            $<TARGET_FILE_DIR:DropPod>)
endif()

if (WIN32 AND SFML_BUILD_AUDIO)
    add_custom_command(TARGET DropPod POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy ${FETCHCONTENT_BASE_DIR}/sfml-src/extlibs/bin/x64/openal32.dll $<TARGET_FILE_DIR:DropPod>
    )
endif()

