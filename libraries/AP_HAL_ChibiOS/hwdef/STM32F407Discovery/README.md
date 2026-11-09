# STM32F407 Discovery (STM32F407G-DISC1)

Development target for the ST STM32F407G-DISC1 board (formerly
STM32F4DISCOVERY). It is intended for working on the ArduPilot bootloader
and ChibiOS HAL on cheap, easily recoverable hardware. The board has no IMU or
barometer supported by ArduPilot, so vehicle firmware boots to an
`INS: unable to initialise driver` config error, but USB and MAVLink still work.

## Hardware summary

- MCU: STM32F407VGT6, 168 MHz, 1024 KB flash, 192 KB RAM
- 8 MHz HSE crystal
- On-board ST-LINK/V2 (mini-USB CN1) for SWD flashing and debugging
- USB OTG FS on micro-USB CN5 (PA11/PA12, VBUS on PA9)
- LEDs (active high): LD4 green PD12, LD3 orange PD13, LD5 red PD14, LD6 blue PD15
- B1 user button on PA0 (active high), B2 is reset

## Flash layout

| Region      | Address      | Size   | Sectors |
|-------------|--------------|--------|---------|
| Bootloader  | `0x08000000` | 32 KB  | 0-1     |
| Parameters  | `0x08008000` | 32 KB  | 2-3     |
| Application | `0x08010000` | 960 KB | 4-11    |

## Pinout

| Function          | Pins                      |
|-------------------|---------------------------|
| SERIAL0           | USB (CN5)                 |
| SERIAL1 (USART2)  | PA2 TX, PA3 RX            |
| SERIAL2 (USART3)  | PD8 TX, PD9 RX            |
| PWM 1-4 (TIM1)    | PE9, PE11, PE13, PE14     |
| Battery voltage   | PC1 (ADC pin 11)          |
| Battery current   | PC2 (ADC pin 12)          |

The bootloader listens on USB and on USART2 (PA2/PA3) at 115200 baud.

## LED meaning

| LED              | State                       | Meaning                                       |
|------------------|-----------------------------|-----------------------------------------------|
| LD6 blue         | Fast blink (~10 Hz)         | Bootloader running, waiting for an upload     |
| LD6 blue         | Slow toggle (1 s)           | Bootloader found no valid application         |
| LD4 green        | Flicker                     | Bootloader protocol activity                  |
| Red/green/blue   | ArduPilot notify patterns   | Application is running                        |

## Building

From the repository root:

```sh
# bootloader: writes Tools/bootloaders/STM32F407Discovery_bl.{bin,hex,elf}
Tools/scripts/build_bootloaders.py STM32F407Discovery

# firmware: writes build/STM32F407Discovery/bin/arducopter{.apj,.bin,_with_bl.hex}
./waf configure --board STM32F407Discovery
./waf copter
```

The firmware build embeds `Tools/bootloaders/STM32F407Discovery_bl.bin`, so
build the bootloader first if you have changed it.

## Testing the bootloader

### 1. Flash the bootloader over SWD

Connect the mini-USB (CN1, ST-LINK). Erase the whole chip so that no
application is present, then write the bootloader at `0x08000000`. Use any
one of these tools:

```sh
# STM32CubeProgrammer
STM32_Programmer_CLI -c port=SWD -e all -w Tools/bootloaders/STM32F407Discovery_bl.bin 0x08000000 -v -rst

# OpenOCD
openocd -f board/stm32f4discovery.cfg \
  -c "init; reset halt; flash erase_sector 0 0 last" \
  -c "program Tools/bootloaders/STM32F407Discovery_bl.hex verify reset exit"

# stlink tools
st-flash erase
st-flash --reset write Tools/bootloaders/STM32F407Discovery_bl.bin 0x08000000
```

Expected: LD6 (blue) toggles slowly, because there is no application yet.

### 2. Check USB enumeration

Connect the micro-USB (CN5) to the PC. The board should enumerate as a CDC
serial port with the product string `STM32F407Discovery-BL`.

- Linux: `ls /dev/serial/by-id/` shows `usb-ArduPilot_STM32F407Discovery-BL_...`
- Windows: a new COM port appears in Device Manager
- WSL: attach the device first with `usbipd attach --wsl --busid <id>`, or run
  the uploader from Windows Python against the COM port

### 3. Identify the board through the bootloader protocol

```sh
Tools/scripts/uploader.py --identify --port /dev/serial/by-id/usb-ArduPilot_STM32F407Discovery-BL*
```

Expected: board ID `99` (`TARGET_HW_PX4_DISCOVERY_V1`), bootloader protocol
revision and a flash size of 960 KB available for the application.

### 4. Upload firmware through the bootloader

```sh
Tools/scripts/uploader.py build/STM32F407Discovery/bin/arducopter.apj
# or, equivalently
./waf copter --upload
```

Expected: erase, program and verify complete, then the bootloader jumps to the
application. The USB port re-enumerates as `STM32F407Discovery` (no `-BL`)
and MAVLink heartbeats are visible, for example with
`mavproxy.py --master /dev/serial/by-id/usb-ArduPilot_STM32F407Discovery_*`.
The `INS: unable to initialise driver` message is expected on this board.

Mission Planner's "Load custom firmware" with the `.apj` file exercises the
same path.

### 5. Re-enter the bootloader

Test each entry path. In each case the blue LED should blink fast and the
port should show up as `STM32F407Discovery-BL` again.

- **MAVLink reboot:** run the uploader again while the application is
  running; it sends `MAV_CMD_PREFLIGHT_REBOOT_SHUTDOWN` with param1 = 3. From
  MAVProxy, `reboot bootloader` does the same.
- **User button:** hold B1 (blue) and press B2 (black reset). The bootloader
  waits indefinitely instead of starting the application.
- **No reset path:** with a valid application, a plain reset (B2 only) should
  start the application immediately.

### 6. Negative tests

- **Invalid application:** erase only the application area and reset. The
  bootloader must not jump to it and LD6 should toggle slowly:

  ```sh
  STM32_Programmer_CLI -c port=SWD -e [4 11] -rst
  # or, with stlink 1.7.0 or later
  st-flash erase 0x08010000 0xF0000
  ```

- **Wrong board ID:** upload an `.apj` built for another board. The uploader
  must refuse with a board ID mismatch unless `--force` is given.
- **Interrupted upload:** unplug USB in the middle of an upload, reconnect,
  and confirm the bootloader stays active and accepts a new upload.

### 7. Serial (USART2) path

Connect a 3.3 V USB-serial adapter: adapter RX to PA2, adapter TX to PA3, GND
to GND. Hold B1 while pressing reset, then:

```sh
Tools/scripts/uploader.py --port /dev/ttyUSB0 --baud-bootloader 115200 \
  build/STM32F407Discovery/bin/arducopter.apj
```

### 8. Updating the bootloader from the application

The application carries a copy of the bootloader in ROMFS. With the
application running, `flashbootloader` in MAVProxy (which sends
`MAV_CMD_FLASH_BOOTLOADER`) rewrites sectors 0-1. Reboot and repeat steps 3
and 5. If this goes wrong the board is recovered by repeating step 1 over SWD.

### 9. Debugging with GDB

```sh
openocd -f board/stm32f4discovery.cfg
# in a second terminal
arm-none-eabi-gdb Tools/bootloaders/STM32F407Discovery_bl.elf \
  -ex "target extended-remote :3333" -ex "monitor reset halt"
```

Useful breakpoints are `main`, `jump_to_app` and `bootloader`.
