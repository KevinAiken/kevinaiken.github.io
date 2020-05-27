---
layout: post
title:  "My experience getting started with STM32"
date:   2020-05-27 18:06:24 -0400
categories: stm32
---

I recently got an STM32F407VG Discovery Kit:
![New STM32f407vg Discovery Kit Board in packaging](/assets/stm32f407vg_discovery_kit_in_package.jpg)
 
I started working with it following some of the guides at 
https://stm32-base.org/. I immediately ran into some difficulties. The documentation the kit comes with says when a 
Mini USB cable is connected from the board to a laptop, 4 LEDs between buttons B1 and B2 
should start blinking. I was unable to get this to work after trying two different mini USB cables, but I perservered, 
assuming I could still flash it, since the board appeared to be powered on.
 
First I setup my development environment following https://stm32-base.org/guides/setup. This took quite awhile. For 
reference, I did this on my laptop, a Dell XPS 15 9550 running Ubuntu 18.04.3. 

Following the directions in https://stm32-base.org/guides/flashing under Device name, I also needed to modify the 
makefile in ./STM32-base/templates/STM32-base-F4-template/Makefile:
```c
DEVICE = STM32F407xG
FLASH  = 0x08000000

USE_ST_CMSIS = true

# Include the main makefile
include ./STM32-base/make/common.mk

```
Line 1 specifies the correct linker, which for my device is the STM32F407xG linker.

At this point I was able to successfully run `make` to compile the application. However, after going through the guides
on STM32-base and a handful of other sites I still had no luck flashing my board.

A friend mentioned to me that a lot of mini USB cables do not carry data, so ordered a mini USB cable that
specifically said it had data lanes, https://www.amazon.com/gp/product/B00NH11N5A/. Once I received that, the board 
immediately worked as expected, blinking 4 LEDs between buttons B1 and B2. 

The output of `lsusb` confirmed that my laptop could see the STMicroelectronics ST-LINK embedded in the discovery kit, 
whereas previously the STM device did not show up:
```
➜  STM32-base-F4-template git:(master) ✗ lsusb
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 003: ID 04f3:21d5 Elan Microelectronics Corp. 
Bus 001 Device 002: ID 0a5c:6410 Broadcom Corp. 
Bus 001 Device 010: ID 0483:374b STMicroelectronics ST-LINK/V2.1 (Nucleo-F103RB)
Bus 001 Device 004: ID 0c45:6713 Microdia 
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

```
I was now able to flash the board successfully.
```
➜  STM32-base-F4-template git:(master) ✗ make flash
st-flash write ./bin/stm32_bin_image.bin 0x08000000
st-flash 1.6.0-162-gecbbd6d
2020-05-27T18:27:59 INFO common.c: F4xx: 192 KiB SRAM, 1024 KiB flash in 16 KiB pages.
2020-05-27T18:27:59 INFO common.c: Attempting to write 3016 (0xbc8) bytes to stm32 address: 134217728 (0x8000000)
Flash page at addr: 0x08000000 erased
2020-05-27T18:27:59 INFO common.c: Finished erasing 1 pages of 16384 (0x4000) bytes
2020-05-27T18:27:59 INFO common.c: Starting Flash write for F2/F4/L4
2020-05-27T18:27:59 INFO flash_loader.c: Successfully loaded flash loader in sram
enabling 32-bit flash writes
size: 3016
2020-05-27T18:27:59 INFO common.c: Starting verification of write complete
2020-05-27T18:27:59 INFO common.c: Flash written and verified! jolly good!
```

However, there still wasn't any blinking. The default program is written to blink PC13, which does not have a light
connected on my board. From looking at 
https://www.st.com/resource/en/user_manual/dm00039084-discovery-kit-with-stm32f407vg-mcu-stmicroelectronics.pdf 
section 6.3, I found that LD3 was on PD13, and would work well for my purposes. So all I needed to do was change the 
default program in provided with the 
STM32-base repo in ./STM32-base/templates/STM32-base-F4-template/src/main.c to say GPIOD instead of GPIOC:
```c
#include "stm32f4xx.h"

// Quick and dirty delay
static void delay (unsigned int time) {
    for (unsigned int i = 0; i < time; i++)
        for (volatile unsigned int j = 0; j < 2000; j++);
}

int main (void) {
    // Turn on the GPIOC peripheral
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIODEN;

    // Put pin 13 in general purpose output mode
    // Note: The only difference here is the name of the register in the
    //       definition, both lines have the same effect.
#if defined(STM32F413xx) || \
    defined(STM32F423xx)
    GPIOD->MODER |= GPIO_MODER_MODE13_0;
#else
    GPIOD->MODER |= GPIO_MODER_MODER13_0;
#endif

    while (1) {
        // Reset the state of pin 13 to output low
#if defined(STM32F413xx) || \
    defined(STM32F423xx)
        GPIOD->BSRR = GPIO_BSRR_BR_13;
#else
        GPIOD->BSRR = GPIO_BSRR_BR13;
#endif

        delay(500);

        // Set the state of pin 13 to output high
#if defined(STM32F413xx) || \
    defined(STM32F423xx)
        GPIOD->BSRR = GPIO_BSRR_BS_13;
#else
        GPIOD->BSRR = GPIO_BSRR_BS13;
#endif

        delay(500);
    }

    // Return 0 to satisfy compiler
    return 0;
}

```

After running `make` then `make flash`, I finally had a blinking light on my new board!
![Blinking STM32f407vg Discovery Kit Board](/assets/stm32f407vg_discovery_kit_blinky.gif)

I'm not sure what I'll do next, but I'm glad I finally confirmed my
board works, and my development environment is correctly setup.