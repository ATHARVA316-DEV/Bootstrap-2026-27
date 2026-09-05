# RGB Reaction Game (Arduino + OLED)

## Description
Test your reflexes! This project is a fast-paced reaction game using an Arduino, an I2C OLED display, an RGB LED, and 4 push buttons. 

The OLED screen will tell you to get ready. After a random delay, the RGB LED will light up in a random color (Red, Green, or Blue). You must press the corresponding colored button as fast as you can! The game records your reaction time in milliseconds over 10 rounds, keeping track of your best time and calculating your average reaction speed.

## Hardware Requirements
- Arduino UNO (or any compatible board)
- I2C OLED Display (128x64 resolution)
- 4-Pin Common-Cathode RGB LED
- 4x Push Buttons (Start, Red, Green, Blue)
- 3x 220Ω Resistors (for the RGB LED)
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

### 2. Action Buttons
Connect one side of each button to the respective Arduino pin, and the other side directly to **GND**. The code uses internal pull-up resistors (`INPUT_PULLUP`), so no external pull-up resistors are needed.

| Button Function | Arduino UNO | Notes |
| :--- | :--- | :--- |
| **START / PAUSE** | D2 | Defined as `START_BUTTON` in code |
| **RED** | D3 | Defined as `RED_BUTTON` in code |
| **GREEN** | D4 | Defined as `GREEN_BUTTON` in code |
| **BLUE** | D5 | Defined as `BLUE_BUTTON` in code |

### 3. RGB LED
Assuming your RGB LED is **common cathode**. Use one 220Ω resistor for **EACH** RGB pin (R, G, B) to limit the current and protect the LED.

| RGB LED Pin | Connection | Arduino UNO |
| :--- | :--- | :--- |
| R (Red) | 220Ω Resistor | D9 |
| G (Green) | 220Ω Resistor | D10 |
| B (Blue) | 220Ω Resistor | D11 |
| Common Cathode | Direct | GND |

## Code

Copy and paste this completely into your Arduino IDE and upload it to your Arduino.

