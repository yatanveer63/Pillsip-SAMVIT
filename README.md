# Pillsip-SAMVIT
PillSip | S.A.M.V.I.T  is an IoT-enabled smart medicine reminder and dispenser system  The system is built around the Seeed Studio XIAO ESP32-C6  integrates a  display,  temperature sensor,  servo motors, a touch module, an  buzzer, and a battery  circuit. It sync time automatically via WiFi  servers, scheduling of  four daily medication reminders.
code for this project :
/*
 * =====================================================================
 *  Smart Medicine Reminder & Dispenser System
 *  Hardware: Seeed Studio XIAO ESP32-C6
 * =====================================================================
 *  Author  : Smart Medicine Project
 *  Version : 1.0.0
 *  Target  : Arduino IDE 2.x + ESP32 Arduino Core 3.x
 * 
 *  PIN MAP:
 *    D0 (GPIO2)  → Servo Motor 1 (Signal)
 *    D1 (GPIO3)  → Servo Motor 2 (Signal)
 *    D2 (GPIO4)  → DHT11 Data
 *    D3 (GPIO5)  → Capacitive Touch (SIG)
 *    D4 (GPIO6)  → Active Buzzer
 *    D6 (GPIO22) → OLED SDA (I2C)
 *    D7 (GPIO23) → OLED SCL (I2C)
 *    A0 (GPIO0)  → Battery Voltage Divider
 * =====================================================================
 */

// ─────────────────────────────────────────────────────────────────────
//  LIBRARY INCLUDES
// ─────────────────────────────────────────────────────────────────────
#include <Arduino.h>
#include <Wire.h>
#include <WiFi.h>
#include <time.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <DHT.h>
#include <ESP32Servo.h>

// ─────────────────────────────────────────────────────────────────────
//  PIN DEFINITIONS
// ─────────────────────────────────────────────────────────────────────
#define SERVO1_PIN       2    // D0 – Servo Motor 1
#define SERVO2_PIN       3    // D1 – Servo Motor 2
#define DHT_PIN          4    // D2 – DHT11 Data
#define TOUCH_PIN        5    // D3 – Capacitive Touch Signal
#define BUZZER_PIN       6    // D4 – Active Buzzer
#define BATTERY_PIN      0    // A0 – Battery Voltage (ADC)

// OLED I2C uses default Wire (SDA=22, SCL=23 on XIAO ESP32-C6)
#define SCREEN_WIDTH     128
#define SCREEN_HEIGHT    32
#define OLED_RESET       -1   // No reset pin
#define OLED_I2C_ADDR    0x3C

// DHT sensor type
#define DHT_TYPE         DHT11

// ─────────────────────────────────────────────────────────────────────
//  WiFi & NTP SETTINGS  ← UPDATE THESE
// ─────────────────────────────────────────────────────────────────────
const char* WIFI_SSID     = "YOUR_WIFI_SSID";
const char* WIFI_PASSWORD = "YOUR_WIFI_PASSWORD";
const char* NTP_SERVER    = "pool.ntp.org";
const char* NTP_SERVER2   = "time.nist.gov";
const long  GMT_OFFSET_SEC   = 28800; // UTC+8 (Philippines/SGT) – adjust as needed
const int   DAYLIGHT_OFFSET  = 0;     // No daylight saving

// ─────────────────────────────────────────────────────────────────────
//  SYSTEM CONSTANTS
// ─────────────────────────────────────────────────────────────────────
#define MAX_REMINDERS         4      // Maximum number of daily reminders
#define REMINDER_WINDOW_MIN   1      // ±1 minute tolerance for trigger
#define MISSED_REMINDER_MS    120000 // 2 minutes in milliseconds
#define SERVO_OPEN_ANGLE      180    // Degrees when dispensing open
#define SERVO_CLOSE_ANGLE     0      // Degrees when dispensing closed
#define BUZZ_ON_MS            200    // Buzzer ON duration per beep
#define BUZZ_OFF_MS           200    // Buzzer OFF between beeps
#define BATTERY_MAX_V         4.2f   // LiPo full voltage
#define BATTERY_MIN_V         3.0f   // LiPo empty voltage
#define ADC_REF_V             3.3f   // ESP32 ADC reference voltage
#define ADC_RESOLUTION        4095.0f// 12-bit ADC
#define SCROLL_SPEED_MS       50     // OLED scroll update interval
#define SENSOR_UPDATE_MS      2000   // DHT11 read interval
#define DISPLAY_UPDATE_MS     1000   // OLED time update interval
#define DEBOUNCE_MS           500    // Touch debounce period

