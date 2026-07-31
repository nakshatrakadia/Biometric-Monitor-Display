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

<iframe width="560" height="315" src="https://www.youtube.com/embed/y3VAmNlER5Y" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>



For my second milestone, I focused on making the biometric heart-rate monitor more compact and organized. I replaced the original LCD with an I2C LCD, which reduced the number of wires needed and made the circuit much simpler. I also removed the breadboard and reorganized the wiring inside the project.

I created a cardboard enclosure to hold and protect the components. I cut openings in the box for the LCD display, the Arduino USB port, and the barrel jack so the project can still be programmed and powered without removing it from the enclosure.

I also updated the Arduino code to make the heart-rate readings more stable and reduce random BPM fluctuations. The program now does a better job of detecting a consistent pulse before displaying a BPM value.

One challenge was fitting all of the components neatly inside the enclosure while keeping the ports accessible. This helped me learn more about planning the physical layout instead of focusing only on the circuit and code.

For my final milestone, I hope to replace the cardboard enclosure with a wooden frame or enclosure to make the final project stronger and more professional-looking.

# First Milestone

<iframe width="560" height="315" src="https://www.youtube.com/embed/1PXAsAyX8vo?si=GI4OxZExRfveXz39" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

My project is an Arduino-based device that measures the user’s pulse and displays the heart rate on an LCD screen. The main components of the project are the Arduino Uno, the pulse sensor, the LCD1602 screen, jumper wires, and a breadboard. The pulse sensor detects the blood flow from the user’s fingertip and displays the calculated beats per minute (BPM) on the screen using the Arduino.

The main progress that has been made includes assembling the circuit with the LCD screen in 4-bit mode, uploading the program to the Arduino, and displaying messages on the screen asking the user to place their finger on the sensor and messages that display the BPM of their heart rate. These messages are also displayed when the Arduino is performing signal filtering to find the BPM of the user.

One of the major challenges for the project was preventing the sensor from detecting random signals when there was no finger on the sensor. The BPM readings would also change if I moved my finger. Through these challenges, I learnt about signal filtering and how to properly create a program for a pulse sensor to detect BPM accurately. In addition, the LCD screen was found to be slightly dim. This is a hardware issue because the contrast of the screen can only be adjusted with the hardware of the LCD screen, not the Arduino code.

The next goal for my project will be to further test and improve the accuracy of the BPM measurements from the sensor. Another goal will be to replace the current LCD screen with an I2C screen that takes up less pins on the Arduino. A I2C LCD module may also provide a clearer or brighter display, although brightness will depend on the backlight of the specific module. I also plan to organize and secure the wiring and eventually create an enclosure for the completed biometric monitor.

# Schematics 

<img width="1428" height="503" alt="Schematic" src="https://github.com/user-attachments/assets/0198a1eb-1c7d-486f-b1a5-75f4e984e180" />

# Code


<div style="
  height: 350px;
  overflow-y: auto;
  overflow-x: hidden;
  background-color: #1e1e1e;
  color: white;
  padding: 15px;
  border-radius: 8px;
">
  <pre style="
    margin: 0;
    white-space: pre-wrap;
    overflow-wrap: anywhere;
    word-break: break-word;
    font-family: Consolas, monospace;
    font-size: 14px;
    line-height: 1.5;
  "><code>#include &lt;Wire.h&gt;
#include &lt;LiquidCrystal_I2C.h&gt;

LiquidCrystal_I2C lcd(0x27, 16, 2); // change to 0x3F if screen is blank

const int PULSE_PIN = A0;

// ---- Dynamic threshold state ----
int signalMin = 1023;
int signalMax = 0;
int thresholdHigh = 512;
int thresholdLow = 480;

const float THRESH_HIGH_RATIO = 0.55;
const float THRESH_LOW_RATIO = 0.45;

// ---- Calibration ----
const unsigned long CALIBRATION_TIME = 4000;
unsigned long calibrationStart = 0;
bool calibrated = false;

