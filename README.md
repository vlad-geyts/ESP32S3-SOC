🔍 Key Rationale & Design Notes
1. ADC Selection (GPIO7 & GPIO8)

    Use ADC1, not ADC2: On ESP32-S3, ADC2 is shared with Wi-Fi/Bluetooth radios. If your project ever enables wireless, ADC2 readings will fail or block. ADC1 is fully independent.
    Both on ADC1: GPIO7 (ADC1_CH6) and GPIO8 (ADC1_CH7) belong to the same controller, allowing unified calibration, attenuation settings, and continuous/oneshot driver usage.
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
	

	
