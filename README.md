🔍 Key Rationale & Design Notes
1. ADC Selection (GPIO7 & GPIO8)

    Use `ADC1`, not `ADC2`: On ESP32-S3, `ADC2` is shared with Wi-Fi/Bluetooth radios. If your project ever enables wireless, `ADC2` readings will fail or block. `ADC1` is fully independent.
    Both on `ADC1: GPIO7 (ADC1_CH6)` and `GPIO8 (ADC1_CH7)` belong to the same controller, allowing unified calibration, attenuation settings, and continuous/oneshot driver usage.
    Calibration is mandatory: ESP32-S3 ADCs are non-linear. Use ESP-IDF's esp_adc_cal or Arduino's analogReadResolution() + lookup table for accurate voltage mapping.
    Attenuation: Set to 11 dB (or ADC_ATTEN_DB_11) for ~0–3.3V range. Lower attenuations cap at ~1.1V or ~1.5V.

2. I2C Selection (GPIO41 & GPIO42)

    Native multi-device support: I2C inherently supports multiple slaves on one bus. You only need:
        Unique 7-bit addresses for both devices
        External pull-up resistors (2.2kΩ–4.7kΩ to 3.3V)
        100kHz is Standard Mode, well within spec
    GPIO41/42 advantages: High-number GPIOs on S3 are free of strapping, boot, USB-JTAG, and ADC functions. They route cleanly through the GPIO matrix to I2C0 or I2C1.
    Bus sharing tip: If devices have identical addresses, you'll need an I2C multiplexer (e.g., TCA9548A). Otherwise, standard bus topology works.

🔄 Alternative Pin Options (if routing conflicts arise)
    For ADC:  GPIO8 & GPIO9  or GPIO1 & GPIO7 (avoid GPIO1 if UART0 is exposed)
    For I2C: GPIO38 & GPIO39 or GPIO15 & GPIO16 (note: these are ADC2 pins, but safe as digital I2C)
	
⚠️ Important Implementation Tips

    I2C Pull-ups: ESP32-S3 internal pull-ups (~40kΩ) are too weak for 100kHz with 2 devices. Add external 4.7kΩ resistors to 3.3V.
    ADC Sampling: Use adc_oneshot driver for simplicity. If you need synchronized readings, consider adc_continuous with DMA.
    Noise Mitigation: Place a 0.1µF decoupling capacitor near each analog input pin to VSS. Keep analog traces away from high-speed SPI lines (your OLED).

3. ESP-IDF/Arduino Setup:

>// I2C Init (ESP-IDF)
>i2c_config_t conf = {
>   .mode = I2C_MODE_MASTER,
>   .sda_io_num = 42,
>   .scl_io_num = 41,
>   .sda_pullup_en = GPIO_PULLUP_ENABLE,
>   .scl_pullup_en = GPIO_PULLUP_ENABLE,
>   .master.clk_speed = 100000,
>};
>i2c_param_config(I2C_NUM_0, &conf);
>i2c_driver_install(I2C_NUM_0, conf.mode, 0, 0, 0);	

(Arduino equivalent uses `Wire.begin(42, 41, 100000)`)

Here is a complete, production-ready Arduino sketch tailored for your ESP32-S3 setup. It uses your +2.5V precision reference to dynamically calibrate the ADC gain, reads the battery through a voltage divider, and calculates State of Charge (SOC) using a standard 2S LiPo discharge curve approximation.

🔍 Key Implementation Notes
1. Voltage Divider Calculation

    You must update `DIVIDER_RATIO` to match your actual resistors:
    DIVIDER_RATIO = R_bottom / (R_top + R_bottom)

    Example: R_top = 120kΩ, R_bottom = 75kΩ → 75 / 195 = 0.3846

    At 8.4V battery: 8.4 × 0.3846 = 3.23V (safe within 3.3V ADC max)
    Use 1% tolerance resistors or measure the actual ratio with a DMM for best accuracy.

2. Why Software Calibration via 2.5V Reference?

    ESP32-S3 ADCs have inherent gain/offset errors that vary between chips and with temperature.
    Using a precision reference at runtime automatically compensates for these errors without needing ESP-IDF's deprecated `esp_adc_cal` or complex multi-point tables.
    The calibration runs once at boot. You can call `calibrateADC()` periodically if your reference source has low drift.

3. ADC Configuration & Arduino Version Compatibility

    The `#if ESP_ARDUINO_VERSION` block handles the API change between Arduino-ESP32 v2.x and v3.x.
    `ADC_11db/ADC_ATTEN_DB_11` sets the input range to `~0-3.3V`. This is required since your divider outputs up to `3.23V`
    
4. SOC Calculation Accuracy

    LiPo discharge curves are non-linear and load-dependent. The piecewise function used here matches standard light-load curves (~0.2C).
    For higher accuracy under varying loads, replace `calculateSOC()` with a 1D lookup table interpolated by voltage, or integrate current (Coulomb counting) if you have a shunt..

5. Hardware Recommendations

    Add a `100nF` ceramic capacitor from `GPIO7` to `GND` (as close to the pin as possible) to filter high-frequency noise from the voltage divider.
    Keep `GPIO8` (2.5V ref) isolated from digital switching noise. Use a short trace and add a `10kΩ` series resistor if the reference source has high output impedance.
    Ensure your voltage divider resistors are `≥100kΩ` total to minimize battery drain (`<50µA` at 8.4V).