// ─────────────────────────────────────────────────────────────────────
//  MEDICINE TIME RANGE → CONTAINER COLOR MAPPING
// ─────────────────────────────────────────────────────────────────────
struct TimeRange {
  int startHour;
  int endHour;
  const char* containerColor;
};

const TimeRange containerMap[] = {
  { 5,  8,  "Yellow" },  // 05:00 – 07:59
  { 8,  11, "Red"    },  // 08:00 – 10:59
  { 13, 16, "White"  },  // 13:00 – 15:59
  { 20, 23, "Black"  }   // 20:00 – 22:59
};
const int containerMapSize = 4;

// ─────────────────────────────────────────────────────────────────────
//  REMINDER STRUCTURE
// ─────────────────────────────────────────────────────────────────────
struct Reminder {
  int  hour;          // 0–23
  int  minute;        // 0–59
  bool active;        // Is this slot configured?
  bool triggeredToday;// Prevent re-trigger in same minute
};

// ─────────────────────────────────────────────────────────────────────
//  SYSTEM STATE ENUM
// ─────────────────────────────────────────────────────────────────────
enum SystemState {
  STATE_IDLE,           // Normal display mode
  STATE_REMINDER,       // Reminder active, waiting for touch
  STATE_DISPENSING,     // Servo open (first touch done)
  STATE_CLOSING,        // Servo closing (second touch done)
  STATE_MISSED          // Reminder missed, second alert sent
};

// ─────────────────────────────────────────────────────────────────────
//  GLOBAL OBJECTS
// ─────────────────────────────────────────────────────────────────────
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);
DHT              dht(DHT_PIN, DHT_TYPE);
Servo            servo1;
Servo            servo2;

// ─────────────────────────────────────────────────────────────────────
//  GLOBAL STATE VARIABLES
// ─────────────────────────────────────────────────────────────────────
Reminder    reminders[MAX_REMINDERS];
int         reminderCount   = 0;
SystemState sysState        = STATE_IDLE;
bool        touchEnabled    = false;
bool        dispenserOpen   = false;
int         touchCount      = 0;         // Counts touches during active reminder
String      activeContainer = "";

// Timing (non-blocking)
unsigned long reminderTriggerMs    = 0; // When current reminder started
unsigned long lastSensorReadMs     = 0;
unsigned long lastDisplayUpdateMs  = 0;
unsigned long lastScrollMs         = 0;
unsigned long lastTouchMs          = 0;
unsigned long buzzScheduledMs      = 0;  // Non-blocking buzzer
unsigned long buzzStep             = 0;

// Sensor data
float  temperature   = 0.0;
float  humidity      = 0.0;
int    batteryPct    = 0;

// OLED scrolling text state
int    scrollX       = SCREEN_WIDTH; // Current X position of scrolling text
String scrollMsg     = "";

// Serial input buffer
String serialBuffer  = "";
bool   configMode    = false;
int    configStep    = 0;   // Which reminder slot is being configured

// ─────────────────────────────────────────────────────────────────────
//  FORWARD DECLARATIONS
// ─────────────────────────────────────────────────────────────────────
void initWiFiNTP();
void initOLED();
void initServos();
void printWelcome();
void handleSerial();
void handleReminderCheck();
void handleTouch();
void handleMissedReminder();
void handleBuzzer();
void handleSensors();
void handleDisplay();
void triggerReminder(const String& container);
void openDispenser();
void closeDispenser();
void scheduleBuzz(int times);
void drawIdleScreen();
void drawReminderScreen(const String& msg);
int  readBatteryPercent();
const char* getContainerColor(int hour);
String getFormattedTime();
bool isReminderTime(int rHour, int rMinute, int curHour, int curMinute);

