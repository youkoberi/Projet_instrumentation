# Projet_instrumentation
code source de projet
//Le programme commence par Inclusion des bibliothèques nécessaires  au bon fonctionnement des composants. 
#include <Wire.h>                // Communication I2C
#include <LiquidCrystal_I2C.h>  // Pour contrôler l'écran LCD I2C
#include <SPI.h>                // Communication SPI (pour RFID)
#include <MFRC522.h>            // Bibliothèque pour le lecteur RFID
#include <ThreeWire.h>          // Utilisé avec le module RTC DS1302
#include <RtcDS1302.h>          // Bibliothèque RTC DS1302

// Initialisation de l'écran LCD I2C (adresse 0x27, 16 colonnes, 2 lignes)
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Connexion des broches du RTC DS1302 (DAT, CLK, RST)
ThreeWire myWire(2, 3, 4);
RtcDS1302<ThreeWire> Rtc(myWire);

// Définition des broches du lecteur RFID
#define SS_PIN 10
#define RST_PIN 9
MFRC522 rfid(SS_PIN, RST_PIN);

// Définition des broches pour les LEDsssss
#define LED_RED_PIN 7    // LED rouge : utilisateur non reconnu ou en retard
#define LED_BLUE_PIN 6   // LED bleue : utilisateur à l'heure

