#include <Arduino.h>
#include <Wire.h>
#include <FS.h>
#include <SD.h>       // Arduinos inbyggda SD-bibliotek
#include <SPI.h>
#include <ESP_Panel_Library.h>
#include "esp_mac.h"
#include "driver/twai.h" 

// Officiella hårdvarupinnar från Waveshare Wiki (v4)
#define I2C0_SDA 8
#define I2C0_SCL 9
#define I2C1_SDA 15
#define I2C1_SCL 7

#define SW6106_ADDRESS 0x3C
#define TCA9554_ADDRESS 0x20

#define CAN_TX_PIN GPIO_NUM_6
#define CAN_RX_PIN GPIO_NUM_0

#define RS485_RX_PIN 43
#define RS485_TX_PIN 44

// SD-kortets hårdvarupinnar (ESP32-S3 standard SPI-buss på detta kort)
#define SD_SPI_MOSI 1
#define SD_SPI_MISO 4
#define SD_SPI_SCK  2
// Sätt en virtuell pinne för CS-kontroll i SD-biblioteket, den faktiska styrs via expandern
#define DUMMY_SD_CS_PIN 5 

ESP_Panel *panel = NULL;
unsigned long lastDiagnosticsCheck = 0;
bool canInitialized = false;
String screenLog = "";

void logOutput(String text) {
  Serial.println(text);
  screenLog += text + "\n";
}

// --- NY FUNKTION: REGLERA IO-EXPANDERN FÖR SD-KORTETS CHIP SELECT (EXIO3) ---
void setExpanderPin(uint8_t pin, bool state) {
  // Läs nuvarande tillstånd
  Wire.beginTransmission(TCA9554_ADDRESS);
  Wire.write(0x01); // Register för utgångsvärden
  Wire.endTransmission();
  Wire.requestFrom(TCA9554_ADDRESS, (uint8_t)1);
  uint8_t current_outputs = Wire.available() ? Wire.read() : 0x00;

  // Ändra specifik bit (EXIO3 = Bit 3)
  if (state) current_outputs |= (1 << pin);
  else current_outputs &= ~(1 << pin);

  // Skriv tillbaka
  Wire.beginTransmission(TCA9554_ADDRESS);
  Wire.write(0x01);
  Wire.write(current_outputs);
  Wire.endTransmission();
}

// --- NY FUNKTION: INITIERA SD-KORT OCH SPARA DIAGNOSTIKFIL ---
void saveLogToSDCard() {
  logOutput("\n[SD-KORT] STARTAR SKRIVNING:");
  
  // 1. Konfigurera TCA9554 så att EXIO3 (SD_CS) är en utgång (sätt bit 3 till 0 i konfig-reg 0x03)
  Wire.beginTransmission(TCA9554_ADDRESS);
  Wire.write(0x03); 
  Wire.endTransmission();
  Wire.requestFrom(TCA9554_ADDRESS, (uint8_t)1);
  uint8_t config = Wire.available() ? Wire.read() : 0xFF;
  config &= ~(1 << 3); // Sätt bit 3 till 0 (Utgång)
  
  Wire.beginTransmission(TCA9554_ADDRESS);
  Wire.write(0x03);
  Wire.write(config);
  Wire.endTransmission();

  // 2. Aktivera hårdvaru-SPI-bussen på kortets dedikerade pinnar [0x1.1.1]
  SPI.begin(SD_SPI_SCK, SD_SPI_MISO, SD_SPI_MOSI, DUMMY_SD_CS_PIN);

  // 3. Dra EXIO3 (SD_CS) LÅG via expandern för att välja SD-kortet
  setExpanderPin(3, false); 
  delay(10);

  // 4. Starta SD-biblioteket
  if (!SD.begin(DUMMY_SD_CS_PIN, SPI, 4000000)) { // 4MHz hastighet
    logOutput("  [FEL] Kunde inte montera SD-kortet! Saknas det i läsaren?");
    setExpanderPin(3, true); // Släpp pinnen om det misslyckas
    return;
  }

  uint8_t cardType = SD.cardType();
  if (cardType == CARD_NONE) {
    logOutput("  [FEL] Inget giltigt SD-kort hittades.");
    setExpanderPin(3, true);
    return;
  }

  logOutput("  -> SD-kort monterat OK!");

  // 5. Skapa och öppna filen "ESP32-S3-Touch-LCD-4.txt" på kortet
  File file = SD.open("/ESP32-S3-Touch-LCD-4.txt", FILE_WRITE);
  if (!file) {
    logOutput("  [FEL] Kunde inte skapa eller öppna loggfilen på kortet.");
    setExpanderPin(3, true);
    return;
  }

  // 6. Skriv hela vår skärm- och diagnostiklogg till filen
  if (file.print(screenLog)) {
    logOutput("  -> Loggfilen sparades framgångsrikt!");
    logOutput("  -> Filnamn: /ESP32-S3-Touch-LCD-4.txt");
  } else {
    logOutput("  [FEL] Det gick inte att skriva data till filen.");
  }
  
  file.close();
  
  // Släpp SD-kortets väljarpinne (EXIO3 sätts hög) så att SPI-bussen lämnas fri
  setExpanderPin(3, true); 
}

