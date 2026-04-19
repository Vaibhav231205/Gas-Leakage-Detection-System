#define BLYNK_TEMPLATE_ID "TMPL3lK3YUf5x"
#define BLYNK_TEMPLATE_NAME "Gas Detector"
#define BLYNK_AUTH_TOKEN "mN6nVxPT97meKIcBHHnUX5k0LL0KZS8Q"

#define BLYNK_PRINT Serial

#include <WiFi.h>
#include <BlynkSimpleEsp32.h>

char ssid[] = "Vaibhav";
char pass[] = "12345678";

#define MQ2_PIN 34
#define LED_PIN 2
#define BUZZER_PIN 4

int gasValue = 0;
int gasPercent = 0;

bool notificationSent = false;

// 🚨 Emergency control
bool alarmActive = false;
unsigned long alarmStartTime = 0;
unsigned long lastBlinkTime = 0;
bool blinkState = false;

void setup() {
  Serial.begin(9600);

  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  analogSetAttenuation(ADC_11db);  // Better ADC range

  Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);
}

void loop() {
  Blynk.run();

  // Read sensor
  gasValue = analogRead(MQ2_PIN);
  gasPercent = map(gasValue, 0, 4095, 0, 100);

  // Send to app
  Blynk.virtualWrite(V0, gasPercent);

  Serial.print("Gas %: ");
  Serial.println(gasPercent);

  // 🚨 Trigger alarm
  if (gasPercent >= 70 && !alarmActive) {
    alarmActive = true;
    alarmStartTime = millis();

    if (!notificationSent) {
      Blynk.logEvent("gas_alert", "⚠️ Gas Leakage Detected!");
      notificationSent = true;
    }
  }

  // 🚨 Alarm running (non-blocking)
  if (alarmActive) {
    // Fast blinking (every 100ms)
    if (millis() - lastBlinkTime >= 100) {
      lastBlinkTime = millis();
      blinkState = !blinkState;

      digitalWrite(LED_PIN, blinkState);
      digitalWrite(BUZZER_PIN, blinkState);
    }

    // Stop after 5 seconds
    if (millis() - alarmStartTime >= 5000) {
      alarmActive = false;
      digitalWrite(LED_PIN, LOW);
      digitalWrite(BUZZER_PIN, LOW);
    }
  }

  // Reset when gas is normal
  if (gasPercent < 70) {
    notificationSent = false;
  }

  delay(200);
}
