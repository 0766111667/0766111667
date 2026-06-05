### $
SrNdAbrCf6
p=6QTQh6tHWJ1
* 218.188.99.55
* 116.50.184.120

#include <WiFi.h>
#include <WiFiClientSecure.h>
#include "soc/soc.h"
#include "soc/rtc_cntl_reg.h"
#include "esp_camera.h"
#include <UniversalTelegramBot.h>
#include <ArduinoJson.h>

// ================== CẤU HÌNH CỦA BẠN ==================
const char* ssid = "69/164_2.4G";
const char* password = "22446688";

String BOTtoken = "8965236284:AAH5FYnOOBrBqcg0xPpV-1LBNa51JaYpA-U";
String CHAT_ID = "6248936531";
// =====================================================

bool sendPhoto = false;
WiFiClientSecure clientTCP;
UniversalTelegramBot bot(BOTtoken, clientTCP);

#define FLASH_LED_PIN 4
bool flashState = false;

int botRequestDelay = 1000;
unsigned long lastTimeBotRan = 0;

// Camera pins cho ESP32-CAM (AI-Thinker)
#define PWDN_GPIO_NUM    32
#define RESET_GPIO_NUM   -1
#define XCLK_GPIO_NUM     0
#define SIOD_GPIO_NUM    26
#define SIOC_GPIO_NUM    27
#define Y9_GPIO_NUM      35
#define Y8_GPIO_NUM      34
#define Y7_GPIO_NUM      39
#define Y6_GPIO_NUM      36
#define Y5_GPIO_NUM      21
#define Y4_GPIO_NUM      19
#define Y3_GPIO_NUM      18
#define Y2_GPIO_NUM       5
#define VSYNC_GPIO_NUM   25
#define HREF_GPIO_NUM    23
#define PCLK_GPIO_NUM    22

void configInitCamera() {
  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer = LEDC_TIMER_0;
  config.pin_d0 = Y2_GPIO_NUM;
  config.pin_d1 = Y3_GPIO_NUM;
  config.pin_d2 = Y4_GPIO_NUM;
  config.pin_d3 = Y5_GPIO_NUM;
  config.pin_d4 = Y6_GPIO_NUM;
  config.pin_d5 = Y7_GPIO_NUM;
  config.pin_d6 = Y8_GPIO_NUM;
  config.pin_d7 = Y9_GPIO_NUM;
  config.pin_xclk = XCLK_GPIO_NUM;
  config.pin_pclk = PCLK_GPIO_NUM;
  config.pin_vsync = VSYNC_GPIO_NUM;
  config.pin_href = HREF_GPIO_NUM;
  config.pin_sccb_sda = SIOD_GPIO_NUM;
  config.pin_sccb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn = PWDN_GPIO_NUM;
  config.pin_reset = RESET_GPIO_NUM;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;
  config.grab_mode = CAMERA_GRAB_LATEST;

  if (psramFound()) {
    config.frame_size = FRAMESIZE_UXGA;
    config.jpeg_quality = 10;
    config.fb_count = 2;
  } else {
    config.frame_size = FRAMESIZE_SVGA;
    config.jpeg_quality = 12;
    config.fb_count = 1;
  }

  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("Camera init failed 0x%x", err);
    delay(1000);
    ESP.restart();
  }
}

