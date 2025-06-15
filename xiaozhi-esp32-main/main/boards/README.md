# Custom Board Guide

This guide describes how to customize a new board initialization program for the Xiaozhi AI voice chatbot project. Xiaozhi AI supports more than 70 ESP32 series boards, and the initialization code for each board is placed in the corresponding directory.

## Important Tips

> **Warning**: For custom boards, when the IO configuration is different from the original board, do not directly overwrite the original board configuration to compile the firmware. You must create a new board type, or use different name and sdkconfig macro definitions in the builds configuration in the config.json file to distinguish them. Use `python scripts/release.py [board directory name]` to compile and package the firmware.
>
> If you directly overwrite the original configuration, your custom firmware may be overwritten by the standard firmware of the original board during future OTA upgrades, causing your device to not work properly. Each board has a unique identifier and a corresponding firmware upgrade channel. It is very important to keep the uniqueness of the board identifier.

## Directory structure

The directory structure of each development board usually contains the following files:

- `xxx_board.cc` - The main board initialization code, which implements the board-related initialization and functions

- `config.h` - Board-level configuration file, which defines the hardware pin mapping and other configuration items

- `config.json` - Compile configuration, specifying the target chip and special compilation options

- `README.md` - Development board related documentation

## Custom development board steps

### 1. Create a new development board directory

First, create a new directory under the `boards/` directory, such as `my-custom-board/`:

```bash
mkdir main/boards/my-custom-board
```

### 2. Create a configuration file

#### config.h

Define all hardware configurations in `config.h`, including:

- Audio sampling rate and I2S pin configuration
- Audio codec chip address and I2C pin configuration
- Button and LED pin configuration
- Display parameters and pin configuration

Reference example (from lichuang-c3-dev):

```c
#ifndef _BOARD_CONFIG_H_
#define _BOARD_CONFIG_H_

#include <driver/gpio.h>

// Audio configuration
#define AUDIO_INPUT_SAMPLE_RATE 24000
#define AUDIO_OUTPUT_SAMPLE_RATE 24000

#define AUDIO_I2S_GPIO_MCLK GPIO_NUM_10
#define AUDIO_I2S_GPIO_WS GPIO_NUM_12
#define AUDIO_I2S_GPIO_BCLK GPIO_NUM_8
#define AUDIO_I2S_GPIO_DIN GPIO_NUM_7
#define AUDIO_I2S_GPIO_DOUT GPIO_NUM_11

#define AUDIO_CODEC_PA_PIN GPIO_NUM_13
#define AUDIO_CODEC_I2C_SDA_PIN GPIO_NUM_0
#define AUDIO_CODEC_I2C_SCL_PIN GPIO_NUM_1
#define AUDIO_CODEC_ES8311_ADDR ES8311_CODEC_DEFAULT_ADDR

//Button configuration
#define BOOT_BUTTON_GPIO GPIO_NUM_9

//Display configuration
#define DISPLAY_SPI_SCK_PIN GPIO_NUM_3
#define DISPLAY_SPI_MOSI_PIN GPIO_NUM_5
#define DISPLAY_DC_PIN GPIO_NUM_6
#define DISPLAY_SPI_CS_PIN GPIO_NUM_4

#define DISPLAY_WIDTH 320
#define DISPLAY_HEIGHT 240
#define DISPLAY_MIRROR_X true
#define DISPLAY_MIRROR_Y false
#define DISPLAY_SWAP_XY true

#define DISPLAY_OFFSET_X 0
#define DISPLAY_OFFSET_Y 0

#define DISPLAY_BACKLIGHT_PIN GPIO_NUM_2
#define DISPLAY_BACKLIGHT_OUTPUT_INVERT true

#endif // _BOARD_CONFIG_H_
```

#### config.json

Define the compilation configuration in `config.json`:

