# RFID Smart Access Control (Arduino UNO)

## Description
This project implements a smart access control system using an Arduino UNO, an MFRC522 RFID reader, and a servo motor to act as a door lock. 
When an authorized RFID card or tag is scanned, the servo motor turns 90 degrees to "unlock" the door, waits for 5 seconds, and then automatically returns to the "locked" position (0 degrees). Unauthorized cards will be denied access, leaving the door locked.

## Hardware Requirements
- Arduino UNO
- MFRC522 RFID Reader Module
- Servo Motor (e.g., SG90 micro servo)
- RFID Cards or Key fobs (13.56MHz)
- Jumper Wires

## Software Dependencies
Install the following library in the Arduino IDE before compiling:
- **MFRC522** by GithubCommunity

*(Note: The `SPI` and `Servo` libraries are built-in and do not need to be installed manually).*

## Connections

### 1. MFRC522 RFID Module
This module uses the SPI communication protocol. 

| MFRC522 Pin | Arduino UNO | Notes |
| :--- | :--- | :--- |
| SDA (SS) | D10 | Defined as `SS_PIN` in code |
| SCK | D13 | SPI Clock |
| MOSI | D11 | SPI Master Out Slave In |
| MISO | D12 | SPI Master In Slave Out |
| IRQ | - | **NOT CONNECTED** |
| GND | GND | Ground |
| RST | D9 | Defined as `RST_PIN` in code |
| 3.3V | 3.3V | **Do not connect to 5V!** |

> **⚠️ Important:** The MFRC522 module strictly operates on **3.3V**. Connecting its power pin to the Arduino's 5V pin can permanently damage the module.

### 2. Servo Motor
| Servo Wire Color | Function | Arduino UNO |
| :--- | :--- | :--- |
| Red | Power | 5V |
| Brown / Black | Ground | GND |
| Orange / Yellow | Signal | D6 |

## How to Authorize Your Own Card
To grant access to your specific card, you need to find its unique UID.
1. Upload the provided code to your Arduino.
2. Open the **Serial Monitor** (set baud rate to `9600`).
3. Scan your card on the RFID reader.
4. The Serial Monitor will output your card's UID, for example: `Card UID: 1A 2B 3C 4D`.
5. Update the `authorizedUID` array at the top of the code to match your card's UID:
   ```cpp
   byte authorizedUID[] = {0x1A, 0x2B, 0x3C, 0x4D};
   ```
6. Re-upload the updated code to the Arduino.

## Code

Copy and paste this completely into your Arduino IDE and upload it to your Arduino UNO.

```cpp
#include <SPI.h>
#include <MFRC522.h>
#include <Servo.h>

#define SS_PIN 10
#define RST_PIN 9
#define SERVO_PIN 6

MFRC522 rfid(SS_PIN, RST_PIN);
Servo doorServo;

// YOUR RFID CARD UID
byte authorizedUID[] = {0x91, 0x69, 0xAC, 0x6D};

void setup() {
  Serial.begin(9600);

  SPI.begin();
  rfid.PCD_Init();

  doorServo.attach(SERVO_PIN);

  // Start in locked position
  doorServo.write(0);

  Serial.println("RFID SMART ACCESS CONTROL");
  Serial.println("Door is LOCKED");
  Serial.println("Scan your card...");
}

void loop() {

  // Check if card is present
  if (!rfid.PICC_IsNewCardPresent()) {
    return;
  }

  // Read card
  if (!rfid.PICC_ReadCardSerial()) {
    return;
  }

  Serial.print("Card UID: ");

  for (byte i = 0; i < rfid.uid.size; i++) {
    if (rfid.uid.uidByte[i] < 0x10) {
      Serial.print("0");
    }

    Serial.print(rfid.uid.uidByte[i], HEX);
    Serial.print(" ");
  }

  Serial.println();

  // Check whether card is authorized
  if (checkUID()) {

    Serial.println("ACCESS GRANTED");
    Serial.println("Door UNLOCKED");

    // Move servo to unlock
    doorServo.write(90);

    delay(5000);

    // Move servo back to lock
    doorServo.write(0);

    Serial.println("Door LOCKED");
    Serial.println("Scan your card again...");

  } else {

    Serial.println("ACCESS DENIED");
    Serial.println("Unauthorized card!");
  }

  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();

  delay(1000);
}


// Function to check UID
bool checkUID() {

  if (rfid.uid.size != 4) {
    return false;
  }

  for (byte i = 0; i < 4; i++) {

    if (rfid.uid.uidByte[i] != authorizedUID[i]) {
      return false;
    }
  }

  return true;
}
```
