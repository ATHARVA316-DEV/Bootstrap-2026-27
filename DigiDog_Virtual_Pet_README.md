# DigiDog Virtual Pet (Arduino + OLED)

## Description
This project implements a cute, interactive "DigiDog" virtual pet on an Arduino using an SPI OLED display! The virtual pet has three main stats: **Hunger (H)**, **Energy (E)**, and **Happiness (:)**. 

You can interact with your DigiDog using four push buttons to feed, pet, play with, or let it sleep. The pet features different animations, facial expressions, and status messages based on how well you take care of it over time. Make sure you don't let it get too hungry or too tired!

## Hardware Requirements
- Arduino UNO (or any compatible board)
- SPI OLED Display (128x64 resolution)
- 4x Push Buttons
- Jumper Wires

## Software Dependencies
Install the following libraries in the Arduino IDE before compiling:
- **Adafruit GFX Library**
- **Adafruit SSD1306**

*(Note: The `SPI` library for hardware SPI communication is built-in).*

## Connections

### 1. SPI OLED Display
| OLED Pin | Arduino UNO | Notes |
| :--- | :--- | :--- |
| VCC | 5V (or 3.3V) | Check your specific module's voltage requirements |
| GND | GND | Ground |
| D1 / MOSI / SDA | D11 | Hardware SPI MOSI |
| D0 / CLK / SCK | D13 | Hardware SPI Clock |
| DC / A0 | D9 | Data/Command |
| CS | D10 | Chip Select |
| RES / RST | D8 | Reset |

### 2. Action Buttons
Connect one side of each button to the respective Arduino pin, and the other side directly to **GND**. The code uses internal pull-up resistors (`INPUT_PULLUP`), so no external pull-up resistors are needed.

| Action | Button Name in Code | Arduino UNO |
| :--- | :--- | :--- |
| **FEED** | FOOD_BUTTON | D2 |
| **PET** | PET_BUTTON | D3 |
| **PLAY** | PLAY_BUTTON | D4 |
| **SLEEP** | SLEEP_BUTTON | D5 |

## Code

Copy and paste this completely into your Arduino IDE and upload it to your Arduino.

