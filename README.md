# iot-camera

/*
 * =============================================================================
 *  IoT Camera System — Camera Node (ESP32 DevKit V1 + OV7670)
 *  Phien ban: 2.0 (MQTT Protocol + HTTP Upload)
 *  Giao thuc: WiFi + MQTT (PubSubClient) + HTTP POST
 * =============================================================================
 *
 *  Chuc nang:
 *    1. Ket noi WiFi voi auto-reconnect
 *    2. Ket noi MQTT Broker voi xac thuc (username/password)
 *    3. Gui heartbeat dinh ky (30s) bao trang thai hoat dong
 *    4. Lang nghe lenh dieu khien tu Server (CAPTURE, STREAM, RESTART, CONFIG)
 *    5. Gui ACK phan hoi sau khi xu ly lenh
 *    6. Last Will & Testament (LWT) - tu dong bao offline khi mat ket noi
 *    7. LED bao trang thai (nhap nhay = dang ket noi, sang = OK, tat = loi)
 *
 *  Thu vien can cai (Arduino IDE > Library Manager):
 *    - PubSubClient by Nick O'Leary (>= 2.8.0)
 *    - ArduinoJson by Benoit Blanchon (>= 7.0.0)
 *
 * =============================================================================
 */

#include <ArduinoJson.h>
#include <HTTPClient.h>
#include <PubSubClient.h>
#include <WiFi.h>


// ========================= CAU HINH MANG =========================

// Thong tin WiFi (THAY DOI THEO MANG CUA BAN)
const char *WIFI_SSID = "VanMinh";
const char *WIFI_PASSWORD = "11111111";

// Thong tin MQTT Broker (IP may tinh chay Mosquitto)
const char *MQTT_SERVER = "192.168.1.101";
const int MQTT_PORT = 1883;
const char *MQTT_USER = "cam_node_01";
const char *MQTT_PASS = "CamNode@2026";

// Thong tin Server HTTP (IP may tinh chay Flask)
const char *SERVER_URL = "http://192.168.1.101:5000";

// ========================= DINH DANH THIET BI =========================

const char *DEVICE_ID = "CAM_NODE_01";
const char *DEVICE_TYPE = "camera";
const char *CLIENT_ID = "esp32_cam_node_01";
const char *FW_VERSION = "2.0.0";

// ========================= MQTT TOPICS =========================

// Topic LANG NGHE (Subscribe)
const char *TOPIC_CMD = "iot/camera/cmd";

// Topic GUI DI (Publish)
const char *TOPIC_ACK = "iot/camera/ack";
const char *TOPIC_STATUS = "iot/camera/status";
const char *TOPIC_HEARTBEAT = "iot/system/heartbeat";
const char *TOPIC_LOG = "iot/system/log";
const char *TOPIC_NEW_IMAGE = "iot/notify/new_image";

// ========================= CAU HINH THOI GIAN =========================

const unsigned long HEARTBEAT_INTERVAL = 1000; // Gui heartbeat moi 30 giay
const unsigned long WIFI_RECONNECT_DELAY =
    5000; // Thu ket noi lai WiFi sau 5 giay
const unsigned long MQTT_RECONNECT_DELAY =
    5000;                                     // Thu ket noi lai MQTT sau 5 giay
const unsigned long LED_BLINK_INTERVAL = 500; // Nhap nhay LED moi 500ms

// ========================= CHAN GPIO =========================

const int LED_STATUS =
    2; // LED trang thai tren ESP32 DevKit (GPIO2 = LED_BUILTIN)

// ========================= BIEN TOAN CUC =========================

WiFiClient espClient;
PubSubClient mqttClient(espClient);

// Theo doi thoi gian (dung millis(), KHONG dung delay())
unsigned long lastHeartbeat = 0;
unsigned long lastWifiRetry = 0;
unsigned long lastMqttRetry = 0;
unsigned long lastLedToggle = 0;
unsigned long deviceStartTime = 0;

// Trang thai thiet bi
bool isStreaming = false;
unsigned long streamInterval = 10000; // Mac dinh stream moi 10 giay
unsigned long lastStreamCapture = 0;
int totalCaptures = 0;

// Trang thai LED
bool ledState = false;

// ========================= SETUP =========================

