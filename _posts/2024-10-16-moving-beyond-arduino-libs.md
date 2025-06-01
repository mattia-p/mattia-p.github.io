---
title: Moving beyond the pre-made arduino sensor libraries
excerpt: "Or how to demystify I2C communication by writing a sensor driver from scratch"
date: 2024-10-17
tags: [I2C, C++, Register, Raspberry Pi, Docker]
layout: post
---

# Moving beyond the pre-made arduino sensor libraries

Content
- The easy way
- Overview of the BME280 sensor
- I2C sensor library
- Integration

## The easy way

If you have ever plugged in any sensor to an arduino or an arduino, chances are you have used one of the available sensor libraries to interact with the sensor.

If you do that, things can get fairly simple and plug-and-play. For example if you have used the BME280 temperature and humidity sensor, the only thing you have to do is:

```cpp
#include "SparkFunBME280.h"
```

And then initialize and read your sensor's data with whatever way the provided examples tell you to:

```cpp
// Create sensor object
BME280 mySensor;

// Do some obscure initialization
mySensor.beginI2C();

// Read the sensor data
Serial.print(mySensor.readTempF(), 2);
```

Done. Now wrap this up into whatever use case you have in mind, and you don't really have to worry about your sensor again. Pretty easy. 

But just relying on the existing sensor library is not ideal if your use case is starting to become complex and involve lots of pieces, and you soon realize that to debug some bug you have to understand what happens behind that library and see if you can optimize it for you. 

What communication protocol is being used? What happens when I call the `readHumidity()` function? Is it blocking the rest of my code from running? How long does it take to run and why? 

I've asked myself these questions for a while but only recently decided to take a deeper look. And to apply these learnings, I've decided to write by own library for 

## Overview of the BME280 sensor

The BME280 is a temperature, humidity and pressure sensor developed by Bosch. The sensor is substantially small so for this project I have been using a version already soldered to a breakout board provided by Sparkfun.
For communication, the sensor provides a SPI and an I2C interface. I will be using the I2C interface throughout this article. 


![Breakout board](/assets/post_1/annotated_breakout_board.jpg){:style="width: 300px; height: auto; display: block; margin: 0 auto;"}
<p style="text-align: center; font-size: 0.9em; color: #555;">Sparkfun BME280 breakout board</p>

