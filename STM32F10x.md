---
title: STM32F10x Development Environment
permalink: /setting-up/stm32f10x/
---

# Setting up the development environment for the STM32F10x with ST-Link

## Installation

### Eclipse IDE for C/C++
We'll be Eclipse as our primary software development tool for the STM32, every other tool will be configured to run through Eclipse.

* Download it from [the official website](https://www.eclipse.org/downloads/packages/) and extract it.
* Run `./eclipse`
* Open **Help -> Eclipse Marketplace** and install the **GNU MCU Eclipse** plugin.

### GCC Arm
The GNU Compiler Collection for the Arm architecture.

* Download it from [GNU Arm Embedded Toolchain](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm) and extract it.
	* You can either add the `bin` directory to the PATH
	* or set the "Toolchain Folder" in the GNU MCU Eclipse plugin settings at **Window -> Preferences -> MCU -> Global ARM Toolchain Paths**

### ST-Link
We'll be using an open source version of the ST Microelectronics ST-Link Tools that runs on Linux. This tool allow us to flash the chip.

Follow the installation instructions from their [README at GitHub](https://github.com/texane/stlink/blob/master/README.md).

After building it, copy `st-flash` to the `/usr/bin/` directory by running:
```
cp st-flash /usr/bin/
```

### OpenOCD
OpenOCD allow us to fully debug software directly on the chip.

Follow the installation instructions at their [instructions page](http://openocd.org/getting-openocd/).
Note: If you are building it from source, don't forget to enable ST-Link.

You also need to set the "Toolchain Folder" in the GNU MCU Eclipse plugin settings at **Window -> Preferences -> MCU -> Global OpenOCD Path**, which needs to point to OpenOCD `bin` directory.

## Compiling

Open Eclipse and create a new *C/C++ project* -> *C Managed Build* -> *STM32F10x C/C++ Project* -> *ARM Cross GCC*.

You may want to set "Use system calls" to "Semihosting" in case you need to print messages into the console.

You should be able to create and compile the project if the GNU MCU Eclipse plugin and the GCC Arm are both installed correctly.

## Debugging

Go to **Run -> Debug Configurations**, add a new launch configuration under *GDB OpenOCD Debugging* for the project open.

You'll need to locate two files inside the OpenOCD installation directory, they are usually located at `./interface/stlink-v2.cfg` and `./target/stm32f1x.cfg` inside the `tcl` or `scripts` directory. You should replace those paths below:
```
-f OPENOCD-PATH/interface/stlink-v2.cfg -f OPENOCD-PATH/target/stm32f1x.cfg
```
Set the parameters above to the **Config options** in the **Debugger** tab.

![OpenOCD Eclipse Config Options](img/openocd-debugger-config.png)

Connect the hardware, with each port mapped as listed below:

| st-linkv2 | stm32f103c8t6 |
| ------------- | ------------------- |
| SWDIO | DIO |
| GND | GND |
| SWCLK | CLK |
| 3.3V | 3.3V |

You should be able to run the newly created debug configuration if OpenOCD was installed and configured correctly. You can test it out by running it and adding breakpoints to your code.

## Flashing
You can configure the flashing process for release through Eclipse by following the instructions below.

Right click the project, go to **Properties -> C/C++ Build -> GNU ARM Cross Create Flash Image -> General** and set the "Output File Format" to "Raw Binary".

Compile the project in release mode (a release folder should be created)

Open **Run -> External Tools -> External Tools Configuration**, create a new configuration under "Program":

* Set the name to "st-link"
* Set the location to the st-link executable (run `whereis st-link`)
* Set the workspace to the project's release directory. (use Browse Workspace)
* Set the arguments to `write ${project_name}.bin 0x8000000`

You should be able to run the configuration to flash the device.

