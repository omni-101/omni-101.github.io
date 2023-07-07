---
layout:     post
title:      Omniverse PhysX vcpkg port
summary:    Deep diving into Omniverse PhysX vcpkg port
categories: omniverse
tags:
 - omniverse
 - physx
 - physics
---

The newest version of the open source [NVIDIA Omniverse PhysX](https://github.com/NVIDIA-Omniverse/PhysX/), much like its predecessors, supports a large variety of compiler switches, configuration presets and dependency management infrastructure. Sometimes this complexity can be quite intimidating/frustrating to newcomers willing to experiment with it (maybe when considering adoption in a bigger physics project).

<figure>
  <a href="/assets/images/physx_linking_errors_diorama.png" target="_blank">
    <img src="/assets/images/physx_linking_errors_diorama.png" alt="Users having issues with PhysX linking">
  </a>
  <figcaption style="text-align: center; font-size: small; font-style: italic;">Compiling/linking PhysX properly can be nontrivial</figcaption>
</figure>

Luckily users can now use a [rather involved contributed vcpkg port](https://github.com/microsoft/vcpkg/pull/31506) to compile and link Omniverse PhysX with GPU acceleration support directly into their projects with just a few lines of CMake code.

Here's a simple `hello_physx_vcpkg` project stub:

* CMakeLists.txt

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.15)

# CUSTOMIZE THESE IF NEEDED BEFORE IMPORTING THE PHYSX PACKAGE
##############################################################
# set(VCPKG_TARGET_TRIPLET x64-windows)
# set(VCPKG_CRT_LINKAGE dynamic)

# set(VCPKG_TARGET_TRIPLET x64-windows-static)
# set(VCPKG_CRT_LINKAGE static)

set(VCPKG_TARGET_TRIPLET x64-linux)
##############################################################

# IMPORTANT: set the following variable to the location of your vcpkg local installation
set(CMAKE_TOOLCHAIN_FILE "$ENV{HOME}/vcpkg/scripts/buildsystems/vcpkg.cmake")

project(HelloPhysX)

add_executable(hello_physx main.cpp)

# Note: if the package cannot be found here, check that you're using the right triplet
find_package(unofficial-omniverse-physx-sdk CONFIG REQUIRED)
target_link_libraries(hello_physx PRIVATE unofficial::omniverse-physx-sdk::sdk)

# Optional: import the defined target to copy over the GPU acceleration libraries
# (3rd party provided by NVIDIA)
if(TARGET unofficial::omniverse-physx-sdk::gpu-library)
    if(UNIX)
        # Add rpath setting to find so libraries on unix based systems
        set_target_properties(hello_physx PROPERTIES
            BUILD_WITH_INSTALL_RPATH TRUE
            INSTALL_RPATH "$ORIGIN"
        )
    endif()
    add_custom_command(TARGET hello_physx POST_BUILD
                    COMMAND ${CMAKE_COMMAND} -E copy_if_different
                    $<TARGET_FILE:unofficial::omniverse-physx-sdk::gpu-library>
                    $<TARGET_FILE_DIR:hello_physx>)
    if(WIN32)
        add_custom_command(TARGET hello_physx POST_BUILD
                        COMMAND ${CMAKE_COMMAND} -E copy_if_different
                        $<TARGET_FILE:unofficial::omniverse-physx-sdk::gpu-device-library>
                        $<TARGET_FILE_DIR:hello_physx>)
    endif()
else()
    message(WARNING "\GPU acceleration library target not defined
 - GPU acceleration will NOT be available!\
")
endif()
```

* main.cpp

```cpp
#include "PxConfig.h"
#include "PxPhysicsAPI.h"
#include <iostream>

using namespace physx;

int main(int argc, char** argv) {
    std::cout << "Starting PhysX up.." << std::endl;
    PxDefaultAllocator allocator;
    PxDefaultErrorCallback error_callback;
    auto foundation = PxCreateFoundation(PX_PHYSICS_VERSION, allocator, error_callback);

    // Create a CUDA context manager
    PxCudaContextManagerDesc cudaContextManagerDesc;
    auto cudaContextManager = PxCreateCudaContextManager(
        *foundation, cudaContextManagerDesc, PxGetProfilerCallback()
    );

    // Check if CUDA context manager is valid
    if(cudaContextManager && cudaContextManager->contextIsValid()) {
        std::cout << "GPU acceleration is available" << std::endl;
    } else {
        std::cout << "GPU acceleration is not available" << std::endl;
    }

    // Create a physics SDK object with GPU acceleration
    PxTolerancesScale tolerancesScale;
    PxSceneDesc sceneDesc(tolerancesScale);
    sceneDesc.gravity = PxVec3(0.0f, -9.81f, 0.0f);
    auto dispatcher = PxDefaultCpuDispatcherCreate(4);
    sceneDesc.cpuDispatcher = dispatcher;
    sceneDesc.filterShader  = PxDefaultSimulationFilterShader;
    sceneDesc.cudaContextManager = cudaContextManager;
    sceneDesc.flags |= PxSceneFlag::eENABLE_GPU_DYNAMICS;
    sceneDesc.broadPhaseType = PxBroadPhaseType::eGPU;
    auto physics_sdk = PxCreatePhysics(
        PX_PHYSICS_VERSION, *foundation, tolerancesScale, true, nullptr
    );
    auto scene = physics_sdk->createScene(sceneDesc);

    std::cout << "PhysX set up" << std::endl;

    // Release resources
    PX_RELEASE(scene);
    PX_RELEASE(dispatcher);
    PX_RELEASE(physics_sdk);
    PX_RELEASE(cudaContextManager);
    PX_RELEASE(foundation);

    std::cout << "Shutting down.." << std::endl;
}
```

After that it's as simple as

```bash
# Git-checkout, compile and install newest PhysX locally
# One could also use 'vcpkg install omniverse-physx-sdk --triplet linux-x64' here
~/hello_physx_vcpkg$ vcpkg install omniverse-physx-sdk
...
~/hello_physx_vcpkg$ mkdir build
~/hello_physx_vcpkg$ cd build
~/hello_physx_vcpkg/build$ cmake -DCMAKE_BUILD_TYPE=Release .. && make
~/hello_physx_vcpkg/build$./hello_physx
Starting PhysX up..
```

or similarly on Windows

```bash
# optionally specify vcpkg-root if cannot be found in PATH
# --triplet x64-windows/x64-windows-static --vcpkg-root C:\Users\user\vcpkg
~/hello_physx_vcpkg$ vcpkg install omniverse-physx-sdk
...
~/hello_physx_vcpkg$ mkdir build
~/hello_physx_vcpkg$ cd build
~/hello_physx_vcpkg/build$ cmake -DCMAKE_BUILD_TYPE=Release ..
~/hello_physx_vcpkg/build$ cmake --build . --config Release
~/hello_physx_vcpkg/build$./Release/hello_physx
Starting PhysX up..
```

The port is rather involved and will be community maintained and updated (**this is NOT an official NVIDIA supported port**). For any issues please open a GitHub issue to the vcpkg official repo and hopefully someone will take care of it.

_Please note that this port's name is `omniverse-physx-sdk` and not just `physx`. Reason for this distinction being that the latter is a community maintained port of the previous generation of the PhysX engine supporting other platforms which were removed from the official supported list in the latest Omniverse PhysX (focused on the Omniverse platform experience). Users can now choose their preferred port for a new project while older projects relying on the `physx` port should not get any disruption by this change._