Most of the information that we need to interact with the sensor is provided in [the Bosch datasheet](https://cdn.sparkfun.com/assets/learn_tutorials/4/1/9/BST-BME280_DS001-10.pdf). 

Here's a simplified diagram of the internal architecture found in the datasheet.

![BME280 block diagram](/assets/post_1/bme280_block.png){:style="width: 400px; height: auto; display: block; margin: 0 auto;"}
<p style="text-align: center; font-size: 0.9em; color: #555;">BME280 block diagram</p>


The main components of the sensor are:
- An **oscillator (OSC)**, to provide a clock signal to the rest of the system.
- Some **power related components**: a voltage regulator, some power pins...Sparkfun BME280 breakout board.
- The **Non Volatile Memory (NVM)** to store factory programmed calibration data.
- The **interface logic**, to connect to other devices on the I2C or SPI buses.
- The three **sensing elements** (pressure, humidity, temperature), connected to an ADC (Analog-to-Digital Converter).
- The **logic component**, which manages the overall functioning of the sensor. This includes managing the different operational modes, handling the processing of the raw data, **register management** ...

But what we are really interested in and what our driver will mostly handle is **the registers**. 


## I2C sensor library

This section describes

### Overview

### Initialization

The sensor driver object is initialized in its constructor. `open` from the `fcntl` library opens a file used to interact with the i2c bus. In my case, `i2cDevice = "/dev/i2c-1"` and `O_RDWR` we are going to both read and write to that file. 
When it comes to i2c communication, the computer connected to the sensor is considered the master and the sensor is a slave. `ioctl` sets the i2c slave address: `0x77`. 

```cpp
BME280::BME280(const char* i2cDevice, uint8_t address) : i2cAddress(address){
    
    // Open i2c bus
    i2cFile = open(i2cDevice, O_RDWR); // O_RDWR (open for reading and writing)
    if (i2cFile < 0)
    {
        std::cerr << "Failed to open the i2c bus \n";
        exit(1);
    }

    // Set the i2c slave address
    if (ioctl(i2cFile, I2C_SLAVE, i2cAddress) < 0) // I/O control. 
    {
        std::cerr << "Failed to acquire bus access or talk to the BME280\n";
        exit(1);
    }
}
```

### Configuration

After we have initiated the i2c communication, the function below parametrizes the sensor by setting some configuration registers to specific values. 
```cpp
bool BME280::begin(){
    
    try {
        writeRegister(BME280_REG_CTRL_HUM, 0x01);  // Humidity oversampling x1
        writeRegister(BME280_REG_CTRL_MEAS, 0x27); // Normal mode, oversampling x1 for temperature
        writeRegister(BME280_REG_CONFIG, 0xA0); // Set config (filter off, standby)
    }catch(const std::exception& e){
        std::cerr << "Caught an exception: " << e.what() << std::endl;
        exit(1);
    }

    return true;
}
```

### Temperature reading

After the above is done, we are left to read the sensor values. Here, as an example, we will just look at the temperature reading. 
This is a two steps process: The first step consists in reading the raw temperature value, and the second in compensating the raw temperature using a compensation formula given in the datasheet. The compensation formula is supposed to correct for known raw data discrepencies, caused by the manufacturing process, environmental conditions, or the aging of the sensor.

![Temperature reading flow](/assets/post_1/read_temp_diagram.png){:style="width: 500px; height: auto; display: block; margin: 0 auto;"}
<p style="text-align: center; font-size: 0.9em; color: #555;">BME280 temperature reading flow</p>


During the first step, the sensor driver reads the raw temperature value stored in three 8 bits registers, forming a 24 bits value. The last 4 bits are not used, so only the first 20 bits are actually holding information we care about. 

The four functions used for reading the temperature can be found below (except the compensation function, which is quite long and not praticularly interesting):

```cpp
float BME280::readTemperature(){
    int32_t rawTemp = readRawTemperature();
    return compensateTemperature(rawTemp);
}

int32_t BME280::readRawTemperature(){
    // The temperature is a 20-bit value spread accross 3 registers
    uint32_t rawTemp = read20(BME280_REG_TEMP_MSB);
    return rawTemp >> 4; // The least significant 4 bits are not part of the temperature
}

uint32_t BME280::read20(uint8_t reg){
    uint8_t buffer[3];

    if (write(i2cFile, &reg, 1) != 1){
        std::cerr << "Failed to write to the register\n";
    }

    if (read(i2cFile, buffer, 3) != 3){
        std::cerr << "Failed to read from register\n";
    }

    return (buffer[0] << 16 | buffer[1] << 8 | buffer[2]);
}
```


![BME280 temp data registers layout](/assets/post_1/temperature_registers_layout.png){:style="width: 500px; height: auto; display: block; margin: 0 auto;"}
<p style="text-align: center; font-size: 0.9em; color: #555;">BME280 temperature data registers layout</p>


To form a 20 bits (coded on 32 bits) raw temperature value, we shift left the most import byte (on `0xFA`) by 16 bits, shift left the least important byte (`0xFB`) by 8 bits and discard the least significant bits of the extra least important byte (`0xFC`). 

After getting the raw temperature, we apply the compensation formulas obtained from the datasheet, which themselves also query for some values in registers, and in the end return the final temperature value that we are interested in.


## Integration

The record and save the temperature throughout time, I have implemented the following architecture.

![Software architecture](/assets/post_1/architecture.png){:style="width: 650px; height: auto; display: block; margin: 0 auto;"}
<p style="text-align: center; font-size: 0.9em; color: #555;">BME280 integration architecture</p>

The c++ binary `i2c_cpp_driver` repeats the same actions every second. It reads the temperature from the BME280 and then publishes it through a pipe for `main.py` to be able to read it through inter process communication. The python file then saves it to a local SQLite database. The database is also continuously read by the flask app that plots the temperature values into a local web browser.


The entire code for this project can be found [in my github repo](https://github.com/mattia-p/bme280_i2c_driver/tree/master). 