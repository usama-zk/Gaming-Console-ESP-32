#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <WiFi.h>
#include <HTTPClient.h>

const char *ssid = "";        // Replace with your WiFi SSID
const char *password = ""; // Replace with your WiFi password
const char *server = "http://api.thingspeak.com/update";
const char *apiKey = ""; // Replace with your ThingSpeak API key


#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64

#define OLED_RESET -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Joystick pins
#define VRX_PIN  32
#define VRY_PIN  33
#define SW_PIN   34 // Joystick button pin
#define BUZZER_PIN 25

// Game selection variables
int selectedGame = 0; // 0: Brick Breaker, 1: Snake
bool gameStarted = false;

// Game parameters shared across games
#define PADDLE_WIDTH 20
#define PADDLE_HEIGHT 3
#define BALL_SIZE 3
#define SNAKE_SIZE 4
#define BRICK_WIDTH 20
#define BRICK_HEIGHT 5
#define NUM_ROWS 3
#define NUM_COLS 6
#define INITIAL_LENGTH 5
#define MAX_LENGTH 50

// Brick Breaker variables
int paddleX = (SCREEN_WIDTH - PADDLE_WIDTH) / 2;
int ballX = SCREEN_WIDTH / 2, ballY = SCREEN_HEIGHT / 2;
int ballDX = 1, ballDY = -1; // Ball direction
bool bricks[NUM_ROWS][NUM_COLS];
int brickBreakerScore = 0;
bool brickBreakerGameOver = false;

// Snake variables
int snakeX[MAX_LENGTH], snakeY[MAX_LENGTH];
int snakeLength = INITIAL_LENGTH;
int foodX, foodY;
int snakeDirection = 1; // 0: up, 1: right, 2: down, 3: left
bool snakeGameOver = false;

// Joystick thresholds
#define LEFT_THRESHOLD  1000
#define RIGHT_THRESHOLD 4000
#define DOWN_THRESHOLD  1000
#define UP_THRESHOLD    4000

void setup() {
  pinMode(SW_PIN, INPUT_PULLUP); // Joystick button input
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW); // Ensure it's off initially

  // Initialize OLED display
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println(F("SSD1306 allocation failed"));
    for (;;);
  }

  display.clearDisplay();
  display.display();

  Serial.begin(9600); // For debugging
  connectToWiFi();

  startAnimation();
  initializeBrickBreaker();
  initializeSnake();
}

void connectToWiFi() {
  Serial.print("Connecting to WiFi");
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("\nConnected to WiFi");
}


void loop() {
  if (!gameStarted) {
    displayMenu();
    handleMenuInput();
  } else {
    if (selectedGame == 0) {
      brickBreakerLoop();
    } else if (selectedGame == 1) {
      snakeLoop();
    }
  }
}

// New mode variable: 0 for Easy, 1 for Survival
int selectedMode = 0;

// === MENU FUNCTIONS ===
void displayMenu() {
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);

  display.setCursor(10, 10);
  display.println(F("Select Game:"));

  // Highlight current selection
  display.setCursor(20, 30);
  display.println(selectedGame == 0 ? "> Brick Breaker" : "  Brick Breaker");

  display.setCursor(20, 50);
  display.println(selectedGame == 1 ? "> Snake" : "  Snake");

  display.display();

  // Optional: Beep on navigation
  static int previousGame = -1;
  if (selectedGame != previousGame) {
    ringBuzzer(50); // Beep for feedback
    previousGame = selectedGame;
  }
}

void handleMenuInput() {
  static unsigned long lastDebounceTime = 0; // Debounce timer
  const unsigned long debounceDelay = 200;  // Debounce delay (200ms)

  int joystickY = analogRead(VRY_PIN);

  // Navigate menu if debounce delay has passed
  if (millis() - lastDebounceTime > debounceDelay) {
    if (joystickY < DOWN_THRESHOLD) { 
      selectedGame = 1; // Select Snake
      if (digitalRead(SW_PIN) == LOW) { // Joystick button pressed (LOW when pressed)
      delay(200);         // Additional debounce delay for the button press
      modeSelectionMenu();
      gameStarted = true; // Start the selected game
      lastDebounceTime = millis();
      } // Reset debounce timer
    } else if (joystickY > UP_THRESHOLD) {
      selectedGame = 0; // Select Brick Breaker
      if (digitalRead(SW_PIN) == LOW) { // Joystick button pressed (LOW when pressed)
      delay(200);         // Additional debounce delay for the button press
      modeSelectionMenu();
      gameStarted = true; // Start the selected game
      lastDebounceTime = millis(); // Reset debounce timer
      }
    }
  }
}

