# Nexus Embedded Ecosystem

*A modular hardware abstraction layer and driver ecosystem for embedded development.*

---

## What This Is

This is my exploration into building a better foundation for embedded projects. I got tired of the same recurring problems:

- **Vendor HALs aren't portable.** You write code for STM32's HAL, you're locked into STM32. Want to try ESP32? You have to rewrite a lot.
- **Drivers get copied everywhere.** I've seen that many projects use some driver (local), then others use that project as a reference, the driver gets copied and forwarded
  to many others. If you find a bug? -> Manual fix everywhere
- **Testing embedded code is painful.** Most HALs aren't designed to be mocked or tested. You end up with hardware-dependent code that's very hard to unit test.
- **Platform migration is expensive.** Business decides to switch from ESP32 to STM32? That's weeks of porting work.

So I'm building something different. The core idea is simple: **separate the interface from the implementation**.

---

## Architecture

The ecosystem has three layers:

1. **Interface** (`nexus-hal-interface`) - Defines what peripherals should do, not how
2. **Implementation** (`nexus-hal-esp32`, `nexus-hal-stm32`) - Platform-specific HAL code  
3. **Drivers** (`nexus-bme280`, `nexus-eeprom-24c32`) - Device drivers that only depend on the interface

```
Application
    ↓
Device Drivers (BME280, EEPROM, etc.)
    ↓  
HAL Interface (I2C, SPI, GPIO definitions)
    ↓
HAL Implementation (ESP32, STM32, nRF implementations)
```
- A BME280 driver written against the interface will work on any platform that implements that interface. No porting needed.
- There could be 4th or as many more layers as you need or want, since once you are creating components **on-top** of drivers,
  you can modularize them straightforwardly.

Finally, there's a layer I designed my projects around that I don't know where to assign in the diagram, which is the "Platform Layer".
The hal is designed taking the idea that this layer will exist in your application, although it may not. At this layer I:
- Hold all the configuration of all peripherals
- Provide methods to initialize all the hardware needed from the platform ( platform_init(..) -> platform_i2c_periph_init(..), platform_spi_periph_init(..), etc )
- Then the drivers only focus on the peripheral's functionality and not its configuration (unless strictly needed)
- The platform source code ties the hal with the hal implementation
I will create an example of this later on.

---

## Current Status

This is early-stage experimentation. I'm working on:

- **Core Interface** - Basic I2C, SPI, GPIO, UART abstractions
- **Advanced Interfaces** - Async, buffered, DMA abstractions
- **ESP32 Implementation** - My first target platform
- **Example Drivers** - Drivers to validate the approach and approach

The goal is to prove the concept works before expanding to more platforms.

---

## Dependencies with West

I'm using [west](https://docs.zephyrproject.org/latest/develop/west/index.html) for dependency management. Each component lives in its own repository with proper versioning.

### Basic Setup

Create a workspace:
```bash
mkdir nexus-workspace
cd $your_path_to-nexus-workspace
```

Create your app repo:
```bash
mkdir my-app
git init .
```

Add a `west.yml` manifest:
```yaml
manifest:
  remotes:
    - name: fo-zi
      url-base: https://github.com/Fo-Zi

  projects:
    - name: nexus-hal-interface
      remote: fo-zi
      revision: main
    - name: nexus-hal-esp32
      remote: fo-zi
      revision: main
    - name: nexus-eeprom-24c32
      remote: fo-zi
      revision: main
```

Initialize and fetch:
```bash
cd $your_path_to-nexus-workspace
west init -l my-app
cd my-app
west update
```

---

## What I'm Learning

Building this is teaching me about:

- **SOLID Principles** - I try to apply this principles throughout the whole ecosystem
- **API design** - What makes a good abstraction?
- **Dependency management** - How to version and distribute embedded components
- **Testing strategies** - Making embedded code testable without hardware
- **Build systems** - Integrating multiple repositories cleanly

The interfaces will definitely evolve as I discover what works and what doesn't.
What are the pain points and what seems like the best approach for them.

---

## Current Components

| Repository | Purpose |
|------------|---------|
| [`nexus-hal-interface`](https://github.com/Fo-Zi/nexus-hal-interface) | Core HAL API definitions |
| [`nexus-hal-esp32`](https://github.com/Fo-Zi/nexus-hal-esp32) | ESP32 implementation |
| [`nexus-eeprom-24c32`](https://github.com/Fo-Zi/nexus-eeprom-24c32) | EEPROM driver example |

More platforms and drivers will come as the architecture stabilizes.

---

## Example Usage

```c
#include "nhal_i2c.h"
#include "eeprom_24c32.h"
#include "platform.h"

int main() {

    // Get I2C Context from platform layer ->
    struct nhal_i2c_context * i2c_ctx = platform_get_i2c_ctx(I2C_EEPROM_0);
    
    // Initialize EEPROM driver
    eeprom_24c32_handle_t eeprom;
    eeprom_24c32_init(&eeprom, i2c_ctx);
    
    // Write/read data
    uint8_t data[] = "Hello";
    eeprom_24c32_write(&eeprom, 0x00, data, sizeof(data));
    
    uint8_t buffer[16];
    eeprom_24c32_read(&eeprom, 0x00, buffer, sizeof(data));
    
    return 0;
}
```

This same code will work on ESP32, STM32, or any other platform with a nexus HAL implementation.

---

This is a work in progress. The interfaces will change, and I'll probably discover better approaches as I build more drivers and support more platforms. But that's the point - to experiment and learn what actually works in practice.
I will iterate until I find something that feels comfortable to work with, until that point, versioning will move fast. And once I get a stable enough API, I will release the first stable v1.0.0 