// =====================================================================
//  SETUP
// =====================================================================
void setup() {
  Serial.begin(115200);
  delay(500);

  // Init pins
  pinMode(TOUCH_PIN,  INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);

  // Battery ADC
  analogReadResolution(12);
  analogSetAttenuation(ADC_11db); // 0–3.3V range

  // Init subsystems
  initOLED();
  initServos();
  dht.begin();
  initWiFiNTP();

  // Init reminder slots as inactive
  for (int i = 0; i < MAX_REMINDERS; i++) {
    reminders[i].active         = false;
    reminders[i].triggeredToday = false;
    reminders[i].hour           = -1;
    reminders[i].minute         = -1;
  }

  printWelcome();

  // Initial sensor read
  temperature = dht.readTemperature();
  humidity    = dht.readHumidity();
  batteryPct  = readBatteryPercent();

  Serial.println(F("[SYS] Setup complete. Entering main loop."));
}

// =====================================================================
//  MAIN LOOP  (non-blocking)
// =====================================================================
void loop() {
  unsigned long now = millis();

  handleSerial();        // Parse Serial input for config
  handleSensors();       // DHT11 & battery reads
  handleReminderCheck(); // Check if any reminder should trigger
  handleTouch();         // Process touch events
  handleMissedReminder(); // Check 2-minute timeout
  handleBuzzer();        // Non-blocking buzzer sequencer
  handleDisplay();       // Update OLED

  // Reset triggeredToday flags at midnight
  struct tm t;
  if (getLocalTime(&t)) {
    if (t.tm_hour == 0 && t.tm_min == 0 && t.tm_sec < 2) {
      for (int i = 0; i < MAX_REMINDERS; i++) {
        reminders[i].triggeredToday = false;
      }
    }
  }
}

// =====================================================================
//  INITIALIZATION FUNCTIONS
// =====================================================================

// ── WiFi + NTP ──────────────────────────────────────────────────────
void initWiFiNTP() {
  Serial.print(F("[WiFi] Connecting to: "));
  Serial.println(WIFI_SSID);

  display.clearDisplay();
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.println(F("Connecting WiFi..."));
  display.display();

  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 30) {
    delay(500);
    Serial.print(F("."));
    attempts++;
  }

  if (WiFi.status() == WL_CONNECTED) {
    Serial.println();
    Serial.print(F("[WiFi] Connected! IP: "));
    Serial.println(WiFi.localIP());

    // Configure NTP
    configTime(GMT_OFFSET_SEC, DAYLIGHT_OFFSET, NTP_SERVER, NTP_SERVER2);

    // Wait for time sync
    Serial.print(F("[NTP] Syncing time..."));
    struct tm t;
    int syncAttempts = 0;
    while (!getLocalTime(&t) && syncAttempts < 20) {
      delay(500);
      Serial.print(F("."));
      syncAttempts++;
    }
    Serial.println();

    if (getLocalTime(&t)) {
      Serial.print(F("[NTP] Time synced: "));
      Serial.println(asctime(&t));
      display.clearDisplay();
      display.setCursor(0, 0);
      display.println(F("Time synced!"));
      display.display();
      delay(1000);
    } else {
      Serial.println(F("[NTP] Sync failed! Time may be inaccurate."));
    }
  } else {
    Serial.println(F("[WiFi] Connection FAILED! Operating without NTP."));
    display.clearDisplay();
    display.setCursor(0, 0);
    display.println(F("WiFi failed!"));
    display.println(F("Manual time only."));
    display.display();
    delay(2000);
  }

  // Disconnect WiFi to save power (time kept in RTC)
  WiFi.disconnect(true);
  WiFi.mode(WIFI_OFF);
  Serial.println(F("[WiFi] Disconnected to save power."));
}