void modeSelectionMenu() {
  while (true) {
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);

    display.setCursor(10, 10);
    display.println(F("Select Mode:"));

    // Highlight current mode selection
    display.setCursor(20, 30);
    display.println(selectedMode == 0 ? "> Easy" : "  Easy");

    display.setCursor(20, 50);
    display.println(selectedMode == 1 ? "> Survival" : "  Survival");

    display.display();

    int joystickY = analogRead(VRY_PIN);

    // Navigate between modes with debounce delay
    static unsigned long lastChangeTime = 0;
    const unsigned long debounceDelay = 500; // 300ms delay for mode switching

    if (millis() - lastChangeTime > debounceDelay) {
      if (joystickY < DOWN_THRESHOLD) { // Moving down
        selectedMode = 1; // Survival
        if (digitalRead(SW_PIN) == LOW) { // Button pressed
          delay(500); // Debounce delay for button press
          gameStarted = true; // Start the selected game with the chosen mode
          break;
        }
        lastChangeTime = millis(); // Update debounce time
      } else if (joystickY > UP_THRESHOLD) { // Moving up
        selectedMode = 0; // Easy
        if (digitalRead(SW_PIN) == LOW) { // Button pressed
          delay(500); // Debounce delay for button press
          gameStarted = true; // Start the selected game with the chosen mode
          break;
        }
        lastChangeTime = millis(); // Update debounce time
      }
    }
    delay(50); // Small delay to ensure smooth input handling
  }
}




// === BRICK BREAKER FUNCTIONS ===
void initializeBrickBreaker() {
  // Initialize bricks
  for (int i = 0; i < NUM_ROWS; i++) {
    for (int j = 0; j < NUM_COLS; j++) {
      bricks[i][j] = true;
    }
  }

  paddleX = (SCREEN_WIDTH - PADDLE_WIDTH) / 2;
  ballX = SCREEN_WIDTH / 2;
  ballY = SCREEN_HEIGHT / 2;
  ballDX = 1;
  ballDY = -1;
  brickBreakerScore = 0;
  brickBreakerGameOver = false;
}

void brickBreakerLoop() {
 if (brickBreakerGameOver) {
    displayGameOver(brickBreakerScore, "Brick Breaker");
    return;
  }

  handleJoystickForPaddle();
  moveBall();

  // Adjust speed based on mode
  int delayTime = selectedMode == 0 ? 20 : 5; // Easy: slower, Survival: faster
  delay(delayTime);

  checkBrickBreakerCollisions();
  drawBrickBreakerGame();
}

void handleJoystickForPaddle() {
  int joystickX = analogRead(VRX_PIN);
  if (joystickX < 1000) {
    paddleX -= 2; // Move left
  } else if (joystickX > 4000) {
    paddleX += 2; // Move right
  }

  // Constrain paddle within screen bounds
  if (paddleX < 0) paddleX = 0;
  if (paddleX > SCREEN_WIDTH - PADDLE_WIDTH) paddleX = SCREEN_WIDTH - PADDLE_WIDTH;
}

void moveBall() {
  ballX += ballDX;
  ballY += ballDY;

  // Bounce off walls
  if (ballX <= 0 || ballX >= SCREEN_WIDTH - BALL_SIZE) {
    ballDX = -ballDX;
  }
  if (ballY <= 0) {
    ballDY = -ballDY;
  }

  // Check if the ball falls off the screen
  if (ballY > SCREEN_HEIGHT) {
    ringBuzzer(500); // Long beep for game over
    brickBreakerGameOver = true;
  }
}

