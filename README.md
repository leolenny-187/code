#include <Wire.h>
#include <Adafruit_PWMServoDriver.h>

// PCA9685 Objekt initialisieren (Standard-I2C-Adresse 0x40)
Adafruit_PWMServoDriver pwm = Adafruit_PWMServoDriver();

// Pin-Definitionen Ultraschallsensor (bleiben am Arduino)
const int TRIG_PIN = 2;
const int ECHO_PIN = 3;

// Kanal-Definitionen auf dem PCA9685 Servotreiber
const int CH_RADAR_PAN     = 0;
const int CH_RADAR_TILT    = 3;
const int CH_LAUNCHER_PAN  = 4;
const int CH_LAUNCHER_TILT = 8;
const int CH_TRIGGER       = 12;

// Parameter & Schwellenwerte
const int TARGET_THRESHOLD_CM = 50;  // Auf 50 cm angepasst
const int TRIGGER_REST_POS    = 105;   // Ruheposition des Auslöse-Servos
const int TRIGGER_PULL_POS    = 575;  // Zugposition des Auslöse-Servos

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
  moveServo(CH_RADAR_TILT, 100);    // Waagerecht
  moveServo(CH_LAUNCHER_TILT, 100); // Waagerecht
  moveServo(CH_RADAR_PAN, 100);     // Zentriert
  moveServo(CH_LAUNCHER_PAN, 100);  // Zentriert

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
  int verticalSteps[] = {100, 110, 120, 130, 140, 150, 160, 170}; // Vertikale Zeilen
  int numRows = 8;

  // Schleife durch alle vertikalen Zeilen
  for (int r = 0; r < numRows; r++) {
    int vAngle = verticalSteps[r];

    // Zick-Zack-Logik
    if (r % 2 == 0) {
      for (int hAngle = 100; hAngle <= 280; hAngle += 15) {
        if (scanAndEngage(hAngle, vAngle)) return true; // Sofort abbrechen bei Treffer
      }
    } else {
      for (int hAngle = 280; hAngle >= 100; hAngle -= 15) {
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
  if (duration == 0) return 999;               // Kein Echo empfangen

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

// HIER WAR DER FEHLER: Die fehlende moveServo-Funktion wurde ergänzt
void moveServo(uint8_t channel, int angle) {
  // Wandelt den Winkel (0 bis 180 Grad) in PCA9685 PWM-Pulse um 
  // (Standard-Werte: 150 bis 600 Puls-Ticks für typische Servos)
  int pulse = map(angle, 0, 180, 150, 600);
  pwm.setPWM(channel, 0, pulse);
}
