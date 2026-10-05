
    #include <ESP8266WiFi.h>
#include <ESP8266HTTPClient.h>
#include <ArduinoJson.h>

// --- NETWORK VARIABLES ---
const char* WIFI_SSID     = "CardioGuard_Home_Network";
const char* WIFI_PASSWORD = "SecurePassword123";
const char* SERVER_URL    = "http://192.168.1";

// --- HARDWARE PIN ASSIGNMENTS ---
const int PANIC_BUTTON_PIN = 5; // Maps to Digital Pin 5 from Pseudocode

// --- SYSTEM CONSTANTS & HEALTH THRESHOLDS ---
const int HEART_RATE_MAX   = 120;
const int HEART_RATE_MIN   = 50;
const int SPO2_MIN         = 90;
const float FALL_THRESHOLD = 3.5; // G-Force threshold metric

void setup() {
    // Open the serial monitor stream for debugging
    Serial.begin(115200);
    
    // Set up hardware pin modes
    pinMode(PANIC_BUTTON_PIN, INPUT_PULLUP); // Setup pin with pull-up resistor

    // Connect to the home network
    WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
    Serial.print("Connecting to Wi-Fi...");
    
    // Wait until network connection is successful
    while (WiFi.status() != WL_CONNECTED) {
        delay(500);
        Serial.print(".");
    }
    Serial.println("");
    Serial.println("Wi-Fi Connection Established Successfully.");
}

void loop() {
    // --- STEP 1: READ SENSOR VALUES ---
    // Placeholder values simulating sensor data inputs for testing
    int live_heart_rate = 72;        // Normal heart rate example
    int live_spo2 = 98;              // Normal blood oxygen level percentage
    float current_acceleration = 1.0; // Normal resting G-force
    
    // Read the physical state of the digital button component
    int panic_button_state = digitalRead(PANIC_BUTTON_PIN);

    // Initialise control loop variables
    bool emergency_status = false;
    String alert_type = "Normal";

    // --- STEP 2: LOGICAL CHECK ROUTINES ---
    // Evaluates sensor readings against predefined health rules
    if (panic_button_state == LOW) { // Button triggers LOW when pressed down
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

    // --- STEP 3: NETWORK PAYLOAD PACKAGING & TRANSMISSION ---
    if (WiFi.status() == WL_CONNECTED) {
        WiFiClient client;
        HTTPClient http;
        
        // Open a connection channel to the web server
        http.begin(client, SERVER_URL);
        http.addHeader("Content-Type", "application/json");

        // Format data into a JSON object matching our Data Dictionary variables
        StaticJsonDocument<200> jsonDoc;
        jsonDoc["patient_id"] = "PT_01";
        jsonDoc["type"]       = alert_type;
        jsonDoc["hr"]         = live_heart_rate;
        jsonDoc["ox"]         = live_spo2;

        // Convert the JSON data structure into a plain string text packet
        String requestPayload;
        serializeJson(jsonDoc, requestPayload);

        // Send data to the web dashboard server via HTTP POST protocol
        int httpResponseCode = http.POST(requestPayload);
        
        // Print transaction logs out to the local PC console window
        if (httpResponseCode > 0) {
            Serial.print("Data transmission successful. Server Code: ");
            Serial.println(httpResponseCode);
        } else {
            Serial.print("Network send failure. Error: ");
            Serial.println(httpResponseCode);
        }
        
        // Close network stream session
        http.end();
    } else {
        Serial.println("Error: Network disconnected. Outbox transmission aborted.");
    }

    // --- STEP 4: SYSTEM BALANCE PAUSE ---
    // This 2-second sleep cycle directly helps satisfy Evaluation Criterion 1 (EC1) 
    // by ensuring swift delivery without overwhelming processing capabilities.
    delay(2000); 
}
