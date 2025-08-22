# Nexus Embedded Ecosystem

*Modular HAL interfaces, implementations, and drivers for embedded development*

---

## 🎯 What is it

I started this project to ease some of the pain points that I have faced when developing software for Embedded Devices, both professionally and for side-hobby projects.

While I will start it as a project to facilitate my development, I'd be pleased if someone wants to: 
- Collaborate
- Offer criticism
- Use it for themselves
- Simply take some ideas


---

## 💭 Motivation and Goals

### 🔍 Motivation

While working with Embedded Systems I have faced some pain-points:

- **🧪 Vendor HALs and others are not usually designed with testability in mind**
  - I have found myself writing shim layers of abstraction for peripherals in order to be able to unit-test some complex driver or component
    (You can of course do it, but I found myself repeating this common step many times)

- **🔌 Drivers were not designed around a common interface**
  - Sometimes the driver wouldn't even be in a separate repo, but copy-pasted and passed among projects. If the driver is simple enough this may seem harmless. But we are humans, we can make small mistakes in the process that could create very subtle bugs that will not be caught until it's already too late
  - If the driver uses something like dependency injection, for example for the hardware functionality it needs, you'd often find yourself having to duplicate a ton of code. Imagine having ten I2C devices, whose drivers all use dependency injection with different function signatures..

- **🔒 The HAL your code depends on, is tied to a specific Framework/MCU/SDK**
  - This goes hand-by-hand with the first point
  - The porting efforts become bigger → Your code and your drivers need some degree of refactoring
  - Usually I have found also that HALs for RTOSes are radically different from HALs for Bare Metal. While this has of course good reasons to be like that, I often found that most drivers depend on the same functionality, what changes could be if you need to take a mutex before accessing a shared bus as a master or not (for example), or some other implementation specific detail.

- **🏗️ Some HALs are very complex and big**
  - For some projects these HALs are an overkill, even if more generic than others

All of these points and probably others that I am missing, pushed me to create this project and experiment with it.

### 🎯 Goals

- **📋 Provide an Isolated HAL-interface**
  - Header-only and no dependencies other than std C
    - You can then use the interface alone, include a HAL-implementation if wanted, or provide your own.
    - Use drivers that depend on the interface, even if you choose to use your own implementation.
  - Exists in its own repo, and has its own versioning
    - Drivers can explicitly depend on a fixed interface version → The interface may then evolve without breaking dependencies

- **⚙️ Provide isolated HAL-implementations**
  - HAL-implementations are then sets of source files that implement the HAL-interface

- **🔧 Create drivers that are designed following a contract-like relationship with the HAL-interface**

- **📦 Use West to manage ecosystem dependencies**
  - West suits well for this architecture, because in a single manifest you can declare:
    - The hal-interface
    - The hal-implementation you will need (Or your own)
    - The drivers you will depend on
  - Using fixed version references for all of them, if long-term stability is a concern (for example). Or using the latest version available, if you always want to have available the latest features of each.
  - All repos will have a manifest declaring their dependencies
  - Very easy to setup a CI process through reproducible containers

---

## 🏛️ Architecture

The ecosystem has three layers:

1. **🎨 Interface** (`nexus-hal-interface`) - Defines what peripherals should do, not how
2. **⚡ Implementation** (`nexus-hal-esp32-idf`, `nexus-hal-cmsis`, `nexus-hal-stm32`) - Platform-specific HAL code  
3. **🔩 Drivers** (`nexus-bme280`, `nexus-eeprom-24c32`) - Device drivers that only depend on the interface

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
- There could be 4th or as many more layers as you need or want, since once you are creating components **on-top** of drivers, you can modularize them straightforwardly.

### 🌐 Platform Layer

Finally, there's a layer I designed my projects around that I don't know where to assign in the diagram, which is the "Platform Layer". The hal is designed taking the idea that this layer will exist in your application, although it may not.

This layer has the following characteristics in mind:
- **🎛️ Generic interface** - Defines user-friendly IDs to access resources
- **🔧 Context management** - Provides methods to get contexts for peripherals as well as getting/setting configuration (Public one)
- **🔗 Implementation bridge** - The source files of this layer link the generic side of the HAL with the implementation specific one 
- **🚀 Initialization** - Provide methods to initialize all the hardware needed from the platform (`platform_init(..)` → `platform_i2c_periph_init(..)`, `platform_spi_periph_init(..)`, etc.)

