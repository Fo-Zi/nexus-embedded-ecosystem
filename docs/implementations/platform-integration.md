# Platform Integration Guide

The Platform Integration Layer connects your application to HAL implementations. It maps logical resources (like `PIN_LED_STATUS`) to physical hardware (like `GPIO 2` on ESP32 or `PC13` on STM32), manages context allocation, and handles platform-specific initialization.

**Prerequisites**: Read [architecture/overview.md](../architecture/overview.md) and [architecture/core-principles.md](../architecture/core-principles.md) first.

---

## What Is Platform Integration?

The Platform Integration Layer sits between your application and the HAL implementation:

```
Application Code
    ↓ (uses logical resources: PIN_LED, I2C_BUS_SENSORS)
Platform Integration Layer
    ↓ (maps to physical hardware: GPIO2, I2C port 0)
HAL Implementation (ESP32 / STM32)
    ↓
Hardware
```

**Responsibilities**:
- Define logical resource identifiers (application view of hardware)
- Map logical resources to physical pins/buses/ports
- Allocate memory for contexts and configurations
- Initialize hardware during platform startup
- Provide getter functions for accessing contexts

---

## Layer Structure

### Platform Header (platform.h)

**Platform-agnostic** - Same header for all platforms:

```c
// platform.h
typedef enum {
    PIN_LED_STATUS,
    PIN_DHT11_DATA,
    PIN_TOTAL_NUM
} pin_func_t;

typedef enum {
    I2C_BUS_SENSORS,    // RTC, EEPROM, etc.
    I2C_TOTAL_NUM
} i2c_bus_func_t;

nhal_result_t platform_init(void);
struct nhal_pin_context* platform_get_pin_ctx(pin_func_t pin);
struct nhal_i2c_context* platform_get_i2c_ctx(i2c_bus_func_t bus);
```

**Key principle**: Logical names describe purpose, not hardware (`PIN_LED_STATUS` not `PIN_GPIO2`)

### Platform Implementation (platform.c)

**Platform-specific** - Different file for each platform, same interface:

- `esp32_platform.c` - Maps to ESP32 GPIOs
- `stm32f103_platform.c` - Maps to STM32 ports/pins

This file changes when porting, but provides the same functions defined in `platform.h`.

---

## Implementation Approaches

### Approach 1: Builder Macros (ESP32)

Use HAL-provided macros to reduce boilerplate:

```c
// esp32_platform.c
#include "nhal_esp32_builders.h"

// Declare resources with macros
NHAL_ESP32_PIN_BUILD(led, GPIO_NUM_2, NHAL_PIN_DIR_OUTPUT, ...)
NHAL_ESP32_I2C_BUILD(sensors, I2C_NUM_0, GPIO_NUM_22, GPIO_NUM_21, 400000, 1000)

// Map to logical IDs
static struct nhal_pin_context* pin_contexts[PIN_TOTAL_NUM];

static void setup_mappings(void) {
    pin_contexts[PIN_LED_STATUS] = NHAL_ESP32_PIN_CONTEXT_REF(led);
}
```