// ── OLED ─────────────────────────────────────────────────────────────
void initOLED() {
  Wire.begin();
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_I2C_ADDR)) {
    Serial.println(F("[OLED] SSD1306 init FAILED! Check wiring."));
    while (true) { delay(1000); } // Halt
  }
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.println(F("Smart Dispenser v1.0"));
  display.println(F("Booting..."));
  display.display();
  Serial.println(F("[OLED] Initialized OK"));
}

// ── SERVOS ───────────────────────────────────────────────────────────
void initServos() {
  servo1.attach(SERVO1_PIN, 500, 2400); // Min/max pulse width µs
  servo2.attach(SERVO2_PIN, 500, 2400);
  servo1.write(SERVO_CLOSE_ANGLE);
  servo2.write(SERVO_CLOSE_ANGLE);
  Serial.println(F("[SERVO] Initialized at closed position (0°)"));
}

// ── SERIAL WELCOME MESSAGE ───────────────────────────────────────────
void printWelcome() {
  Serial.println(F("\n=========================================="));
  Serial.println(F("  Smart Medicine Reminder & Dispenser"));
  Serial.println(F("  Seeed XIAO ESP32-C6  |  v1.0.0"));
  Serial.println(F("=========================================="));
  Serial.println(F("Commands:"));
  Serial.println(F("  SET   → Configure medicine reminder times"));
  Serial.println(F("  LIST  → Show current reminders"));
  Serial.println(F("  CLEAR → Clear all reminders"));
  Serial.println(F("  TIME  → Show current system time"));
  Serial.println(F("  TEST  → Trigger test reminder"));
  Serial.println(F("  BUZZ  → Test buzzer"));
  Serial.println(F("  OPEN  → Open dispenser manually"));
  Serial.println(F("  CLOSE → Close dispenser manually"));
  Serial.println(F("==========================================\n"));
}

