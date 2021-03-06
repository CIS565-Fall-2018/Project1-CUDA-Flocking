cmake_minimum_required(VERSION 3.1)

project(cis565_boids)

set(CL_PATH "D:/Program Files (x86)/Microsoft Visual Studio/2017/Community/VC/Tools/MSVC/14.16.27023/bin/Hostx86/x64/cl.exe") 

set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake" ${CMAKE_MODULE_PATH})

# Set up include and lib paths
set(EXTERNAL "external")
include_directories("${EXTERNAL}/include")
include_directories("${EXTERNAL}/src")
if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(EXTERNAL_LIB_PATH "${EXTERNAL}/lib/osx")
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    set(EXTERNAL_LIB_PATH "${EXTERNAL}/lib/linux" "/usr/lib64")
elseif(WIN32)
    if(${MSVC_VERSION} MATCHES "1916") 
        set(EXTERNAL_LIB_PATH "${EXTERNAL}/lib/win/vc2017") 
        if ("${CL_PATH}" MATCHES "NULL")
            MESSAGE("Error: please set CL_PATH to a correct path to cl.exe. Edit this in CMakeLists.txt under the project root.") 
        endif() 
        set(CUDA_HOST_COMPILER "${CL_PATH}") 
    elseif(${MSVC_VERSION} MATCHES "1900") 
        set(EXTERNAL_LIB_PATH "${EXTERNAL}/lib/win/vc2015")
    elseif(${MSVC_VERSION} MATCHES "1800")
        set(EXTERNAL_LIB_PATH "${EXTERNAL}/lib/win/vc2013")
    elseif(${MSVC_VERSION} MATCHES "1700")
        set(EXTERNAL_LIB_PATH "${EXTERNAL}/lib/win/vc2012")
    else()
        MESSAGE("Error: unsupported MSVC_VERSION: " ${MSVC_VERSION})
    endif()
endif()
link_directories(${EXTERNAL_LIB_PATH})
list(APPEND CMAKE_LIBRARY_PATH "${EXTERNAL_LIB_PATH}")

# Find up and set up core dependency libs

set(GLFW_INCLUDE_DIR "${EXTERNAL}/include")
set(GLFW_LIBRARY_DIR "${CMAKE_LIBRARY_PATH}")
find_library(GLFW_LIBRARY "glfw3" HINTS "${GLFW_LIBRARY_DIR}")

set(GLEW_INCLUDE_DIR "${EXTERNAL}/include")
set(GLEW_LIBRARY_DIR "${CMAKE_LIBRARY_PATH}")
add_definitions(-DGLEW_STATIC)
find_package(GLEW)

find_package(OpenGL)

set(CORELIBS
    "${GLFW_LIBRARY}"
    "${OPENGL_LIBRARY}"
    "${GLEW_LIBRARY}"
    )

# Enable C++11 for host code
set(CMAKE_CXX_STANDARD 11)

# Enable CUDA debug info in debug mode builds
list(APPEND CUDA_NVCC_FLAGS_DEBUG -G -g)

# OSX-specific hacks/fixes
if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    list(APPEND CORELIBS "-framework IOKit")
    list(APPEND CORELIBS "-framework Cocoa")
    list(APPEND CORELIBS "-framework CoreVideo")
endif()

# Linux-specific hacks/fixes
if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    find_package(X11 REQUIRED)
    list(APPEND CORELIBS ${X11_LIBRARIES} ${X11_Xrandr_LIB} ${X11_Xxf86vm_LIB} ${X11_Xi_LIB})
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -lX11 -lXxf86vm -lXrandr -lXi")
endif()

# Crucial magic for CUDA linking
find_package(Threads REQUIRED)
find_package(CUDA 10.0 REQUIRED)

set(CUDA_ATTACH_VS_BUILD_RULE_TO_CUDA_FILE ON)
set(CUDA_SEPARABLE_COMPILATION ON)

if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(CUDA_PROPAGATE_HOST_FLAGS OFF)
endif()

add_subdirectory(src)

cuda_add_executable(${CMAKE_PROJECT_NAME}
    "src/main.hpp"
    "src/main.cpp"
    )

target_link_libraries(${CMAKE_PROJECT_NAME}
    src
    ${CORELIBS}
    )

add_custom_command(
    TARGET ${CMAKE_PROJECT_NAME}
    PRE_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
        ${CMAKE_SOURCE_DIR}/shaders
        ${CMAKE_BINARY_DIR}/shaders
    )