```cpp
#include <SPI.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// =================================================
// OLED
// =================================================

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64

#define OLED_MOSI 11
#define OLED_CLK  13
#define OLED_DC   9
#define OLED_CS   10
#define OLED_RESET 8

Adafruit_SSD1306 display(
  SCREEN_WIDTH,
  SCREEN_HEIGHT,
  &SPI,
  OLED_DC,
  OLED_RESET,
  OLED_CS
);


// =================================================
// BUTTONS
// (Buzzer removed entirely - no sound hardware/pins used)
// =================================================

#define FOOD_BUTTON   2
#define PET_BUTTON    3
#define PLAY_BUTTON   4
#define SLEEP_BUTTON  5


// =================================================
// BUTTON DEBOUNCE (edge-triggered - fires once per press)
// =================================================

struct Button {
  uint8_t pin;
  bool lastReading;
  bool stableState;
  unsigned long lastDebounceTime;
};

Button btnFood  = { FOOD_BUTTON,  HIGH, HIGH, 0 };
Button btnPet   = { PET_BUTTON,   HIGH, HIGH, 0 };
Button btnPlay  = { PLAY_BUTTON,  HIGH, HIGH, 0 };
Button btnSleep = { SLEEP_BUTTON, HIGH, HIGH, 0 };

const unsigned long DEBOUNCE_MS = 50;

// Returns true exactly once, on the transition from released -> pressed.
bool buttonPressed(Button &b) {

  bool reading = digitalRead(b.pin);

  if (reading != b.lastReading) {
    b.lastDebounceTime = millis();
  }

  bool pressedEdge = false;

  if ((millis() - b.lastDebounceTime) > DEBOUNCE_MS) {

    if (reading != b.stableState) {

      b.stableState = reading;

      if (b.stableState == LOW) {
        pressedEdge = true;
      }
    }
  }

  b.lastReading = reading;

  return pressedEdge;
}


// =================================================
// DOG STATS
// =================================================

int hunger = 40;
int happiness = 70;
int energy = 70;


// =================================================
// TIMERS
// =================================================

unsigned long lastStats = 0;
unsigned long messageUntil = 0;


// =================================================
// DOG STATE
// =================================================

String message = "";

bool actionAnimation = false;

unsigned long animationStart = 0;


// =================================================
// SETUP
// =================================================

void setup() {

  pinMode(FOOD_BUTTON, INPUT_PULLUP);
  pinMode(PET_BUTTON, INPUT_PULLUP);
  pinMode(PLAY_BUTTON, INPUT_PULLUP);
  pinMode(SLEEP_BUTTON, INPUT_PULLUP);


  // OLED
  if (!display.begin(SSD1306_SWITCHCAPVCC)) {

    while (true) {
      // Display failed to init - halt here.
    }
  }


  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);


  // Startup
  display.setTextSize(2);
  display.setCursor(27, 10);
  display.println("DigiDog");

  display.setTextSize(1);
  display.setCursor(40, 35);
  display.println("Woof! <3");

  display.display();

  delay(1200);

  randomSeed(analogRead(A0));

  lastStats = millis();
}


// =================================================
// LOOP
// =================================================

void loop() {

  checkButtons();

  updateStats();

  drawDog();
}


// =================================================
// BUTTON HANDLING
// =================================================

void checkButtons() {

  if (buttonPressed(btnFood)) {
    feedDog();
  }

  if (buttonPressed(btnPet)) {
    petDog();
  }

  if (buttonPressed(btnPlay)) {
    playDog();
  }

  if (buttonPressed(btnSleep)) {
    sleepDog();
  }
}


// =================================================
// FEED
// =================================================

void feedDog() {

  hunger -= 30;

  if (hunger < 0)
    hunger = 0;


  happiness += 5;

  if (happiness > 100)
    happiness = 100;


  energy -= 5;

  if (energy < 0)
    energy = 0;


  message = "YUMMY! <3";

  messageUntil = millis() + 1200;

  startAnimation();
}


// =================================================
// PET
// =================================================

void petDog() {

  happiness += 20;

  if (happiness > 100)
    happiness = 100;


  message = "HEHE! <3";

  messageUntil = millis() + 1200;

  startAnimation();
}


// =================================================
// PLAY
// =================================================

void playDog() {

  if (energy < 15) {

    message = "TOO TIRED...";

    messageUntil = millis() + 1200;

    return;
  }


  happiness += 25;
  energy -= 20;
  hunger += 10;


  if (happiness > 100)
    happiness = 100;

  if (energy < 0)
    energy = 0;

  if (hunger > 100)
    hunger = 100;


  message = "YAY! BALL!";

  messageUntil = millis() + 1200;

  startAnimation();
}


// =================================================
// SLEEP
// =================================================

void sleepDog() {

  energy += 30;

  if (energy > 100)
    energy = 100;


  hunger += 5;

  if (hunger > 100)
    hunger = 100;


  message = "Z z z...";

  messageUntil = millis() + 1500;

  startAnimation();
}


// =================================================
// ANIMATION
// =================================================

void startAnimation() {

  actionAnimation = true;

  animationStart = millis();
}


// =================================================
// UPDATE STATS
// =================================================

void updateStats() {

  if (millis() - lastStats >= 5000) {

    hunger += 2;

    energy -= 1;

    happiness -= 1;


    if (hunger > 100)
      hunger = 100;

    if (energy < 0)
      energy = 0;

    if (happiness < 0)
      happiness = 0;


    lastStats = millis();
  }
}


// =================================================
// DRAW DOG
// =================================================

void drawDog() {

  display.clearDisplay();


  // =================================================
  // HEADER
  // =================================================

  display.setTextSize(1);

  display.setCursor(2, 0);
  display.print("DigiDog");


  // Heart indicator
  display.setCursor(103, 0);

  if (happiness > 70)
    display.print("<3");
  else
    display.print(":(");


  // =================================================
  // DOG
  // =================================================

  drawCuteDog();


  // =================================================
  // MESSAGE
  // =================================================

  display.setTextSize(1);

  display.setCursor(25, 48);


  if (millis() < messageUntil) {

    display.print(message);

  }

  else {

    if (hunger >= 80) {

      display.print("FEED ME! :(");

    }

    else if (energy <= 20) {

      display.print("Sleepy... zzz");

    }

    else if (happiness <= 25) {

      display.print("I'm sad...");

    }

    else if (happiness >= 80) {

      display.print("I LOVE YOU!");

    }

    else {

      display.print("Woof! Woof!");
    }
  }


  // =================================================
  // STATS
  // =================================================

  display.setCursor(2, 58);

  display.print("H:");
  display.print(hunger);

  display.print(" E:");
  display.print(energy);

  display.print(" :)");


  display.display();
}


// =================================================
// CUTE DOG DRAWING (v3 - chibi sitting puppy, new design)
// =================================================

void drawCuteDog() {

  bool animating = actionAnimation &&
                   millis() - animationStart < 1200;

  if (!animating) {
    actionAnimation = false;
  }

  int wagOffset = animating ? 7 : (int)(2 * sin(millis() / 220.0));


  // =================================================
  // TAIL (curls up behind the body, wags)
  // =================================================

  display.drawLine(88, 40, 98, 32 - wagOffset, SSD1306_WHITE);
  display.drawLine(98, 32 - wagOffset, 96, 24 - wagOffset, SSD1306_WHITE);
  display.drawLine(96, 24 - wagOffset, 90, 22 - wagOffset, SSD1306_WHITE);


  // =================================================
  // BODY (round, sitting)
  // =================================================

  display.fillRoundRect(46, 34, 36, 18, 8, SSD1306_WHITE);


  // =================================================
  // PAWS (little feet poking out front)
  // =================================================

  display.fillCircle(54, 51, 4, SSD1306_WHITE);
  display.fillCircle(74, 51, 4, SSD1306_WHITE);


  // =================================================
  // HEAD (big round chibi head, perked on top of body)
  // =================================================

  display.fillCircle(64, 24, 17, SSD1306_WHITE);


  // =================================================
  // EARS (perked / pointed, not floppy - new style)
  // =================================================

  // Left ear
  display.fillTriangle(
    42, 16,
    49, 2,
    56, 14,
    SSD1306_WHITE
  );

  // Right ear
  display.fillTriangle(
    72, 14,
    79, 2,
    86, 16,
    SSD1306_WHITE
  );

  // Inner ear shading
  display.fillTriangle(
    46, 13,
    49, 6,
    53, 12,
    SSD1306_BLACK
  );

  display.fillTriangle(
    75, 12,
    79, 6,
    82, 13,
    SSD1306_BLACK
  );


  // =================================================
  // EYES (big and round)
  // =================================================

  if (energy <= 20) {

    // sleepy eyes
    display.drawLine(53, 24, 61, 24, SSD1306_BLACK);
    display.drawLine(67, 24, 75, 24, SSD1306_BLACK);

  } else {

    display.fillCircle(57, 23, 4, SSD1306_BLACK);
    display.fillCircle(71, 23, 4, SSD1306_BLACK);

    display.fillCircle(58, 21, 1, SSD1306_WHITE);
    display.fillCircle(72, 21, 1, SSD1306_WHITE);
  }


  // Happy eyebrow arcs
  if (happiness >= 80 && energy > 20) {

    display.drawLine(52, 15, 58, 13, SSD1306_BLACK);
    display.drawLine(70, 13, 76, 15, SSD1306_BLACK);
  }


  // =================================================
  // SNOUT (rounded patch lower on the face)
  // =================================================

  display.fillCircle(64, 32, 8, SSD1306_WHITE);
  display.drawCircle(64, 32, 8, SSD1306_BLACK);


  // Nose
  display.fillCircle(64, 30, 3, SSD1306_BLACK);


  // Mouth
  display.drawLine(64, 33, 60, 36, SSD1306_BLACK);
  display.drawLine(64, 33, 68, 36, SSD1306_BLACK);


  // Tongue while giggling
  if (animating && message == "HEHE! <3") {
    display.fillRoundRect(61, 35, 6, 5, 2, SSD1306_BLACK);
  }


  // =================================================
  // BLUSH
  // =================================================

  display.drawCircle(49, 29, 2, SSD1306_BLACK);
  display.drawCircle(79, 29, 2, SSD1306_BLACK);


  // =================================================
  // PLAY SPARKLES
  // =================================================

  if (animating && message == "YAY! BALL!") {

    display.drawPixel(30, 18, SSD1306_WHITE);
    display.drawPixel(98, 16, SSD1306_WHITE);
    display.drawPixel(102, 24, SSD1306_WHITE);
    display.drawPixel(26, 26, SSD1306_WHITE);
  }
}
```
