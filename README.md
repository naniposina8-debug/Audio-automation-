# Audio automation player 

RTC based automatic devotional song player using Arduino, DFPlayer Mini,
SD card, LCD and buttons.

Features:
- Date & Time set using LCD + buttons
- Morning & Evening slots
- Automatic rotation mode
- Manual mode with resume
- EEPROM memory support# Audio-automation-
RTC based automatic devotional song rotation system using Arduino &amp; DFPlayer

code:
#include <LiquidCrystal_I2C.h>
#include <RTClib.h>
#include <Wire.h>
#include <SoftwareSerial.h>
#include <DFRobotDFPlayerMini.h>
#include <EEPROM.h>

/*
  Devotional Weekly Rotation Player - Fixed & Optimized
  - Hardware: Arduino UNO, RTC DS3231, 16x2 I2C LCD, DFPlayer Mini
  - Buttons: UP(D2), DOWN(D3), SELECT(D4), MANUAL(D5)
*/

// ----------------- PIN CONFIG -----------------
const uint8_t PIN_UP     = 2;   // button, active LOW
const uint8_t PIN_DOWN   = 3;   // button, active LOW
const uint8_t PIN_SELECT = 4;   // button, active LOW
const uint8_t PIN_MANUAL = 5;   // manual toggle, active LOW
const uint8_t PIN_LED    = 6;   // status LED
const uint8_t PIN_RELAY  = 7;   // relay IN (5V) for amplifier

// DFPlayer Pins
const uint8_t DF_SERIAL_RX_PIN = 8;
const uint8_t DF_SERIAL_TX_PIN = 9;

// ----------------- HARDWARE OBJECTS -----------------
RTC_DS3231 rtc;
// If your LCD doesn't work, try changing 0x27 to 0x3F
LiquidCrystal_I2C lcd(0x27, 16, 2); 
SoftwareSerial dfSerial(DF_SERIAL_RX_PIN, DF_SERIAL_TX_PIN);
DFRobotDFPlayerMini player;

// ----------------- CONFIG -----------------
const int SONGS_PER_DAY = 100;
const int VOLUME = 24;
const unsigned long SAVE_EEPROM_INTERVAL = 10000UL;
const unsigned long TRACK_TIMEOUT_MS = 240000UL; // 4 minutes fallback

// EEPROM Addresses
const int ADDR_SLOT_BASE = 0;
const int ADDR_ROT_BASE  = 10;
const int ADDR_MAN_IDX   = 40;

// ----------------- GLOBAL VARIABLES -----------------
uint16_t rotationNext[8]; // Indices 1..7 used
uint16_t manualNext = 1;

uint8_t morningStart = 6, morningEnd = 7;
uint8_t eveningStart = 18, eveningEnd = 19;

bool manualMode = false;
bool autoPlaying = false;
unsigned long lastEepromSaveTime = 0;
unsigned long lastPlayStart = 0;
unsigned long lastInteractionTime = 0; // For menu timeout

// Menu State
enum MenuState { MS_MAIN, MS_MENU, MS_SET_DATE, MS_SET_TIME, MS_SET_MORNING, MS_SET_EVENING };
MenuState menuState = MS_MAIN;
int menuPos = 0;

// Edit temporary variables
int editDay = 1, editMonth = 1, editYear = 2025;
int editHour = 6, editMinute = 0;
int editMStart = 6, editMEnd = 7;
int editEStart = 18, editEEnd = 19;

// Button Debounce
unsigned long lastDebounce[6] = {0};
const unsigned long DEBOUNCE_MS = 200; // Increased slightly for stability

// DFPlayer State
bool dfpAvailable = false;
unsigned long lastDFPCheck = 0;
const unsigned long DFP_CHECK_INTERVAL = 300; 

// ----------------- HELPER FUNCTIONS -----------------

// Button handler with interaction timer update
bool pressed(uint8_t pin, int idx) {
  if (digitalRead(pin) == LOW) {
    unsigned long t = millis();
    if (t - lastDebounce[idx] > DEBOUNCE_MS) {
      lastDebounce[idx] = t;
      lastInteractionTime = t; // Reset menu timeout on any button press
      return true;
    }
  }
  return false;
}