Although it's hard to make this layer generic, because of the reasons I just mentioned, I will create a template for it, and a couple
of examples to showcase what I had in mind. You are of course free to use it, or ignore it and do it your way.

---

## 🚧 Current Status

This is early-stage experimentation. I'm working on:

- **📐 Core Interface** - Basic I2C, SPI, GPIO, UART abstractions
- **⚡ Advanced Interfaces** - Async, buffered, DMA abstractions
- **🔧 ESP32 Implementation** - My first target platform
- **🎯 CMSIS Implementation** - To provide support for many vendors/MCUs (ARM-centric)
- **🔩 Example Drivers** - Drivers to validate the approach
- **📋 Platform Layer Template** - While it contains impl. specific code, I will provide templates to showcase how I pictured it
- **🔨 Complementary Tooling** - For the CMSIS layer for example, I'm designing a way to define your specific MCU and automatically including the right headers + macros + configuration + sets of linker/openocd/etc scripts

The goal is to prove the concept works before expanding to more platforms or adding too many drivers.

---

## 📦 Dependencies with West

I'm using [west](https://docs.zephyrproject.org/latest/develop/west/index.html) for dependency management. Each component lives in its own repository with proper versioning.

### 🛠️ Basic Setup

**Create a workspace:**
```bash
mkdir nexus-workspace
cd $your_path_to-nexus-workspace
```

**Create your app repo:**
```bash
mkdir my-app
git init ./my-app
```

**Add a `./my-app/west.yml` manifest:**
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

**Initialize and fetch:**
```bash
cd $your_path_to-nexus-workspace
west init -l my-app
west update
```

---

## ⚠️ Challenges

So far I have identified the following challenges, that I will be experimenting on different ways to address them:

- **🛠️ This is not "an SDK"** - While the architecture offers some interesting advantages, it's far from "an SDK". This means that the development workflow depends for now heavily on what hal-implementation you are using. For example:
  - If using `nhal-esp32-idf`, you will have access to idf tooling, allowing you to flash and debug using Python-based scripting
  - If using `nhal-cmsis`, you will have to delve more into low level details to flash and debug your device, likely reusing some linker script and openocd config file.

- **🖥️ IDE integration is manual** - If you want to use a full IDE to develop Software through this ecosystem, you will not have an extension to install everything nor make your IDE recognize all dependencies. I personally just instruct CMake to export the build commands in a JSON format to a location I know of, and then create a `.clangd` file indicating where it should find it, this works at least with VS code, Zed editor, and most Vim/Nvim setups. (You would still need to personalize your IDE interaction if you want to build/flash/debug using only GUI)

- **⚖️ Design trade-offs** - There's a clear tension and trade-off relationship between some design choices:
  - **Portability** vs **Easiness of use**
  - **Easiness of adding support for new MCUs/HAL impl./etc** vs **Tooling complexity and abstraction**
  - **Standardizing build/flash/debug ops among different hal implementations** vs **Freedom of customization**

---

## 📚 Current Components

| Repository | Purpose | Status |
|------------|---------|---------|
| [`nexus-hal-interface`](https://github.com/Fo-Zi/nexus-hal-interface) | 📐 Core HAL API definitions | 🚧 Active |
| [`nexus-hal-esp32`](https://github.com/Fo-Zi/nexus-hal-esp32) | ⚡ ESP32 implementation | 🚧 Active |
| [`nexus-eeprom-24c32`](https://github.com/Fo-Zi/nexus-eeprom-24c32) | 🔩 EEPROM driver example | 🚧 Active |

More platforms and drivers will come as the architecture stabilizes.

---

## 🔄 Development Philosophy

This is a work in progress. The interfaces will change, and I'll probably discover better approaches as I build more drivers and support more platforms. But that's the point - to experiment and learn what actually works in practice.

I will iterate until I find something that feels comfortable to work with, until that point, versioning will move fast. And once I get a stable enough API, I will release the first stable v1.0.0