// =====================================================================
//  SERIAL INPUT HANDLER
// =====================================================================
void handleSerial() {
  while (Serial.available()) {
    char c = Serial.read();

    if (c == '\n' || c == '\r') {
      serialBuffer.trim();

      if (serialBuffer.length() == 0) {
        serialBuffer = "";
        return;
      }

      // ── Config mode: expecting HH:MM input ──────────────────────
      if (configMode) {
        if (serialBuffer.equalsIgnoreCase("DONE") || serialBuffer.equalsIgnoreCase("SKIP")) {
          configMode = false;
          Serial.println(F("[CONFIG] Configuration complete."));
          printReminderList();
        } else {
          // Parse HH:MM format
          int colonIdx = serialBuffer.indexOf(':');
          if (colonIdx != -1 && configStep < MAX_REMINDERS) {
            int h = serialBuffer.substring(0, colonIdx).toInt();
            int m = serialBuffer.substring(colonIdx + 1).toInt();
            if (h >= 0 && h <= 23 && m >= 0 && m <= 59) {
              reminders[configStep].hour           = h;
              reminders[configStep].minute         = m;
              reminders[configStep].active         = true;
              reminders[configStep].triggeredToday = false;
              Serial.print(F("[CONFIG] Reminder "));
              Serial.print(configStep + 1);
              Serial.print(F(" set to "));
              if (h < 10) Serial.print(F("0"));
              Serial.print(h);
              Serial.print(F(":"));
              if (m < 10) Serial.print(F("0"));
              Serial.println(m);
              // Identify container
              const char* col = getContainerColor(h);
              Serial.print(F("[CONFIG] → Container: "));
              Serial.println(col);
              configStep++;
              reminderCount = configStep;
              if (configStep < MAX_REMINDERS) {
                Serial.print(F("[CONFIG] Enter time for Reminder "));
                Serial.print(configStep + 1);
                Serial.println(F(" (HH:MM), or type DONE:"));
              } else {
                configMode = false;
                Serial.println(F("[CONFIG] All 4 reminder slots filled."));
                printReminderList();
              }
            } else {
              Serial.println(F("[CONFIG] Invalid time. Use HH:MM (e.g. 08:00)"));
            }
          } else {
            Serial.println(F("[CONFIG] Invalid format. Use HH:MM (e.g. 14:30)"));
          }
        }
      }
      // ── Command mode ────────────────────────────────────────────
      else {
        String cmd = serialBuffer;
        cmd.toUpperCase();

        if (cmd == "SET") {
          configMode  = true;
          configStep  = 0;
          reminderCount = 0;
          for (int i = 0; i < MAX_REMINDERS; i++) reminders[i].active = false;
          Serial.println(F("[CONFIG] Enter medicine times (HH:MM, 24-hr)."));
          Serial.println(F("[CONFIG] Enter time for Reminder 1 (HH:MM), or DONE:"));

        } else if (cmd == "LIST") {
          printReminderList();

        } else if (cmd == "CLEAR") {
          for (int i = 0; i < MAX_REMINDERS; i++) reminders[i].active = false;
          reminderCount = 0;
          Serial.println(F("[CONFIG] All reminders cleared."));

        } else if (cmd == "TIME") {
          Serial.print(F("[TIME] Current time: "));
          Serial.println(getFormattedTime());

        } else if (cmd == "TEST") {
          Serial.println(F("[TEST] Triggering test reminder..."));
          triggerReminder("Yellow");

        } else if (cmd == "BUZZ") {
          Serial.println(F("[TEST] Testing buzzer..."));
          scheduleBuzz(2);

        } else if (cmd == "OPEN") {
          Serial.println(F("[SERVO] Manual open command."));
          openDispenser();

        } else if (cmd == "CLOSE") {
          Serial.println(F("[SERVO] Manual close command."));
          closeDispenser();

        } else {
          Serial.print(F("[CMD] Unknown command: "));
          Serial.println(serialBuffer);
          Serial.println(F("[CMD] Type SET, LIST, CLEAR, TIME, TEST, BUZZ, OPEN, CLOSE"));
        }
      }

      serialBuffer = "";
    } else {
      serialBuffer += c;
    }
  }
}

// Print helper for reminder list
void printReminderList() {
  Serial.println(F("[LIST] ── Configured Reminders ──────────"));
  bool any = false;
  for (int i = 0; i < MAX_REMINDERS; i++) {
    if (reminders[i].active) {
      Serial.print(F("  ["));
      Serial.print(i + 1);
      Serial.print(F("] "));
      if (reminders[i].hour < 10) Serial.print(F("0"));
      Serial.print(reminders[i].hour);
      Serial.print(F(":"));
      if (reminders[i].minute < 10) Serial.print(F("0"));
      Serial.print(reminders[i].minute);
      Serial.print(F("  → "));
      Serial.print(getContainerColor(reminders[i].hour));
      Serial.println(F(" container"));
      any = true;
    }
  }
  if (!any) Serial.println(F("  (no reminders set)"));
  Serial.println(F("[LIST] ───────────────────────────────"));
}

// =====================================================================
//  REMINDER CHECK  (runs every loop iteration)
// =====================================================================
void handleReminderCheck() {
  if (sysState != STATE_IDLE) return; // Only check when idle
  if (reminderCount == 0) return;

  struct tm t;
  if (!getLocalTime(&t)) return;

  for (int i = 0; i < MAX_REMINDERS; i++) {
    if (!reminders[i].active || reminders[i].triggeredToday) continue;

    if (isReminderTime(reminders[i].hour, reminders[i].minute,
                        t.tm_hour, t.tm_min)) {
      const char* color = getContainerColor(t.tm_hour);
      Serial.print(F("[ALARM] Reminder "));
      Serial.print(i + 1);
      Serial.print(F(" triggered! Container: "));
      Serial.println(color);
      reminders[i].triggeredToday = true;
      triggerReminder(String(color));
      break; // Only one reminder at a time
    }
  }
}

