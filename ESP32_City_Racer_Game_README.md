# City Racer Game (ESP32 + TFT)

## Description
Get ready to race! This project is a retro-style top-down racing game built for the ESP32 using a 1.8" SPI TFT display. 
You control a blue car driving down a 3-lane highway, dodging incoming enemy cars. You steer left and right using an analog joystick. The game speed progressively increases as your score goes up, making it harder to survive! 

## Hardware Requirements
- ESP32 Development Board
- 1.8" SPI TFT Display (ST7735 controller, 128x160 resolution)
- Analog Joystick Module
- 1x Push Button
- Jumper Wires

## Software Dependencies
Install the following libraries in the Arduino IDE before compiling:
- **Adafruit GFX Library**
- **Adafruit ST7735 and ST7789 Library**

*(Note: The `SPI` library is built-in).*

## Connections

### 1. SPI TFT Display (ST7735)
| TFT Pin | ESP32 | Notes |
| :--- | :--- | :--- |
| VCC | 3.3V | Power |
| GND | GND | Ground |
| CS | GPIO 5 | Defined as `TFT_CS` |
| RESET | GPIO 4 | Defined as `TFT_RST` |
| A0 / DC | GPIO 2 | Defined as `TFT_DC` |
| SDA / MOSI | GPIO 23 | Hardware VSPI MOSI |
| SCK / SCLK | GPIO 18 | Hardware VSPI CLK |
| LED / BLK | 3.3V | Backlight Power |

### 2. Analog Joystick
| Joystick Pin | ESP32 | Notes |
| :--- | :--- | :--- |
| VCC | 3.3V | Power |
| GND | GND | Ground |
| VRx | GPIO 34 | Analog input for horizontal steering |
| VRy | NOT CONNECTED | Vertical movement is not used |
| SW | NOT CONNECTED | Joystick button is not used |

### 3. Start / Restart Button
Connect one side of the button to the ESP32 and the other side directly to **GND**. The code uses `INPUT_PULLUP`, so no external resistor is needed.

| Button | ESP32 | Notes |
| :--- | :--- | :--- |
| One terminal | GPIO 26 | Defined as `BUTTON_PIN` |
| Other terminal | GND | Ground |

## Code

Copy and paste this completely into your Arduino IDE and upload it to your ESP32.