void setup() {
  Serial.begin(115200);
  delay(100);

  Serial.println();
  Serial.println(
      "============================================================");
  Serial.println("  IoT Camera System — Camera Node v2.0");
  Serial.println("  Giao thuc: WiFi + MQTT + HTTP");
  Serial.println(
      "============================================================");

  // Khoi tao LED trang thai
  pinMode(LED_STATUS, OUTPUT);
  digitalWrite(LED_STATUS, LOW);

  // Ghi nhan thoi diem khoi dong
  deviceStartTime = millis();

  // Buoc 1: Ket noi WiFi
  setupWiFi();

  // Buoc 2: Cau hinh MQTT Client
  setupMQTT();

  // Buoc 3: Ket noi MQTT Broker
  connectMQTT();

  // Buoc 4: Khoi tao Camera OV7670
  // initCamera();  // TODO: Phase 4C - Cau hinh OV7670 registers

  Serial.println("\n[SETUP] Hoan tat! Thiet bi san sang.\n");
}

// ========================= LOOP CHINH =========================

void loop() {
  // --- Kiem tra ket noi WiFi ---
  if (WiFi.status() != WL_CONNECTED) {
    blinkLED(); // Nhap nhay LED bao mat WiFi
    reconnectWiFi();
    return; // Khong lam gi them khi chua co WiFi
  }

  // --- Kiem tra ket noi MQTT ---
  if (!mqttClient.connected()) {
    blinkLED(); // Nhap nhay LED bao mat MQTT
    reconnectMQTT();
    return;
  }

  // --- WiFi + MQTT OK → LED sang lien tuc ---
  digitalWrite(LED_STATUS, HIGH);

  // --- Xu ly MQTT messages (BAT BUOC goi trong loop) ---
  mqttClient.loop();

  // --- Gui Heartbeat dinh ky ---
  if (millis() - lastHeartbeat >= HEARTBEAT_INTERVAL) {
    sendHeartbeat();
    lastHeartbeat = millis();
  }

  // --- Che do Stream: chup anh lien tuc ---
  if (isStreaming && (millis() - lastStreamCapture >= streamInterval)) {
    captureAndUpload();
    lastStreamCapture = millis();
  }
}

// ========================= WIFI =========================

void setupWiFi() {
  Serial.printf("[WiFi] Dang ket noi den '%s'...\n", WIFI_SSID);

  WiFi.mode(WIFI_STA);         // Station mode (client)
  WiFi.setAutoReconnect(true); // Tu dong reconnect khi mat song
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  // Cho ket noi (toi da 20 giay)
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 40) {
    delay(500);
    Serial.print(".");
    attempts++;
    // Nhap nhay LED trong luc cho
    digitalWrite(LED_STATUS, !digitalRead(LED_STATUS));
  }

  if (WiFi.status() == WL_CONNECTED) {
    Serial.println();
    Serial.printf("[WiFi] Ket noi thanh cong!\n");
    Serial.printf("[WiFi] IP Address: %s\n", WiFi.localIP().toString().c_str());
    Serial.printf("[WiFi] RSSI: %d dBm\n", WiFi.RSSI());
    Serial.printf("[WiFi] MAC: %s\n", WiFi.macAddress().c_str());
    digitalWrite(LED_STATUS, HIGH);
  } else {
    Serial.println();
    Serial.println("[WiFi] THAT BAI! Khong ket noi duoc WiFi.");
    Serial.println("[WiFi] Se thu lai trong loop()...");
    digitalWrite(LED_STATUS, LOW);
  }
}

void reconnectWiFi() {
  if (millis() - lastWifiRetry < WIFI_RECONNECT_DELAY)
    return;
  lastWifiRetry = millis();

  Serial.println("[WiFi] Mat ket noi! Dang thu lai...");
  WiFi.disconnect();
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
}

// ========================= MQTT =========================

void setupMQTT() {
  mqttClient.setServer(MQTT_SERVER, MQTT_PORT);
  mqttClient.setCallback(onMqttMessage);

  // Tang buffer size cho message lon (mac dinh chi 256 bytes)
  mqttClient.setBufferSize(1024);

  Serial.printf("[MQTT] Server: %s:%d\n", MQTT_SERVER, MQTT_PORT);
  Serial.printf("[MQTT] Client ID: %s\n", CLIENT_ID);
  Serial.printf("[MQTT] Username: %s\n", MQTT_USER);
}