void checkBrickBreakerCollisions() {
  // Paddle collision
  if (ballY + BALL_SIZE >= SCREEN_HEIGHT - PADDLE_HEIGHT &&
      ballX + BALL_SIZE >= paddleX &&
      ballX <= paddleX + PADDLE_WIDTH) {
    ballDY = -ballDY;
  }

  // Brick collision
  for (int i = 0; i < NUM_ROWS; i++) {
    for (int j = 0; j < NUM_COLS; j++) {
      if (bricks[i][j]) {
        int brickX = j * BRICK_WIDTH;
        int brickY = i * BRICK_HEIGHT;
        if (ballX + BALL_SIZE >= brickX &&
            ballX <= brickX + BRICK_WIDTH &&
            ballY + BALL_SIZE >= brickY &&
            ballY <= brickY + BRICK_HEIGHT) {
          bricks[i][j] = false;
          ballDY = -ballDY;
          brickBreakerScore++;
          ringBuzzer(100); // Short beep for scoring
        }
      }
    }
  }
}

void drawBrickBreakerGame() {
  display.clearDisplay();

  // Draw paddle
  display.fillRect(paddleX, SCREEN_HEIGHT - PADDLE_HEIGHT, PADDLE_WIDTH, PADDLE_HEIGHT, SSD1306_WHITE);

  // Draw ball
  display.fillRect(ballX, ballY, BALL_SIZE, BALL_SIZE, SSD1306_WHITE);

  // Draw bricks
  for (int i = 0; i < NUM_ROWS; i++) {
    for (int j = 0; j < NUM_COLS; j++) {
      if (bricks[i][j]) {
        display.fillRect(j * BRICK_WIDTH, i * BRICK_HEIGHT, BRICK_WIDTH, BRICK_HEIGHT, SSD1306_WHITE);
      }
    }
  }

  // Draw score
  display.setCursor(0, 0);
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.print(F("Score: "));
  display.print(brickBreakerScore);

  display.display();
}

// === SNAKE FUNCTIONS ===
void initializeSnake() {
  for (int i = 0; i < snakeLength; i++) {
    snakeX[i] = 64 - i * SNAKE_SIZE;
    snakeY[i] = 32;
  }

  spawnFood();
  snakeLength = INITIAL_LENGTH;
  snakeDirection = 1; // Right
  snakeGameOver = false;
}

void snakeLoop() {
  if (snakeGameOver) {
    displayGameOver(snakeLength - INITIAL_LENGTH, "Snake");
    return;
  }

  readJoystickForSnake();
  moveSnake();

  // Adjust speed based on mode
  int delayTime = selectedMode == 0 ? 100 : 50; // Easy: slower, Survival: faster
  delay(delayTime);

  checkSnakeCollisions();
  drawSnakeGame();
}

void readJoystickForSnake() {
  int valueX = analogRead(VRX_PIN);
  int valueY = analogRead(VRY_PIN);

  if (valueX < LEFT_THRESHOLD && snakeDirection != 1) snakeDirection = 3; // Left
  if (valueX > RIGHT_THRESHOLD && snakeDirection != 3) snakeDirection = 1; // Right
  if (valueY < DOWN_THRESHOLD && snakeDirection != 0) snakeDirection = 2; // Down
  if (valueY > UP_THRESHOLD && snakeDirection != 2) snakeDirection = 0; // Up
}

void moveSnake() {
  for (int i = snakeLength - 1; i > 0; i--) {
    snakeX[i] = snakeX[i - 1];
    snakeY[i] = snakeY[i - 1];
  }

  switch (snakeDirection) {
    case 0: snakeY[0] -= SNAKE_SIZE; break; // Up
    case 1: snakeX[0] += SNAKE_SIZE; break; // Right
    case 2: snakeY[0] += SNAKE_SIZE; break; // Down
    case 3: snakeX[0] -= SNAKE_SIZE; break; // Left
  }
}

void checkSnakeCollisions() {
  if (snakeX[0] < 0 || snakeX[0] >= SCREEN_WIDTH || snakeY[0] < 0 || snakeY[0] >= SCREEN_HEIGHT) {
    ringBuzzer(500);
    snakeGameOver = true;
  }

  for (int i = 1; i < snakeLength; i++) {
    if (snakeX[0] == snakeX[i] && snakeY[0] == snakeY[i]) {
      ringBuzzer(500);
      snakeGameOver = true;
      break;
    }
  }

  if (snakeX[0] == foodX && snakeY[0] == foodY) {
    if (snakeLength < MAX_LENGTH) snakeLength++;
    spawnFood();
    ringBuzzer(100);
  }
}