// Fonction pour retrouver un nom à partir de l'UID d'une carte
String getNameFromUID(String uid) {
  if (uid == "C7C56B7B") return "Khadija";
  else if (uid == "59B9ADD4") return "Yassine";
  else if (uid == "4614C7F7") return "Ibtissam";
  else if (uid == "E7F85619") return "Fatimaezzahra";
  else if (uid == "D9FC4AB3") return "Badr";
  else return "Unknown"; // Carte non reconnue
}
//Dans la partie setup() de notre code, nous avons initialisé tous les composants nécessaires au fonctionnement du système.
void setup() {
// Nous avons d’abord lancé la communication série avec Excel via PLX-DAQ pour pouvoir enregistrer les données,
  Serial.begin(9600);  // Démarre la communication avec PLX-DAQ
//puis envoyé les entêtes des colonnes. 
  Serial.println("CLEARDATA");
  Serial.println("LABEL,Date,Heure,UID,Nom,Statut");
//Ensuite, nous avons initialisé l’écran LCD et activé son rétroéclairage pour l’affichage des messages.
  lcd.init();         // Initialisation de l'écran LCD
  lcd.backlight();    // Allumer le rétroéclairage
// Le module RFID a été préparé pour lire les cartes grâce à la communication SPI
  SPI.begin();        // Initialisation SPI
  rfid.PCD_Init();    // Initialiser le module RFID
//Nous avons aussi mis en route l’horloge temps réel (RTC),
  Rtc.Begin();        // Initialiser le module RTC
  if (Rtc.GetIsWriteProtected()) Rtc.SetIsWriteProtected(false);
  if (!Rtc.GetIsRunning()) Rtc.SetIsRunning(true);
//vérifié son état, et corrigé automatiquement la date si elle était invalide.
  // Initialiser l'heure si elle est invalide (année < 2024)
  if (Rtc.GetDateTime().Year() < 2024) {
    Rtc.SetDateTime(RtcDateTime(__DATE__, __TIME__));
  }
// Les broches des LEDs ont été configurées en sortie et éteintes par défaut.

  // Configuration des LEDs
  pinMode(LED_RED_PIN, OUTPUT);
  pinMode(LED_BLUE_PIN, OUTPUT);
  digitalWrite(LED_RED_PIN, LOW);
  digitalWrite(LED_BLUE_PIN, LOW);
//Enfin, un message de démarrage a été affiché brièvement sur l’écran pour indiquer que le système est prêt.
  lcd.setCursor(0, 0);
  lcd.print("System Ready...");
  delay(2000);
  lcd.clear();
}
//Dans la boucle loop(), le système lit en continu l’heure actuelle à l’aide du module RTC, 
//puis l’affiche en temps réel sur l’écran LCD. 
// l’écran est vidé et le système continue à surveiller une nouvelle carte.
void loop() {
  RtcDateTime now = Rtc.GetDateTime(); // Lire l'heure actuelle

  // Afficher la date sur la 1ère ligne du LCD
  lcd.setCursor(0, 0);
  lcd.print("Date: ");
  print2digits(now.Day());
  lcd.print('/');
  print2digits(now.Month());
  lcd.print('/');
  lcd.print(now.Year());

  // Afficher l'heure sur la 2e ligne du LCD
  lcd.setCursor(0, 1);
  lcd.print("Time: ");
  print2digits(now.Hour());
  lcd.print(':');
  print2digits(now.Minute());
  lcd.print(':');
  print2digits(now.Second());
//Il surveille ensuite la présence d’une carte RFID.
  // Détection d'une carte RFID
  if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Card detected");
// Lorsqu’une carte est détectée, le programme lit son UID, vérifie si elle est reconnue,
    // Lire l'UID de la carte scannée
    String uid = "";
    for (byte i = 0; i < rfid.uid.size; i++) {
      uid += String(rfid.uid.uidByte[i] < 0x10 ? "0" : "");
      uid += String(rfid.uid.uidByte[i], HEX);
    }
    uid.toUpperCase(); // Mettre l'UID en majuscule
 //et récupère le nom associé.
    // Obtenir le nom associé à l'UID
    String name = getNameFromUID(uid);
    int hour = now.Hour();
    int minute = now.Minute();
    String status;

    lcd.clear();
    lcd.setCursor(0, 0);

    if (name == "Unknown") {
      // Si la carte est inconnue
      lcd.print("Unknown card");
      status = "Denied";
      digitalWrite(LED_BLUE_PIN, LOW);
      digitalWrite(LED_RED_PIN, HIGH);
      delay(1000);
      digitalWrite(LED_RED_PIN, LOW);
    } else {
      // Carte reconnue
      // Selon l’heure de passage, il affiche sur le LCD un message de bienvenue ou de retard, et allume la LED bleue ou rouge en conséquence.
      if (hour == 14 && minute <= 8) {
        status = "On time";
        lcd.print("Welcome " + name);
        lcd.setCursor(0, 1);
        lcd.print("You are on time");
        digitalWrite(LED_BLUE_PIN, HIGH);
        digitalWrite(LED_RED_PIN, LOW);
        delay(1000);
        digitalWrite(LED_BLUE_PIN, LOW);
      } else {
        status = "Late";
        lcd.print("Hi " + name);
        lcd.setCursor(0, 1);
        lcd.print("You are late!");
        digitalWrite(LED_BLUE_PIN, LOW);
        digitalWrite(LED_RED_PIN, HIGH);
        delay(1000);
        digitalWrite(LED_RED_PIN, LOW);
      }
    }
// Les informations (date, heure, UID, nom et statut) sont ensuite envoyées automatiquement vers Excel via PLX-DAQ. 
//Enfin, après une courte pause,l'écran s'efface afin de pouvoir répéter le processus.
    // Envoi des données vers Excel (PLX-DAQ)
    Serial.print("DATA,");
    Serial.print(now.Day()); Serial.print("/");
    Serial.print(now.Month()); Serial.print("/");
    Serial.print(now.Year()); Serial.print(",");

    printSerial2digits(now.Hour()); Serial.print(":");
    printSerial2digits(now.Minute()); Serial.print(":");
    printSerial2digits(now.Second()); Serial.print(",");

    Serial.print(uid); Serial.print(",");
    Serial.print(name); Serial.print(",");
    Serial.println(status);
//effacement de l'écran 
    delay(4000); // Attente avant 
    lcd.clear();
  }

  delay(1000); // Pause entre chaque boucle
}

// Afficher un chiffre à deux chiffres sur le LCD
void print2digits(int num) {
  if (num < 10) lcd.print('0');
  lcd.print(num);
}

// Afficher un chiffre à deux chiffres dans le Serial (Excel)
void printSerial2digits(int num) {
  if (num < 10) Serial.print('0');
  Serial.print(num);
}