// ---- Beat detection ----
bool aboveThreshold = false;
unsigned long lastBeatTime = 0;
const unsigned long REFRACTORY_PERIOD = 350;

// ---- Averaging buffer ----
const int BUFFER_SIZE = 8;
unsigned long intervalBuffer[BUFFER_SIZE];
int bufferIndex = 0;
int bufferCount = 0;
unsigned long runningAvgInterval = 0;

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

  if (sensorValue &lt; signalMin) signalMin = sensorValue;
  if (sensorValue &gt; signalMax) signalMax = sensorValue;

  if (!calibrated) {
    if (now - calibrationStart &gt;= CALIBRATION_TIME) {
      finishCalibration();
    } else {
      return;
    }
  }

  if (now - lastRangeReset &gt;= RANGE_RESET_INTERVAL) {
    if ((signalMax - signalMin) &gt; 20) {
      updateThresholds();
    }

    signalMin = sensorValue;
    signalMax = sensorValue;
    lastRangeReset = now;
  }

  if (sensorValue &gt; thresholdHigh &amp;&amp; !aboveThreshold) {
    if (now - lastBeatTime &gt; REFRACTORY_PERIOD) {
      unsigned long interval = now - lastBeatTime;

      if (lastBeatTime != 0 &amp;&amp; interval &gt; 0) {
        handleNewInterval(interval);
      }

      lastBeatTime = now;
      lastSignalTime = now;
    }

    aboveThreshold = true;
  } else if (sensorValue &lt; thresholdLow) {
    aboveThreshold = false;
  }

  if (now - lastDisplayUpdate &gt;= DISPLAY_INTERVAL) {
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
  thresholdHigh =
    signalMin + (signalMax - signalMin) * THRESH_HIGH_RATIO;

  thresholdLow =
    signalMin + (signalMax - signalMin) * THRESH_LOW_RATIO;
}

