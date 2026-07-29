Biometric Monitor & Display:
This project involves the creation of a system that can monitor and display biometrics using a Arduino UNO, LCD display, and a pulse sensor. The Arduino Integrated Development (IDE) was used to write the program that would be uploaded onto the Arduino board. This code enables the Arduino board to read from the sensors and display the results from those sensors. 

| **Engineer** | **School** | **Area of Interest** | **Grade** |
|:--:|:--:|:--:|:--:|
| Nakshatra K | Harry Ainlay | Aeronautical Engineering | Incoming Senior

**Replace the BlueStamp logo below with an image of yourself and your completed project. Follow the guide [here](https://tomcam.github.io/least-github-pages/adding-images-github-pages-site.html) if you need help.**

![Headstone Image](logo.svg)
  
# Final Milestone

**Don't forget to replace the text below with the embedding for your milestone video. Go to Youtube, click Share -> Embed, and copy and paste the code to replace what's below.**

<iframe width="560" height="315" src="https://www.youtube.com/embed/F7M7imOVGug" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

For your final milestone, explain the outcome of your project. Key details to include are:
- What you've accomplished since your previous milestone
- What your biggest challenges and triumphs were at BSE
- A summary of key topics you learned about
- What you hope to learn in the future after everything you've learned at BSE



# Second Milestone

**Don't forget to replace the text below with the embedding for your milestone video. Go to Youtube, click Share -> Embed, and copy and paste the code to replace what's below.**

<iframe width="560" height="315" src="https://www.youtube.com/embed/y3VAmNlER5Y" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

For your second milestone, explain what you've worked on since your previous milestone. You can highlight:
- Technical details of what you've accomplished and how they contribute to the final goal
- What has been surprising about the project so far
- Previous challenges you faced that you overcame
- What needs to be completed before your final milestone 

# First Milestone

<iframe width="560" height="315" src="https://www.youtube.com/embed/1PXAsAyX8vo?si=GI4OxZExRfveXz39" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

My project is an Arduino-based device that measures the user’s pulse and displays the heart rate on an LCD screen. The main components of the project are the Arduino Uno, the pulse sensor, the LCD1602 screen, jumper wires, and a breadboard. The pulse sensor detects the blood flow from the user’s fingertip and displays the calculated beats per minute (BPM) on the screen using the Arduino.

The main progress that has been made includes assembling the circuit with the LCD screen in 4-bit mode, uploading the program to the Arduino, and displaying messages on the screen asking the user to place their finger on the sensor and messages that display the BPM of their heart rate. These messages are also displayed when the Arduino is performing signal filtering to find the BPM of the user.

One of the major challenges for the project was preventing the sensor from detecting random signals when there was no finger on the sensor. The BPM readings would also change if I moved my finger. Through these challenges, I learnt about signal filtering and how to properly create a program for a pulse sensor to detect BPM accurately. In addition, the LCD screen was found to be slightly dim. This is a hardware issue because the contrast of the screen can only be adjusted with the hardware of the LCD screen, not the Arduino code.

The next goal for my project will be to further test and improve the accuracy of the BPM measurements from the sensor. Another goal will be to replace the current LCD screen with an I2C screen that takes up less pins on the Arduino. A I2C LCD module may also provide a clearer or brighter display, although brightness will depend on the backlight of the specific module. I also plan to organize and secure the wiring and eventually create an enclosure for the completed biometric monitor.

# Schematics 

<img width="1310" height="848" alt="Screenshot 2026-07-22 083251" src="https://github.com/user-attachments/assets/624d70d8-8cd0-460f-aa04-744e6845cd13" />




# Code


<div style="
  height: 350px;
  overflow-y: scroll;
  overflow-x: hidden;
  background-color: #1e1e1e;
  color: white;
  padding: 15px;
  border-radius: 8px;
">
  <pre style="
    margin: 0;
    white-space: pre-wrap;
    overflow-wrap: break-word;
    word-break: break-word;
  "><code>
/*
  Heart Rate Monitor with AUTO-CALIBRATING threshold
  ----------------------------------------------------
  Hardware:
    - Pulse Sensor  -> Signal pin to A0, + to 5V, - to GND
    - I2C LCD (16x2)-> SDA to A4, SCL to A5, VCC to 5V, GND to GND

  Library needed:
    - LiquidCrystal_I2C

  Fixes in this version:
    - LCD lines are now always padded to exactly 16 characters,
      so leftover characters from a previous longer message can't
      "stick" on screen (this was causing a stray "d" to appear).
    - Beat intervals that are wildly different from the recent
      average (e.g. roughly double or half) are now rejected as
      noise/double-triggers instead of being averaged in, which
      was causing the BPM to jump between two values like 74/141.
    - Hysteresis (two thresholds) added to reduce false re-triggers
      from signal noise near the threshold line.
*/

#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2); // change to 0x3F if screen is blank

const int PULSE_PIN = A0;

// ---- Dynamic threshold state ----
int signalMin = 1023;
int signalMax = 0;
int thresholdHigh = 512; // must rise above this to count as a beat
int thresholdLow = 480;  // must fall below this before it can trigger again

// Where between min/max to place the two thresholds (0.0 - 1.0)
const float THRESH_HIGH_RATIO = 0.55;
const float THRESH_LOW_RATIO = 0.45;

// ---- Calibration ----
const unsigned long CALIBRATION_TIME = 4000;
unsigned long calibrationStart = 0;
bool calibrated = false;

// ---- Beat detection ----
bool aboveThreshold = false;
unsigned long lastBeatTime = 0;
const unsigned long REFRACTORY_PERIOD = 350; // ms, max ~170bpm

// ---- Averaging buffer ----
const int BUFFER_SIZE = 8;
unsigned long intervalBuffer[BUFFER_SIZE];
int bufferIndex = 0;
int bufferCount = 0;
unsigned long runningAvgInterval = 0; // ms, used for outlier rejection

// ---- Display timing ----
unsigned long lastDisplayUpdate = 0;
const unsigned long DISPLAY_INTERVAL = 1000;

// ---- Signal-lost detection ----
unsigned long lastSignalTime = 0;
const unsigned long SIGNAL_TIMEOUT = 3000;

// ---- Slow ongoing recalibration ----
unsigned long lastRangeReset = 0;
const unsigned long RANGE_RESET_INTERVAL = 8000;

void setup() {
  Serial.begin(9600);

  lcd.init();
  lcd.backlight();
  printLine(0, "Calibrating...");
  printLine(1, "Keep finger on");

  calibrationStart = millis();
  lastRangeReset = millis();
  lastSignalTime = millis();
}

void loop() {
  int sensorValue = analogRead(PULSE_PIN);
  unsigned long now = millis();

  if (sensorValue < signalMin) signalMin = sensorValue;
  if (sensorValue > signalMax) signalMax = sensorValue;

  if (!calibrated) {
    if (now - calibrationStart >= CALIBRATION_TIME) {
      finishCalibration();
    } else {
      return;
    }
  }

  if (now - lastRangeReset >= RANGE_RESET_INTERVAL) {
    if ((signalMax - signalMin) > 20) {
      updateThresholds();
    }
    signalMin = sensorValue;
    signalMax = sensorValue;
    lastRangeReset = now;
  }

  // ---- Beat detection with hysteresis ----
  if (sensorValue > thresholdHigh && !aboveThreshold) {
    if (now - lastBeatTime > REFRACTORY_PERIOD) {
      unsigned long interval = now - lastBeatTime;

      if (lastBeatTime != 0 && interval > 0) {
        handleNewInterval(interval);
      }

      lastBeatTime = now;
      lastSignalTime = now;
    }
    aboveThreshold = true;
  } else if (sensorValue < thresholdLow) {
    aboveThreshold = false;
  }

  if (now - lastDisplayUpdate >= DISPLAY_INTERVAL) {
    lastDisplayUpdate = now;
    updateDisplay(now);
  }

  Serial.print("Raw:");
  Serial.print(sensorValue);
  Serial.print(" Hi:");
  Serial.print(thresholdHigh);
  Serial.print(" Lo:");
  Serial.println(thresholdLow);
}

void updateThresholds() {
  thresholdHigh = signalMin + (signalMax - signalMin) * THRESH_HIGH_RATIO;
  thresholdLow = signalMin + (signalMax - signalMin) * THRESH_LOW_RATIO;
}

void finishCalibration() {
  if (signalMax - signalMin < 10) {
    lcd.clear();
    printLine(0, "No signal seen");
    printLine(1, "Check finger/wire");
    delay(2000);
    signalMin = 1023;
    signalMax = 0;
    calibrationStart = millis();
    printLine(0, "Calibrating...");
    printLine(1, "Keep finger on");
    return;
  }

  updateThresholds();
  calibrated = true;
  lastRangeReset = millis();
  lcd.clear();
}

// Reject beat intervals that are wildly different from the recent
// average (likely a missed beat or a noise double-trigger) instead
// of letting them corrupt the displayed BPM.
void handleNewInterval(unsigned long interval) {
  if (bufferCount == 0) {
    // First interval - accept it to get things started
    addInterval(interval);
    return;
  }

  // Reject if less than 60% or more than 165% of current average
  // (catches roughly-double or roughly-half glitches)
  if (interval < runningAvgInterval * 0.6 || interval > runningAvgInterval * 1.65) {
    Serial.println("Rejected outlier interval");
    return;
  }

  addInterval(interval);
}

void addInterval(unsigned long interval) {
  intervalBuffer[bufferIndex] = interval;
  bufferIndex = (bufferIndex + 1) % BUFFER_SIZE;
  if (bufferCount < BUFFER_SIZE) bufferCount++;

  unsigned long sum = 0;
  for (int i = 0; i < bufferCount; i++) sum += intervalBuffer[i];
  runningAvgInterval = sum / bufferCount;
}

int getAverageBPM() {
  if (bufferCount == 0 || runningAvgInterval == 0) return 0;
  return (int)(60000UL / runningAvgInterval);
}

// Prints text to a given LCD row, padded with spaces to fill
// the full 16 characters so no leftover text from a previous
// (longer) message can remain on screen.
void printLine(int row, String text) {
  while (text.length() < 16) {
    text += " ";
  }
  if (text.length() > 16) {
    text = text.substring(0, 16);
  }
  lcd.setCursor(0, row);
  lcd.print(text);
}

void updateDisplay(unsigned long now) {
  if (now - lastSignalTime > SIGNAL_TIMEOUT) {
    printLine(0, "No pulse found");
    printLine(1, "Place finger...");
    bufferCount = 0;
    bufferIndex = 0;
    lastBeatTime = 0;
    runningAvgInterval = 0;
    return;
  }

  int bpm = getAverageBPM();

  if (bpm > 0) {
    printLine(0, "BPM: " + String(bpm));
  } else {
    printLine(0, "BPM: --");
  }

  if (bufferCount < BUFFER_SIZE) {
    printLine(1, "Calibrating...");
  } else {
    printLine(1, "Status: Stable");
  }
}
</code></pre>
</div>


# Bill of Materials


| **Part** | **Note** | **Price** | **Link** |
|:--:|:--:|:--:|:--:|
| Super Starter Kit UNO R3 Project | Provides the Arduino board, jumper wires, LCD, and other components | $35.98 | <a href="https://www.amazon.com/Arduino-A000066-ARDUINO-UNO-R3/dp/B008GRTSV6/](https://www.amazon.com/gp/product/B01D8KOZF4/ref=ox_sc_act_title_7?smid=A2WWHQ25ENKVJ1&psc=1"> Link </a> |
| Pulse Sensor | Detects changes in blood flow to measure the user’s pulse rate in beats per minute | $7.99 | <a href="https://www.amazon.com/gp/product/B07V6VV8CM/ref=ox_sc_act_title_5?smid=AYZI0TO4JGBW9&psc=1"> Link </a> |

# Other Resources/Examples

- [Schematic Maker](tinkercad.com)
- [Arduino IDE](https://www.arduino.cc/en/software/)
- [Example Project](https://how2electronics.com/pulse-rate-bpm-monitor-arduino-pulse-sensor/)
- [Code Maker](https://claude.ai/new)
