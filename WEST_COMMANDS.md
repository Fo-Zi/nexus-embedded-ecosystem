# Nexus Ecosystem West Commands

This document describes how to use the West extension commands provided by the Nexus Ecosystem and how to extend them with your own custom runners.

## Overview

The Nexus Ecosystem provides West extension commands that work across all projects in your workspace. These commands are automatically available when you import the `hal-interface` project in your West manifest.

## Available Commands

### `west build` - Build Projects

Build one or more projects in your workspace with automatic project type detection.

**Usage:**
```bash
# Build current directory project
west build

# Build specific project
west build my-sensor-app
west build my-project

# Build all projects in workspace
west build --all

# Clean before building
west build --clean

# Verbose output
west build -v
```

**Supported Project Types:**
- **ESP-IDF**: Projects with `CMakeLists.txt` + `sdkconfig`
- **CMake**: Projects with `CMakeLists.txt`
- **Make**: Projects with `Makefile`

### `west flash` - Flash Projects

Flash built projects to target hardware using automatic runner detection.

**Basic Usage:**
```bash
# Flash current directory project
west flash

# Flash specific project
west flash my-sensor-app
west flash my-project

# Flash specific binary/target
west flash --target blinky
west flash my-project --target firmware

# Flash with specific runner
west flash --runner openocd
west flash --runner stflash
west flash --runner makeflash
```

**Flash Options:**
```bash
# ESP-IDF specific options
west flash --port /dev/ttyUSB0 --baud 921600

# List all flashable projects
west flash --list

# List available runners for current project
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

Remove build artifacts and intermediate files.

**Usage:**
```bash
# Clean current directory project
west clean

# Clean specific project
west clean my-project

# Clean all projects
west clean --all
```

## Flash Runners

The flash system uses "runners" - specialized handlers for different flash tools and targets.

### Available Runners

| Runner | Description | Requirements |
|--------|-------------|--------------|
| `openocd` | OpenOCD for embedded targets | `openocd.cfg` file + built ELF files |
| `stflash` | ST-Link tools | `st-flash` command + built ELF files |
| `makeflash` | Makefile targets | `Makefile` with flash targets |
| `espidf` | ESP-IDF projects | `idf.py` command + `sdkconfig` file |

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

## Adding Custom Flash Runners

You can extend the flash system by adding your own runners for new tools or targets.

### 1. Create a Runner Class

Edit `hal-interface/scripts/nexus_commands/runners.py` and add your runner:

```python
class JLinkRunner(FlashRunner):
    """J-Link flash runner for ARM targets"""
    
    def can_flash(self):
        """Check if J-Link is available and project is compatible"""
        try:
            # Check if J-Link tools are installed
            subprocess.run(['JLinkExe', '-?'], 
                          capture_output=True, check=True)
            
            # Check for J-Link config or device support
            return (self.build_dir.exists() and 
                   (self.project_path / "jlink.cfg").exists())
        except:
            return False
    
    def get_available_binaries(self):
        """Get list of flashable binaries"""
        if not self.build_dir.exists():
            return []
        
        binaries = []
        for item in self.build_dir.iterdir():
            if item.is_file() and item.suffix == '.hex':
                binaries.append(item.name)
        return binaries
    
    def flash(self, binary=None, device="ATSAM3X8E", **kwargs):
        """Flash using J-Link"""
        if binary is None:
            available = self.get_available_binaries() 
            if not available:
                raise RuntimeError("No binaries found to flash")
            binary = available[0]
        
        binary_path = self.build_dir / binary
        if not binary_path.exists():
            raise RuntimeError(f"Binary {binary} not found")
        
        # Create J-Link script
        script_content = f"""
device {device}
si 1
speed 4000
loadfile {binary_path} 0x8000000
r
g
qc
"""
        
        script_path = self.project_path / "flash_script.jlink"
        with open(script_path, 'w') as f:
            f.write(script_content)
        
        print(f"Flashing {binary} using J-Link...")
        
        cmd = ['JLinkExe', '-commandfile', str(script_path)]
        result = subprocess.run(cmd, cwd=self.project_path)
        
        # Clean up
        script_path.unlink()
        
        return result.returncode == 0
```

### 2. Register Your Runner

Add your runner to the registry:

```python
# At the bottom of runners.py
FLASH_RUNNERS = [
    JLinkRunner,      # Add your new runner
    OpenOCDRunner,
    STFlashRunner,
    MakeFlashRunner,
    ESPIDFRunner,
]
```

### 3. Test Your Runner

```bash
# List available runners (should show your new runner)
west flash --list-runners

