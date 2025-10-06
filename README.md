//Code ESP8266

#include <ESP8266WiFi.h>
#include <WiFiClient.h>
#include <ESP8266WebServer.h>

// --- BIẾN TOÀN CỤC ĐỂ LƯU DỮ LIỆU TỪ STM32 ---
String temperature = "N/A"; // Giá trị nhiệt độ (mặc định N/A)
String humidity = "N/A";    // Giá trị độ ẩm (mặc định N/A)

// --- THÔNG TIN WI-FI (THAY ĐỔI THEO MẠNG CỦA BẠN) ---
const char* ssid = "hang_phong";
const char* password = "16102012";

// --- KHAI BÁO SERVER ---
ESP8266WebServer server(80);

// --- GIAO DIỆN WEB ---
String webPage = R"rawliteral(
<!DOCTYPE html>
<html>
<head>
  <meta http-equiv='Content-Type' content='text/html; charset=utf-8'>
  <title>ESP8266 Monitor</title>
  <meta name='viewport' content='width=device-width, initial-scale=1'>
  <meta http-equiv='refresh' content='10'>
  <style>
    :root {
      --color-bg: #f4f7f6;
      --color-sensor-bg: #e3f2fd;
    }
    body { font-family: sans-serif; background-color: var(--color-bg); margin: 0; }
    .container { max-width: 400px; margin: 50px auto; padding: 20px; text-align: center; }
    h1 { color: #333; margin-bottom: 30px; font-size: 1.5em; }
    .sensor-card {
      padding: 20px;
      background-color: var(--color-sensor-bg);
      border-radius: 12px;
      box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
      border-left: 5px solid #2196f3;
      text-align: left;
      animation: fadeIn 0.5s ease-out;
    }
    @keyframes fadeIn { from { opacity: 0; transform: translateY(-10px); } to { opacity: 1; transform: translateY(0); } }
    .sensor-card p { margin: 10px 0; font-size: 1.5em; color: #333; }
    .sensor-card strong { color: #0d47a1; }
  </style>
</head>
<body>
<div class='container'>
  <h1>THEO DÕI NHIỆT ĐỘ & ĐỘ ẨM</h1>
  <div class='sensor-card'>
    <p>🌡️ Nhiệt độ: <strong>%TEMPERATURE% &deg;C</strong></p>
    <p>💧 Độ ẩm: <strong>%HUMIDITY% &percnt;</strong></p>
  </div>
</div>
</body>
</html>
)rawliteral";

// --- HÀM XỬ LÝ GỬI TRANG WEB ---
void handleRoot() {
  String page = webPage;
  page.replace("%TEMPERATURE%", temperature);
  page.replace("%HUMIDITY%", humidity);
  server.send(200, "text/html", page);
}

// --- SETUP ---
void setup() {
  // Bắt đầu Serial ở baud rate 115200 để giao tiếp với STM32
  Serial.begin(115200);
  
  // Kết nối Wi-Fi
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("");
  Serial.println("WiFi connected");
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());
  
  // Cấu hình đường dẫn cho trang chủ
  server.on("/", handleRoot);
  
  server.begin();
  Serial.println("HTTP server started");
  Serial.println("=== DEBUG: Chờ dữ liệu từ STM32 ===");
}

// --- LOOP ---
void loop() {
  // Kiểm tra dữ liệu từ STM32
  if (Serial.available() > 0) {
    String data = Serial.readStringUntil('\n');
    data.trim();
    
    // DEBUG: In dữ liệu nhận được
    Serial.print("Received: [");
    Serial.print(data);
    Serial.println("]");
    
    // Kiểm tra lỗi từ STM32 (bao gồm cả "DHT11 not found")
    if (data.indexOf("Error reading DHT11") != -1 || data.indexOf("DHT11 not found") != -1) {
      Serial.println("DEBUG: Nhận lỗi từ DHT11");
      temperature = "N/A";
      humidity = "N/A";
    }
    // Parse dữ liệu dạng "TEMP:xx.x,HUM:yy.y"
    else if (data.startsWith("TEMP:") && data.indexOf(",HUM:") != -1) {
      float tempValue, humValue;
      int parsed = sscanf(data.c_str(), "TEMP:%f,HUM:%f", &tempValue, &humValue);
      if (parsed == 2 && tempValue >= -40 && tempValue <= 125 && humValue >= 0 && humValue <= 100) {
        temperature = String(tempValue, 1); // Format 1 chữ số thập phân
        humidity = String(humValue, 1);
        Serial.println("DEBUG: Parse thành công - Temp: " + temperature + ", Hum: " + humidity);
      } else {
        Serial.println("DEBUG: Giá trị không hợp lệ hoặc parse lỗi");
        temperature = "N/A";
        humidity = "N/A";
      }
    } else {
      Serial.println("DEBUG: Format dữ liệu không hợp lệ: " + data);
      temperature = "N/A";
      humidity = "N/A";
    }
  }

  // Xử lý yêu cầu từ trình duyệt web
  server.handleClient();
  yield(); // Cho phép hệ thống xử lý các tác vụ khác
}