void connectMQTT() {
  Serial.println("[MQTT] Dang ket noi voi Broker...");

  // --------- LAST WILL & TESTAMENT (LWT) ---------
  // Khi ESP32 mat ket noi DOT NGOT (mat dien, mat WiFi, hang),
  // Broker se TU DONG publish message nay len topic heartbeat
  // de Server biet thiet bi da offline.
  //
  // Day la tinh nang QUAN TRONG cua MQTT, giup phat hien
  // thiet bi offline NGAY LAP TUC thay vi phai cho timeout.

  // Tao JSON payload cho LWT
  String lwt_payload = "{";
  lwt_payload += "\"device_id\":\"" + String(DEVICE_ID) + "\",";
  lwt_payload += "\"device_type\":\"" + String(DEVICE_TYPE) + "\",";
  lwt_payload += "\"status\":\"offline\",";
  lwt_payload += "\"event\":\"LWT_TRIGGERED\",";
  lwt_payload += "\"message\":\"Thiet bi mat ket noi dot ngot\"";
  lwt_payload += "}";

  // Ket noi voi LWT: neu mat ket noi → Broker gui lwt_payload len
  // TOPIC_HEARTBEAT
  bool connected = mqttClient.connect(CLIENT_ID,       // Client ID
                                      MQTT_USER,       // Username
                                      MQTT_PASS,       // Password
                                      TOPIC_HEARTBEAT, // LWT Topic
                                      1,     // LWT QoS (1 = at least once)
                                      false, // LWT Retain
                                      lwt_payload.c_str() // LWT Message
  );

  if (connected) {
    Serial.println("[MQTT] Ket noi thanh cong!");

    // Subscribe vao topic nhan lenh tu Server
    mqttClient.subscribe(TOPIC_CMD, 1); // QoS 1
    Serial.printf("[MQTT] Da subscribe: %s (QoS 1)\n", TOPIC_CMD);

    // Gui log thong bao da ket noi
    publishLog("INFO", "DEVICE_CONNECTED",
               "Camera Node da ket noi MQTT Broker thanh cong. FW: " +
                   String(FW_VERSION));

    // Gui heartbeat ngay lap tuc khi vua ket noi
    sendHeartbeat();

  } else {
    Serial.printf("[MQTT] Ket noi THAT BAI! Ma loi: %d\n", mqttClient.state());
    Serial.println(
        "[MQTT] Ma loi: -1=TIMEOUT -2=CONNECT_REFUSED -4=CONNECTION_LOST");
    Serial.println("[MQTT] Kiem tra: IP Broker, Port, Username/Password");
  }
}

void reconnectMQTT() {
  if (millis() - lastMqttRetry < MQTT_RECONNECT_DELAY)
    return;
  lastMqttRetry = millis();

  Serial.println("[MQTT] Mat ket noi! Dang thu reconnect...");
  connectMQTT();
}

// ========================= XU LY LENH MQTT =========================

void onMqttMessage(char *topic, byte *payload, unsigned int length) {
  /*
   * Callback duoc goi khi nhan message tu Broker.
   * Topic duy nhat subscribe la "iot/camera/cmd"
   *
   * Format lenh tu Server:
   * {
   *   "cmd_id": "cmd_20260228_183000_a1b2",
   *   "command": "CAPTURE",
   *   "timestamp": "2026-02-28T18:30:00Z",
   *   "params": { "resolution": "320x240" }
   * }
   */

  // Chuyen payload thanh String
  String message = "";
  for (unsigned int i = 0; i < length; i++) {
    message += (char)payload[i];
  }

  Serial.printf("\n[MQTT] Nhan lenh tu topic: %s\n", topic);
  Serial.printf("[MQTT] Noi dung: %s\n", message.c_str());

  // Parse JSON
  JsonDocument doc;
  DeserializationError error = deserializeJson(doc, message);

  if (error) {
    Serial.printf("[MQTT] Loi parse JSON: %s\n", error.c_str());
    return;
  }

  // Lay thong tin lenh
  const char *cmd_id = doc["cmd_id"] | "UNKNOWN";
  const char *command = doc["command"] | "";
  JsonObject params = doc["params"].as<JsonObject>();

  Serial.printf("[CMD] Xu ly lenh: %s (ID: %s)\n", command, cmd_id);

  // Phan loai va xu ly lenh
  String cmdStr = String(command);

  if (cmdStr == "CAPTURE") {
    handleCapture(cmd_id, params);
  } else if (cmdStr == "STREAM_ON") {
    handleStreamOn(cmd_id, params);
  } else if (cmdStr == "STREAM_OFF") {
    handleStreamOff(cmd_id);
  } else if (cmdStr == "RESTART") {
    handleRestart(cmd_id);
  } else if (cmdStr == "CONFIG") {
    handleConfig(cmd_id, params);
  } else {
    Serial.printf("[CMD] Lenh khong ho tro: %s\n", command);
    sendACK(cmd_id, "ERROR", "Lenh khong duoc ho tro: " + cmdStr);
  }
}