// Helper: check if current time matches a reminder (±1 min tolerance)
bool isReminderTime(int rH, int rM, int cH, int cM) {
  int rTotal = rH * 60 + rM;
  int cTotal = cH * 60 + cM;
  return abs(rTotal - cTotal) <= REMINDER_WINDOW_MIN;
}

// =====================================================================
//  TRIGGER REMINDER
// =====================================================================
void triggerReminder(const String& container) {
  activeContainer     = container;
  sysState            = STATE_REMINDER;
  touchEnabled        = true;
  touchCount          = 0;
  dispenserOpen       = false;
  reminderTriggerMs   = millis();

  String msg = "It's time to take your medicine from the "
               + container + " container.";

  Serial.println(F("[REMINDER] ─────────────────────────────"));
  Serial.println("[REMINDER] " + msg);
  Serial.println(F("[REMINDER] Touch the sensor to dispense."));
  Serial.println(F("[REMINDER] ─────────────────────────────"));

  scrollMsg = msg;
  scrollX   = SCREEN_WIDTH;
  scheduleBuzz(2); // Two beeps
  drawReminderScreen(msg);
}

// =====================================================================
//  TOUCH HANDLER
// =====================================================================
void handleTouch() {
  if (!touchEnabled) return;
  if (sysState != STATE_REMINDER && sysState != STATE_DISPENSING) return;

  unsigned long now = millis();
  if (now - lastTouchMs < DEBOUNCE_MS) return; // Debounce

  if (digitalRead(TOUCH_PIN) == HIGH) {
    lastTouchMs = now;
    touchCount++;
    Serial.print(F("[TOUCH] Touch detected! Count: "));
    Serial.println(touchCount);

    if (touchCount == 1) {
      // ── First touch: OPEN dispenser ───────────────────────────
      Serial.println(F("[SERVO] First touch → Opening dispenser (180°)"));
      openDispenser();
      sysState = STATE_DISPENSING;
      display.clearDisplay();
      display.setCursor(0, 0);
      display.println(F("Dispensing..."));
      display.println(F("Touch again to close"));
      display.display();

    } else if (touchCount >= 2) {
      // ── Second touch: CLOSE dispenser ─────────────────────────
      Serial.println(F("[SERVO] Second touch → Closing dispenser (0°)"));
      closeDispenser();
      sysState     = STATE_IDLE;
      touchEnabled = false;
      touchCount   = 0;
      activeContainer = "";
      Serial.println(F("[SYS] Medicine taken. Returning to idle."));

      display.clearDisplay();
      display.setCursor(0, 8);
      display.println(F("Medicine taken!"));
      display.println(F("Stay healthy!"));
      display.display();
      delay(2000);

      scrollX = SCREEN_WIDTH; // Reset scroll
    }
  }
}

// =====================================================================
//  MISSED REMINDER HANDLER  (2-minute check)
// =====================================================================
void handleMissedReminder() {
  if (sysState != STATE_REMINDER) return;

  unsigned long elapsed = millis() - reminderTriggerMs;

  if (elapsed >= MISSED_REMINDER_MS && sysState == STATE_REMINDER) {
    sysState = STATE_MISSED;
    String msg = "Please take your medicine from the "
                 + activeContainer + " container.";

    Serial.println(F("[MISSED] ────────────────────────────────"));
    Serial.println("[MISSED] " + msg);
    Serial.println(F("[MISSED] ────────────────────────────────"));

    scrollMsg = msg;
    scrollX   = SCREEN_WIDTH;
    scheduleBuzz(2);
    drawReminderScreen(msg);

    // After missed alert, state stays MISSED
    // Touch still works (touchEnabled = true)
    // But don't re-trigger: update reminderTriggerMs to prevent loop
    reminderTriggerMs = millis() + 999999999UL; // Prevent repeated missed alerts
  }
}

// =====================================================================
//  NON-BLOCKING BUZZER SEQUENCER
// =====================================================================
// buzzStep tracks position in the beep sequence:
//   0 = idle
//   1 = 1st ON
//   2 = 1st OFF
//   3 = 2nd ON
//   4 = 2nd OFF → done

