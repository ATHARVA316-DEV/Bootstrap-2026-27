# Electronic Piano / Mini Theremin (Arduino UNO)

## Description
A simple electronic piano built with an Arduino UNO, a buzzer module, and 4 push buttons. 
Pressing each button plays a different musical note through the buzzer.

## Hardware Requirements
- Arduino UNO (or compatible board)
- Buzzer Module (with VCC, GND, and I/O pins)
- 4x Push Buttons
- Jumper Wires

## Connections

### 1. Buzzer Module
| Buzzer Pin | Arduino UNO |
| :--- | :--- |
| VCC | 5V |
| GND | GND |
| IO (Signal) | D8 |

### 2. Piano Buttons
Connect one side of each button to the respective Arduino digital pin, and the other side directly to **GND**. The Arduino will use internal pull-up resistors (`INPUT_PULLUP`), so no external resistors are required.

| Component | Pin / Leg | Arduino UNO |
| :--- | :--- | :--- |
| **Button 1** | One side | D2 |
| | Other side | GND |
| **Button 2** | One side | D3 |
| | Other side | GND |
| **Button 3** | One side | D4 |
| | Other side | GND |
| **Button 4** | One side | D5 |
| | Other side | GND |

## Code

*(The Arduino code for this project will be added here later...)*