// ========================= XU LY TUNG LENH =========================

void handleCapture(const char *cmd_id, JsonObject params) {
  Serial.println("[CMD] Bat dau chup anh...");

  // TODO Phase 4C: Thay bang code OV7670 thuc te
  // Hien tai: tao anh gia lap (test gradient) de kiem tra luong MQTT+HTTP

  bool success = captureAndUpload();

  if (success) {
    sendACK(cmd_id, "OK", "Chup va upload anh thanh cong");
    totalCaptures++;
  } else {
    sendACK(cmd_id, "ERROR", "Loi khi chup hoac upload anh");
  }
}

void handleStreamOn(const char *cmd_id, JsonObject params) {
  // Lay interval tu params (mac dinh 10 giay)
  streamInterval = params["interval"] | 10000;

  isStreaming = true;
  lastStreamCapture = millis();

  Serial.printf("[CMD] BAT che do Stream (interval: %lu ms)\n", streamInterval);
  sendACK(cmd_id, "OK",
          "Stream ON - interval " + String(streamInterval) + "ms");

  // Bao trang thai len Server
  publishStatus("streaming");
}

void handleStreamOff(const char *cmd_id) {
  isStreaming = false;

  Serial.println("[CMD] TAT che do Stream");
  sendACK(cmd_id, "OK", "Stream OFF");

  // Bao trang thai len Server
  publishStatus("idle");
}

void handleRestart(const char *cmd_id) {
  Serial.println("[CMD] Nhan lenh KHOI DONG LAI!");
  sendACK(cmd_id, "OK", "Dang khoi dong lai...");

  // Gui log truoc khi restart
  publishLog("WARNING", "DEVICE_RESTART",
             "Camera Node dang khoi dong lai theo lenh tu Server");

  delay(1000);   // Cho message gui xong
  ESP.restart(); // Khoi dong lai ESP32
}

void handleConfig(const char *cmd_id, JsonObject params) {
  Serial.println("[CMD] Nhan lenh cau hinh:");

  // Xu ly tung tham so cau hinh
  if (params.containsKey("resolution")) {
    const char *res = params["resolution"];
    Serial.printf("  - Resolution: %s\n", res);
    // TODO: Doi resolution camera OV7670
  }

  if (params.containsKey("quality")) {
    int quality = params["quality"];
    Serial.printf("  - Quality: %d\n", quality);
    // TODO: Doi JPEG quality
  }

  if (params.containsKey("stream_interval")) {
    streamInterval = params["stream_interval"];
    Serial.printf("  - Stream interval: %lu ms\n", streamInterval);
  }

  sendACK(cmd_id, "OK", "Da cap nhat cau hinh");
}

// ========================= CHUP & UPLOAD ANH =========================

