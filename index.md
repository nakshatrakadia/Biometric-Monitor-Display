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

For your first milestone, describe what your project is and how you plan to build it. You can include:
- An explanation about the different components of your project and how they will all integrate together
- Technical progress you've made so far
- Challenges you're facing and solving in your future milestones
- What your plan is to complete your project

# Schematics 

<img width="1310" height="848" alt="Screenshot 2026-07-22 083251" src="https://github.com/user-attachments/assets/624d70d8-8cd0-460f-aa04-744e6845cd13" />




# Code

```c++
#include <LiquidCrystal.h>

LiquidCrystal lcd(12, 11, 5, 4, 3, 2);

byte heartChar[8] = {
  0b00000,
  0b01010,
  0b11111,
  0b11111,
  0b11111,
  0b01110,
  0b00100,
  0b00000
};

const int PULSE_PIN = A0;

const unsigned long SAMPLE_INTERVAL_MS = 4;
unsigned long lastSampleTime = 0;

float baseline = 0;
bool baselineInitialized = false;
const float BASELINE_ALPHA = 0.02;

float acNoiseLevel = 2.0;
const float NOISE_ALPHA = 0.05;

const float THRESHOLD_MULTIPLIER = 1.8;
const float MIN_THRESHOLD = 12.0;

bool aboveThreshold = false;
float currentPeakValue = 0;
unsigned long currentPeakTime = 0;
unsigned long lastAcceptedPeakTime = 0;

const unsigned long MIN_BEAT_INTERVAL_MS = 333;
const unsigned long MAX_BEAT_INTERVAL_MS = 1500;

const int INTERVAL_HISTORY_SIZE = 5;
unsigned long intervalHistory[INTERVAL_HISTORY_SIZE];
int intervalCount = 0;
int intervalIndex = 0;

const int REQUIRED_CONSISTENT_INTERVALS = 4;
const float MAX_INTERVAL_DEVIATION = 0.25;

int consecutiveRejects = 0;
const int MAX_CONSECUTIVE_REJECTS = 4;

const int SATURATION_LOW = 3;
const int SATURATION_HIGH = 1020;

enum PulseState {
  NO_SIGNAL,
  ACQUIRING,
  VALID_PULSE
};

PulseState state = NO_SIGNAL;
PulseState lastDisplayedState = NO_SIGNAL;

bool anyActivitySeen = false;
unsigned long lastPeakAttemptTime = 0;
unsigned long lastValidBeatTime = 0;

const unsigned long NO_SIGNAL_TIMEOUT_MS = 3000;
const unsigned long VALID_PULSE_TIMEOUT_MS = 4000;

float currentBPM = 0;

int lastDisplayedBPM = -1;
unsigned long lastLcdUpdate = 0;
const unsigned long LCD_MIN_UPDATE_MS = 200;

void setup() {
  lcd.begin(16, 2);
  lcd.createChar(0, heartChar);
  showNoHeartbeat();
}

void loop() {
  unsigned long now = millis();

  if (now - lastSampleTime >= SAMPLE_INTERVAL_MS) {
    lastSampleTime = now;
    processSample(now);
  }

  updateStateTimeouts(now);
  updateDisplay(now);
}

void processSample(unsigned long now) {
  int raw = analogRead(PULSE_PIN);

  if (raw <= SATURATION_LOW || raw >= SATURATION_HIGH) {
    resetDetector();
    return;
  }

  if (!baselineInitialized) {
    baseline = raw;
    baselineInitialized = true;
  } else {
    baseline += BASELINE_ALPHA * ((float)raw - baseline);
  }

  float ac = (float)raw - baseline;

  if (!aboveThreshold) {
    acNoiseLevel += NOISE_ALPHA * (fabs(ac) - acNoiseLevel);

    if (acNoiseLevel < 1.0) {
      acNoiseLevel = 1.0;
    }
  }

  float threshold = acNoiseLevel * THRESHOLD_MULTIPLIER;

  if (threshold < MIN_THRESHOLD) {
    threshold = MIN_THRESHOLD;
  }

  bool peakDetected = false;

  if (!aboveThreshold) {
    if (ac > threshold &&
        now - lastAcceptedPeakTime > MIN_BEAT_INTERVAL_MS) {

      aboveThreshold = true;
      currentPeakValue = ac;
      currentPeakTime = now;
      anyActivitySeen = true;
      lastPeakAttemptTime = now;
    }
  } else {
    if (ac > currentPeakValue) {
      currentPeakValue = ac;
      currentPeakTime = now;
    }

    if (ac < threshold * 0.5) {
      aboveThreshold = false;
      peakDetected = true;
    }
  }

  if (!peakDetected) {
    return;
  }

  if (lastAcceptedPeakTime == 0) {
    lastAcceptedPeakTime = currentPeakTime;
    state = ACQUIRING;
    return;
  }

  unsigned long interval =
      currentPeakTime - lastAcceptedPeakTime;

  if (interval < MIN_BEAT_INTERVAL_MS ||
      interval > MAX_BEAT_INTERVAL_MS) {

    registerReject();
    return;
  }

  if (intervalCount >= 2 &&
      intervalDeviatesTooMuch(interval)) {

    registerReject();
    return;
  }

  intervalHistory[intervalIndex] = interval;
  intervalIndex =
      (intervalIndex + 1) % INTERVAL_HISTORY_SIZE;

  if (intervalCount < INTERVAL_HISTORY_SIZE) {
    intervalCount++;
  }

  consecutiveRejects = 0;
  lastAcceptedPeakTime = currentPeakTime;
  lastValidBeatTime = now;

  if (intervalCount >= REQUIRED_CONSISTENT_INTERVALS) {
    unsigned long median = medianInterval();
    currentBPM = 60000.0 / (float)median;
    state = VALID_PULSE;
  } else {
    state = ACQUIRING;
  }
}

bool intervalDeviatesTooMuch(unsigned long interval) {
  unsigned long median = medianInterval();

  if (median == 0) {
    return false;
  }

  float deviation =
      fabs((float)interval - (float)median) /
      (float)median;

  return deviation > MAX_INTERVAL_DEVIATION;
}

unsigned long medianInterval() {
  unsigned long sorted[INTERVAL_HISTORY_SIZE];

  for (int i = 0; i < intervalCount; i++) {
    sorted[i] = intervalHistory[i];
  }

  for (int i = 1; i < intervalCount; i++) {
    unsigned long value = sorted[i];
    int j = i - 1;

    while (j >= 0 && sorted[j] > value) {
      sorted[j + 1] = sorted[j];
      j--;
    }

    sorted[j + 1] = value;
  }

  return sorted[intervalCount / 2];
}

void registerReject() {
  consecutiveRejects++;

  if (consecutiveRejects >= MAX_CONSECUTIVE_REJECTS) {
    intervalCount = 0;
    intervalIndex = 0;
    consecutiveRejects = 0;
    lastAcceptedPeakTime = 0;
    currentBPM = 0;
    state = ACQUIRING;
  }
}

void resetDetector() {
  intervalCount = 0;
  intervalIndex = 0;
  consecutiveRejects = 0;

  aboveThreshold = false;
  currentBPM = 0;
  lastAcceptedPeakTime = 0;

  baselineInitialized = false;
  anyActivitySeen = false;

  state = NO_SIGNAL;
}

void updateStateTimeouts(unsigned long now) {
  if (state == VALID_PULSE &&
      now - lastValidBeatTime >
          VALID_PULSE_TIMEOUT_MS) {

    intervalCount = 0;
    intervalIndex = 0;
    consecutiveRejects = 0;
    currentBPM = 0;
    state = ACQUIRING;
  }

  if (!anyActivitySeen ||
      now - lastPeakAttemptTime >
          NO_SIGNAL_TIMEOUT_MS) {

    if (state != NO_SIGNAL) {
      intervalCount = 0;
      intervalIndex = 0;
      consecutiveRejects = 0;
      currentBPM = 0;
      state = NO_SIGNAL;
    }
  }
}

void updateDisplay(unsigned long now) {
  if (now - lastLcdUpdate < LCD_MIN_UPDATE_MS) {
    return;
  }

  if (state == NO_SIGNAL) {
    if (lastDisplayedState != NO_SIGNAL) {
      showNoHeartbeat();
      lastDisplayedState = NO_SIGNAL;
      lastDisplayedBPM = -1;
    }
  } else if (state == ACQUIRING) {
    if (lastDisplayedState != ACQUIRING) {
      showAcquiring();
      lastDisplayedState = ACQUIRING;
      lastDisplayedBPM = -1;
    }
  } else {
    int bpm = (int)round(currentBPM);

    if (lastDisplayedState != VALID_PULSE ||
        bpm != lastDisplayedBPM) {

      showValidPulse(bpm);
      lastDisplayedState = VALID_PULSE;
      lastDisplayedBPM = bpm;
    }
  }

  lastLcdUpdate = now;
}

void writeLine(int row, const char* text) {
  lcd.setCursor(0, row);

  char buffer[17];
  int i = 0;

  while (text[i] != '\0' && i < 16) {
    buffer[i] = text[i];
    i++;
  }

  while (i < 16) {
    buffer[i] = ' ';
    i++;
  }

  buffer[16] = '\0';
  lcd.print(buffer);
}

void showNoHeartbeat() {
  writeLine(0, "No heartbeat");
  writeLine(1, "Place finger");
}

void showAcquiring() {
  writeLine(0, "Reading pulse");
  writeLine(1, "Keep still...");
}

void showValidPulse(int bpm) {
  lcd.setCursor(0, 0);
  lcd.write(byte(0));
  lcd.print(" HeartBeat!    ");

  char secondLine[17];
  snprintf(
      secondLine,
      sizeof(secondLine),
      "BPM: %-11d",
      bpm
  );

  writeLine(1, secondLine);
}
```

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
