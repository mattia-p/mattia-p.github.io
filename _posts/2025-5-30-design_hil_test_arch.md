---
title: Designing a Hardware In the Loop test architecture
description: Or how to convince your robot it’s not in your living room 
date: 2025-5-30
image: /assets/post_4/post3_hil_arch.png
tech_stack: [HIL testing, Pytest, Simulation]
layout: post
---

# Designing a Hardware In the Loop test architecture
## Or how to convince your robot it’s not in your living room

<br>

**Table of content**

1. Summary
2. Introduction and goals
3. High level architecture of the planned HIL setup
4. Software architecture
5. Next steps

## Summary

This post outlines the design and architecture of a custom Hardware-in-the-Loop (HIL) test setup for an autonomous RC race car. The goal is to safely and systematically validate the Vehicle Control Interface (VCI)—the system's low-level control unit—by simulating real-world inputs and monitoring outputs in a controlled environment. The HIL setup enables repeatable, automated testing, fault injection, and closed-loop simulation without needing the full vehicle, significantly improving development speed and reliability.

## Introduction and goals

I've been working on building an autonomous RC race car for some time now. Built on an RC car chassis, the system integrates GPS and IMU sensors, and uses two compute units: a microcontroller for low-level control, and a more powerful CPU for high-level processing.

A high level diagram of the vehicle architecture can be found below:

<a href="/assets/post_4/system_arch.png" target="_blank">
  <img src="/assets/post_4/system_arch.png" alt="Image 1: Autonomous RC car high level diagram"
       style="width: 600px; height: auto; display: block; margin: 0 auto;" />
</a>
<p style="text-align: center; font-size: 0.9em; color: #555;">Image 1: Autonomous RC car high level diagram</p>

The **Central Control Unit** is responsible for running the autonomy stack, including localization, planning, and control. The **Vehicle Control Interface**, on the other hand, translates high-level commands from the Central Control Unit into low-level actuator commands (powertrain, steering, handbrake). It also handles mode switching between autonomous and manual control by listening to the RC receiver and monitoring the engagement channel for mode-change commands.

For testing, I’ve primarily relied on the following approaches:

- **Software-in-the-Loop (SIL) testing**: In combination with a custom-built simulation tool, this setup allows me to run the autonomy stack while simulating GPS sensor input and feedback from the Vehicle Control Interface. While this enables verification of the stack’s nominal behavior on the actual Central Control Unit hardware, it doesn’t support fault injection or hardware-level validation.
- **Log replay**: This approach lets me replay previously recorded sensor data logs through the autonomy stack, enabling analysis and debugging using real-world scenarios without having to re-run tests on the vehicle.
- **Vehicle testing**: Conducted with the full hardware stack on the real vehicle, this is the most realistic form of testing. However, it is also the most time-consuming, risk-prone, and resource-intensive—especially when faults are present or edge cases need to be evaluated.

While these test platforms are valuable, they come with limitations: none are well-suited for isolating and testing the **Vehicle Control Interface** on its own, and none support **fault injection**. This creates a gap in my testing strategy—particularly when trying to derisk full vehicle testing, where failures can lead to costly damage or setbacks.
To address this, I’ve been working on a new **Hardware-in-the-Loop (HIL)** test setup that can simulate sensor inputs and actuator feedback in real time, enabling safer, more targeted testing of the Vehicle Control Interface and autonomous behaviors under both nominal and faulty conditions.

## High level architecture of the planned HIL setup

The primary goal of this new HIL setup is to build a test bench that can replicate all the real-world inputs the Vehicle Control Interface would encounter, while also observing and verifying its outputs—and critically, enabling the injection of faults to test edge-case behavior and system robustness.

With this goal in mind, the following high-level architecture was designed to meet the functional and testing requirements of the HIL setup.

<a href="/assets/post_4/post3_hil_arch.png" target="_blank">
  <img src="/assets/post_4/post3_hil_arch.png" alt="Image 2: HIL high level architecture"
       style="width: 700px; height: auto; display: block; margin: 0 auto;" />
</a>
<p style="text-align: center; font-size: 0.9em; color: #555;">Image 2: HIL high level architecture</p>

Let’s now dive deeper into each component of the architecture to understand its role and how it interacts with the rest of the system.

**Vehicle Control Interface (VCI):**
This is the Device Under Test—the actual hardware and firmware we want to validate. It's identical to what would be deployed on a real vehicle during field testing. In this HIL setup, it receives simulated sensor inputs and generates actuator commands just as it would in a live environment.

**Tester PC**:
The Tester PC acts as the central orchestrator of the entire test bench. In this case, it’s a Raspberry Pi 4 running Ubuntu 24.04, and it takes on several key responsibilities:
- It hosts the main test infrastructure and executes the test logic, using tools like  pytest alongside a custom testing framework to manage all test components.
- Its primary role is to coordinate the test: setting it up, executing it, tearing it down, and injecting faults when necessary to validate system robustness.
- It also runs a vehicle simulation, which models the behavior of a virtual vehicle (position, velocity, feedback signals, etc.) based on the current inputs. These simulated states are used to generate sensor signals for the VCI.
- Finally, it hosts the Central Control Unit Simulator (CCU SIM)—a software instance running nearly the full autonomy stack, just as it would on the real onboard computer. While it doesn't run on the same hardware or receive real sensor input, the core software components remain unchanged. The CCU SIM communicates with the Vehicle Control Interface over rosserial, preserving the same interface and communication pathway used in the production system.

