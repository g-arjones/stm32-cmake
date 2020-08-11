# About

This project is used to develop applications for the STM32 - ST's ARM Cortex-Mx MCUs. 
It uses cmake and GCC, along with newlib (libc), STM32Cube. Supports F0 G0 L0 F1 L1 F2 F3 F4 G4 L4 F7 H7 device families.

## Requirements

* cmake >= 3.8
* GCC toolchain with newlib (optional).
* STM32Cube package for appropriate STM32 family.

## Project contains

* CMake toolchain file, that configures cmake to use the arm toolchain: [cmake/stm32_gcc.cmake](cmake/stm32_gcc.cmake).
* CMake module that contains useful functions: [cmake/stm32/common.cmake](cmake/stm32/common.cmake)
* CMake modules that contains information about each family - RAM/flash sizes, CPU types, device types and device naming (e.g. it can tell that STM32F407VG is F4 family with 1MB flash, 128KB RAM with CMSIS type F407xx)
* CMake toolchain file that can generate a tunable linker script [cmake/stm32/linker_ld.cmake](cmake/stm32/linker_ld.cmake)
* CMake module to find and configure CMSIS library [cmake/FindCMSIS.cmake](cmake/FindCMSIS.cmake)
* CMake module to find and configure STM32 HAL library [cmake/FindHAL.cmake](cmake/FindHAL.cmake)
* CMake project template and examples [examples](examples)
* Some testing project to check cmake scripts working properly [tests](tests)

## Examples

* `template` ([examples/template](examples/template)) - project template, empty source linked compiled with CMSIS.
* `custom-linker-script` ([examples/custom-linker-script](examples/custom-linker-script)) - similiar to `template` but using custom linker script.
* `blinky` ([examples/blinky](examples/blinky)) - blink led using STM32 HAL library and SysTick.

# Usage

First of all you need to configure toolchain and library pathes using CMake varibles. 
You can do this by passing values through command line during cmake run or by setting variables inside your CMakeLists.txt

## Configuration

* `TOOLCHAIN_PREFIX` - where toolchain is located, **default**: `/usr`
* `TARGET_TRIPLET` - toolchain target triplet, **default**: `arm-none-eabi`
* `STM32_CUBE_<FAMILY>_PATH` - path to STM32Cube directory, where `<FAMILY>` is one of `F0 G0 L0 F1 L1 F2 F3 F4 G4 L4 F7 H7` **default**: `/opt/STM32Cube<FAMILY>`

## Common usage

First thing that you need to do after toolchain configration in your `CMakeLists.txt` script is to find CMSIS package:
```
find_package(CMSIS COMPONENTS STM32F4 REQUIRED)
```
You can specify STM32 family or even specific device (`STM32F407VG`) in `COMPONENTS` or omit `COMPONENTS` totally - in that case stm32-cmake will find ALL sources for ALL families and ALL chips (you'll need ALL STM32Cube packages somewhere).

Each STM32 device can be categorized into family and device type groups, for example STM32F407VG is device from `F4` family, with type `F407xx`

CMSIS consists of three main components:

* Family-specific headers, e.g. `stm32f4xx.h`
* Device type-specific startup sources (e.g. `startup_stm32f407xx.s`)
* Device-specific linker scripts which requires information about memory sizes

stm32-cmake uses modern CMake features notably imported targets and target properties.
Every CMSIS component is CMake's target (aka library), which defines compiler definitions, compiler flags, include dirs, sources, etc. to build and propagates them as dependencies. So in simple use-case all you need is to link your executable with library `CMSIS::STM32::<device>`:
```
add_executable(stm32-template main.c)
target_link_libraries(stm32-template CMSIS::STM32::F407VG)
```
That will add include directories, startup source, linker script and compiler flags to your executable.

CMSIS creates following targets:

* `CMSIS::STM32::<FAMILY>` (e.g. `CMSIS::STM32::F4`) - common includes, compiler flags and defines for family
* `CMSIS::STM32::<TYPE>` (e.g. `CMSIS::STM32::F407xx`) - common startup source for device type, depends on `CMSIS::STM32::<FAMILY>`
* `CMSIS::STM32::<DEVICE>` (e.g. `CMSIS::STM32::F407VG`) - linker script for device, depends on `CMSIS::STM32::<TYPE>`

So, if you don't need linker script, you can link only `CMSIS::STM32::<TYPE>` library and provide own script using `stm32_add_linker_script` function

Also, there is special library `STM32::NoSys` which adds `--specs=nosys.specs` to compiler flags.

## HAL

STM32 HAL can be used similiar to CMSIS.
```
find_package(HAL COMPONENTS STM32F4 REQUIRED)
set(CMAKE_INCLUDE_CURRENT_DIR TRUE)
```
*`CMAKE_INCLUDE_CURRENT_DIR` here because HAL requires `stm32<family>xx_hal_conf.h` file being in include headers path.*

HAL module will search all drivers supported by family and create following targets:

* `HAL::STM32::<FAMILY>` (e.g. `HAL::STM32::F4`) - common HAL source, depends on `CMSIS::STM32::<FAMILY>`
* `HAL::STM32::<FAMILY>::<DRIVER>` (e.g. `HAL::STM32::F4::GPIO`) - HAL driver <DRIVER>, depends on `HAL::STM32::<FAMILY>`
* `HAL::STM32::<FAMILY>::<DRIVER>Ex` (e.g. `HAL::STM32::F4::ADCEx`) - HAL Extension driver , depends on `HAL::STM32::<FAMILY>::<DRIVER>`
* `HAL::STM32::<FAMILY>::LL_<DRIVER>` (e.g. `HAL::STM32::F4::LL_ADC`) - HAL LL (Low-Level) driver , depends on `HAL::STM32::<FAMILY>`

Here is typical usage:

```
add_executable(stm32-blinky-f4 blinky.c stm32f4xx_hal_conf.h)
target_link_libraries(stm32-blinky-f4 
    HAL::STM32::F4::RCC
    HAL::STM32::F4::GPIO
    HAL::STM32::F4::CORTEX
    CMSIS::STM32::F407VG
    STM32::NoSys 
)
```

### Building

```
    $ cmake -DCMAKE_TOOLCHAIN_FILE=<path_to_gcc_stm32.cmake> -DCMAKE_BUILD_TYPE=Debug <path_to_sources>
    $ make
```

## Linker script & variables

CMSIS package will generate linker script for your device automatically (target `CMSIS::STM32::<DEVICE>`). To specify a custom linker script, use `stm32_add_linker_script` function.

## Useful cmake function

* `stm32_get_chip_info(CHIP FAMILY TYPE DEVICE)` - classify device using name, will return device family, type and canonical name (uppercase without any package codes)
* `stm32_get_memory_info(FAMILY DEVICE CORE FLASH_SIZE RAM_SIZE CCRAM_SIZE STACK_SIZE HEAP_SIZE FLASH_ORIGIN RAM_ORIGIN CCRAM_ORIGIN)` - get information about device memories. Linker script generator uses values from this function
* `stm32_get_devices_by_family(FAMILY DEVICES)` - return into `DEVICES` all supported devices by family

