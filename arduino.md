#include <Arduino.h>

#define BUZZER_PIN 5
#define LED_RED_PIN 9
#define LED_YELLOW_PIN 10
#define LED_GREEN_PIN 11

// Function prototypes
void buzzBuzzer();
void setLED(int ledPin);

void setup() {
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LED_RED_PIN, OUTPUT);
  pinMode(LED_YELLOW_PIN, OUTPUT);
  pinMode(LED_GREEN_PIN, OUTPUT);

  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(LED_RED_PIN, LOW);
  digitalWrite(LED_YELLOW_PIN, LOW);
  digitalWrite(LED_GREEN_PIN, LOW);

  Serial.begin(9600);
}

void loop() {
  if (Serial.available()) {
    String cmd = Serial.readStringUntil('\n');
    cmd.trim();

    if (cmd == "BUZZER_ON") {
      buzzBuzzer();
    } else if (cmd == "LED_RED") {
      setLED(LED_RED_PIN);
    } else if (cmd == "LED_YELLOW") {
      setLED(LED_YELLOW_PIN);
    } else if (cmd == "LED_GREEN") {
      setLED(LED_GREEN_PIN);
    }
  }
}

void buzzBuzzer() {
  digitalWrite(BUZZER_PIN, HIGH);
  delay(2000);
  digitalWrite(BUZZER_PIN, LOW);
}

void setLED(int ledPin) {
  digitalWrite(LED_RED_PIN, LOW);
  digitalWrite(LED_YELLOW_PIN, LOW);
  digitalWrite(LED_GREEN_PIN, LOW);

  digitalWrite(ledPin, HIGH);
}