void drawSnakeGame() {
  display.clearDisplay();

  for (int i = 0; i < snakeLength; i++) {
    display.fillRect(snakeX[i], snakeY[i], SNAKE_SIZE, SNAKE_SIZE, SSD1306_WHITE);
  }

  display.fillRect(foodX, foodY, SNAKE_SIZE, SNAKE_SIZE, SSD1306_WHITE);

  display.setCursor(0, 0);
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.print(F("Score: "));
  display.print(snakeLength - INITIAL_LENGTH);

  display.display();
}

void spawnFood() {
  foodX = random(0, SCREEN_WIDTH / SNAKE_SIZE) * SNAKE_SIZE;
  foodY = random(0, SCREEN_HEIGHT / SNAKE_SIZE) * SNAKE_SIZE;
}

// === COMMON FUNCTIONS ===
void ringBuzzer(int duration) {
  digitalWrite(BUZZER_PIN, HIGH);
  delay(duration);
  digitalWrite(BUZZER_PIN, LOW);
}

void displayGameOver(int score, const char *gameName) {
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(10, 20);
  display.println(F("GAME OVER"));

  display.setCursor(10, 40);
  display.print(F("Game: "));
  display.println(gameName);

  display.setCursor(10, 50);
  display.print(F("Score: "));
  display.print(score);

  display.display();

  // Send score to ThingSpeak
  sendScoreToThingSpeak(score, gameName);
  delay(2000); // 2 seconds for game over screen

  endAnimation(gameName, score);
  gameStarted = false;  // Return to the menu
  displayMenu();
}

void startAnimation() {
  display.clearDisplay();
  display.setTextSize(2);
  display.setTextColor(SSD1306_WHITE);

  // Bouncing text animation
  int xPos = 0;
  while (xPos < SCREEN_WIDTH) {
    display.clearDisplay();
    display.setCursor(xPos, SCREEN_HEIGHT / 2 - 10);
    display.println(F("Let's Play!"));
    display.display();
    delay(50);
    xPos += 5;
  }

  // Pause before starting the menu
  delay(500);
}

void endAnimation(const char *gameName, int score) {
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);

  // "Game Over" Text Effect
  for (int i = 0; i < 5; i++) {
    display.setCursor(10, 20);
    display.print(F("GAME OVER"));
    display.display();
    delay(300);
    display.clearDisplay();
    display.display();
    delay(300);
  }

  // Sad Emoji Animation
  display.clearDisplay();
  display.setTextSize(2);
  display.setCursor((SCREEN_WIDTH - 12) / 2, (SCREEN_HEIGHT - 16) / 2);
  display.println(F(":("));
  display.display();

  // Sad Buzzer Sound
  for (int i = 0; i < 3; i++) {
    digitalWrite(BUZZER_PIN, HIGH);
    delay(300);
    digitalWrite(BUZZER_PIN, LOW);
    delay(200);
  }

  // Display Final Score
  display.clearDisplay();
  display.setTextSize(1);
  display.setCursor(10, 30);
  display.print(F("Game: "));
  display.println(gameName);

  display.setCursor(10, 40);
  display.print(F("Score: "));
  display.println(score);
  display.display();

  // Pause and return to the menu
  delay(3000);
}

void sendScoreToThingSpeak(int score, const char *gameName) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    String url = String(server) + "?api_key=" + apiKey;

    if (strcmp(gameName, "Brick Breaker") == 0) {
      url += "&field1=" + String(score); // Field 1 for Brick Breaker
    } else if (strcmp(gameName, "Snake") == 0) {
      url += "&field2=" + String(score); // Field 2 for Snake
    }
    url += "&field3=" + String(score); // Combined score

    http.begin(url);
    int httpResponseCode = http.GET();
    if (httpResponseCode > 0) {
      Serial.println("Score sent to ThingSpeak");
    } else {
      Serial.println("Error sending data to ThingSpeak");
    }
    http.end();
  } else {
    Serial.println("WiFi not connected. Unable to send score.");
  }
}
