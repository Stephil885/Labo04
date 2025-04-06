
#include <LCD_I2C.h>
#include <HCSR04.h>
#include <AccelStepper.h>

// LCD
const int LCD_ADDRESS = 0x27;
const int LCD_COLUMNS = 16;
const int LCD_ROWS = 2;
const int LCD_LINE_0 = 0;
const int LCD_LINE_1 = 1;
const int LCD_CURSOR_COL_0 = 0;

// Moteur
const int MOTOR_INTERFACE_TYPE = 4;
const int IN_1 = 31;
const int IN_2 = 33;
const int IN_3 = 35;
const int IN_4 = 37;

// Capteur ultrason
const int TRIGGER_PIN = 9;
const int ECHO_PIN = 10;

// Délais et intervalles
const unsigned long INTERVAL_DISTANCE_CHECK = 50;
const unsigned long INTERVAL_LCD_UPDATE = 100;
const unsigned long INTERVAL_SERIAL_PRINT = 100;
const unsigned long DELAY_LCD_START = 2000;

// Distance & angle
const int DIST_MIN = 30;
const int DIST_MAX = 60;
const int DEG_MIN = 10;
const int DEG_MAX = 170;
const int MOTOR_STEP_MIN = 57;
const int MOTOR_STEP_MAX = 967;

// Stepper
const long STEPPER_MAX_SPEED = 500;
const long STEPPER_ACCELERATION = 100;

// Infos d'affichage
const char* IDENTIFIANT = "2403493";
const char* NOM_LABO = "Labo 4b";

// Initialisation des objets
LCD_I2C lcd(LCD_ADDRESS, LCD_COLUMNS, LCD_ROWS);
HCSR04 hc(TRIGGER_PIN, ECHO_PIN);
AccelStepper myStepper(MOTOR_INTERFACE_TYPE, IN_1, IN_3, IN_2, IN_4);

// Gestion du temps
unsigned long lastDistanceCheck = 0;
unsigned long lastSerialPrint = 0;
unsigned long lastLcdUpdate = 0;

// État actuel
enum Etat {
  TROP_PRES,
  TROP_LOIN,
  DANS_ZONE
};
Etat etatActuel = TROP_LOIN;

// Variables principales
float distance = 0;
int angle = 0;
int lastAngle = -1;

void allumage() {
  lcd.begin();
  lcd.backlight();
  lcd.setCursor(LCD_CURSOR_COL_0, LCD_LINE_0);
  lcd.print(IDENTIFIANT);
  lcd.setCursor(LCD_CURSOR_COL_0, LCD_LINE_1);
  lcd.print(NOM_LABO);
  delay(DELAY_LCD_START);
  lcd.clear();
}

void setup() {
  Serial.begin(115200);
  myStepper.setMaxSpeed(STEPPER_MAX_SPEED);
  myStepper.setAcceleration(STEPPER_ACCELERATION);
  allumage();
}



void updateEtatEtAngle() {
  if (distance < DIST_MIN) {
    etatActuel = TROP_PRES;
    angle = DEG_MIN;
  } else if (distance > DIST_MAX) {
    etatActuel = TROP_LOIN;
    angle = DEG_MAX;
  } else {
    etatActuel = DANS_ZONE;
    angle = map(distance, DIST_MIN, DIST_MAX, DEG_MIN, DEG_MAX);
  }
}

void afficherLCD() {
  lcd.clear();
  lcd.setCursor(LCD_CURSOR_COL_0, LCD_LINE_0);
  lcd.print("Dist:");
  lcd.print((int)distance);
  lcd.print("cm");
  lcd.setCursor(LCD_CURSOR_COL_0, LCD_LINE_1);
  if (etatActuel == TROP_PRES) {
    lcd.print("obj :Trop pres");
  } else if (etatActuel == TROP_LOIN) {
    lcd.print("obj :Trop loin");
  } else {
    lcd.print("obj:");
    lcd.print(angle);
    lcd.print("deg");
  }
}

void loop() {
  unsigned long now = millis();

  if (now - lastDistanceCheck >= INTERVAL_DISTANCE_CHECK) {
    distance = hc.dist();
    updateEtatEtAngle();
    lastDistanceCheck = now;
  }

  if (now - lastLcdUpdate >= INTERVAL_LCD_UPDATE) {
    afficherLCD();
    lastLcdUpdate = now;
  }

  if (now - lastSerialPrint >= INTERVAL_SERIAL_PRINT) {
    Serial.print("etd:");
    Serial.print(IDENTIFIANT);
    Serial.print(",dist:");
    Serial.print((int)distance);
    Serial.print(",deg:");
    Serial.println(angle);
    lastSerialPrint = now;
  }

  if (etatActuel == DANS_ZONE) {
    if (angle != lastAngle) {
      int steps = map(angle, DEG_MIN, DEG_MAX, MOTOR_STEP_MIN, MOTOR_STEP_MAX);
      myStepper.enableOutputs();
      myStepper.moveTo(steps);
      lastAngle = angle;
    }
    myStepper.run();
  } else {
    myStepper.disableOutputs();
  }
}