int           buzzTotal   = 0; // Total beeps requested
int           buzzCurrent = 0; // Current beep in sequence
bool          buzzing     = false;

void scheduleBuzz(int times) {
  buzzTotal      = times;
  buzzCurrent    = 0;
  buzzing        = true;
  buzzScheduledMs = millis();
  buzzStep       = 0;
  digitalWrite(BUZZER_PIN, HIGH); // Start first beep immediately
  Serial.print(F("[BUZZ] Beeping "));
  Serial.print(times);
  Serial.println(F(" time(s)"));
}

void handleBuzzer() {
  if (!buzzing) return;
  unsigned long now     = millis();
  unsigned long elapsed = now - buzzScheduledMs;

  // State machine for buzzer
  // Each beep = ON for BUZZ_ON_MS, then OFF for BUZZ_OFF_MS
  unsigned long period = BUZZ_ON_MS + BUZZ_OFF_MS;
  unsigned long cyclePos = elapsed % period;

  int beepsDone = elapsed / period;

  if (beepsDone >= buzzTotal) {
    // All beeps done
    digitalWrite(BUZZER_PIN, LOW);
    buzzing = false;
    return;
  }

  if (cyclePos < BUZZ_ON_MS) {
    digitalWrite(BUZZER_PIN, HIGH);
  } else {
    digitalWrite(BUZZER_PIN, LOW);
  }
}

// =====================================================================
//  SENSOR HANDLER  (DHT11 + Battery)
// =====================================================================
void handleSensors() {
  unsigned long now = millis();
  if (now - lastSensorReadMs < SENSOR_UPDATE_MS) return;
  lastSensorReadMs = now;

  float t = dht.readTemperature();
  float h = dht.readHumidity();

  if (!isnan(t) && !isnan(h)) {
    temperature = t;
    humidity    = h;
    Serial.print(F("[DHT11] Temp: "));
    Serial.print(temperature, 1);
    Serial.print(F("°C  Hum: "));
    Serial.print(humidity, 1);
    Serial.println(F("%"));
  } else {
    Serial.println(F("[DHT11] Read error!"));
  }

  batteryPct = readBatteryPercent();
  Serial.print(F("[BATT] "));
  Serial.print(batteryPct);
  Serial.println(F("%"));
}

// ── Battery percentage from ADC ─────────────────────────────────────
int readBatteryPercent() {
  // Average 10 samples for stability
  long sum = 0;
  for (int i = 0; i < 10; i++) {
    sum += analogRead(BATTERY_PIN);
    delayMicroseconds(100);
  }
  float adcAvg   = sum / 10.0f;
  float vMeasured = (adcAvg / ADC_RESOLUTION) * ADC_REF_V;
  float vBattery  = vMeasured * 2.0f; // ×2 for voltage divider

  int pct = (int)(((vBattery - BATTERY_MIN_V) / (BATTERY_MAX_V - BATTERY_MIN_V)) * 100.0f);
  return constrain(pct, 0, 100);
}

