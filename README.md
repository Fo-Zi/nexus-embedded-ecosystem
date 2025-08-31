# Nexus Embedded Ecosystem

*Contract-based and modular Ecosystem for Embedded Development*

---

## Contents
- [🎯 What is it](#-what-is-it)
  - [Quick Simple Example](#quick-simple-example)
- [💭Motivation and Goals](#-motivation-and-goals)
  - [🔍Motivation](#-motivation)
  - [🎯Goals](#-goals)
- [🏛️Architecture](#️-architecture)
- [🛠️Development Workflow & Tooling](#️-development-workflow--tooling)
  - [Unified Build System](#unified-build-system)
- [📚Current Components](#-current-components)
- [🚀Roadmap](#-roadmap)
  - [Phase 1: Foundation & Core Interface ✅](#phase-1-foundation-and-core-interface)
  - [Phase 2: Basic Drivers and HAL implementation ✅](#phase-2-basic-drivers-and-hal-implementation)
  - [Phase 3: More Drivers and Lower Level HAL Implementation 🏗️](#phase-3-more-drivers-and-a-lower-level-hal-implementation)
  - [Phase 4: Framework Adapter HAL Implementation 🗓️](#phase-4-framework-adapter-hal-implementation)
  - [Phase 5: Portable Project, Documentation and First stable release 🗓️](#phase-5-portable-project-documentation-and-first-stable-release)
  - [Optional Advanced Features 🔮](#optional-advanced-features)
- [⚠️Challenges](#-challenges)
- [🔄Last words](#-last-words)

---

## 🎯 What is it
I call it "Ecosystem", because it started merely as a shim HAL but quickly mutated to be more than that.
This project is composed by the following independent components:
- **Hardware Interface** (`nexus-hal-interface`) → An isolated repo ; Only contains platform-independent interfaces ('the contracts')
- **Platform Implementation** ( i.e.: `nexus-hal-esp32-idf`, `nexus-hal-cmsis`, etc ) → Isolated repos ; Each constitutes an implementation for the `nexus-hal-interface`
- **Drivers** (i.e.: `nexus-eeprom-24c32`) → Drivers built on top of **the interface** ;
- **Complementary tooling** (i.e.: `nexus-ci`) → Other components that ease the development workflow ;

### Quick Simple Example
```c
// This BME280 driver works on ESP32, STM32, etc:
#include "nhal_i2c_master.h"  // ← Only depends on interface

static const uint8_t BME280_REG_TEMP_MSB = 0xFA; // ← Defined at some bme280_reg_defs.h or similar

bme280_result_t bme280_read_temp(bme280_handle_t *sensor, float *temp) {
    uint8_t data[3];
    nhal_result_t result = nhal_i2c_master_write_read_reg(
        sensor->i2c_ctx,     // ← NHAL I2C Context you provide
        sensor->dev_addr,    // ← I2C device address (some devices allow setting it via hardware pins)
        &BME280_REG_TEMP_MSB, 1
        data, sizeof(data)
    );

    // Process data...
    return BME280_OK;
}
```

No platform #ifdefs, no vendor-specific APIs.

→ [Complete working example](https://github.com/Fo-Zi/nexus-environmental-monitor) ((🗓️ Planned))

---

## 💭 Motivation and Goals

### 🔍 Motivation

While working with Embedded Systems I have faced some pain-points:

- **🧪 Vendor HALs and others are not usually designed with testability in mind**
  - I have found myself writing shim layers of abstraction for peripherals in order to be able to unit-test some complex driver or component
    (You can of course do it, but I found myself repeating this common step many times)

- **🔌 Drivers were not designed around a common interface**
  - If the driver uses something like dependency injection, for example for the hardware functionality it needs, you'd often find yourself having to duplicate a ton of code. Imagine having ten I2C devices, whose drivers all use dependency injection with different function signatures..

- **🔒 The HAL your code depends on, is tied to a specific Framework/MCU/SDK**
  - This goes hand-by-hand with the first point
  - The porting efforts become bigger → Your code and your drivers need some degree of refactoring
  - Usually I have found also that HALs for RTOSes are radically different from HALs for Bare Metal. While this has of course good reasons to be like that, I often found that most drivers depend on the same functionality, what changes could be if you need to take a mutex before accessing a shared bus as a master or not (for example), or some other implementation specific detail.

- **🏗️ Some HALs are very complex and big**
  - For some projects these HALs are an overkill, even if more generic than others

All of these points and probably others that I am missing, pushed me to create this project and experiment with it.

→ [Detailed pain points & experiences](docs/motivation.md) (🗓️ Planned)

---
### 🎯 Goals

- **Provide platform-independent interfaces that drivers can depend on**
  - Drivers only depend on the nexus-hal-interface, and are transparent to platform-details.
- **Not to force a specific Implementation of the HAL interface**
  - I will provide basic implementations for me to test the approach on a few platforms. You can use those, or
    provide your own implementations with the focus you need (performance, memory footprint, power consumption, etc)
  - This way, you can prototype using a common interface, and if you later need to optimize its implementation, you can.
- **To design the contracts as 'allocation free'**
  - The HAL implementation and consumer determine the allocation scheme. You may want to use a static or dynamic approach, that's up to you.
- **To leave room for extension and implementation specific details**
  - It's not possible to create config/context structures that are applicable **anywhere**, there is always some niche case that you haven't considered.

---

## 🏛️ Architecture

The ecosystem separates the **what** from the **how**:

```
Your Application
    ↓
Nexus Drivers (BME280, DS3231, etc.) ← Only depend on interface
    ↓
nexus-hal-interface ← Contract definitions (header-only)
    ↓
nexus-hal-esp32 / nexus-hal-stm32 ← Platform implementations
```

**Key principles**:
- **Contract-based**: Drivers depend on interface contracts, not implementations
- **Implementation freedom**:
  - The interface doesn't force allocation patterns. Implementations can use mutexes, static locks, interrupt disabling, or pools - whatever fits their platform and your needs.
  - Each platform can optimize for its needs (RTOS vs bare metal, performance vs size, etc.)
- **Context ownership**: You own and manage all contexts and configuration structures - HAL **initializes** your memory, doesn't allocate it for you

**Platform Integration Layer**:
- This is your hardware-specific glue code - pin assignments, clock configs, etc.
- Not part of the Ecosystem, since it cannot be defined generically. But examples and templates will be provided that help to
  isolate: platform-specific vs platform-agnostic (🗓️ Planned)

→ [Detailed architecture & design decisions](docs/architecture.md) (🗓️ Planned)
→ [Core principles & contracts](docs/core-principles.md) (🗓️ Planned)

---

## 🛠️ Development Workflow & Tooling

While the ecosystem's modular architecture offers flexibility, I recognize that **unified tooling** is essential for productivity. To address the "this is not an SDK" challenge mentioned above, I've developed West extension commands that provide consistent build and flash operations across all ecosystem components.

### Unified Build System

I've integrated [West](https://docs.zephyrproject.org/latest/west/index.html) to automatically detect project types and provide consistent commands. The commands are implemented in the [`nexus-hal-interface`](https://github.com/Fo-Zi/nexus-hal-interface) repository - while this might seem odd at first, it makes sense because this is the common dependency shared across all projects in the ecosystem:

```bash
# Build any project in your workspace
west build my-sensor-driver
west build stm32-firmware

# Flash to target with automatic tool detection
west flash --runner openocd    # For projects with OpenOCD configs
west flash --runner stflash    # For STM32 with ST-Link
west flash --runner espidf     # For ESP32 projects

# List all available projects
west list-projects
```

**What this gives you:**
- **Automatic project detection** - Supports ESP-IDF, CMake, and Make projects
- **Extensible flash runners** - OpenOCD, ST-Link, Make targets, ESP-IDF tools
- **Custom runner support** - Add your own flash tools and configurations
- **Cross-platform consistency** - Same commands work across different target platforms

This approach means you can work with STM32, ESP32, and other platforms using the same commands, while each platform retains its specific optimizations and tooling under the hood.

→ **[Complete West Commands Documentation](WEST_COMMANDS.md)** - Comprehensive guide with examples, runner configuration, and extending the system

---

## 📚 Current Components

| Repository | Purpose | Status |
|------------|---------|---------|
| [`nexus-hal-interface`](https://github.com/Fo-Zi/nexus-hal-interface) | 📐 Core HAL API definitions | ✅ Active |
| [`nexus-hal-esp32-idf`](https://github.com/Fo-Zi/nexus-hal-esp32) | ⚡ ESP32 IDF implementation | ✅ Active |
| [`nexus-eeprom-24c32`](https://github.com/Fo-Zi/nexus-eeprom-24c32) | 🔩 EEPROM driver | ✅ Active |
| [`nexus-dht11`](https://github.com/Fo-Zi/nexus-dht11) | 🔩 Sensor driver | ✅ Active |
| [`nexus-ci`](https://github.com/Fo-Zi/nexus-ci) | 🐳 Docker Images for CI | ✅ Active |
| [`environmental-monitor`](https://github.com/Fo-Zi/environmental-monitor) | 🎯 Project Application example | 🗓️ Planned |

More platforms and drivers will come as the architecture stabilizes.

→ [Component compatibility matrix](docs/ecosystem-registry.md) (🗓️ Planned)

---

## 🚀 Roadmap

### Phase 1: Foundation and Core Interface
✅ **Completed**

**Goal**: Establish the core interfaces, and necessary scripts to get a buid going

#### 1.1 Core HAL Interface
- ✅ **`nexus-hal-interface`**
  - I2C master operations (sync only)
  - SPI master operations (sync only)
  - GPIO pin control
  - UART basic operations
  - Common error types and timeouts
  - State management patterns

#### 1.2 Build System & Dependencies
- ✅ **CMake and Manifest files**
  - West manifest structure
  - CMake integration patterns
  - Versioning strategy

### Phase 2: Basic Drivers and HAL implementation 
✅ **Completed**

**Goal**: Test the interface with basic drivers and a high level HAL wrapper implementation ; Analyze the weak points of the interface, and
modify/iterate until reaching a comfortable while portable approach.

#### 2.1 High-Level Framework Wrapper
- ✅ **`nexus-hal-esp32-idf`**
  - Wraps ESP-IDF peripheral drivers
  - Maintains ESP-IDF ecosystem compatibility
  - Complete tooling integration (idf.py build/flash/monitor)

#### 2.2 I2C EEPROM Driver (eeprom-24c32)
- ✅ **`nexus-eeprom-24c32`**
  - Implement the driver, depending exclusively on the `nexus-hal-interface`
  - Implement basic unit-tests for it, to validate the driver and mocks of the interface
  - Analyze how comfortable is the `I2C interface` to use

#### 2.3 DHT11 Driver
- ✅ **`nexus-dth11`**
  - Implement the driver, depending exclusively on the `nexus-hal-interface`
  - Implement basic unit-tests for it, to validate the driver and mocks of the interface
  - Analyze how comfortable are the `pin` and `delay interfaces` to use

### Phase 3: More Drivers and a Lower Level HAL Implementation
🏗️ **In progress**

**Goal**: Test the interface with drivers that depend on other interfaces, as well as a Bare Metal HAL implementation ; Analyze how the HAL interface fits and interact with
a low level implementation

#### 3.1 Bare-Metal Implementation 🏗️ **In progress**
- [ ] **`nexus-hal-stm32f407vg`** (or similar popular MCU)
  - Register-level implementation (No external HALs, no framework)
  - Basic necessary complementary tooling to build/flash/debug:
    - Linker scripts
    - OpenOCD configuration
    - Startup code
    - CMake integration with flash/debug targets

#### 3.2 Passive Buzz (PWM) Driver 🗓️ *Planned*
- [ ] **`nexus-006ky`**
  - This will depend on `PWM interface`, which will need to be implemented on `nexus-hal-interface`
  - An implementation of `PWM interface` needs then to be developed
  - Then the driver can be tested both through software and hardware

### Phase 4: Framework Adapter HAL Implementation 
🗓️ **Planned**

**Goal**: Support an MCU through CMSIS, and design a scalable approach to ease and facilitate adding support for new MCUs already supported in the framework

#### 4.1 CMSIS Framework Adapter
- [ ] **`nexus-hal-cmsis-core`**
  - Support at least one MCU through [CMSIS core](https://arm-software.github.io/CMSIS_5/Core/html/index.html)
  - Create a template system to facilitate adding support for new MCUs

#### 4.2 Photoresistor (ADC) Driver 🗓️ *Planned*
- [ ] **`nexus-018ky`**
  - This will depend on `ADC interface`, which will need to be implemented on `nexus-hal-interface`
  - An implementation of `ADC interface` needs then to be developed
  - Then the driver can be tested both through software and hardware

### Phase 5: Portable Project, Documentation and First stable release
🗓️ **Planned**

**Goal**: To create a project based on the Ecosystem, showing the Platform Integration Layer, SOLID principles, and
how to build the project for different supported platforms ; To document the approach and patterns used ; To release the first
stable version of the HAL Interface!

#### 5.1 Showcase Project 🗓️ *Planned*
- [ ] **`nexus-showcase-project`**
  - A project that showcases how the Ecosystem was envisioned
  - To provide easy-to-use commands to compile and change the target platform
  - To apply SOLID principles throughout the project, and how they fit with the ecosystem
  - To use a modern approach, with CI,automated testing,automated doc, etc

#### 5.2 Documentation 🏗️ *In progress*
- [ ] **`nexus-embedded-ecosystem`**
  - General Documentation:
      - Architecture
      - Design decisions and tradeoffs
      - Basic Guides
  - To document the Showcase Project and Patterns used

#### 5.3 🚀 First Stable Version Release v1.0.0 🗓️ *Planned*
- [ ] **`nexus-hal-interface` v1.0.0**
  - At this point, there should be enough experimentation to be
    able to release the first stable version.
  - Does it mean that the interface will not evolve since this point?
    No. It means:
    - Core interfaces have been tested and will likely not be modified.
    - Advanced or more specific interfaces can be designed and added, as long as
      they don't break other components (CI will be in place for this)


### Optional Advanced Features
🔮 **Future**
**Goal**: There are some things I'd like to explore, which would complement or strengthen the ecosystem; Advanced drivers that likely depend on async interfaces ; Explore other interfaces that HALs usually don't bother on abstracting but make sense to do so ; With all 
the learned lessons of implementing the interface in C, create a Modern C++ variant and compare results: How much template metaprogramming allows fine grain control while remaining generic, resulting binary sizes, runtime performance ,etc.

#### Display Driver (ST7789 TFT) 🔮 *Future*
- [ ] **`nexus-st7789`**
  - Implement the driver, depending exclusively on the `nexus-hal-interface`
  - Implement basic unit-tests for it, to validate the driver and mocks of the interface
  - Analyze how comfortable is the `SPI interface` to use

#### Power Management Interface 🔮 *Future*
- [ ] **`nexus-hal-interface`**
  - Power management is usually HIGHLY dependant on the platform, but in practice, usually not more
    than 3-4 power modes are used (Like for example: normal, light-sleep, deep-sleep, stand-by ).
  - An interesting idea would be to abstract these modes, and to provide some way for the
    Platform Integration Layer to decide on "the how" (i.e. what set of peripherals,clocks, and interrupts are
    enabled/disabled for each)

#### Modern C++ Approach 🔮 *Future*
- [ ] **`nexus-hal-interface-cpp`,`nexus-hal-***-cpp`**
  - To create a pure modern C++ interface version of the NHAL. Using features like: Namespaces ,Templates,
    template specialization, concepts, constexpr, variants, optionals, etc.
  - Compile-time code generation and metaprogramming would be a great fit for frameworks that
    need to be generic/well abstracted, while not losing efficiency in the process.
  - The Platform Integration Layer could be implemented in a much cleaner way using this approach. Even
    with some code generation

---

## ⚠ Challenges
So far I have identified the following challenges, that I will be experimenting on different ways to address them:

- **🛠️ This is not "an SDK"** - While the architecture offers some interesting advantages, it's far from "an SDK". This means that the development workflow depends for now heavily on what hal-implementation you are using. For example:
  - If using `nexus-hal-esp32-idf`, you will have access to idf tooling, allowing you to flash and debug using Python-based scripting
  - If using `nexus-hal-cmsis`, you will have to delve more into low level details to flash and debug your device, likely reusing some linker script and openocd config file.
- **🖥️ IDE integration is manual**
  - If you want to use a full IDE to develop Software through this ecosystem, you will not have an extension to install everything nor make your IDE recognize all dependencies.
    I personally just instruct CMake to export the build commands in a JSON format to a location I know of, and then create a `.clangd` file indicating where it should find it. I will add a guide on how to do this later on.
- **⚖️ Design Tradeoffs**
  - **Genericity** vs **Easeness of Implementation**:
    - Some drivers require fine grain control over hardware capabilities, and there's a clear tension between exposing hardware details
      and making the interfaces generic. For example:
      - The DHT11 requires to set the same pin as input and output. The input requires either an internal or external pullup:
        - If the interface allows to set the input pullup (available in most platforms, but not for all pins), the driver would work
        just fine, and at initialization, a flag indicating if internal or external pullup should be used can be set.
        - The problems arise in the details: What if by accident the driver gets assigned a pin that doesn't support this
        and no external pullup is used? Yes, **the driver would fail silently**, probably by timeout.
        - If the HAL would want to expose hardware capabilities, and then drivers evaluate if the implementation
        supports the features it needs, then each HAL implementation would require a huge effort (Describing and exposing
        hardware details at a per-peripheral basis)
  - 


→ [Full challenges & potential solutions](docs/challenges_and_solutions.md) (🗓️ Planned)

---

## 🔄 Last words

This is a work in progress. The interfaces will change, and I'll probably discover better approaches as I build more drivers and support more platforms. But that's the point - to experiment and learn what actually works in practice.

I will iterate until I find something that feels comfortable to work with, until that point, versioning will move fast. And once I get a stable enough API, I will release the first stable v1.0.0
