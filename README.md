#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <FastLED.h>

// LCD setup
LiquidCrystal_I2C lcd(0x27, 20, 4);

// Pins for the arcade buttons
const int trueButtonPin = 2;
const int falseButtonPin = 3;

// Pin for WS2811B pixel string lights
const int pixelPin = 6;

// Number of pixels
const int numPixels = 24;

// Create a CRGB array for the pixels
CRGB pixels[numPixels];

// Colors in RGB format
#define RAINBOW_INTERVAL 10 // Time interval for faster rainbow effect

// Define the questions and answers
struct Question {
  String text;
  bool answer;
  String correctAnswer;
};

Question questions[] = {
  {"The Beatles were originally called The Quarrymen.", true, "True"},
  {"Elvis Presley was known as the 'King of Pop'.", false, "False, he was known as the 'King of Rock and Roll'."},
  {"The movie 'Titanic' won 11 Academy Awards.", true, "True"},
  {"The character Harry Potter has a lightning-shaped scar on his forehead.", true, "True"},
  {"The TV show 'Friends' is set in Chicago.", false, "False, it is set in New York City."},
  {"Lady Gaga's real name is Stefani Joanne Angelina Germanotta.", true, "True"},
  {"The movie 'Inception' was directed by Steven Spielberg.", false, "False, it was directed by Christopher Nolan."},
  {"The song 'Rolling in the Deep' was performed by Beyonce.", false, "False, it was performed by Adele."},
  {"The character Thor in the Marvel movies is played by Chris Hemsworth.", true, "True"},
  {"The band ABBA is from Germany.", false, "False, they are from Sweden."},
  {"The movie 'The Lion King' was released in 1994.", true, "True"},
  {"The TV show 'Breaking Bad' is about a chemistry teacher turned drug dealer.", true, "True"},
  {"The band Nirvana originated in Seattle, Washington.", true, "True"},
  {"The movie 'Avatar' is set on a planet called Pandora.", true, "True"},
  {"The singer known as 'The Weekend' is from Canada.", true, "True"},
  {"The character Wonder Woman is an Amazonian princess.", true, "True"},
  {"The TV show 'Stranger Things' is set in the 1980s.", true, "True"},
  {"The movie 'Jurassic Park' was based on a novel by Michael Crichton.", true, "True"},
  {"The song 'Hotel California' was performed by The Eagles.", true, "True"},
  {"The character James Bond is also known by the code number 008.", false, "False, he is known by the code number 007."},
  {"The movie 'The Matrix' stars Keanu Reeves as the main character.", true, "True"},
  {"The band Queen's lead singer was David Bowie.", false, "False, the lead singer was Freddie Mercury."},
  {"The movie 'Frozen' features a song called 'Let It Go' performed by Idina Menzel.", true, "True"},
  {"The character Luke Skywalker is from the 'Star Wars' franchise.", true, "True"}
};

// Variables to track the game state
int currentQuestion = 0;
int score = 0;
bool displayingResult = false;
unsigned long resultDisplayTime = 0;
unsigned long lastScrollTime = 0;
int scrollIndex = 0;
bool gameActive = false;

// Button state variables
bool lastTrueButtonState = HIGH;
bool lastFalseButtonState = HIGH;

void setup() {
  // Initialize the LCD
  lcd.init();
  lcd.backlight();
  // Set the button pins as input with pull-up resistors
  pinMode(trueButtonPin, INPUT_PULLUP);
  pinMode(falseButtonPin, INPUT_PULLUP);

  // Initialize the FastLED library with RGB color order
  FastLED.addLeds<WS2811, pixelPin, RGB>(pixels, numPixels);
  FastLED.show(); // Initialize all pixels to 'off'

  // Display the ready screen
  displayReadyScreen();
}

