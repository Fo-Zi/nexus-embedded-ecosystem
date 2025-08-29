# Architecture Overview

The Nexus Embedded Ecosystem is built around a simple idea: separate **what peripherals do** from **how they do it**. This document explains the architecture in detail.

## The Problem We're Solving

Traditional embedded development ties your code to specific platforms:

```c
// ESP32 project
#include "driver/i2c.h"
void read_sensor(void) {
    uint8_t data[6];
    i2c_master_write_read_device(I2C_NUM_0, 0x76, NULL, 0, data, 6, 1000);
}

// STM32 project - completely different API
#include "stm32f4xx_hal.h"
void read_sensor(void) {
    uint8_t data[6];
    HAL_I2C_Master_Receive(&hi2c1, 0x76<<1, data, 6, 1000);
}
```

Porting means rewriting drivers, learning new APIs, and often subtle bugs from copy-paste engineering.

## The Solution: Contract-Based Architecture

Instead of depending on platform-specific APIs, we define **contracts** that any platform can implement:

```c
// Same code works on ESP32, STM32, Nordic, etc.
#include "nhal_i2c_master.h"
void read_sensor(struct nhal_i2c_context *i2c) {
    uint8_t data[6];
    nhal_i2c_master_read(i2c, 0x76, data, 6, 1000);
}
```

## Architecture Layers

### Layer 1: Interface (Contract Definitions)

**Repository**: `nexus-hal-interface`
**Purpose**: Define what peripherals can do, not how they do it
**Content**: Header-only C interfaces, no implementation code

```c
// nhal_i2c_master.h - Defines the contract
nhal_result_t nhal_i2c_master_write(
    struct nhal_i2c_context *i2c_ctx,
    uint8_t dev_address,
    const uint8_t *data,
    size_t len,
    nhal_timeout_ms timeout
);
```

**Key characteristics**:
- Platform-agnostic function signatures
- Standardized error codes (`nhal_result_t`)
- Context-based design (no global state)
- Opaque implementation structures

### Layer 2: Implementation (Platform-Specific Code)

**Repositories**: `nexus-hal-esp32`, `nexus-hal-stm32`, etc.
**Purpose**: Fulfill the interface contracts for specific platforms
**Content**: C source files that implement the interface functions

```c
// nhal_i2c.c in nexus-hal-esp32
nhal_result_t nhal_i2c_master_write(
    struct nhal_i2c_context *i2c_ctx,
    uint8_t dev_address,
    const uint8_t *data,
    size_t len,
    nhal_timeout_ms timeout
) {
    // ESP-IDF specific implementation
    esp_err_t result = i2c_master_write_to_device(
        i2c_ctx->i2c_bus_id,
        dev_address,
        data, len,
        pdMS_TO_TICKS(timeout)
    );
    return nhal_map_esp_err(result);
}
```

**Implementation freedom**:
- Thread safety approach (mutexes, interrupt disable, etc.)
- Memory allocation patterns (static, dynamic, pools)
- Performance optimizations (DMA, CPU copy, caching)
- Error handling detail level
- Resource management (power, clocks, pins)

### Layer 3: Drivers (Device-Specific Logic)

**Repositories**: `nexus-bme280`, `nexus-ds3231`, etc.
**Purpose**: Control specific devices using only interface contracts
**Content**: Device drivers that work on any compliant implementation

```c
// bme280.c - Works on any platform
#include "nhal_i2c_master.h"  // ← Only interface dependency

bme280_result_t bme280_read_temperature(bme280_handle_t *handle, float *temp) {
    uint8_t reg_addr = BME280_REG_TEMP_MSB;
    uint8_t data[3];

    nhal_result_t result = nhal_i2c_master_write_read_reg(
        handle->i2c_ctx,
        BME280_I2C_ADDR,
        &reg_addr, 1,
        data, sizeof(data),
        1000
    );

    if (result == NHAL_OK) {
        // Process raw temperature data
        int32_t raw_temp = (data[0] << 12) | (data[1] << 4) | (data[2] >> 4);
        *temp = compensate_temperature(handle, raw_temp);
        return BME280_OK;
    }

    return BME280_ERR_I2C_ERROR;
}
```

