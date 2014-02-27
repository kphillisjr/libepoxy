## Require Cmake verison 2.8.6 due to improved python handling.
cmake_minimum_required(VERSION 2.8.6)

INCLUDE (CheckCCompilerFlag)
INCLUDE (CheckCSourceCompiles)
INCLUDE (CheckCXXCompilerFlag)
INCLUDE (CheckFunctionExists)
INCLUDE (CheckIncludeFile)
INCLUDE (FindPkgConfig)

project (libepoxy)
#Find Python 2.6 interpreter.
find_package(PythonInterp 2.6 REQUIRED)

if(NOT PYTHONINTERP_FOUND)
    message(FATAL_ERROR "python version 2.x (where x >= 6) required")
endif(NOT PYTHONINTERP_FOUND)

set(python ${PYTHON_EXECUTABLE})

if(WIN32)
    set(EPOXY_BUILD_GL TRUE)
    set(EPOXY_BUILD_WGL TRUE)
else()

    # check for libEGL
    pkg_check_modules(PKG_EGL REQUIRED egl)
    if(PKG_EGL_FOUND)
        set(EPOXY_BUILD_EGL TRUE)
    endif()

    # Check for libGL
	pkg_check_modules(PKG_OPENGL REQUIRED gl)
	if(PKG_OPENGL_FOUND)
        set(EPOXY_BUILD_GL TRUE)
	endif()
    
    # Check for GLX
    find_package(X11)
    if(X11_FOUND AND PKG_OPENGL_FOUND)
        set(EPOXY_BUILD_GLX TRUE)
    endif()
endif()


check_include_file(KHR/khrplatform.h  HAVE_KHRPLATFORM_H)
check_include_file(stdbool.h  HAVE_STDBOOL_H)
check_include_file(stdint.h   HAVE_STDINT_H)
if(NOT MSVC)
    # OS X defaults to having -Wint-conversion ("warn when passing
    # uintptr_t to a void *") by default.  Kill that.
    CHECK_C_COMPILER_FLAG("-Wno-int-conversion" C_COMPILER_FLAG_NOINTCONVERSION)
    if(C_COMPILER_FLAG_NOINTCONVERSION)
        SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-int-conversion")
    endif()
    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=gnu90")
else()
    # Silence all warnings about old CRT calls
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
    
    file(DOWNLOAD
        http://www.khronos.org/registry/egl/api/KHR/khrplatform.h 
        ${CMAKE_BINARY_DIR}/include/KHR/khrplatform.h)
        # beta
endif()

set(LIBEPOXY_INCLUDES src/dispatch_common.h)
set(LIBEPOXY_SOURCES src/dispatch_common.c)

add_custom_target(libepoxy_dispatch
    COMMENT "Generated libepoxy dispatch Requirements.")

function(libepoxy_generate_dispatch name)
    list(REMOVE_AT ARGV 0)
    # Generate the generated files at configure time.
    execute_process(
        COMMAND ${ARGV}
        OUTPUT_QUIET
        ERROR_QUIET
        RESULT_VARIABLE ${name}_gen_dispatch_error_code)
    if(NOT ${name}_gen_dispatch_error_code EQUAL 0)
        message(FATAL_ERROR "python Failed to generate ${name} Dispatch.")
    endif()
    # regenerate the generated files on build.
    add_custom_target( libepoxy_dispatch_${name}
        COMMAND ${ARGV}
        COMMENT "Generating ${name} Dispatch."
        )
    add_dependencies(libepoxy_dispatch libepoxy_dispatch_${name})
endfunction(libepoxy_generate_dispatch)

set(PLATFORM_HAS_EGL 0)
set(PLATFORM_HAS_GLX 0)
set(PLATFORM_HAS_WGL 0)
set(PLATFORM_HAS_OPENGL 0)
if(EPOXY_BUILD_EGL)
    list(APPEND LIBEPOXY_SOURCES src/dispatch_egl.c)
    list(APPEND LIBEPOXY_INCLUDES include/epoxy/egl.h)
    set(PLATFORM_HAS_EGL 1)
    libepoxy_generate_dispatch("EGL" ${python} "${CMAKE_SOURCE_DIR}/src/gen_dispatch.py" --dir "${CMAKE_BINARY_DIR}" "${CMAKE_SOURCE_DIR}/registry/egl.xml")
endif()

if(EPOXY_BUILD_GLX)
    list(APPEND LIBEPOXY_SOURCES src/dispatch_glx.c)
    list(APPEND LIBEPOXY_INCLUDES include/epoxy/glx.h)
    set(PLATFORM_HAS_GLX 1)
    libepoxy_generate_dispatch("GLX" ${python} "${CMAKE_SOURCE_DIR}/src/gen_dispatch.py" --dir "${CMAKE_BINARY_DIR}" "${CMAKE_SOURCE_DIR}/registry/glx.xml")
endif()

if(EPOXY_BUILD_WGL)
    list(APPEND LIBEPOXY_SOURCES src/dispatch_wgl.c)
    list(APPEND LIBEPOXY_INCLUDES include/epoxy/wgl.h)
    set(PLATFORM_HAS_WGL 1)
    libepoxy_generate_dispatch("WGL" ${python} "${CMAKE_SOURCE_DIR}/src/gen_dispatch.py" --dir "${CMAKE_BINARY_DIR}" "${CMAKE_SOURCE_DIR}/registry/wgl.xml")
endif()
if(EPOXY_BUILD_GL)
    set(PLATFORM_HAS_OPENGL 1)
    libepoxy_generate_dispatch("OpenGL" ${python} "${CMAKE_SOURCE_DIR}/src/gen_dispatch.py" --dir "${CMAKE_BINARY_DIR}" "${CMAKE_SOURCE_DIR}/registry/gl.xml")
endif()
configure_file(
    "${CMAKE_SOURCE_DIR}/epoxy_config.cmake.in"
    "${CMAKE_BINARY_DIR}/include/epoxy/config.h"
)

file(GLOB_RECURSE LIBEPOXY_GENERATED_HEADERS ${CMAKE_BINARY_DIR}/include/*.h)
file(GLOB_RECURSE LIBEPOXY_GENERATED_SOURCES ${CMAKE_BINARY_DIR}/src/*.c)


include_directories(
    src
    include
    ${CMAKE_BINARY_DIR}/include
    )

add_library(libepoxy SHARED 
    ${LIBEPOXY_SOURCES} ${LIBEPOXY_INCLUDES}
    ${LIBEPOXY_GENERATED_SOURCES} ${LIBEPOXY_GENERATED_HEADERS})
add_dependencies(libepoxy libepoxy_dispatch)