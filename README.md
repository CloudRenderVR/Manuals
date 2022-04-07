# Contents
- [Client](https://github.com/CloudRenderVR/Manuals/edit/main/README.md#client)
- [Server](https://github.com/CloudRenderVR/Manuals/edit/main/README.md#server)
- [Pose capture](https://github.com/CloudRenderVR/Manuals/edit/main/README.md#pose-capture)
- [Pose prediction](https://github.com/CloudRenderVR/Manuals/edit/main/README.md#pose-prediction)

# Client

### The client is responsible for connecting to and establishing a video streaming connection (via H.264) with the server. Once a connection is made, it relays any input (such as pose data) to the server for rendering. When video frames are received, it first decodes, then reprojects, and finally displays on the local screen. The client software can be found [here](https://github.com/CloudRenderVR/Client).

## Overview

*High level overview of how the client works*

![ClientHighLevelArch](https://user-images.githubusercontent.com/18013792/162125973-2a864a35-d4b2-4fef-963f-2b05f9b08ff1.png)

*Available scheduling choices for the client on the Xavier NX*,  **OUT OF DATE: We now have working HW and cpu-only decoding**

![ClientSchedulingChoices2](https://user-images.githubusercontent.com/18013792/162126015-47daf6a4-ea71-48e1-a889-45762412814b.png)

## Installing

*While the client software should be able to run on other unix-based platforms, we have only tested on Xavier NX.*

First off, some necessary dependencies: `$ apt-get install ffmpeg libavcodec-dev libavformat-dev libswscale-dev libsfml-dev libopencv-dev`. This is the minimum required, and you'll need additional packages if you want, for example, software decoding. You'll also need to `$ git clone https://github.com/CloudRenderVR/Client` to get the actual repo.

## Configuring

There are various ways to change the behavior of the client, including the static scheduling choices. The primary way to alter the config is to edit `premake5.lua`. In this file, you'll see a `BUILD CONFIGURATION` section at the top, followed by a series of parameters that can be tweaked.
```lua
UseVPI = true  -- Set to false if VPI isn't available on the target platform. Falls back to OpenCV.
UseBuiltinDecoding = false  -- Set to true if we don't want a custom ffmpeg build.
UseHwAccelDecoding = false  -- For HW accel, must have UseBuiltinDecoding set to true. Will attempt to use cuda.
Debugging = false  -- Set to true if we want to debug the executable, this will make it perform considerably slower.
Profiling = true  -- Set to true if we want Tracy to compile and generate profiling zones.
```
These allow you to change between OpenCV vs VPI, and which ffmpeg decoding type to use, but how do we change if we want VPI on the cpu or VPI on the HW? To do this, all you have to do is navigate to [this line](https://github.com/CloudRenderVR/Client/blob/b198f1fc3c5cc28f036843bcec5d9fce12d696bb/CloudRenderVR/src/ReprojectVPI.cpp#L62) and change the `backend` variable to whatever VPI backend you want. Options are as follows: `VPI_BACKEND_CPU`, `VPI_BACKEND_VIC`, `VPI_BACKEND_CUDA`.

## Building

#### Note: you probably need sudo permissions to compile if you are configuring the build to make a custom FFmpeg build, since it needs to update linker paths!

Simply run the script `build.sh` which will make a build folder and compile for you (on Xavier NX or other ARM64 machines).

If you aren't compiling on ARM64, or want to build yourself, just run: `$ ./premake5[_x86] my_build_system`, where `my_build_system` is `gmake2` for unix-based systems. Note the two different executables, `premake5` and `premake5_x86`. The former is for ARM64 machines and the later for x86.

## Running

Build products are placed in the `build` folder when running the standard `build.sh` script. To run from your home directory, run the following command: `$ ./build/bin/client`. The client will then begin attempting to connect to the server, whose IP is hardcoded at [this line](https://github.com/CloudRenderVR/Client/blob/b198f1fc3c5cc28f036843bcec5d9fce12d696bb/CloudRenderVR/src/Main.cpp#L35).

## Profiling

We used two main forms of profiling for this project, Tracy and Nsight Systems. For Tracy profiling, make sure `Profiling = true` in the build script. After that, just run the client from the commandline and attach the server application. You can either compile the server yourself from the [tracy source code](https://github.com/wolfpld/tracy), just make sure the protocol versions line up. There is an existing compiled executable with the correct protocol on the Pikespeak machine, named `TracyServer.exe`.

For Nsight profiling, 

# Server

# Pose capture

# Pose prediction
