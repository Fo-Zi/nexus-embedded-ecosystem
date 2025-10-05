# Nexus Ecosystem West Commands

Custom West commands for building, flashing, and managing multi-platform embedded projects. These commands abstract platform-specific build systems (ESP-IDF, CMake, Make) behind a unified interface.

## Table of Contents

- [Overview](#overview)
- [Available Commands](#available-commands)
- [Platform Configuration (`platform_builds.yaml`)](#platform-configuration-platform_buildsyaml)
- [Supported Build Systems](#supported-build-systems)
- [Supported Flash Runners](#supported-flash-runners)
- [Customizing West Commands](#customizing-west-commands)
- [Common Workflows](#common-workflows)
- [Integration with IDEs](#integration-with-ides)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)
- [Extending the System](#extending-the-system)

## Overview

The ecosystem extends West with commands that handle multi-platform projects. Commands are automatically available when `hal-interface` is in your West manifest—they live there since it's the common dependency across all ecosystem projects.

## Available Commands

### `west build` - Build Projects

Build projects for specific platforms using `platform_builds.yaml` configuration.

**Basic Usage:**
```bash
# Build for specific platform
west build -b esp32
west build -b stm32f103

# Verbose output
west build -b esp32 -v

# Clean before building
west build -b stm32f103 --clean
```

**How it works:**

Projects define supported platforms in `platform_builds.yaml`:

```yaml
platforms:
  esp32:
    platform_build_path: ./platform-builds/esp32
    build_system: esp-idf
    description: "ESP32 with ESP-IDF"

  stm32f103:
    platform_build_path: ./platform-builds/stm32f103
    build_system: cmake
    description: "STM32F103 bare-metal"
```

When you run `west build -b esp32`:
1. West reads `platform_builds.yaml`
2. Finds the `esp32` platform configuration
3. Navigates to `./platform-builds/esp32`
4. Detects build system (ESP-IDF in this case)
5. Runs the appropriate build commands

**Supported Build Systems:**
- **esp-idf**: ESP-IDF projects (uses `idf.py build`)
- **cmake**: CMake projects (uses `cmake` + `make`)
- **make**: Makefile projects (uses `make`)

### `west flash` - Flash Projects

Flash firmware to target hardware. Uses the last built platform automatically.

**Basic Usage:**
```bash
# Flash the last built platform
west flash

# Flash with specific runner (optional)
west flash --runner openocd
west flash --runner stflash
```

**Platform-Specific Options:**
```bash
# ESP32: Specify serial port and baud rate
west flash --port /dev/ttyUSB0
west flash --port /dev/ttyUSB0 --baud 921600

# STM32: Usually just needs the runner
west flash --runner openocd
```

**How it works:**

After building with `west build -b esp32`, a `.last_build_platform` file tracks which platform was built. `west flash` reads this and flashes using the appropriate method:

- **ESP32**: Uses `idf.py flash`
- **STM32 (with OpenOCD)**: Uses OpenOCD runner
- **Other platforms**: Falls back to available runners

**Available Runners:**
```bash
# List runners for current project
west flash --list-runners
```

### `west list-projects` - List Projects

Display all buildable projects in the workspace.

**Usage:**
```bash
# List projects in table format
west list-projects

# Output in JSON format
west list-projects --json
```

### `west clean` - Clean Projects

Remove build artifacts from the last built platform.

**Usage:**
```bash
# Clean the last built platform
west clean
```

**How it works:**

Reads `.last_build_platform` and cleans the corresponding `platform_build_path` directory. For ESP-IDF projects, runs `idf.py fullclean`. For CMake/Make projects, removes the build directory.

## Platform Configuration (`platform_builds.yaml`)

Multi-platform projects use `platform_builds.yaml` to define build configurations for different targets.

**Structure:**

```yaml
platforms:
  <platform-name>:
    platform_build_path: <path-to-build-directory>
    build_system: <esp-idf|cmake|make>
    description: <human-readable-description>
```

**Example:**

```yaml
platforms:
  esp32:
    platform_build_path: ./platform-builds/esp32
    build_system: esp-idf
    description: "ESP32 development kit with ESP-IDF"

  stm32f103:
    platform_build_path: ./platform-builds/stm32f103
    build_system: cmake
    description: "STM32F103 Blue Pill bare metal build"
```

Each platform configuration specifies where its build files live and which build system to use. See the [showcase project](https://github.com/Fo-Zi/nexus-showcase-project) for a complete example.

## Supported Build Systems

The `west build` command automatically detects and uses the appropriate build system:

| Build System | Detection | Build Command |
|--------------|-----------|---------------|
| **esp-idf** | `CMakeLists.txt` + `sdkconfig` | `idf.py build` |
| **cmake** | `CMakeLists.txt` | `cmake --build build` |
| **make** | `Makefile` | `make` |

## Supported Flash Runners

Flash runners are automatically detected based on your project configuration:

| Runner | Auto-Detected When | Manual Override |
|--------|-------------------|-----------------|
| **openocd** | `openocd.cfg` exists + ELF files in build/ | `--runner openocd` |
| **stflash** | `st-flash` installed + ELF files | `--runner stflash` |
| **makeflash** | `Makefile` with flash targets | `--runner makeflash` |
| **espidf** | ESP-IDF project (has `sdkconfig`) | Automatic |

### OpenOCD Runner

Automatically detects projects with OpenOCD configuration.

**Requirements:**
- `openocd.cfg` or `openocd-alt.cfg` in project root
- Built ELF files in `build/` directory

**Features:**
- Automatic binary detection
- Program + verify + reset
- Supports custom OpenOCD configurations

**Example OpenOCD Configurations:**

*For STM32 targets:*
```
source [find interface/stlink.cfg]
source [find target/stm32f1x.cfg]
adapter speed 1000
```

*For ESP32 targets:*
```
source [find interface/ftdi/esp32_devkitj_v1.cfg]
source [find target/esp32.cfg]
```

*For generic ARM Cortex-M:*
```
source [find interface/cmsis-dap.cfg]
source [find target/at91samdXX.cfg]
```

### ST-Flash Runner

Uses ST-Link tools for flashing (primarily STM32 targets).

**Requirements:**
- `st-flash` command available
- Built ELF files (automatically converted to binary)

**Features:**
- Automatic ELF to BIN conversion
- Configurable flash address (default: 0x8000000)

### Make Flash Runner

Delegates to your existing Makefile flash targets.

**Requirements:**
- `Makefile` with flash targets

**Supported Targets:**
```makefile
# Default flash target
flash: firmware

# Specific targets (detected automatically)
flash-firmware: build
    flashtool --write build/firmware.bin --address 0x8000000

flash-bootloader: build
    flashtool --write build/bootloader.bin --address 0x8000000
```

**Usage:**
```bash
west flash --runner makeflash                    # Uses 'flash' target
west flash --runner makeflash --target firmware    # Uses 'flash-firmware' target
```

## Customizing West Commands

### Adding Custom Flash Runners

Want to add support for new flash tools? The West commands live in `hal-interface/scripts/nexus_commands/`.

**Basic steps:**

1. Create a runner class in `runners.py` that inherits from `FlashRunner`
2. Implement `can_flash()`, `flash()`, and `get_available_binaries()`
3. Add your runner to the `FLASH_RUNNERS` list
4. Test with `west flash --list-runners`

See the existing runners (`OpenOCDRunner`, `STFlashRunner`, etc.) as examples.

### Modifying Build Behavior

Custom build logic can be added by modifying `build.py`. Each build system has its own build method (`_build_esp_idf_project()`, `_build_cmake_project()`, etc.).

### Learning More About West

These commands are West extensions. For deep customization, see:

- [Official West Documentation](https://docs.zephyrproject.org/latest/develop/west/index.html)
- [West Extension Commands](https://docs.zephyrproject.org/latest/develop/west/extensions.html)
- [Nexus hal-interface repository](https://github.com/Fo-Zi/nexus-hal-interface) - Source code for these commands

## Common Workflows

### Multi-Platform Project Workflow

**Typical development cycle:**

```bash
# Build for ESP32
west build -b esp32

# Flash to ESP32 board
west flash --port /dev/ttyUSB0

# Switch to STM32
west build -b stm32f103

# Flash to STM32 (uses OpenOCD automatically)
west flash

# Clean STM32 build
west clean

# Back to ESP32
west build -b esp32
west flash
```

### Using OpenOCD for STM32

**Setup:** Add `openocd.cfg` to your platform build directory:

```
# openocd.cfg
source [find interface/stlink.cfg]
source [find target/stm32f1x.cfg]
adapter speed 1000
```

**Build and flash:**

```bash
west build -b stm32f103
west flash --runner openocd
```

### ESP32 Serial Port Selection

**Find your port:**

```bash
# Linux
ls /dev/ttyUSB*
ls /dev/ttyACM*

# macOS
ls /dev/cu.usbserial*

# Windows
# Check Device Manager or use
mode
```

**Flash with specific port:**

```bash
west build -b esp32
west flash --port /dev/ttyUSB0
west flash --port /dev/ttyUSB0 --baud 921600
```

## Integration with IDEs

### VS Code Integration

Add to your `.vscode/tasks.json`:

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "West Build",
            "type": "shell",
            "command": "west",
            "args": ["build"],
            "group": "build",
            "presentation": {
                "echo": true,
                "reveal": "always"
            }
        },
        {
            "label": "West Flash",
            "type": "shell",
            "command": "west",
            "args": ["flash"],
            "group": "build",
            "dependsOn": "West Build"
        }
    ]
}
```

### Shell Aliases

Add to your `.bashrc` or `.zshrc`:

```bash
# Short aliases for common commands
alias wb='west build'
alias wf='west flash'
alias wc='west clean'
alias wl='west list-projects'

# Build and flash in one command
alias wbf='west build && west flash'
```

## Troubleshooting

### Commands Not Found

West commands missing? Check:

1. `hal-interface` is in your `west.yml` with `west-commands: scripts/west-commands.yml`
2. Run `west update` to sync
3. You're in a West workspace (has `.west/` directory)

### Flash Issues

| Problem | Fix |
|---------|-----|
| "No flash runners available" | Run `west build -b <platform>` first |
| "OpenOCD config not found" | Add `openocd.cfg` to platform build directory |
| "st-flash not found" | Install: `sudo apt install stlink-tools` |
| "Permission denied" (Linux) | Add to dialout: `sudo usermod -a -G dialout $USER` then logout/login |
| ESP32 port not found | Check `ls /dev/ttyUSB*` or try different USB port |

### Debug Output

```bash
# Verbose mode
west -v build -b esp32
west -v flash

# Check available runners
west flash --list-runners

# List all projects
west list-projects
```

## Best Practices

**Platform build configs**: Keep `openocd.cfg` and other flash configs in platform build directories (`platform-builds/*/`)

**Version control**: Commit `platform_builds.yaml`, `.gitignore` build artifacts

**Port permissions** (Linux): Add yourself to `dialout` group once: `sudo usermod -a -G dialout $USER`

**Shell aliases**: Speed up development:
```bash
alias wb='west build'
alias wf='west flash'
alias wc='west clean'
```

## Extending the System

Want to add custom runners or modify build behavior? The West commands live in `hal-interface/scripts/nexus_commands/`.

See the "Adding Custom Flash Runners" section above for examples. Pull requests welcome at the [hal-interface repository](https://github.com/Fo-Zi/nexus-hal-interface).