bool inRotationSlot(DateTime now) {
  int h = now.hour();
  if ((h >= morningStart) && (h < morningEnd)) return true;
  if ((h >= eveningStart) && (h < eveningEnd)) return true;
  return false;
}

// ----------------- EEPROM FUNCTIONS -----------------
void loadSlotsFromEEPROM() {
  uint8_t a = EEPROM.read(ADDR_SLOT_BASE + 0);
  uint8_t b = EEPROM.read(ADDR_SLOT_BASE + 1);
  uint8_t c = EEPROM.read(ADDR_SLOT_BASE + 2);
  uint8_t d = EEPROM.read(ADDR_SLOT_BASE + 3);
  if (a <= 23) { editMStart = morningStart = a; }
  if (b >= 1 && b <= 24) { editMEnd = morningEnd = b; }
  if (c <= 23) { editEStart = eveningStart = c; }
  if (d >= 1 && d <= 24) { editEEnd = eveningEnd = d; }
}

void saveSlotsToEEPROM() {
  EEPROM.update(ADDR_SLOT_BASE + 0, (uint8_t)editMStart);
  EEPROM.update(ADDR_SLOT_BASE + 1, (uint8_t)editMEnd);
  EEPROM.update(ADDR_SLOT_BASE + 2, (uint8_t)editEStart);
  EEPROM.update(ADDR_SLOT_BASE + 3, (uint8_t)editEEnd);
  morningStart = editMStart; morningEnd = editMEnd;
  eveningStart = editEStart; eveningEnd = editEEnd;
  lastEepromSaveTime = millis();
}

void loadRotationFromEEPROM() {
  for (int d = 1; d <= 7; d++) {
    int addr = ADDR_ROT_BASE + (d - 1) * 2;
    uint16_t v;
    EEPROM.get(addr, v);
    if (v < 1 || v > SONGS_PER_DAY) v = 1;
    rotationNext[d] = v;
  }
}

void saveRotationToEEPROM() {
  for (int d = 1; d <= 7; d++) {
    int addr = ADDR_ROT_BASE + (d - 1) * 2;
    EEPROM.put(addr, rotationNext[d]);
  }
  lastEepromSaveTime = millis();
}

void loadManualFromEEPROM() {
  uint16_t v; EEPROM.get(ADDR_MAN_IDX, v);
  if (v < 1) v = 1;
  manualNext = v;
}

void saveManualToEEPROM() {
  EEPROM.put(ADDR_MAN_IDX, manualNext);
  lastEepromSaveTime = millis();
}

void saveAllEEPROMNow() {
  saveSlotsToEEPROM();
  saveRotationToEEPROM();
  saveManualToEEPROM();
}

// ----------------- PLAYBACK FUNCTIONS -----------------
bool playRotationNext(uint8_t dayIndex) {
  if (dayIndex < 1 || dayIndex > 7) return false;
  if (rotationNext[dayIndex] < 1 || rotationNext[dayIndex] > SONGS_PER_DAY) rotationNext[dayIndex] = 1;
  
  uint16_t idx = rotationNext[dayIndex];
  Serial.print("Auto Play Day: "); Serial.print(dayIndex); Serial.print(" Track: "); Serial.println(idx);
  
  player.playFolder(dayIndex, idx);
  
  lastPlayStart = millis();
  rotationNext[dayIndex]++;
  if (rotationNext[dayIndex] > SONGS_PER_DAY) rotationNext[dayIndex] = 1;
  
  if (millis() - lastEepromSaveTime > SAVE_EEPROM_INTERVAL) saveRotationToEEPROM();
  
  digitalWrite(PIN_LED, HIGH);
  return true;
}

