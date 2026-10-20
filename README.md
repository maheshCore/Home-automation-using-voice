#define BLYNK_TEMPLATE_ID "TMPL3RATyWFzJ"
#define BLYNK_TEMPLATE_NAME "Aithinker home automation"
#define BLYNK_AUTH_TOKEN "CLAzFcdD-epxZn24zgmQZ0DZlEHS2Tl6"

#include <Arduino.h>
#include <WiFi.h>
#include <Wire.h>
#include <BlynkSimpleEsp32.h>
#include <LiquidCrystal_I2C.h>

// ================= WIFI =================

char ssid[] = "Pixel";
char pass[] = "12345678";

BlynkTimer timer;

// ================= VC-02 =================

#define VC02_RX 16
#define VC02_TX 17

HardwareSerial VC02(2);

// ================= LCD =================

LiquidCrystal_I2C lcd(0x27, 16, 2);

// ================= RELAYS =================

#define LIGHT_RELAY 18
#define FAN_RELAY   19
#define AC_RELAY    23
#define PUMP_RELAY  25

#define RELAY_ON  LOW
#define RELAY_OFF HIGH

// ================= BUTTONS =================

#define LIGHT_BUTTON 13
#define FAN_BUTTON   14
#define AC_BUTTON    32
#define PUMP_BUTTON  33

// ================= SENSORS =================

#define MOISTURE_PIN 34
#define PIR_PIN      27
#define BUZZER_PIN   26

// ================= MOISTURE CALIBRATION =================

#define DRY_VALUE 3000
#define WET_VALUE 1200

#define PUMP_ON_LEVEL  20
#define PUMP_OFF_LEVEL 90

// ================= STATES =================

bool lightState = false;
bool fanState = false;
bool acState = false;
bool pumpState = false;

bool autoMode = true;

int moistureRaw = 0;
int moisturePercent = 0;

bool motionDetected = false;

// ================= VOICE COMMAND DATA =================

uint8_t previousByte = 0;
bool havePreviousByte = false;

// ================= LCD =================

int screenNumber = 0;

unsigned long lastScreenChange = 0;

const unsigned long screenInterval = 3000;

// ================= BUTTON STRUCTURE =================

const unsigned long debounceDelay = 40;

struct Button
{
  uint8_t pin;
  bool lastReading;
  bool stableState;
  unsigned long lastChange;
};

Button lightButton = {LIGHT_BUTTON, HIGH, HIGH, 0};
Button fanButton   = {FAN_BUTTON, HIGH, HIGH, 0};
Button acButton    = {AC_BUTTON, HIGH, HIGH, 0};
Button pumpButton  = {PUMP_BUTTON, HIGH, HIGH, 0};

// ================= MOTION COUNTER =================

int motionCount = 0;

bool previousMotion = LOW;
bool buzzerActive = false;

unsigned long buzzerStartTime = 0;

const unsigned long buzzerDuration = 1000;

// =================================================
// RELAY INITIALIZATION
// =================================================

void applyRelayStates()
{
  digitalWrite(LIGHT_RELAY, lightState ? RELAY_ON : RELAY_OFF);
  digitalWrite(FAN_RELAY, fanState ? RELAY_ON : RELAY_OFF);
  digitalWrite(AC_RELAY, acState ? RELAY_ON : RELAY_OFF);
  digitalWrite(PUMP_RELAY, pumpState ? RELAY_ON : RELAY_OFF);
}

// =================================================
// BLYNK UPDATE — CORRECTED V4 AND V5
// V4 = SOIL MOISTURE
// V5 = PIR MOTION
// =================================================

void updateBlynk()
{
  if (!Blynk.connected())
  {
    return;
  }

  Blynk.virtualWrite(V0, lightState ? 1 : 0);
  Blynk.virtualWrite(V1, fanState ? 1 : 0);
  Blynk.virtualWrite(V2, acState ? 1 : 0);
  Blynk.virtualWrite(V3, pumpState ? 1 : 0);

  // CORRECT: V4 is soil moisture percentage
  Blynk.virtualWrite(V4, moisturePercent);

  // CORRECT: V5 is PIR motion status
  Blynk.virtualWrite(V5, motionDetected ? 1 : 0);

  Blynk.virtualWrite(V6, autoMode ? 1 : 0);
}

