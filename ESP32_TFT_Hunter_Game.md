# ESP32 TFT Hunter Game

## Description

A simple 2D hunter game built with an ESP32, a 1.8" 128x160 SPI TFT display, an analog joystick, and a push button. Navigate your hunter around the screen using the joystick, aim, and press the external button to shoot the prey while avoiding the trees. Survive 5 levels to win the game!

## Hardware Requirements

- ESP32 Development Board
- 1.8" 128x160 SPI TFT Display (ST7735)
- Analog Joystick Module
- Push Button (for shooting)
- Jumper wires

## Software Dependencies

Install the following libraries in the Arduino IDE before compiling:

- **Adafruit GFX Library**
- **Adafruit ST7735 and ST7789 Library**

## Connections

### 1. TFT — 1.8" 128×160 SPI V1.1

| TFT Pin | ESP32 | Function |
| :--- | :--- | :--- |
| VCC | 3.3V | Power |
| GND | GND | Ground |
| CS | GPIO 5 | Chip Select |
| RESET | GPIO 4 | Reset |
| A0 | GPIO 2 | Data/Command |
| SDA | GPIO 23 | MOSI |
| SCK | GPIO 18 | SPI Clock |
| LED | 3.3V | Backlight |

### 2. Joystick

| Joystick Pin | ESP32 |
| :--- | :--- |
| VCC | 3.3V |
| GND | GND |
| VRx | GPIO 34 |
| VRy | GPIO 35 |
| SW | NOT CONNECTED |

### 3. External Shooting Button

| Button | ESP32 |
| :--- | :--- |
| One terminal | GPIO 27 |
| Other terminal | GND |

*(No resistor is required because the code uses `INPUT_PULLUP`.)*

### Complete Wiring Diagram

```text
                     ESP32
                ┌───────────────┐
                │               │
 TFT VCC ───────┤ 3.3V          │
 TFT LED ───────┤ 3.3V          │
                │               │
 TFT GND ───────┤ GND           │
 JOY GND ───────┤ GND           │
 BUTTON ────────┤ GND           │
                │               │
 TFT SCK ───────┤ GPIO 18       │
 TFT SDA ───────┤ GPIO 23       │
 TFT CS ────────┤ GPIO 5        │
 TFT A0 ────────┤ GPIO 2        │
 TFT RESET ─────┤ GPIO 4        │
                │               │
 JOY VRx ───────┤ GPIO 34       │
 JOY VRy ───────┤ GPIO 35       │
                │               │
 SHOOT BUTTON ──┤ GPIO 27       │
                │               │
                └───────────────┘
```

## Code

Copy and paste the following code into your Arduino IDE and upload it to the ESP32.

