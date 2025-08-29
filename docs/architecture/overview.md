# Architecture Overview

## Intro
The Nexus Embedded Ecosystem is built as an attempt to solve the [pain-points] already mentioned. 
The approach I have taken to solve each one of them is as follows:

- Vendor APIs issue -> Create a set of interfaces that are platform-agnostic
- Testability Difficulty -> Same as previous; Make the interfaces live isolated from any implementation ;
- Driver Portability -> Make drivers depend **only** on the interfaces ;
- Versioning and Dependencies -> Each component has its own versioning and lives in its own repo ; Dependencies are pulled using West

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
    size_t len
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
        pdMS_TO_TICKS(i2c_ctx->impl_ctx->timeout)
    );
    return nhal_map_esp_err(result);
}
```

**Implementation freedom**:
- As long as you fulfill the contracts, the implementations have total freedom.
- The implementation details are opaque to drivers, but not to the Platform Integration Layer
    - This means that **the implementation** chooses what to expose and what to omit.
    - You could have total and fine grain control over configuration details in this layer if needed.
- The interface implementation and Platform Integration Layer decide the memory allocation scheme used.
 

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
        handle->dev_addr,
        &reg_addr, 1,
        data, sizeof(data)
    );

    if (result == NHAL_OK) {
        // Process raw temperature data
        int32_t raw_temp = (data[0] << 12) | (data[1] << 4) | (data[2] >> 4);
        *temp = compensate_temperature(handle, raw_temp);
        return BME280_OK;
    }

    return result;
}
```

### Layer 4: Platform Integration (Your Application Glue)

**Not part of the ecosystem** - every project is different
**Purpose**: Connect your specific hardware to the HAL implementation
**Content**: Hardware configuration, context management, initialization

While there's no universal way of implementing it, I usually have a header that defines
**logical-user-friendly** resources definitions and helpers:
```c
// platform_config.h - Ideally doesn't expose any platform specifics, declares what HW resources the project uses 

// Pins -> 
typedef enum {
    PIN_LED_GREEN,
    PIN_RTC_INTERRUPT,
    // +++++
    PIN_TOTAL_NUM,
}Pin_Func;

// I2C buses -> 
typedef enum {
    I2C_BUS_0, // RTC xxx, Data EEPROM xxx, Sensor X
    I2C_BUS_1, // Config EEPROM xxx
    // +++++
    I2C_TOTAL_NUM,
}I2C_BusFunc;

// ******

// This method will iterate over each peripheral set and initialize/configure it ->
void platform_init(void); 

struct * nhal_i2c_ctx platform_get_i2c_ctx(I2C_BusFunc i2c_bus); // <- How upper layers get the contexts drivers need
struct * nhal_pin_ctx platform_get_pin_ctx(Pin_Func pin_func); // <- How upper layers get the contexts drivers need

```

Then, the associated source **IS** necessarily platform-specific, since it will **translate** the resources
you will use to a specific board layout/platform:
```c
// platform_config.c - Links implementation with specific project resources (pins, buses, ports, etc) 
#include "nhal_esp32_helpers.h"

// Define I2C bus for sensors with ESP32-specific configuration
NHAL_ESP32_I2C_DEFINE(sensor_bus, I2C_NUM_0, GPIO_21, GPIO_22); // <- Macro helper provided by the implementation

void platform_init(void) {


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
> [!NOTE]
> This is a rather bloated example where the NHAL is merely a wrapper over another HAL!
> Since the interface is not tied to any specific implementation, the latter can be optimized as much
> as needed and wanted, even using raw assembly or register manipulation implementation.

## Context-Based Design

Instead of global peripheral instances (`I2C1`, `SPI0`), everything works with contexts:

```c
struct nhal_i2c_context {
    nhal_i2c_bus_id i2c_bus_id;
    struct nhal_i2c_impl_ctx *impl_ctx;  // ← Implementation-specific data
};
```

**Benefits**:
- **Multiple instances**: Multiple I2C buses, same interface
- **Testability**: Easy to inject mock contexts
- **Thread safety**: Implementation can add mutexes or other to context
- **State encapsulation**: No global variables, clear ownership

## Implementation Contracts

Every implementation must:

1. **Implement all interface functions** with documented behavior
2. **Follow the state machine** (uninitialized → initialized → configured → operational)
3. **Return appropriate error codes** mapped from platform errors
5. **Document allocation behavior** (what gets allocated when)
6. **Document threading model** (thread-safe, not thread-safe, per-context safety)

## Benefits of This Architecture

**For Application Developers**:
- Write drivers and application code once, run on any supported platform
- Easy unit testing with mock implementations
- Clear separation of concerns
- No vendor lock-in

**For Driver Writing**:
- Same interface (contract) accross different platforms
- Testable without hardware

**For Platform Implementers**:
- Freedom to optimize for platform strengths
- Can focus on platform integration, not driver logic
- Clear contracts to implement

## Trade-offs

**Advantages**:
- Code portability
- Good testability
- Separation of concerns
- Implementation flexibility

**Disadvantages**:
- Since for some peripherals the configuration varies a lot through different platforms:
    - A big deal of configuration for these **IS** implementation specific
    - You can configure it fully in the Platform Integration Layer still, but if you want to 
    for example allow for reconfiguration at runtime, then you'll be obligued to use platform-specific
    code for it. (This would be applicable to any framework, but still worht mentioning)
- Integration complexity for Platform Integration Layer
    - Since there are nested pointers to implementation specific data on both context and config structures,
    initializing both can be annoying (The interface takes **pointers** to these, so you must allocate it)
    - Because of this issue, implementations could provide "builders" as Macros or other, that help
    ease the initialization. I do this with all the implementations of the ecosystem, documentation can be found 
    at []
- More effort needed to get to a first "blinky" project
    - Since you need to implement the Platform Integration Layer before anything, it needs more time upfront.
    - This lost time should be quickly compensated by the time saved on porting and rewriting drivers and
    application code.


