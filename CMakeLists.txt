cmake_minimum_required(VERSION 3.13 FATAL_ERROR)
project("NanoGUI" LANGUAGES C CXX)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake/Modules")

# ccache
find_program(CCACHE_FOUND ccache)
if(CCACHE_FOUND)
    message(STATUS "Found CCache")
    set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
    set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ccache)
endif(CCACHE_FOUND)

include(CMakeDependentOption)
# superbuild settings
include(SuperBuildDepends)

# cpp standard settings
superbuild_option(cpp17)
superbuild_option(modern-cpp)

# dependencies settings
superbuild_depend(enoki)
superbuild_depend(glfw)

# always generate PIC code, no matter static or shared library
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# project directories settings
set(NANOGUI_DIR ${CMAKE_CURRENT_SOURCE_DIR})
set(NANOGUI_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/include)
set(NANOGUI_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/src)
# set(NANOGUI_BINARY_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR}/include)
# set(NANOGUI_TEST_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/test)

# some build options

if(WIN32)
    set(NANOGUI_USE_GLAD_DEFAULT ON)
else()
    set(NANOGUI_USE_GLAD_DEFAULT OFF)
endif()

option(NANOGUI_BUILD_EXAMPLE "Build NanoGUI example application" ON)
# option(NANOGUI_BUILD_SHARED  "Build NanoGUI as a shared library" ON)
option(NANOGUI_USE_GLAD "Use Glad OpenGL loader library?" ${NANOGUI_USE_GLAD_DEFAULT})

if(NOT NANOGUI_BACKEND)
    if(APPLE)
        set(NANOGUI_BACKEND_DEFAULT "Metal")
    else()
        set(NANOGUI_BACKEND_DEFAULT "OpenGL")
    endif()

    set(NANOGUI_BACKEND ${NANOGUI_BACKEND_DEFAULT} CACHE STRING "Choose the backend used for rendering (OpenGL/Metal)" FORCE)
  endif()

set_property(CACHE NANOGUI_BACKEND PROPERTY STRINGS "OpenGL" "Metal")

set(NANOGUI_EXTRA_DEFS "")
set(NANOGUI_EXTRA_LIBS "")
set(NANOGUI_EXTRA_INCS "")
set(NANOGUI_EXTRA_GLOB "")

set(LIBNANOGUI_EXTRA_SOURCE "")

# if(NANOGUI_BUILD_SHARED)
#     list(APPEND NANOGUI_EXTRA_DEFS -DNANOGUI_SHARED -DNVG_SHARED)
# endif()

list(APPEND NANOGUI_EXTRA_DEFS -DNANOGUI_BUILD -DNVG_BUILD -DNVG_STB_IMAGE_IMPLEMENTATION)

if(NANOGUI_BACKEND STREQUAL "OpenGL")
    list(APPEND LIBNANOGUI_EXTRA_SOURCE
        "${CMAKE_CURRENT_SOURCE_DIR}/src/texture_gl.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/src/shader_gl.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/src/renderpass_gl.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/src/opengl.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/src/opengl_check.h"
    )
    list(APPEND NANOGUI_EXTRA_DEFS
        -DNANOGUI_USE_OPENGL
    )
    list(APPEND NANOGUI_EXTRA_GLOB
        "${CMAKE_CURRENT_SOURCE_DIR}/resources/*.gl"
    )
    list(APPEND NANOGUI_EXTRA_INCS
        "${CMAKE_CURRENT_SOURCE_DIR}/ext/nanovg/src"
    )
    message(STATUS "NanoGUI: using OpenGL backend.")
endif()

