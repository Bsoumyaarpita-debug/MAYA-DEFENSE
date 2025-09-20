#include "heltec.h"
#include <RadioLib.h>

#define LORA_NSS   8
#define LORA_RST   12
#define LORA_DIO1  14
#define LORA_BUSY  13
#define LORA_FREQ  433.0

SX1262 lora = new Module(LORA_NSS, LORA_DIO1, LORA_RST, LORA_BUSY);

unsigned long lastSend = 0;
int counter = 0;

void setup() {
  Serial.begin(115200);
  Heltec.begin(true, false, true);

  int state = lora.begin(LORA_FREQ);
  Heltec.display->clear();
  if (state == RADIOLIB_ERR_NONE) Heltec.display->drawString(0, 0, "LoRa init OK");
  else Heltec.display->drawString(0, 0, "LoRa init Fail");
  Heltec.display->display();
  Serial.print("LoRa begin: "); Serial.println(state);
  delay(1500);
}

void loop() {
  unsigned long now = millis();
  if (now - lastSend > 5000) {
    lastSend = now;
    char message[16];
    snprintf(message, sizeof(message), "Hello %d", counter++);
    Serial.print("Sending: "); Serial.println(message);

    int state = lora.transmit((uint8_t*)message, strlen(message));
    Heltec.display->clear();
    if (state == RADIOLIB_ERR_NONE) {
      Heltec.display->drawString(0, 0, "Sent:");
      Heltec.display->drawString(0, 16, message);
    } else {
      Heltec.display->drawString(0, 0, "Transmit failed");
    }
    Heltec.display->display();
  }
}
