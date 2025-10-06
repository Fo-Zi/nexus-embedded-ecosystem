# Architecture Overview

The ecosystem uses contract-based design to separate "what" peripherals do from "how" they do it. This document details the architectural layers, data flow, and design patterns.

## Core Approach

The architecture addresses portability through four strategies:

- **Vendor lock-in** → Platform-agnostic interface definitions (contracts)
- **Testability** → Interfaces isolated from implementations
- **Driver portability** → Drivers depend only on interfaces
- **Dependency management** → Multi-repo with West, explicit versioning

Drivers depend on contracts, not platform APIs:

```c
#include "nhal_i2c_master.h"  // Works on ESP32, STM32, etc.

void read_sensor(struct nhal_i2c_context *i2c) {
    uint8_t data[6];
    nhal_i2c_master_write_read(i2c, 0x76, data, 6);
}
```

## Architecture Layers

### Layer 1: Interface (Contract Definitions)

**Repository**: [`nexus-hal-interface`](https://github.com/Fo-Zi/nexus-hal-interface)
**Purpose**: Define peripheral behavior contracts
**Content**: Header-only C interfaces

```c
// nhal_i2c_master.h
nhal_result_t nhal_i2c_master_write(
    struct nhal_i2c_context *i2c_ctx,
    uint8_t dev_address,
    const uint8_t *data,
    size_t len
);
```

**Design patterns**:
- Context-based (no globals)
- Standardized error codes (`nhal_result_t`)
- Opaque implementation structures
- State machine contracts (init → configure → operate)

### Layer 2: Implementation (Platform HAL)

**Repositories**: [`nexus-hal-esp32-idf`](https://github.com/Fo-Zi/nexus-hal-esp32), [`nexus-llhal-stm32f103`](https://github.com/Fo-Zi/nexus-llhal-stm32f103)
**Purpose**: Fulfill interface contracts for specific platforms
**Content**: Platform-specific implementations

```c
// nhal_i2c.c in nexus-hal-esp32-idf
nhal_result_t nhal_i2c_master_write(
    struct nhal_i2c_context *i2c_ctx,
    uint8_t dev_address,
    const uint8_t *data,
    size_t len
) {
    esp_err_t result = i2c_master_write_to_device(
        i2c_ctx->i2c_bus_id, dev_address, data, len,
        pdMS_TO_TICKS(i2c_ctx->impl_ctx->timeout)
    );
    return nhal_map_esp_err(result);  // Map platform errors to NHAL_RESULT
}
```

**Implementation freedom**:
- Choose synchronization primitives (mutexes, critical sections, lock-free)
- Control memory allocation strategy (static, dynamic, hybrid)
- Optimize for platform strengths (DMA, interrupts, polling)
- Expose platform-specific config through `impl_ctx` structures


### Layer 3: Drivers (Hardware-Agnostic)

**Repositories**: [`nexus-dht11`](https://github.com/Fo-Zi/nexus-dht11), [`nexus-rtc-ds3231`](https://github.com/Fo-Zi/nexus-rtc-ds3231), [`nexus-eeprom-24c32`](https://github.com/Fo-Zi/nexus-eeprom-24c32)
**Purpose**: Control devices using only interface contracts
**Content**: Platform-independent drivers

```c
// ds3231.c - RTC driver works on any platform
#include "nhal_i2c_master.h"  // Only interface dependency

ds3231_result_t ds3231_read_time(ds3231_handle_t *handle, ds3231_time_t *time) {
    uint8_t reg_addr = DS3231_REG_TIME;
    uint8_t data[7];

    nhal_result_t result = nhal_i2c_master_write_read_reg(
        handle->i2c_ctx,
        DS3231_I2C_ADDR,
        &reg_addr, 1,
        data, sizeof(data)
    );

    if (result == NHAL_OK) {
        time->seconds = bcd_to_dec(data[0]);
        time->minutes = bcd_to_dec(data[1]);
        time->hours = bcd_to_dec(data[2]);
        return DS3231_OK;
    }

    return DS3231_ERROR_I2C;
}
```

**Key principle**: Drivers never know what platform they're running on.

### Layer 4: Platform Integration (Application Glue)

**Not part of the ecosystem** - application-specific
**Purpose**: Map logical resources to physical hardware
**Content**: Hardware configuration, context management

This layer translates your application's logical resources (I2C_BUS_SENSORS, PIN_LED) to physical hardware (GPIO2, I2C0 port). The pattern I use:

**Define logical resources:**
```c
// platform_config.h - Application view of hardware
typedef enum {
    PIN_LED_STATUS,
    PIN_DHT11_DATA,
    PIN_TOTAL
} pin_id_t;

typedef enum {
    I2C_BUS_SENSORS,  // RTC, EEPROM, BME280
    I2C_BUS_DISPLAY,  // OLED display
    I2C_TOTAL
} i2c_bus_id_t;

void platform_init(void);
nhal_i2c_context* platform_get_i2c(i2c_bus_id_t bus);
nhal_pin_context* platform_get_pin(pin_id_t pin);
```

**Map to physical hardware (platform-specific):**
```c
// platform_config.c for ESP32
#include "nhal_esp32_helpers.h"

// Maps PIN_DHT11_DATA to GPIO 4 with pull-up
NHAL_ESP32_PIN_BUILD(dht11_data, 4, NHAL_PIN_DIR_BIDIR, 1, 0, 0)

void platform_init(void) {
    nhal_pin_context *ctx = NHAL_ESP32_PIN_CONTEXT_REF(dht11_data);
    nhal_pin_config *cfg = NHAL_ESP32_PIN_CONFIG_REF(dht11_data);

    nhal_pin_init(ctx);
    nhal_pin_set_config(ctx, cfg);
}
```

The same `platform_config.h` works across platforms—only the `.c` file changes when porting.

→ [Complete integration example](https://github.com/Fo-Zi/nexus-showcase-project)

## Data Flow

Temperature reading through the layers:

```
1. Application        → ds3231_read_time(&rtc, &time)
2. Driver            → nhal_i2c_master_write_read_reg(ctx, DS3231_ADDR, ...)
3. HAL (ESP32)       → i2c_master_write_read_device(bus, ...)
4. Platform Driver   → Hardware I2C transaction
5. Result propagates back up through each layer
```

Each layer only knows the layer directly below it. The DS3231 driver doesn't know if it's running on ESP32, STM32, or another platform.

**Note**: ESP32 implementation wraps ESP-IDF, which itself wraps hardware. This is convenient but adds overhead. The STM32 bare-metal implementation goes directly to registers—same interface, different optimization strategy.

## Context-Based Design

No globals. Everything operates on context structures:

```c
struct nhal_i2c_context {
    nhal_i2c_bus_id bus_id;
    struct nhal_i2c_impl_ctx *impl_ctx;  // Implementation-specific
};

// Usage
nhal_i2c_context *i2c = platform_get_i2c(I2C_BUS_SENSORS);
nhal_i2c_master_write(i2c, addr, data, len);
```

**Why contexts**:
- **Multiple instances**: Same interface, different buses
- **Testability**: Inject mock contexts for unit tests
- **Thread safety**: Implementations can embed mutexes in `impl_ctx`
- **Explicit ownership**: Application owns contexts, HAL initializes them

## Implementation Requirements

HAL implementations must fulfill these contracts:

1. **Implement interface functions** with documented behavior
2. **Follow state machine**: uninitialized → initialized → configured → operational
3. **Map error codes**: Platform errors → `nhal_result_t`
4. **Document allocation**: What gets allocated when (static/dynamic)
5. **Document thread safety**: Per-context, global, or none

Drivers can trust these contracts across all platforms.

## What This Provides

**Portability**: Write once, deploy on ESP32, STM32, or any platform with a HAL implementation

**Testability**: Mock implementations for unit testing without hardware

**Modularity**: Drivers and platforms evolve independently

**Flexibility**: Implementations optimize for their constraints (RTOS vs bare-metal, performance vs memory)

## Tradeoffs

**Platform integration overhead**: Mapping logical resources to hardware takes upfront time. Helper macros reduce boilerplate, but you're still setting up more infrastructure than using vendor HAL directly.

**Configuration complexity**: Some peripherals have vastly different config options across platforms. The interface can't abstract everything—platform-specific configuration exists in `impl_ctx` structures. Runtime reconfiguration often requires platform-specific code.

**Initial steeper curve**: Getting to "blinky" takes longer—you need the platform integration layer first. Time investment pays off when porting or testing, less so for single-platform prototypes.

→ [Detailed tradeoff analysis](../challenges.md)
