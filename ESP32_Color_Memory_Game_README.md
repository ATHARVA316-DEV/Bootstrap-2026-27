# ESP32 Color Memory Game

## Description
A Color Memory Game built using an ESP32, an I2C OLED display, a common-cathode RGB LED, and 4 push buttons. 
The OLED shows the current level, and the RGB LED flashes a random color sequence. Your goal is to remember the sequence and press the corresponding colored buttons in the exact same order!

- **Level 1** → 1 color
- **Level 2** → 2 colors
- **Level 3** → 3 colors
- **Level 4** → 4 colors
- **Level 5** → 5 colors
- **Level 6–10** → 5 colors, but with random/tougher combinations

Colors can repeat, so patterns aren't predictable (e.g., Level 5 could be R B B G R). The **START** button is used to begin the game and can also function as a **PAUSE/RESUME** button during gameplay.

## Hardware Requirements
- ESP32 Development Board
- I2C OLED Display
- Common-Cathode RGB LED
- 4x Push Buttons (Red, Green, Blue, Start)
- 3x 330Ω Resistors (for the RGB LED)
- Jumper wires

## Software Dependencies
Install the following libraries in the Arduino IDE before compiling:
- **Adafruit GFX Library**
- **Adafruit SSD1306**

## Connections

### 1. OLED (I2C)
| OLED Pin | ESP32 |
| :--- | :--- |
| VCC | 3.3V |
| GND | GND |
| SDA | GPIO 21 |
| SCL | GPIO 22 |

### 2. Buttons
Connect one side of each button to the respective ESP32 GPIO pin and the other side to **GND**. The code uses `INPUT_PULLUP`, so no external pull-up resistors are required for the buttons.

| Button | ESP32 |
| :--- | :--- |
| RED Button | GPIO 25 |
| GREEN Button | GPIO 26 |
| BLUE Button | GPIO 27 |
| START / PAUSE | GPIO 14 |

### 3. RGB LED (Common Cathode)
Use 3 separate 330Ω resistors in series, one for each color pin of the RGB LED.

| RGB LED Pin | Connection | ESP32 |
| :--- | :--- | :--- |
| RED | 330Ω Resistor | GPIO 16 |
| GREEN | 330Ω Resistor | GPIO 17 |
| BLUE | 330Ω Resistor | GPIO 18 |
| Common Cathode | Direct | GND |

## Code

Copy and paste the following code into your Arduino IDE and upload it to your ESP32.

