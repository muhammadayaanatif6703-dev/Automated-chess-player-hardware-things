 HARDWARE TEST / DIAGNOSTIC SKETCH  (Arduino Nano)
  =====================================================================
  Purpose: bypass ALL chess logic and just directly exercise every
  actuator one at a time, with serial prints before/after each action,
  so you can see exactly which part is working and which isn't.

  HOW TO USE:
    1. Upload this sketch.
    2. Open Serial Monitor, set baud to 115200, set line ending to
       "Newline" (important - if it's set to "No line ending" your
       commands won't be recognized).
    3. Power the 7.2V supply to the motor drivers AND make sure the
       Arduino has a stable 5V (see POWER CHECKLIST below before you
       even touch this code).
    4. Type single-letter commands (listed below) and press Enter.
       Watch both the Serial Monitor text AND the physical hardware.

  COMMANDS:
    1  = pulse stepper 200 steps forward (watch + listen for movement
         or buzzing)
    2  = pulse stepper 200 steps backward
    3  = sweep servo 0 -> 180 -> 0
    4  = run lift motor "up" direction for 1 second
    5  = run lift motor "down" direction for 1 second
    6  = run claw motor "close" direction for 1 second
    7  = run claw motor "open" direction for 1 second
    8  = print live encoder count for 5 seconds (manually move the
         lift by hand during this time and watch if the number changes)
    9  = print raw pin states of everything (diagnostic snapshot)

  =====================================================================
  POWER CHECKLIST -- check this BEFORE assuming it's a code problem.
  This is the single most common reason "everything is on but nothing
  moves":
  =====================================================================
  [ ] Is the Arduino's GND connected to the SAME ground as the 7.2V
      battery's GND / the L298N's GND / the DRV8825's GND?
      --> If the grounds are not common, the Arduino's digital HIGH
          signal has no shared reference voltage with the driver
          boards, and NOTHING will respond even though everything
          looks "powered."  This is the #1 cause of this exact
          symptom (powered, no errors, total silence from motors).
  [ ] Is the L298N's own 5V regulator jumper/pin set correctly for
      a 7.2V input? Many L298N boards have a small jumper labeled
      "5V-EN" - if 7.2V is going into the 12V screw terminal, that
      jumper should stay ON so the board makes its own logic 5V
      (don't also feed 5V into that pin externally at the same time).
  [ ] Does the DRV8825 have its own logic power (3.3-5V) on the VDD
      pin? Some DRV8825 modules need logic power supplied separately
      from motor power (VMOT). If your DRV8825 isn't getting logic
      power, the chip is fully dead even if VMOT (7.2V) is present.
  [ ] Is the DRV8825 missing a motor connected, or VMOT capacitor
      installed backwards? these chips often refuse to do anything
      (no heat, no movement, totally inert) if wired wrong, as a
      protection feature.
  [ ] Multimeter check: with everything powered, is there really
      7.2V present at the L298N's 12V terminal and the DRV8825's
      VMOT pin? (battery connectors / switches sometimes look
      connected but aren't seated.)
  =====================================================================
*/

const uint8_t PIN_STEP        = A2;
const uint8_t PIN_DIR         = A3;

const uint8_t PIN_SERVO       = 3;

const uint8_t PIN_LIFT_IN1    = 8;
const uint8_t PIN_LIFT_IN2    = 7;
const uint8_t PIN_CLAW_IN3    = 6;
const uint8_t PIN_CLAW_IN4    = 5;

const uint8_t PIN_ENC_A       = 11;
const uint8_t PIN_ENC_B       = 12;

#include <Servo.h>
Servo testServo;

volatile long encoderCount = 0;
volatile int8_t lastEncState = 0;

void setup() {
  Serial.begin(115200);
  delay(200);

  pinMode(PIN_STEP, OUTPUT);
  pinMode(PIN_DIR, OUTPUT);
  digitalWrite(PIN_STEP, LOW);
  digitalWrite(PIN_DIR, LOW);

  pinMode(PIN_LIFT_IN1, OUTPUT);
  pinMode(PIN_LIFT_IN2, OUTPUT);
  pinMode(PIN_CLAW_IN3, OUTPUT);
  pinMode(PIN_CLAW_IN4, OUTPUT);
  digitalWrite(PIN_LIFT_IN1, LOW);
  digitalWrite(PIN_LIFT_IN2, LOW);
  digitalWrite(PIN_CLAW_IN3, LOW);
  digitalWrite(PIN_CLAW_IN4, LOW);

  pinMode(PIN_ENC_A, INPUT_PULLUP);
  pinMode(PIN_ENC_B, INPUT_PULLUP);
  PCICR  |= (1 << PCIE0);
  PCMSK0 |= (1 << PCINT3) | (1 << PCINT4);
  lastEncState = (digitalRead(PIN_ENC_A) << 1) | digitalRead(PIN_ENC_B);

  testServo.attach(PIN_SERVO);
  testServo.write(90);

  Serial.println(F("==================================="));
  Serial.println(F("HARDWARE TEST SKETCH READY"));
  Serial.println(F("==================================="));
  Serial.println(F("1 = stepper forward 200 steps"));
  Serial.println(F("2 = stepper backward 200 steps"));
  Serial.println(F("3 = servo sweep 0->180->0"));
  Serial.println(F("4 = lift motor UP direction, 1 sec"));
  Serial.println(F("5 = lift motor DOWN direction, 1 sec"));
  Serial.println(F("6 = claw motor CLOSE direction, 1 sec"));
  Serial.println(F("7 = claw motor OPEN direction, 1 sec"));
  Serial.println(F("8 = print live encoder count, 5 sec"));
  Serial.println(F("9 = print raw pin diagnostic snapshot"));
  Serial.println(F("Type a number and press Enter."));
}

void loop() {
  if (Serial.available() > 0) {
    String line = Serial.readStringUntil('\n');
    line.trim();
    if (line.length() == 0) return;

    char c = line[0];
    switch (c) {
      case '1': testStepperForward(); break;
      case '2': testStepperBackward(); break;
      case '3': testServoSweep(); break;
      case '4': testLiftDirection(true); break;
      case '5': testLiftDirection(false); break;
      case '6': testClawDirection(true); break;
      case '7': testClawDirection(false); break;
      case '8': testEncoderLive(); break;
      case '9': printDiagnosticSnapshot(); break;
      default:
        Serial.print(F("Unrecognized: "));
        Serial.println(line);
    }
  }
}

void testStepperForward() {
  Serial.println(F("--- STEPPER FORWARD 200 steps ---"));
  Serial.println(F("Watch/listen now..."));
  digitalWrite(PIN_DIR, HIGH);
  for (int i = 0; i < 200; i++) {
    digitalWrite(PIN_STEP, HIGH);
    delayMicroseconds(800);
    digitalWrite(PIN_STEP, LOW);
    delayMicroseconds(800);
  }
  Serial.println(F("Done. Did the shaft turn, or did it just buzz/vibrate in place?"));
}

void testStepperBackward() {
  Serial.println(F("--- STEPPER BACKWARD 200 steps ---"));
  digitalWrite(PIN_DIR, LOW);
  for (int i = 0; i < 200; i++) {
    digitalWrite(PIN_STEP, HIGH);
    delayMicroseconds(800);
    digitalWrite(PIN_STEP, LOW);
    delayMicroseconds(800);
  }
  Serial.println(F("Done."));
}

void testServoSweep() {
  Serial.println(F("--- SERVO SWEEP 0 -> 180 -> 0 ---"));
  for (int a = 0; a <= 180; a += 2) {
    testServo.write(a);
    delay(15);
  }
  for (int a = 180; a >= 0; a -= 2) {
    testServo.write(a);
    delay(15);
  }
  testServo.write(90);
  Serial.println(F("Done. Did the servo physically move?"));
}

void testLiftDirection(bool up) {
  Serial.print(F("--- LIFT MOTOR "));
  Serial.print(up ? F("UP") : F("DOWN"));
  Serial.println(F(" direction, 1 second ---"));
  if (up) {
    digitalWrite(PIN_LIFT_IN1, HIGH);
    digitalWrite(PIN_LIFT_IN2, LOW);
  } else {
    digitalWrite(PIN_LIFT_IN1, LOW);
    digitalWrite(PIN_LIFT_IN2, HIGH);
  }
  delay(1000);
  digitalWrite(PIN_LIFT_IN1, LOW);
  digitalWrite(PIN_LIFT_IN2, LOW);
  Serial.println(F("Done. Did the lift motor spin?"));
}

void testClawDirection(bool close) {
  Serial.print(F("--- CLAW MOTOR "));
  Serial.print(close ? F("CLOSE") : F("OPEN"));
  Serial.println(F(" direction, 1 second ---"));
  if (close) {
    digitalWrite(PIN_CLAW_IN3, HIGH);
    digitalWrite(PIN_CLAW_IN4, LOW);
  } else {
    digitalWrite(PIN_CLAW_IN3, LOW);
    digitalWrite(PIN_CLAW_IN4, HIGH);
  }
  delay(1000);
  digitalWrite(PIN_CLAW_IN3, LOW);
  digitalWrite(PIN_CLAW_IN4, LOW);
  Serial.println(F("Done. Did the claw motor spin?"));
}

void testEncoderLive() {
  Serial.println(F("--- LIVE ENCODER COUNT for 5 seconds ---"));
  Serial.println(F("Manually rotate/move the lift mechanism by hand now."));
  unsigned long start = millis();
  long lastPrinted = -999999;
  while (millis() - start < 5000) {
    if (encoderCount != lastPrinted) {
      Serial.print(F("encoderCount = "));
      Serial.println(encoderCount);
      lastPrinted = encoderCount;
    }
  }
  Serial.println(F("Done. If the number never changed while you moved it by"));
  Serial.println(F("hand, the encoder wiring/power is the problem, not the code."));
}

void printDiagnosticSnapshot() {
  Serial.println(F("--- RAW PIN SNAPSHOT ---"));
  Serial.print(F("PIN_STEP (A2) state: "));      Serial.println(digitalRead(PIN_STEP));
  Serial.print(F("PIN_DIR  (A3) state: "));       Serial.println(digitalRead(PIN_DIR));
  Serial.print(F("PIN_LIFT_IN1 (D8): "));         Serial.println(digitalRead(PIN_LIFT_IN1));
  Serial.print(F("PIN_LIFT_IN2 (D7): "));         Serial.println(digitalRead(PIN_LIFT_IN2));
  Serial.print(F("PIN_CLAW_IN3 (D6): "));         Serial.println(digitalRead(PIN_CLAW_IN3));
  Serial.print(F("PIN_CLAW_IN4 (D5): "));         Serial.println(digitalRead(PIN_CLAW_IN4));
  Serial.print(F("PIN_ENC_A (D11) raw: "));       Serial.println(digitalRead(PIN_ENC_A));
  Serial.print(F("PIN_ENC_B (D12) raw: "));       Serial.println(digitalRead(PIN_ENC_B));
  Serial.print(F("encoderCount: "));              Serial.println(encoderCount);
  Serial.println(F("------------------------"));
}

ISR(PCINT0_vect) {
  int a = digitalRead(PIN_ENC_A);
  int b = digitalRead(PIN_ENC_B);
  int state = (a << 1) | b;
  static const int8_t qtable[16] = {
    0, -1, 1, 0,
    1, 0, 0, -1,
    -1, 0, 0, 1,
    0, 1, -1, 0
  };
  int idx = (lastEncState << 2) | state;
  encoderCount += qtable[idx & 0x0F];
  lastEncState = state;
}
