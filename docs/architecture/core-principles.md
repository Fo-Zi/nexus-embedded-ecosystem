# Core Principles

Design patterns and contracts that make the ecosystem work. These principles guide interface design and implementation requirements.

## Contract-Based Design

Interfaces define **what** peripherals do, implementations define **how**:

```c
// Interface contract - platform-agnostic
nhal_result_t nhal_i2c_master_write(
    nhal_i2c_context_t *ctx,
    uint8_t addr,
    const uint8_t *data,
    size_t len
);

// Implementations fulfill contract differently:
// - ESP32: Uses ESP-IDF i2c_master_write_to_device()
// - STM32: Direct register manipulation
// - Mock: Returns test data
```

Drivers depend on contracts, never on implementations.

## Context-Based API

No globals. Peripherals operate on context structures:

```c
// Global approach (typical vendor HAL)
extern I2C_HandleTypeDef hi2c1;
HAL_I2C_Master_Transmit(&hi2c1, ...);

// Context approach (NHAL)
nhal_i2c_context_t *i2c = platform_get_i2c(I2C_BUS_SENSORS);
nhal_i2c_master_write(i2c, ...);
```

**Why contexts**:
- **Multiple instances**: `i2c_sensors` and `i2c_display` use same interface
- **Thread safety**: Implementation embeds mutex in context
- **Testability**: Inject mock contexts
- **Explicit ownership**: Clear lifecycle management

## State Machine Contract

All peripherals follow a predictable state machine:

```
Uninitialized → init() → Initialized → set_config() → Configured → Operations
     ↑                                                                   |
     └────────────────────── deinit() ─────────────────────────────────┘
```

**Requirements**:
- `init()` sets up basic peripheral state
- `set_config()` applies operational parameters
- Operations only work in Configured state
- `deinit()` returns to Uninitialized

This contract ensures predictable behavior across all platforms.

## Memory Management

The application owns all memory—no hidden allocations:

```c
// Application allocates
nhal_i2c_context_t i2c_ctx;
nhal_i2c_config_t i2c_cfg;

// HAL initializes your memory
nhal_i2c_init(&i2c_ctx);
nhal_i2c_set_config(&i2c_ctx, &cfg);
```

**Contract**:
- Applications allocate context and config structures
- HAL implementations initialize/manage the allocated memory
- No hidden dynamic allocation by HAL
- Clear ownership: app owns structs, HAL owns their contents

## Error Handling

Unified error codes across all implementations:

```c
typedef enum {
    NHAL_OK = 0,
    NHAL_ERR_INVALID_ARG,
    NHAL_ERR_INVALID_CONFIG,
    NHAL_ERR_NOT_INITIALIZED,
    NHAL_ERR_NOT_CONFIGURED,
    NHAL_ERR_BUSY,
    NHAL_ERR_TIMEOUT,
    NHAL_ERR_HW_FAILURE,
    // ...
} nhal_result_t;
```

Implementations map platform-specific errors to these codes. Drivers trust the mapping.

## Opaque Implementation Data

Interface defines structure, implementation defines contents:

```c
// Interface defines contract
struct nhal_i2c_context {
    nhal_i2c_bus_id bus_id;              // Visible to interface
    struct nhal_i2c_impl_ctx *impl_ctx;  // Opaque, implementation-specific
};

// ESP32 implementation defines what it needs
struct nhal_i2c_impl_ctx {
    i2c_port_t port;
    SemaphoreHandle_t mutex;
    uint32_t timeout_ms;
};

// STM32 bare-metal implementation might have
struct nhal_i2c_impl_ctx {
    I2C_TypeDef *registers;
    uint32_t timeout_ticks;
    volatile uint8_t busy_flag;
};
```

**Benefits**:
- Implementation freedom for optimization
- Interface stability across platforms
- Platform-specific features available when needed

## Testing Design

### Mock-Friendly Interfaces

Every interface can be mocked:

```c
// Production code
nhal_i2c_context_t *i2c = platform_get_i2c(I2C_BUS_SENSORS);
ds3231_read_time(i2c, &time);

// Test code
nhal_i2c_context_t mock_i2c = create_mock_i2c();
ds3231_read_time(&mock_i2c, &time);  // Same function, testable
```

Business logic tests without hardware.

### Dependency Injection

Drivers receive their dependencies, don't create them:

```c
// Driver handle
typedef struct {
    nhal_i2c_context_t *i2c_ctx;  // Injected dependency
    uint8_t device_address;
} ds3231_handle_t;

// Platform integration provides contexts
ds3231_handle_t rtc = {
    .i2c_ctx = platform_get_i2c(I2C_BUS_SENSORS),
    .device_address = DS3231_I2C_ADDR
};
```

## Design Tradeoffs

### Implementation Complexity vs Flexibility

The nested pointer pattern requires more setup:

```c
// What you actually need to allocate
nhal_i2c_context_t ctx;
nhal_i2c_config_t cfg;
nhal_i2c_impl_ctx_t impl_ctx;      // Implementation-specific
nhal_i2c_impl_config_t impl_cfg;   // Implementation-specific

// Link them
ctx.impl_ctx = &impl_ctx;
cfg.impl_config = &impl_cfg;

// Configure platform-specific details
impl_cfg.timeout_ms = 1000;
impl_cfg.clock_speed = 400000;

// Then standard flow
nhal_i2c_init(&ctx);
nhal_i2c_set_config(&ctx, &cfg);
```

**Reality**: This is verbose. Implementations provide builder macros to hide complexity:

```c
// ESP32 builder macro
NHAL_ESP32_I2C_BUILD(sensors_i2c, I2C_NUM_0, 1000, GPIO_21, GPIO_22);

nhal_i2c_context_t *ctx = NHAL_ESP32_I2C_CONTEXT_REF(sensors_i2c);
nhal_i2c_config_t *cfg = NHAL_ESP32_I2C_CONFIG_REF(sensors_i2c);

nhal_i2c_init(ctx);
nhal_i2c_set_config(ctx, cfg);
```

The tradeoff: more initial setup for implementation flexibility and testability.

### Type Safety vs Genericity

Strong typing over generic void pointers:

```c
// Type-safe contexts (NHAL approach)
nhal_i2c_context_t *i2c_ctx;
nhal_spi_context_t *spi_ctx;

// Generic handles (avoided)
void *peripheral_handle;  // Type-unsafe
```

Catch errors at compile time, not runtime.

### Performance vs Portability

Prioritize portability, allow performance optimization at implementation level:

- Interface defines capability contracts
- Implementations optimize (DMA, interrupts, bare metal)
- Applications choose implementations based on needs

For critical paths, use platform HAL directly and NHAL for everything else.

## Implementation Checklist

When creating a HAL implementation, ensure:

- [ ] All interface functions implemented with correct signatures
- [ ] State machine enforced (init → config → operations)
- [ ] Platform errors mapped to `nhal_result_t` codes
- [ ] Context and config structures properly defined
- [ ] Memory allocation behavior documented
- [ ] Thread safety model documented
- [ ] Timeout semantics implemented consistently
- [ ] Helper macros provided to reduce boilerplate

---

These principles emerged from embedded development experience and continue to evolve with the ecosystem.
