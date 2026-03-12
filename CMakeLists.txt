cmake_minimum_required(VERSION 3.20)
project(VisualImGui)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

get_property(IS_MULTI_CONFIG GLOBAL PROPERTY GENERATOR_IS_MULTI_CONFIG)

if(IS_MULTI_CONFIG)
    set(CMAKE_CONFIGURATION_TYPES "Debug;Release;Dist" CACHE STRING "Build configurations" FORCE)
else()
    if(NOT CMAKE_BUILD_TYPE)
        set(CMAKE_BUILD_TYPE Debug CACHE STRING "Build type" FORCE)
    endif()
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug;Release;Dist")
endif()

set(CMAKE_CXX_FLAGS_DIST           "${CMAKE_CXX_FLAGS_RELEASE}"           CACHE STRING "" FORCE)
set(CMAKE_C_FLAGS_DIST             "${CMAKE_C_FLAGS_RELEASE}"             CACHE STRING "" FORCE)
set(CMAKE_EXE_LINKER_FLAGS_DIST    "${CMAKE_EXE_LINKER_FLAGS_RELEASE}"    CACHE STRING "" FORCE)
set(CMAKE_SHARED_LINKER_FLAGS_DIST "${CMAKE_SHARED_LINKER_FLAGS_RELEASE}" CACHE STRING "" FORCE)

set(OUTPUT_BASE "${CMAKE_SOURCE_DIR}/Build/Binaries/${CMAKE_SYSTEM_NAME}-${CMAKE_HOST_SYSTEM_PROCESSOR}")

if(IS_MULTI_CONFIG)
    foreach(CFG Debug Release Dist)
        string(TOUPPER ${CFG} CFG_UPPER)
        set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_${CFG_UPPER} "${OUTPUT_BASE}/${CFG}")
        set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_${CFG_UPPER} "${OUTPUT_BASE}/${CFG}")
        set(CMAKE_LIBRARY_OUTPUT_DIRECTORY_${CFG_UPPER} "${OUTPUT_BASE}/${CFG}")
    endforeach()
else()
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${OUTPUT_BASE}/${CMAKE_BUILD_TYPE}")
    set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${OUTPUT_BASE}/${CMAKE_BUILD_TYPE}")
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${OUTPUT_BASE}/${CMAKE_BUILD_TYPE}")
endif()

if(UNIX AND NOT APPLE)
    add_compile_options(-fPIC)
    add_link_options(-pthread)
elseif(WIN32)
    add_compile_options(/EHsc /Zc:preprocessor /Zc:__cplusplus)
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>" CACHE STRING "" FORCE)
endif()

set(INCLUDE_GLAD      "${CMAKE_SOURCE_DIR}/Dependencies/glad/include")
set(INCLUDE_GLFW      "${CMAKE_SOURCE_DIR}/Dependencies/glfw/include")
set(INCLUDE_IMGUI     "${CMAKE_SOURCE_DIR}/Dependencies/imgui")
set(INCLUDE_JSON      "${CMAKE_SOURCE_DIR}/Dependencies/json")
set(INCLUDE_MINIAUDIO "${CMAKE_SOURCE_DIR}/Dependencies/miniaudio")
set(INCLUDE_SOIL      "${CMAKE_SOURCE_DIR}/Dependencies/soil")

add_library(GLAD STATIC
    Dependencies/glad/src/glad.c
)
target_include_directories(GLAD PUBLIC ${INCLUDE_GLAD})
set_target_properties(GLAD PROPERTIES POSITION_INDEPENDENT_CODE ON)

set(GLFW_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(GLFW_BUILD_TESTS    OFF CACHE BOOL "" FORCE)
set(GLFW_BUILD_DOCS     OFF CACHE BOOL "" FORCE)
set(GLFW_INSTALL        OFF CACHE BOOL "" FORCE)
add_subdirectory(Dependencies/glfw)

add_library(IMGUI STATIC
    Dependencies/imgui/imgui.cpp
    Dependencies/imgui/imgui_draw.cpp
    Dependencies/imgui/imgui_tables.cpp
    Dependencies/imgui/imgui_widgets.cpp
    Dependencies/imgui/imgui_demo.cpp
    Dependencies/imgui/backends/imgui_impl_glfw.cpp
    Dependencies/imgui/backends/imgui_impl_opengl3.cpp
)
target_include_directories(IMGUI PUBLIC
    ${INCLUDE_IMGUI}
    ${INCLUDE_IMGUI}/backends
    ${INCLUDE_GLAD}
    ${INCLUDE_GLFW}
)
target_compile_definitions(IMGUI PUBLIC IMGUI_IMPL_OPENGL_LOADER_GLAD)
target_link_libraries(IMGUI PUBLIC GLAD glfw)
set_target_properties(IMGUI PROPERTIES POSITION_INDEPENDENT_CODE ON)

add_library(MINIAUDIO STATIC
    Dependencies/miniaudio/miniaudio.c
)
target_include_directories(MINIAUDIO PUBLIC ${INCLUDE_MINIAUDIO})
set_target_properties(MINIAUDIO PROPERTIES POSITION_INDEPENDENT_CODE ON)

if(UNIX AND NOT APPLE)
    target_link_libraries(MINIAUDIO PUBLIC dl m pthread)
endif()

file(GLOB_RECURSE EDITOR_SOURCES
    "Editor/*.h"
    "Editor/*.hpp"
    "Editor/*.cpp"
    "Editor/*.c"
)

add_executable(Editor ${EDITOR_SOURCES})

target_include_directories(Editor PRIVATE
    "${CMAKE_SOURCE_DIR}/Editor"
    ${INCLUDE_GLAD}
    ${INCLUDE_GLFW}
    ${INCLUDE_IMGUI}
    ${INCLUDE_JSON}
    ${INCLUDE_MINIAUDIO}
    ${INCLUDE_SOIL}
)

target_link_libraries(Editor PRIVATE
    GLAD
    glfw
    IMGUI
    MINIAUDIO
)

target_compile_definitions(Editor PRIVATE
    $<$<CONFIG:Debug>:DEBUG>
    $<$<CONFIG:Release>:RELEASE>
    $<$<CONFIG:Dist>:DIST>
)

if(UNIX AND NOT APPLE)
    target_compile_definitions(Editor PRIVATE VIG_PLATFORM_LINUX)
    target_link_libraries(Editor PRIVATE GL dl pthread X11 Xrandr Xi Xxf86vm Xcursor Xinerama)
elseif(WIN32)
    target_compile_definitions(Editor PRIVATE VIG_PLATFORM_WINDOWS _CRT_SECURE_NO_WARNINGS)
    target_link_libraries(Editor PRIVATE opengl32 comdlg32)
endif()

set_target_properties(Editor PROPERTIES
    CXX_STANDARD          20
    CXX_STANDARD_REQUIRED ON
)
