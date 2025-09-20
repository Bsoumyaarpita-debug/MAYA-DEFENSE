#include "esp_camera.h"
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <HTTPClient.h>

// Replace with your network credentials
const char* ssid = "Galaxy A23 5G E06F";
const char* password = "dtte7337";

// AI Server HTTPS endpoint
const char* serverUrl = "https://hq2.example.com/api/ai_snapshot";  // Replace with your AI server URL

// Root CA certificate of the AI server (PEM format)
const char* rootCACertificate = \
"-----BEGIN CERTIFICATE-----\n" \
"MIID...YourRootCACertHere...==\n" \
"-----END CERTIFICATE-----\n";

// Camera configuration (depends on your ESP32-CAM module)
void setupCamera() {
  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer = LEDC_TIMER_0;
  config.pin_d0 = 5;
  config.pin_d1 = 18;
  config.pin_d2 = 19;
  config.pin_d3 = 21;
  config.pin_d4 = 36;
  config.pin_d5 = 39;
  config.pin_d6 = 34;
  config.pin_d7 = 35;
  config.pin_xclk = 0;
  config.pin_pclk = 22;
  config.pin_vsync = 25;
  config.pin_href = 23;
  config.pin_sscb_sda = 26;
  config.pin_sscb_scl = 27;
  config.pin_pwdn = 32;
  config.pin_reset = -1;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG; 

  // init with high specs to reduce image size and latency 
  config.frame_size = FRAMESIZE_SVGA; // 800x600
  config.jpeg_quality = 12;  // 0-63 lower means higher quality
  config.fb_count = 1;

  // Camera init
  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("Camera init failed with error 0x%x", err);
    while (true);
  }
}

void setup() {
  Serial.begin(115200);
  setupCamera();

  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi.");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println(" connected");
}

void loop() {
  // Take a picture
  camera_fb_t * fb = esp_camera_fb_get();
  if (!fb) {
    Serial.println("Camera capture failed");
    delay(2000);
    return;
  }

  WiFiClientSecure client;
  client.setCACert(rootCACertificate);  // Set root CA for TLS verification

  HTTPClient https;
  if (https.begin(client, serverUrl)) {  // HTTPS connection
    https.addHeader("Content-Type", "image/jpeg");

    Serial.println("Sending snapshot to AI server...");
    int httpCode = https.POST(fb->buf, fb->len);

    if (httpCode > 0) {
      Serial.printf("HTTP POST code: %d\n", httpCode);
      if (httpCode == HTTP_CODE_OK) {
        String payload = https.getString();
        Serial.println("Response: " + payload); // AI result from server
      }
    } else {
      Serial.printf("HTTP POST failed, error: %s\n", https.errorToString(httpCode).c_str());
    }
    https.end();
  } else {
    Serial.println("Unable to start connection");
  }

  esp_camera_fb_return(fb);  // Return the frame buffer back to driver
  delay(10000);  // Wait 10 seconds before next snapshot/post
}
