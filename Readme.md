# ResQLink - Ambulance Management System (IoT + Web)

🚑 This project is developed using IoT with Web.

▶️ Watch the demo video here: [Project Demo Video]([https://www.youtube.com/watch?v=YOUR_VIDEO_ID](https://www.youtube.com/shorts/WhvAWLVJU74))

CODE OF CIRCUIT

#include <esp_now.h>
#include <WiFi.h>
#include <HTTPClient.h>

// ---------- WiFi Credentials ----------
#define WIFI_SSID     "realme C25Y"
#define WIFI_PASSWORD "Aamna786"

// ---------- Firebase Legacy Setup ----------
const char* DATABASE_URL    = "https://traffic-iot-5985f-default-rtdb.firebaseio.com/";
const char* DATABASE_SECRET = "RZdULBEvluzvdLVdiZ8l5kX3lOkqn7JtLmVJyyFk";
const char* NODE            = "trafficLogs";   // Firebase node to push data

// ---------- Slave MAC ----------
uint8_t slaveAddress[] = {0x5C, 0x01, 0x3B, 0x33, 0x92, 0x34};

// ---------- Traffic Light Pins ----------
const int RED_PIN    = 26;
const int YELLOW_PIN = 27;
const int GREEN_PIN  = 25;

unsigned long lastSend = 0;
unsigned long lastChange = 0;
int currentLight = 0; // 0=RED, 1=YELLOW, 2=GREEN

// ---------- Force Green Variables ----------
bool forceGreen = false;
unsigned long forceGreenUntil = 0;
unsigned long forceGreenStart = 0;
unsigned long forceGreenEnd   = 0;
bool logPending = false; // flag to log to Firebase after force green ends

// ---------- Utility ----------
void setTrafficLight(int state) {
  digitalWrite(RED_PIN, state == 0 ? HIGH : LOW);
  digitalWrite(YELLOW_PIN, state == 1 ? HIGH : LOW);
  digitalWrite(GREEN_PIN, state == 2 ? HIGH : LOW);
}

// ---------- ESP-NOW Receive ----------
// ---------- ESP-NOW Receive ----------
void onDataRecv(const esp_now_recv_info_t *info, const uint8_t *incomingData, int len) {
  int rssi = info->rx_ctrl->rssi;

  if (rssi > -60) { // ambulance close
    if (!forceGreen) { // only trigger once
      Serial.println("🚑 ALERT: Forcing GREEN (ambulance detected)");
      forceGreen = true;
      forceGreenStart = millis();
      logPending = false; // reset log state
    }
  } else {
    // ambulance moved away
    if (forceGreen) {
      forceGreen = false;
      forceGreenEnd = millis();
      logPending = true; // mark log to be sent
      Serial.println("✅ Ambulance moved away, ending force GREEN");
    }
  }
}


// ---------- Firebase Log ----------
void sendLogToFirebase(unsigned long startMs, unsigned long endMs) {
  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("⚠️ WiFi not connected, skipping log.");
    return;
  }

  HTTPClient http;

  String url = String(DATABASE_URL) + NODE + ".json?auth=" + DATABASE_SECRET;
  http.begin(url);

  // Get ESP32 MAC
  String mac = WiFi.macAddress();

  // Build JSON
  unsigned long totalTime = endMs - startMs;
  String json = "{";
  json += "\"mac\":\"" + mac + "\",";
  json += "\"forceGreenStart\":" + String(startMs) + ",";
  json += "\"forceGreenEnd\":" + String(endMs) + ",";
  json += "\"totalTimeTakenMs\":" + String(totalTime) + ",";
  json += "\"sentTimestamp\":" + String(millis());
  json += "}";

  http.addHeader("Content-Type", "application/json");
  int httpResponseCode = http.POST(json);

  if (httpResponseCode > 0) {
    Serial.println("✅ Log sent: " + http.getString());
  } else {
    Serial.print("❌ Error sending log: ");
    Serial.println(httpResponseCode);
  }
  http.end();
}

// ---------- Setup ----------
void setup() {
  Serial.begin(115200);

  WiFi.mode(WIFI_STA);

  // Connect to WiFi for Firebase
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\n✅ WiFi connected!");

  if (esp_now_init() != ESP_OK) {
    Serial.println("❌ ESP-NOW init failed!");
    return;
  }

  esp_now_register_recv_cb(onDataRecv);

  // Add Slave as peer
  esp_now_peer_info_t peerInfo = {};
  memcpy(peerInfo.peer_addr, slaveAddress, 6);
  peerInfo.channel = 0;
  peerInfo.encrypt = false;
  if (esp_now_add_peer(&peerInfo) != ESP_OK) {
    Serial.println("❌ Failed to add peer");
    return;
  }

  // Traffic light setup
  pinMode(RED_PIN, OUTPUT);
  pinMode(YELLOW_PIN, OUTPUT);
  pinMode(GREEN_PIN, OUTPUT);

  setTrafficLight(0); // Start with RED
  Serial.println("✅ Master Ready");
}

// ---------- Main Loop ----------
void loop() {
  // Force Green handling
  if (forceGreen) {
    setTrafficLight(2);   // keep GREEN
    currentLight = 2;
  } else if (millis() - lastChange >= 7000) {
    currentLight = (currentLight + 1) % 3;
    setTrafficLight(currentLight);
    lastChange = millis();
  }

  // Send log to Firebase once force green ends
  if (logPending) {
    sendLogToFirebase(forceGreenStart, forceGreenEnd);
    logPending = false;
  }

  // Keep sending heartbeat to slave every 2s
  if (millis() - lastSend > 2000) {
    const char *msg = "Hello from Master";
    esp_now_send(slaveAddress, (uint8_t *)msg, strlen(msg));
    Serial.println("Sent: Hello from Master");
    lastSend = millis();
  }
}











#include <esp_now.h>
#include <WiFi.h>
#include <HTTPClient.h>

// ---------- WiFi Credentials ----------
#define WIFI_SSID     "realme C25Y"
#define WIFI_PASSWORD "Aamna786"

// ---------- Firebase Legacy Setup ----------
const char* DATABASE_URL    = "https://traffic-iot-5985f-default-rtdb.firebaseio.com/";
const char* DATABASE_SECRET = "RZdULBEvluzvdLVdiZ8l5kX3lOkqn7JtLmVJyyFk";
const char* NODE            = "trafficLogs";   // Firebase node to push data

// ---------- Slave MAC ----------
uint8_t slaveAddress[] = {0x5C, 0x01, 0x3B, 0x33, 0x92, 0x34};

// ---------- Traffic Light Pins ----------
const int RED_PIN    = 26;
const int YELLOW_PIN = 27;
const int GREEN_PIN  = 25;

unsigned long lastSend = 0;
unsigned long lastChange = 0;
int currentLight = 0; // 0=RED, 1=YELLOW, 2=GREEN

// ---------- Force Green Variables ----------
bool forceGreen = false;
unsigned long forceGreenUntil = 0;
unsigned long forceGreenStart = 0;
unsigned long forceGreenEnd   = 0;
bool logPending = false; // flag to log to Firebase after force green ends

// ---------- Utility ----------
void setTrafficLight(int state) {
  digitalWrite(RED_PIN, state == 0 ? HIGH : LOW);
  digitalWrite(YELLOW_PIN, state == 1 ? HIGH : LOW);
  digitalWrite(GREEN_PIN, state == 2 ? HIGH : LOW);
}

// ---------- ESP-NOW Receive ----------
// ---------- ESP-NOW Receive ----------
void onDataRecv(const esp_now_recv_info_t *info, const uint8_t *incomingData, int len) {
  int rssi = info->rx_ctrl->rssi;
  float distance = rssiToDistance(rssi);

  Serial.print("📡 RSSI: ");
  Serial.print(rssi);
  Serial.print(" dBm | Approx Distance: ");
  Serial.print(distance, 2);
  Serial.println(" meters");

  // if (rssi > -30) { // ambulance close
  if (distance <= 0.12) {
    if (!forceGreen) { // only trigger once
      Serial.println("🚑 ALERT: Forcing GREEN (ambulance detected)");
      forceGreen = true;
      forceGreenStart = millis();
      logPending = false; // reset log state
    }
  } else {
    // ambulance moved away
    if (forceGreen) {
      forceGreen = false;
      forceGreenEnd = millis();
      logPending = true; // mark log to be sent
      Serial.println("✅ Ambulance moved away, ending force GREEN");
    }
  }
}


// ---------- RSSI to Distance Conversion ----------
float rssiToDistance(int rssi) {
  int txPower = -59;  // RSSI value at 1 meter (calibrate this for accuracy)
  float n = 2.0;      // Path-loss exponent (2=open space, higher indoors)
  return pow(10.0, ((float)txPower - (float)rssi) / (10.0 * n));
}



// ---------- Firebase Log ----------
void sendLogToFirebase(unsigned long startMs, unsigned long endMs) {
  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("⚠️ WiFi not connected, skipping log.");
    return;
  }

  HTTPClient http;

  String url = String(DATABASE_URL) + NODE + ".json?auth=" + DATABASE_SECRET;
  http.begin(url);

  // Get ESP32 MAC
  String mac = WiFi.macAddress();

  // Build JSON
  unsigned long totalTime = endMs - startMs;
  String json = "{";
  json += "\"mac\":\"" + mac + "\",";
  json += "\"forceGreenStart\":" + String(startMs) + ",";
  json += "\"forceGreenEnd\":" + String(endMs) + ",";
  json += "\"totalTimeTakenMs\":" + String(totalTime) + ",";
  json += "\"sentTimestamp\":" + String(millis());
  json += "}";

  http.addHeader("Content-Type", "application/json");
  int httpResponseCode = http.POST(json);

  if (httpResponseCode > 0) {
    Serial.println("✅ Log sent: " + http.getString());
  } else {
    Serial.print("❌ Error sending log: ");
    Serial.println(httpResponseCode);
  }
  http.end();
}