```json
{
"target": "esp32s3", // Target chip model: esp32, esp32s3, esp32c3, etc.
"builds": [
{
"name": "my-custom-board", // Development board name
"sdkconfig_append": [
// Additional required compilation configuration
"CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y",
"CONFIG_PARTITION_TABLE_CUSTOM_FILENAME=\"partitions/v1/8m.csv\"",
"CONFIG_PARTITION_TABLE_CUSTOM=y",
"CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"",
"CONFIG_PARTITION_TABLE_FILENAME="partitions.csv"",
"CONFIG_PARTITION_TABLE_OFFSET=0x8000",
"CONFIG_PARTITION_TABLE_MD5=y",
]
}
]
}
```

### 3. Write board-level initialization code

Create a `my_custom_board.cc` file to implement all initialization logic for the development board.

A basic development board class definition includes the following parts:

1. **Class definition**: inherited from `WifiBoard` or `Ml307Board`

2. **Initialization function**: including initialization of I2C, display, button, IoT and other components

3. **Virtual function rewrite**: such as `GetAudioCodec()`, `GetDisplay()`, `GetBacklight()`, etc.

4. **Register development board**: Use `DECLARE_BOARD` macro to register the development board

```cpp
#include "wifi_board.h"
#include "audio_codecs/es8311_audio_codec.h"
#include "display/lcd_display.h"
#include "application.h"
#include "button.h"
#include "config.h"
#include "iot/thing_manager.h"

#include <esp_log.h>
#include <driver/i2c_master.h>
#include <driver/spi_common.h>

#define TAG "MyCustomBoard"

// declare font
LV_FONT_DECLARE(font_puhui_16_4);
LV_FONT_DECLARE(font_awesome_16_4);

class MyCustomBoard : public WifiBoard {
private: 
i2c_master_bus_handle_t codec_i2c_bus_; 
Button boot_button_; 
LcdDisplay* display_; 

//I2C initialization 
void InitializeI2c() { 
i2c_master_bus_config_t i2c_bus_cfg = { 
.i2c_port = I2C_NUM_0, 
.sda_io_num = AUDIO_CODEC_I2C_SDA_PIN, 
.scl_io_num = AUDIO_CODEC_I2C_SCL_PIN, 
.clk_source = I2C_CLK_SRC_DEFAULT, 
.glitch_ignore_cnt = 7, 
.intr_priority = 0, 
.trans_queue_depth = 0, 
.flags = { 
.enable_internal_pullup = 1, 
}, 
}; 
ESP_ERROR_CHECK(i2c_new_master_bus(&i2c_bus_cfg, &codec_i2c_bus_)); 
} 

// SPI initialization (for display) 
void InitializeSpi() { 
spi_bus_config_t buscfg = {}; 
buscfg.mosi_io_num = DISPLAY_SPI_MOSI_PIN; 
buscfg.miso_io_num = GPIO_NUM_NC; buscfg.sclk_io_num = DISPLAY_SPI_SCK_PIN; 
buscfg.quadwp_io_num = GPIO_NUM_NC; 
buscfg.quadhd_io_num = GPIO_NUM_NC; 
buscfg.max_transfer_sz = DISPLAY_WIDTH * DISPLAY_HEIGHT * sizeof(uint16_t); 
ESP_ERROR_CHECK(spi_bus_initialize(SPI2_HOST, &buscfg, SPI_DMA_CH_AUTO)); 
} 

//Button initialization 
void InitializeButtons() { 
boot_button_.OnClick([this]() { 
auto& app = Application::GetInstance(); 
if (app.GetDeviceState() == kDeviceStateStarting && !WifiStation::GetInstance().IsConnected()) { 
ResetWifiConfiguration(); } 
app.ToggleChatState(); 
}); 
} 

//Display initialization (take ST7789 as an example) 
void InitializeDisplay() { 
esp_lcd_panel_io_handle_t panel_io = nullptr; 
esp_lcd_panel_handle_t panel = nullptr; 

esp_lcd_panel_io_spi_config_t io_config = {}; 
io_config.cs_gpio_num = DISPLAY_SPI_CS_PIN; 
io_config.dc_gpio_num = DISPLAY_DC_PIN; 
io_config.spi_mode = 2; 
io_config.pclk_hz = 80 * 1000 * 1000; 
io_config.trans_queue_depth =