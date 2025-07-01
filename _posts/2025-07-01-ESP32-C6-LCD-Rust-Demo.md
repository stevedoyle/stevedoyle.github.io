---
title: "ESP32-C6-LCD-1.47 LCD Demo in Rust"
tags:
  - Rust
---

The **ESP32-C6** is a powerful and versatile microcontroller from Espressif. With
its Wi-Fi 6, Bluetooth 5 (LE), and Zigbee/Thread support, it's a fantastic
choice for a wide range of connected devices.

Rust is quickly becoming a contender for embedded systems, where reliability and
efficiency are paramount. It offers memory safety guarantees without sacrificing
performance.

While a wealth of information exists for programming Espressif devices with
Rust, particularly from the excellent [esp-rs](https://github.com/esp-rs)
project, I found it difficult to find an example for using Rust to program the
**ST7789** LCD display on the **ESP32-C6-LCD-1.47** development kit.

All of the code described in this post is available on GitHub at:
[stevedoyle/esp32-c6-lcd-demo-rs](https://github.com/stevedoyle/esp32-c6-lcd-demo-rs/tree/master).

## Prerequisites

- A ESP32 board with an LCD screen. I used a ESP32-C6-LCD-1.47 kit which has a
  ST7789 LCD display with a resolution of 172 x 320 pixels.
- Setup an Rust ESP development environment and tools as described in [The Rust
  on ESP Book](https://docs.esp-rs.org/book/).

## Rust Crate Dependencies

The project uses the following key dependencies:

- [esp-idf-svc](https://docs.esp-rs.org/esp-idf-svc/esp_idf_svc/): ESP-IDF
  service layer for Rust
- [mipidsi](https://docs.rs/mipidsi/latest/mipidsi/): MIPI DSI driver for
  various display controllers
- [embedded-graphics](https://docs.rs/embedded-graphics/latest/embedded_graphics/):
  2D graphics library for embedded devices
- [display-interface-spi](https://docs.rs/display-interface-spi/latest/display_interface_spi/):
  SPI interface for display communication

## LCD Pin Configuration

The following GPIO pins are used for the ST7789 display interface:

| Function | GPIO Pin | Description |
|----------|----------|-------------|
| SCLK     | GPIO7    | SPI Clock |
| MOSI/SDA | GPIO6    | SPI Data Out |
| MISO     | GPIO5    | SPI Data In (required by SPI driver) |
| DC       | GPIO15   | Data/Command selection |
| RST      | GPIO21   | Reset pin |
| CS       | GPIO14   | Chip Select (optional) |
| Backlight| GPIO22   | Backlight control |

**Note**: These pin assignments match the ESP32-C6-LCD-1.47 board. Adjust according to your specific board configuration.

## Configuring the Display

At a high level, programming the display is quite straightforward:

1. Setup the display hardware using `create_portrait_display()`.
2. Clear the display.
3. Draw content on the display. This is done by `draw_examples()` in the demo.

```rust
let peripherals = Peripherals::take()?;

let mut display = create_portrait_display(peripherals)?;

display.clear(Rgb565::BLACK)?;
draw_examples(&mut display)?;
```

Let's start off with some helper structures for encapsulating the display
configuration, with defaults set for ESP32-C6-LCD-1.47 with an LCD size of
172x320 pixels and a top left offset of (34, 0).

```rust
pub enum DisplayOrientation {
    Portrait,
    Landscape,
}

pub struct DisplayConfig {
    pub width: u16,
    pub height: u16,
    pub offset_x: u16,
    pub offset_y: u16,
    pub orientation: DisplayOrientation,
}

impl DisplayConfig {
    pub fn default() -> Self {
        Self {
            width: 172,
            height: 320,
            offset_x: 34,
            offset_y: 0,
            orientation: DisplayOrientation::Portrait,
        }
    }
}
```

And now the code for configuring the display hardware.

```rust
// Convenience functions for creating displays
pub fn create_portrait_display(
    peripherals: Peripherals,
) -> Result<impl DrawTarget<Color = Rgb565>, EspError> {
    create_display(peripherals, DisplayConfig::default())
}
```

```rust
pub fn create_display(
    peripherals: Peripherals,
    config: DisplayConfig,
) -> Result<impl DrawTarget<Color = Rgb565>, EspError> {
    // Configure SPI pins for ESP32-C6-LCD
    let spi = SpiDriver::new(
        peripherals.spi2,
        peripherals.pins.gpio7,       // SCLK
        peripherals.pins.gpio6,       // MOSI
        Some(peripherals.pins.gpio5), // MISO (required by SPI driver)
        &esp_idf_svc::hal::spi::config::DriverConfig::new(),
    )?;

    // Configure DC pin (Data/Command)
    let dc = PinDriver::output(peripherals.pins.gpio15)?;
    // Configure RST pin (Reset)
    let rst = PinDriver::output(peripherals.pins.gpio21)?;

    // Configure backlight pin
    let mut backlight = PinDriver::output(peripherals.pins.gpio22)?;
    backlight.set_high()?; // Turn on backlight

    let spi_device = SpiDeviceDriver::new(
        spi,
        Some(peripherals.pins.gpio14), // Optional CS pin
        &Config::new(),
    )?;

    // Create display interface
    let di = SPIInterface::new(spi_device, dc);

    let orientation = match config.orientation {
        DisplayOrientation::Portrait => Orientation::new().rotate(Rotation::Deg0), // Portrait mode
        DisplayOrientation::Landscape => Orientation::new().rotate(Rotation::Deg90), // Landscape mode
    };

    // Initialize ST7789 display using mipidsi
    let display = Builder::new(ST7789, di)
        .reset_pin(rst)
        .display_size(config.width, config.height)
        .display_offset(config.offset_x, config.offset_y)
        .color_order(ColorOrder::Rgb)
        .invert_colors(ColorInversion::Inverted)
        .orientation(orientation) // Set orientation to landscape, no mirroring
        .init(&mut FreeRtos)
        .map_err(|_| EspError::from_non_zero(std::num::NonZeroI32::new(1).unwrap()))?;

    // Keep backlight alive by forgetting it (it will stay on)
    std::mem::forget(backlight);

    Ok(display)
}
```

This is broken into two parts.

1. Configure the SPI driver. This involves mapping the GPIO pins to the SPI interface based on the hardware configuration.
2. Configure the LCD display itself using `mioidsi`. This involves configuring the display type, size, offsets, colors and orientation (portrait or landscape).

### Adding Graphics

The [embedded-graphics](https://docs.rs/embedded-graphics/latest/embedded_graphics/) crate provides a rich set of functionality for drawing text and simple graphics.

Text ...

```rust
    // Define text style
    let text_style = MonoTextStyle::new(&FONT_9X15, Rgb565::WHITE);
    let large_text_style = MonoTextStyle::new(&FONT_10X20, Rgb565::YELLOW);

    // Draw title
    Text::with_baseline(
        "ESP32-C6-LCD",
        Point::new(10, 30),
        large_text_style,
        Baseline::Top,
    )
    .draw(display)
    .map_err(|_| EspError::from_non_zero(std::num::NonZeroI32::new(1).unwrap()))?;
```

Simple graphics ...

```rust
    // Draw a red rectangle
    Rectangle::new(Point::new(10, 80), Size::new(100, 60))
        .into_styled(PrimitiveStyle::with_fill(Rgb565::RED))
        .draw(display)
        .map_err(|_| EspError::from_non_zero(std::num::NonZeroI32::new(1).unwrap()))?;
```

## The Full Demo

The full source code for this demo is available at [stevedoyle/esp32-c6-lcd-demo-rs](https://github.com/stevedoyle/esp32-c6-lcd-demo-rs/tree/master).

To run the demo ..

1. Make sure you have the ESP-IDF and Rust toolchain installed

2. Set up the ESP32-C6 target:

    ```bash
    espup install
    . $HOME/export-esp.sh
    ```

3. Build the project:

    ```bash
    cargo build
    ```

4. Flash to your ESP32-C6:

    ```bash
    cargo run
    ```
