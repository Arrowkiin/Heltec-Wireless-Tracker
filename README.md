#include <WiFi.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <AccelStepper.h>

// ======== OLED CONFIG (HTIT-WB32LAF / Heltec V3) ========
#define SDA_PIN 17
#define SCL_PIN 18
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET 21  // explicit reset pin
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// ======== WIFI CONFIG ========
const char* ssid = "......................";       // <--- your WiFi
const char* password = ".................";    // <--- your password

WiFiServer server1(4001);
WiFiServer server2(4002);
WiFiClient client1;
WiFiClient client2;

// ======== STEPPER CONFIG ========
#define STEPS_PER_REV 4076.0

// --- AZIMUTH ---
#define AZ_IN1 7
#define AZ_IN2 6
#define AZ_IN3 5
#define AZ_IN4 4

// --- ELEVATION ---
#define EL_IN1 3
#define EL_IN2 2
#define EL_IN3 1
#define EL_IN4 38

AccelStepper az(AccelStepper::HALF4WIRE, AZ_IN1, AZ_IN3, AZ_IN2, AZ_IN4);
AccelStepper el(AccelStepper::HALF4WIRE, EL_IN1, EL_IN3, EL_IN2, EL_IN4);

// ======== COMMAND BUFFER ========
#define CMD_BUFFER_LEN 64
char cmdBuffer[CMD_BUFFER_LEN];
uint8_t cmdIndex = 0;

// ======== FUNCTION DECLARATIONS ========
void handleCommand(String cmd);
void moveAzTo(float deg);
void moveElTo(float deg);
void reportPosition();
void updateDisplay(String line1, String line2);
void readClient(WiFiClient &client);

// ======== SETUP ========
void setup() {
  Serial.begin(115200);
  delay(500);
  Serial.println("Booting HTIT-WB32LAF Stepper Controller...");

  // --- OLED INIT ---
  Wire.begin(SDA_PIN, SCL_PIN);
  bool oled_ok = display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  if (!oled_ok) {
    Serial.println("SSD1306 init at 0x3C failed, retrying 0x3D...");
    oled_ok = display.begin(SSD1306_SWITCHCAPVCC, 0x3D);
  }
  if (!oled_ok) {
    Serial.println("SSD1306 allocation failed on both addresses!");
    while (1);
  }

  // --- Increase brightness like Meshtastic ---
  display.ssd1306_command(SSD1306_SETCONTRAST);
  display.ssd1306_command(200);       // maximum contrast
  display.invertDisplay(false);       // try false; set true if it looks better

  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println("Connecting WiFi...");
  display.display();

  // --- WIFI CONNECT ---
  WiFi.begin(ssid, password);
  int tries = 0;
  while (WiFi.status() != WL_CONNECTED && tries < 40) {
    delay(500);
    Serial.print(".");
    tries++;
  }

  display.clearDisplay();
  if (WiFi.status() == WL_CONNECTED) {
    server1.begin();
    server2.begin();
    Serial.println("\nWiFi connected!");
    Serial.print("IP: ");
    Serial.println(WiFi.localIP());
    display.setCursor(0, 0);
    display.println("WiFi connected!");
    display.setCursor(0, 10);
    display.println(WiFi.localIP());
    display.setCursor(0, 20);
    display.println("Ports: 4001/4002");
  } else {
    Serial.println("\nWiFi failed!");
    display.setCursor(0, 0);
    display.println("WiFi FAILED!");
  }
  display.display();

  // --- STEPPERS ---
  az.setMaxSpeed(200);
  az.setAcceleration(100);
  el.setMaxSpeed(200);
  el.setAcceleration(100);
  az.setCurrentPosition(0);
  el.setCurrentPosition(0);
}

// ======== LOOP ========
void loop() {
  az.run();
  el.run();

  // handle client connections
  if (!client1 || !client1.connected()) client1 = server1.available();
  if (!client2 || !client2.connected()) client2 = server2.available();

  if (client1 && client1.available()) readClient(client1);
  if (client2 && client2.available()) readClient(client2);
}

// ======== READ CLIENT ========
void readClient(WiFiClient &client) {
  while (client.available()) {
    char c = client.read();
    if (c == '\n' || c == '\r') {
      if (cmdIndex > 0) {
        cmdBuffer[cmdIndex] = '\0';
        handleCommand(String(cmdBuffer));
        cmdIndex = 0;
      }
    } else {
      if (cmdIndex < CMD_BUFFER_LEN - 1)
        cmdBuffer[cmdIndex++] = c;
    }
  }
}

// ======== COMMAND PARSER ========
void handleCommand(String cmd) {
  cmd.trim();
  cmd.toUpperCase();

  if (cmd == "S") {
    az.stop();
    el.stop();
    updateDisplay("STOP", "");
    return;
  }

  if (cmd == "C") {
    reportPosition();
    return;
  }

  float azDeg = NAN;
  float elDeg = NAN;

  int azIndex = cmd.indexOf("AZ");
  if (azIndex >= 0) {
    int end = cmd.indexOf(' ', azIndex);
    String val = (end > 0) ? cmd.substring(azIndex + 2, end)
                           : cmd.substring(azIndex + 2);
    azDeg = val.toFloat();
  }

  int elIndex = cmd.indexOf("EL");
  if (elIndex >= 0) {
    int end = cmd.indexOf(' ', elIndex);
    String val = (end > 0) ? cmd.substring(elIndex + 2, end)
                           : cmd.substring(elIndex + 2);
    elDeg = val.toFloat();
  }

  if (!isnan(azDeg)) moveAzTo(azDeg);
  if (!isnan(elDeg)) moveElTo(elDeg);
}

// ======== MOVEMENT ========
void moveAzTo(float deg) {
  long targetStep = (long)(deg / 360.0 * STEPS_PER_REV);
  az.moveTo(targetStep);
  Serial.print("Moving AZ to ");
  Serial.println(deg);
  updateDisplay("AZ moving", String(deg) + "°");
}

void moveElTo(float deg) {
  deg = constrain(deg, -90, 90);
  long targetStep = (long)(deg / 360.0 * STEPS_PER_REV);
  el.moveTo(targetStep);
  Serial.print("Moving EL to ");
  Serial.println(deg);
  updateDisplay("EL moving", String(deg) + "°");
}

// ======== REPORT POSITION ========
void reportPosition() {
  float azPosDeg = az.currentPosition() * 360.0 / STEPS_PER_REV;
  float elPosDeg = el.currentPosition() * 360.0 / STEPS_PER_REV;
  String msg = "AZ=" + String(azPosDeg, 1) + " EL=" + String(elPosDeg, 1);
  Serial.println(msg);
  updateDisplay("Position", msg);
}

// ======== OLED HELPER ========
void updateDisplay(String line1, String line2) {
  display.clearDisplay();
  display.setCursor(0, 0);
  display.println("WiFi Rotator");
  display.setCursor(0, 10);
  display.println(line1);
  display.setCursor(0, 20);
  display.println(line2);
  display.setCursor(0, 40);
  display.println(WiFi.localIP());
  display.display();
}