bool captureAndUpload() {
  /*
   * Luong xu ly:
   *   1. Chup anh tu OV7670 vao buffer RAM
   *   2. Encode JPEG (hoac gui raw)
   *   3. HTTP POST multipart len /api/upload
   *   4. Tra ve true/false
   *
   * Hien tai (Phase 4A): Tao anh test don gian de kiem tra luong giao thuc
   * TODO Phase 4C: Thay bang capture frame tu OV7670 thuc te
   */

  Serial.println("[CAPTURE] Dang chup anh...");

  // ----- TAO ANH TEST (128 bytes dummy JPEG header) -----
  // Day chi la placeholder, se thay bang OV7670 capture o Phase 4C
  // Tao mot JPEG nho (1x1 pixel mau do) de test upload flow
  static const uint8_t testJpeg[] = {
      0xFF, 0xD8, 0xFF, 0xE0, 0x00, 0x10, 0x4A, 0x46, 0x49, 0x46, 0x00, 0x01,
      0x01, 0x00, 0x00, 0x01, 0x00, 0x01, 0x00, 0x00, 0xFF, 0xDB, 0x00, 0x43,
      0x00, 0x08, 0x06, 0x06, 0x07, 0x06, 0x05, 0x08, 0x07, 0x07, 0x07, 0x09,
      0x09, 0x08, 0x0A, 0x0C, 0x14, 0x0D, 0x0C, 0x0B, 0x0B, 0x0C, 0x19, 0x12,
      0x13, 0x0F, 0x14, 0x1D, 0x1A, 0x1F, 0x1E, 0x1D, 0x1A, 0x1C, 0x1C, 0x20,
      0x24, 0x2E, 0x27, 0x20, 0x22, 0x2C, 0x23, 0x1C, 0x1C, 0x28, 0x37, 0x29,
      0x2C, 0x30, 0x31, 0x34, 0x34, 0x34, 0x1F, 0x27, 0x39, 0x3D, 0x38, 0x32,
      0x3C, 0x2E, 0x33, 0x34, 0x32, 0xFF, 0xC0, 0x00, 0x0B, 0x08, 0x00, 0x01,
      0x00, 0x01, 0x01, 0x01, 0x11, 0x00, 0xFF, 0xC4, 0x00, 0x1F, 0x00, 0x00,
      0x01, 0x05, 0x01, 0x01, 0x01, 0x01, 0x01, 0x01, 0x00, 0x00, 0x00, 0x00,
      0x00, 0x00, 0x00, 0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08,
      0x09, 0x0A, 0x0B, 0xFF, 0xDA, 0x00, 0x08, 0x01, 0x01, 0x00, 0x00, 0x3F,
      0x00, 0x54, 0xDB, 0x2E, 0x44, 0xE8, 0x3F, 0xFF, 0xD9};
  int imageSize = sizeof(testJpeg);

  // ----- HTTP POST MULTIPART UPLOAD -----
  Serial.printf("[UPLOAD] Dang gui %d bytes len Server...\n", imageSize);

  HTTPClient http;
  String uploadUrl = String(SERVER_URL) + "/api/upload";
  http.begin(uploadUrl);
  http.setTimeout(15000); // Timeout 15 giay

  // Tao multipart boundary
  String boundary = "----ESP32Boundary" + String(millis());
  http.addHeader("Content-Type", "multipart/form-data; boundary=" + boundary);

  // Xay dung body multipart
  String bodyStart = "--" + boundary + "\r\n";
  bodyStart += "Content-Disposition: form-data; name=\"device_id\"\r\n\r\n";
  bodyStart += String(DEVICE_ID) + "\r\n";
  bodyStart += "--" + boundary + "\r\n";
  bodyStart += "Content-Disposition: form-data; name=\"resolution\"\r\n\r\n";
  bodyStart += "160x120\r\n";
  bodyStart += "--" + boundary + "\r\n";
  bodyStart += "Content-Disposition: form-data; name=\"image\"; "
               "filename=\"capture.jpg\"\r\n";
  bodyStart += "Content-Type: image/jpeg\r\n\r\n";

  String bodyEnd = "\r\n--" + boundary + "--\r\n";

  // Ghep tat ca thanh mot buffer
  int totalSize = bodyStart.length() + imageSize + bodyEnd.length();
  uint8_t *fullBody = (uint8_t *)malloc(totalSize);

  if (!fullBody) {
    Serial.println("[UPLOAD] LOI: Khong du RAM!");
    http.end();
    return false;
  }

  memcpy(fullBody, bodyStart.c_str(), bodyStart.length());
  memcpy(fullBody + bodyStart.length(), testJpeg, imageSize);
  memcpy(fullBody + bodyStart.length() + imageSize, bodyEnd.c_str(),
         bodyEnd.length());

  // Gui HTTP POST voi retry (3 lan)
  int httpCode = -1;
  for (int retry = 0; retry < 3; retry++) {
    httpCode = http.POST(fullBody, totalSize);

    if (httpCode == 201) {
      Serial.printf("[UPLOAD] Thanh cong! HTTP %d\n", httpCode);
      String response = http.getString();
      Serial.printf("[UPLOAD] Response: %s\n", response.c_str());
      break;
    }

    Serial.printf("[UPLOAD] That bai (lan %d/3), HTTP code: %d\n", retry + 1,
                  httpCode);

    if (retry < 2) {
      int delayTime = (retry + 1) * 2000; // Exponential backoff: 2s, 4s
      Serial.printf("[UPLOAD] Thu lai sau %d ms...\n", delayTime);
      delay(delayTime);
    }
  }

  free(fullBody);
  http.end();

  return (httpCode == 201);
}

