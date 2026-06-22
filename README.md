#include <WiFi.h>
#include <HTTPClient.h>
#include <UrlEncode.h>
#include <ESP32Servo.h>
#include <LiquidCrystal_I2C.h>

#define SENSOR1 26
#define SENSOR2 35

long currentMillis = 0;
long previousMillis = 0;
int interval = 1000;
float calibrationFactor = 4.5;

// Variables for flow meter 1
volatile byte pulseCount1;
byte pulse1Sec1 = 0;
float flowRate1;
unsigned int flowMilliLitres1;
unsigned long totalMilliLitres1;

// Variables for flow meter 2
volatile byte pulseCount2;
byte pulse1Sec2 = 0;
float flowRate2;
unsigned int flowMilliLitres2;
unsigned long totalMilliLitres2;

const int trigPin = 5;
const int echoPin = 19;
const int buzzer = 4;
#define sensor_cek 14
#define sensor_cek1 15

// Wifi
const char* ssid = "KONSIKI LT1";
const char* password = "KonsIKI1";
String phoneNumber = "6285645905467";
String apiKey = "1437155";
String kondisi_pintu = "Baik";
String derajat_servo;

// Interval pengiriman data ke web
unsigned long lastSendDataTime = 0;
const int sendDataInterval = 5000;

// Inisialisasi Servo dan LCD
Servo servo_1;
Servo servo_2;
LiquidCrystal_I2C lcd(0x27, 20, 4);

// Fungsi untuk menghitung aliran dari masing-masing sensor
void IRAM_ATTR pulseCounter1() {
  pulseCount1++;
}

void IRAM_ATTR pulseCounter2() {
  pulseCount2++;
}

// Fungsi untuk mengambil data dari web
String get_data_web() {
  HTTPClient http;
  String url = "https://jagoiot.pythonanywhere.com/ivan/skripsi";
  http.begin(url);
  int httpResponCode = http.GET();
  String resp = http.getString();
  http.end();
  return resp;
}


void setup() {
  Serial.begin(115200);

  // Koneksi WiFi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi...");
  }
  Serial.println("Connected to WiFi");
  tone(buzzer, 1000);  // Turn on the buzzer
  delay(2000);
  noTone(buzzer);  // Turn off the buzzer

  //cek pintu
  pinMode(sensor_cek, INPUT_PULLUP);
  pinMode(sensor_cek1, INPUT_PULLUP);

  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);

  // Inisialisasi Servo
  ESP32PWM::allocateTimer(0);
  ESP32PWM::allocateTimer(1);
  ESP32PWM::allocateTimer(2);
  ESP32PWM::allocateTimer(3);

  servo_1.setPeriodHertz(50);
  servo_2.setPeriodHertz(50);

  servo_1.attach(18, 1000, 2000);
  servo_2.attach(25, 1000, 2000);

  // Setup for flow meter 1
  pinMode(SENSOR1, INPUT_PULLUP);
  pulseCount1 = 0;
  flowRate1 = 0.0;
  flowMilliLitres1 = 0;
  totalMilliLitres1 = 0;

  // Setup for flow meter 2
  pinMode(SENSOR2, INPUT_PULLUP);
  pulseCount2 = 0;
  flowRate2 = 0.0;
  flowMilliLitres2 = 0;
  totalMilliLitres2 = 0;

  previousMillis = 0;

  attachInterrupt(digitalPinToInterrupt(SENSOR1), pulseCounter1, FALLING);
  attachInterrupt(digitalPinToInterrupt(SENSOR2), pulseCounter2, FALLING);

  // Inisialisasi LCD
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("    TA BENDUNGAN    ");
}