if(NANOGUI_BACKEND STREQUAL "Metal")
    list(APPEND LIBNANOGUI_EXTRA_SOURCE
        "${CMAKE_CURRENT_SOURCE_DIR}/ext/nanovg_metal/src/nanovg_mtl.m"
        "${CMAKE_CURRENT_SOURCE_DIR}/ext/nanovg_metal/src/nanovg_mtl.h"
        "${CMAKE_CURRENT_SOURCE_DIR}/src/texture_metal.mm"
        "${CMAKE_CURRENT_SOURCE_DIR}/src/shader_metal.mm"
        "${CMAKE_CURRENT_SOURCE_DIR}/src/renderpass_metal.mm"
    )
    list(APPEND NANOGUI_EXTRA_DEFS
        -DNANOGUI_USE_METAL
    )
    list(APPEND NANOGUI_EXTRA_GLOB
        "${CMAKE_CURRENT_SOURCE_DIR}/resources/*.metal"
    )
    list(APPEND NANOGUI_EXTRA_INCS
        "${CMAKE_CURRENT_SOURCE_DIR}/ext/nanovg_metal/src"
    )
    message(STATUS "NanoGUI: using Metal backend.")
endif()

if(NANOGUI_USE_GLAD)
    list(APPEND LIBNANOGUI_EXTRA_SOURCE
        "${CMAKE_CURRENT_SOURCE_DIR}/ext/glad/src/glad.c"
        "${CMAKE_CURRENT_SOURCE_DIR}/ext/glad/include/glad/glad.h"
        "${CMAKE_CURRENT_SOURCE_DIR}/ext/glad/include/KHR/khrplatform.h"
    )
    if(MSVC)
      set_source_files_properties("${CMAKE_CURRENT_SOURCE_DIR}/ext/glad/src/glad.c"
        PROPERTIES COMPILE_FLAGS "/wd4055")
    endif()
    list(APPEND NANOGUI_EXTRA_DEFS -DNANOGUI_GLAD)
    list(APPEND NANOGUI_EXTRA_INCS "${CMAKE_CURRENT_SOURCE_DIR}/ext/glad/include")
endif()

# Required core libraries on various platforms
if(WIN32)
    list(APPEND NANOGUI_EXTRA_LIBS opengl32)
elseif(APPLE)
    find_library(cocoa_library Cocoa)
    find_library(iokit_library IOKit)
    list(APPEND NANOGUI_EXTRA_LIBS ${cocoa_library} ${iokit_library})

    if(NANOGUI_BACKEND STREQUAL "OpenGL")
        find_library(opengl_library OpenGL)
        find_library(corevideo_library CoreVideo)
        list(APPEND NANOGUI_EXTRA_LIBS ${opengl_library} ${corevideo_library})
    else()
        find_library(metal_library Metal)
        find_library(quartzcore_library QuartzCore)
        list(APPEND NANOGUI_EXTRA_LIBS ${metal_library} ${quartzcore_library})
    endif()

    list(APPEND LIBNANOGUI_EXTRA_SOURCE
        "${CMAKE_CURRENT_SOURCE_DIR}/src/darwin.mm"
    )
elseif(CMAKE_SYSTEM MATCHES "Linux" OR CMAKE_SYSTEM_NAME MATCHES "BSD")
    list(APPEND NANOGUI_EXTRA_LIBS GL Xxf86vm Xrandr Xinerama Xcursor Xi X11 pthread)
    if(NANOGUI_BACKEND STREQUAL "OpenGL")
        list(APPEND NANOGUI_EXTRA_LIBS GL)
    endif()
    if (NOT CMAKE_SYSTEM_NAME MATCHES "OpenBSD")
        list(APPEND NANOGUI_EXTRA_LIBS rt)
    endif()
    if(CMAKE_SYSTEM MATCHES "Linux")
        list(APPEND NANOGUI_EXTRA_LIBS dl)
    endif()
endif()

# Run simple cmake converter to put font files into the data segment
# Glob up resource files
file(GLOB resources
    "${CMAKE_CURRENT_SOURCE_DIR}/resources/*.ttf"
    ${NANOGUI_EXTRA_GLOB}
)

