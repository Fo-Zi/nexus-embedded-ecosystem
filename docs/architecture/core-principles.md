# Core Principles & Contracts

This document outlines the fundamental design principles that guide the Nexus Embedded Ecosystem.

## <� Design Philosophy

### Contract-Based Architecture

**Principle**: Separate the **what** from the **how**.

```c
// Interface defines WHAT a peripheral can do
nhal_result_t nhal_i2c_master_write(nhal_i2c_context_t *ctx, 
                                    uint8_t addr, const uint8_t *data, size_t len);

// Implementation defines HOW it's done (ESP-IDF, STM32 HAL, bare metal, etc.)
```

**Why**: Drivers depend on capabilities, not specific implementations.

### Context-Based Design  

**Principle**: No global state, everything works with contexts.

```c
// Instead of global peripheral instances
extern I2C_HandleTypeDef hi2c1;  // L Global state

// Use context structures  
nhal_i2c_context_t i2c_bus_0;    //  Explicit context
nhal_i2c_context_t i2c_bus_1;    //  Multiple instances
```

**Benefits**:
- Multiple peripheral instances of same type
- Thread safety (implementation can add mutexes to context)
- Easy testing (mock contexts)
- Clear ownership and lifecycle

### State Management Lifecycle

**Principle**: All peripherals follow a predictable state machine.

```
Uninitialized � init() � Initialized � set_config() � Configured � Operations
     �                                                                   �
     ���������������� deinit() ������������������������������������
```

**Implementation Contract**:
- `init()` sets up basic peripheral state
- `set_config()` applies operational parameters
- Operations only work in Configured state
- `deinit()` returns to Uninitialized state

**Why**: Predictable behavior across all platforms and peripherals.

## =' Implementation Contracts

### Memory Management

**Principle**: The interface assumes allocation-free operation.

```c
// Application owns all memory
nhal_i2c_context_t my_i2c_ctx;           //  App allocates
nhal_i2c_config_t my_i2c_cfg;            //  App allocates

// Interface initializes your memory
nhal_i2c_init(&my_i2c_ctx);              //  HAL initializes
nhal_i2c_set_config(&my_i2c_ctx, &cfg);  //  HAL configures
```

**Contract**:
- Applications allocate context and config structures
- HAL implementations initialize/manage the allocated memory
- No hidden dynamic allocation by HAL implementations
- Clear ownership: app owns structs, HAL owns their contents

### Error Handling

**Principle**: Unified error codes across all implementations.

```c
typedef enum {
    NHAL_OK = 0,
    // Argument/validation errors
    NHAL_ERR_INVALID_ARG,
    NHAL_ERR_INVALID_CONFIG,
    // State errors  
    NHAL_ERR_NOT_INITIALIZED,
    NHAL_ERR_NOT_CONFIGURED,
    NHAL_ERR_BUSY,
    // Operation errors
    NHAL_ERR_TIMEOUT,
    NHAL_ERR_HW_FAILURE,
    // ... etc
} nhal_result_t;
```

**Implementation Responsibility**: Map platform-specific errors to NHAL codes.

### Opaque Implementation Data

**Principle**: Interface defines structure, implementation defines contents.

```c
// Interface defines the contract
struct nhal_i2c_context {
    nhal_i2c_bus_id bus_id;                    //  Interface-visible
    struct nhal_i2c_impl_ctx *impl_ctx;        //  Implementation-specific
};

// Implementation defines what it needs
struct nhal_i2c_impl_ctx {
    // ESP32 implementation might have:
    i2c_port_t port;
    TaskHandle_t owner_task;
    SemaphoreHandle_t mutex;
    
    // STM32 implementation might have:  
    I2C_HandleTypeDef *hal_handle;
    uint32_t timeout_ms;
    osMutexId_t mutex;
};
```

**Benefits**:
- Implementation freedom for optimization  
- Interface stability across platforms
- Platform-specific features available when needed

## >� Testing Principles

### Mock-Friendly Design

**Principle**: Every interface can be easily mocked.

```c
// Production code
nhal_i2c_context_t real_i2c = get_platform_i2c();
sensor_read(&real_i2c);

// Test code  
nhal_i2c_context_t mock_i2c = create_mock_i2c();
sensor_read(&mock_i2c);  //  Same function, testable
```

**Why**: Business logic should be testable without hardware.

### Dependency Injection

**Principle**: Drivers receive their dependencies, don't create them.

```c
// Driver doesn't know about platform specifics
typedef struct {
    nhal_i2c_context_t *i2c_ctx;    //  Injected dependency
    uint8_t device_address;
} bme280_handle_t;

// Platform Integration Layer provides contexts
bme280_handle_t sensor = {
    .i2c_ctx = platform_get_i2c_bus(I2C_BUS_SENSORS),  //  Platform-specific
    .device_address = 0x76
};
```

## � Design Trade-offs

### Flexibility vs Simplicity

**Choice**: Favor implementation flexibility over API simplicity.

```c
// The actual complexity - you must allocate implementation-specific structs too
nhal_i2c_context_t ctx;
nhal_i2c_config_t cfg;

// Implementation-specific structures that also need allocation
nhal_i2c_impl_ctx_t impl_ctx;           // ✅ Must allocate
nhal_i2c_impl_config_t impl_cfg;        // ✅ Must allocate

// Link them together
ctx.impl_ctx = &impl_ctx;
cfg.impl_config = &impl_cfg;

// Configure implementation-specific details
impl_cfg.timeout_ms = 1000;
impl_cfg.use_dma = true;
// ... platform-specific configuration

// Then the standard flow
nhal_i2c_init(&ctx);
nhal_i2c_set_config(&ctx, &cfg);
```

**Reality**: This nested pointer allocation can be annoying, which is why implementations provide builder macros:

```c
// ESP32 builder macro hides the complexity
NHAL_ESP32_I2C_BUILD(my_i2c, I2C_NUM_0, 1000, GPIO_NUM_21, GPIO_NUM_22);
nhal_i2c_context_t *ctx = NHAL_ESP32_I2C_CONTEXT_REF(my_i2c);
nhal_i2c_config_t *cfg = NHAL_ESP32_I2C_CONFIG_REF(my_i2c);
```

### Type Safety vs Genericity

**Choice**: Strong typing over generic void pointers.

```c
// Type-safe contexts
nhal_i2c_context_t *i2c_ctx;      //  Type-specific
nhal_spi_context_t *spi_ctx;      //  Type-specific

// Instead of generic handles  
void *peripheral_handle;           // L Type-unsafe
```

### Performance vs Portability

**Choice**: Prioritize portability, allow performance optimization at implementation level.

- Interface defines capability contracts
- Implementations can optimize (DMA, interrupts, bare metal, etc.)
- Applications choose implementations based on performance needs

## =� Implementation Checklist

When creating a new HAL implementation, ensure:

- [ ] All interface functions implemented with correct signatures
- [ ] State machine properly enforced (init � config � operations)  
- [ ] Platform errors mapped to `nhal_result_t` codes
- [ ] Context and config structures properly defined
- [ ] Memory allocation behavior documented
- [ ] Thread safety model documented
- [ ] Timeout semantics implemented consistently
- [ ] Mock implementations available for testing

---

*These principles emerged from practical embedded development experience and continue to evolve as the ecosystem grows.*