// --- EXISTERANDE DIAGNOSTIKFUNKTIONER ---
void identifyNetworkInterfaces() {
  uint8_t mac;
  logOutput("[1/5] TRÅDLÖSA INTERFACES:");
  if (esp_read_mac(mac, ESP_MAC_BASE) == ESP_OK) {
    char buf[50]; sprintf(buf, "  Base MAC: %02X:%02X:%02X:%02X:%02X:%02X", mac, mac, mac, mac, mac, mac);
    logOutput(String(buf));
  }
  if (esp_read_mac(mac, ESP_MAC_BT) == ESP_OK) {
    char buf[50]; sprintf(buf, "  BLE MAC:  %02X:%02X:%02X:%02X:%02X:%02X", mac, mac, mac, mac, mac, mac);
    logOutput(String(buf));
  }
}

void scanI2CBus(TwoWire &bus, int sda, int scl, const char* busName, int busNum) {
  logOutput("\n[2/5] SKANNAR I2C BUS " + String(busNum) + ":");
  bus.begin(sda, scl, 100000);
  int devicesFound = 0;
  for (byte address = 1; address < 127; address++) {
    bus.beginTransmission(address);
    if (bus.endTransmission() == 0) {
      String deviceName = "Okänt chip";
      if (address == 0x20) deviceName = "TCA9554 Expander";
      else if (address == 0x3C) deviceName = "SW6106 Laddare";
      else if (address == 0x51) deviceName = "PCF85063 RTC";
      else if (address == 0x5D || address == 0x14) deviceName = "GT911 Touch";
      char buf[50]; sprintf(buf, "  Hittad 0x%02X -> ", address);
      logOutput(String(buf) + deviceName);
      devicesFound++;
    }
  }
}

void readBatteryStatus() {
  keepSW6106Alive();
  uint8_t soc = readSW6106Register(0x31);
  uint8_t status = readSW6106Register(0x32);
  logOutput("\n[3/5] STRÖM- OCH BATTERISTATUS:");
  if (soc == 0xFF || soc == 0 || soc > 100) {
    logOutput("  Inget batteri detekterat (USB-C)");
  } else {
    bool isCharging = (status & 0x40) || (status & 0x80); 
    logOutput("  Batteri anslutet: " + String(soc) + "%");
    logOutput(isCharging ? "  Status: Laddar..." : "  Status: Urladdas (Batteridrift)");
  }
}

void initCANBus() {
  logOutput("\n[4/5] CAN-BUSS (TWAI) STATUS:");
  twai_general_config_t g_config = TWAI_GENERAL_CONFIG_DEFAULT(CAN_TX_PIN, CAN_RX_PIN, TWAI_MODE_LISTEN_ONLY);
  twai_timing_config_t t_config = TWAI_TIMING_CONFIG_500KBITS(); 
  twai_filter_config_t f_config = TWAI_FILTER_CONFIG_ACCEPT_ALL();
  if (twai_driver_install(&g_config, &t_config, &f_config) == ESP_OK) {
    if (twai_start() == ESP_OK) {
      logOutput("  Aktiv (Listen-Only, 500kbps)");
      canInitialized = true;
    }
  }
}

