# Platform Integration
This document describes the platform integration process for the Nexus Embedded Ecosystem.
Showcasing examples, tips, and best practices.
In order to understand this document, it's recommended to have read:
*architecture/overview*
*architecture/core-principles*

## What is the Platform Integration Layer
Given how the Ecosystem and HAL are structured, the missing step to use the drivers and HAL implementations (or provide your own),
is to define a layer that **defines the actual HW resources used**.
This means:
- Defining which: GPIOs, I2C buses, SPI port, etc; will your board/app use
- The platform specific configuration and contexts:
  - For example, if using esp-idf, you could include a mutex handle on each bus context.
  -
-

## How I envisioned this Layer
I envisioned this layer composed by at least two files:
- Platform Integration Header
- Platform Integration Source Files

### Platform Integration Header
This layer header would:
- Define logical definitions for resources.
- Offer methods to:
  - Initialize all the hardware resources
  - Get references to resource contexts
  - Some other methods that you want to be **public**

Example:
```c
// platform.h header
// Logical resources defs ->
typedef enum{
    GPIO_FUNC_GREEN_LED_0,
    GPIO_FUNC_RTC_INTERRUPT,
    GPIO_FUNC_TOTAL_NUM
}Gpio_Func;

// Global init ->
void platform_init(void);

// Context getters ->
nhal_gpio_ctx * platform_get_gpio_ctx(Gpio_Func func);

// Other board-specific methods that you want to be public ->
void set_secondary_power_source(bool on_off);

```

### Platform Integration Source Files
This layer would:
- "Map" the logical (abstract) resources to the actual Hardware
- Link and/or define implementation specific data
- Allocate the memory for the contexts and configurations

```c
// platform.c file
#include "platform.h"

// Memory allocation of main resources ->
static nhal_gpio_ctx gpio_ctxs[GPIO_FUNC_TOTAL_NUM];
static nhal_gpio_config gpio_configs[GPIO_FUNC_TOTAL_NUM];

// Memory allocation of platform specific resources ->
static nhal_gpio_impl_ctx gpio_impl_ctxs[GPIO_FUNC_TOTAL_NUM];
static nhal_gpio_impl_config gpio_impl_configs[GPIO_FUNC_TOTAL_NUM];

// Link implementation specific data ->
// ****** Intentionally left blank, approaches to do this in the next sections ******

// Global hardware init ->
void platform_init(){
    // GPIOs init ->
    for (int i = 0; i < GPIO_FUNC_TOTAL_NUM; i++){
        nhal_gpio_init(&gpio_ctxs[i]);
        nhal_gpio_set_config(&gpio_ctxs[i],&gpio_configs[i]);
    }

    // Other inits ->

}
```

## Different approaches to declare Implementation Specific Data
As I mentioned before, now what's left to understand how to 'tie' everything together and
have the full Platform Integration Layer is to link the implementation specific data.
In order to do this, we can use different approaches. I propose some:
- Approach 1: Single Source File - No macros
- Approach 2: Single Source File - Helper Macros
- Approach 3: Multiple Source Files

### Single Source File - No macros
As in the 'platform.c' file example, we can just declare all the structures in the same file, and then
assign the nested pointers to the implementation specific data.

We can for example define the 'generic data first':
```c
// platform.c file
static nhal_gpio_config gpio_configs[GPIO_FUNC_TOTAL_NUM] = {
    [GPIO_FUNC_GREEN_LED_0] = {
        .mode = GPIO_MODE_OUTPUT,
        .input_pull = GPIO_PULL_NONE,
    },
    // <- OTHER GPIOs
};
```
Then the implementation specific:
```c
// platform.c file
static nhal_gpio_impl_config gpio_impl_configs[GPIO_FUNC_TOTAL_NUM] = {
    [GPIO_FUNC_GREEN_LED_0] = {
        .output_mode = GPIO_MODE_OUTPUT_OD ,
        .alternate_mode = ,
    },
    // <- OTHER GPIOs
};
```

### Single Source File - Helper Macros

### Multiple Source Files
