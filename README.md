#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>
#include <SoftwareSerial.h>

// ===== LCD =====
LiquidCrystal_I2C lcd(0x3F, 16, 2);

// ===== GSM =====
SoftwareSerial gsm(3, 4); // RX, TX
String phoneNumber = "+919043011454"; // CHANGE NUMBER
bool alertSent = false;

// ===== Ultrasonic =====
const int trigPin = 9;
const int echoPin = 8;

// ===== Alcohol =====
const int alcoholPin = A0;

// ===== LDR =====
const int ldrPin = A1;

// ===== Vibration =====
const int vibrationPin = A2;

// ===== Buzzer & Camera =====
const int buzzer = 10;
const int cam = 12;

// ===== IR SPEED SENSOR =====
const int irPin = 2;

// ===== SEAT BELT BUTTON =====
const int seatBeltPin = 11;

// ===== SPEED =====
volatile int pulseCount = 0;
unsigned long lastTime = 0;
float speed = 0;

// ===== DHT =====
#define DHTPIN 7
#define DHTTYPE DHT22
DHT dht(DHTPIN, DHTTYPE);

// ===== VARIABLES =====
long duration;
int distance;

// ===== INTERRUPT =====
void countPulse() {
  pulseCount++;
}

// ===== SEND SMS =====
void sendSMS(String message) {
  gsm.println("AT+CMGF=1");
  delay(1000);

  gsm.println("AT+CMGS=\"" + phoneNumber + "\"");
  delay(1000);

  gsm.print(message);
  delay(500);

  gsm.write(26); // CTRL+Z
  delay(5000);
}

void setup() {
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
  pinMode(buzzer, OUTPUT);
  pinMode(cam, OUTPUT);

  pinMode(irPin, INPUT);
  pinMode(seatBeltPin, INPUT_PULLUP);

  lcd.init();
  lcd.backlight();

  Serial.begin(9600);
  gsm.begin(9600);

  dht.begin();

  attachInterrupt(digitalPinToInterrupt(irPin), countPulse, FALLING);

  // GSM INIT
  delay(2000);
  gsm.println("AT");
  delay(1000);
  gsm.println("AT+CMGF=1");
  delay(1000);

  lastTime = millis();
}

void loop() {

  // ===== ULTRASONIC =====
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);

  duration = pulseIn(echoPin, HIGH);
  distance = duration * 0.034 / 2;

  // ===== SENSOR VALUES =====
  float alcohol = analogRead(alcoholPin) * (100.0 / 1023.0);
  float ldr = analogRead(ldrPin) * (100.0 / 1023.0);

  float vibration = analogRead(vibrationPin) * (100.0 / 1023.0);
  vibration = 100 - vibration;

  // ===== DHT =====
  float temp = dht.readTemperature();
  float hum = dht.readHumidity();

  if (isnan(temp) || isnan(hum)) {
    temp = 0;
    hum = 0;
  }

  // ===== SPEED =====
  unsigned long currentTime = millis();
  if (currentTime - lastTime >= 1000) {
    speed = pulseCount * 1.0;
    pulseCount = 0;
    lastTime = currentTime;
  }

  // ===== SEAT BELT =====
  bool seatBelt = digitalRead(seatBeltPin); // LOW = ON

  // ===== ABNORMAL CHECK =====
  bool abnormal = false;
  String msg = "ALERT:\n";

  if (seatBelt == HIGH) {
    abnormal = true;
    msg += "Seat Belt OFF\n";
  }

  if (alcohol > 55) {
    abnormal = true;
    msg += "Alcohol Detected\n";
  }

  if (temp > 45) {
    abnormal = true;
    msg += "High Temp\n";
  }

  if (vibration > 45) {
    abnormal = true;
    msg += "High Vibration\n";
  }

  if (ldr > 70) {
    abnormal = true;
    msg += "Too Bright\n";
  }

  if (distance < 25) {
    abnormal = true;
    msg += "Object Close\n";
    digitalWrite(cam, HIGH);
    delay(500);
  } else {
    digitalWrite(cam, LOW);
  }

  // ===== BUZZER =====
  if (abnormal) {
    digitalWrite(buzzer, HIGH);
  } else {
    digitalWrite(buzzer, LOW);
    alertSent = false;
  }

  // ===== SEND SMS =====
  if (abnormal && !alertSent) {
    sendSMS(msg);
    alertSent = true;
  }

  // ===== SERIAL OUTPUT =====
  Serial.print(distance); Serial.print(",");
  Serial.print(speed); Serial.print(",");
  Serial.print(alcohol); Serial.print(",");
  Serial.print(temp); Serial.print(",");
  Serial.print(hum); Serial.print(",");
  Serial.print(vibration); Serial.print(",");
  Serial.print(ldr); Serial.print(",");
  Serial.println('\n');

  // ===== LCD DISPLAY =====
  if (abnormal) {

    // SHOW MULTIPLE ALERTS ONE BY ONE
    if (seatBelt == HIGH) {
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("!!! ALERT !!!");
      lcd.setCursor(0, 1);
      lcd.print("SeatBelt OFF");
      delay(1500);
    }

    if (alcohol > 55) {
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("!!! ALERT !!!");
      lcd.setCursor(0, 1);
      lcd.print("Alcohol Detected");
      delay(1500);
    }

    if (temp > 45) {
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("!!! ALERT !!!");
      lcd.setCursor(0, 1);
      lcd.print("High Temp");
      delay(1500);
    }

    if (vibration > 45) {
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("!!! ALERT !!!");
      lcd.setCursor(0, 1);
      lcd.print("High Vibration");
      delay(1500);
    }

    if (ldr > 70) {
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("!!! ALERT !!!");
      lcd.setCursor(0, 1);
      lcd.print("Too Bright");
      delay(1500);
    }

    if (distance < 25) {
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("!!! ALERT !!!");
      lcd.setCursor(0, 1);
      lcd.print("Object Close");
      delay(1500);
    }

  } else {

    // ===== NORMAL DISPLAY =====
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("D:");
    lcd.print(distance);
    lcd.print(" S:");
    lcd.print(speed);

    lcd.setCursor(0, 1);
    lcd.print("A:");
    lcd.print(alcohol);
    lcd.print(" SB:");

    if (seatBelt == LOW)
      lcd.print("ON");
    else
      lcd.print("OFF");

    delay(2000);

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("T:");
    lcd.print(temp);
    lcd.print(" L:");
    lcd.print(ldr);

    lcd.setCursor(0, 1);
    lcd.print("H:");
    lcd.print(hum);
    lcd.print(" V:");
    lcd.print(vibration);

    delay(2000);
  }
}
