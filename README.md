# esp32-IRtxIRrx-Test
Testen der IR-Sende und Empfangsmodule
Das folgende Sketch sendet IR-Signale über das AZ-Delivery KY-005 Modul und das daneben platzierte KY-022 IR Receiver Modul empfängt die Signale.  
Dadurch habe ich bemerkt, das die Signale nicht sauber gesendet werden.
**Ich sende für AI Toggle: 0x6A49, empfange aber 0x492B**
Auch bei den Signalen für Rechts und Links kommen die Signale nicht sauber an.
In den Pausen nach den Signalen kann ich mit der original Fernbedienung testen und sehe, das mein Empfänger die Signale richtig empfängt. Demzufolge funktioniert das KY-005 IR Infrared Emmission Sensor Modul nicht korrekt.


```cpp
#include <IRremote.hpp>

#define IR_SEND_PIN    4   // GPIO für IR-Senden (KY-005)
#define IR_RECEIVE_PIN 15  // GPIO für IR-Empfang (KY-022)

enum State {
  IDLE,
  AI_TOGGLE,
  WAIT_AFTER_AI,
  PTZ_RIGHT,
  WAIT_AFTER_RIGHT,
  PTZ_LEFT,
  WAIT_AFTER_LEFT
};

State currentState = IDLE;
unsigned long stateStartTime = 0;
unsigned long lastSendTime = 0;
int sendCount = 0;

void setup() {
  Serial.begin(115200);
  delay(500);
  IrSender.begin(IR_SEND_PIN, ENABLE_LED_FEEDBACK, USE_DEFAULT_FEEDBACK_LED_PIN);
  IrReceiver.begin(IR_RECEIVE_PIN, ENABLE_LED_FEEDBACK);
  Serial.println("Starte IR-Sendesequenz mit Pausen für manuelle Tests.");
}

void loop() {
  // Empfang immer möglich
  if (IrReceiver.decode()) {
    Serial.print("Empfangen: 0x");
    Serial.println(IrReceiver.decodedIRData.decodedRawData, HEX);
    Serial.print("Protokoll: ");
    Serial.println(getProtocolString(IrReceiver.decodedIRData.protocol));
    IrReceiver.resume();
  }

  unsigned long now = millis();

  switch (currentState) {
    case IDLE:
      currentState = AI_TOGGLE;
      stateStartTime = now;
      sendCount = 0;
      Serial.println("== AI TOGGLE ==");
      break;

    case AI_TOGGLE:
      if (sendCount < 5 && now - lastSendTime > 150) {
        IrSender.sendSony(0x6A49, 15);
        lastSendTime = now;
        sendCount++;
      }
      if (sendCount >= 5 && now - lastSendTime > 500) {
        currentState = WAIT_AFTER_AI;
        stateStartTime = now;
        Serial.println("== Warten nach AI (10s) für Tests ==");
      }
      break;

    case WAIT_AFTER_AI:
      if (now - stateStartTime >= 10000) {
        currentState = PTZ_RIGHT;
        stateStartTime = now;
        sendCount = 0;
        Serial.println("== PTZ RECHTS ==");
      }
      break;

    case PTZ_RIGHT:
      if (now - lastSendTime > 100 && now - stateStartTime < 3000) {
        IrSender.sendSony(0x51D1C, 15);
        lastSendTime = now;
      }
      if (now - stateStartTime >= 3000) {
        currentState = WAIT_AFTER_RIGHT;
        stateStartTime = now;
        Serial.println("== Warten nach Rechts (10s) für Tests ==");
      }
      break;

    case WAIT_AFTER_RIGHT:
      if (now - stateStartTime >= 10000) {
        currentState = PTZ_LEFT;
        stateStartTime = now;
        lastSendTime = 0;
        Serial.println("== PTZ LINKS ==");
      }
      break;

    case PTZ_LEFT:
      if (now - lastSendTime > 100 && now - stateStartTime < 3000) {
        IrSender.sendSony(0x51D1D, 15);
        lastSendTime = now;
      }
      if (now - stateStartTime >= 3000) {
        currentState = WAIT_AFTER_LEFT;
        stateStartTime = now;
        Serial.println("== Warten nach Links (10s) für Tests ==");
      }
      break;

    case WAIT_AFTER_LEFT:
      if (now - stateStartTime >= 10000) {
        currentState = IDLE;
        stateStartTime = now;
        Serial.println("== Zyklus abgeschlossen – Neuer Durchlauf ==");
      }
      break;
  }
}

```
