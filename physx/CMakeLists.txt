## Redistribution and use in source and binary forms, with or without
## modification, are permitted provided that the following conditions
## are met:
##  * Redistributions of source code must retain the above copyright
##    notice, this list of conditions and the following disclaimer.
##  * Redistributions in binary form must reproduce the above copyright
##    notice, this list of conditions and the following disclaimer in the
##    documentation and/or other materials provided with the distribution.
##  * Neither the name of NVIDIA CORPORATION nor the names of its
##    contributors may be used to endorse or promote products derived
##    from this software without specific prior written permission.
##
## THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS ''AS IS'' AND ANY
## EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
## IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
## PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL THE COPYRIGHT OWNER OR
## CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
## EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
## PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
## PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY
## OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
## (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
## OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
##
## Copyright (c) 2008-2025 NVIDIA Corporation. All rights reserved.

# CPU-only, static PhysX for plain CMake consumers. No packman, no CUDA/GPU.
#
# Feeds the SDK the variables the packman/python generator normally injects, then
# builds it. Consume with FetchContent:
#
#   include(FetchContent)
#   FetchContent_Declare(physx
#       GIT_REPOSITORY https://github.com/erincatto/physx_cm.git
#       GIT_TAG        main
#       SOURCE_SUBDIR  physx)   # this file
#   FetchContent_MakeAvailable(physx)
#   target_link_libraries(your_target PRIVATE PhysX::CPU)
#
# Consuming in-tree makes the whole solution adopt PhysX's config names
# (debug;checked;profile;release). Use debug + release, ignore the rest.

cmake_minimum_required(VERSION 3.21)
project(PhysXCpu C CXX)

set(PHYSX_ROOT_DIR ${CMAKE_CURRENT_SOURCE_DIR})

if(WIN32)
    set(TARGET_BUILD_PLATFORM windows)
elseif(UNIX)
    set(TARGET_BUILD_PLATFORM linux)
else()
    message(FATAL_ERROR "PhysXCpu wrapper supports windows and linux only")
endif()

# x86 family. 64-bit selection comes from the toolchain (-A x64).
set(PX_OUTPUT_ARCH x86)

# GameWorks output roots. NvidiaBuildOptions FATAL_ERRORs without them.
set(PX_OUTPUT_LIB_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(PX_OUTPUT_BIN_DIR ${CMAKE_CURRENT_BINARY_DIR})

# CMP0077: plain variables here satisfy the option() calls inside the SDK.
set(PX_GENERATE_STATIC_LIBRARIES  ON)
set(PX_GENERATE_GPU_PROJECTS      OFF)
set(PX_GENERATE_GPU_PROJECTS_ONLY OFF)
set(PX_BUILDSNIPPETS              OFF)
set(PX_BUILDPVDRUNTIME            OFF)
set(PX_FLOAT_POINT_PRECISE_MATH   OFF)
set(PX_SCALAR_MATH                OFF)

# Dynamic CRT (/MD, /MDd), matching a typical consumer. Both sides must agree on
# the CRT and on _ITERATOR_DEBUG_LEVEL across the static link boundary.
set(NV_USE_STATIC_WINCRT          OFF)
set(NV_USE_DEBUG_WINCRT           OFF)

# The public tree ships as PUBLIC_RELEASE, which is mandatory here: a non public
# build references internal device sources that are stripped from this checkout.
# PUBLIC_RELEASE in turn copies a freeglut DLL used only by snippets, which we do
# not build. Point that copy at placeholder DLLs so nothing comes from packman.
if(WIN32)
    set(freeglut_stub ${CMAKE_CURRENT_BINARY_DIR}/freeglut_stub)
    file(WRITE ${freeglut_stub}/win64/freeglut.dll "")
    file(WRITE ${freeglut_stub}/win64/freeglutd.dll "")
    set(PHYSX_SLN_FREEGLUT_PATH ${freeglut_stub})
endif()

# compiler/public is the canonical public entry (sets PUBLIC_RELEASE=1). With GPU,
# snippets and pvdruntime all OFF it adds only the core SDK.
add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/compiler/public
                 ${CMAKE_CURRENT_BINARY_DIR}/physx_sdk)

# In static mode the low level libs, SceneQuery, SimulationController and
# PhysXTask are folded into PhysX, so consumers only need these six.
add_library(physx_cpu INTERFACE)
target_link_libraries(physx_cpu INTERFACE
    PhysX PhysXExtensions PhysXCooking PhysXCommon PhysXPvdSDK PhysXFoundation)
target_include_directories(physx_cpu INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/include)
target_compile_definitions(physx_cpu INTERFACE PX_PHYSX_STATIC_LIB)
add_library(PhysX::CPU ALIAS physx_cpu)