// =================================================
// LIGHT CONTROL
// =================================================

void setLight(bool state)
{
  lightState = state;

  digitalWrite(LIGHT_RELAY, lightState ? RELAY_ON : RELAY_OFF);

  Serial.println(lightState ? "LIGHT ON" : "LIGHT OFF");

  updateBlynk();
}

// =================================================
// FAN CONTROL
// =================================================

void setFan(bool state)
{
  fanState = state;

  digitalWrite(FAN_RELAY, fanState ? RELAY_ON : RELAY_OFF);

  Serial.println(fanState ? "FAN ON" : "FAN OFF");

  updateBlynk();
}

// =================================================
// AC CONTROL
// =================================================

void setAC(bool state)
{
  acState = state;

  digitalWrite(AC_RELAY, acState ? RELAY_ON : RELAY_OFF);

  Serial.println(acState ? "AC ON" : "AC OFF");

  updateBlynk();
}

// =================================================
// PUMP CONTROL
// =================================================

void setPump(bool state)
{
  pumpState = state;

  digitalWrite(PUMP_RELAY, pumpState ? RELAY_ON : RELAY_OFF);

  Serial.println(pumpState ? "PUMP ON" : "PUMP OFF");

  updateBlynk();
}

// =================================================
// AUTOMATIC / MANUAL MODE
// =================================================

void setAutoMode(bool state)
{
  autoMode = state;

  Serial.println(autoMode ? "MODE: AUTO" : "MODE: MANUAL");

  updateBlynk();
}

// =================================================
// READ SOIL MOISTURE
// =================================================

void readMoisture()
{
  moistureRaw = analogRead(MOISTURE_PIN);

  moisturePercent = map(
    moistureRaw,
    DRY_VALUE,
    WET_VALUE,
    0,
    100
  );

  moisturePercent = constrain(moisturePercent, 0, 100);

  Serial.print("Soil Raw: ");
  Serial.print(moistureRaw);

  Serial.print(" | Moisture: ");
  Serial.print(moisturePercent);
  Serial.println("%");

  // V4 receives moisture only
  if (Blynk.connected())
  {
    Blynk.virtualWrite(V4, moisturePercent);
  }
}

// =================================================
// AUTOMATIC IRRIGATION
// =================================================

void automaticIrrigation()
{
  if (!autoMode)
  {
    return;
  }

  if (moisturePercent <= PUMP_ON_LEVEL && !pumpState)
  {
    Serial.println("SOIL DRY - PUMP ON");

    setPump(true);
  }
  else if (moisturePercent >= PUMP_OFF_LEVEL && pumpState)
  {
    Serial.println("SOIL WET - PUMP OFF");

    setPump(false);
  }
}

// =================================================
// PIR MOTION AND BUZZER
// BUZZER AFTER 5 MOTION EVENTS
// =================================================

void updateMotion()
{
  bool currentMotion = digitalRead(PIR_PIN);

  motionDetected = (currentMotion == HIGH);

  if (currentMotion == HIGH && previousMotion == LOW)
  {
    motionCount++;

    Serial.print("Motion count: ");
    Serial.println(motionCount);

    if (motionCount >= 5)
    {
      digitalWrite(BUZZER_PIN, HIGH);

      buzzerActive = true;
      buzzerStartTime = millis();

      motionCount = 0;

      Serial.println("FIVE MOTIONS DETECTED - BUZZER ON");
    }
  }

  previousMotion = currentMotion;

  if (buzzerActive &&
      millis() - buzzerStartTime >= buzzerDuration)
  {
    digitalWrite(BUZZER_PIN, LOW);

    buzzerActive = false;

    Serial.println("BUZZER OFF");
  }

  // V5 receives PIR status only
  if (Blynk.connected())
  {
    Blynk.virtualWrite(V5, motionDetected ? 1 : 0);
  }
}

// =================================================
// PROCESS VC-02 COMMANDS
// =================================================

