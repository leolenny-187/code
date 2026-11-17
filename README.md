# code
pp
#include <Wire.h>
#include <Adafruit_PWMServoDriver.h>

// PCA9685 Objekt initialisieren (Standard-I2C-Adresse 0x40)
Adafruit_PWMServoDriver pwm = Adafruit_PWMServoDriver();

// Pin-Definitionen Ultraschallsensor (bleiben am Arduino)
const int TRIG_PIN = 2;
const int ECHO_PIN = 3;

// Kanal-Definitionen auf dem PCA9685 Servotreiber
const int CH_RADAR_PAN     = 1;
const int CH_RADAR_TILT    = 4;
const int CH_LAUNCHER_PAN  = 5;
const int CH_LAUNCHER_TILT = 9;
const int CH_TRIGGER       = 13;

// Parameter & Schwellenwerte
const int TARGET_THRESHOLD_CM = 50;  // Auf 50 cm angepasst
const int TRIGGER_REST_POS    = 0;   // Ruheposition des Auslöse-Servos
const int TRIGGER_PULL_POS    = 90;  // Zugposition des Auslöse-Servos

// Pulsweiten-Grenzen (in Mikrosekunden) für die Servos
// PWMax auf 500 (bzw. anpassbar je nach Servo-Typ)
const int PWM_MIN = 150; // Entspricht ca. 0 Grad
const int PWM_MAX = 500; // Entspricht max. Pulsweite (hier auf 500 eingestellt, ggf. bis 2400 erweiterbar)

// Funktionsprototypen
bool scanAndEngage(int targetPanAngle, int targetTiltAngle);
long readDistanceCM();
void fireMechanism();
void moveServo(uint8_t channel, int angle);

void setup() {
  Serial.begin(9600);

  // Ultraschall-Pins konfigurieren
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  // I2C und PCA9685 starten
  Wire.begin();
  pwm.begin();
  pwm.setPWMFreq(50); // Standard-Frequenz für analoge Servos (50Hz)

  // Grundstellung anfahren
  moveServo(CH_TRIGGER, TRIGGER_REST_POS);
  moveServo(CH_RADAR_TILT, 90);    // Waagerecht
  moveServo(CH_LAUNCHER_TILT, 90); // Waagerecht
  moveServo(CH_RADAR_PAN, 90);     // Zentriert
  moveServo(CH_LAUNCHER_PAN, 90);  // Zentriert

  delay(1000);
  Serial.println("System mit PCA9685 bereit. Starte Umgebungsscan...");
}

void loop() {
  // Startet den Raster-Scan bei jedem Durchlauf der Hauptschleife
  if (runRasterScan()) {
    delay(500);  
  }
}

// Funktion für den vollständigen Raster-Durchlauf
bool runRasterScan() {
  int verticalSteps[] = {90, 100, 110, 120, 130, 140, 150, 160}; // Vertikale Zeilen
  int numRows = 8;

  // Schleife durch alle vertikalen Zeilen
  for (int r = 0; r < numRows; r++) {
    int vAngle = verticalSteps[r];

    // Zick-Zack-Logik
    if (r % 2 == 0) {
      for (int hAngle = 0; hAngle <= 180; hAngle += 15) {
        if (scanAndEngage(hAngle, vAngle)) return true; // Sofort abbrechen bei Treffer
      }
    } else {
      for (int hAngle = 180; hAngle >= 0; hAngle -= 15) {
        if (scanAndEngage(hAngle, vAngle)) return true; // Sofort abbrechen bei Treffer
      }
    }
  }
  
  return false;
}

// Scannt einen Punkt und löst bei Zielkontakt den Schuss aus
bool scanAndEngage(int targetPanAngle, int targetTiltAngle) {
  // 1. Radar ausrichten
  moveServo(CH_RADAR_PAN, targetPanAngle);
  moveServo(CH_RADAR_TILT, targetTiltAngle);
  delay(150); // Warten, bis der Servo die Position erreicht hat

  // 2. Distanz messen
  long distance = readDistanceCM();

  // 3. Auswertung: Ist ein Objekt in Reichweite?
  if (distance > 0 && distance <= TARGET_THRESHOLD_CM) {
    Serial.print("ZIEL ERKANNT! Distanz: ");
    Serial.print(distance);
    Serial.print(" cm | Pan-Winkel: ");
    Serial.println(targetPanAngle);

    // Abschussrohr auf Ziel ausrichten
    moveServo(CH_LAUNCHER_PAN, targetPanAngle);
    moveServo(CH_LAUNCHER_TILT, targetTiltAngle);
    delay(500); // Warten auf Ausrichtung des Rohrs

    // Schuss auslösen
    fireMechanism();

    // Abklingzeit nach Abschuss
    delay(3000);
    return true;
  }

  return false;
}

// Funktion zur Ultraschall-Messung
long readDistanceCM() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH, 30000); // Timeout nach 30ms (~5m Max)
  if (duration == 0) return 999;                  // Kein Echo empfangen

  return duration * 0.0343 / 2;
}

// Mechanischer Auslösevorgang
void fireMechanism() {
  Serial.println(">>> FEUER! Schnur wird gezogen... <<<");
  moveServo(CH_TRIGGER, TRIGGER_PULL_POS);
  delay(800);
  moveServo(CH_TRIGGER, TRIGGER_REST_POS);
  delay(300);
}

// Hilfsfunktion zur Steuerung der Servos über den PCA9685 mit konfigurierbarem PWM-Bereich
void moveServo(uint8_t channel, int angle) {
  // Begrenzung des Winkels auf 0 bis 180 Grad
  angle = constrain(angle, 0, 180);
  
  // Mappt den Winkel (0-180) auf den gewünschten Pulsweitenbereich (PWM_MIN bis PWM_MAX)
  int pulse = map(angle, 0, 180, PWM_MIN, PWM_MAX);
  
  // PCA9685 erwartet Ticks (0 bis 4095). Alternativ kann writeMicroseconds genutzt werden, 
  // hier mappen wir direkt auf die Ticks für den 50Hz-Takt.
  // 1 Sekunde = 20000 Mikrosekunden bei 50Hz -> 4096 Ticks = 20ms -> 1 Tick ≈ 4.88µs
  int tick = map(pulse, 0, 20000, 0, 4095);
  
  pwm.setPWM(channel, 0, tick);
}


Test: 
#include <Wire.h>
#include <Adafruit_PWMServoDriver.h>
#include <Servo.h>
// PCA9685 Objekt initialisieren (Standard-I2C-Adresse 0x40)
int PWMMax = 550;
int PWMMin = 100;

Adafruit_PWMServoDriver pwm = Adafruit_PWMServoDriver();
void setup() {
  // put your setup code here, to run once:
pwm.begin();
pwm.setPWMFreq(50);
}

void loop() {
  // put your main code here, to run repeatedly:
pwm.setPWM(0,0,PWMMax);
delay(1000);
pwm.setPWM(0,0,PWMMin);
delay(1000);

}






pwm stuff: 
// Pulsweiten-Grenzen (in Mikrosekunden) für die Servos
// PWMax auf 500 (bzw. anpassbar je nach Servo-Typ)
const int PWM_MIN = 105; // Entspricht ca. 0 Grad
const int PWM_MAX = 565; // Entspricht max. Pulsweite (hier auf 500 eingestellt, ggf. bis 2400 erweiterbar)
