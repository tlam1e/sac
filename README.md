#include <ESP8266WiFi.h>
#include <ESP8266HTTPClient.h>
#include <ArduinoJson.h>

// --- NETWORK CONFIGURATION ---
const char* WIFI_SSID     = "CardioGuard_Home_Network";
const char* WIFI_PASSWORD = "SecurePassword123";
const char* SERVER_URL    = "http://192.168.1";

// --- PIN ASSIGNMENTS ---
const int PANIC_BUTTON_PIN = D5;

// --- CRITICAL THRESHOLDS ---
const int HEART_RATE_MAX   = 120;
const int HEART_RATE_MIN   = 50;
const int SPO2_MIN         = 90;
const float FALL_THRESHOLD = 3.50; // Measured in G-forces

void setup() {
    Serial.begin(115200);
    pinMode(PANIC_BUTTON_PIN, INPUT_PULLUP); // Active LOW configuration

    // Initialise Wi-Fi Connectivity
    WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
    Serial.print("Connecting to Wi-Fi...");
    while (WiFi.status() != WL_CONNECTED) {
        delay(500);
        Serial.print(".");
    }
    Serial.println("\nWi-Fi Connection Established Successfully.");
}

void loop() {
    // --- SIMULATED/READ SENSOR VALUES (MOCK INPUTS FOR PHYSICAL VALIDATION) ---
    int live_heart_rate = 72;        // Standard normal placeholder
    int live_spo2 = 98;              // Standard normal placeholder
    float current_acceleration = 1.0; // Rest state G-force
    
    // Read physical hardware components
    int panic_button_state = digitalRead(PANIC_BUTTON_PIN);

    bool emergency_status = false;
    String alert_type = "Normal";

    // --- ALGORITHMIC EVALUATION LOGIC ---
    if (panic_button_state == LOW) { // Button closed to GND
        emergency_status = true;
        alert_type = "Manual Panic Pressed";
    } 
    else if (current_acceleration > FALL_THRESHOLD) {
        emergency_status = true;
        alert_type = "Automated Fall Detected";
    } 
    else if (live_heart_rate > HEART_RATE_MAX || live_heart_rate < HEART_RATE_MIN) {
        emergency_status = true;
        alert_type = "Heart Rate Anomaly";
    } 
    else if (live_spo2 < SPO2_MIN) {
        emergency_status = true;
        alert_type = "Critical Oxygen Drop";
    }

    // --- PACKAGING AND NETWORK TRANSMISSION ---
    if (WiFi.status() == WL_CONNECTED) {
        WiFiClient client;
        HTTPClient http;
        
        http.begin(client, SERVER_URL);
        http.addHeader("Content-Type", "application/json");

        // Construct JSON document payload buffer
        StaticJsonDocument<200> jsonDoc;
        jsonDoc["patient_id"] = "PT_01";
        jsonDoc["type"]       = alert_type;
        jsonDoc["hr"]         = live_heart_rate;
        jsonDoc["ox"]         = live_spo2;

        String requestPayload;
        serializeJson(jsonDoc, requestPayload);

        // Dispatched compiled dataset via HTTP POST
        int httpResponseCode = http.POST(requestPayload);
        
        if (httpResponseCode > 0) {
            Serial.printf("[HTTP] POST Response Status Code: %d\n", httpResponseCode);
        } else {
            Serial.printf("[HTTP] Error sending telemetry payload: %s\n", http.errorToString(httpResponseCode).c_str());
        }
        
        http.end();
    } else {
        Serial.println("[ERROR] Wi-Fi Link Disconnected. Buffering aborted.");
    }

    // Delay interval loop explicitly balancing EC1 (Latency) and EC2 (Power efficiency)
    delay(2000); 
}
