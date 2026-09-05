# Ultrasonic Distance Radar (Arduino + OLED)

## Description
This project acts as a mini radar system using an Arduino, an HC-SR04 ultrasonic sensor, an I2C OLED display, and a buzzer. 
The OLED screen visually graphs the distance of an object relative to the sensor up to 100cm. If an object gets too close (less than or equal to 5cm), the screen will display a "DANGER" warning and the buzzer will sound an alarm!

## Hardware Requirements
- Arduino UNO (or any compatible board)
- I2C OLED Display (128x64 resolution)
- HC-SR04 Ultrasonic Sensor Module
- Buzzer
- Jumper Wires

## Software Dependencies
Install the following libraries in the Arduino IDE before compiling:
- **Adafruit GFX Library**
- **Adafruit SSD1306**

*(Note: The `Wire` library for I2C communication is built-in).*

## Connections

### 1. I2C OLED Display
| OLED Pin | Arduino UNO |
| :--- | :--- |
| VCC | 5V (or 3.3V depending on your module) |
| GND | GND |
| SDA | A4 |
| SCL | A5 |

### 2. HC-SR04 Ultrasonic Sensor
| HC-SR04 Pin | Arduino UNO | Notes |
| :--- | :--- | :--- |
| VCC | 5V | Power |
| GND | GND | Ground |
| TRIG | D9 | Defined as `TRIG_PIN` in the code |
| ECHO | D10 | Defined as `ECHO_PIN` in the code |

### 3. Buzzer
| Buzzer Pin | Arduino UNO | Notes |
| :--- | :--- | :--- |
| Positive (+) | D8 | Defined as `BUZZER_PIN` in the code |
| Negative (-) | GND | Ground |

## Code

Copy and paste this completely into your Arduino IDE and upload it to your Arduino.

```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64

#define OLED_RESET -1
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Pins
#define TRIG_PIN 9
#define ECHO_PIN 10
#define BUZZER_PIN 8

// Settings
#define MAX_DISTANCE 100
#define DANGER_DISTANCE 5


// =================================================
// GET ULTRASONIC DISTANCE
// =================================================

float getDistance() {

  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);

  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);

  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH, 30000);

  // No echo
  if (duration == 0) {
    return MAX_DISTANCE;
  }

  float distance = duration * 0.0343 / 2.0;

  // Limit distance
  if (distance < 2) {
    distance = 2;
  }

  if (distance > MAX_DISTANCE) {
    distance = MAX_DISTANCE;
  }

  return distance;
}


// =================================================
// DRAW DISPLAY
// =================================================

void drawRadar(float distance) {

  display.clearDisplay();

  display.setTextColor(SSD1306_WHITE);

  // -----------------------------------------------
  // TITLE
  // -----------------------------------------------

  display.setTextSize(1);

  display.setCursor(35, 0);
  display.print("DISTANCE RADAR");


  // -----------------------------------------------
  // SENSOR
  // -----------------------------------------------

  int sensorX = 64;
  int sensorY = 57;

  // Sensor
  display.fillCircle(sensorX, sensorY, 3, SSD1306_WHITE);


  // -----------------------------------------------
  // DETECTION BEAM
  // -----------------------------------------------

  display.drawLine(
    sensorX,
    sensorY,
    sensorX,
    15,
    SSD1306_WHITE
  );


  // -----------------------------------------------
  // DISTANCE SCALE
  // -----------------------------------------------

  display.drawLine(55, 47, 73, 47, SSD1306_WHITE);
  display.drawLine(55, 37, 73, 37, SSD1306_WHITE);
  display.drawLine(55, 27, 73, 27, SSD1306_WHITE);
  display.drawLine(55, 17, 73, 17, SSD1306_WHITE);


  // -----------------------------------------------
  // OBJECT POSITION
  // -----------------------------------------------

  if (distance <= MAX_DISTANCE) {

    /*
       FAR  = TOP
       NEAR = BOTTOM

       100cm -> y = 15
       50cm  -> y ≈ 36
       20cm  -> y ≈ 49
       5cm   -> y ≈ 55
    */

    int objectY = map(
      (int)distance,
      5,
      MAX_DISTANCE,
      53,
      15
    );

    objectY = constrain(objectY, 15, 53);

    // Draw object
    display.fillCircle(
      sensorX,
      objectY,
      4,
      SSD1306_WHITE
    );
  }


  // -----------------------------------------------
  // DISTANCE TEXT
  // -----------------------------------------------

  display.setCursor(2, 54);

  display.print("D:");

  if (distance >= MAX_DISTANCE) {

    display.print(">100");

  } else {

    display.print(distance, 1);
  }

  display.print("cm");


  // -----------------------------------------------
  // STATUS
  // -----------------------------------------------

  display.setCursor(85, 54);

  if (distance <= DANGER_DISTANCE) {

    display.print("DANGER");

  } else {

    display.print("CLEAR");
  }


  display.display();
}


// =================================================
// BUZZER
// =================================================

void controlBuzzer(float distance) {

  /*
     IMPORTANT:

     <= 5 cm  = BUZZER ON
     > 5 cm   = BUZZER OFF
  */

  if ( DANGER_DISTANCE <= distance ) {

    tone(BUZZER_PIN, 2500);

  } else {

    noTone(BUZZER_PIN);
  }
}


// =================================================
// SETUP
// =================================================

void setup() {

  Serial.begin(9600);

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  // Make sure buzzer starts OFF
  noTone(BUZZER_PIN);


  // OLED
  if (!display.begin(
        SSD1306_SWITCHCAPVCC,
        OLED_ADDR
      )) {

    Serial.println("OLED ERROR!");

    while (1);
  }


  // Startup
  display.clearDisplay();

  display.setTextColor(SSD1306_WHITE);

  display.setTextSize(2);

  display.setCursor(25, 20);

  display.print("RADAR");

  display.display();

  delay(1500);
}


// =================================================
// LOOP
// =================================================

void loop() {

  float distance = getDistance();


  // Serial monitor
  Serial.print("Distance = ");
  Serial.print(distance);
  Serial.println(" cm");


  // OLED
  drawRadar(distance);


  // Buzzer
  controlBuzzer(distance);


  delay(100);
}
```
