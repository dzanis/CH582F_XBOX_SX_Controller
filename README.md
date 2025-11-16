[RU](README_RU.md)

# CH582 as a Host for Xbox Series X Gamepad

This project allows the CH582 microcontroller to act as a BLE Central host, connecting to a Microsoft Xbox Series X gamepad.

The project manages the entire process: BLE scanning, pairing (Bonding), saving the gamepad's MAC address to EEPROM, and receiving HID Input Reports (stick and button data).

## LED Indication

The onboard LED on the CH582 indicates the current connection state:

| State | LED Activity | Meaning & Required Action |
| :--- | :--- | :--- |
| **Scanning** | Slow Blink (1/sec) | The CH582 is scanning for the gamepad. <br> **Action:** Press and hold the "Connect" button on your gamepad. |
| **Pairing** | Fast Blink (multi/sec) | The device is found and the pairing/setup process is active. <br> **Action:** Wait for the process to complete. |
| **Connected** | Solid On | The gamepad is successfully connected, the channel is encrypted, and data is streaming. <br> **Action:** Ready to use! |

---

## How to Connect (Detailed Guide)

### First-Time Connection (Pairing & Saving)

The goal of this step is to "introduce" the CH582 to the gamepad and save its MAC address to the CH582's EEPROM.

1.  Power on the CH582. The LED will start **blinking slowly** (Scanning mode).
2.  Take the Xbox gamepad. Press and hold the **"Connect" button** (small circular button on top) until the "Xbox" button starts blinking rapidly.
3.  The CH582 will discover the gamepad. The LED on the CH582 will switch to a **fast blink** (Pairing mode).
4.  After a few seconds, the UART log will show a `Bond save success` message. The gamepad's MAC address is now saved.
5.  **IMPORTANT:** At this stage, notifications will likely **not** work. The CH582 LED will continue its fast blink, and the gamepad will also continue searching (Xbox button blinking). *This is normal and expected.*
6.  To start receiving data, you must **long-press the "Connect" button on the gamepad again** or power-cycle the Xbox controller (turn it off and back on).

### Subsequent Connections

After you have performed the **First-Time Connection** once, the devices are bonded.

Now, you only need to turn on the gamepad by pressing the "Xbox" button. It will automatically find the CH582, and the LEDs on **both** devices will immediately go **solid on**, skipping the setup process.

---

# How to use the `xbox_sx_controller` Library

## 1. General Logic

1.  **Register:** In `main.c`, we register our callback function (`handle_xbox_data`) with the `central.c` module.
2.  **Callback:** The `central.c` module automatically calls `handle_xbox_data` every time a new data packet (notification) arrives from the gamepad.
3.  **Parse:** Inside `handle_xbox_data`, we use functions from the `xbox_sx_controller` library (like `Xbox_Update_State`) to update our global state structure, `xc_state`.
4.  **Read:** All other logic (like logging) reads data from this prepared `xc_state` structure.

## 2. Usage Example (from `main.c`)

This complete example shows how to integrate the library.

### 2.1. Setup and Initialization (`main.c`)

In `main()`, we must initialize `central.c` and—most importantly—register our `handle_xbox_data` handler using `Central_RegisterGamepadInputCallback`.

```c
#include "xbox_sx_controller.h" // Our parser library

// Global variable to store the controller's state
static XboxControllerState_t xc_state = {0}; 

// Our callback function handler
void handle_xbox_data(uint8_t* ble_data, uint16_t data_len);

int main(void)
{
    // ... (System, UART, BLE Init) ...
    
    // !!! THE MOST IMPORTANT STEP !!!
    // Register our 'handle_xbox_data' function in central.c
    Central_RegisterGamepadInputCallback(handle_xbox_data);
    
    // ...
    Main_Circulation();
}
````

### 2.2. Handling Data (The Callback)

This function is called automatically by `central.c`. Here, we use the library to parse the raw data.

```c
void handle_xbox_data(uint8_t* ble_data, uint16_t data_len) 
{
    // Optional: Check for duplicate data to avoid processing the same packet.
    // NOTE: tmos_memcmp returns 1 if the memory is identical.
    if (tmos_memcmp(&xc_state, ble_data, XBOX_CONTROLLER_TOTAL_PACKAGE_SIZE))
    {
        return; // Skip duplicate packet
    }

    // Call the parser function
    // It takes the raw 'ble_data' and updates our 'xc_state'
    uint8_t error = Xbox_Update_State(&xc_state, ble_data, data_len);

    // Use the parsed data
    if (error == 0) // If parsing was successful
    {
        //  Print to log (example)
        Xbox_Print_State(&xc_state);

        // Read buttons (example)
        if (Xbox_Is_Pressed(&xc_state, SHARE)) // Using the helper function
        {
            PRINT("Share pressed!\n");
        }
    }
    else
    {
        PRINT("Error parsing Xbox data: %u\n", error);
    }
}
```

### 2.3. Reading the State (`xc_state`)

After `Xbox_Update_State` has run, you can access the gamepad's current state at any time from the global `xc_state` variable.

The states of the A, B, X, Y, and View, Menu, Xbox, LS, RS buttons can be obtained by checking the `Xbox_Is_Pressed` method.

```c
// Example: Checking the A button
if (Xbox_Is_Pressed(&xc_state, A)) {
    // A button is pressed
}

// ...or checking the bitmask directly:
if (xc_state.buttonsABXY & MASK_A) {
    // A button is pressed
}

// Example: Reading the Left Stick X-axis
uint16_t lx = xc_state.leftX;
if (lx < 30000) {
    // Stick is tilted left
}

// Example: Reading the D-Pad
if (xc_state.dpad == DPAD_UP) { // 1 = Up
    // D-Pad Up is pressed
}
```