// ========================= MQTT PUBLISH HELPERS =========================

void sendHeartbeat() {
  /*
   * Gui thong tin "nhip tim" len Broker moi 30 giay.
   * Server nhan heartbeat nay de biet thiet bi con Online.
   * Neu > 90 giay khong nhan → Server danh dau Offline.
   */

  JsonDocument doc;
  doc["device_id"] = DEVICE_ID;
  doc["device_type"] = DEVICE_TYPE;
  doc["ip_address"] = WiFi.localIP().toString();
  doc["wifi_rssi"] = WiFi.RSSI();
  doc["free_heap"] = ESP.getFreeHeap();
  doc["uptime_s"] = (millis() - deviceStartTime) / 1000;
  doc["fw_version"] = FW_VERSION;
  doc["is_streaming"] = isStreaming;
  doc["total_captures"] = totalCaptures;

  String payload;
  serializeJson(doc, payload);

  if (mqttClient.publish(TOPIC_HEARTBEAT, payload.c_str())) {
    Serial.printf(
        "[HEARTBEAT] Gui OK — RSSI: %d dBm | Heap: %d bytes | Uptime: %lus\n",
        WiFi.RSSI(), ESP.getFreeHeap(), (millis() - deviceStartTime) / 1000);
  } else {
    Serial.println("[HEARTBEAT] LOI gui heartbeat!");
  }
}

void sendACK(const char *cmd_id, const char *status, String message) {
  /*
   * Gui phan hoi ACK ve Server sau khi xu ly lenh.
   * Server nhan ACK → cap nhat trang thai lenh trong Database
   *                  → forward len Dashboard qua WebSocket
   */

  JsonDocument doc;
  doc["cmd_id"] = cmd_id;
  doc["device_id"] = DEVICE_ID;
  doc["status"] = status;
  doc["message"] = message;

  String payload;
  serializeJson(doc, payload);

  mqttClient.publish(TOPIC_ACK, payload.c_str());
  Serial.printf("[ACK] %s → %s: %s\n", cmd_id, status, message.c_str());
}

void publishStatus(const char *status) {
  /*
   * Gui thong bao thay doi trang thai cua Camera.
   * Vi du: "idle", "capturing", "streaming", "error"
   */

  JsonDocument doc;
  doc["device_id"] = DEVICE_ID;
  doc["status"] = status;

  String payload;
  serializeJson(doc, payload);

  mqttClient.publish(TOPIC_STATUS, payload.c_str());
  Serial.printf("[STATUS] Trang thai: %s\n", status);
}

void publishLog(const char *level, const char *event, String message) {
  /*
   * Gui log su kien len Server de luu tru va hien thi tren Dashboard.
   * Level: INFO, WARNING, ERROR
   */

  JsonDocument doc;
  doc["device_id"] = DEVICE_ID;
  doc["level"] = level;
  doc["event"] = event;
  doc["message"] = message;

  String payload;
  serializeJson(doc, payload);

  mqttClient.publish(TOPIC_LOG, payload.c_str());
  Serial.printf("[LOG] [%s] %s: %s\n", level, event, message.c_str());
}

// ========================= LED TRANG THAI =========================

void blinkLED() {
  /*
   * Nhap nhay LED khi dang ket noi hoac gap loi.
   * Dung millis() thay vi delay() de khong block CPU.
   */
  if (millis() - lastLedToggle >= LED_BLINK_INTERVAL) {
    ledState = !ledState;
    digitalWrite(LED_STATUS, ledState);
    lastLedToggle = millis();
  }
}
