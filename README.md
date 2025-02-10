# Syfyminer-
#include <ArduinoJson.h>
#include <U8g2lib.h>
#include "mbedtls/sha256.h"
#include "scrypt.h"  // Scrypt mining library

#ifdef ESP8266
  #include <ESP8266WiFi.h>
  #define OLED_SCL 5
  #define OLED_SDA 4
  #define CHIP "ESP8266"
#elif defined(ESP32)
  #include <WiFi.h>
  #define OLED_SCL 22
  #define OLED_SDA 21
  #define CHIP "ESP32"
#endif

#define HASH_RATE_TARGET 280000  // Target kH/s for optimization
#define POOL_URL "stratum+tcp://your.pool.address:3333"
#define POOL_USER "your_wallet_address"
#define POOL_PASS "x"

// OLED Display Setup
U8G2_SSD1306_128X64_NONAME_F_SW_I2C oled(U8G2_R0, OLED_SCL, OLED_SDA, U8X8_PIN_NONE);

// Wi-Fi AP Mode Credentials
const char* ssid = "SyfyMiner";
const char* password = "mine1234";

// Mining Variables
uint8_t blockHeader[80];
uint8_t hashOutput[32];

bool useSHA256 = true;  // Default to SHA-256 (Bitcoin)
bool useScrypt = false; // Enable for Litecoin/Dogecoin

void setup() {
    Serial.begin(115200);
    WiFi.softAP(ssid, password);

    if (oled.begin()) {
        oled.clearBuffer();
        oled.setFont(u8g2_font_ncenB08_tr);
        oled.drawStr(10, 20, "SyfyMiner v1.0");
        oled.drawStr(10, 40, CHIP);
        oled.sendBuffer();
    }

    delay(2000);
}

void loop() {
    if (useSHA256) {
        mineSHA256();
    } else if (useScrypt) {
        mineScrypt();
    }
}

void mineSHA256() {
    uint32_t nonce = 0;
    while (true) {
        memcpy(blockHeader + 76, &nonce, 4);
        mbedtls_sha256_ret(blockHeader, 80, hashOutput, 0);

        if (hashOutput[0] == 0 && hashOutput[1] == 0) {
            Serial.println("SHA-256 Block Found!");
            sendShareToPool(nonce);
            break;
        }

        nonce++;
        if (nonce % 100000 == 0) {
            updateDisplay(nonce);
        }
    }
}

void mineScrypt() {
    uint32_t nonce = 0;
    while (true) {
        scrypt_1024_1_1_256(blockHeader, hashOutput);

        if (hashOutput[0] == 0 && hashOutput[1] == 0) {
            Serial.println("Scrypt Block Found!");
            sendShareToPool(nonce);
            break;
        }

        nonce++;
        if (nonce % 100000 == 0) {
            updateDisplay(nonce);
        }
    }
}

void sendShareToPool(uint32_t nonce) {
    WiFiClient client;
    if (client.connect(POOL_URL, 3333)) {
        StaticJsonDocument<200> jsonDoc;
        jsonDoc["method"] = "mining.submit";
        jsonDoc["params"][0] = POOL_USER;
        jsonDoc["params"][1] = "job_id"; // Replace with actual job ID
        jsonDoc["params"][2] = nonce;
        jsonDoc["id"] = 1;

        String jsonStr;
        serializeJson(jsonDoc, jsonStr);
        client.println(jsonStr);
        client.stop();
    }
}

void updateDisplay(uint32_t nonce) {
    oled.clearBuffer();
    oled.setFont(u8g2_font_ncenB08_tr);
    oled.setCursor(10, 20);
    oled.print("Nonce: ");
    oled.print(nonce);
    oled.sendBuffer();
}