**Advantages**: Minimal boilerplate, easy to add resources
**See**: [esp32_platform.c](https://github.com/Fo-Zi/nexus-showcase-project/blob/main/platform/platform_impl/esp32/esp32_platform.c)

### Approach 2: Manual Arrays (STM32 Bare-Metal)

Explicit allocation and linking:

```c
// stm32f103_platform.c
static struct nhal_pin_context pin_contexts[PIN_TOTAL_NUM];
static struct nhal_pin_config pin_configs[PIN_TOTAL_NUM];
static struct nhal_pin_impl_config pin_impl_configs[PIN_TOTAL_NUM];

static void setup_mappings(void) {
    pin_configs[PIN_LED_STATUS] = (struct nhal_pin_config){
        .direction = NHAL_PIN_DIR_OUTPUT,
        .pull_mode = NHAL_PIN_PMODE_NONE,
        .impl_config = &pin_impl_configs[PIN_LED_STATUS]
    };

    pin_contexts[PIN_LED_STATUS].pin_id = &pin_ids[PIN_LED_STATUS];
}
```

**Advantages**: Complete control, no hidden allocations
**Disadvantages**: More verbose, manual pointer linking
**See**: [stm32f103_platform.c](https://github.com/Fo-Zi/nexus-showcase-project/blob/main/platform/platform_impl/stm32f103/stm32f103_platform.c)

### Approach 3: Hybrid

Mix macros and manual configuration:

```c
// Pins use macros
NHAL_ESP32_PIN_BUILD(led, GPIO_NUM_2, ...)

// I2C manual for fine control
static struct nhal_i2c_context i2c_sensors;
static struct nhal_i2c_config i2c_sensors_cfg;
// ... manual setup
```

---

## Application Usage

Once platform layer is set up, applications use logical resources:

```c
// main.c
#include "platform.h"
#include "nexus_dht11.h"

int main(void) {
    platform_init();

    // Get contexts from platform
    struct nhal_pin_context *dht11_pin = platform_get_pin_ctx(PIN_DHT11_DATA);

    // Initialize driver with platform context
    dht11_handle_t dht11;
    dht11_init(&dht11, dht11_pin);

    // Use driver
    dht11_reading_t reading;
    dht11_read(&dht11, &reading);

    return 0;
}
```

**Key points**:
- Application never includes platform-specific headers
- Same code works on ESP32 and STM32
- Only `platform.c` changes when porting

---

## Common Patterns

### Platform-Specific Initialization

Some platforms need special setup (clocks, power):

```c
nhal_result_t platform_init(void) {
    #ifdef STM32F103
        // STM32 needs clock configuration
        stm32f103_clock_config_t cfg;
        stm32f103_get_default_72mhz_config(&cfg);
        stm32f103_clock_init(&cfg, NULL);
    #endif

    setup_mappings();
    initialize_peripherals();
    return NHAL_OK;
}
```

### Error Handling During Init

Initialize critical peripherals first (like debug UART):

```c
nhal_result_t platform_init(void) {
    nhal_result_t result;

    result = setup_debug_uart();
    if (result != NHAL_OK) {
        return result;  // Can't continue without logging
    }

    // Now can log errors from other peripherals
    for (int i = 0; i < I2C_TOTAL_NUM; i++) {
        result = nhal_i2c_init(&i2c_contexts[i]);
        if (result != NHAL_OK) {
            platform_log("Failed I2C init: %d", i);
            return result;
        }
    }

    return NHAL_OK;
}
```

### Runtime Validation

Validate inputs in getter functions:

```c
struct nhal_pin_context* platform_get_pin_ctx(pin_func_t pin) {
    if (pin >= PIN_TOTAL_NUM) {
        return NULL;  // Invalid pin ID
    }
    return &pin_contexts[pin];
}
```

---

## Best Practices

**Keep platform.h platform-agnostic** - No ESP32/STM32 specifics in header

**Use descriptive logical names** - `PIN_SENSOR_POWER` not `PIN_GPIO5`

**Document hardware mapping** - Comment which physical pins map to logical IDs

**Initialize in dependency order** - Clock → UART (logging) → Other peripherals

**Static allocation by default** - Avoid dynamic allocation unless necessary

**Group related resources** - Separate enums for sensors, LEDs, buttons

---

## Troubleshooting

**Context pointers are NULL**:
- Verify `platform_init()` called before using getters
- Check mapping tables are set up correctly
- Ensure array indices match enum values

**Build fails with "undefined reference"**:
- Platform implementation file not added to build system
- Wrong HAL implementation linked (ESP32 code with STM32 HAL)

**Hardware doesn't respond**:
- Verify physical pin mapping matches code
- Check platform-specific initialization (clocks, power)
- Use debugger to inspect context contents

**Drivers fail with NHAL_ERR_NOT_CONFIGURED**:
- Ensure both `init()` and `set_config()` called in `platform_init()`
- Check `impl_config` pointers are set correctly

---

## Complete Working Example

The [showcase project](https://github.com/Fo-Zi/nexus-showcase-project) demonstrates complete platform integration for ESP32 and STM32.

**Project structure**:
```
showcase-project/
├── platform/
│   ├── platform.h                    # Platform-agnostic header
│   └── platform_impl/
│       ├── esp32/
│       │   └── esp32_platform.c      # ESP32 implementation (builder macros)
│       └── stm32f103/
│           └── stm32f103_platform.c  # STM32 implementation (manual arrays)
├── src/
│   └── main.c                        # Application uses platform.h only
└── platform-builds/
    ├── esp32/                        # ESP32 build configuration
    └── stm32f103/                    # STM32 build configuration
```

**Study these files**:
- **[platform.h](https://github.com/Fo-Zi/nexus-showcase-project/blob/main/platform/platform.h)** - Logical resource definitions
- **[esp32_platform.c](https://github.com/Fo-Zi/nexus-showcase-project/blob/main/platform/platform_impl/esp32/esp32_platform.c)** - ESP32 with builder macros
- **[stm32f103_platform.c](https://github.com/Fo-Zi/nexus-showcase-project/blob/main/platform/platform_impl/stm32f103/stm32f103_platform.c)** - STM32 with manual arrays
- **[main.c](https://github.com/Fo-Zi/nexus-showcase-project/blob/main/src/main.c)** - Application using platform layer

These files show the patterns in practice with complete, working code.

---

This layer is where platform-specific details live. Get it right once, and your application code stays portable across platforms.