void finishCalibration() {
  if (signalMax - signalMin &lt; 10) {
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

void handleNewInterval(unsigned long interval) {
  if (bufferCount == 0) {
    addInterval(interval);
    return;
  }

  if (
    interval &lt; runningAvgInterval * 0.6 ||
    interval &gt; runningAvgInterval * 1.65
  ) {
    Serial.println("Rejected outlier interval");
    return;
  }

  addInterval(interval);
}

void addInterval(unsigned long interval) {
  intervalBuffer[bufferIndex] = interval;
  bufferIndex = (bufferIndex + 1) % BUFFER_SIZE;

  if (bufferCount &lt; BUFFER_SIZE) {
    bufferCount++;
  }

  unsigned long sum = 0;

  for (int i = 0; i &lt; bufferCount; i++) {
    sum += intervalBuffer[i];
  }

  runningAvgInterval = sum / bufferCount;
}

int getAverageBPM() {
  if (bufferCount == 0 || runningAvgInterval == 0) {
    return 0;
  }

  return (int)(60000UL / runningAvgInterval);
}

void printLine(int row, String text) {
  while (text.length() &lt; 16) {
    text += " ";
  }

  if (text.length() &gt; 16) {
    text = text.substring(0, 16);
  }

  lcd.setCursor(0, row);
  lcd.print(text);
}

void updateDisplay(unsigned long now) {
  if (now - lastSignalTime &gt; SIGNAL_TIMEOUT) {
    printLine(0, "No pulse found");
    printLine(1, "Place finger...");

    bufferCount = 0;
    bufferIndex = 0;
    lastBeatTime = 0;
    runningAvgInterval = 0;

    return;
  }

  int bpm = getAverageBPM();

  if (bpm &gt; 0) {
    printLine(0, "BPM: " + String(bpm));
  } else {
    printLine(0, "BPM: --");
  }

  if (bufferCount &lt; BUFFER_SIZE) {
    printLine(1, "Calibrating...");
  } else {
    printLine(1, "Status: Stable");
  }
}</code></pre>
</div>


# Bill of Materials


| **Part** | **Note** | **Price** | **Link** |
|:--:|:--:|:--:|:--:|
| Super Starter Kit UNO R3 Project | Provides the Arduino board, jumper wires, LCD, and other components | $35.98 | <a href="https://www.amazon.com/Arduino-A000066-ARDUINO-UNO-R3/dp/B008GRTSV6/](https://www.amazon.com/gp/product/B01D8KOZF4/ref=ox_sc_act_title_7?smid=A2WWHQ25ENKVJ1&psc=1"> Link </a> |
| Pulse Sensor | Detects changes in blood flow to measure the user’s pulse rate in beats per minute | $7.99 | <a href="https://www.amazon.com/gp/product/B07V6VV8CM/ref=ox_sc_act_title_5?smid=AYZI0TO4JGBW9&psc=1"> Link </a> |
| Cardboard Boxes (only 1 required) | Used as a lightweight enclosure to hold the Arduino, LCD, pulse sensor, and wires in place while giving the project a cleaner appearance. | $36.99 | <a href="https://www.amazon.ca/MEBRUDY-Shipping-Corrugated-Cardboard-Literature/dp/B0FDQ3HN77/ref=sr_1_4_sspa?crid=1G6TDKJ9M4CQG&dib=eyJ2IjoiMSJ9.F-wkrtn70CfIHy7YiY5a3OHewFo3-1jaL8QSDRmzLQf8jTmFP7jm3UTprNO17IXhVcweG-HeppdYE9CY0IwzWgxoW570ZVcqEMaVutvSTazJwA1phw1CJiz_Kse_U2cs58nc2ktdLlUUpxTWQILOIaBteHqFM8Cl7V-UslgaX1CK1A7u3qwZ0c9iW6Jr_Apd6n-yV0ZqafPPZfmDTzLM0DuHA2B7_-U1e38Yfa-G4wd9RQVO7kp30cc_KEmS0mgW98G2zrNZJDsGjXMZx4LJBIdX69yJ2aLNC-VOS91Teec.eYjh1gYYKtM1zbj4174QJeL-fo2XXi2fe9-8FnuSZIQ&dib_tag=se&keywords=very%2Bsmall%2Bcardboard%2Bbox&qid=1785341703&sprefix=very%2Bsmall%2Bcardboard%2Bbox%2Caps%2C180&sr=8-4-spons&sp_csd=d2lkZ2V0TmFtZT1zcF9hdGY&th=1"> Link </a> |
| Electrical Tape | Used mainly to secure the wires and hold the components firmly in place inside the box, keeping the setup organized and preventing parts from moving. | $4.75 | <a href="https://www.amazon.ca/Scotch-Electrical-Tape-75-Inch-007-Inch-66-Feet/dp/B001AXD0EY/ref=sr_1_5?crid=3B8JYFY3K0DDO&dib=eyJ2IjoiMSJ9.O0Qvq8fRCwf8ZFpARWiylmWh-aeB8nyTWz0DP67nN75AqxwEKcQqkHGPgqA-zdrDwfkv_jmsjU5LzjQFMjrWAUjfDMGaj4VICYImXoZpAsG_0zhZ_r8O9J5Z-R0eEUuJldIrl0cLdfOIGBJJHegHnMa3qXoNTEQIcQEjbZf0rRmwN16_3XD6jU6xLPAaEQ2SGO14ws1go14oXyVWsA_S3TRTtb4YcI0p3Paq1X3dUZYe9nUE3VcqISqZ6qhGAGU3no_tkXF3vN-wHSzr9384aGphaT0dWKIZ51nS8qoK23s.ScHW3rbjx2EfcGZglvoS4rHYm4mk_hVn90dqLgNTRK0&dib_tag=se&keywords=electrical%2Btape&qid=1785342072&sprefix=electrical%2Btape%2Caps%2C175&sr=8-5&th=1"> Link </a> |

# Other Resources/Examples

- [Schematic Maker](tinkercad.com)
- [Arduino IDE](https://www.arduino.cc/en/software/)
- [Example Project](https://how2electronics.com/pulse-rate-bpm-monitor-arduino-pulse-sensor/)
- [Code Maker](https://claude.ai/new)
