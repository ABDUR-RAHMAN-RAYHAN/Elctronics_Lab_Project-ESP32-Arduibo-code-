Robot unit code:-
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include "BluetoothSerial.h"

// ---------------- OLED ----------------
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// ---------------- BLUETOOTH ----------------
BluetoothSerial SerialBT;

// ---------------- MOTOR PINS ----------------
#define IN1 25
#define IN2 26
#define IN3 27
#define IN4 14

// ---------------- SENSORS ----------------
#define TRIG 18
#define ECHO 19
#define IR_LEFT 32
#define IR_RIGHT 33

// ---------------- SPEAKER ----------------
#define SPEAKER 23

// ---------------- DISTANCE SETTINGS ----------------
#define FOLLOW_DISTANCE 40   // cm
#define STOP_DISTANCE   10   // cm

bool rescueMode = false;

// ---------------- SETUP ----------------
void setup() {
  Serial.begin(115200);

  SerialBT.begin("Rescue_Robot", true);   // MASTER
  SerialBT.connect("GROUND_BT");

  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  stopMotors();

  pinMode(TRIG, OUTPUT);
  pinMode(ECHO, INPUT);

  // IMPORTANT: Use internal pullups for IR sensors
  pinMode(IR_LEFT, INPUT_PULLUP);
  pinMode(IR_RIGHT, INPUT_PULLUP);

  pinMode(SPEAKER, OUTPUT);

  Wire.begin(21, 22);
  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  display.clearDisplay();
  display.setCursor(0,0);
  display.println("Robot Ready");
  display.display();
}

// ---------------- LOOP ----------------
void loop() {

  if (SerialBT.available()) {
    String msg = SerialBT.readStringUntil('\n');
    msg.trim();
    if (msg == "ALERT") rescueMode = true;
  }

  if (rescueMode) {
    rescueAction();
  } else {
    followHuman();
  }
}

// ---------------- FOLLOW MODE ----------------
void followHuman() {

  float distance = readDistance();

  // NO TARGET → STOP
  if (distance > FOLLOW_DISTANCE) {
    stopMotors();
    return;
  }

  // TOO CLOSE → STOP
  if (distance <= STOP_DISTANCE) {
    stopMotors();
    return;
  }

  // IR is ACTIVE LOW
  bool leftDetected  = digitalRead(IR_LEFT)  == LOW;
  bool rightDetected = digitalRead(IR_RIGHT) == LOW;

  // Direction assist ONLY
  if (rightDetected && !leftDetected) {
    turnRightSlow();
  }
  else if (leftDetected && !rightDetected) {
    turnLeftSlow();
  }
  else {
    moveForward();
  }
}

// ---------------- MOTOR CONTROL ----------------
void moveForward() {
  digitalWrite(IN1,HIGH); digitalWrite(IN2,LOW);
  digitalWrite(IN3,HIGH); digitalWrite(IN4,LOW);
}

void turnLeftSlow() {
  digitalWrite(IN1,LOW); digitalWrite(IN2,LOW);
  digitalWrite(IN3,HIGH); digitalWrite(IN4,LOW);
}

void turnRightSlow() {
  digitalWrite(IN1,HIGH); digitalWrite(IN2,LOW);
  digitalWrite(IN3,LOW);  digitalWrite(IN4,LOW);
}

void stopMotors() {
  digitalWrite(IN1,LOW); digitalWrite(IN2,LOW);
  digitalWrite(IN3,LOW); digitalWrite(IN4,LOW);
}

// ---------------- ULTRASONIC ----------------
float readDistance() {
  digitalWrite(TRIG,LOW); delayMicroseconds(2);
  digitalWrite(TRIG,HIGH); delayMicroseconds(10);
  digitalWrite(TRIG,LOW);

  long duration = pulseIn(ECHO,HIGH,30000);
  if (duration == 0) return 999;

  return duration * 0.034 / 2;
}

// ---------------- RESCUE MODE ----------------
void rescueAction() {
  stopMotors();

  display.clearDisplay();
  display.setCursor(0,0);
  display.println("!! ALERT !!");
  display.println("EARTHQUAKE");
  display.display();

  tone(SPEAKER,3000); delay(200);
  tone(SPEAKER,3500); delay(200);
  noTone(SPEAKER);
}