**DAQ (Data Acquisition):**
The DAQ is a custom Arduino Mega-based interface that acts as the physical link between the Tester PC and the Vehicle Control Interface. It serves both as a signal generator and a monitoring tool, enabling real-time interaction with the VCI through standard electrical interfaces.

On the **input side**, it simulates signals from key vehicle sensors and controls:
- It generates three PWM signals—**steering, throttle, and hand brake**—that emulate commands from a typical RC receiver, as if a human driver were in control.
- It also simulates an **IMU (Inertial Measurement Unit)** by sending fake sensor readings (e.g., acceleration and angular velocity) over the I2C bus, mimicking a real sensor interface.

On the **output side**, the DAQ passively listens to the actuator commands generated by the VCI:
- It reads back the PWM output signals for steering and throttle, allowing the Tester PC to monitor how the VCI reacts to different inputs and scenarios.

Through this bidirectional setup, the DAQ enables closed-loop testing between simulated inputs and observed outputs—essential for validating the control logic and for enabling fault injection experiments.

**Limitations**
While this HIL setup provides a powerful and flexible test environment, there are a few limitations to be aware of:
- **Latency and Timing Accuracy**: The system relies on general-purpose hardware and operating systems (like Linux on the Raspberry Pi), which may introduce unpredictable latency and jitter, especially for real-time signal generation and sensing.
- **Sensor Fidelity**: Simulated sensor data (e.g., IMU over I2C) may not fully capture the noise, drift, and edge cases of real-world sensors, limiting realism.
- **Simplified Autonomy Stack**: The CCU Simulator uses a simplified version of the real autonomy code, which may behave differently from the production software under certain conditions.
- **Hardware Constraints**: The DAQ is implemented on an Arduino Mega, which may have limitations in terms of processing power, memory, and I/O bandwidth when simulating multiple real-time signals simultaneously.
- **Scalability**: Expanding the setup to support more sensors, actuators, or higher-frequency signals may require more robust hardware or optimized code.
These limitations don't prevent valuable testing, but they do highlight the importance of eventually validating the system in more realistic scenarios and on the full production hardware.

## Software architecture

With the high-level hardware architecture in place, it's time to dive into the software system that ties everything together. The following diagram outlines how the autonomy software, vehicle simulation, DAQ, and Vehicle Control Interface (VCI) interact under the supervision of a centralized test infrastructure.


<a href="/assets/post_4/hil_sw_arch.png" target="_blank">
  <img src="/assets/post_4/hil_sw_arch.png" alt="Image 3: HIL software overview"
       style="width: 900px; height: auto; display: block; margin: 0 auto;" />
</a>
<p style="text-align: center; font-size: 0.9em; color: #555;">Image 3: HIL software overview</p>

Each component plays a specific role in enabling Hardware-in-the-Loop (HIL) testing. Let’s take a closer look at how they work together:

**Test infrastructure**
At the core of the system is the test infrastructure, which orchestrates each test scenario. Built around the pytest framework in Python, it provides a flexible environment for writing and executing automated tests. Its responsibilities include:
- Launching all other components in the correct initial state
- Verifying that all systems are active and communicating
- Injecting faults where necessary
- Shutting down the setup gracefully
- Reporting test results
This infrastructure acts as the conductor of the system—ensuring coordination, repeatability, and observability throughout the test cycle.

**Autonomy software**
The autonomy software is nearly identical to what runs on the real Central Control Unit. While I initially considered running a simplified replica to ease fault injection and reduce complexity, I ultimately decided to run the full autonomy stack. The system is still compact enough for this to be manageable, and doing so keeps the tests closer to reality.
The only component excluded is GPS data acquisition, since there's no real sensor in the loop. Instead, the GPS input is emulated by the vehicle simulation module.

**Vehicle simulation**
The vehicle simulation replaces both the vehicle’s physical dynamics and its GPS sensor input. It receives actuator commands—throttle, steering, and brake—via the DAQ, which reads PWM signals from the VCI and forwards them to the tester PC.
Using a basic vehicle model, the simulation computes the vehicle's updated state (e.g., position, velocity) and feeds that information back into the autonomy software. This closes the loop and enables a realistic control-feedback cycle, without needing to move an actual vehicle.

## Next steps

This post has focused on the early design and architecture phase of the HIL system. The next major milestone is building the full setup—both hardware and software. That includes designing and assembling the custom PCB, wiring all components, and implementing the remaining code for orchestration and simulation.
Future posts will document this process and share lessons learned, along with initial test results from the completed platform.