bool playManualNext() {
  if (manualNext < 1) manualNext = 1;
  Serial.print("Manual Play Track: "); Serial.println(manualNext);
  
  player.playFolder(8, manualNext); // Folder 08 is manual
  
  lastPlayStart = millis();
  manualNext++;
  if (manualNext > 999) manualNext = 1;
  
  if (millis() - lastEepromSaveTime > SAVE_EEPROM_INTERVAL) saveManualToEEPROM();
  
  digitalWrite(PIN_LED, HIGH);
  return true;
}

void stopAllPlaybackSave() {
  player.stop();
  digitalWrite(PIN_LED, LOW);
  saveAllEEPROMNow();
  manualMode = false;
  autoPlaying = false;
  Serial.println("Playback Stopped & Saved");
}

// ----------------- DISPLAY FUNCTIONS -----------------
void showMainScreen(DateTime now) {
  lcd.clear();
  char buf[17];
  snprintf(buf, sizeof(buf), "DATE:%02d-%02d-%04d", now.day(), now.month(), now.year());
  lcd.setCursor(0,0); lcd.print(buf);
  
  int hr = now.hour();
  int mn = now.minute();
  bool pm = hr >= 12;
  int dispHr = hr % 12; if (dispHr == 0) dispHr = 12;
  
  snprintf(buf, sizeof(buf), "TIME:%02d:%02d %s", dispHr, mn, pm ? "PM":"AM");
  lcd.setCursor(0,1); lcd.print(buf);
  
  lcd.setCursor(15,1);
  lcd.print(manualMode ? "M" : "A");
}

void showMenuEntry(int pos) {
  lcd.clear(); lcd.setCursor(0,0); lcd.print("Menu:");
  lcd.setCursor(0,1);
  switch(pos) {
    case 0: lcd.print("1.Set Date"); break;
    case 1: lcd.print("2.Set Time"); break;
    case 2: lcd.print("3.Set Morning"); break;
    case 3: lcd.print("4.Set Evening"); break;
    case 4: lcd.print("5.Exit"); break;
  }
}

// ----------------- EVENT CHECKER -----------------
void checkDFPEvents() {
  if (!dfpAvailable) return;
  if (millis() - lastDFPCheck < DFP_CHECK_INTERVAL) return;
  lastDFPCheck = millis();

  if (player.available()) {
    uint8_t type = player.readType();
    int value = player.read();
    
    // Check for "Play Finished" event (Type 3)
    if (type == DFPlayerPlayFinished) {
      Serial.println("Event: Track Finished");
      DateTime now = rtc.now();
      uint8_t dayIndex = now.dayOfTheWeek() + 1;
      
      if (autoPlaying) {
        if (inRotationSlot(now)) {
           playRotationNext(dayIndex);
        } else {
           stopAllPlaybackSave();
           digitalWrite(PIN_RELAY, LOW);
        }
      } else if (manualMode) {
        playManualNext();
      } else {
        digitalWrite(PIN_LED, LOW);
      }
    }
  }
}

// ----------------- SETUP -----------------
void setup() {
  // Pin Setup
  pinMode(PIN_UP, INPUT_PULLUP);
  pinMode(PIN_DOWN, INPUT_PULLUP);
  pinMode(PIN_SELECT, INPUT_PULLUP);
  pinMode(PIN_MANUAL, INPUT_PULLUP);
  pinMode(PIN_LED, OUTPUT);
  pinMode(PIN_RELAY, OUTPUT);
  
  digitalWrite(PIN_LED, LOW);
  digitalWrite(PIN_RELAY, LOW);

  Serial.begin(115200);
  Wire.begin();
  
  lcd.init(); 
  lcd.backlight();

  // RTC Check
  if (!rtc.begin()) {
    lcd.clear();
    lcd.print("RTC Error!");
    while (1); // Halt if RTC fails
  }

  // DFPlayer Setup
  dfSerial.begin(9600);
  if (!player.begin(dfSerial)) {
    Serial.println("DFPlayer Error!");
    lcd.clear(); lcd.print("MP3 Error");
    dfpAvailable = false;
    delay(2000);
  } else {
    dfpAvailable = true;
    player.volume(VOLUME);
    player.EQ(DFPLAYER_EQ_NORMAL);
  }

  // Load Saved Data
  loadSlotsFromEEPROM();
  loadRotationFromEEPROM();
  loadManualFromEEPROM();
  lastEepromSaveTime = millis();
  lastInteractionTime = millis();

  lcd.clear();
  lcd.setCursor(0,0); lcd.print("   Welcome    ");
  lcd.setCursor(0,1); lcd.print(" System Ready ");
  delay(1000);
  menuState = MS_MAIN;
}

