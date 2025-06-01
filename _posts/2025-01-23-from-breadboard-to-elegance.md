---
title: From breadboard chaos to Raspberry Pi zero elegance - Downsizing and solidifying our IoT solution
excerpt: "Or how to architect a ligthweight, containerized sensor pipeline"
date: 2025-1-23
tags: [Docker, Raspberry Pi, Flask, IPC, C++, Python]
layout: post
---

# From breadboard chaos to Raspberry Pi zero elegance: Downsizing and solidifying our IoT solution

<br>

**Content**
- Introduction
- Another platform, another OS
- Migrating the codebase from the Pi4 to the Pi Zero
- No more jumper wires
- Put it in a box

## Introduction

In a previous post, I described how I built a C++ I2C driver for the **BME280** sensor (temperature, pressure, humidity). That driver was then used to record and store temperature data regularly and was being published on a flask web app for visualization. What I didn't describe in detail is what the hardware setup looked like. 
I was running all the software on a Raspberry Pi 4. The sensor was installed on a breadboard and connected to the Raspberry Pi with 4 jumper wires (4 wires because of I2C) and the Raspberry Pi itself was connected to my laptop with an ethernet RJ45 cable. 


That solution was not optimal for many reasons, including:
- The jumper wires can easily be disconnected
- The raspberry pi 4 is probably overkill that such a small application
- The raspberry pi (and hence the sensor) has to be physically close to my laptop

I've decided to miniaturize the whole setup and I am documenting the whole process in this blog post. 

Some of the goals I would like to achieve are:
- Miniaturize the hardware, to make it less expensive and so we can position the sensor in more places
- Stop using jumper wires and the breadboard
- Change the OS from Ubuntu Desktop to Ubuntu Server


## Another platform, another OS

The first modification I started working on is the change of computing platform. I decided to go with a Raspberry Pi Zero 2W instead of the Raspberry Pi 4, because:
- It's a lot less expensive (~$15)
- It's a more appropriate platform for the low-power sensor and small software we are running
- It can still run Ubuntu, so porting the software shouldn't be too hard
- It has wifi embedded, so we can remotely ssh into it, without needing additional cables
- It has GPIO pins, including I2C pins, so the connections should be the same as with the Pi 4


![Breakout board](/assets/post_2/pi_zero.png){:style="width: 400px; height: auto; display: block; margin: 0 auto;"}
<p style="text-align: center; font-size: 0.9em; color: #555;">Raspberry Pi Zero 2W</p>


Here are some of the steps I followed to start using the Pi Zero:

I've used Raspberry Pi Imager to create a bootable SD card with Ubuntu Server. The software already has a variety of OS to choose from, and is easy to use. Don't forget to configure your network setting if you are planning remotely ssh-ing into it, and never use a screen. 

![Breakout board](/assets/post_2/imager.png){:style="width: 600px; height: auto; display: block; margin: 0 auto;"}
<p style="text-align: center; font-size: 0.9em; color: #555;">Raspberry Pi Imager</p>


Once the image file is ready, insert the disk into the Pi zero, connect your Pi to power. The pi will automatically connect to your network and should be visible on your network.

Ssh into it the Pi:

```bash
ssh user@<IP address>
```
Make sure you have the latest package lists and installs available updates: 
- apt update
- apt upgrade

And you should be good to go. The pi should be ready to install the software next.

![Breakout board](/assets/post_2/ubuntu.png){:style="width: 800px; height: auto; display: block; margin: 0 auto;"}
<p style="text-align: center; font-size: 0.9em; color: #555;">Ubuntu server 24.04</p>


## Migrating the codebase from the Pi4 to the Pi Zero

In this paragraph I'll be documenting how I migrated [the codebase](https://github.com/mattia-p/bme280_i2c_driver) from the Pi 4 to the Pi Zero.
In theory not much is needed since the app runs on a Docker container, so the only two packages that need to be installed locally are Docker and git. 


Once this is done, I'm able to clone my repo and start the container:

```bash
docker compose up -d
```
```bash
docker exec -it bme280_i2c_driver-cpp-dev-1 /bin/bash
```
Once in the container, I attempted to build the software the same way I used to:

```bash
bazel build //src/cpp_driver:bme280_cpp_driver_executable
```

Using the above command ended up never working, or at least not finishing in a reasonable amount of time. Using Bazel on the Raspberry Pi Zero was slow and ultimately unsuccessful because the Pi Zero's limited processing power and memory struggled to handle the demands of Bazel, which is optimized for more powerful systems.

I have also tried the following, without more success. The --jobs option specifies the maximum number of concurrent jobs (tasks) Bazel can run during the build process. Running only one job at a time reduces CPU and memory consumption, preventing the system from being overloaded.

```bash
bazel build --jobs=1 //src/cpp_driver:bme280_cpp_driver_executable
```

I ended up stopping to use bazel, and changed everything to CMake, which is more lightweight and allows for simpler configuration for resource-constrained environments. 

Changing everything to Cmake was pretty short, since the only file we are actually building is the C++ driver. Here is what the CMakeLists.txt looks like:

```cmake
cmake_minimum_required(VERSION 3.16)
project(BME280Sensor)

# Set the C++ standard
set(CMAKE_CXX_STANDARD 17)

# Set the include directory for the cpp_driver library
include_directories(src/cpp_driver)

add_library(bme280_cpp_driver
	src/cpp_driver/bme280_cpp_driver.cpp
)

# Add the executable from main.cpp
add_executable(bme280_cpp_driver_executable
	src/cpp_driver/main.cpp
)

# Link the executable with the bme280_cpp_driver library
target_link_libraries(bme280_cpp_driver_executable bme280_cpp_driver)

```

Once this was done, I had to make a couple of adjustments, including:
- Changing how I build and start the executables from the start.sh script, to use CMake instead of Bazel
- Modifying where I store the local sqlite3 database, which was previously stored in a bazel-specific folder before

The last thing remaining now is to make the app reachable from anywhere in the network, which I did by binding the web app server to all available network interfaces with:

```python
app.run(host="0.0.0.0", port=5000, debug=True)
```
And it works fine now! 

## No more jumper wires

Next step is to get rid of the jumper wires and the breadboard. It might be overkill for such a small sensor and use case, but I've decided to make a custom PCB shield to securely mount and connect the sensor. Designing the PCB was pretty straightforward using EasyEda.

![PCB](/assets/post_2/pcb.png){:style="width: 800px; height: auto; display: block; margin: 0 auto;"}
<p style="text-align: center; font-size: 0.9em; color: #555;">PCB</p>

A couple of minutes of soldering and this is what I was left with:

![Circuit](/assets/post_2/circuit.jpg){:style="width: 500px; height: auto; display: block; margin: 0 auto;"}
<p style="text-align: center; font-size: 0.9em; color: #555;">Pi zero with custom BME280 shield</p>

## All together

![Architecture diagram](/assets/post_2/arch_diagram.png){:style="width: 700px; height: auto; display: block; margin: 0 auto;"}
<p style="text-align: center; font-size: 0.9em; color: #555;">Architecture diagram</p>