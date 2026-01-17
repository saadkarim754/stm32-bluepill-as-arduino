# STM32 Blue Pill: From "Clone Hell" to Arduino USB Success

## 1. Project Overview
**Objective:** To set up a cheap STM32F103C8T6 "Blue Pill" development board to work with the Arduino IDE, allowing for **direct USB uploads** and **Serial Monitor debugging**, completely bypassing the need for an external programmer after the initial setup.

**The Challenge:**
Most cheap Blue Pills and ST-Link dongles are "clones" (copies). They often suffer from:
* Incorrect pinout labels.
* Incompatibility with old ST software.
* Driver signature issues on Windows 10/11.
* "Bricked" states where code locks out the debug pins.

This guide documents the exact steps to overcome these hardware/software lies and establish a robust workflow.

---

## 2. Protocols & Concepts Explained
Before starting, understand the "languages" we are using:

* **SWD (Serial Wire Debug):** A 2-wire protocol (Clock + Data) used by the ST-Link dongle. It talks directly to the chip's hardware debugging unit. It is used **ONLY ONCE** in this guide to flash the bootloader or unbrick a dead board.
* **USB DFU (Device Firmware Upgrade):** A protocol that allows a device to update its own firmware over USB. The bootloader we install speaks this language, allowing the Arduino IDE to upload code.
* **USB CDC (Communications Device Class):** A protocol that allows the USB port to pretend to be a Virtual Serial Port (COM Port). We enable this in software so we can use `Serial.println()` for debugging.

---

## 3. Prerequisites

### Hardware
* **STM32 "Blue Pill" Board** (STM32F103C8T6 or clone CKS32/CS32).
* **ST-Link V2 USB Dongle** (Clone).
* **4x Female-Female Jumper Wires.**
* **Micro-USB Cable.**

### Software
1.  **[STM32CubeProgrammer](https://www.st.com/en/development-tools/stm32cubeprog.html):** The modern tool for flashing. *Do not use the old "ST-Link Utility".*
2.  **[Arduino IDE](https://www.arduino.cc/en/software):** The development environment.
3.  **[STM32duino Bootloader Binary](https://github.com/rogerclarkmelbourne/STM32duino-bootloader/releases):** Download `generic_boot20_pc13.bin`.
4.  **[Arduino_STM32 Drivers](https://github.com/rogerclarkmelbourne/Arduino_STM32/tree/master/drivers):** Specifically the `drivers/win` folder.

---

## 4. Phase 1: The Hardware "Truth" (Connecting ST-Link)
*Common Trap: Clone dongles often have incorrect pin labels on the metal shell.*

### 4.1 Wiring
Remove the metal shell of the ST-Link dongle to verify pinout on the PCB if connection fails.
**Wiring Table:**

| ST-Link Dongle | Blue Pill Board | Notes |
| :--- | :--- | :--- |
| **GND** | **G** (GND) | Common Ground is critical. |
| **SWCLK** | **CLK** (PA14) | Clock Signal. |
| **SWDIO** | **IO** (PA13) | Data Signal. |
| **3.3V** | **3.3** | **CRITICAL:** If connection is unstable, disconnect this and power the Blue Pill via its own USB port (phone charger) instead. |
| **RST** | **R** (Reset) | Required for "Connect Under Reset" mode. |

---

## 5. Phase 2: Unbricking & Flashing the Bootloader
*Goal: Use the ST-Link to install the "STM32duino" bootloader, which teaches the chip how to speak USB.*

### 5.1 Configure STM32CubeProgrammer
Clone chips/dongles require specific "slow and forceful" settings.
1.  **Port:** SWD
2.  **Frequency:** `480 kHz` (Low speed is mandatory for clones/loose wires).
3.  **Mode:** `Under Reset` (Requires RST wire connected).
4.  **Reset Mode:** `Hardware Reset`.
5.  Click **Connect**.

* **Troubleshooting:** If it fails, move the Blue Pill **BOOT0 Jumper** to **1** (High), cycle power, and try again. This forces "System Memory" boot.

### 5.2 Flash the Bootloader
1.  In CubeProgrammer, go to the **"Open File"** tab.
2.  Select `generic_boot20_pc13.bin`.
3.  Ensure address is `0x08000000`.
4.  Click **Download**.
5.  **Disconnect the ST-Link.** You are done with it!

---

## 6. Phase 3: The Windows Driver Battle
*Goal: Make Windows recognize the board as "Maple DFU" so Arduino IDE can talk to it.*

### 6.1 Disabling Driver Signature Enforcement
Windows 10/11 blocks the old Maple drivers by default.
1.  **Settings** > Update & Security > Recovery > **Advanced Startup (Restart Now)**.
2.  Troubleshoot > Advanced Options > Startup Settings > Restart.
3.  Press **7** (Disable driver signature enforcement).

### 6.2 Installing the Driver
1.  Plug in the Blue Pill via USB. It will appear as `Maple 003` (with a warning).
2.  Open **Device Manager**.
3.  Right-click `Maple 003` > **Update Driver** > **Browse my computer**.
4.  Select the `drivers/win` folder downloaded from the Arduino_STM32 repo.
5.  **Accept the Red Warning** ("Install this driver software anyway").
6.  Success: Device should now list as **"Maple DFU"**.

---

## 7. Phase 4: Arduino IDE Configuration
*Goal: Configure the IDE to upload via USB and enable Serial debugging.*

### 7.1 Board Manager
1.  Install the **"STM32F1xx / STM32duino"** core via Board Manager.
    * *(Note: Using the Roger Clark core or official STM core works, but settings below are for the Roger Clark/Maple method used).*

### 7.2 Tool Settings (CRITICAL)
Set your `Tools` menu exactly like this:
* **Board:** `Generic STM32F1 series` (or `BluePill F103C8`).
* **Upload Method:** `Maple DFU Bootloader 2.0` (Matches the bin file we flashed).
* **CPU Speed:** `72MHz (Normal)`.
* **USB Support:** `CDC (generic 'Serial' supersede U(S)ART)`.
    * *Why? This enables the Virtual COM Port for Serial Monitor.*

---

## 8. Phase 5: The Code
Upload this sketch to test functionality.

```cpp
// Blue Pill LED is usually on PC13
#define LED_PIN PC13

void setup() {
  // Initialize the specific LED pin
  pinMode(LED_PIN, OUTPUT);
  
  // Start Serial (Baud rate is ignored for USB CDC, but 115200 is standard)
  Serial.begin(115200);
}

void loop() {
  // Write to Serial Monitor
  Serial.println("System Alive: USB Serial Working!");
  
  // Blink LED (Note: LOW is usually ON for Blue Pill)
  digitalWrite(LED_PIN, LOW); 
  delay(100); 
  digitalWrite(LED_PIN, HIGH);
  delay(500);
}