void processCommand(uint8_t command1, uint8_t command2)
{
  Serial.print("VC-02 HEX: ");
  Serial.print(command1, HEX);
  Serial.print(" ");
  Serial.println(command2, HEX);

  if (command1 == 0xA1 && command2 == 0x90)
  {
    setLight(true);
  }
  else if (command1 == 0xA1 && command2 == 0x91)
  {
    setLight(false);
  }
  else if (command1 == 0xA2 && command2 == 0x80)
  {
    setFan(true);
  }
  else if (command1 == 0xA2 && command2 == 0x85)
  {
    setFan(false);
  }
  else if (command1 == 0xA3 && command2 == 0x90)
  {
    setAC(true);
  }
  else if (command1 == 0xA3 && command2 == 0x95)
  {
    setAC(false);
  }
  else if (command1 == 0xB1 && command2 == 0x90)
  {
    setAutoMode(false);
    setPump(true);
  }
  else if (command1 == 0xB1 && command2 == 0x95)
  {
    setAutoMode(false);
    setPump(false);
  }
  else
  {
    Serial.println("Unknown VC-02 command");
  }
}

// =================================================
// READ VC-02
// =================================================

void readVC02()
{
  while (VC02.available() > 0)
  {
    uint8_t incomingByte = VC02.read();

    Serial.print("Received HEX: 0x");
    Serial.println(incomingByte, HEX);

    if (incomingByte == 0xA1 ||
        incomingByte == 0xA2 ||
        incomingByte == 0xA3 ||
        incomingByte == 0xB1)
    {
      previousByte = incomingByte;
      havePreviousByte = true;
    }
    else if (havePreviousByte)
    {
      processCommand(previousByte, incomingByte);
      havePreviousByte = false;
    }
  }
}

// =================================================
// PUSH BUTTON DEBOUNCING
// =================================================

bool buttonPressed(Button &button)
{
  bool reading = digitalRead(button.pin);

  unsigned long now = millis();

  if (reading != button.lastReading)
  {
    button.lastChange = now;
    button.lastReading = reading;
  }

  if (now - button.lastChange >= debounceDelay &&
      reading != button.stableState)
  {
    button.stableState = reading;

    if (button.stableState == LOW)
    {
      return true;
    }
  }

  return false;
}

// =================================================
// PUSH BUTTON CONTROL
// =================================================

void handlePushButtons()
{
  if (buttonPressed(lightButton))
  {
    setLight(!lightState);
  }

  if (buttonPressed(fanButton))
  {
    setFan(!fanState);
  }

  if (buttonPressed(acButton))
  {
    setAC(!acState);
  }

  if (buttonPressed(pumpButton))
  {
    setAutoMode(false);
    setPump(!pumpState);
  }
}

// =================================================
// LCD DISPLAY
// =================================================

void updateLCD()
{
  unsigned long now = millis();

  if (now - lastScreenChange >= screenInterval)
  {
    lastScreenChange = now;

    screenNumber++;

    if (screenNumber > 2)
    {
      screenNumber = 0;
    }

    lcd.clear();
  }

  if (screenNumber == 0)
  {
    lcd.setCursor(0, 0);

    lcd.print("L:");
    lcd.print(lightState ? "ON " : "OFF");

    lcd.print(" F:");
    lcd.print(fanState ? "ON " : "OFF");

    lcd.setCursor(0, 1);

    lcd.print("AC:");
    lcd.print(acState ? "ON " : "OFF");

    lcd.print(" P:");
    lcd.print(pumpState ? "ON " : "OFF");

    lcd.print(" ");
  }
  else if (screenNumber == 1)
  {
    lcd.setCursor(0, 0);
    lcd.print("Soil Moisture");

    lcd.setCursor(0, 1);

    lcd.print(moisturePercent);
    lcd.print("% ");

    lcd.print(autoMode ? "AUTO " : "MANUAL");
  }
  else
  {
    lcd.setCursor(0, 0);
    lcd.print("PIR Motion");

    lcd.setCursor(0, 1);

    lcd.print(motionDetected
              ? "MOTION DETECTED"
              : "NO MOTION      ");
  }
}

// =================================================
// BLYNK CONNECTED
// =================================================

