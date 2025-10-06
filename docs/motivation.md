# Motivation

The problems that led to building this ecosystem.

## Vendor Lock-In

Vendor HALs tie your code to specific platforms:

```c
// ESP32 project
#include "driver/i2c.h"
void read_sensor(void) {
    uint8_t data[6];
    i2c_master_write_read_device(I2C_NUM_0, 0x76, NULL, 0, data, 6, 1000);
}

// STM32 project
#include "stm32f4xx_hal.h"
void read_sensor(void) {
    uint8_t data[6];
    HAL_I2C_Master_Receive(&hi2c1, 0x76<<1, data, 6, 1000);
}
```

**The problem**: Porting between platforms means rewriting drivers, learning new APIs, and dealing with subtle bugs from adaptation.

## Testability

Vendor HALs aren't designed for unit testing. Functions buried in macros, hardware dependencies everywhere. To test drivers in isolation, you end up writing shim layers:

```c
#ifdef TESTING
#define HAL_I2C_Master_Receive mock_i2c_receive
#endif
```

or:

```c
// Provide your own implementation
HAL_StatusTypeDef __wrap_HAL_I2C_Master_Receive(...) {
    return mock_i2c_receive(...);
}
```

Or link-time wrapping:

```bash
gcc ... -Wl,--wrap,HAL_I2C_Master_Receive
```

**The problem**: Every platform needs different mocking strategies. The same driver requires different test infrastructure on different platforms.

## Driver Portability

Even drivers using dependency injection end up with incompatible signatures:

**Modbus driver** → depends on
```c
uart_write(void* ctx, const uint8_t* buf, size_t len)
```

**SIMxxxx driver** → depends on 
```c
uart_send(void* ctx, const char* str) and uart_read(void* ctx, char* buf, size_t max)
```

**GPS driver** → depends on
```c
uart_readline(void* ctx, char* buf, size_t max, uint32_t timeout)
```

**The problem**: Three UART drivers, three different function signatures. Now you need 5+ adapter functions, all doing the same thing: reading and writing bytes.

## Dependency Management

Embedded firmware often gets structured as a monolith. When starting a new project with similar requirements, developers copy-paste code from previous projects and adapt it.

**Problems with this approach:**
- **Copy-paste bugs**: The person copying didn't write the original code, may miss critical details
- **No versioning**: Bug fixes don't propagate back to older projects
- **Inconsistent dependency management**: Some use git submodules, some vendor code, some clone repositories

**The result**: Bug fixes stay localized, improvements don't spread, and each project diverges further from the others.

## The Solution

These problems pointed to a common need: **portable, testable drivers with consistent interfaces**. That's what this ecosystem attempts to provide.