void loop() {

  // Mengukur tinggi air dengan sensor ultrasonik
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  long durasi = pulseIn(echoPin, HIGH);
  float tinggi_air = 13.89 - (durasi * 0.034) / 2;
  if (tinggi_air < 0.0) {
    tinggi_air = 0.0;
  }
  if (tinggi_air > 10.0) {
    tinggi_air = 10.0;
  }
  

  //servo
  String response = get_data_web();
  int responServo = response.toInt();
  Serial.print("Response Servo: ");
  Serial.println(responServo);
  if (responServo == 45) {
    derajat_servo = "25%";
    servo_1.write(70);
    servo_2.write(90);
  } else if (responServo == 90) {
    derajat_servo = "50%";
    servo_1.write(80);
    servo_2.write(60);
  } else if (responServo == 135) {
    derajat_servo = "75%";
    servo_1.write(10);
    servo_2.write(30);
  } else if (responServo == 180) {
    derajat_servo = "100%";
    servo_1.write(0);
    servo_2.write(0);
  }/*else if (responServo == 255){
    derajat_servo = "0%";
    servo_1.write(180);
    servo_2.write)180);
  }*/

  
  //Otomatis
  if (responServo == 255) {
    int servoAngle = map(tinggi_air, 0, 10, 40, 180);
    servoAngle = constrain(servoAngle, 40, 180);
    derajat_servo = String(map(tinggi_air, 0, 10, 0, 100));
    servo_1.write(servoAngle);
    servo_2.write(servoAngle);
  }
  if (tinggi_air < 10){
    int servoAngle = map(tinggi_air, 0, 10, 40, 180);
    servoAngle = constrain(servoAngle, 40, 180);
    servo_1.write(servoAngle);
    servo_2.write(servoAngle);
  }
  if (tinggi_air >= 10) {
    int servoAngel = map(tinggi_air, 0, 10, 40, 180);
    servoAngel = constrain(servoAngel, 40, 45);
    servo_1.write(servoAngel);
    servo_2.write(servoAngel);
    tone(buzzer, 1000);  // Turn on the buzzer
    delay(2000);
    noTone(buzzer);  // Turn off the buzzer
    sendMessage("Peringatan!!!! Air bendungan " + String(tinggi_air) + " cm melibihi batas maksimal");
  }

  if (responServo != 255 && (digitalRead(sensor_cek) == LOW || digitalRead(sensor_cek1) == LOW)) {
    kondisi_pintu = "baik";
  } else if (responServo != 255 && (digitalRead(sensor_cek) == HIGH || digitalRead(sensor_cek1) == HIGH)) {
    kondisi_pintu = "rusak";
  }


  currentMillis = millis();
  if (currentMillis - previousMillis > interval) {
    // Flow meter 1 calculations
    pulse1Sec1 = pulseCount1;
    pulseCount1 = 0;
    flowRate1 = ((1000.0 / (millis() - previousMillis)) * pulse1Sec1) / calibrationFactor;
    flowMilliLitres1 = (flowRate1 / 60) * 1000;
    totalMilliLitres1 += flowMilliLitres1;

    // Flow meter 2 calculations
    pulse1Sec2 = pulseCount2;
    pulseCount2 = 0;
    flowRate2 = ((1000.0 / (millis() - previousMillis)) * pulse1Sec2) / calibrationFactor;
    flowMilliLitres2 = (flowRate2 / 60) * 1000;
    totalMilliLitres2 += flowMilliLitres2;

    previousMillis = millis();

    // Print flow meter 1 data
    Serial.print("Flow meter 1 - Flow rate: ");
    Serial.print(int(flowRate1));  // Print the integer part of the variable
    Serial.print(" L/min");
    Serial.print("\t");  // Print tab space
    Serial.print("Output Liquid Quantity: ");
    Serial.print(totalMilliLitres1);
    Serial.print(" mL / ");
    Serial.print(totalMilliLitres1 / 1000);
    Serial.println(" L");

    // Print flow meter 2 data
    Serial.print("Flow meter 2 - Flow rate: ");
    Serial.print(int(flowRate2));  // Print the integer part of the variable
    Serial.print(" L/min");
    Serial.print("\t");  // Print tab space
    Serial.print("Output Liquid Quantity: ");
    Serial.print(totalMilliLitres2);
    Serial.print(" mL / ");
    Serial.print(totalMilliLitres2 / 1000);
    Serial.println(" L");
  }

  int flowRate = flowRate1 + flowRate2;
  // Kirim data ke web setiap 5 detik sekali
  if (millis() - lastSendDataTime >= sendDataInterval) {
    String url = "https://jagoiot.pythonanywhere.com/sensor?a=" + String(tinggi_air) + "&b=" + String(derajat_servo) + "&desc_a=" + kondisi_pintu + "&c=" + String(flowRate);
    HTTPClient http;
    http.begin(url);
    int httpResponseCode = http.POST(url);

    if (httpResponseCode == 200) {
      Serial.println("Berhasil mengirim ke web");
    } else {
      Serial.println("Error sending the message");
      Serial.print("HTTP response code: ");
      Serial.println(httpResponseCode);
    }
    http.end();
    lastSendDataTime = millis();
  }
  // Output ke Serial Monitor dan LCD
  Serial.print("Total Flow rate: ");
  Serial.print(flowRate, DEC);
  Serial.println(" L/minute");
  Serial.print("Water level: ");
  Serial.print(tinggi_air);
  Serial.println(" cm");

  lcd.setCursor(0, 0);
  lcd.print("    TA BENDUNGAN    ");
  lcd.setCursor(0, 1);
  lcd.print("Status pintu: ");
  lcd.print(kondisi_pintu);
  lcd.setCursor(0, 2);
  lcd.print("Flow rate: ");
  lcd.print(flowRate, DEC);
  lcd.print(" L/min");
  lcd.setCursor(0, 3);
  lcd.print("Tinggi air: ");
  lcd.print(tinggi_air);
  lcd.print(" cm");
}

// Fungsi untuk mengirim pesan ke WhatsApp
void sendMessage(String message) {
  String url = "https://api.callmebot.com/whatsapp.php?phone=" + phoneNumber + "&text=" + urlEncode(message) + "&apikey=" + apiKey;
  HTTPClient http;
  http.begin(url);
  http.addHeader("Content-Type", "application/x-www-form-urlencoded");

  int httpResponseCode = http.POST(url);
  if (httpResponseCode == 200) {
    Serial.print("Berhasil mengirim ke Bot WA");
  } else {
    Serial.println("Error sending the message");
    Serial.print("HTTP response code: ");
    Serial.println(httpResponseCode);
  }
  http.end();
}