// ---------- Setup ----------
void setup() {
  Serial.begin(115200);

  WiFi.mode(WIFI_STA);

  // Connect to WiFi for Firebase
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\n✅ WiFi connected!");

  if (esp_now_init() != ESP_OK) {
    Serial.println("❌ ESP-NOW init failed!");
    return;
  }

  esp_now_register_recv_cb(onDataRecv);

  // Add Slave as peer
  esp_now_peer_info_t peerInfo = {};
  memcpy(peerInfo.peer_addr, slaveAddress, 6);
  peerInfo.channel = 0;
  peerInfo.encrypt = false;
  if (esp_now_add_peer(&peerInfo) != ESP_OK) {
    Serial.println("❌ Failed to add peer");
    return;
  }

  // Traffic light setup
  pinMode(RED_PIN, OUTPUT);
  pinMode(YELLOW_PIN, OUTPUT);
  pinMode(GREEN_PIN, OUTPUT);

  setTrafficLight(0); // Start with RED
  Serial.println("✅ Master Ready");
}

// ---------- Main Loop ----------
void loop() {
  // Force Green handling
  if (forceGreen) {
    setTrafficLight(2);   // keep GREEN
    currentLight = 2;
  } else if (millis() - lastChange >= 7000) {
    currentLight = (currentLight + 1) % 3;
    setTrafficLight(currentLight);
    lastChange = millis();
  }

  // Send log to Firebase once force green ends
  if (logPending) {
    sendLogToFirebase(forceGreenStart, forceGreenEnd);
    logPending = false;
  }

  // Keep sending heartbeat to slave every 2s
  if (millis() - lastSend > 2000) {
    const char *msg = "Hello from Master";
    esp_now_send(slaveAddress, (uint8_t *)msg, strlen(msg));
    Serial.println("Sent: Hello from Master");
    lastSend = millis();
  }
}


