# 🚀 Bootstrap 2026–27 — Arduino & ESP32 Projects

> A growing collection of hands-on embedded systems projects built during **Bootstrap 2026–27**.  
> Each project includes full wiring diagrams, pin mappings, library requirements, and complete source code.

---

## 📦 Projects

### 🎮 Games

| Project | Hardware | Description |
| :--- | :--- | :--- |
| [🏹 ESP32 TFT Hunter Game](ESP32_TFT_Hunter_Game.md) | ESP32 + 1.8" TFT + Joystick + Button | A top-down hunting game — move a character, shoot prey, dodge trees |
| [🏎️ ESP32 City Racer Game](ESP32_City_Racer_Game_README.md) | ESP32 + 1.8" TFT + Joystick + Button | Endless top-down car dodging game with city scenery and increasing speed |
| [🌈 ESP32 Color Memory Game](ESP32_Color_Memory_Game_README.md) | ESP32 + OLED + RGB LED + 4 Buttons | Remember and repeat a flashing color sequence across 10 levels |
| [⚡ RGB Reaction Game](Reaction_Game_README.md) | Arduino UNO + OLED + RGB LED + 4 Buttons | React to a random color as fast as you can — measures your reaction time in ms |
| [🏃 Salmon Bai Run](Salmon_Bai_Run_Game_README.md) | Arduino UNO + OLED + IR Sensor | Hands-free endless runner — wave your hand over the IR sensor to jump |
| [🐾 DigiDog Virtual Pet](DigiDog_Virtual_Pet_README.md) | Arduino UNO + SPI OLED + 4 Buttons | A Tamagotchi-style virtual dog — feed, pet, play, and sleep to keep it happy |

---

### 🔬 Sensors & Hardware

| Project | Hardware | Description |
| :--- | :--- | :--- |
| [💡 LDR RGB Auto Light](Arduino_UNO_LDR_RGB_LED_README.md) | Arduino UNO + LDR Module + RGB LED | Automatically changes LED color based on ambient light levels |
| [📡 Ultrasonic Distance Radar](Ultrasonic_Distance_Radar_README.md) | Arduino UNO + OLED + HC-SR04 + Buzzer | Visualizes object distance on an OLED and beeps on danger |
| [🔐 RFID Smart Access Control](RFID_Smart_Access_Control_README.md) | Arduino UNO + MFRC522 + Servo | Scans an RFID card and unlocks a servo motor door if authorized |

---

### 🎵 Audio

| Project | Hardware | Description |
| :--- | :--- | :--- |
| [🎹 Electronic Piano](Electronic_Piano_README.md) | Arduino UNO + Buzzer + 4 Buttons | Press buttons to play musical notes through a buzzer |

---

## 🛠️ Common Libraries Used

| Library | Used In |
| :--- | :--- |
| `Adafruit GFX` | TFT & OLED display projects |
| `Adafruit ST7735` | ESP32 TFT projects |
| `Adafruit SSD1306` | OLED projects |
| `MFRC522` | RFID Access Control |
| `Servo` | RFID Door Lock |
| `SPI` / `Wire` | SPI & I2C communication |

---

## ⚡ Quick Start

1. Install the **Arduino IDE** and the required libraries listed in each project's README.
2. Select the correct board (**Arduino UNO** or **ESP32**) under `Tools → Board`.
3. Wire up the components according to the connection tables in each README.
4. Copy the code into the Arduino IDE, compile, and upload!

---

## 📁 Repository Structure

```
Bootstrap-2026-27/
├── README.md                            ← You are here
├── ESP32_TFT_Hunter_Game.md
├── ESP32_City_Racer_Game_README.md
├── ESP32_Color_Memory_Game_README.md
├── Reaction_Game_README.md
├── Salmon_Bai_Run_Game_README.md
├── DigiDog_Virtual_Pet_README.md
├── Arduino_UNO_LDR_RGB_LED_README.md
├── Ultrasonic_Distance_Radar_README.md
├── RFID_Smart_Access_Control_README.md
└── Electronic_Piano_README.md
```

---

*Made with ❤️ for Bootstrap 2026–27*