file(MAKE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/resources")

# Precompile .metal shaders to .metallib files
foreach(fname_in IN LISTS resources)
  if (NANOGUI_BACKEND STREQUAL "Metal" AND fname_in MATCHES "\\.metal")
    get_filename_component(fname_out ${fname_in} NAME)
    set(fname_out "${CMAKE_CURRENT_BINARY_DIR}/resources/${fname_out}lib")
    add_custom_command(
        OUTPUT ${fname_out}
        DEPENDS ${fname_in}
        COMMAND xcrun -sdk macosx metal -std=osx-metal2.0 -O3 "${fname_in}" -o "${fname_out}"
        VERBATIM
    )
  else()
    set(fname_out "${fname_in}")
  endif()
  list(APPEND resources_processed ${fname_out})
endforeach()

# Concatenate resource files into a comma separated string
string(REGEX REPLACE "([^\\]|^);" "\\1," resources_string "${resources_processed}")
string(REGEX REPLACE "[\\](.)" "\\1"     resources_string "${resources_string}")

# Create command line for running bin2c cmake script
set(bin2c_cmdline
    -DOUTPUT_C=nanogui_resources.cpp
    -DOUTPUT_H=nanogui_resources.h
    "-DINPUT_FILES=${resources_string}"
    -P "${CMAKE_CURRENT_SOURCE_DIR}/resources/bin2c.cmake"
)

# Run bin2c on resource files
add_custom_command(
    OUTPUT nanogui_resources.cpp nanogui_resources.h
    COMMAND ${CMAKE_COMMAND} ARGS ${bin2c_cmdline}
    DEPENDS ${resources} ${resources_processed}
    COMMENT "Running bin2c"
    PRE_BUILD VERBATIM
)

set(NANOGUI_PUBLIC_HEADERS
    ${NANOGUI_INCLUDE_DIR}/nanogui/arcball.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/button.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/canvas.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/checkbox.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/colorpicker.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/colorwheel.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/combobox.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/common.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/formhelper.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/graph.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/icons.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/imagepanel.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/imageview.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/label.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/layout.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/messagedialog.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/nanogui.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/object.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/opengl.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/popup.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/popupbutton.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/progressbar.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/renderpass.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/screen.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/shader.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/slider.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/tabwidget.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/textarea.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/textbox.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/texture.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/theme.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/toolbutton.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/vscrollpanel.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/widget.h
    ${NANOGUI_INCLUDE_DIR}/nanogui/window.h
    ${CMAKE_CURRENT_BINARY_DIR}/nanogui_resources.h
    ${CMAKE_CURRENT_SOURCE_DIR}/ext/nanovg/src/stb_image.h
)

set(NANOGUI_PUBLIC_SOURCES
)

set(NANOGUI_PRIVATE_HEADERS
)

set(NANOGUI_PRIVATE_SOURCES
    ${NANOGUI_SOURCE_DIR}/button.cpp
    ${NANOGUI_SOURCE_DIR}/canvas.cpp
    ${NANOGUI_SOURCE_DIR}/checkbox.cpp
    ${NANOGUI_SOURCE_DIR}/colorpicker.cpp
    ${NANOGUI_SOURCE_DIR}/colorwheel.cpp
    ${NANOGUI_SOURCE_DIR}/combobox.cpp
    ${NANOGUI_SOURCE_DIR}/common.cpp
    ${NANOGUI_SOURCE_DIR}/graph.cpp
    ${NANOGUI_SOURCE_DIR}/imagepanel.cpp
    ${NANOGUI_SOURCE_DIR}/imageview.cpp
    ${NANOGUI_SOURCE_DIR}/label.cpp
    ${NANOGUI_SOURCE_DIR}/layout.cpp
    ${NANOGUI_SOURCE_DIR}/messagedialog.cpp
    ${NANOGUI_SOURCE_DIR}/popup.cpp
    ${NANOGUI_SOURCE_DIR}/popupbutton.cpp
    ${NANOGUI_SOURCE_DIR}/progressbar.cpp
    ${NANOGUI_SOURCE_DIR}/screen.cpp
    ${NANOGUI_SOURCE_DIR}/shader.cpp
    ${NANOGUI_SOURCE_DIR}/slider.cpp
    ${NANOGUI_SOURCE_DIR}/tabwidget.cpp
    ${NANOGUI_SOURCE_DIR}/textarea.cpp
    ${NANOGUI_SOURCE_DIR}/textbox.cpp
    ${NANOGUI_SOURCE_DIR}/texture.cpp
    ${NANOGUI_SOURCE_DIR}/theme.cpp
    ${NANOGUI_SOURCE_DIR}/vscrollpanel.cpp
    ${NANOGUI_SOURCE_DIR}/widget.cpp
    ${NANOGUI_SOURCE_DIR}/window.cpp
    ${CMAKE_CURRENT_BINARY_DIR}/nanogui_resources.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/ext/nanovg/src/nanovg.c
    ${LIBNANOGUI_EXTRA_SOURCE}
)

add_library(nanogui)
target_compile_definitions(nanogui
    PUBLIC
        ${NANOGUI_EXTRA_DEFS}
        -DGL_SILENCE_DEPRECATION
)
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    target_compile_options(nanogui
        PUBLIC
            -Qunused-arguments
            -mllvm
            -inline-threshold=5000
            -g
    )
    if (APPLE)
        # Use automatic reference counting for Objective-C portions
        target_compile_options(nanogui
            PUBLIC
                -fobjc-arc
        )
    endif()
elseif(CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    target_compile_definitions(nanogui
        PUBLIC
            _CRT_SECURE_NO_WARNINGS
    )
    target_compile_options(nanogui
        PUBLIC
            /MP
            $<$<NOT:$<EQUAL:${CMAKE_SIZEOF_VOID_P},8>>:
                /arch:SSE2
                /DNANOGUI_EIGEN_DONT_ALIGN
            >
    )
endif()
set_target_properties(nanogui PROPERTIES CXX_EXTENSIONS OFF) # -std=c++17 instead of -std=g++17
target_sources(nanogui
    PRIVATE
        ${NANOGUI_PUBLIC_HEADERS}
        ${NANOGUI_PUBLIC_SOURCES}
        ${NANOGUI_PRIVATE_HEADERS}
        ${NANOGUI_PRIVATE_SOURCES}
)
target_include_directories(nanogui
    PUBLIC
        ${NANOGUI_INCLUDE_DIR}
        ${NANOGUI_EXTRA_INCS}
        # ${NANOGUI_BINARY_INCLUDE_DIR}
        ${CMAKE_CURRENT_BINARY_DIR}
        ${CMAKE_CURRENT_SOURCE_DIR}/ext/nanovg/src
    PRIVATE
        ${NANOGUI_SOURCE_DIR}
)
target_link_libraries(nanogui
    PUBLIC
        options::modern-cpp
        depends::enoki
        depends::glfw
        ${NANOGUI_EXTRA_LIBS}
)

set(NANOGUI_LIBRARY_DIR ${CMAKE_CURRENT_BINARY_DIR} CACHE INTERNAL "")


# Build example application if desired
if(NANOGUI_BUILD_EXAMPLE)
    add_executable(example1      utils/example1.cpp)
    add_executable(example2      utils/example2.cpp)
    add_executable(example3      utils/example3.cpp)
    add_executable(example4      utils/example4.cpp)
    add_executable(example_icons utils/example_icons.cpp)
    target_link_libraries(example1      PRIVATE nanogui)
    target_link_libraries(example2      PRIVATE nanogui)
    target_link_libraries(example3      PRIVATE nanogui)
    target_link_libraries(example4      PRIVATE nanogui)
    target_link_libraries(example_icons PRIVATE nanogui)

    # Copy icons for example application
    file(COPY resources/icons DESTINATION ${CMAKE_CURRENT_BINARY_DIR})
endif()