```cpp
#include <SPI.h>
#include <Adafruit_GFX.h>
#include <Adafruit_ST7735.h>

// =====================================================
// TFT CONNECTIONS
// =====================================================

#define TFT_CS    5
#define TFT_RST   4
#define TFT_DC    2
#define TFT_MOSI  23
#define TFT_SCLK  18

// =====================================================
// CONTROLS
// =====================================================

#define JOYSTICK_X 34
#define BUTTON_PIN 26

// =====================================================
// TFT
// =====================================================

Adafruit_ST7735 tft = Adafruit_ST7735(
  TFT_CS,
  TFT_DC,
  TFT_RST
);

// =====================================================
// DISPLAY
// =====================================================

#define SCREEN_W 160
#define SCREEN_H 128

// =====================================================
// COLORS
// =====================================================

#define SKY       0x7D7C
#define ROAD      0x4208
#define GRASS     0x0320
#define WHITE     ST77XX_WHITE
#define BLACK     ST77XX_BLACK
#define RED       ST77XX_RED
#define BLUE      ST77XX_BLUE
#define YELLOW    ST77XX_YELLOW
#define GREEN     ST77XX_GREEN
#define CYAN      ST77XX_CYAN
#define ORANGE    0xFD20
#define GRAY      0x8410

// =====================================================
// ROAD
// =====================================================

const int ROAD_LEFT  = 35;
const int ROAD_RIGHT = 125;

const int ROAD_WIDTH = ROAD_RIGHT - ROAD_LEFT;

// Lane positions
const int LANE1 = 50;
const int LANE2 = 80;
const int LANE3 = 110;

// =====================================================
// PLAYER CAR
// =====================================================

int playerX = 80;
int playerY = 98;

const int CAR_W = 14;
const int CAR_H = 24;

// =====================================================
// ENEMIES
// =====================================================

#define MAX_ENEMIES 3

int enemyX[MAX_ENEMIES];
int enemyY[MAX_ENEMIES];
int enemySpeed[MAX_ENEMIES];

bool enemyActive[MAX_ENEMIES];

// Different enemy colors
uint16_t enemyColors[MAX_ENEMIES] = {
  RED,
  BLUE,
  YELLOW
};

// =====================================================
// GAME
// =====================================================

enum GameState
{
  START,
  PLAYING,
  GAMEOVER
};

GameState gameState = START;

// =====================================================
// SPEED
// =====================================================

float roadSpeed = 2.0;

// =====================================================
// SCORE
// =====================================================

unsigned long score = 0;

// =====================================================
// ROAD ANIMATION
// =====================================================

int roadOffset = 0;

// =====================================================
// BUTTON DEBOUNCE
// =====================================================

bool lastButton = HIGH;

unsigned long lastButtonTime = 0;

const unsigned long debounceTime = 180;

// =====================================================
// FRAME TIMER
// =====================================================

unsigned long lastFrame = 0;

// =====================================================
// SETUP
// =====================================================

void setup()
{
  Serial.begin(115200);

  pinMode(JOYSTICK_X, INPUT);

  pinMode(BUTTON_PIN, INPUT_PULLUP);

  // Start SPI
  SPI.begin(
    TFT_SCLK,
    -1,
    TFT_MOSI,
    TFT_CS
  );

  // Start display
  tft.initR(INITR_BLACKTAB);

  // Landscape
  tft.setRotation(1);

  tft.fillScreen(BLACK);

  randomSeed(analogRead(34));

  showStartScreen();
}

// =====================================================
// MAIN LOOP
// =====================================================

void loop()
{
  if (gameState == START)
  {
    startScreenLoop();
  }

  else if (gameState == PLAYING)
  {
    gameLoop();
  }

  else if (gameState == GAMEOVER)
  {
    gameOverLoop();
  }
}

// =====================================================
// START SCREEN
// =====================================================

void showStartScreen()
{
  tft.fillScreen(BLACK);

  // CITY
  tft.setTextColor(CYAN);
  tft.setTextSize(2);

  tft.setCursor(30, 12);
  tft.print("CITY");

  tft.setCursor(30, 32);
  tft.print("RACER");

  // Small car
  drawCar(72, 58, RED);

  // Instructions
  tft.setTextSize(1);

  tft.setTextColor(WHITE);

  tft.setCursor(27, 88);
  tft.print("JOYSTICK = STEER");

  tft.setTextColor(YELLOW);

  tft.setCursor(29, 103);
  tft.print("PRESS BUTTON");

  tft.setCursor(47, 115);
  tft.print("START");
}

// =====================================================
// START SCREEN LOOP
// =====================================================

void startScreenLoop()
{
  if (buttonPressed())
  {
    startGame();
  }
}

// =====================================================
// START GAME
// =====================================================

void startGame()
{
  gameState = PLAYING;

  playerX = 80;

  score = 0;

  roadSpeed = 2.0;

  roadOffset = 0;

  // Clear screen
  tft.fillScreen(BLACK);

  // Turn off enemies
  for (int i = 0; i < MAX_ENEMIES; i++)
  {
    enemyActive[i] = false;
  }

  // Create first enemies
  spawnEnemy(0);
  spawnEnemy(1);

  drawEntireScene();
}

// =====================================================
// GAME LOOP
// =====================================================

void gameLoop()
{
  unsigned long now = millis();

  // ~30 FPS
  if (now - lastFrame < 33)
    return;

  lastFrame = now;

  // -----------------------------------------------
  // INPUT
  // -----------------------------------------------

  readJoystick();

  // -----------------------------------------------
  // MOVE ROAD
  // -----------------------------------------------

  roadOffset += roadSpeed;

  if (roadOffset >= 20)
    roadOffset = 0;

  // -----------------------------------------------
  // MOVE ENEMIES
  // -----------------------------------------------

  moveEnemies();

  // -----------------------------------------------
  // SCORE
  // -----------------------------------------------

  score++;

  // -----------------------------------------------
  // INCREASE SPEED
  // -----------------------------------------------

  if (score % 500 == 0)
  {
    roadSpeed += 0.3;

    if (roadSpeed > 5.0)
      roadSpeed = 5.0;
  }

  // -----------------------------------------------
  // COLLISION
  // -----------------------------------------------

  if (checkCollision())
  {
    gameState = GAMEOVER;

    showGameOver();

    return;
  }

  // -----------------------------------------------
  // DRAW
  // -----------------------------------------------

  drawEntireScene();
}

// =====================================================
// JOYSTICK
// =====================================================

void readJoystick()
{
  int x = analogRead(JOYSTICK_X);

  // Dead zone
  if (x < 1500)
  {
    playerX -= 3;
  }

  else if (x > 2600)
  {
    playerX += 3;
  }

  // Keep car on road
  int minX = ROAD_LEFT + 4;
  int maxX = ROAD_RIGHT - CAR_W - 4;

  if (playerX < minX)
    playerX = minX;

  if (playerX > maxX)
    playerX = maxX;
}

// =====================================================
// MOVE ENEMIES
// =====================================================

void moveEnemies()
{
  for (int i = 0; i < MAX_ENEMIES; i++)
  {
    if (!enemyActive[i])
      continue;

    enemyY[i] += roadSpeed + enemySpeed[i];

    // Enemy passed player
    if (enemyY[i] > SCREEN_H + 30)
    {
      spawnEnemy(i);
    }
  }
}

// =====================================================
// SPAWN ENEMY
// =====================================================

void spawnEnemy(int i)
{
  int lane = random(0, 3);

  if (lane == 0)
    enemyX[i] = LANE1;

  else if (lane == 1)
    enemyX[i] = LANE2;

  else
    enemyX[i] = LANE3;

  // Spawn above screen
  enemyY[i] = -random(30, 100);

  enemySpeed[i] = random(1, 3);

  enemyActive[i] = true;
}

// =====================================================
// DRAW COMPLETE SCENE
// =====================================================

void drawEntireScene()
{
  // -----------------------------------------------
  // Background
  // -----------------------------------------------

  tft.fillScreen(GRASS);

  // -----------------------------------------------
  // CITY BUILDINGS
  // -----------------------------------------------

  drawCity();

  // -----------------------------------------------
  // ROAD
  // -----------------------------------------------

  tft.fillRect(
    ROAD_LEFT,
    0,
    ROAD_WIDTH,
    SCREEN_H,
    ROAD
  );

  // -----------------------------------------------
  // ROAD EDGES
  // -----------------------------------------------

  tft.drawFastVLine(
    ROAD_LEFT,
    0,
    SCREEN_H,
    WHITE
  );

  tft.drawFastVLine(
    ROAD_RIGHT,
    0,
    SCREEN_H,
    WHITE
  );

  // -----------------------------------------------
  // LANE MARKINGS
  // -----------------------------------------------

  drawLaneMarks();

  // -----------------------------------------------
  // ENEMY CARS
  // -----------------------------------------------

  for (int i = 0; i < MAX_ENEMIES; i++)
  {
    if (enemyActive[i])
    {
      drawCar(
        enemyX[i],
        enemyY[i],
        enemyColors[i]
      );
    }
  }

  // -----------------------------------------------
  // PLAYER
  // -----------------------------------------------

  drawCar(
    playerX,
    playerY,
    CYAN
  );

  // -----------------------------------------------
  // SCORE
  // -----------------------------------------------

  drawScore();
}

// =====================================================
// CITY BUILDINGS
// =====================================================

void drawCity()
{
  // Left buildings

  tft.fillRect(3, 5, 20, 28, GRAY);
  tft.fillRect(6, 9, 4, 5, YELLOW);
  tft.fillRect(15, 9, 4, 5, YELLOW);
  tft.fillRect(6, 20, 4, 5, YELLOW);
  tft.fillRect(15, 20, 4, 5, YELLOW);

  tft.fillRect(5, 45, 24, 35, BLUE);
  tft.fillRect(9, 50, 5, 5, WHITE);
  tft.fillRect(19, 50, 5, 5, WHITE);
  tft.fillRect(9, 62, 5, 5, WHITE);
  tft.fillRect(19, 62, 5, 5, WHITE);

  tft.fillRect(2, 88, 25, 30, ORANGE);

  tft.fillRect(7, 93, 5, 5, YELLOW);
  tft.fillRect(17, 93, 5, 5, YELLOW);
  tft.fillRect(7, 105, 5, 5, YELLOW);
  tft.fillRect(17, 105, 5, 5, YELLOW);

  // Right buildings

  tft.fillRect(133, 2, 23, 32, BLUE);

  tft.fillRect(137, 7, 5, 5, WHITE);
  tft.fillRect(147, 7, 5, 5, WHITE);
  tft.fillRect(137, 18, 5, 5, WHITE);
  tft.fillRect(147, 18, 5, 5, WHITE);

  tft.fillRect(130, 43, 27, 35, GRAY);

  tft.fillRect(135, 48, 5, 5, YELLOW);
  tft.fillRect(147, 48, 5, 5, YELLOW);
  tft.fillRect(135, 60, 5, 5, YELLOW);
  tft.fillRect(147, 60, 5, 5, YELLOW);

  tft.fillRect(132, 87, 24, 31, RED);

  tft.fillRect(137, 92, 5, 5, WHITE);
  tft.fillRect(148, 92, 5, 5, WHITE);
}

// =====================================================
// ROAD LANE MARKINGS
// =====================================================

void drawLaneMarks()
{
  int offset = roadOffset;

  // Lane 1 / 2
  for (int y = -20 + offset; y < SCREEN_H; y += 20)
  {
    tft.fillRect(
      64,
      y,
      3,
      10,
      WHITE
    );
  }

  // Lane 2 / 3
  for (int y = -20 + offset; y < SCREEN_H; y += 20)
  {
    tft.fillRect(
      94,
      y,
      3,
      10,
      WHITE
    );
  }
}

// =====================================================
// DRAW CAR
// =====================================================

void drawCar(
  int x,
  int y,
  uint16_t color
)
{
  // Don't draw if completely offscreen
  if (y > SCREEN_H || y < -CAR_H)
    return;

  // Body
  tft.fillRoundRect(
    x,
    y,
    CAR_W,
    CAR_H,
    3,
    color
  );

  // Windows
  tft.fillRect(
    x + 3,
    y + 4,
    8,
    6,
    BLACK
  );

  // Front window
  tft.fillRect(
    x + 3,
    y + 13,
    8,
    5,
    BLACK
  );

  // Wheels
  tft.fillRect(
    x - 2,
    y + 4,
    2,
    6,
    BLACK
  );

  tft.fillRect(
    x + CAR_W,
    y + 4,
    2,
    6,
    BLACK
  );

  tft.fillRect(
    x - 2,
    y + 16,
    2,
    6,
    BLACK
  );

  tft.fillRect(
    x + CAR_W,
    y + 16,
    2,
    6,
    BLACK
  );

  // Headlights
  tft.fillRect(
    x + 2,
    y + 1,
    3,
    2,
    YELLOW
  );

  tft.fillRect(
    x + 9,
    y + 1,
    3,
    2,
    YELLOW
  );
}

// =====================================================
// COLLISION
// =====================================================

bool checkCollision()
{
  // Player hitbox
  int pLeft   = playerX + 2;
  int pRight  = playerX + CAR_W - 2;
  int pTop    = playerY + 2;
  int pBottom = playerY + CAR_H - 2;

  for (int i = 0; i < MAX_ENEMIES; i++)
  {
    if (!enemyActive[i])
      continue;

    int eLeft   = enemyX[i] + 2;
    int eRight  = enemyX[i] + CAR_W - 2;
    int eTop    = enemyY[i] + 2;
    int eBottom = enemyY[i] + CAR_H - 2;

    if (
      pLeft < eRight &&
      pRight > eLeft &&
      pTop < eBottom &&
      pBottom > eTop
    )
    {
      return true;
    }
  }

  return false;
}

// =====================================================
// SCORE
// =====================================================

void drawScore()
{
  // Black box
  tft.fillRect(
    2,
    2,
    55,
    12,
    BLACK
  );

  tft.setTextSize(1);

  tft.setTextColor(WHITE);

  tft.setCursor(5, 5);

  tft.print("SCORE:");

  tft.print(score / 10);
}

// =====================================================
// BUTTON
// =====================================================

bool buttonPressed()
{
  bool current = digitalRead(BUTTON_PIN);

  bool pressed = false;

  if (
    lastButton == HIGH &&
    current == LOW
  )
  {
    if (
      millis() - lastButtonTime >
      debounceTime
    )
    {
      pressed = true;

      lastButtonTime = millis();
    }
  }

  lastButton = current;

  return pressed;
}

// =====================================================
// GAME OVER
// =====================================================

void showGameOver()
{
  tft.fillScreen(BLACK);

  // GAME
  tft.setTextSize(2);

  tft.setTextColor(RED);

  tft.setCursor(25, 20);

  tft.print("CRASH!");

  // Score
  tft.setTextSize(1);

  tft.setTextColor(WHITE);

  tft.setCursor(40, 55);

  tft.print("SCORE: ");

  tft.print(score / 10);

  // Restart
  tft.setTextColor(YELLOW);

  tft.setCursor(23, 82);

  tft.print("PRESS BUTTON");

  tft.setCursor(42, 100);

  tft.print("RESTART");
}

// =====================================================
// GAME OVER LOOP
// =====================================================

void gameOverLoop()
{
  if (buttonPressed())
  {
    startGame();
  }
}
```