```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// ================= OLED =================

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// ================= BUTTONS =================

#define START_BUTTON 2
#define RED_BUTTON   3
#define GREEN_BUTTON 4
#define BLUE_BUTTON  5

// ================= RGB LED =================

#define RED_LED   9
#define GREEN_LED 10
#define BLUE_LED  11

// ================= GAME SETTINGS =================

#define TOTAL_ROUNDS 10

// Game states
enum GameState {
  MENU,
  READY,
  WAITING,
  SHOWING_COLOR,
  PAUSED,
  RESULT,
  GAME_OVER
};

GameState state = MENU;

// Current target color
int targetColor = 0;

// Reaction timing
unsigned long colorStartTime = 0;
unsigned long reactionTime = 0;

// Score
int score = 0;
int roundNumber = 0;

unsigned long totalReactionTime = 0;
unsigned long bestTime = 999999;

// Button debounce
unsigned long lastStartPress = 0;
unsigned long lastRedPress = 0;
unsigned long lastGreenPress = 0;
unsigned long lastBluePress = 0;

const unsigned long debounceTime = 180;

// ======================================================
// RGB LED FUNCTIONS
// ======================================================

void setRGB(bool r, bool g, bool b) {
  digitalWrite(RED_LED, r);
  digitalWrite(GREEN_LED, g);
  digitalWrite(BLUE_LED, b);
}

void turnOffLED() {
  setRGB(LOW, LOW, LOW);
}

void showRed() {
  setRGB(HIGH, LOW, LOW);
}

void showGreen() {
  setRGB(LOW, HIGH, LOW);
}

void showBlue() {
  setRGB(LOW, LOW, HIGH);
}

// ======================================================
// OLED FUNCTIONS
// ======================================================

void clearDisplay() {
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
}

void showMenu() {

  clearDisplay();

  display.setTextSize(2);
  display.setCursor(10, 5);
  display.println("REACTION");

  display.setCursor(28, 25);
  display.println("GAME");

  display.setTextSize(1);
  display.setCursor(22, 48);
  display.println("Press START");

  display.display();
}

void showReady() {

  clearDisplay();

  display.setTextSize(2);
  display.setCursor(25, 5);
  display.println("GET");

  display.setCursor(25, 27);
  display.println("READY!");

  display.setTextSize(1);
  display.setCursor(15, 52);
  display.println("Wait for the color");

  display.display();
}

void showColorScreen() {

  clearDisplay();

  display.setTextSize(2);

  if (targetColor == 0) {
    display.setCursor(28, 5);
    display.println("RED");
  }

  else if (targetColor == 1) {
    display.setCursor(22, 5);
    display.println("GREEN");
  }

  else {
    display.setCursor(22, 5);
    display.println("BLUE");
  }

  display.setTextSize(1);
  display.setCursor(20, 32);
  display.println("PRESS BUTTON!");

  display.setCursor(15, 50);
  display.print("Round ");
  display.print(roundNumber);
  display.print("/");
  display.print(TOTAL_ROUNDS);

  display.display();
}

void showPaused() {

  clearDisplay();

  display.setTextSize(2);
  display.setCursor(25, 8);
  display.println("PAUSED");

  display.setTextSize(1);
  display.setCursor(10, 35);
  display.println("START = Resume");

  display.setCursor(10, 50);
  display.println("Hold? Press again");

  display.display();
}

void showReactionTime() {

  clearDisplay();

  display.setTextSize(1);
  display.setCursor(25, 3);
  display.println("REACTION TIME");

  display.setTextSize(2);
  display.setCursor(25, 20);
  display.print(reactionTime);
  display.println(" ms");

  display.setTextSize(1);

  display.setCursor(5, 48);
  display.print("Score: ");
  display.print(score);

  display.display();
}

void showGameOver() {

  clearDisplay();

  display.setTextSize(2);
  display.setCursor(18, 2);
  display.println("GAME");

  display.setCursor(18, 23);
  display.println("OVER!");

  display.setTextSize(1);

  unsigned long average = 0;

  if (score > 0) {
    average = totalReactionTime / score;
  }

  display.setCursor(10, 45);
  display.print("Score: ");
  display.print(score);
  display.print("/");
  display.print(TOTAL_ROUNDS);

  display.setCursor(10, 56);

  if (score > 0) {
    display.print("Avg: ");
    display.print(average);
    display.print(" ms");
  }

  display.display();
}

// ======================================================
// RANDOM COLOR
// ======================================================

void chooseRandomColor() {

  targetColor = random(0, 3);

  if (targetColor == 0) {
    showRed();
  }

  else if (targetColor == 1) {
    showGreen();
  }

  else {
    showBlue();
  }

  colorStartTime = millis();

  showColorScreen();

  state = SHOWING_COLOR;
}

// ======================================================
// START NEW ROUND
// ======================================================

void startRound() {

  roundNumber++;

  turnOffLED();

  showReady();

  state = READY;

  // Random waiting time
  unsigned long waitTime = random(1000, 3000);

  delay(waitTime);

  chooseRandomColor();
}

// ======================================================
// CHECK COLOR BUTTON
// ======================================================

void checkColorButton() {

  // RED
  if (digitalRead(RED_BUTTON) == LOW) {

    if (millis() - lastRedPress > debounceTime) {

      lastRedPress = millis();

      if (targetColor == 0) {
        reactionTime = millis() - colorStartTime;

        score++;

        totalReactionTime += reactionTime;

        if (reactionTime < bestTime) {
          bestTime = reactionTime;
        }

        turnOffLED();

        showReactionTime();

        state = RESULT;

        delay(1200);

        if (roundNumber < TOTAL_ROUNDS) {
          startRound();
        }

        else {
          state = GAME_OVER;
          showGameOver();
        }
      }

      else {
        // WRONG BUTTON

        turnOffLED();

        clearDisplay();

        display.setTextSize(2);
        display.setCursor(30, 15);
        display.println("WRONG!");

        display.setTextSize(1);
        display.setCursor(15, 42);
        display.println("That wasn't the color");

        display.display();

        delay(1000);

        if (roundNumber < TOTAL_ROUNDS) {
          startRound();
        }

        else {
          state = GAME_OVER;
          showGameOver();
        }
      }
    }
  }

  // GREEN
  if (digitalRead(GREEN_BUTTON) == LOW) {

    if (millis() - lastGreenPress > debounceTime) {

      lastGreenPress = millis();

      if (targetColor == 1) {

        reactionTime = millis() - colorStartTime;

        score++;

        totalReactionTime += reactionTime;

        if (reactionTime < bestTime) {
          bestTime = reactionTime;
        }

        turnOffLED();

        showReactionTime();

        state = RESULT;

        delay(1200);

        if (roundNumber < TOTAL_ROUNDS) {
          startRound();
        }

        else {
          state = GAME_OVER;
          showGameOver();
        }
      }

      else {

        turnOffLED();

        clearDisplay();

        display.setTextSize(2);
        display.setCursor(30, 15);
        display.println("WRONG!");

        display.setTextSize(1);
        display.setCursor(15, 42);
        display.println("That wasn't the color");

        display.display();

        delay(1000);

        if (roundNumber < TOTAL_ROUNDS) {
          startRound();
        }

        else {
          state = GAME_OVER;
          showGameOver();
        }
      }
    }
  }

  // BLUE
  if (digitalRead(BLUE_BUTTON) == LOW) {

    if (millis() - lastBluePress > debounceTime) {

      lastBluePress = millis();

      if (targetColor == 2) {

        reactionTime = millis() - colorStartTime;

        score++;

        totalReactionTime += reactionTime;

        if (reactionTime < bestTime) {
          bestTime = reactionTime;
        }

        turnOffLED();

        showReactionTime();

        state = RESULT;

        delay(1200);

        if (roundNumber < TOTAL_ROUNDS) {
          startRound();
        }

        else {
          state = GAME_OVER;
          showGameOver();
        }
      }

      else {

        turnOffLED();

        clearDisplay();

        display.setTextSize(2);
        display.setCursor(30, 15);
        display.println("WRONG!");

        display.setTextSize(1);
        display.setCursor(15, 42);
        display.println("That wasn't the color");

        display.display();

        delay(1000);

        if (roundNumber < TOTAL_ROUNDS) {
          startRound();
        }

        else {
          state = GAME_OVER;
          showGameOver();
        }
      }
    }
  }
}

// ======================================================
// START / PAUSE BUTTON
// ======================================================

void checkStartButton() {

  if (digitalRead(START_BUTTON) == LOW) {

    if (millis() - lastStartPress > debounceTime) {

      lastStartPress = millis();

      // MENU → START GAME

      if (state == MENU) {

        score = 0;
        roundNumber = 0;
        totalReactionTime = 0;
        bestTime = 999999;

        startRound();
      }

      // SHOWING COLOR → PAUSE

      else if (state == SHOWING_COLOR) {

        turnOffLED();

        state = PAUSED;

        showPaused();
      }

      // PAUSED → RESUME

      else if (state == PAUSED) {

        showColorScreen();

        if (targetColor == 0) {
          showRed();
        }

        else if (targetColor == 1) {
          showGreen();
        }

        else {
          showBlue();
        }

        // Restart reaction timing after resume
        colorStartTime = millis();

        state = SHOWING_COLOR;
      }

      // GAME OVER → NEW GAME

      else if (state == GAME_OVER) {

        score = 0;
        roundNumber = 0;
        totalReactionTime = 0;
        bestTime = 999999;

        startRound();
      }
    }
  }
}

// ======================================================
// SETUP
// ======================================================

void setup() {

  Serial.begin(9600);

  // Buttons
  pinMode(START_BUTTON, INPUT_PULLUP);
  pinMode(RED_BUTTON, INPUT_PULLUP);
  pinMode(GREEN_BUTTON, INPUT_PULLUP);
  pinMode(BLUE_BUTTON, INPUT_PULLUP);

  // RGB LED
  pinMode(RED_LED, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);
  pinMode(BLUE_LED, OUTPUT);

  turnOffLED();

  // OLED
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {

    Serial.println("OLED not found!");

    while (1);
  }

  display.clearDisplay();
  display.display();

  // Random seed
  randomSeed(analogRead(A0));

  showMenu();
}

// ======================================================
// LOOP
// ======================================================

void loop() {

  checkStartButton();

  if (state == SHOWING_COLOR) {
    checkColorButton();
  }

  // GAME OVER
  if (state == GAME_OVER) {

    // Start button handled above
  }
}
```
