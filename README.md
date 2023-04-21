# Contents
- [Overview](https://github.com/CloudRenderVR/Manuals#the-big-picture)
- [Client](https://github.com/CloudRenderVR/Manuals#client)
- [Server](https://github.com/CloudRenderVR/Manuals#server)
- [Pose capture](https://github.com/CloudRenderVR/Manuals#pose-capture)
- [Pose prediction](https://github.com/CloudRenderVR/Manuals#pose-prediction)

# The big picture

![image](https://user-images.githubusercontent.com/18013792/162675063-855730b8-779e-49d6-8c49-e007fd31a572.png)

Note that the pose prediction code is our least-developed area, and for now we're hoping to just bypass it (sending pose data from the capture directly to the client).

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

Note: dynamic linking requirements are as follows for hardware acceleration (opencv-cuda and ffmpeg-nvdec)
 > LD_LIBRARY_PATH=/usr/local/lib:/usr/lib/ffmpegNvidia2

## Profiling

We used two main forms of profiling for this project, Tracy and Nsight Systems. For Tracy profiling, make sure `Profiling = true` in the build script. After that, just run the client from the commandline and attach the server application. You can either compile the server yourself from the [tracy source code](https://github.com/wolfpld/tracy), just make sure the protocol versions line up. There is an existing compiled executable with the correct protocol on the Pikespeak machine, named `TracyServer.exe`, a shortcut to it is on the desktop.

For Nsight profiling, the following is a convience script to generate an Nsight trace:
```
nsys profile --accelerator-trace=nvmedia --trace=cuda,opengl,nvtx,nvmedia --process-scope=system-wide ./build/bin/client
```
Which can be ran from the client root directory. Running this creates either a `.qdrep` or `.nsys-rep` file, which you can then move over to the Pikespeak machine (or some other compute with Nsight Systems) via something like WinSCP. Opening this files on a Windows machine allows you to inspect the trace data. NOTE: there is a bug with Nsight Systems version 2022.3.3 where profiling the VIC accelerator will crash nsys when generating the report. To fix this, run profiling under `sudo` permissions.

# Server

The server code lives [here](https://github.com/CloudRenderVR/Server). We just provide our custom `PixelStreaming` module, instead of a full Unreal Engine source build. To build the server, see the readme in the server repo. Note I believe we used unreal engine [4.27](https://github.com/EpicGames/UnrealEngine/tree/4.27).

To get the server running, just create a new Unreal Engine project (we used the fps template), and make sure to enable the `Pixel Streaming` plugin in the project. After that, you should be able to simply package your project and run it, which will automatically start up our streaming system, provided you gave the proper commandline arguments (see server repo readme for details). The server provides various logs to the console for debugging connections and streaming.

# Pose capture

Pose capture uses the intel realsense depth camera to isolate a BODY25 skeleton of the human in frame. Detailed information on calibrating the camera is located in the [PoseExtracton_DepthCamera repo README](https://github.com/CloudRenderVR/PoseExtraction_DepthCamera/blob/master/README.md).

*Important setup notes:*

1. For some reason, running `main.py` from powershell did not function for me initially, throwing memory shape errors in OpenCV internals, to fix this, I ran the code from VSCode (made a launch.json and ran as a python file).
2. To get the model data, go to the root directory of the pose repo, and run `git clone https://github.com/CMU-Perceptual-Computing-Lab/openpose`. Then, navigate to `openpose/models` and run `getModels.bat` or `getModels.sh`.
3. In addition to a cuda-enabled OpenCV build (tested 4.5.5), you'll need zlib. Linux is a standard build via cmake, but on Windows I had to rename the dll. Steps for getting zlib on windows:
    1. Download `https://zlib.net/zlib1212.zip`, or get a more recent version if available from `zlib.net`.
    2. Open cmake-gui, pointing to the folder that contains the CMakeLists.txt file (the root directory of zlib). Make a folder called "build" in this folder and set that build directory to be the build output directory in cmake-gui.
    3. Configure and Generate.
    4. Open the solution files and either build all or just build `zlib` and `install`.
    5. Copy `zlib.dll` from the bin directory either into your system path or just right next to `main.py`.
    6. Rename `zlib.dll` to `zlibwapi.dll`.

# Pose reprojection

Pose reprojection takes the BODY25 skeleton and reprojects it to a H99 skeleton. The file responsible for this transition is located in the human motion prediction distro, specifically under the PythonVersionSafety branch. This is required because the model only accepts H99 poses. 

# Pose prediction

Pose Prediction is done statistically using a seq2seq model (takes a sequence and outputs a sequence). This model has been edited to focus on the eye movement, which isn't specifically recorded in the skeleton pose. To ensure everything works make sure to use the PythonVersionSafety branch, which has been edited to ensure that running it with python is possible (master includes some python3 commands). src/continuous prediction is our file which isolates head movement and iterates through the dataset. As well, if predicting on the Xavier device, ensure to use the command: 'source h-motion/bin/activate' (while in the gitStuff directory) to use the proper python virtuelenv. 