```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// ================= OLED =================

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// ================= BUTTONS =================

#define RED_BUTTON    25
#define GREEN_BUTTON  26
#define BLUE_BUTTON   27
#define START_BUTTON  14

// ================= RGB LED =================

#define RED_LED    16
#define GREEN_LED  17
#define BLUE_LED   18

// ================= GAME =================

#define MAX_LEVEL 10
#define MAX_SEQUENCE 5

int sequence[MAX_SEQUENCE];

int level = 1;
int playerPosition = 0;

bool gameStarted = false;
bool paused = false;


// =================================================
// RGB LED
// =================================================

void setColor(int color)
{
  if (color == 0)       // RED
  {
    analogWrite(RED_LED, 255);
    analogWrite(GREEN_LED, 0);
    analogWrite(BLUE_LED, 0);
  }
  else if (color == 1)  // GREEN
  {
    analogWrite(RED_LED, 0);
    analogWrite(GREEN_LED, 255);
    analogWrite(BLUE_LED, 0);
  }
  else if (color == 2)  // BLUE
  {
    analogWrite(RED_LED, 0);
    analogWrite(GREEN_LED, 0);
    analogWrite(BLUE_LED, 255);
  }
}

void ledOff()
{
  analogWrite(RED_LED, 0);
  analogWrite(GREEN_LED, 0);
  analogWrite(BLUE_LED, 0);
}


// =================================================
// OLED
// =================================================

void showText(String a, String b = "", String c = "")
{
  display.clearDisplay();

  display.setTextColor(SSD1306_WHITE);

  display.setTextSize(2);
  display.setCursor(5, 5);
  display.println(a);

  display.setTextSize(1);

  if (b != "")
  {
    display.setCursor(5, 32);
    display.println(b);
  }

  if (c != "")
  {
    display.setCursor(5, 48);
    display.println(c);
  }

  display.display();
}


// =================================================
// START BUTTON
// =================================================

bool startPressed()
{
  if (digitalRead(START_BUTTON) == LOW)
  {
    delay(30);

    if (digitalRead(START_BUTTON) == LOW)
    {
      while (digitalRead(START_BUTTON) == LOW)
      {
        delay(10);
      }

      delay(100);
      return true;
    }
  }

  return false;
}


// =================================================
// PAUSE
// =================================================

void pauseGame()
{
  paused = true;

  ledOff();

  showText(
    "PAUSED",
    "Press START",
    "to resume"
  );

  while (paused)
  {
    if (startPressed())
    {
      paused = false;
    }

    delay(10);
  }
}


// =================================================
// PAUSE-AWARE DELAY
// =================================================

void gameDelay(int time)
{
  int elapsed = 0;

  while (elapsed < time)
  {
    if (startPressed())
    {
      pauseGame();
    }

    delay(10);
    elapsed += 10;
  }
}


// =================================================
// SEQUENCE LENGTH
// =================================================

int getLength()
{
  if (level == 1)
    return 1;

  if (level == 2)
    return 2;

  if (level == 3)
    return 3;

  if (level == 4)
    return 4;

  return 5;
}


// =================================================
// GENERATE RANDOM SEQUENCE
// =================================================

void generateSequence()
{
  int length = getLength();

  for (int i = 0; i < length; i++)
  {
    sequence[i] = random(0, 3);
  }
}


// =================================================
// SHOW LEVEL
// =================================================

void showLevel()
{
  display.clearDisplay();

  display.setTextColor(SSD1306_WHITE);

  display.setTextSize(2);
  display.setCursor(20, 5);
  display.println("LEVEL");

  display.setTextSize(3);
  display.setCursor(50, 30);
  display.println(level);

  display.display();

  gameDelay(1000);
}


// =================================================
// SHOW COLOR SEQUENCE
// =================================================

void showSequence()
{
  int length = getLength();

  int flashTime = 700 - ((level - 1) * 40);

  if (flashTime < 340)
    flashTime = 340;

  showText(
    "WATCH!",
    "Remember colors"
  );

  gameDelay(1000);

  for (int i = 0; i < length; i++)
  {
    if (startPressed())
    {
      pauseGame();
    }

    setColor(sequence[i]);

    display.clearDisplay();

    display.setTextColor(SSD1306_WHITE);

    display.setTextSize(2);
    display.setCursor(15, 5);
    display.println("WATCH");

    display.setTextSize(1);

    display.setCursor(25, 35);

    display.print("Color ");
    display.print(i + 1);
    display.print(" / ");
    display.println(length);

    display.display();

    gameDelay(flashTime);

    ledOff();

    gameDelay(150);
  }

  showText(
    "YOUR TURN",
    "RED GREEN BLUE"
  );

  gameDelay(700);
}


// =================================================
// READ PLAYER BUTTON
// =================================================

int readPlayerButton()
{
  if (digitalRead(RED_BUTTON) == LOW)
  {
    delay(30);

    if (digitalRead(RED_BUTTON) == LOW)
    {
      while (digitalRead(RED_BUTTON) == LOW)
        delay(10);

      return 0;
    }
  }

  if (digitalRead(GREEN_BUTTON) == LOW)
  {
    delay(30);

    if (digitalRead(GREEN_BUTTON) == LOW)
    {
      while (digitalRead(GREEN_BUTTON) == LOW)
        delay(10);

      return 1;
    }
  }

  if (digitalRead(BLUE_BUTTON) == LOW)
  {
    delay(30);

    if (digitalRead(BLUE_BUTTON) == LOW)
    {
      while (digitalRead(BLUE_BUTTON) == LOW)
        delay(10);

      return 2;
    }
  }

  return -1;
}


// =================================================
// PLAYER TURN
// =================================================

bool playerTurn()
{
  int length = getLength();

  playerPosition = 0;

  while (playerPosition < length)
  {
    if (startPressed())
    {
      pauseGame();
    }

    int button = readPlayerButton();

    if (button != -1)
    {
      setColor(button);

      delay(180);

      ledOff();

      if (button != sequence[playerPosition])
      {
        return false;
      }

      playerPosition++;

      display.clearDisplay();

      display.setTextColor(SSD1306_WHITE);

      display.setTextSize(2);
      display.setCursor(20, 5);
      display.println("GOOD!");

      display.setTextSize(1);

      display.setCursor(25, 35);

      display.print(playerPosition);
      display.print(" / ");
      display.println(length);

      display.display();

      delay(250);
    }

    delay(10);
  }

  return true;
}


// =================================================
// LEVEL PASSED
// =================================================

void levelPassed()
{
  ledOff();

  if (level < MAX_LEVEL)
  {
    showText(
      "CORRECT!",
      "Level passed",
      "Next level..."
    );

    gameDelay(1200);

    level++;
  }
  else
  {
    display.clearDisplay();

    display.setTextColor(SSD1306_WHITE);

    display.setTextSize(2);
    display.setCursor(15, 5);
    display.println("YOU WIN!");

    display.setTextSize(1);

    display.setCursor(20, 35);
    display.println("All 10 levels");

    display.setCursor(30, 50);
    display.println("completed!");

    display.display();

    gameDelay(3000);

    gameStarted = false;
    level = 1;

    startScreen();
  }
}


// =================================================
// GAME OVER
// =================================================

void gameOver()
{
  ledOff();

  display.clearDisplay();

  display.setTextColor(SSD1306_WHITE);

  display.setTextSize(2);
  display.setCursor(5, 5);
  display.println("GAME OVER");

  display.setTextSize(1);

  display.setCursor(20, 32);

  display.print("Level: ");
  display.println(level);

  display.setCursor(15, 50);
  display.println("START = retry");

  display.display();

  while (true)
  {
    if (startPressed())
    {
      level = 1;
      gameStarted = true;

      generateSequence();

      return;
    }

    delay(10);
  }
}


// =================================================
// START SCREEN
// =================================================

void startScreen()
{
  display.clearDisplay();

  display.setTextColor(SSD1306_WHITE);

  display.setTextSize(2);

  display.setCursor(10, 5);
  display.println("COLOR");

  display.setCursor(10, 28);
  display.println("MEMORY");

  display.setTextSize(1);

  display.setCursor(25, 52);
  display.println("START TO PLAY");

  display.display();
}


// =================================================
// SETUP
// =================================================

void setup()
{
  Serial.begin(115200);

  pinMode(RED_BUTTON, INPUT_PULLUP);
  pinMode(GREEN_BUTTON, INPUT_PULLUP);
  pinMode(BLUE_BUTTON, INPUT_PULLUP);
  pinMode(START_BUTTON, INPUT_PULLUP);

  pinMode(RED_LED, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);
  pinMode(BLUE_LED, OUTPUT);

  ledOff();

  Wire.begin(21, 22);

  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C))
  {
    while (true)
    {
      delay(100);
    }
  }

  randomSeed(micros());

  startScreen();
}


// =================================================
// MAIN LOOP
// =================================================

void loop()
{
  if (!gameStarted)
  {
    if (startPressed())
    {
      gameStarted = true;
      level = 1;

      generateSequence();
    }
    else
    {
      delay(20);
      return;
    }
  }

  showLevel();

  generateSequence();

  showSequence();

  bool correct = playerTurn();

  if (correct)
  {
    levelPassed();

    if (level < MAX_LEVEL)
    {
      generateSequence();
    }
  }
  else
  {
    gameOver();
  }

  delay(300);
}
```