```cpp
#include <SPI.h>
#include <Adafruit_GFX.h>
#include <Adafruit_ST7735.h>

// ============================================================
//                         TFT PINS
// ============================================================

#define TFT_CS   5
#define TFT_DC   2
#define TFT_RST  4

Adafruit_ST7735 tft = Adafruit_ST7735(
  TFT_CS,
  TFT_DC,
  TFT_RST
);


// ============================================================
//                       JOYSTICK
// ============================================================

#define JOY_X 34
#define JOY_Y 35

// Joystick SW is NOT USED


// ============================================================
//                     SHOOT BUTTON
// ============================================================

#define SHOOT_BUTTON 27


// ============================================================
//                       SCREEN
// ============================================================

#define SCREEN_W 128
#define SCREEN_H 160

#define GAME_TOP 18
#define GAME_BOTTOM 158


// ============================================================
//                     GAME SETTINGS
// ============================================================

#define MAX_PREY 5
#define MAX_BULLETS 3
#define MAX_TREES 5

#define PLAYER_RADIUS 5
#define PREY_RADIUS 4
#define BULLET_RADIUS 2

#define JOY_DEADZONE 400


// ============================================================
//                     GAME STATE
// ============================================================

enum GameState {
  PLAYING,
  GAME_OVER
};

GameState gameState = PLAYING;


// ============================================================
//                        PLAYER
// ============================================================

float playerX = 64;
float playerY = 135;

// Last joystick direction
// Used as shooting direction
float lastDirX = 0;
float lastDirY = -1;


// ============================================================
//                         BULLETS
// ============================================================

struct Bullet {

  bool active;

  float x;
  float y;

  float vx;
  float vy;
};

Bullet bullets[MAX_BULLETS];


// ============================================================
//                           PREY
// ============================================================

struct Prey {

  bool active;

  float x;
  float y;

  float vx;
  float vy;

  int type;

  unsigned long changeDirectionTime;
};

Prey prey[MAX_PREY];


// ============================================================
//                           TREES
// ============================================================

struct Tree {

  int x;
  int y;
  int r;
};

Tree trees[MAX_TREES] = {

  {20, 48, 8},
  {105, 42, 9},
  {35, 100, 7},
  {92, 115, 8},
  {65, 55, 7}
};


// ============================================================
//                      GAME VARIABLES
// ============================================================

int score = 0;

int lives = 3;

int ammo = 12;

int level = 1;

int preyAlive = 3;

int preyKilled = 0;


// ============================================================
//                     JOYSTICK CENTER
// ============================================================

int centerX = 2048;
int centerY = 2048;


// ============================================================
//                         BUTTON
// ============================================================

bool lastShootButton = HIGH;

unsigned long lastShootTime = 0;

const unsigned long DEBOUNCE = 180;


// ============================================================
//                         TIMING
// ============================================================

unsigned long lastFrame = 0;

const int FRAME_TIME = 45;


// ============================================================
//                          SETUP
// ============================================================

void setup() {

  Serial.begin(115200);

  // Joystick analog inputs
  pinMode(JOY_X, INPUT);
  pinMode(JOY_Y, INPUT);

  // External shooting button
  pinMode(
    SHOOT_BUTTON,
    INPUT_PULLUP
  );

  // TFT
  tft.initR(INITR_BLACKTAB);

  tft.setRotation(0);

  tft.fillScreen(ST77XX_BLACK);

  // Random seed
  randomSeed(
    analogRead(32)
  );

  // Calibrate joystick
  calibrateJoystick();

  // Start automatically
  startGame();
}


// ============================================================
//                           LOOP
// ============================================================

void loop() {

  // Check shooting button
  handleShootButton();

  // If game over
  if (gameState == GAME_OVER) {

    delay(10);

    return;
  }

  // Control frame rate
  if (
    millis() - lastFrame <
    FRAME_TIME
  ) {

    return;
  }

  lastFrame = millis();

  // Move player
  updatePlayer();

  // Move prey
  updatePrey();

  // Move bullets
  updateBullets();

  // Check collisions
  checkCollisions();

  // Draw game
  drawGame();
}


// ============================================================
//                   JOYSTICK CALIBRATION
// ============================================================

void calibrateJoystick() {

  long totalX = 0;
  long totalY = 0;

  Serial.println(
    "Keep joystick centered..."
  );

  for (int i = 0; i < 50; i++) {

    totalX += analogRead(JOY_X);
    totalY += analogRead(JOY_Y);

    delay(5);
  }

  centerX = totalX / 50;
  centerY = totalY / 50;

  Serial.print("Joystick X center: ");
  Serial.println(centerX);

  Serial.print("Joystick Y center: ");
  Serial.println(centerY);
}


// ============================================================
//                       START GAME
// ============================================================

void startGame() {

  gameState = PLAYING;

  score = 0;

  lives = 3;

  ammo = 12;

  level = 1;

  preyKilled = 0;

  preyAlive = 3;

  playerX = 64;
  playerY = 135;

  lastDirX = 0;
  lastDirY = -1;


  // Clear bullets
  for (
    int i = 0;
    i < MAX_BULLETS;
    i++
  ) {

    bullets[i].active = false;
  }


  // Create prey
  createPrey();

  drawGame();
}


// ============================================================
//                       CREATE PREY
// ============================================================

void createPrey() {

  for (
    int i = 0;
    i < MAX_PREY;
    i++
  ) {

    prey[i].active = false;
  }

  for (
    int i = 0;
    i < preyAlive;
    i++
  ) {

    spawnPrey(i);
  }
}


// ============================================================
//                        SPAWN PREY
// ============================================================

void spawnPrey(int index) {

  prey[index].active = true;

  bool valid = false;

  while (!valid) {

    prey[index].x =
      random(
        10,
        SCREEN_W - 10
      );

    prey[index].y =
      random(
        GAME_TOP + 10,
        GAME_BOTTOM - 10
      );

    valid = true;

    // Don't spawn near player
    if (
      distance(
        prey[index].x,
        prey[index].y,
        playerX,
        playerY
      ) < 30
    ) {

      valid = false;
    }

    // Don't spawn inside tree
    for (
      int t = 0;
      t < MAX_TREES;
      t++
    ) {

      if (
        distance(
          prey[index].x,
          prey[index].y,
          trees[t].x,
          trees[t].y
        )
        <
        trees[t].r + 8
      ) {

        valid = false;
      }
    }
  }

  // Random movement direction
  float angle =
    random(0, 360) *
    PI / 180.0;

  float speed =
    random(8, 16) / 10.0;

  prey[index].vx =
    cos(angle) * speed;

  prey[index].vy =
    sin(angle) * speed;

  prey[index].type =
    random(0, 3);

  prey[index].changeDirectionTime =
    millis() +
    random(700, 1800);
}


// ============================================================
//                       UPDATE PLAYER
// ============================================================

void updatePlayer() {

  int rawX =
    analogRead(JOY_X);

  int rawY =
    analogRead(JOY_Y);

  int dx =
    rawX - centerX;

  int dy =
    rawY - centerY;


  // Dead zone
  if (
    abs(dx) <
    JOY_DEADZONE
  ) {

    dx = 0;
  }

  if (
    abs(dy) <
    JOY_DEADZONE
  ) {

    dy = 0;
  }


  float moveX =
    dx / 2048.0;

  float moveY =
    dy / 2048.0;


  float magnitude =
    sqrt(
      moveX * moveX +
      moveY * moveY
    );


  if (
    magnitude > 0.1
  ) {

    // Normalize
    moveX /= magnitude;
    moveY /= magnitude;


    // Save shooting direction
    lastDirX = moveX;
    lastDirY = moveY;


    float speed = 2.5;


    float newX =
      playerX +
      moveX * speed;

    float newY =
      playerY +
      moveY * speed;


    // Screen boundaries

    if (newX < 6)
      newX = 6;

    if (
      newX >
      SCREEN_W - 6
    )
      newX =
        SCREEN_W - 6;

    if (
      newY <
      GAME_TOP + 6
    )
      newY =
        GAME_TOP + 6;

    if (
      newY >
      GAME_BOTTOM - 6
    )
      newY =
        GAME_BOTTOM - 6;


    // Tree collision
    if (
      !insideTree(
        newX,
        newY
      )
    ) {

      playerX = newX;
      playerY = newY;
    }
  }
}


// ============================================================
//                        UPDATE PREY
// ============================================================

void updatePrey() {

  for (
    int i = 0;
    i < MAX_PREY;
    i++
  ) {

    if (
      !prey[i].active
    ) {

      continue;
    }


    // Random direction change
    if (
      millis() >
      prey[i].changeDirectionTime
    ) {

      float angle =
        random(0, 360) *
        PI / 180.0;

      float speed =
        random(8, 16) / 10.0;

      prey[i].vx =
        cos(angle) * speed;

      prey[i].vy =
        sin(angle) * speed;

      prey[i].changeDirectionTime =
        millis() +
        random(700, 1800);
    }


    float newX =
      prey[i].x +
      prey[i].vx;

    float newY =
      prey[i].y +
      prey[i].vy;


    // Bounce off screen

    if (
      newX < 7 ||
      newX >
      SCREEN_W - 7
    ) {

      prey[i].vx *= -1;

      newX =
        prey[i].x +
        prey[i].vx;
    }


    if (
      newY <
      GAME_TOP + 7 ||
      newY >
      GAME_BOTTOM - 7
    ) {

      prey[i].vy *= -1;

      newY =
        prey[i].y +
        prey[i].vy;
    }


    // Tree collision

    if (
      insideTree(
        newX,
        newY
      )
    ) {

      prey[i].vx *= -1;
      prey[i].vy *= -1;

    } else {

      prey[i].x = newX;
      prey[i].y = newY;
    }


    // Prey hits hunter

    if (
      distance(
        prey[i].x,
        prey[i].y,
        playerX,
        playerY
      ) < 9
    ) {

      loseLife(i);
    }
  }
}


// ============================================================
//                     SHOOT BUTTON
// ============================================================

void handleShootButton() {

  bool currentState =
    digitalRead(
      SHOOT_BUTTON
    );


  // Button pressed
  if (
    lastShootButton == HIGH &&
    currentState == LOW
  ) {

    if (
      millis() -
      lastShootTime >
      DEBOUNCE
    ) {

      lastShootTime =
        millis();


      // Shoot during game
      if (
        gameState ==
        PLAYING
      ) {

        shoot();
      }


      // Restart after game over
      else if (
        gameState ==
        GAME_OVER
      ) {

        startGame();
      }
    }
  }


  lastShootButton =
    currentState;
}


// ============================================================
//                          SHOOT
// ============================================================

void shoot() {

  if (ammo <= 0) {

    return;
  }


  int slot = -1;


  // Find free bullet
  for (
    int i = 0;
    i < MAX_BULLETS;
    i++
  ) {

    if (
      !bullets[i].active
    ) {

      slot = i;

      break;
    }
  }


  if (slot == -1) {

    return;
  }


  ammo--;


  bullets[slot].active = true;

  bullets[slot].x =
    playerX;

  bullets[slot].y =
    playerY;


  float bulletSpeed =
    6.0;


  bullets[slot].vx =
    lastDirX *
    bulletSpeed;

  bullets[slot].vy =
    lastDirY *
    bulletSpeed;
}


// ============================================================
//                     UPDATE BULLETS
// ============================================================

void updateBullets() {

  for (
    int i = 0;
    i < MAX_BULLETS;
    i++
  ) {

    if (
      !bullets[i].active
    ) {

      continue;
    }


    bullets[i].x +=
      bullets[i].vx;

    bullets[i].y +=
      bullets[i].vy;


    // Outside screen
    if (
      bullets[i].x < 0 ||
      bullets[i].x >= SCREEN_W ||
      bullets[i].y < GAME_TOP ||
      bullets[i].y >= SCREEN_H
    ) {

      bullets[i].active =
        false;

      continue;
    }


    // Bullet hits tree
    if (
      insideTree(
        bullets[i].x,
        bullets[i].y
      )
    ) {

      bullets[i].active =
        false;
    }
  }
}


// ============================================================
//                     COLLISION CHECK
// ============================================================

void checkCollisions() {

  for (
    int b = 0;
    b < MAX_BULLETS;
    b++
  ) {

    if (
      !bullets[b].active
    ) {

      continue;
    }


    for (
      int p = 0;
      p < MAX_PREY;
      p++
    ) {

      if (
        !prey[p].active
      ) {

        continue;
      }


      float d =
        distance(
          bullets[b].x,
          bullets[b].y,
          prey[p].x,
          prey[p].y
        );


      if (
        d <
        PREY_RADIUS +
        BULLET_RADIUS +
        2
      ) {

        // HIT

        bullets[b].active =
          false;

        prey[p].active =
          false;


        score +=
          10 * level;

        preyKilled++;


        // Explosion
        showHit(
          prey[p].x,
          prey[p].y
        );


        // Check if all prey killed

        bool allDead = true;


        for (
          int j = 0;
          j < MAX_PREY;
          j++
        ) {

          if (
            prey[j].active
          ) {

            allDead = false;
          }
        }


        if (allDead) {

          nextLevel();
        }
      }
    }
  }
}


// ============================================================
//                       NEXT LEVEL
// ============================================================

void nextLevel() {

  level++;


  // Complete after level 5
  if (
    level > 5
  ) {

    victory();

    return;
  }


  // More prey
  preyAlive =
    2 + level;


  if (
    preyAlive >
    MAX_PREY
  ) {

    preyAlive =
      MAX_PREY;
  }


  // More ammo
  ammo += 6;


  createPrey();


  showLevelScreen();


  delay(700);
}


// ============================================================
//                       LOSE LIFE
// ============================================================

void loseLife(
  int preyIndex
) {

  prey[preyIndex].active =
    false;


  lives--;


  if (
    lives <= 0
  ) {

    gameOver();

    return;
  }


  // Respawn prey
  spawnPrey(
    preyIndex
  );


  // Reset player
  playerX = 64;
  playerY = 135;


  showLifeLost();


  delay(400);
}


// ============================================================
//                       DRAW GAME
// ============================================================

void drawGame() {

  tft.fillScreen(
    ST77XX_BLACK
  );


  // ---------------- HUD ----------------

  tft.setTextSize(1);


  // Score

  tft.setTextColor(
    ST77XX_YELLOW
  );

  tft.setCursor(2, 2);

  tft.print("S:");

  tft.print(score);


  // Health

  tft.setTextColor(
    ST77XX_RED
  );

  tft.setCursor(43, 2);

  tft.print("HP:");

  tft.print(lives);


  // Level

  tft.setTextColor(
    ST77XX_WHITE
  );

  tft.setCursor(78, 2);

  tft.print("LV:");

  tft.print(level);


  // Ammo

  tft.setCursor(2, 10);

  tft.setTextColor(
    ST77XX_CYAN
  );

  tft.print("AMMO:");

  tft.print(ammo);


  // Game border

  tft.drawLine(
    0,
    GAME_TOP - 1,
    SCREEN_W - 1,
    GAME_TOP - 1,
    ST77XX_WHITE
  );


  // Trees

  drawTrees();


  // Prey

  drawPrey();


  // Bullets

  drawBullets();


  // Hunter

  drawPlayer();
}


// ============================================================
//                       DRAW TREES
// ============================================================

void drawTrees() {

  for (
    int i = 0;
    i < MAX_TREES;
    i++
  ) {

    // Tree trunk

    tft.fillRect(
      trees[i].x - 2,
      trees[i].y,
      5,
      8,
      ST77XX_YELLOW
    );


    // Leaves

    tft.fillCircle(
      trees[i].x,
      trees[i].y,
      trees[i].r,
      ST77XX_GREEN
    );
  }
}


// ============================================================
//                       DRAW PLAYER
// ============================================================

void drawPlayer() {

  int x =
    (int)playerX;

  int y =
    (int)playerY;


  // Hunter body

  tft.fillCircle(
    x,
    y,
    PLAYER_RADIUS,
    ST77XX_WHITE
  );


  // Hat

  tft.fillRect(
    x - 5,
    y - 7,
    10,
    2,
    ST77XX_GREEN
  );


  // Gun

  int gunX =
    x +
    lastDirX * 8;

  int gunY =
    y +
    lastDirY * 8;


  tft.drawLine(
    x,
    y,
    gunX,
    gunY,
    ST77XX_YELLOW
  );


  // Aim point

  tft.drawPixel(
    gunX,
    gunY,
    ST77XX_RED
  );
}


// ============================================================
//                       DRAW PREY
// ============================================================

void drawPrey() {

  for (
    int i = 0;
    i < MAX_PREY;
    i++
  ) {

    if (
      !prey[i].active
    ) {

      continue;
    }


    int x =
      (int)prey[i].x;

    int y =
      (int)prey[i].y;


    // Body

    tft.fillCircle(
      x,
      y,
      PREY_RADIUS,
      ST77XX_WHITE
    );


    // Rabbit

    if (
      prey[i].type == 0
    ) {

      tft.drawLine(
        x - 3,
        y - 4,
        x - 2,
        y - 8,
        ST77XX_WHITE
      );

      tft.drawLine(
        x + 3,
        y - 4,
        x + 2,
        y - 8,
        ST77XX_WHITE
      );
    }


    // Small animal

    else if (
      prey[i].type == 1
    ) {

      tft.drawPixel(
        x - 5,
        y - 2,
        ST77XX_WHITE
      );

      tft.drawPixel(
        x + 5,
        y - 2,
        ST77XX_WHITE
      );
    }


    // Bird

    else {

      tft.drawLine(
        x - 5,
        y,
        x - 8,
        y - 3,
        ST77XX_WHITE
      );

      tft.drawLine(
        x + 5,
        y,
        x + 8,
        y - 3,
        ST77XX_WHITE
      );
    }
  }
}


// ============================================================
//                       DRAW BULLETS
// ============================================================

void drawBullets() {

  for (
    int i = 0;
    i < MAX_BULLETS;
    i++
  ) {

    if (
      !bullets[i].active
    ) {

      continue;
    }


    tft.fillCircle(
      (int)bullets[i].x,
      (int)bullets[i].y,
      BULLET_RADIUS,
      ST77XX_YELLOW
    );
  }
}


// ============================================================
//                        DISTANCE
// ============================================================

float distance(
  float x1,
  float y1,
  float x2,
  float y2
) {

  float dx =
    x1 - x2;

  float dy =
    y1 - y2;


  return sqrt(
    dx * dx +
    dy * dy
  );
}


// ============================================================
//                     TREE COLLISION
// ============================================================

bool insideTree(
  float x,
  float y
) {

  for (
    int i = 0;
    i < MAX_TREES;
    i++
  ) {

    float d =
      distance(
        x,
        y,
        trees[i].x,
        trees[i].y
      );


    if (
      d <
      trees[i].r + 5
    ) {

      return true;
    }
  }


  return false;
}


// ============================================================
//                       HIT EFFECT
// ============================================================

void showHit(
  float x,
  float y
) {

  for (
    int r = 2;
    r <= 7;
    r += 2
  ) {

    tft.drawCircle(
      (int)x,
      (int)y,
      r,
      ST77XX_RED
    );

    delay(15);
  }
}


// ============================================================
//                       LEVEL SCREEN
// ============================================================

void showLevelScreen() {

  tft.fillScreen(
    ST77XX_BLACK
  );


  tft.setTextSize(2);

  tft.setTextColor(
    ST77XX_GREEN
  );


  tft.setCursor(
    25,
    45
  );

  tft.println(
    "LEVEL"
  );


  tft.setCursor(
    52,
    70
  );

  tft.println(
    level
  );


  tft.setTextSize(1);

  tft.setTextColor(
    ST77XX_WHITE
  );


  tft.setCursor(
    20,
    100
  );

  tft.println(
    "NEW PREY!"
  );
}


// ============================================================
//                     LIFE LOST
// ============================================================

void showLifeLost() {

  tft.fillScreen(
    ST77XX_BLACK
  );


  tft.setTextSize(2);

  tft.setTextColor(
    ST77XX_RED
  );


  tft.setCursor(
    25,
    45
  );

  tft.println(
    "OUCH!"
  );


  tft.setTextSize(1);

  tft.setTextColor(
    ST77XX_WHITE
  );


  tft.setCursor(
    25,
    80
  );

  tft.print(
    "LIVES: "
  );

  tft.println(
    lives
  );
}


// ============================================================
//                      GAME OVER
// ============================================================

void gameOver() {

  gameState =
    GAME_OVER;


  tft.fillScreen(
    ST77XX_BLACK
  );


  tft.setTextSize(2);

  tft.setTextColor(
    ST77XX_RED
  );


  tft.setCursor(
    20,
    30
  );

  tft.println(
    "GAME"
  );


  tft.setCursor(
    20,
    53
  );

  tft.println(
    "OVER"
  );


  tft.setTextSize(1);

  tft.setTextColor(
    ST77XX_YELLOW
  );


  tft.setCursor(
    30,
    85
  );

  tft.print(
    "SCORE: "
  );

  tft.println(
    score
  );


  tft.setTextColor(
    ST77XX_WHITE
  );


  tft.setCursor(
    15,
    115
  );

  tft.println(
    "PRESS BUTTON"
  );


  tft.setCursor(
    20,
    130
  );

  tft.println(
    "TO RESTART"
  );
}


// ============================================================
//                        VICTORY
// ============================================================

void victory() {

  gameState =
    GAME_OVER;


  tft.fillScreen(
    ST77XX_BLACK
  );


  tft.setTextSize(2);

  tft.setTextColor(
    ST77XX_GREEN
  );


  tft.setCursor(
    20,
    40
  );

  tft.println(
    "YOU WIN!"
  );


  tft.setTextSize(1);

  tft.setTextColor(
    ST77XX_YELLOW
  );


  tft.setCursor(
    28,
    80
  );

  tft.print(
    "SCORE: "
  );

  tft.println(
    score
  );


  tft.setTextColor(
    ST77XX_WHITE
  );


  tft.setCursor(
    15,
    110
  );

  tft.println(
    "PRESS BUTTON"
  );


  tft.setCursor(
    25,
    125
  );

  tft.println(
    "TO PLAY AGAIN"
  );
}
```
