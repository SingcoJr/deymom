#include <SPI.h> // Include SPI library for RFID communication #include <MFRC522.h> // Include library for MFRC522 RFID reader #include <Wire.h> // Include Wire library for I2C communication #include <LiquidCrystal_I2C.h> // Include library for I2C LCD display #include <Servo.h> // Include library to control servo motor #include <Keypad.h> // Include library to handle keypad input

// Function Prototype void showMessage(String line1, String line2 = "", int wait = 0); // Declare function for showing messages on LCD

// Pin Definitions #define RST_PIN 9 // Define pin for RFID Reset #define SS_PIN 10 // Define Slave Select pin for RFID #define SERVO_PIN 6 // Define Servo motor control pin

LiquidCrystal_I2C lcd(0x27, 16, 2); // Initialize LCD with I2C address 0x27 and 16x2 size MFRC522 rfid(SS_PIN, RST_PIN); // Initialize MFRC522 RFID module Servo servo; // Create servo object for controlling lock

const byte ROWS = 4; // Define number of keypad rows const byte COLS = 4; // Define number of keypad columns

// Define the keys on the keypad char keys[ROWS][COLS] = { {'1','2','3','A'}, {'4','5','6','B'}, {'7','8','9','C'}, {'*','0','#','D'} };

// Define Arduino pins connected to keypad rows byte rowPins[ROWS] = {2, 3, 4, 5};

// Define Arduino pins connected to keypad columns byte colPins[COLS] = {7, 8, A0, A1};

// Create Keypad object with defined key map and pins Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

// Define valid RFID UIDs (authorized cards) byte validUIDs[][4] = { {0xE5, 0x82, 0x1B, 0x2F}, {0x82, 0xED, 0x1E, 0x2F} };

const String correctPassword = "1234"; // Set the correct password String enteredPassword = ""; // Store the inputted password

int wrongAttempts = 0; // Count the number of wrong attempts bool cooldownActive = false; // Flag for cooldown state bool lockOpen = false; // Flag to check if lock is open unsigned long unlockTime = 0; // Time when lock was opened

void setup() { Serial.begin(9600); // Initialize serial monitor for debugging SPI.begin(); // Initialize SPI bus for RFID rfid.PCD_Init(); // Initialize RFID reader lcd.begin(16, 2); // Initialize the LCD screen lcd.backlight(); // Turn on LCD backlight servo.attach(SERVO_PIN); // Attach the servo to the defined pin lockServo(0); // Lock the door initially (0 degrees) showIdleScreen(); // Show default screen (scan card or enter password) }

void loop() { if (lockOpen && millis() - unlockTime >= 6000) { // If lock is open and 6 seconds passed lockServo(0); // Close the lock (0 degrees) lockOpen = false; // Mark lock as closed showMessage("Lock Closed!", "", 1500); // Show "Lock Closed" message showIdleScreen(); // Show the idle screen again } else if (cooldownActive) { // If system is in cooldown state handleCooldown(); // Execute cooldown function } else { // Normal operation handleRFID(); // Check for RFID card handleKeypad(); // Check for keypad input } }

void handleRFID() { if (!rfid.PICC_IsNewCardPresent() || !rfid.PICC_ReadCardSerial()) return; // If no new card, exit function

Serial.print("Card UID: "); // Print UID to serial monitor for (byte i = 0; i < rfid.uid.size; i++) { Serial.print(rfid.uid.uidByte[i] < 0x10 ? " 0" : " "); // Add leading 0 for single digits Serial.print(rfid.uid.uidByte[i], HEX); // Print UID in hexadecimal } Serial.println();

showMessage("Scanning Card...", "", 1000); // Display "Scanning Card" on LCD

if (isAuthorized(rfid.uid.uidByte)) { // If the scanned card is valid showMessage("RFID Accepted!", "", 1300); // Show "RFID Accepted" showMessage("WELCOME HOME!!", "", 1300); // Show welcome message grantAccess(); // Unlock the door } else { wrongAccess("Invalid RFID!"); // Show "Invalid RFID" if unauthorized } rfid.PICC_HaltA(); // Halt communication with current card }

void handleKeypad() { char key = keypad.getKey(); // Read pressed key if (!key) return; // Exit if no key pressed

if (enteredPassword.length() == 0
