## The Pain Points

### Vendor APIs 

Vendor HALs ties your code to specific platforms:

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

Porting means rewriting drivers, learning new APIs, and often subtle bugs from copy-pasting and adapting.

### Testability difficulty

Many times, the HAL interface is desinged and buried under layers of macros and includes, or the functions
are macros themselves, which makes it pretty hard to test and debug. If you really need to unit-test, you will
need to either create a shim abstraction layer on top, or do something like:

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
and then :
```
# Replace HAL functions at link time
gcc ... -Wl,--wrap,HAL_I2C_Master_Receive
```

Then, for a differnt platform, you'll deal with a different
kind of issues, and have to re-implement wrappers or abstraction
layers just to be able to mock or test your upper layers/drivers.


### Driver Portability

Because of the previous reasons, a driver, that's only concerned
about "writes" or "reads", or "switching a pin", needs some degree
of rewriting and porting, since the APIs and wrappers change.

Another scenario is that the driver uses **dependency injection**, so
you implement and set the hardware-dependant functions it depends on, by using function pointers. 
This is great I think for the portability/testability of the driver itself. But in my experience, I have found
that the function signatures of course can vary, then you end up with something like this:

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

See the problem? Now we need to implement 5+ different UART functions, that then "adapt" each 
signature to the underlaying HW UART functions, and then pass by reference to each driver, 
which all basically do the same thing: Read and write bytes from a UART


### Versioning and Dependencies

Another "pattern" I've seen while working with Embedded Projects, is that sometimes the firmware
is not thought or designed as isolated components that interact together to form the application,
but as something that's all glued together. Then, the next project that has some similarities but
also new features and/or runs on a different board, gets picked up by someone else, and that person
copy-pastes the parts of the firmware needed, and adapts them to fit the new requirements.

I see some problems with this:
- The copy-paste can introduce subtle bugs. After all, the new person copying didn't create
the component, so it may miss important details on the process.
- The components are not versioned nor exist as an isolated component, so **IF** some bug is found, the fix will be local to the project where it was found. If the older projects want the fix, it needs to be done by hand.
- There's no "standard" way of fetching dependencies. Some clone a repo and statically use its source files, some use a git submodule, etc.
