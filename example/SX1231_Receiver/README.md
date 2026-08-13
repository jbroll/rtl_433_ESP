# SX1231_Receiver

Minimal rtl_433_ESP receiver for an SX1231/RFM69 radio at 433.92 MHz. Decodes
OOK/ASK sensors and logs each one to the serial port.

## Build and flash

    pio run -e esp32s3-generic
    pio run -e esp32s3-generic -t upload
    pio device monitor

## Wiring

`platformio.ini` maps the radio to an ESP32-S3 as follows. Change the
`RF_MODULE_*` values to match your board.

| Signal | GPIO |
|---|---|
| MISO | 1 |
| MOSI | 42 |
| SCK | 41 |
| CS (NSS) | 40 |
| RST | 39 |
| DIO0 (IRQ) | 38 |
| DIO1 | 47 |
| DIO2 (data) | 21 |

DIO2 carries the continuous demodulated data and is the pin the decoder reads;
`RF_MODULE_RECEIVER_GPIO` is set to it automatically for `RF_RF69`.

## Tuning

`OOK_FIXED_THRESHOLD` sets the OOK slicer level. `0x50` works here; raise it in
a noisy band, lower it for weak sensors.

`MINIMUM_SIGNAL_DURATION` is the shortest capture handed to the decoder. It
defaults to `MINIMUM_SIGNAL_LENGTH`, which also sets the dropout bridging
window; set it alone to admit packets shorter than that window. One Acurite
592TXR packet is 39.1 ms.
