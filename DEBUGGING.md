# Debugging the STM32L476RCT6 with Raspberry Pi Pico

This guide explains how to use a Raspberry Pi Pico as a debug probe to debug the STM32L476RCT6 microcontroller on the Ledger Blue Developer Edition board.

## Overview

The Ledger Blue Developer Edition includes a 5-pin SWD (Serial Wire Debug) header that allows direct debugging of the STM32L476RCT6 non-secure microcontroller. By flashing the Raspberry Pi Pico with the Picoprobe firmware, you can turn it into a powerful and affordable CMSIS-DAP debug probe.

## Hardware Requirements

- **Ledger Blue Developer Edition board** (this PCB)
- **Raspberry Pi Pico** (to be used as debug probe)
- **5 female-to-female jumper wires** (or soldering equipment if you want permanent connections)
- **USB cable** (Micro-USB for Raspberry Pi Pico)
- **Computer** running Windows, Linux, or macOS

## Debug Header Pinout

The debug header (P3) on the Ledger Blue board has the following pins (from left to right when looking at the board):

| Pin # | Signal Name | Description |
|-------|-------------|-------------|
| 1 | MAIN_POWER | Power supply (3.3V) |
| 2 | SWCLK | Serial Wire Clock |
| 3 | GND | Ground |
| 4 | SWDIO | Serial Wire Data I/O |
| 5 | NRST | Reset (active low) |

![Debug Header](render-jtag.jpg)

## Pin Connections

Connect the Raspberry Pi Pico to the Ledger Blue debug header as follows:

| Ledger Blue Header | Signal | Raspberry Pi Pico Pin |
|--------------------|--------|----------------------|
| Pin 1 (MAIN_POWER) | 3.3V | Pin 36 (3V3(OUT)) |
| Pin 2 (SWCLK) | SWCLK | Pin 4 (GP2) |
| Pin 3 (GND) | GND | Pin 3, 8, 13, 18, 23, 28, 33, or 38 (any GND) |
| Pin 4 (SWDIO) | SWDIO | Pin 5 (GP3) |
| Pin 5 (NRST) | RESET | Pin 6 (GP4) |

**Important Notes:**
- The MAIN_POWER connection allows the Pico to power the STM32 through the debug header. If your board is already powered through USB or battery, you can omit this connection, but ensure both devices share a common ground.
- Use Pin 3 (GND) on the Pico for convenience as it's right next to the SWD pins.

## Step-by-Step Setup Guide

### Step 1: Flash Picoprobe Firmware on Raspberry Pi Pico

1. **Download the Picoprobe firmware:**
   - Get the latest `picoprobe.uf2` file from: https://github.com/raspberrypi/picoprobe/releases
   - Alternatively, build it yourself from the source: https://github.com/raspberrypi/picoprobe

2. **Enter bootloader mode on the Pico:**
   - Disconnect the Pico from USB
   - Hold down the BOOTSEL button on the Pico
   - While holding BOOTSEL, connect the Pico to your computer via USB
   - The Pico will appear as a USB mass storage device (like a USB drive)

3. **Flash the firmware:**
   - Copy the `picoprobe.uf2` file to the Pico's USB drive
   - The Pico will automatically reboot with the new firmware
   - The USB drive will disappear, and the Pico is now a debug probe

### Step 2: Connect the Hardware

1. **Power off the Ledger Blue board** (disconnect USB and battery)
2. **Connect the 5 wires** between the Raspberry Pi Pico and the Ledger Blue debug header according to the pin connections table above
3. **Double-check all connections** to avoid damaging the boards

### Step 3: Install OpenOCD

OpenOCD (Open On-Chip Debugger) is the software that communicates with the Picoprobe and controls the STM32.

#### On Linux (Ubuntu/Debian):
```bash
sudo apt-get update
sudo apt-get install openocd
```

#### On macOS:
```bash
brew install openocd
```

#### On Windows:
- Download pre-built OpenOCD binaries from: https://github.com/xpack-dev-tools/openocd-xpack/releases
- Extract and add to your PATH, or use the full path when running commands

### Step 4: Install Debugging Software

You'll need a toolchain and debugger. We recommend using GDB with the ARM toolchain.

#### Install ARM GCC Toolchain:

**Linux (Ubuntu/Debian):**
```bash
sudo apt-get install gcc-arm-none-eabi gdb-multiarch
```

**macOS:**
```bash
brew install --cask gcc-arm-embedded
brew install gdb
```

**Windows:**
- Download from: https://developer.arm.com/downloads/-/gnu-rm
- Install and add to PATH

### Step 5: Create OpenOCD Configuration

Create a file named `ledger-blue-picoprobe.cfg` with the following content:

```tcl
# Raspberry Pi Picoprobe adapter configuration
adapter driver cmsis-dap
adapter speed 1000

# STM32L476RCT6 target configuration
source [find target/stm32l4x.cfg]

# Reset configuration
reset_config srst_only srst_nogate

# If you want to halt on connect
init
reset init
```

### Step 6: Test the Connection

1. **Connect the Raspberry Pi Pico to your computer via USB**
2. **Power on the Ledger Blue board** (if not powered through the debug header)
3. **Run OpenOCD:**

```bash
openocd -f ledger-blue-picoprobe.cfg
```