String sendPhotoTelegram() {
  camera_fb_t * fb = NULL;
  fb = esp_camera_fb_get();
  if (!fb) {
    Serial.println("Camera capture failed");
    return "Camera capture failed";
  }

  clientTCP.setTimeout(30000);
  if (!clientTCP.connect("api.telegram.org", 443)) {
    Serial.println("Connection to Telegram failed");
    esp_camera_fb_return(fb);
    return "Connection failed";
  }

  String head = "--RandomNerdTutorials\r\nContent-Disposition: form-data; name=\"chat_id\"; \r\n\r\n" + CHAT_ID + "\r\n--RandomNerdTutorials\r\nContent-Disposition: form-data; name=\"photo\"; filename=\"esp32cam.jpg\"\r\nContent-Type: image/jpeg\r\n\r\n";
  String tail = "\r\n--RandomNerdTutorials--\r\n";

  uint32_t imageLen = fb->len;
  uint32_t totalLen = head.length() + imageLen + tail.length();

  clientTCP.println("POST /bot"+BOTtoken+"/sendPhoto HTTP/1.1");
  clientTCP.println("Host: api.telegram.org");
  clientTCP.println("Content-Length: " + String(totalLen));
  clientTCP.println("Content-Type: multipart/form-data; boundary=RandomNerdTutorials");
  clientTCP.println();
  clientTCP.print(head);

  uint8_t *fbBuf = fb->buf;
  size_t fbLen = fb->len;
  for (size_t n=0; n<fbLen; n=n+1024) {
    if (n+1024 < fbLen) {
      clientTCP.write(fbBuf, 1024);
      fbBuf += 1024;
    } else if (fbLen%1024>0) {
      size_t remainder = fbLen%1024;
      clientTCP.write(fbBuf, remainder);
    }
  }  
  clientTCP.print(tail);
  
  esp_camera_fb_return(fb);

  while (clientTCP.connected()) {
    String line = clientTCP.readStringUntil('\n');
    if (line == "\r") break;
  }
  return "Photo sent";
}

void handleNewMessages(int numNewMessages) {
  for (int i = 0; i < numNewMessages; i++) {
    String chat_id = String(bot.messages[i].chat_id);
    if (chat_id != CHAT_ID) {
      bot.sendMessage(chat_id, "Unauthorized!", "");
      continue;
    }

    String text = bot.messages[i].text;
    if (text == "/start") {
      String welcome = "ESP32-CAM Telegram Bot\n\n";
      welcome += "/photo - Chụp và gửi ảnh\n";
      welcome += "/flash - Bật/Tắt đèn flash thủ công\n";
      bot.sendMessage(CHAT_ID, welcome, "");
    }
    if (text == "/photo") {
      sendPhoto = true;
    }
    if (text == "/flash") {
      flashState = !flashState;
      digitalWrite(FLASH_LED_PIN, flashState);
      bot.sendMessage(CHAT_ID, flashState ? "Đèn Flash ✅ BẬT" : "Đèn Flash ❌ TẮT", "");
    }
  }
}

void setup() {
  WRITE_PERI_REG(RTC_CNTL_BROWN_OUT_REG, 0); // Tắt brownout chống sập nguồn
  Serial.begin(115200);
  pinMode(FLASH_LED_PIN, OUTPUT);
  digitalWrite(FLASH_LED_PIN, flashState);

  configInitCamera();

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  clientTCP.setCACert(TELEGRAM_CERTIFICATE_ROOT); // Thiết lập chứng chỉ bảo mật cho Telegram

  Serial.print("Đang kết nối WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi đã kết nối!");
  Serial.print("IP: "); Serial.println(WiFi.localIP());
}

void loop() {
  if (sendPhoto) {
    Serial.println("📸 Đang chụp và gửi ảnh...");
    
    // Tự động bật flash nếu đang tắt để ảnh không bị tối
    bool tempFlash = false;
    if (!flashState) {
      digitalWrite(FLASH_LED_PIN, HIGH);
      tempFlash = true;
      delay(500); // Chờ đèn flash sáng ổn định trước khi bắt hình
    }

    String result = sendPhotoTelegram();
    Serial.println(result);
    
    // Tắt flash sau khi chụp xong nếu trước đó nó được bật tự động
    if (tempFlash) {
      digitalWrite(FLASH_LED_PIN, LOW);
    }

    sendPhoto = false;
  }

  if (millis() > lastTimeBotRan + botRequestDelay) {
    int numNewMessages = bot.getUpdates(bot.last_message_received + 1);
    while (numNewMessages) {
      handleNewMessages(numNewMessages);
      numNewMessages = bot.getUpdates(bot.last_message_received + 1);
    }
    lastTimeBotRan = millis();
  }
}