void initRS485() {
  logOutput("\n[5/5] RS485-BUSS STATUS:");
  Serial2.begin(115200, SERIAL_8N1, RS485_RX_PIN, RS485_TX_PIN);
  logOutput("  Aktiv (115200 baud, 8N1)");
}

void checkCANTraffic() {
  if (!canInitialized) return;
  twai_message_t message;
  if (twai_receive(&message, 0) == ESP_OK) {
    Serial.printf("  [CAN In] ID: 0x%03X | Data: ", message.identifier);
    for (int i = 0; i < message.data_length; i++) Serial.printf("%02X ", message.data[i]);
    Serial.println();
  }
}

void checkRS485Traffic() {
  if (Serial2.available()) {
    Serial.print("  [RS485 In] ");
    while (Serial2.available()) {
      char c = Serial2.read();
      if (isprint(c)) Serial.print(c);
      else Serial.printf("[0x%02X]", c);
    }
    Serial.println();
  }
}

void beepBuzzer(bool state) {
  Wire.beginTransmission(TCA9554_ADDRESS);
  Wire.write(0x03); 
  Wire.write(0x00); 
  Wire.endTransmission();
  Wire.beginTransmission(TCA9554_ADDRESS);
  Wire.write(0x01); 
  Wire.write(state ? 0x20 : 0x00); 
  Wire.endTransmission();
}

void drawDiagnosticScreen() {
  auto lcd = panel->getLcd();
  if (!lcd) return;
  lcd->clear();
  lcd->setTextColor(Color16b(255, 255, 255)); 
  int yOffset = 20;
  int startIdx = 0;
  while (startIdx < screenLog.length()) {
    int endIdx = screenLog.indexOf('\n', startIdx);
    if (endIdx == -1) endIdx = screenLog.length();
    String line = screenLog.substring(startIdx, endIdx);
    lcd->drawString(20, yOffset, line.c_str());
    yOffset += 18;
    startIdx = endIdx + 1;
    if (yOffset > 460) break;
  }
}

// --- SETUP ---
void setup() {
  Serial.begin(115200);
  delay(2500); 
  
  logOutput("=======================================================");
  logOutput("   WAVESHARE ESP32-S3-TOUCH-LCD-4 v4 TOTALDIAGNOSTIK   ");
  logOutput("=======================================================");

  // Kör all hårdvaruinventering först till logg-bufferten
  identifyNetworkInterfaces();
  scanI2CBus(Wire, I2C0_SDA, I2C0_SCL, "Intern I2C-Buss (Kanal 0)", 0);
  scanI2CBus(Wire1, I2C1_SDA, I2C1_SCL, "Touch & RTC Buss (Kanal 1)", 1);
  readBatteryStatus();
  initCANBus();
  initRS485();

  // EFTER att I2C0 har startats kan vi köra SD-kortskrivningen
  saveLogToSDCard();

  // Starta skärmen och rita upp allt
  panel = new ESP_Panel();
  panel->init();
  panel->begin();
  if (panel->getBacklight()) panel->getBacklight()->on();
  drawDiagnosticScreen();

  if (panel->getTouch()) {
    beepBuzzer(true); delay(150); beepBuzzer(false);
  } else {
    while(1) { delay(1000); }
  }
}

// --- LOOP ---
void loop() {
  checkCANTraffic();
  checkRS485Traffic();

  if (millis() - lastDiagnosticsCheck > 5000) {
    keepSW6106Alive(); 
    Serial2.println("ESP32_Diagnostic_Pulse"); 
    lastDiagnosticsCheck = millis();
  }

  if (panel->getTouch()) {
    panel->getTouch()->readData();
    int touch_points = panel->getTouch()->getTouchPointsNum();
    if (touch_points > 0) {
      Serial.printf("[Touch] Antal fingrar: %d\n", touch_points);
      for (int i = 0; i < touch_points; i++) {
        Serial.printf("  -> X: %d | Y: %d\n", panel->getTouch()->getPointX(i), panel->getTouch()->getPointY(i));
      }
    }
  }
  delay(10); 
}
