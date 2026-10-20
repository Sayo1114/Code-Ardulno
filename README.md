/*
  Project: Piezoelectric Energy-Harvesting Tile
  Author: นายธนกร บุญสอน (ส้มโอ)
  Description: โครงสร้างโค้ดพื้นฐานสำหรับอ่านค่าจาก Piezo Sensor และแสดงผล
*/


#include <LiquidCrystal.h>

// ----- LCD SETUP (comment out if no LCD attached) -----
#define USE_LCD true
LiquidCrystal lcd(12, 11, 5, 4, 3, 6);

// ----- PIN DEFINITIONS -----
const int VOLTAGE_SENSE_PIN = A0;   // harvested voltage (after divider)
const int CHARGE_RELAY_PIN  = 7;    // switches charging path / indicator LED
const int FULL_LED_PIN      = 8;    // "battery full" indicator

// ----- VOLTAGE DIVIDER CALIBRATION -----
// Vout_divider = Vharvested * (R2 / (R1 + R2))
// With R1 = 100k, R2 = 10k  ->  ratio = 10/110 = 0.0909
const float R1 = 100000.0;
const float R2 = 10000.0;
const float DIVIDER_RATIO = R2 / (R1 + R2);
const float ADC_REF_VOLTAGE = 5.0;
const int   ADC_RESOLUTION  = 1023;

// ----- THRESHOLDS (tune to your hardware) -----
const float STEP_DETECT_THRESHOLD = 0.5;   // volts, spike counted as a step
const float CHARGE_START_VOLTAGE  = 1.0;   // volts, enough to start charging relay
const float BATTERY_FULL_VOLTAGE  = 4.2;   // volts, treat as "full" (Li-ion example)

// ----- STATE VARIABLES -----
unsigned long stepCount = 0;
float lastVoltage = 0.0;
float peakVoltage = 0.0;
unsigned long lastStepTime = 0;
const unsigned long STEP_DEBOUNCE_MS = 150; // avoid double counting one step

// Rough energy accumulator (very approximate, for display purposes only)
float estimatedEnergy_mJ = 0.0; // millijoules
unsigned long lastSampleTime = 0;

void setup() {
  Serial.begin(9600);

  pinMode(VOLTAGE_SENSE_PIN, INPUT);
  pinMode(CHARGE_RELAY_PIN, OUTPUT);
  pinMode(FULL_LED_PIN, OUTPUT);
  digitalWrite(CHARGE_RELAY_PIN, LOW);
  digitalWrite(FULL_LED_PIN, LOW);

#if USE_LCD
  lcd.begin(16, 2);
  lcd.print("Footstep Power");
  lcd.setCursor(0, 1);
  lcd.print("Generator Init..");
  delay(1500);
  lcd.clear();
#endif

  lastSampleTime = millis();
  Serial.println("Footstep Power Generator - System Ready");
}

void loop() {
  float harvestedVoltage = readHarvestedVoltage();

  // Track peak voltage seen (useful for calibration/debug)
  if (harvestedVoltage > peakVoltage) {
    peakVoltage = harvestedVoltage;
  }

  // ---- Step detection: rising edge above threshold with debounce ----
  unsigned long now = millis();
  if (harvestedVoltage >= STEP_DETECT_THRESHOLD &&
      lastVoltage < STEP_DETECT_THRESHOLD &&
      (now - lastStepTime) > STEP_DEBOUNCE_MS) {
    stepCount++;
    lastStepTime = now;
    Serial.print("Step detected! Total steps: ");
    Serial.println(stepCount);
  }

  // ---- Very rough energy estimate: P ~ V^2 / R_load, integrated over time ----
  unsigned long dt = now - lastSampleTime; // ms
  if (dt > 0) {
    float assumedLoadResistance = 1000.0; // ohms, adjust to your storage circuit
    float power_mW = (harvestedVoltage * harvestedVoltage / assumedLoadResistance) * 1000.0;
    estimatedEnergy_mJ += power_mW * (dt / 1000.0);
    lastSampleTime = now;
  }

  // ---- Charging control ----
  if (harvestedVoltage >= CHARGE_START_VOLTAGE && harvestedVoltage < BATTERY_FULL_VOLTAGE) {
    digitalWrite(CHARGE_RELAY_PIN, HIGH); // enable charging path
    digitalWrite(FULL_LED_PIN, LOW);
  } else if (harvestedVoltage >= BATTERY_FULL_VOLTAGE) {
    digitalWrite(CHARGE_RELAY_PIN, LOW);  // stop charging, battery full
    digitalWrite(FULL_LED_PIN, HIGH);
  } else {
    digitalWrite(CHARGE_RELAY_PIN, LOW);  // not enough voltage yet
    digitalWrite(FULL_LED_PIN, LOW);
  }

  // ---- Display ----
  updateDisplay(harvestedVoltage);

  lastVoltage = harvestedVoltage;
  delay(50); // sample rate ~20Hz, adjust as needed
}

// Reads ADC pin, converts back through divider ratio to real harvested voltage
float readHarvestedVoltage() {
  int raw = analogRead(VOLTAGE_SENSE_PIN);
  float adcVoltage = (raw / (float)ADC_RESOLUTION) * ADC_REF_VOLTAGE;
  float actualVoltage = adcVoltage / DIVIDER_RATIO;
  return actualVoltage;
}

void updateDisplay(float voltage) {
  Serial.print("Voltage: ");
  Serial.print(voltage, 2);
  Serial.print(" V | Steps: ");
  Serial.print(stepCount);
  Serial.print(" | Energy: ");
  Serial.print(estimatedEnergy_mJ, 1);
  Serial.println(" mJ");

#if USE_LCD
  lcd.setCursor(0, 0);
  lcd.print("V:");
  lcd.print(voltage, 2);
  lcd.print("V St:");
  lcd.print(stepCount);
  lcd.print("   "); // clear leftover chars

  lcd.setCursor(0, 1);
  if (voltage >= BATTERY_FULL_VOLTAGE) {
    lcd.print("Battery FULL    ");
  } else if (voltage >= CHARGE_START_VOLTAGE) {
    lcd.print("Charging...     ");
  } else {
    lcd.print("Waiting steps.. ");
  }
#endif
}
