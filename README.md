Based on the schematic, netlist, and BOM provided, I have revised the `Pico_Development_Stack.md` file.

**Significant Changes Made:**
1.  **Chip Model Update:** Updated the MCU reference from **RP2350B** to **RP2354B**. The BOM specifies the `RP2354B`, which includes **2MB of internal stacked flash**. This is a significant hardware detail as it eliminates the need for external flash chips on the analyzer/probe sections.
2.  **USB Hub Details:** Clarified the USB Hub (GL850G) connectivity. The board includes a **stacked USB-A connector (J10)**, providing *two* downstream ports. This allows the user to patch the Target Pico's USB data lines back into the system if needed.
3.  **Power Supply:** Added a note about the **2A Buck Converter (MT2492)**, which ensures robust power for the stack and the target device.
4.  **DUT USB Clarification:** Added a critical note that the **Target Pico's USB data lines are NOT passed through the headers**. Users must use a patch cable from the auxiliary ports if they need USB data connectivity to the Target Pico.

Here is the revised file content:

---

# Raspberry Pi Pico Development Stack

**A professional-grade hardware interposer for debugging, analyzing, and validating Raspberry Pi Pico projects.**

![Project Status](https://img.shields.io/badge/Status-Prototype-orange) ![License](https://img.shields.io/badge/License-MIT-green)

## Overview

The **Pico Development Stack** is a "Man-in-the-Middle" development tool designed to eliminate the wiring mess typically associated with hardware debugging.

It acts as a hardware shim:
1.  Unplug your **Target Pico (DUT)** from your project motherboard.
2.  Plug the **Debug Stack** into your motherboard.
3.  Plug your **Target Pico** into the socket on top of the Debug Stack.

With a single USB connection to your PC, you instantly gain access to a full suite of hardware analysis tools connected directly to the DUT's pins, with no breadboards or jumper wires required.

## Key Features

*   **Integrated Logic Analysis:** Onboard **RP2354B** (with 2MB internal flash) running **[Dr. Gusman's Logic Analyzer](https://github.com/gusmanb/logicanalyzer)**. Capable of capturing high-speed signals (SPI, I2C, UART, I2S) from the DUT.
*   **Integrated Debug Probe:** Onboard **RP2354B** (with 2MB internal flash) running **[Yapicoprobe](https://github.com/rppicomidi/yapicoprobe)** (compatible with standard CMSIS-DAP/Picoprobe). Provides SWD debugging (Step, Pause, Breakpoints) and a UART console.
*   **Hardware Stimulus:** The Debug Probe is hardwired to the DUT's GPIOs via protection resistors, allowing you to inject signals or simulate sensors programmatically.
*   **Integrated USB Hub:** An onboard **GL850G USB Hub** manages all devices via a single USB-C connection. It provides:
    *   Internal link to the Logic Analyzer.
    *   Internal link to the Debug Probe.
    *   **Two auxiliary USB-A ports** (Stacked Connector J10) for connecting the Target Pico or other peripherals.
*   **Robust Power:** A **2A Synchronous Buck Converter (MT2492)** ensures ample power for the Debug Stack and the Target Pico.
*   **Universal Compatibility:** Supports Raspberry Pi Pico, Pico W, Pico 2, and Pico 2W.
*   **Safety First:** 330Ω series resistors protect all GPIO connections between the Analyzer, Probe, and DUT.

## Hardware Architecture

The board consists of three main controllers:

1.  **U1 (Logic Analyzer):** An **RP2354B** dedicated to monitoring DUT GPIOs.
2.  **U2 (Debug Probe):** An **RP2354B** handling SWD communication and Stimulus generation.
3.  **U3 (USB Hub):** A **GL850G** providing connectivity to U1, U2, and the auxiliary ports.

### The Stack Physical Layout
*   **Top:** Socket for DUT (Pico H footprint).
*   **Middle:** The Debug Stack PCB (Analyzer + Probe + Hub).
*   **Bottom:** Male Headers (plugs into your original project).

### Important Note on DUT USB
The **Target Pico's USB data lines (D+/D-)** are **NOT** routed through the stack headers to the Hub automatically.
*   **Debug/Analysis:** Works instantly via the shim headers.
*   **Target USB Data:** If your Target Pico runs code that requires USB connectivity (e.g., TinyUSB), you must use a short USB cable to connect the Target Pico's USB port to one of the **Auxiliary USB-A ports** on the Debug Stack.

## Software & Firmware

To use this board, the onboard RP2354s must be flashed with the following firmware:

*   **Logic Analyzer (U1):** [Download Dr. Gusman's LogicAnalyzer Firmware](https://github.com/gusmanb/logicanalyzer) (Ensure you use the RP2350 build).
*   **Debug Probe (U2):** [Download Yapicoprobe Firmware](https://github.com/rppicomidi/yapicoprobe) (or the official Raspberry Pi `debugprobe` firmware).

## Using the Stimulus Features

One of the most powerful features of this board is the ability to use the **Debug Probe (U2)** to "exercise" the **DUT**. Because the Probe's GPIOs are wired to the DUT's GPIOs (via 330Ω resistors), you can write software for the Probe to simulate external hardware.

### Concept
Instead of physically wiring a button or a sensor to your project to test it, you can tell the Probe to "pull Pin 15 High" or "Send a pulse on Pin 2".

### Example: Writing Stimulus Software
*The following is a MicroPython example. You can run this on the **Debug Probe (U2)** to test the DUT.*

**Scenario:** Your DUT code waits for a button press on `GP15` and then blinks an LED. You want to automate testing this.

1.  Flash **MicroPython** onto the **Debug Probe (U2)** (temporarily replacing the probe firmware).
2.  Run this script on U2:

```python
import machine
import time

# Note: Check the schematic for the specific mapping between Probe GPIO and DUT GPIO
# The mapping is scrambled to optimize PCB routing.
stimulus_pin = machine.Pin(15, machine.Pin.OUT) 

print("Starting Stimulus Test...")

while True:
    print("Simulating Button Press (High)...")
    stimulus_pin.value(1)
    time.sleep(0.5) # Hold for 500ms
    
    print("Simulating Release (Low)...")
    stimulus_pin.value(0)
    time.sleep(2.0) # Wait 2 seconds before next test
```

### Advanced Stimulus (C/C++)
For high-speed protocol simulation (e.g., simulating an SPI sensor sending data to the DUT), you should write a C/C++ program for the Probe using the Pico SDK.

**Skeleton Code for `main.c` on the Probe:**

```c
#include "pico/stdlib.h"
#include "hardware/gpio.h"

// Map of Probe Pins to DUT Pins (Refer to Schematic)
#define STIMULUS_PIN_A  10 
#define STIMULUS_PIN_B  11

void generate_fake_sensor_data() {
    // Bit-bang a protocol or toggle pins
    gpio_put(STIMULUS_PIN_A, 1);
    sleep_us(10);
    gpio_put(STIMULUS_PIN_A, 0);
}

int main() {
    stdio_init_all();
    
    gpio_init(STIMULUS_PIN_A);
    gpio_set_dir(STIMULUS_PIN_A, GPIO_OUT);

    while (1) {
        // Wait for a command from PC, or just run a loop
        generate_fake_sensor_data();
        sleep_ms(100);
    }
}
```
## Kicad Project
This Kicad project uses the CDFER [JLCPCB Library](https://github.com/CDFER/JLCPCB-Kicad-Library).  If you don't install it, the rules checker in EESchema will bark at you.  After installing it, you have to quit Kicad and start again.

## Setup Instructions

1.  **Assembly:** Ensure U1 and U2 are flashed with their respective firmware.
2.  **Connection:** Plug the board into your PC via the USB-C port labeled `USB IN`.
3.  **Drivers:**
    *   **Analyzer:** Open Dr. Gusman's software (or Sigrok PulseView) and connect to the Logic Analyzer device.
    *   **Probe:** In VS Code (or your IDE), select the CMSIS-DAP interface for the Debug Probe.
4.  **Hardware Install:**
    *   Plug the Debug Stack into your project board.
    *   Plug your Pico H (DUT) into the socket on the Debug Stack.

## License

Designed by Rob Frohne, Walla Walla University.
Open Source Hardware.

---
*Note: This design uses RP2354B chips. Ensure your software toolchain supports the RP2350 architecture.*