void loop() {
  // Check for button presses to start the game
  if (!gameActive && (digitalRead(trueButtonPin) == LOW || digitalRead(falseButtonPin) == LOW)) {
    gameActive = true;
    currentQuestion = 0;
    score = 0;
    displayQuestion();
    delay(500); // Debounce delay
  }

  // Check for button presses during the game
  if (gameActive) {
    bool currentTrueButtonState = digitalRead(trueButtonPin);
    bool currentFalseButtonState = digitalRead(falseButtonPin);

    if (currentTrueButtonState == LOW && lastTrueButtonState == HIGH) {
      checkAnswer(true);
      delay(500); // Debounce delay
    }
    if (currentFalseButtonState == LOW && lastFalseButtonState == HIGH) {
      checkAnswer(false);
      delay(500); // Debounce delay
    }

    lastTrueButtonState = currentTrueButtonState;
    lastFalseButtonState = currentFalseButtonState;

    // Handle displaying results
    if (displayingResult && millis() - resultDisplayTime >= 2000) {
      displayingResult = false;
      if (currentQuestion < 24) {
        displayQuestion();
      } else {
        displayFinalScore();
      }
    }

    // Scroll the question text if not displaying results
    if (!displayingResult && currentQuestion < 24) {
      scrollQuestion();
    }
  }
}

void displayQuestion() {
  lcd.clear();
  scrollIndex = 0;
  lastScrollTime = millis();
}

void scrollQuestion() {
  if (millis() - lastScrollTime >= 500) { // Adjust scroll speed here (slower)
    lastScrollTime = millis();
    String text = questions[currentQuestion].text + " ";
    for (int row = 0; row < 4; row++) {
      lcd.setCursor(0, row);
      lcd.print(text.substring(scrollIndex + row * 20, scrollIndex + (row + 1) * 20));
    }
    scrollIndex++;
    if (scrollIndex >= text.length()) {
      scrollIndex = 0;
    }
  }
}

void checkAnswer(bool playerAnswer) {
  displayingResult = true;
  resultDisplayTime = millis();

  if (playerAnswer == questions[currentQuestion].answer) {
    score++;
    lcd.clear();
    lcd.setCursor(0, 2);
    lcd.print("Correct! Score: ");
    lcd.print(score);
    setPixelColor(score - 1); // Light up the corresponding LED
  } else {
    lcd.clear();
    lcd.setCursor(0, 2);
    lcd.print("Wrong! ");
    lcd.setCursor(0, 3);
    lcd.print(questions[currentQuestion].correctAnswer);
  }

  // Move to the next question
  currentQuestion++;
  if (currentQuestion >= 24) {
    displayFinalScore();
  }
}

void setPixelColor(int index) {
  pixels[index] = CHSV(random(0, 255), 255, 255); // Set a random color (full spectrum, no white)
  FastLED.show();
}

void displayFinalScore() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Game Over!");
  lcd.setCursor(0, 1);
  lcd.print("Final Score: ");
  lcd.print(score);
  lcd.print("/24");

  if (score == 24) {
    flashRainbow();
  } else {
    delay(10000); // Show the game over screen for 10 seconds
    displayReadyScreen();
  }
}

void flashRainbow() {
  unsigned long start = millis();
  while (millis() - start < 10000) { // Lasts for 10 seconds
    for (int i = 0; i < numPixels; i++) {
      pixels[i] = CHSV((millis() / RAINBOW_INTERVAL + i * 10) % 255, 255, 255); // Light chase effect
    }
    FastLED.show();
    delay(50); // Adjust for chase speed
  }
  FastLED.clear();
  FastLED.show();
  displayReadyScreen();
}

void displayReadyScreen() {
  lcd.clear();
  scrollIndex = 0;
  gameActive = false;
  while (!gameActive) {
    if (millis() - lastScrollTime >= 300) { // Adjust scroll speed here
      lastScrollTime = millis();
      String text = "Press a button to play! ";
      lcd.setCursor(0, 1);
      lcd.print(text.substring(scrollIndex, scrollIndex + 20));
      scrollIndex++;
      if (scrollIndex >= text.length()) {
        scrollIndex = 0;
      }
    }

    // Check for button presses to start the game
    if (digitalRead(trueButtonPin) == LOW || digitalRead(falseButtonPin) == LOW) {
      gameActive = true;
      currentQuestion = 0;
      score = 0;
      displayQuestion();
      delay(500); // Debounce delay
    }
  }
}
