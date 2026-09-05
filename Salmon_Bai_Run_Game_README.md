# Salmon Bai Run (OLED + IR Sensor Game)

## Description
This project is an endless runner game for Arduino featuring "Salmon Bai", a Bollywood-style action hero! 
The game is played on a 128x64 I2C OLED display. Instead of pressing physical buttons, you control the character hands-free by waving your hand in front of an IR sensor to make him jump over the oncoming cacti. 

## Hardware Requirements
- Arduino UNO (or any compatible board)
- I2C OLED Display (128x64 resolution)
- IR Obstacle Avoidance Sensor Module
- Jumper Wires

## Software Dependencies
Install the following libraries in the Arduino IDE before compiling:
- **Adafruit GFX Library**
- **Adafruit SSD1306**

*(Note: The `Wire` library for I2C communication is built-in).*

## Connections

### 1. OLED Display (I2C)
| OLED Pin | Arduino UNO |
| :--- | :--- |
| VCC | 5V (or 3.3V depending on your module) |
| GND | GND |
| SDA | A4 |
| SCL | A5 |

### 2. IR Sensor Module
| IR Sensor Pin | Arduino UNO | Notes |
| :--- | :--- | :--- |
| VCC | 5V | Power |
| GND | GND | Ground |
| OUT / D0 | D2 | Defined as `IR_PIN` in the code |

## How to Play
1. **Start the Game**: Once the code is uploaded, the screen will display the title and "Wave hand to jump!". 
2. **Jump**: Trigger the IR sensor by bringing your hand close to it to make "Salmon Bai" jump over the incoming cactus obstacles.
3. **Game Over**: If you hit a cactus, the game ends and your final score is displayed.
4. **Restart**: Wave your hand in front of the IR sensor again to restart the game and try to beat your high score.

## Code

Copy and paste this completely into your Arduino IDE and upload it to your Arduino.

```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64

#define OLED_RESET -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// IR sensor
#define IR_PIN 2

// Game settings
int dinoX = 15;
int dinoY = 45;

int velocityY = 0;
bool jumping = false;

int cactusX = 128;
int cactusY = 48;

int score = 0;

bool gameOver = false;

unsigned long lastFrame = 0;

// --------------------------------------------------
// Draw Bollywood-style action hero Dino
// --------------------------------------------------
void drawDino(int x, int y) {

  // Head
  display.fillCircle(x + 8, y - 7, 7, SSD1306_WHITE);

  // Hair
  display.fillTriangle(
    x + 2, y - 12,
    x + 8, y - 16,
    x + 14, y - 11,
    SSD1306_WHITE
  );

  // Sunglasses
  display.fillRect(x + 2, y - 9, 5, 3, SSD1306_BLACK);
  display.fillRect(x + 9, y - 9, 5, 3, SSD1306_BLACK);

  // Body / jacket
  display.fillRoundRect(
    x + 2, y,
    13, 15,
    3,
    SSD1306_WHITE
  );

  // Shirt
  display.fillTriangle(
    x + 8, y + 2,
    x + 5, y + 10,
    x + 11, y + 10,
    SSD1306_BLACK
  );

  // Left arm
  display.drawLine(
    x + 2, y + 3,
    x - 4, y + 9,
    SSD1306_WHITE
  );

  // Right arm
  display.drawLine(
    x + 15, y + 3,
    x + 20, y + 9,
    SSD1306_WHITE
  );

  // Legs
  display.drawLine(
    x + 5, y + 14,
    x + 2, y + 22,
    SSD1306_WHITE
  );

  display.drawLine(
    x + 12, y + 14,
    x + 15, y + 22,
    SSD1306_WHITE
  );

  // Shoes
  display.drawLine(
    x + 2, y + 22,
    x - 1, y + 22,
    SSD1306_WHITE
  );

  display.drawLine(
    x + 15, y + 22,
    x + 18, y + 22,
    SSD1306_WHITE
  );
}


// --------------------------------------------------
// Draw cactus
// --------------------------------------------------
void drawCactus(int x) {

  display.fillRect(x, 48, 5, 15, SSD1306_WHITE);

  display.fillRect(x - 4, 53, 4, 3, SSD1306_WHITE);
  display.fillRect(x - 4, 50, 3, 6, SSD1306_WHITE);

  display.fillRect(x + 5, 55, 4, 3, SSD1306_WHITE);
  display.fillRect(x + 6, 52, 3, 6, SSD1306_WHITE);
}


// --------------------------------------------------
// Collision detection
// --------------------------------------------------
bool collision() {

  int dinoLeft = dinoX;
  int dinoRight = dinoX + 20;

  int dinoBottom = dinoY + 22;

  int cactusLeft = cactusX - 4;
  int cactusRight = cactusX + 9;

  int cactusTop = 48;

  if (dinoRight > cactusLeft &&
      dinoLeft < cactusRight &&
      dinoBottom > cactusTop) {

    return true;
  }

  return false;
}


// --------------------------------------------------
// Setup
// --------------------------------------------------
void setup() {

  pinMode(IR_PIN, INPUT);

  Serial.begin(9600);

  if (!display.begin(
        SSD1306_SWITCHCAPVCC,
        0x3C)) {

    while (1);
  }

  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);

  display.setTextSize(2);
  display.setCursor(18, 10);
  display.println("SAMON BAI");

  display.setTextSize(1);
  display.setCursor(22, 35);
  display.println("SALMON BAI RUN");

  display.setCursor(15, 52);
  display.println("Wave hand to jump!");

  display.display();

  delay(2000);
}


// --------------------------------------------------
// Main game loop
// --------------------------------------------------
void loop() {

  // IR sensor triggered
  // Most modules output LOW when object is detected
  if (digitalRead(IR_PIN) == LOW) {

    if (!jumping && !gameOver) {

      velocityY = -9;
      jumping = true;
    }
  }


  // Restart after game over
  if (gameOver) {

    display.clearDisplay();

    display.setTextSize(2);
    display.setCursor(20, 15);
    display.println("GAME");

    display.setCursor(28, 35);
    display.println("OVER");

    display.setTextSize(1);
    display.setCursor(20, 55);
    display.print("Score: ");
    display.print(score);

    display.display();

    // IR trigger = restart
    if (digitalRead(IR_PIN) == LOW) {

      delay(500);

      cactusX = 128;
      score = 0;

      dinoY = 45;
      velocityY = 0;

      jumping = false;
      gameOver = false;
    }

    return;
  }


  // Control frame rate
  if (millis() - lastFrame < 45) {
    return;
  }

  lastFrame = millis();


  // ------------------------------
  // Dino physics
  // ------------------------------

  if (jumping) {

    dinoY += velocityY;

    velocityY += 1;

    // Ground
    if (dinoY >= 45) {

      dinoY = 45;

      velocityY = 0;

      jumping = false;
    }
  }


  // ------------------------------
  // Move cactus
  // ------------------------------

  cactusX -= 3;

  if (cactusX < -10) {

    cactusX = 128;

    score++;
  }


  // ------------------------------
  // Collision
  // ------------------------------

  if (collision()) {

    gameOver = true;
  }


  // ------------------------------
  // Draw screen
  // ------------------------------

  display.clearDisplay();


  // Ground
  display.drawLine(
    0, 63,
    127, 63,
    SSD1306_WHITE
  );


  // Dino
  drawDino(
    dinoX,
    dinoY
  );


  // Cactus
  drawCactus(
    cactusX
  );


  // Score
  display.setTextSize(1);
  display.setCursor(90, 2);
  display.print(score);


  display.display();
}
```