### Layer 4: Platform Integration (Your Application Glue)

**Not part of the ecosystem** - every project is different
**Purpose**: Connect your specific hardware to the HAL implementation
**Content**: Hardware configuration, context management, initialization

```c
// platform_config.c - Your project-specific code
#include "nhal_esp32_helpers.h"

// Define I2C bus for sensors with ESP32-specific configuration
NHAL_ESP32_I2C_DEFINE(sensor_bus, I2C_NUM_0, GPIO_21, GPIO_22);

void platform_init(void) {
    // Get configured context
    struct nhal_i2c_context *sensor_i2c = NHAL_ESP32_I2C_GET_CTX(sensor_bus);

    // Initialize drivers with the context
    bme280_init(&temperature_sensor, sensor_i2c, 1000);
    ds3231_init(&rtc, sensor_i2c, 1000);
}
```

This layer handles:
- Pin assignments and hardware routing
- Clock configuration and power management
- Context creation and lifecycle management
- Platform-specific initialization sequences

## Data Flow Example

Here's how a temperature reading flows through the architecture:

```
1. Application calls: bme280_read_temperature(&sensor, &temp)
                                    ↓
2. BME280 driver calls: nhal_i2c_master_write_read_reg(ctx, ...)
                                    ↓
3. ESP32 implementation: i2c_master_write_read_device(bus, ...)
                                    ↓
4. ESP-IDF driver: Hardware I2C transaction
                                    ↓
5. Result bubbles back up through each layer
```

Each layer only knows about the layer directly below it. The BME280 driver has no idea it's running on ESP32 vs STM32.

## Context-Based Design

Instead of global peripheral instances (`I2C1`, `SPI0`), everything works with contexts:

```c
struct nhal_i2c_context {
    nhal_i2c_bus_id i2c_bus_id;
    nhal_i2c_operation_mode_t current_mode;
    struct nhal_i2c_impl_ctx *impl_ctx;  // ← Implementation-specific state
    // Conditional features based on compile-time flags
};
```

**Benefits**:
- **Multiple instances**: Multiple I2C buses, same interface
- **Testability**: Easy to inject mock contexts
- **Thread safety**: Implementation can add mutexes to context
- **State encapsulation**: No global variables, clear ownership

## Memory Model

**You control application memory**:
- All contexts are allocated by your application (stack, heap, static, pools)
- All configuration structures are yours
- All data buffers are yours

**Implementation controls internal resources**:
- May allocate mutexes, queues, DMA descriptors as documented
- Internal state goes in opaque `impl_ctx` structures
- Resource allocation strategy varies by implementation

```c
// Example: You decide allocation strategy
struct nhal_i2c_context i2c_ctx;        // Stack allocation
static struct nhal_i2c_context i2c_ctx; // Static allocation
struct nhal_i2c_context *i2c_ctx = pool_alloc(); // Pool allocation

// HAL initializes your memory, doesn't allocate new memory
nhal_i2c_master_init(&i2c_ctx);
```

## Implementation Contracts

Every implementation must:

1. **Implement all interface functions** with documented behavior
2. **Follow the state machine** (uninitialized → initialized → configured → operational)
3. **Return appropriate error codes** mapped from platform errors
4. **Handle context validation** (NULL checks, state validation)
5. **Document allocation behavior** (what gets allocated when)
6. **Document threading model** (thread-safe, not thread-safe, per-context safety)

## Benefits of This Architecture

**For Application Developers**:
- Write once, run on any supported platform
- Easy unit testing with mock implementations
- Clear separation of concerns
- No vendor lock-in

**For Driver Writers**:
- Single codebase for all platforms
- Consistent interface across peripherals
- Testable without hardware

**For Platform Implementers**:
- Freedom to optimize for platform strengths
- Can focus on platform integration, not driver logic
- Clear contracts to implement

## Trade-offs

**Advantages**:
- True code portability
- Excellent testability
- Clean separation of concerns
- Implementation flexibility

**Disadvantages**:
- Additional abstraction layer
- May not expose all platform-specific features
- Integration complexity for platform layer
- Learning curve for the architecture

The trade-off is essentially: slightly more complexity upfront for significantly easier long-term development and maintenance.
