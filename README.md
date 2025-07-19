# MAX17048 Fuel Gauge Driver

ESP-IDF driver component for the MAX17048 battery fuel gauge IC with I2C interface and runtime configuration support.

## Features

- **Accurate Battery Monitoring**: Read State of Charge (SOC), voltage, and charge/discharge rate
- **I2C Master API**: Uses latest ESP-IDF I2C master driver (no deprecation warnings)
- **Runtime Configuration**: Flexible initialization without Kconfig dependencies
- **Shared Bus Support**: Works with multiple I2C devices on the same bus
- **Low Power**: Sleep mode support for power-conscious applications
- **Easy Integration**: Simple API for quick battery monitoring implementation

## Hardware Connection

The MAX17048 communicates via I2C and requires minimal external components:

```
ESP32           MAX17048
=====           ========
GPIO 0     →    SCL
GPIO 1     →    SDA
3.3V       →    VDD
GND        →    GND/VSS
Battery+   →    CELL+ (via 1kΩ resistor)
Battery-   →    CELL-
```

**Note**: External pull-up resistors (10kΩ) recommended for SCL/SDA lines.

## Usage Example

### Basic Initialization (New I2C Master API)

```c
#include "max17048.h"
#include "driver/i2c_master.h"

void app_main(void)
{
    // Setup I2C master bus
    i2c_master_bus_handle_t i2c_bus;
    i2c_master_bus_config_t i2c_config = {
        .clk_source = I2C_CLK_SRC_DEFAULT,
        .i2c_port = I2C_NUM_0,
        .scl_io_num = GPIO_NUM_0,
        .sda_io_num = GPIO_NUM_1,
        .glitch_ignore_cnt = 7,
        .flags.enable_internal_pullup = false,  // Use external pull-ups
    };
    ESP_ERROR_CHECK(i2c_new_master_bus(&i2c_config, &i2c_bus));
    
    // Configure MAX17048
    max17048_config_t max_config;
    max17048_get_default_config(&max_config);
    max_config.i2c_bus_handle = i2c_bus;
    max_config.device_address = 0x36;  // MAX17048 I2C address
    max_config.i2c_freq_hz = 100000;   // 100kHz I2C frequency
    
    // Initialize fuel gauge
    ESP_ERROR_CHECK(max17048_init_on_bus_with_config(&max_config));
    
    // Read battery data
    float soc, voltage, charge_rate;
    if (max17048_get_soc(&soc) == ESP_OK) {
        printf("Battery SOC: %.2f%%\\n", soc);
    }
    
    if (max17048_get_voltage(&voltage) == ESP_OK) {
        printf("Battery Voltage: %.3fV\\n", voltage);
    }
    
    if (max17048_get_crate(&charge_rate) == ESP_OK) {
        printf("Charge Rate: %.2f%%/hr\\n", charge_rate);
    }
}
```

### Shared I2C Bus Example

```c
// Multiple devices on same I2C bus
i2c_master_bus_handle_t shared_i2c_bus;

// Initialize MAX17048
max17048_config_t max_config;
max17048_get_default_config(&max_config);
max_config.i2c_bus_handle = shared_i2c_bus;
max_config.device_address = 0x36;
ESP_ERROR_CHECK(max17048_init_on_bus_with_config(&max_config));

// Initialize other I2C device on same bus
// other_device_init(shared_i2c_bus, other_address);
```

### Battery Monitoring Task

```c
void battery_monitor_task(void *pvParameters)
{
    while (1) {
        float soc, voltage, charge_rate;
        
        if (max17048_get_soc(&soc) == ESP_OK &&
            max17048_get_voltage(&voltage) == ESP_OK &&
            max17048_get_crate(&charge_rate) == ESP_OK) {
            
            printf("Battery Status: SOC=%.1f%%, V=%.2fV, Rate=%.1f%%/hr\\n", 
                   soc, voltage, charge_rate);
                   
            // Check for low battery
            if (soc < 10.0f) {
                printf("WARNING: Low battery!\\n");
            }
            
            // Check charging state
            if (charge_rate > 0.0f) {
                printf("Battery charging at %.1f%%/hr\\n", charge_rate);
            } else if (charge_rate < 0.0f) {
                printf("Battery discharging at %.1f%%/hr\\n", -charge_rate);
            }
        }
        
        vTaskDelay(pdMS_TO_TICKS(5000)); // Update every 5 seconds
    }
}
```

## API Reference

### Configuration Functions

- `max17048_get_default_config()` - Get default configuration structure
- `max17048_init_on_bus_with_config()` - Initialize with runtime configuration
- `max17048_deinit()` - Deinitialize and cleanup resources

### Battery Reading Functions

- `max17048_get_soc()` - Read State of Charge (percentage)
- `max17048_get_voltage()` - Read battery voltage (volts)
- `max17048_get_crate()` - Read charge/discharge rate (%/hour)

### Device Information

- `max17048_get_version()` - Read device version
- `max17048_get_config_reg()` - Read configuration register

### Power Management

- `max17048_sleep()` - Enter sleep mode
- `max17048_wake()` - Wake from sleep mode
- `max17048_quick_start()` - Force quick-start for accurate SOC

### Legacy Support (Deprecated)

- `max17048_init()` - Legacy initialization (returns ESP_ERR_NOT_SUPPORTED)
- `max17048_init_custom()` - Legacy custom init (returns ESP_ERR_NOT_SUPPORTED)

## Configuration Structure

```c
typedef struct {
    i2c_master_bus_handle_t i2c_bus_handle;  // I2C master bus handle
    uint8_t device_address;                  // I2C device address (0x36)
    uint32_t i2c_freq_hz;                    // I2C frequency in Hz
} max17048_config_t;
```

## Technical Specifications

- **Supply Voltage**: 2.5V to 5.0V
- **I2C Address**: 0x36 (7-bit)
- **I2C Speed**: Up to 400kHz
- **SOC Range**: 0% to 100% (0.00390625% resolution)
- **Voltage Range**: 0V to 5.12V (78.125µV resolution)
- **Rate Range**: ±32%/hr (0.208%/hr resolution)
- **Operating Temperature**: -40°C to +85°C

## Error Codes

- `ESP_OK` - Success
- `ESP_ERR_INVALID_ARG` - Invalid argument
- `ESP_ERR_NOT_FOUND` - Device not found on I2C bus
- `ESP_FAIL` - I2C communication error
- `ESP_ERR_NOT_SUPPORTED` - Legacy function not supported

## Dependencies

- `driver` - ESP-IDF driver framework
- `esp_common` - ESP common utilities  
- `freertos` - FreeRTOS kernel
- `log` - ESP logging framework

## Migration from Legacy Driver

The component now uses the new I2C master API. Update your code:

```c
// Old (deprecated)
max17048_init_custom(I2C_NUM_0, SDA_PIN, SCL_PIN);

// New (recommended)
max17048_config_t config;
max17048_get_default_config(&config);
config.i2c_bus_handle = your_i2c_bus_handle;
config.device_address = 0x36;
max17048_init_on_bus_with_config(&config);
```

## Troubleshooting

- **Device not found**: Check I2C wiring and pull-up resistors
- **Inaccurate readings**: Perform quick-start calibration
- **Communication errors**: Verify I2C frequency and bus configuration
- **Power issues**: Ensure proper supply voltage (2.5V-5.0V)

## License

This component is provided under the MIT license.

## Datasheet

For detailed specifications, refer to the [MAX17048 Datasheet](https://www.analog.com/media/en/technical-documentation/data-sheets/MAX17048-MAX17049.pdf).