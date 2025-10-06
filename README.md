# Nexus Embedded Ecosystem

*Contract-based and modular ecosystem for embedded development*

A multi-platform embedded firmware architecture built to address vendor lock-in and driver portability. This project explores hardware abstraction patterns that enable writing firmware once and deploying across different MCU platforms (ESP32, STM32, etc.) without code changes.

Started as an exploration of better abstraction patterns in embedded systems, it has evolved into a working ecosystem I use for my own projects. The architecture separates HAL interfaces from platform implementations, demonstrating practical approaches to modular firmware design.

**Project components**: HAL interface definitions, platform implementations (ESP32 IDF, STM32 bare-metal), hardware-agnostic drivers, and unified build tooling using West.

→ [Getting Started Guide](docs/getting-started.md) | [Motivation & Philosophy](docs/motivation.md)

---

## Contents
- [🎯 What is it](#-what-is-it)
  - [Quick Simple Example](#quick-simple-example)
- [💭 Motivation and Goals](#-motivation-and-goals)
  - [🔍 Motivation](#-motivation)
  - [🎯 Goals](#-goals)
- [🏛️ Architecture](#️-architecture)
- [🛠️ Development Workflow & Tooling](#️-development-workflow--tooling)
  - [Unified Build System](#unified-build-system)
- [📚 Current Components](#-current-components)
- [🚀 Roadmap](#-roadmap)
  - [Phase 1: Foundation and Core Interface ✅](#phase-1-foundation-and-core-interface)
  - [Phase 2: Basic Drivers and HAL Implementation ✅](#phase-2-basic-drivers-and-hal-implementation)
  - [Phase 3: Bare-Metal HAL and Additional Drivers ✅](#phase-3-bare-metal-hal-and-additional-drivers)
  - [Phase 4: Integration Project and Documentation ✅](#phase-4-integration-project-and-documentation)
  - [Phase 5: Complex Real-World Application & v1.0.0 Release 🗓️](#phase-5-complex-real-world-application--v100-release)
  - [Future Exploration 🔮](#future-exploration)
- [⚠️ Challenges & Tradeoffs](#️-challenges--tradeoffs)
- [🔄 Closing Thoughts](#-closing-thoughts)
- [📚 Documentation](#-documentation)

---

## 🎯 What is it
The ecosystem separates hardware abstraction into independent, composable components:

- **HAL Interface** (`nexus-hal-interface`) - Platform-independent API definitions (the contracts)
- **Platform Implementations** (`nexus-hal-esp32-idf`, `nexus-llhal-stm32f103`) - Concrete implementations for specific platforms
- **Hardware Drivers** (`nexus-dht11`, `nexus-eeprom-24c32`) - Drivers built against the interface, not specific platforms
- **Build Tooling** - Unified commands across different build systems and platforms

This separation emerged from experimenting with different abstraction patterns—what started as a simple HAL shim evolved into a multi-repository architecture that balances flexibility with practical constraints.

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

No platform #ifdefs, no vendor-specific APIs. The driver is portable by design.

→ [Complete showcase project](https://github.com/Fo-Zi/nexus-showcase-project) - Multi-platform firmware example
→ [Getting Started Guide](docs/getting-started.md) - Build and run in 5 minutes

---

## 💭 Motivation and Goals

### 🔍 Motivation

Working across different embedded platforms, I kept hitting the same issues:

**Testability**: Vendor HALs aren't designed for unit testing. I'd write abstraction shims for every project just to test drivers in isolation—a pattern I was repeating too often.

**Driver portability**: Each I2C device driver expected different function signatures. Ten devices meant ten different integration patterns. There was no common contract.

**Platform lock-in**: Switching from ESP-IDF to bare-metal STM32 meant rewriting not just peripheral code, but drivers too. RTOS-based HALs and bare-metal HALs looked completely different, even though most drivers need the same basic operations.

**Unnecessary complexity**: Some HAL layers are heavyweight for simple use cases. I wanted something modular—use what you need, skip what you don't.

These problems became the design constraints for this project.

→ [Detailed experiences & design rationale](docs/motivation.md)

---
### 🎯 Goals

**Platform-independent driver development**: Drivers depend on interface contracts, not platform specifics. Write a sensor driver once, use it across ESP32, STM32, or any platform with a HAL implementation.

**Implementation flexibility**: The interface defines behavior, not implementation details. Platform implementations can optimize for RTOS vs bare-metal, performance vs memory, or specific hardware features—whatever fits the target constraints.

**Allocation agnostic**: No forced allocation patterns. Implementations and applications choose static or dynamic allocation based on their needs.

**Extensibility**: Leave room for platform-specific features through implementation config structures. Not every platform supports every feature, and the interface acknowledges that.

These goals guide the design, though the specifics continue to evolve based on what works in practice.

→ [Core design principles & contracts](docs/architecture/core-principles.md)

---

## 🏛️ Architecture

The architecture evolved through iteration, settling on a structure that separates contracts from implementations:

```
Your Application
    ↓
Nexus Drivers (BME280, DS3231, etc.) ← Only depend on interface
    ↓
nexus-hal-interface ← Contract definitions (header-only)
    ↓
nexus-hal-esp32 / nexus-hal-stm32 ← Platform implementations
```

**Design principles**:

**Contract-based dependencies**: Drivers and application code depend on interface contracts, not concrete implementations. This inverts the typical dependency direction—platforms implement interfaces rather than applications depending on platform APIs.

**Implementation freedom**: The interface defines behavior, not mechanisms. Implementations choose their own synchronization primitives (mutexes, critical sections, lock-free), memory strategies, and optimizations based on platform constraints.

**Explicit ownership**: Applications allocate and own all context structures. The HAL initializes memory you provide—no hidden allocations, no lifecycle ambiguity.

**Platform integration**: Hardware-specific code (pin mappings, clock configuration, startup) lives in a platform layer outside the ecosystem. The [showcase project](https://github.com/Fo-Zi/nexus-showcase-project) demonstrates this pattern.

This structure emerged from trying several approaches—monolithic HALs, plugin systems, compile-time polymorphism—and finding what balanced flexibility with practical constraints.

→ [Architecture deep-dive & design decisions](docs/architecture/overview.md)
→ [Interface design patterns](docs/architecture/core-principles.md)

---

## 🛠️ Development Workflow & Tooling

A multi-repository architecture needs consistent tooling across different platforms and build systems. The ecosystem should provide unified commands for building, flashing, and managing dependencies regardless of whether you're working with ESP-IDF, bare-metal CMake, or other build systems.

### Unified Build System

After evaluating options (Git submodules, CMake FetchContent, Bazel), I chose [West](https://docs.zephyrproject.org/latest/west/index.html)—Zephyr's meta-tool—because it handles multi-repo versioning, detects version conflicts automatically, supports custom commands, and provides extensible flash runners.

The custom commands live in [`nexus-hal-interface`](https://github.com/Fo-Zi/nexus-hal-interface)—the common dependency across all ecosystem projects. This centralizes tooling while keeping individual components independent.

**Example workflow**:

```bash
# Build for specific platform
west build -b esp32
west build -b stm32f103

# Flash to device
west flash

# Clean build artifacts
west clean

# List all available projects
west list-projects
```

**What this provides:**
- **Automatic project detection** - Supports ESP-IDF, CMake, and Make projects
- **Extensible flash runners** - OpenOCD, ST-Link, ESP-IDF tools, custom runners
- **Cross-platform consistency** - Same commands across STM32, ESP32, and other targets
- **Dependency version enforcement** - Catches incompatible dependency versions early

The tooling abstracts platform differences while preserving access to platform-specific optimizations when needed.

→ **[Complete West Commands Documentation](WEST_COMMANDS.md)** - Detailed guide with examples and extension patterns

---

## 📚 Current Components

Components built so far, each exploring different aspects of the architecture:

| Repository | Purpose | Status | Notes |
|------------|---------|---------|-------|
| [`nexus-hal-interface`](https://github.com/Fo-Zi/nexus-hal-interface) | 📐 Core HAL API definitions | ✅ Active | Learned API design patterns |
| [`nexus-hal-esp32-idf`](https://github.com/Fo-Zi/nexus-hal-esp32) | ⚡ ESP32 IDF implementation | ✅ Active | Framework wrapper experience |
| [`nexus-llhal-stm32f103`](https://github.com/Fo-Zi/nexus-llhal-stm32f103) | 🔧 STM32 bare-metal HAL | ✅ Active | Register-level programming |
| [`nexus-dht11`](https://github.com/Fo-Zi/nexus-dht11) | 🌡️ DHT11 sensor driver | ✅ Active | Pin & timing interface testing |
| [`nexus-rtc-ds3231`](https://github.com/Fo-Zi/nexus-rtc-ds3231) | ⏰ DS3231 RTC driver | ✅ Active | I2C master interface validation |
| [`nexus-eeprom-24c32`](https://github.com/Fo-Zi/nexus-eeprom-24c32) | 💾 EEPROM driver | ✅ Active | I2C transfer interface validation |
| [`nexus-showcase-project`](https://github.com/Fo-Zi/nexus-showcase-project) | 🎯 Multi-platform example | ✅ Active | Complete integration demo |
| [`nexus-ci`](https://github.com/Fo-Zi/nexus-ci) | 🐳 CI Docker images | ✅ Active | Build automation tooling |

Each component taught something different about the ecosystem—ESP32 implementation about framework integration, STM32 bare-metal about low-level control, drivers about interface usability.

---

## 🚀 Roadmap

This roadmap reflects the development path—starting with foundational patterns and progressively tackling more complex integration challenges:

### Phase 1: Foundation and Core Interface
✅ **Completed**

**Goal**: Establish core interface patterns and build infrastructure

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

### Phase 2: Basic Drivers and HAL Implementation
✅ **Completed**

**Goal**: Validate the interface with real drivers and a framework wrapper HAL implementation. Iterate based on practical usage friction.

#### 2.1 High-Level Framework Wrapper
- ✅ **`nexus-hal-esp32-idf`**
  - Wraps ESP-IDF peripheral drivers
  - Maintains ESP-IDF ecosystem compatibility
  - Complete tooling integration (idf.py build/flash/monitor)

#### 2.3 I2C EEPROM Driver (eeprom-24c32)
- ✅ **`nexus-eeprom-24c32`**
  - Implement the driver, depending exclusively on the `nexus-hal-interface`
  - Implement basic unit-tests for it, to validate the driver and mocks of the interface
  - Analyze how comfortable is the `I2C transfer interface` to use

#### 2.4 DHT11 Driver
- ✅ **`nexus-dht11`**
  - Implement the driver, depending exclusively on the `nexus-hal-interface`
  - Implement basic unit-tests for it, to validate the driver and mocks of the interface
  - Analyze how comfortable are the `pin` and `delay interfaces` to use

### Phase 3: Bare-Metal HAL and Additional Drivers
✅ **Completed**

**Goal**: Implement a register-level HAL without framework dependencies. Explore how the interface patterns work with bare-metal constraints.

#### 3.1 Bare-Metal Implementation
- ✅ **`nexus-llhal-stm32f103`**
  - Register-level implementation (no vendor HAL, no framework)
  - Complete bare-metal tooling:
    - Linker scripts and startup code
    - OpenOCD flash/debug configuration
    - CMake build integration

#### 3.2 DS3231 RTC Driver
- ✅ **`nexus-rtc-ds3231`**
  - Implement the driver, depending exclusively on the `nexus-hal-interface`
  - Implement basic unit-tests for it, to validate the driver and mocks of the interface
  - Analyze how comfortable is the `I2C master interface` to use

#### 3.3 SPI Interface Validation
- ✅ **MCU-to-MCU SPI Communication**
  - Validated SPI master interface through direct MCU communication
  - Tested data transfer protocols between ESP32 and STM32
  - Note: Initial plan for display driver (ST7789) was not completed due to hardware failure

### Phase 4: Integration Project and Documentation
✅ **Completed**

**Goal**: Demonstrate the complete ecosystem with a multi-platform project. Document architecture patterns and design decisions.

#### 4.1 Showcase Project
- ✅ **`nexus-showcase-project`**
  - Multi-platform firmware (ESP32 + STM32) with shared application code
  - Platform abstraction layer demonstrating logical-to-physical resource mapping
  - Unified build commands across different platforms
  - Complete example of the ecosystem in action

#### 4.2 Documentation
- ✅ **`nexus-embedded-ecosystem`**
  - Core architecture and motivation documented
  - Design decisions and tradeoffs analysis
  - Showcase project documentation complete
  - Integration patterns demonstrated

### Phase 5: Complex Real-World Application & v1.0.0 Release
🗓️ **Planned**

**Goal**: Validate the ecosystem with a production-grade application. Demonstrate practical abstraction boundaries and finalize v1.0.0 release.

#### 5.1 Vibration Analyzer Project
- 🗓️ **Complex Real-World Application**
  - DSP-based vibration analysis system
  - Performance-critical data paths with DMA
  - Hybrid approach: Use NHAL for standard peripherals (I2C sensors, GPIO), platform HAL directly for DMA/DSP
  - Demonstrates when to abstract vs. when to use platform-specific optimizations
  - Proves ecosystem viability beyond demonstration projects

#### 5.2 First Stable Release
- 🗓️ **`nexus-hal-interface` v1.0.0**
  - Released after real-world validation in vibration analyzer
  - Signifies:
    - Core interfaces tested across multiple platforms and use cases
    - Production-validated in performance-critical application
    - Breaking changes unlikely for foundational APIs
    - Additional interfaces can be added without breaking existing components


### Future Exploration
🔮 **Potential Directions**

Areas I'd like to explore if time permits — more advanced abstraction patterns, interface extensions, and alternative implementation approaches.

**Display drivers** (ST7789 TFT) - Test SPI interface with more complex peripherals

**Power management interface** - Explore abstracting sleep modes while leaving platform-specific details to implementations

**Modern C++ variant** - Experiment with template metaprogramming, concepts, and compile-time optimizations to compare against the C implementation

---

## ⚠️ Challenges & Tradeoffs

Building this ecosystem surfaced several practical challenges and design tradeoffs:

**Not a complete SDK**: This is an abstraction layer, not a full SDK. Development workflows still depend on the underlying platform—ESP-IDF projects use `idf.py` tooling, bare-metal STM32 needs OpenOCD and linker scripts. West unifies commands where possible, but platform-specific details remain.

**IDE integration**: No plug-and-play IDE extensions. I use `compile_commands.json` with clangd for code completion—manual but functional. A guide for this setup is planned.

**Genericity vs. implementation complexity**: The core tension. Example: DHT11 needs a pin configured as both input and output with pull-up support. If the interface exposes pull-up configuration, the driver works—but only if the implementation supports it. Silent failures happen when a pin doesn't support the required feature. The alternative (exposing full hardware capabilities) makes implementations complex.

Current approach: Keep interfaces minimal and let platform integration layers handle hardware-specific validation. Not perfect, but pragmatic.

**Abstraction overhead**: Every abstraction adds a layer of indirection. For performance-critical code, implementations can bypass the interface and use platform APIs directly. The ecosystem doesn't forbid mixing abstraction levels—use what fits the constraint.

These tradeoffs are inherent to hardware abstraction. The goal is finding a balance that works for most use cases while acknowledging the edge cases.

→ [Detailed tradeoff analysis](docs/challenges.md)

---

## 🔄 Closing Thoughts

This project represents my exploration of embedded systems architecture—how to build firmware that's portable without sacrificing control, modular without adding excessive complexity.

Each component taught me something: ESP-IDF implementation about framework integration patterns, bare-metal STM32 about register-level programming, drivers about API usability, and the showcase project about bringing it all together into a cohesive system.

The ecosystem is functional and I use it for my own projects, but it's still evolving. Interfaces will continue to change as I discover better patterns. That's deliberate—I'm iterating toward something that works well in practice, not rushing to a v1.0.0 before the design is validated.

This work demonstrates my approach to systems design: identify real problems, experiment with solutions, make informed tradeoffs, and iterate based on what works. It's as much about the process as the result.

---

## 📚 Documentation

**Getting Started**:
- [Getting Started Guide](docs/getting-started.md) - 5-minute quick start
- [Showcase Project](https://github.com/Fo-Zi/nexus-showcase-project) - Complete working example

**Understanding the Ecosystem**:
- [Motivation](docs/motivation.md) - Problems and design rationale
- [Architecture Overview](docs/architecture/overview.md) - How the layers work together
- [Core Principles](docs/architecture/core-principles.md) - Design patterns and contracts
- [Challenges & Tradeoffs](docs/challenges.md) - Design decisions and limitations

**Building with the Ecosystem**:
- [Platform Integration Guide](docs/implementations/platform-integration.md) - Create the glue layer
- [West Commands](WEST_COMMANDS.md) - Build system reference

**Browse all docs**: [docs/](docs/)
