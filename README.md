#include <WiFi.h>
#include <PubSubClient.h>
#include <ESP32Servo.h>

// === WIFI et MQTT ===
const char* ssid = "Wokwi-GUEST";
const char* password = "";  
const char* mqttServer = "broker.hivemq.com"; 
const int mqttPort = 1883;

WiFiClient espClient;
PubSubClient client(espClient);

// === PINS ===
const int flamePin = 25;
const int buzzerPin = 27;

const int gasPin = 34;
const int gasLedPin = 14;
Servo gasServo;

const int pirPin = 18;
const int relayPin = 5;

// --- Buzzer ---
void buzzerOn() { tone(buzzerPin, 1000); }
void buzzerOff() { noTone(buzzerPin); }

// --- Connexion WiFi ---
void setup_wifi() {
  delay(10);
  Serial.println("Connexion au WiFi...");
  WiFi.begin(ssid, password);

  unsigned long startAttemptTime = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - startAttemptTime < 10000) {
    delay(500);
    Serial.print(".");
  }

  if(WiFi.status() != WL_CONNECTED){
    Serial.println("\n❌ Impossible de se connecter au WiFi !");
  } else {
    Serial.println("\n✅ WiFi connecté !");
    Serial.print("Adresse IP : ");
    Serial.println(WiFi.localIP());
  }
}

// --- Reconnexion MQTT ---
void reconnect() {
  while (!client.connected()) {
    Serial.print("Connexion MQTT...");
    if (client.connect("ESP32_SmartHome")) {
      Serial.println("✅ Connecté !");
    } else {
      Serial.print("❌ Erreur, rc=");
      Serial.print(client.state());
      delay(2000);
    }
  }
}

// --- Setup ---
void setup() {
  Serial.begin(115200);

  pinMode(flamePin, INPUT);
  pinMode(buzzerPin, OUTPUT);
  buzzerOff();

  pinMode(gasPin, INPUT);
  pinMode(gasLedPin, OUTPUT);

  // --- Servo setup ---
  ESP32PWM::allocateTimer(0);
  ESP32PWM::allocateTimer(1);
  ESP32PWM::allocateTimer(2);
  ESP32PWM::allocateTimer(3);

  gasServo.setPeriodHertz(50); // fréquence PWM = 50 Hz
  gasServo.attach(19, 500, 2400); // pin 13 = signal servo
  gasServo.write(0); // vanne ouverte

  pinMode(pirPin, INPUT);
  pinMode(relayPin, OUTPUT);
  digitalWrite(relayPin, LOW);

  setup_wifi();
  client.setServer(mqttServer, mqttPort);
}

// --- Loop ---
void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  // === Flamme ===
  int flameState = digitalRead(flamePin);
  if (flameState == HIGH) {
    buzzerOn();
    Serial.println("🔥 Flamme détectée !");
    client.publish("smarthome/flame", "1");
  } else {
    buzzerOff();
    Serial.println("✅ Pas de flamme");
    client.publish("smarthome/flame", "0");
  }

  // === Gaz ===
  int gasState = digitalRead(gasPin);
  if (gasState == HIGH) {
    digitalWrite(gasLedPin, HIGH);
    gasServo.write(90); // ferme vanne
    Serial.println("⚠  Gaz détecté ! ");
    client.publish("smarthome/gas", "1");
  } else {
    digitalWrite(gasLedPin, LOW);
    gasServo.write(0); // ouvre vanne
    Serial.println("✅ Pas de gaz");
    client.publish("smarthome/gas", "0");
  }

  // === PIR ===
  int pirState = digitalRead(pirPin);
  if (pirState == HIGH) {
    digitalWrite(relayPin, HIGH);
    Serial.println("👤 Mouvement détecté !");
    client.publish("smarthome/motion", "1");
  } else {
    digitalWrite(relayPin, LOW);
    Serial.println("🚫 Aucun mouvement");
    client.publish("smarthome/motion", "0");
  }

  Serial.println("----------------------");
  delay(500);
}
