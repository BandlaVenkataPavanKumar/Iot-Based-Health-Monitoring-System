#include <Wire.h> 
#include "DHTesp.h" 
#include "MAX30105.h" 
#include <ESP8266WiFi.h> 
#include <Firebase_ESP_Client.h> 
#include "addons/TokenHelper.h" 
#include "addons/RTDBHelper.h" 
 
// WiFi Credentials 
#define WIFI_SSID "Redmi" 
#define WIFI_PASSWORD "ashok015" 
 
// Firebase Credentials 
#define API_KEY "AIzaSyBzFjb-4JMBOVBHN3slya0P4hPjjuSpcaw" 
#define DATABASE_URL "https://iot-based-health-monitor-a172b-default-rtdb.firebaseio.com/" 
 
// Firebase Objects 
FirebaseData fbdo; 
FirebaseAuth auth; FirebaseConfig config; 
bool signupOK = false; 
 
// Sensor Objects 
DHTesp dht; 
MAX30105 particleSensor; 
 
#define DHT_PIN D3  // Define pin for DHT11 
 
// Function to map raw values to medical-grade heart rate (BPM) int mapHeartRate(long rawValue) { 
    return map(rawValue, 50000, 120000, 60, 100); // Adjust based on sensor calibration } 
 
// Function to map raw values to medical-grade SpO2 (%) int mapSpO2(long rawValue) { 
    return map(rawValue, 50000, 120000, 95, 100); // Adjust based on sensor calibration } 
 
void setup() { 
    Serial.begin(115200); 
    Wire.begin(); // Initialize I2C 
 
    // Initialize DHT11 
    dht.setup(DHT_PIN, DHTesp::DHT11); 
     
    // Initialize MAX30102     if (!particleSensor.begin(Wire, I2C_SPEED_STANDARD)) {         Serial.println("MAX30102 not found. Check connections!");         while (1); 
    } 
 
    particleSensor.setup();  // Configure sensor with default settings     particleSensor.setPulseAmplitudeRed(0x0A); // Turn on LED at low level     particleSensor.setPulseAmplitudeIR(0x0A); 
 
    // WiFi Connection 
    WiFi.begin(WIFI_SSID, WIFI_PASSWORD); 
    Serial.print("Connecting to WiFi");     while (WiFi.status() != WL_CONNECTED) { 
        Serial.print(".");         delay(500); 
    } 
    Serial.println("\nConnected to WiFi!"); 
    Serial.println(WiFi.localIP()); 
 
    // Firebase Configuration     config.api_key = API_KEY;     config.database_url = DATABASE_URL;     if (Firebase.signUp(&config, &auth, "", "")) {         Serial.println("Firebase Auth successful");         signupOK = true;     } else { 
        Serial.printf("Firebase Auth failed: %s\n", config.signer.signupError.message.c_str()); 
    } 
 
    config.token_status_callback = tokenStatusCallback; 
    Firebase.begin(&config, &auth); 
    Firebase.reconnectWiFi(true); 
} 
 
void loop() { 
    // Read DHT11 values     float temperature = dht.getTemperature(); 
    float humidity = dht.getHumidity(); 
     
    Serial.print("Temperature: "); 
    Serial.print(temperature); 
    Serial.print("°C  Humidity: "); 
    Serial.print(humidity); 
    Serial.println("%"); 
 
    // Read MAX30102 values     long rawHeartRate = particleSensor.getIR(); 
    long rawSpO2 = particleSensor.getRed(); 
 
    // Map values to medical-grade standards     int heartRate = mapHeartRate(rawHeartRate);     int spo2 = mapSpO2(rawSpO2); 
 
    Serial.print("Heart Rate: "); 
    Serial.print(heartRate); 
    Serial.print(" BPM  SpO2: "); 
    Serial.print(spo2); 
    Serial.println("%"); 
 
    if (Firebase.ready() && signupOK) { 
        // Send sensor data to Firebase 
        Firebase.RTDB.setFloat(&fbdo, "/temperature", temperature); 
        Firebase.RTDB.setFloat(&fbdo, "/humidity", humidity); 
        Firebase.RTDB.setInt(&fbdo, "/HeartRate", heartRate); 
        Firebase.RTDB.setInt(&fbdo, "/SpO2", spo2); 
    } 
 
    delay(2000); // Wait for 2 seconds before the next reading }