If successful, you should see output similar to:
```
Open On-Chip Debugger 0.12.0
Licensed under GNU GPL v2
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : Using CMSIS-DAPv2 interface
Info : CMSIS-DAP: SWD supported
Info : clock speed 1000 kHz
Info : STM32L476RCTx.cpu: Cortex-M4 r0p1 processor detected
Info : target halted due to debug-request, current mode: Thread
Info : Listening on port 3333 for gdb connections
```

### Step 7: Debug with GDB

With OpenOCD running in one terminal, open a second terminal and start GDB:

```bash
# For Linux
gdb-multiarch your_firmware.elf

# For macOS/Windows
arm-none-eabi-gdb your_firmware.elf
```

In GDB, connect to OpenOCD:
```gdb
(gdb) target extended-remote localhost:3333
(gdb) monitor reset halt
(gdb) load
(gdb) continue
```

## Common GDB Commands

- `monitor reset halt` - Reset and halt the target
- `monitor reset init` - Reset and initialize the target
- `load` - Load firmware to the target
- `continue` (or `c`) - Continue execution
- `break main` - Set breakpoint at main function
- `step` (or `s`) - Step into function
- `next` (or `n`) - Step over function
- `print variable` - Print variable value
- `info registers` - Show register contents
- `backtrace` (or `bt`) - Show call stack

## Using with VS Code

For a better debugging experience, you can use Visual Studio Code with the Cortex-Debug extension:

1. **Install VS Code** from https://code.visualstudio.com/
2. **Install the Cortex-Debug extension** by marus25
3. **Add a launch configuration** to your `.vscode/launch.json`:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug STM32 (Picoprobe)",
            "type": "cortex-debug",
            "request": "launch",
            "servertype": "openocd",
            "cwd": "${workspaceRoot}",
            "executable": "${workspaceRoot}/build/your_firmware.elf",
            "device": "STM32L476RCT6",
            "configFiles": [
                "ledger-blue-picoprobe.cfg"
            ],
            "svdFile": "${workspaceRoot}/STM32L476.svd",
            "runToEntryPoint": "main",
            "showDevDebugOutput": "both"
        }
    ]
}
```

## Flashing Firmware

You can also use OpenOCD to flash firmware without debugging:

```bash
openocd -f ledger-blue-picoprobe.cfg -c "program your_firmware.elf verify reset exit"
```

Or for binary files:
```bash
openocd -f ledger-blue-picoprobe.cfg -c "program your_firmware.bin 0x08000000 verify reset exit"
```

## Troubleshooting

### OpenOCD can't find the Picoprobe
- **Check USB connection:** Ensure the Raspberry Pi Pico is connected and recognized by your computer
- **Check permissions (Linux):** You may need udev rules. Create `/etc/udev/rules.d/99-picoprobe.rules`:
  ```
  SUBSYSTEM=="usb", ATTR{idVendor}=="2e8a", ATTR{idProduct}=="000c", MODE="0666"
  ```
  Then reload: `sudo udevadm control --reload-rules && sudo udevadm trigger`

### Can't connect to STM32
- **Check wiring:** Verify all 5 connections are secure
- **Check power:** Ensure the Ledger Blue board is powered
- **Check NRST:** Make sure the reset line is connected properly
- **Try lower speed:** In the config file, change `adapter speed 1000` to `adapter speed 100`

### Debug connection drops
- **Power supply:** Make sure power supply is stable
- **Wire quality:** Use shorter wires and avoid loose connections
- **Interference:** Keep wires away from noise sources

### GDB won't connect
- **Check OpenOCD is running:** OpenOCD must be running before starting GDB
- **Check port:** Default is 3333, make sure it's not blocked by firewall
- **Check target state:** Use `monitor reset halt` if target is in weird state

## Technical Details

### STM32L476RCT6 Specifications
- **Core:** ARM Cortex-M4 with FPU
- **Flash:** 256 KB
- **RAM:** 96 KB SRAM
- **Max Clock:** 80 MHz
- **Debug Interface:** SWD (Serial Wire Debug)

### SWD Protocol
The SWD protocol uses only 2 data pins (SWCLK and SWDIO) compared to JTAG's 4-5 pins, making it more suitable for space-constrained applications. The protocol allows:
- Reading/writing memory
- Accessing CPU registers
- Setting breakpoints and watchpoints
- Halting and resuming execution
- Flash programming

## Additional Resources

- **STM32L476 Reference Manual:** https://www.st.com/resource/en/reference_manual/rm0351-stm32l47xxx-stm32l48xxx-stm32l49xxx-and-stm32l4axxx-advanced-armbased-32bit-mcus-stmicroelectronics.pdf
- **Picoprobe GitHub:** https://github.com/raspberrypi/picoprobe
- **OpenOCD Documentation:** https://openocd.org/doc/html/index.html
- **ARM Cortex-M4 Generic User Guide:** https://developer.arm.com/documentation/dui0553/latest/

## Safety Notes

⚠️ **Important Safety Information:**
- Always disconnect power before making or changing connections
- Double-check polarity and pin connections to avoid damage
- Do not connect external power supplies with different voltages
- The Ledger Blue contains a secure element (ST31G480) - debugging only works on the STM32L476, not the secure element
- Be careful not to short any pins

## License

This documentation is provided as-is for the Ledger Blue Developer Edition. Refer to the main repository LICENSE for terms.