// =====================================================================
//  DISPLAY HANDLER
// =====================================================================
void handleDisplay() {
  unsigned long now = millis();

  if (sysState == STATE_IDLE) {
    // Refresh time every second
    if (now - lastDisplayUpdateMs >= DISPLAY_UPDATE_MS) {
      lastDisplayUpdateMs = now;
      drawIdleScreen();
    }
  } else if (sysState == STATE_REMINDER || sysState == STATE_MISSED) {
    // Scroll the reminder message
    if (now - lastScrollMs >= SCROLL_SPEED_MS) {
      lastScrollMs = now;
      scrollX -= 2;
      if (scrollX < -(int)(scrollMsg.length() * 6)) {
        scrollX = SCREEN_WIDTH;
      }
      display.clearDisplay();
      display.setTextSize(1);
      display.setTextColor(SSD1306_WHITE);
      // Header line
      display.setCursor(0, 0);
      display.print(sysState == STATE_MISSED ? F("!! MISSED !!") : F(">> REMINDER <<"));
      // Scrolling message line
      display.setCursor(scrollX, 12);
      display.print(scrollMsg);
      // Bottom: container color hint
      display.setCursor(0, 24);
      display.print(F("["));
      display.print(activeContainer);
      display.print(F("] Touch Sensor"));
      display.display();
    }
  } else if (sysState == STATE_DISPENSING) {
    if (now - lastDisplayUpdateMs >= DISPLAY_UPDATE_MS) {
      lastDisplayUpdateMs = now;
      display.clearDisplay();
      display.setCursor(0, 0);
      display.print(F("DISPENSING..."));
      display.setCursor(0, 12);
      display.print(F("Servo OPEN 180"));
      display.write(247); // Degree symbol
      display.setCursor(0, 24);
      display.print(F("Touch to close"));
      display.display();
    }
  }
}

// ── Idle screen: Time + Battery + Temp + Scroll ──────────────────────
void drawIdleScreen() {
  struct tm t;
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);

  if (getLocalTime(&t)) {
    // Row 0: Time
    display.setCursor(0, 0);
    char timeBuf[12];
    snprintf(timeBuf, sizeof(timeBuf), "%02d:%02d:%02d", t.tm_hour, t.tm_min, t.tm_sec);
    display.print(timeBuf);

    // Row 0 right: Battery %
    display.setCursor(80, 0);
    display.print(F("Bat:"));
    display.print(batteryPct);
    display.print(F("%"));

    // Row 1: Temp & Humidity
    display.setCursor(0, 12);
    display.print(F("T:"));
    display.print(temperature, 1);
    display.write(247);
    display.print(F("C H:"));
    display.print(humidity, 0);
    display.print(F("%"));

  } else {
    display.setCursor(0, 0);
    display.print(F("No time sync"));
  }

  // Row 2: Scrolling idle message
  scrollX -= 1;
  String idleMsg = "  * Smart Medicine Dispenser * Take meds on time! * ";
  if (scrollX < -(int)(idleMsg.length() * 6)) scrollX = SCREEN_WIDTH;
  display.setCursor(scrollX, 24);
  display.print(idleMsg);

  display.display();
}

// ── Reminder/Missed screen ───────────────────────────────────────────
void drawReminderScreen(const String& msg) {
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println(F("** MEDICINE TIME! **"));
  display.setCursor(0, 12);
  display.print(msg.substring(0, 20));
  display.setCursor(0, 22);
  display.print(msg.substring(20, 40));
  display.display();
}

// =====================================================================
//  SERVO CONTROL
// =====================================================================
void openDispenser() {
  servo1.write(SERVO_OPEN_ANGLE);
  servo2.write(SERVO_OPEN_ANGLE);
  dispenserOpen = true;
  Serial.println(F("[SERVO] Both servos → 180° (OPEN)"));
}

void closeDispenser() {
  servo1.write(SERVO_CLOSE_ANGLE);
  servo2.write(SERVO_CLOSE_ANGLE);
  dispenserOpen = false;
  Serial.println(F("[SERVO] Both servos → 0° (CLOSED)"));
}

// =====================================================================
//  UTILITY FUNCTIONS
// =====================================================================

// ── Get container color based on current hour ─────────────────────
const char* getContainerColor(int hour) {
  for (int i = 0; i < containerMapSize; i++) {
    if (hour >= containerMap[i].startHour && hour < containerMap[i].endHour) {
      return containerMap[i].containerColor;
    }
  }
  return "Unknown";
}

// ── Get formatted time string ─────────────────────────────────────
String getFormattedTime() {
  struct tm t;
  if (!getLocalTime(&t)) return "--:--:--";
  char buf[20];
  snprintf(buf, sizeof(buf), "%02d:%02d:%02d", t.tm_hour, t.tm_min, t.tm_sec);
  return String(buf);
}

/* =====================================================================
   END OF FILE
   ===================================================================== */