# Use your custom runner
west flash --runner jlink
west flash --runner jlink --target myapp.hex
```

## Project Configuration Examples

### STM32 Project with OpenOCD

**File Structure:**
```
my-stm32-project/
├── CMakeLists.txt
├── openocd.cfg          # OpenOCD configuration
├── src/
│   └── main.c
└── build/               # Built binaries (after west build)
    ├── firmware.elf
    └── bootloader.elf
```

**openocd.cfg:**
```
# Debug adapter
source [find interface/stlink.cfg]

# Target configuration  
source [find target/stm32f1x.cfg]

# Adapter speed
adapter speed 1000

# Reset configuration
reset_config srst_only srst_pulls_trst
```

**Usage:**
```bash
west build my-stm32-project
west flash my-stm32-project --runner openocd
west flash my-stm32-project --target bootloader --runner openocd
```

### ESP32 Project

**File Structure:**
```
my-esp32-project/
├── CMakeLists.txt
├── sdkconfig            # ESP-IDF configuration
├── main/
│   ├── CMakeLists.txt
│   └── main.c
└── build/               # Built binaries (after west build)
```

**Usage:**
```bash
west build my-esp32-project
west flash my-esp32-project --port /dev/ttyUSB0
west flash my-esp32-project --port COM3 --baud 921600
```

### Mixed Project with Makefile

**File Structure:**
```
my-custom-project/
├── Makefile
├── src/
├── build/
└── scripts/
    └── flash.sh         # Custom flash script
```

**Makefile with flash targets:**
```makefile
.PHONY: flash flash-app flash-bootloader

# Default flash target
flash: flash-app

# Application flash
flash-app: build/app.elf
	./scripts/flash.sh $< 0x8000000

# Bootloader flash  
flash-bootloader: build/bootloader.elf
	./scripts/flash.sh $< 0x8000000

# Custom tool flash
flash-custom: build/app.hex
	my-custom-tool --flash $< --verify
```

**Usage:**
```bash
west build my-custom-project  
west flash my-custom-project --runner makeflash
west flash my-custom-project --runner makeflash --target bootloader
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

### Command Not Found

If West doesn't recognize the commands:

1. **Check manifest**: Ensure `hal-interface` has `west-commands: scripts/west-commands.yml`
2. **Update projects**: Run `west update` to sync repositories  
3. **Check workspace**: Make sure you're in a West workspace (has `.west/` directory)

### Flash Failures

Common flash issues and solutions:

| Issue | Solution |
|-------|----------|
| "No flash runners available" | Build project first with `west build` |
| "OpenOCD config not found" | Add `openocd.cfg` to project root |
| "st-flash not found" | Install ST-Link tools: `sudo apt install stlink-tools` |
| "Binary not found" | Check build directory exists and contains binaries |
| "Permission denied" | Add user to dialout group: `sudo usermod -a -G dialout $USER` |

### Adding Debug Output

For troubleshooting runners, add debug output:

```bash
# Verbose West output
west -v flash --runner openocd

# List what runners can see
west flash --list-runners

# See all projects West can detect
west list-projects
```

## Best Practices

### Project Organization

- **Keep flash configs in project root**: `openocd.cfg`, `jlink.cfg`
- **Use consistent build directories**: Always use `build/` 
- **Document flash requirements**: Add README with hardware setup
- **Version control configs**: Include flash configs in git

### Runner Development

- **Test thoroughly**: Ensure `can_flash()` is accurate
- **Handle errors gracefully**: Provide clear error messages
- **Support options**: Accept common parameters like port, baud
- **Clean up**: Remove temporary files after flashing
- **Document usage**: Add help text and examples

### Workspace Setup

- **One workspace per ecosystem**: Don't mix different hardware families
- **Use consistent naming**: Project names should be descriptive
- **Share configurations**: Put common configs in shared repositories
- **Document dependencies**: List required tools in README

## Contributing

To contribute improvements to the West commands:

1. **Fork the hal-interface repository**
2. **Make your changes** in `scripts/nexus_commands/`
3. **Test thoroughly** with different project types
4. **Update documentation** in this file
5. **Submit a pull request**

For questions or support, please open an issue in the hal-interface repository.