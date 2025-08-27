# Nexus Embedded Ecosystem

*Contract-based and modular Ecosystem for Embedded Development*

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

## ⚠️ Challenges

So far I have identified the following challenges, that I will be experimenting on different ways to address them:

- **🛠️ This is not "an SDK"** - While the architecture offers some interesting advantages, it's far from "an SDK". This means that the development workflow depends for now heavily on what hal-implementation you are using. For example:
  - If using `nexus-hal-esp32-idf`, you will have access to idf tooling, allowing you to flash and debug using Python-based scripting
  - If using `nexus-hal-cmsis`, you will have to delve more into low level details to flash and debug your device, likely reusing some linker script and openocd config file.

- **🖥️ IDE integration is manual**
  - If you want to use a full IDE to develop Software through this ecosystem, you will not have an extension to install everything nor make your IDE recognize all dependencies.
    I personally just instruct CMake to export the build commands in a JSON format to a location I know of, and then create a `.clangd` file indicating where it should find it. This works at least with VS code, Zed editor, and most Vim/Nvim setups. (You would still need to personalize your IDE interaction if you want to build/flash/debug using only GUI) → I will add a guide on how to do this later on

- **⚖️ Design trade-offs** - There's a clear tension and trade-off relationship between some design choices:
  - **Portability** vs **Easiness of use**
  - **Easiness of adding support for new MCUs/HAL impl./etc** vs **Tooling complexity and abstraction**
  - **Standardizing build/flash/debug ops among different hal implementations** vs **Freedom of customization**

→ [Full challenges & potential solutions](docs/challenges.md) (🗓️ Planned)

---

## 📚 Current Components

| Repository | Purpose | Status |
|------------|---------|---------|
| [`nexus-hal-interface`](https://github.com/Fo-Zi/nexus-hal-interface) | 📐 Core HAL API definitions | ✅ Active |
| [`nexus-hal-esp32-idf`](https://github.com/Fo-Zi/nexus-hal-esp32) | ⚡ ESP32 IDF implementation | ✅ Active |
| [`nexus-eeprom-24c32`](https://github.com/Fo-Zi/nexus-eeprom-24c32) | 🔩 EEPROM driver | ✅ Active |
| [`nexus-ci`](https://github.com/Fo-Zi/nexus-ci) | 🐳 Docker Images for CI | ✅ Active |
| [`environmental-monitor`](https://github.com/Fo-Zi/environmental-monitor) | 🎯 Project Application example | 🗓️ Planned |

More platforms and drivers will come as the architecture stabilizes.

→ [Component compatibility matrix](docs/ecosystem-registry.md) (🗓️ Planned)

---

## 🔄 Development Philosophy

This is a work in progress. The interfaces will change, and I'll probably discover better approaches as I build more drivers and support more platforms. But that's the point - to experiment and learn what actually works in practice.

I will iterate until I find something that feels comfortable to work with, until that point, versioning will move fast. And once I get a stable enough API, I will release the first stable v1.0.0
