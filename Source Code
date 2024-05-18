#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

#define SS_PIN 10
#define RST_PIN 9
#define BUZZER_PIN 8

MFRC522 rfid(SS_PIN, RST_PIN);  // Create MFRC522 instance.
LiquidCrystal_I2C lcd(0x27, 16, 2); // Set the LCD I2C address and dimensions (address, columns, rows)

// Define the pool of numbers
const int numbers[] = {101, 102, 103, 104, 105};
const int maxNumbers = sizeof(numbers) / sizeof(numbers[0]);
bool lockerAvailable[maxNumbers] = {true, true, true, true, true}; // Availability of each locker

struct CardAssignment {
  byte uid[10];   // Store up to 10 bytes of UID
  byte uidSize;
  int number;
  unsigned long timestamp; // Time when the card was scanned
};

CardAssignment assignments[maxNumbers]; // Array to hold assignments
int assignmentCount = 0;

// Target UID to show all available lockers
byte targetUID[] = {0x03, 0x4F, 0x34, 0xA6};
const int targetUIDSize = sizeof(targetUID) / sizeof(targetUID[0]);

void setup() {
  Serial.begin(9600);  // Initialize serial communications with the PC.
  SPI.begin();         // Init SPI bus.
  rfid.PCD_Init();     // Init MFRC522.
  pinMode(BUZZER_PIN, OUTPUT); // Set buzzer pin as output

  lcd.init();          // Initialize the LCD
  lcd.backlight();     // Turn on the backlight
  lcd.setCursor(0, 0);
  lcd.print("Scan Student ID");
  Serial.println("Scan Student ID");
}

void loop() {
  // Look for new cards
  if (!rfid.PICC_IsNewCardPresent() || !rfid.PICC_ReadCardSerial()) {
    return;
  }

  // Get the UID of the scanned card
  byte uid[10];
  byte uidSize = rfid.uid.size;
  memcpy(uid, rfid.uid.uidByte, uidSize);

  // Check if the scanned UID matches the target UID
  if (uidSize == targetUIDSize && memcmp(uid, targetUID, uidSize) == 0) {
    displayAvailableLockers();
  } else {
    handleCard(uid, uidSize);
  }

  // Halt PICC
  rfid.PICC_HaltA();
  // Stop encryption on PCD
  rfid.PCD_StopCrypto1();

  delay(2000); // Wait for 2 seconds before clearing the LCD
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Scan Student ID");
}

void displayAvailableLockers() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Available Lockers:");
  lcd.setCursor(0, 1);
  bool lockersAvailable = false;
  for (int i = 0; i < maxNumbers; i++) {
    if (lockerAvailable[i]) {
      lcd.print(numbers[i]);
      lcd.print(" ");
      lockersAvailable = true;
    }
  }
  if (!lockersAvailable) {
    lcd.print("None");
  }
  
  // Fast beep-beep sound
  for (int i = 0; i < 3; i++) { // 3 beeps
    digitalWrite(BUZZER_PIN, HIGH);
    delay(100); // 100 ms on
    digitalWrite(BUZZER_PIN, LOW);
    delay(100); // 100 ms off
  }
}

void handleCard(byte *uid, byte uidSize) {
  int assignedNumber = findAssignment(uid, uidSize);
  unsigned long currentTime = millis();

  // Clear the LCD
  lcd.clear();

  if (assignedNumber != -1) {
    unsigned long elapsedTime = currentTime - getTimestamp(uid, uidSize);
    if (elapsedTime > 20000) { // 20 seconds
      // Card has been used for more than 20 seconds, unlock the locker
      removeAssignment(uid, uidSize);
      lcd.setCursor(0, 0);
      lcd.print("OVER TIME");
      lcd.setCursor(0, 1);
      lcd.print("Unlocked Locker:");
      lcd.setCursor(0, 1);
      lcd.print(assignedNumber);
      digitalWrite(BUZZER_PIN, HIGH);
      delay(3000); // 3 seconds long beep
      digitalWrite(BUZZER_PIN, LOW);
    } else {
      // Card is already assigned, so unlock the number
      removeAssignment(uid, uidSize);
      lcd.setCursor(0, 0);
      lcd.print("Unlocked Locker:");
      lcd.setCursor(0, 1);
      lcd.print(assignedNumber);
      
      // Fast beep-beep sound
      for (int i = 0; i < 3; i++) { // 3 beeps
        digitalWrite(BUZZER_PIN, HIGH);
        delay(100); // 100 ms on
        digitalWrite(BUZZER_PIN, LOW);
        delay(100); // 100 ms off
      }
    }
  } else {
    // Card is not assigned, so assign a new number
    assignedNumber = findAvailableLocker();
    if (assignedNumber != -1) {
      addAssignment(uid, uidSize, assignedNumber);
      lcd.setCursor(0, 0);
      lcd.print("Assigned Locker:");
      lcd.setCursor(0, 1);
      lcd.print(assignedNumber);
      
      // Fast beep-beep sound
      for (int i = 0; i < 3; i++) { // 3 beeps
        digitalWrite(BUZZER_PIN, HIGH);
        delay(100); // 100 ms on
        digitalWrite(BUZZER_PIN, LOW);
        delay(100); // 100 ms off
      }
    } else {
      lcd.setCursor(0, 0);
      lcd.print("No Locker");
      lcd.setCursor(0, 1);
      lcd.print("Available");
      
      // Fast beep-beep sound indicating no locker available
      for (int i = 0; i < 3; i++) { // 3 beeps
        digitalWrite(BUZZER_PIN, HIGH);
        delay(100); // 100 ms on
        digitalWrite(BUZZER_PIN, LOW);
        delay(100); // 100 ms off
      }
    }
  }
}

int findAssignment(byte *uid, byte uidSize) {
  for (int i = 0; i < assignmentCount; i++) {
    if (assignments[i].uidSize == uidSize && memcmp(assignments[i].uid, uid, uidSize) == 0) {
      return assignments[i].number;
    }
  }
  return -1;
}

unsigned long getTimestamp(byte *uid, byte uidSize) {
  for (int i = 0; i < assignmentCount; i++) {
    if (assignments[i].uidSize == uidSize && memcmp(assignments[i].uid, uid, uidSize) == 0) {
      return assignments[i].timestamp;
    }
  }
  return 0;
}

int findAvailableLocker() {
  for (int i = 0; i < maxNumbers; i++) {
    if (lockerAvailable[i]) {
      lockerAvailable[i] = false;
      return numbers[i];
    }
  }
  return -1;
}

void addAssignment(byte *uid, byte uidSize, int number) {
  memcpy(assignments[assignmentCount].uid, uid, uidSize);
  assignments[assignmentCount].uidSize = uidSize;
  assignments[assignmentCount].number = number;
  assignments[assignmentCount].timestamp = millis(); // Store the current time
  assignmentCount++;
}

void removeAssignment(byte *uid, byte uidSize) {
  for (int i = 0; i < assignmentCount; i++) {
    if (assignments[i].uidSize == uidSize && memcmp(assignments[i].uid, uid, uidSize) == 0) {
      // Mark the locker as available again
      for (int j = 0; j < maxNumbers; j++) {
        if (numbers[j] == assignments[i].number) {
          lockerAvailable[j] = true;
          break;
        }
      }

      // Shift remaining assignments left to fill the gap
      for (int j = i; j < assignmentCount - 1; j++) {
        assignments[j] = assignments[j + 1];
      }
      assignmentCount--;
      break;
    }
  }
}
