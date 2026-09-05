# Automatic RGB Light Using LDR (Arduino UNO)

## Description
This project uses an Arduino UNO, an LDR (Light Dependent Resistor) module, and a 4-pin RGB LED to automatically change colors based on ambient light levels. 
The setup works according to the actual analog LDR readings:
- **Very bright / phone flash (0–100)** → 🔴 RED
- **Normal / bright room (101–350)** → 🟢 GREEN
- **Dark (351+)** → 🔵 BLUE

## Hardware Requirements
- Arduino UNO
- LDR Sensor Module (with Analog Out)
- 4-Pin Common-Cathode RGB LED
- 3x 220Ω Resistors
- Jumper Wires

## Connections

### 1. LDR Module
| LDR Module | Arduino UNO |
| :--- | :--- |
| VCC | 5V |
| GND | GND |
| A0 (Analog Out) | A0 |
| D0 (Digital Out) | NOT CONNECTED |

### 2. RGB LED
Assuming your RGB LED is **common cathode**. Use one 220Ω resistor for **EACH** RGB pin (R, G, B) to limit the current and protect the LED.

| RGB LED Pin | Connection | Arduino UNO |
| :--- | :--- | :--- |
| R (Red) | 220Ω Resistor | D9 |
| G (Green) | 220Ω Resistor | D10 |
| B (Blue) | 220Ω Resistor | D11 |
| Common Cathode | Direct | GND |

> **⚠️ Important:** The longest leg on the RGB LED is usually the common pin, but confirm this before connecting. The order of the R, G, and B pins can also vary between different RGB LEDs.

### Wiring Schematic Example
```text
                 ┌── 220Ω ── D9
RGB LED R ───────┤
                 
                 ┌── 220Ω ── D10
RGB LED G ───────┤

                 ┌── 220Ω ── D11
RGB LED B ───────┤

RGB LED COMMON ───────────── GND
```

## Expected Result
| Lighting Condition | LDR Reading | LED Color |
| :--- | :--- | :--- |
| **Phone Flash** | 10–50 | 🔴 RED |
| **Normal Room Light** | ≈ 300 | 🟢 GREEN |
| **Dark Room** | ≈ 400–500 | 🔵 BLUE |

## Code

Copy and paste this completely into your Arduino IDE and upload it to your Arduino UNO.

```cpp
// =====================================================
// AUTOMATIC RGB LIGHT USING LDR
// Arduino UNO + LDR Module + 4-Pin RGB LED
// =====================================================

// ---------------- PIN ASSIGNMENTS ----------------

const int ldrPin = A0;

// RGB LED pins
const int redPin = 9;
const int greenPin = 10;
const int bluePin = 11;


// ---------------- SETUP ----------------

void setup() {

  pinMode(redPin, OUTPUT);
  pinMode(greenPin, OUTPUT);
  pinMode(bluePin, OUTPUT);

  Serial.begin(9600);

  // Start with RGB LED OFF
  analogWrite(redPin, 0);
  analogWrite(greenPin, 0);
  analogWrite(bluePin, 0);
}


// ---------------- MAIN LOOP ----------------

void loop() {

  // Read LDR value
  int ldrValue = analogRead(ldrPin);


  // Turn all RGB colors OFF first
  analogWrite(redPin, 0);
  analogWrite(greenPin, 0);
  analogWrite(bluePin, 0);


  // =================================================
  // VERY BRIGHT → RED
  // LDR: 0 - 100
  // =================================================

  if (ldrValue <= 100) {

    analogWrite(redPin, 255);

  }


  // =================================================
  // NORMAL / BRIGHT ROOM → GREEN
  // LDR: 101 - 350
  // =================================================

  else if (ldrValue <= 350) {

    analogWrite(greenPin, 255);

  }


  // =================================================
  // DARK → BLUE
  // LDR: 351+
  // =================================================

  else {

    analogWrite(bluePin, 255);

  }


  // ---------------- SERIAL MONITOR ----------------

  Serial.print("LDR Reading: ");
  Serial.print(ldrValue);

  Serial.print(" | Color: ");

  if (ldrValue <= 100) {
    Serial.println("RED - Very Bright");
  }

  else if (ldrValue <= 350) {
    Serial.println("GREEN - Normal Light");
  }

  else {
    Serial.println("BLUE - Dark");
  }


  delay(100);
}
```
