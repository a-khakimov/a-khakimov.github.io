---
layout: post
title: "STM32: How to start in Linux"
date: 2018-05-26 00:01:30 +0000
categories: [STM32]
tags: [microcontroller, stm32, stlink, stm32cubemx]
---

----------------

When developing software for STM32 microcontrollers, I usually include the following steps.

1. Generation of basic project on **STM32CubeMX**
2. Writing code in any comfortable text editor (I use [Sublime Text](https://www.sublimetext.com))
3. Compilation of code with use **arm-none-eabi-gcc**
4. Firmware of microcontroller with use **st-link**

The whole process of installation and start-up are described below.

![](/assets/img/stm32-how-start/stm32.jpg){: .dark .w-75 .shadow .rounded-10 w='500' }

----------------

## Stack

* STM32CubeMX
* ST-link

----------------

## STM32CubeMX

I prefer to work with STM32CubeMX. It allows you to quickly generate a project with the setting of the periphery and clock frequency of the microcontroller, as well as the ability to connect such cool things as [FreeRTOS](https://www.freertos.org/) or [lwIP](https://savannah.nongnu.org/projects/lwip/).

----------------

### Installation procedure

* Download [STM32CubeMX](https://my.st.com/content/my_st_com/en/products/development-tools/software-development-tools/stm32-software-development-tools/stm32-configurators-and-code-generators/stm32cubemx.license=1527168412147.html)
* For installation need unzip archive and perform of command

```bash
sudo ./SetupSTM32CubeMX-4.25.1.linux
```

* Add in environment variables 

```bash
STM32CUBEMX_HOME=~/.prog/STM32CubeMX
export PATH=$STM32CUBEMX_HOME:$PATH
```

* Restart **.bashrc**

```bash
source .bashrc
```

![cubemx](/assets/img/stm32-how-start/cubemx.png)

After configuring the peripherals and clock frequencies, need to generate a project. Also need to configure of the generation of the project. I set the generation **Makefile** and put a tick to generate *.c and *.h files for each peripheral.

![](/assets/img/stm32-how-start/proj-settings.png){: .dark .w-75 .shadow .rounded-10 w='550' }

You may have to wait for the required libraries to be downloaded.

![](/assets/img/stm32-how-start/load.gif){: .dark .w-75 .shadow .rounded-10 w='350' }

----------------

## ST-link

To firmware and debug the program on the microcontroller, you will need st-link. It can be cloned from [repository](https://github.com/texane/stlink) and assemble it yourself. I recommend installing from the repository of your distribution - it's faster.

Also will need a compiler and debugger to build:

* [arm-none-eabi-gcc](https://gnu-mcu-eclipse.github.io/toolchain/arm/install/)
* [arm-none-eabi-gdb](https://developer.arm.com/open-source/gnu-toolchain/gnu-rm/downloads)

They can also be taken from the repository.

After successful project generation, you will see a bunch of files and folders, that need to be compiled.

![sublime](/assets/img/stm32-how-start/sublime.png)

The first, need say **Makefile** is where your compiler is.

```bash
#######################################
# binaries
#######################################
BINPATH = /bin
PREFIX = arm-none-eabi-
```

After all this, need to build a project.

```bash
$ make
```

And fill firmware into the microcontroller using **st-link**.

You can use the graphical version of **STlink-GUI**.

```bash
$ stlink-gui
```

![STlinkGUI](/assets/img/stm32-how-start/stlink.png)

I prefer do it from console.

```bash
$ st-flash write ./build/how-run.bin 0x08000000
st-flash 1.5.0
2018-05-26T17:00:50 INFO common.c: Loading device parameters....
2018-05-26T17:00:50 INFO common.c: Device connected is: F76xxx device, id 0x10016451
2018-05-26T17:00:50 INFO common.c: SRAM size: 0x80000 bytes (512 KiB), Flash: 0x200000 bytes (2048 KiB) in pages of 2048 bytes
2018-05-26T17:00:50 INFO common.c: Attempting to write 4004 (0xfa4) bytes to stm32 address: 134217728 (0x8000000)
Flash page at addr: 0x08000000 erased
2018-05-26T17:00:50 INFO common.c: Finished erasing 1 pages of 32768 (0x8000) bytes
2018-05-26T17:00:50 INFO common.c: Starting Flash write for F2/F4/L4
2018-05-26T17:00:50 INFO flash_loader.c: Successfully loaded flash loader in sram
enabling 32-bit flash writes
size: 4004
2018-05-26T17:00:50 INFO common.c: Starting verification of write complete
2018-05-26T17:00:50 INFO common.c: Flash written and verified! jolly good!
```

----------------

## Obstacles, that may stand between you and STM32

----------------

### Make error

When you try to build a project, you will see the following error:

![make-eror](/assets/img/stm32-how-start/make-bug.png)

If you open **Makefile**, in block **sources** you will see repeated lines.

```bash
######################################
# source
######################################
# C sources
C_SOURCES =  \
/Src/system_stm32f7xx.c \
Src/main.c \
Drivers/STM32F7xx_HAL_Driver/Src/stm32f7xx_hal_cortex.c \
Drivers/STM32F7xx_HAL_Driver/Src/stm32f7xx_hal_gpio.c \
Src/main.c \
Src/stm32f7xx_it.c \
Drivers/STM32F7xx_HAL_Driver/Src/stm32f7xx_hal_dma_ex.c \
Drivers/STM32F7xx_HAL_Driver/Src/stm32f7xx_hal_pwr_ex.c \
Drivers/STM32F7xx_HAL_Driver/Src/stm32f7xx_hal_tim.c \
Src/gpio.c \
Src/stm32f7xx_hal_msp.c \
Src/stm32f7xx_it.c \
Drivers/STM32F7xx_HAL_Driver/Src/stm32f7xx_hal_dma.c \
Drivers/STM32F7xx_HAL_Driver/Src/stm32f7xx_hal_i2c.c \
Drivers/STM32F7xx_HAL_Driver/Src/stm32f7xx_hal_tim_ex.c \
Src/gpio.c \
Drivers/STM32F7xx_HAL_Driver/Src/stm32f7xx_hal_flash.c \
Drivers/STM32F7xx_HAL_Driver/Src/stm32f7xx_hal_i2c_ex.c \
Drivers/STM32F7xx_HAL_Driver/Src/stm32f7xx_hal_rcc.c \
Drivers/STM32F7xx_HAL_Driver/Src/stm32f7xx_hal.c \
Src/stm32f7xx_hal_msp.c \
Drivers/STM32F7xx_HAL_Driver/Src/stm32f7xx_hal_pwr.c \
Drivers/STM32F7xx_HAL_Driver/Src/stm32f7xx_hal_rcc_ex.c \
Drivers/STM32F7xx_HAL_Driver/Src/stm32f7xx_hal_flash_ex.c
```

In my case is following lines:

```bash
Src/main.c \
Src/gpio.c \
Src/stm32f7xx_it.c \
Src/stm32f7xx_hal_msp.c \
```

This lines need delete and project should be compiling. This is bug of code generation in **STM32CubeMX**. We wrote to the developers of **STM32CubeMX** about this bug. It remains to wait until they fix and release an update.

----------------

## No libraries

You may encounter the following error during build.

![make-eror](/assets/img/stm32-how-start/make-error.png)

Compiler say what he cannot find library **stdint.h**. For fix it error need install 

* **arm-none-eabi-newlib**

It can also be taken from the repository.

----------------
