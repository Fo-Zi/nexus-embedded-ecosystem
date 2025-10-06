# Getting Started

This guide gets you from zero to a working multi-platform firmware build in 5 minutes.

**Goal**: Build and run the [showcase project](https://github.com/Fo-Zi/nexus-showcase-project) on ESP32 or STM32F103 to see the ecosystem in action.

---

## Prerequisites

### Required Tools

**1. Python 3.8+ and West**
```bash
pip install west
```

**2. Platform-specific toolchains**

Choose based on your target hardware:

<details>
<summary><b>For ESP32</b></summary>

- **ESP-IDF v5.0+** - [Installation guide](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/)
- Verify installation:
  ```bash
  idf.py --version
  ```

</details>

<details>
<summary><b>For STM32F103</b></summary>

- **ARM GCC toolchain**:
  ```bash
  sudo apt install gcc-arm-none-eabi
  ```
- **OpenOCD** (for flashing):
  ```bash
  sudo apt install openocd
  ```
- Verify installation:
  ```bash
  arm-none-eabi-gcc --version
  openocd --version
  ```

</details>

### Hardware

<details>
<summary><b>ESP32 Setup</b></summary>

- ESP32 development board (DevKit, NodeMCU, etc.)
- DHT11 temperature/humidity sensor
- Wiring:
  - DHT11 data pin → GPIO32
  - DHT11 VCC → 3.3V
  - DHT11 GND → GND
- USB cable for programming

</details>

<details>
<summary><b>STM32F103 Setup</b></summary>

- STM32F103C8T6 "Blue Pill" board
- DHT11 temperature/humidity sensor
- Wiring:
  - DHT11 data pin → PA3
  - DHT11 VCC → 3.3V
  - DHT11 GND → GND
- ST-Link V2 programmer
- USB-to-serial adapter for UART output (PA9/PA10)

</details>

---

## Quick Start

### 1. Initialize Workspace

```bash
# Clone the showcase project with West
west init -m https://github.com/Fo-Zi/nexus-showcase-project nexus-workspace
cd nexus-workspace/nexus-showcase-project

# Fetch all dependencies (HAL implementations, drivers)
west update
```

**What this does**: West clones the project and pulls all dependencies defined in `west.yml` (HAL interface, platform implementations, drivers).

### 2. Build for Your Platform

**For ESP32**:
```bash
west build -b esp32
```

**For STM32F103**:
```bash
west build -b stm32f103
```

West automatically:
- Detects the build system (ESP-IDF vs CMake)
- Configures platform-specific settings
- Builds the firmware

### 3. Flash to Hardware

**ESP32**:
```bash
west flash --port /dev/ttyUSB0
```

**STM32F103**:
```bash
west flash  # Uses OpenOCD automatically
```

### 4. Monitor Output

**ESP32**:
```bash
idf.py monitor
```

**STM32F103**:
```bash
screen /dev/ttyUSB0 115200
# Or: minicom -D /dev/ttyUSB0 -b 115200
```

**Expected output**:
```
ESP32 platform initializing...
ESP32 platform initialized successfully
[5234 ms] Temperature: 23.0°C | Humidity: 45.0%
[10234 ms] Temperature: 23.2°C | Humidity: 44.8%
...
```

---

## What Just Happened?

You built and ran **the same application code** on two different platforms:

1. **Application code** (`src/main.c`) - Platform-agnostic, uses only HAL interfaces
2. **Platform layer** (`platform/`) - Maps logical resources to physical hardware
3. **HAL implementations** - ESP32-IDF or STM32 bare-metal
4. **West build system** - Manages dependencies and builds

The application doesn't know if it's running on ESP32 or STM32—it only knows about `PIN_DHT11_DATA` and NHAL interfaces.

---

## Next Steps

### Explore the Showcase Project

**Understand the architecture**:
- Read [`src/main.c`](https://github.com/Fo-Zi/nexus-showcase-project/blob/main/src/main.c) - Platform-agnostic application
- Check [`platform/platform.h`](https://github.com/Fo-Zi/nexus-showcase-project/blob/main/platform/platform.h) - Logical resource definitions
- Compare [`esp32_platform.c`](https://github.com/Fo-Zi/nexus-showcase-project/blob/main/platform/platform_impl/esp32/esp32_platform.c) vs [`stm32f103_platform.c`](https://github.com/Fo-Zi/nexus-showcase-project/blob/main/platform/platform_impl/stm32f103/stm32f103_platform.c)

**Build for the other platform**:
```bash
# Built for ESP32? Try STM32
west build -b stm32f103
west flash

# Built for STM32? Try ESP32
west build -b esp32
west flash
```

Same code, different hardware, no changes.

### Learn the Architecture

1. **[Architecture Overview](architecture/overview.md)** - How the layers work together
2. **[Core Principles](architecture/core-principles.md)** - Design patterns and contracts
3. **[Platform Integration Guide](implementations/platform-integration.md)** - How to create the platform layer

### Build Your Own Project

**Start from the showcase template**:

1. Copy the showcase project structure
2. Modify `platform.h` for your hardware resources
3. Update platform implementations for your board
4. Write your application using HAL interfaces

**Or start from scratch**:

1. Create project with `west.yml` manifest
2. Add HAL dependencies (interface + implementations)
3. Add drivers you need
4. Create platform integration layer
5. Write application code

→ [Platform Integration Guide](implementations/platform-integration.md) walks through this process

---

## Troubleshooting

**West commands not found**:
```bash
# Verify West installation
west --version

# Reinstall if needed
pip install --upgrade west
```

**Build fails - ESP32**:
- Ensure ESP-IDF is activated: `. ~/esp/esp-idf/export.sh`
- Check ESP-IDF version: `idf.py --version` (need v5.0+)

**Build fails - STM32**:
- Verify ARM toolchain: `arm-none-eabi-gcc --version`
- Check CMake version: `cmake --version` (need 3.15+)

**Flash fails - Permission denied (Linux)**:
```bash
# Add yourself to dialout group
sudo usermod -a -G dialout $USER
# Then logout/login
```

**Flash fails - OpenOCD can't find device**:
- Check ST-Link connection
- Try running OpenOCD manually: `openocd -f openocd.cfg`
- Verify OpenOCD config in `platform-builds/stm32f103/openocd.cfg`

**Wrong serial port**:
```bash
# List available ports
ls /dev/tty*    # Linux
ls /dev/cu.*    # macOS

# ESP32 usually: /dev/ttyUSB0 or /dev/ttyACM0
# STM32 UART usually: /dev/ttyUSB0 (via USB-to-serial adapter)
```

**No sensor readings**:
- Verify DHT11 wiring (data pin, VCC, GND)
- Check pull-up resistor on DHT11 data line (some modules have it built-in)
- Confirm correct GPIO mapping in platform implementation

---

## Common Commands Reference

```bash
# Build commands
west build -b esp32           # Build for ESP32
west build -b stm32f103       # Build for STM32F103
west build -b esp32 --clean   # Clean build

# Flash commands
west flash                    # Flash last built platform
west flash --port /dev/ttyUSB0  # ESP32 with specific port

# Utility commands
west clean                    # Clean build artifacts
west list-projects            # List all buildable projects
west update                   # Update dependencies
```

For complete command reference, see [WEST_COMMANDS.md](../WEST_COMMANDS.md).

---

## Learn More

**Documentation**:
- [Motivation](motivation.md) - Why this ecosystem exists
- [Architecture](architecture/overview.md) - How it works
- [Platform Integration](implementations/platform-integration.md) - Build the glue layer
- [Challenges & Tradeoffs](challenges.md) - Design decisions

**Example Projects**:
- [Showcase Project](https://github.com/Fo-Zi/nexus-showcase-project) - Multi-platform DHT11 example

**Ecosystem Components**:
- [nexus-hal-interface](https://github.com/Fo-Zi/nexus-hal-interface) - Interface definitions
- [nexus-hal-esp32-idf](https://github.com/Fo-Zi/nexus-hal-esp32) - ESP32 implementation
- [nexus-llhal-stm32f103](https://github.com/Fo-Zi/nexus-llhal-stm32f103) - STM32 bare-metal
- [nexus-dht11](https://github.com/Fo-Zi/nexus-dht11) - DHT11 driver

---

**You're ready to explore multi-platform embedded development. Build once, deploy everywhere.**