// ----------------- LOOP -----------------
void loop() {
  DateTime now = rtc.now();
  uint8_t dow = now.dayOfTheWeek(); // 0 (Sun) .. 6 (Sat)
  uint8_t dayIndex = dow + 1;       // 1 .. 7

  // --- Display Logic ---
  static unsigned long lastMainUpdate = 0;
  
  if (menuState == MS_MAIN) {
    if (millis() - lastMainUpdate > 800) {
      showMainScreen(now);
      lastMainUpdate = millis();
    }
  } else if (menuState == MS_MENU) {
    showMenuEntry(menuPos);
  } else if (menuState == MS_SET_DATE) {
    lcd.clear(); lcd.setCursor(0,0); lcd.print("SET DATE:");
    char s[17]; snprintf(s,sizeof(s), "%02d-%02d-%04d", editDay, editMonth, editYear);
    lcd.setCursor(0,1); lcd.print(s);
  } else if (menuState == MS_SET_TIME) {
    lcd.clear(); lcd.setCursor(0,0); lcd.print("SET TIME:");
    char s[17]; snprintf(s,sizeof(s), "%02d:%02d", editHour, editMinute);
    lcd.setCursor(0,1); lcd.print(s);
  } else if (menuState == MS_SET_MORNING) {
    lcd.clear(); lcd.setCursor(0,0); lcd.print("MORNING START:");
    char s[17]; snprintf(s,sizeof(s), "%02d:00-%02d:00", editMStart, editMEnd);
    lcd.setCursor(0,1); lcd.print(s);
  } else if (menuState == MS_SET_EVENING) {
    lcd.clear(); lcd.setCursor(0,0); lcd.print("EVENING START:");
    char s[17]; snprintf(s,sizeof(s), "%02d:00-%02d:00", editEStart, editEEnd);
    lcd.setCursor(0,1); lcd.print(s);
  }

  // --- SELECT Button ---
  if (pressed(PIN_SELECT, 2)) {
    if (menuState == MS_MAIN) {
      menuState = MS_MENU; menuPos = 0; delay(200);
    } else if (menuState == MS_MENU) {
      // Enter specific menu
      if (menuPos == 0) {
        editDay = now.day(); editMonth = now.month(); editYear = now.year();
        menuState = MS_SET_DATE;
      } else if (menuPos == 1) {
        editHour = now.hour(); editMinute = now.minute();
        menuState = MS_SET_TIME;
      } else if (menuPos == 2) {
        loadSlotsFromEEPROM(); menuState = MS_SET_MORNING;
      } else if (menuPos == 3) {
        loadSlotsFromEEPROM(); menuState = MS_SET_EVENING;
      } else {
        menuState = MS_MAIN;
      }
      delay(200);
    } else {
      // Save settings
      if (menuState == MS_SET_DATE) {
         if (editDay < 1) editDay = 1; if (editDay > 31) editDay = 31;
         if (editMonth < 1) editMonth = 1; if (editMonth > 12) editMonth = 12;
         if (editYear < 2000) editYear = 2025;
         rtc.adjust(DateTime(editYear, editMonth, editDay, now.hour(), now.minute(), 0));
      } else if (menuState == MS_SET_TIME) {
         if (editHour < 0) editHour = 0; if (editHour > 23) editHour = 23;
         if (editMinute < 0) editMinute = 0; if (editMinute > 59) editMinute = 59;
         rtc.adjust(DateTime(now.year(), now.month(), now.day(), editHour, editMinute, 0));
      } else if (menuState == MS_SET_MORNING) {
         if (editMStart < 0) editMStart = 0; if (editMStart > 23) editMStart = 23;
         if (editMEnd <= editMStart) editMEnd = (editMStart + 1) % 24;
         saveSlotsToEEPROM();
      } else if (menuState == MS_SET_EVENING) {
         if (editEStart < 0) editEStart = 0; if (editEStart > 23) editEStart = 23;
         if (editEEnd <= editEStart) editEEnd = (editEStart + 1) % 24;
         saveSlotsToEEPROM();
      }
      menuState = MS_MAIN;
      delay(250);
    }
  }

  // --- UP / DOWN Buttons ---
  if (pressed(PIN_UP, 0)) {
    if (menuState == MS_MENU) { menuPos = (menuPos + 4) % 5; }
    else if (menuState == MS_SET_DATE) { editDay++; if (editDay > 31) editDay = 1; }
    else if (menuState == MS_SET_TIME) { editHour++; if (editHour > 23) editHour = 0; }
    else if (menuState == MS_SET_MORNING) { editMStart++; if (editMStart > 23) editMStart = 0; }
    else if (menuState == MS_SET_EVENING) { editEStart++; if (editEStart > 23) editEStart = 0; }
    delay(100);
  }

  if (pressed(PIN_DOWN, 1)) {
    if (menuState == MS_MENU) { menuPos = (menuPos + 1) % 5; }
    else if (menuState == MS_SET_DATE) { editDay--; if (editDay < 1) editDay = 31; }
    else if (menuState == MS_SET_TIME) { editHour--; if (editHour < 0) editHour = 23; }
    else if (menuState == MS_SET_MORNING) { editMStart--; if (editMStart < 0) editMStart = 23; }
    else if (menuState == MS_SET_EVENING) { editEStart--; if (editEStart < 0) editEStart = 23; }
    delay(100);
  }

  // --- MANUAL Button ---
  if (pressed(PIN_MANUAL, 3)) {
    if (manualMode) {
      stopAllPlaybackSave();
      if (!inRotationSlot(rtc.now())) digitalWrite(PIN_RELAY, LOW); 
    } else {
      if (autoPlaying) stopAllPlaybackSave();
      manualMode = true;
      autoPlaying = false;
      digitalWrite(PIN_RELAY, HIGH);
      digitalWrite(PIN_LED, HIGH);
      playManualNext();
    }
    delay(300);
  }

  // --- AUTO Logic (Priority) ---
  bool slotNow = inRotationSlot(now);
  if (slotNow) {
    if (manualMode) {
      Serial.println("Slot Time Reached - Overriding Manual");
      saveManualToEEPROM();
      player.stop();
      manualMode = false;
      autoPlaying = true;
      digitalWrite(PIN_RELAY, HIGH);
      playRotationNext(dayIndex);
    } else if (!autoPlaying) {
      autoPlaying = true;
      digitalWrite(PIN_RELAY, HIGH);
      playRotationNext(dayIndex);
    }
  } else {
    // Outside slot time
    if (autoPlaying) {
      stopAllPlaybackSave();
      digitalWrite(PIN_RELAY, LOW);
    }
    // Note: If manualMode is true here, it stays true (Manual Override)
  }

  // --- Player Event & Timeout Check ---
  if (dfpAvailable) checkDFPEvents();

  // Safety Timeout: If playing but no events received for too long
  if (autoPlaying || manualMode) {
    if (millis() - lastPlayStart > TRACK_TIMEOUT_MS) {
      if (autoPlaying) playRotationNext(dayIndex);
      else playManualNext();
    }
    // Periodic Save
    if (millis() - lastEepromSaveTime > SAVE_EEPROM_INTERVAL) {
      saveRotationToEEPROM();
      saveManualToEEPROM();
    }
  }

  // --- Menu Idle Timeout (Logic Fixed) ---
  if (menuState != MS_MAIN) {
    // If no buttons pressed for 60 seconds, return to main
    if (millis() - lastInteractionTime > 60000UL) {
       menuState = MS_MAIN;
       lcd.clear();
    }
  }

  delay(80);
}