BLYNK_CONNECTED()
{
  Serial.println("BLYNK CONNECTED");

  // Send current readings using the correct pin mapping.
  Blynk.virtualWrite(V0, lightState ? 1 : 0);
  Blynk.virtualWrite(V1, fanState ? 1 : 0);
  Blynk.virtualWrite(V2, acState ? 1 : 0);
  Blynk.virtualWrite(V3, pumpState ? 1 : 0);

  Blynk.virtualWrite(V4, moisturePercent);
  Blynk.virtualWrite(V5, motionDetected ? 1 : 0);
  Blynk.virtualWrite(V6, autoMode ? 1 : 0);
}

// =================================================
// BLYNK V0 - LIGHT
// =================================================

BLYNK_WRITE(V0)
{
  setLight(param.asInt() == 1);
}

// =================================================
// BLYNK V1 - FAN
// =================================================

BLYNK_WRITE(V1)
{
  setFan(param.asInt() == 1);
}

// =================================================
// BLYNK V2 - AC
// =================================================

BLYNK_WRITE(V2)
{
  setAC(param.asInt() == 1);
}

// =================================================
// BLYNK V3 - PUMP
// =================================================

BLYNK_WRITE(V3)
{
  setAutoMode(false);
  setPump(param.asInt() == 1);
}

// =================================================
// BLYNK V6 - AUTO / MANUAL
// =================================================

BLYNK_WRITE(V6)
{
  setAutoMode(param.asInt() == 1);

  if (autoMode)
  {
    automaticIrrigation();
  }
}

// =================================================
// WIFI AND BLYNK RECONNECTION
// =================================================

void reconnectConnections()
{
  if (WiFi.status() != WL_CONNECTED)
  {
    Serial.println("WiFi disconnected - reconnecting");
    WiFi.reconnect();
    return;
  }

  if (!Blynk.connected())
  {
    Serial.println("Reconnecting to Blynk");
    Blynk.connect(1000);
  }
}

// =================================================
// SETUP
// =================================================

void setup()
{
  Serial.begin(115200);

  // Relays
  pinMode(LIGHT_RELAY, OUTPUT);
  pinMode(FAN_RELAY, OUTPUT);
  pinMode(AC_RELAY, OUTPUT);
  pinMode(PUMP_RELAY, OUTPUT);

  lightState = false;
  fanState = false;
  acState = false;
  pumpState = false;

  applyRelayStates();

  // Buttons
  pinMode(LIGHT_BUTTON, INPUT_PULLUP);
  pinMode(FAN_BUTTON, INPUT_PULLUP);
  pinMode(AC_BUTTON, INPUT_PULLUP);
  pinMode(PUMP_BUTTON, INPUT_PULLUP);

  // Sensors
  pinMode(MOISTURE_PIN, INPUT);
  pinMode(PIR_PIN, INPUT);

  // Buzzer
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);

  analogReadResolution(12);

  // LCD
  Wire.begin(21, 22);

  lcd.init();
  lcd.backlight();

  lcd.setCursor(0, 0);
  lcd.print("Smart Home");

  lcd.setCursor(0, 1);
  lcd.print("Starting...");

  // Voice module
  VC02.begin(9600, SERIAL_8N1, VC02_RX, VC02_TX);

  // WiFi
  WiFi.mode(WIFI_STA);
  WiFi.setAutoReconnect(true);
  WiFi.begin(ssid, pass);

  // Blynk
  Blynk.config(BLYNK_AUTH_TOKEN);

  // Read sensors and update Blynk regularly
  timer.setInterval(2000L, readMoisture);
  timer.setInterval(100L, updateMotion);
  timer.setInterval(500L, automaticIrrigation);
  timer.setInterval(1000L, updateBlynk);
  timer.setInterval(5000L, reconnectConnections);

  // Initial measurement
  readMoisture();

  Serial.println("Smart Home Automation Started");
  Serial.println("V4 = SOIL MOISTURE");
  Serial.println("V5 = PIR MOTION");
}

// =================================================
// LOOP
// =================================================

void loop()
{
  if (Blynk.connected())
  {
    Blynk.run();
  }

  timer.run();

  readVC02();

  handlePushButtons();

  updateLCD();
}


