# ⛽ IoT-Based Diesel Fuel Quality and Level Monitoring

An ESP32-based IoT system that continuously monitors diesel fuel **quality (contamination)** and **tank level** in real-time — triggering instant alerts to prevent fuel loss, engine damage, and theft.

---

## Overview

Fuel contamination and undetected fuel theft cause significant losses in industrial and transport sectors. This project provides an automated, always-on monitoring system:

1. A water/contamination sensor detects if the diesel is adulterated or mixed with water
2. An ultrasonic sensor measures the fuel level inside the tank
3. Real-time data is sent over WiFi to an IoT dashboard (Blynk / ThingSpeak)
4. Alerts are triggered via buzzer and LED when contamination or low fuel is detected

---

## Components List

| Component | Quantity | Purpose |
|---|---|---|
| ESP32 Dev Board | 1 | Main controller + WiFi connectivity |
| Ultrasonic Sensor (HC-SR04) | 1 | Fuel level measurement |
| Water / Liquid Contamination Sensor | 1 | Diesel quality / contamination detection |
| OLED Display (0.96" I2C, SSD1306) | 1 | On-device real-time data display |
| Red LED | 1 | Alert indicator (contamination / low fuel) |
| Green LED | 1 | Normal status indicator |
| Active Buzzer | 1 | Audible alert |
| Resistors (220Ω) | 2 | Current limit for LEDs |
| Breadboard | 1 | Prototyping connections |
| Jumper Wires | ~25 | Component connections |
| Micro USB Cable | 1 | Power and programming |
| 5V Power Supply / Power Bank | 1 | Standalone deployment power |



---

## Circuit / Wiring Diagram

> 📷 *See `/circuit/wiring_diagram.png` for the full Tinkercad/Proteus diagram.*

### Pin Connection Table

| Component | Component Pin | ESP32 Pin | Notes |
|---|---|---|---|
| HC-SR04 (Ultrasonic) | VCC | VIN (5V) | Needs 5V supply |
| HC-SR04 (Ultrasonic) | GND | GND | Ground |
| HC-SR04 (Ultrasonic) | TRIG | GPIO 5 | Trigger pulse |
| HC-SR04 (Ultrasonic) | ECHO | GPIO 18 | Echo return signal |
| Contamination Sensor | VCC | 3.3V | Power |
| Contamination Sensor | GND | GND | Ground |
| Contamination Sensor | A0 (Analog) | GPIO 34 (ADC) | Analog reading |
| OLED Display | VCC | 3.3V | Power |
| OLED Display | GND | GND | Ground |
| OLED Display | SDA | GPIO 21 | I2C data |
| OLED Display | SCL | GPIO 22 | I2C clock |
| Red LED | Anode (+) | GPIO 25 via 220Ω | Alert indicator |
| Red LED | Cathode (−) | GND | Ground |
| Green LED | Anode (+) | GPIO 26 via 220Ω | Normal status |
| Green LED | Cathode (−) | GND | Ground |
| Buzzer | + | GPIO 27 | Alert sound |
| Buzzer | − | GND | Ground |

### Wiring Notes

- **ESP32 ADC:** GPIO 34 is input-only and ADC-capable — ideal for the contamination sensor analog output.
- **HC-SR04:** This sensor requires 5V. Use the ESP32's VIN pin (which passes through the USB 5V) to power it. The echo pin outputs 5V logic — use a voltage divider (1kΩ + 2kΩ) before connecting to GPIO 18 to protect the ESP32's 3.3V GPIO.
- **OLED I2C:** SDA → GPIO 21, SCL → GPIO 22 are the default I2C pins on ESP32. No pull-up resistors needed if using a module with built-in pull-ups.
- **Contamination Threshold:** Adjust `CONTAMINATION_THRESHOLD` in the code based on your sensor's clean fuel baseline reading.
- **Tank Height:** Set `TANK_HEIGHT_CM` in the code to the actual height of your fuel tank for accurate percentage calculation.

---

## Code Explanation

> 📄 *Full source code in `/src/fuel_monitor.ino`*

### Libraries Used

```cpp
#include <WiFi.h>             // ESP32 built-in WiFi
#include <Wire.h>             // I2C for OLED
#include <Adafruit_SSD1306.h> // OLED display driver
#include <Adafruit_GFX.h>     // OLED graphics library
#include <BlynkSimpleEsp32.h> // Blynk IoT platform (optional)
```

### Key Constants

```cpp
// WiFi Credentials
const char* SSID     = "Your_WiFi_Name";
const char* PASSWORD = "Your_WiFi_Password";

// Tank & Sensor Config
#define TANK_HEIGHT_CM        50    // Physical tank height in cm
#define CONTAMINATION_THRESHOLD 500 // ADC value above this = contaminated
#define LOW_FUEL_PERCENT      20    // Alert when fuel drops below 20%

// Pin Definitions
#define TRIG_PIN     5
#define ECHO_PIN    18
#define SENSOR_PIN  34   // Contamination sensor analog input
#define RED_LED     25
#define GREEN_LED   26
#define BUZZER      27
```

### Fuel Level Measurement

```cpp
float getFuelLevelPercent() {
    // Send 10µs pulse on TRIG pin
    digitalWrite(TRIG_PIN, LOW);
    delayMicroseconds(2);
    digitalWrite(TRIG_PIN, HIGH);
    delayMicroseconds(10);
    digitalWrite(TRIG_PIN, LOW);

    // Measure echo duration → convert to distance
    long duration = pulseIn(ECHO_PIN, HIGH);
    float distanceCm = duration * 0.034 / 2;

    // Distance from sensor to fuel surface → fuel depth
    float fuelDepth = TANK_HEIGHT_CM - distanceCm;
    float percent   = (fuelDepth / TANK_HEIGHT_CM) * 100;

    return constrain(percent, 0, 100);  // Clamp to 0–100%
}
```

### Contamination Detection

```cpp
bool isFuelContaminated() {
    int rawValue = analogRead(SENSOR_PIN);   // 0–4095 (12-bit ADC)
    return rawValue > CONTAMINATION_THRESHOLD;
}
```

### Alert System

```cpp
void triggerAlert(String reason) {
    digitalWrite(RED_LED,   HIGH);
    digitalWrite(GREEN_LED, LOW);
    digitalWrite(BUZZER,    HIGH);
    Serial.println("[ALERT] " + reason);
    // Also send to Blynk dashboard
    Blynk.logEvent("fuel_alert", reason);
}

void clearAlert() {
    digitalWrite(RED_LED,   LOW);
    digitalWrite(GREEN_LED, HIGH);
    digitalWrite(BUZZER,    LOW);
}
```

### Main Loop Logic

```cpp
void loop() {
    float   fuelPercent   = getFuelLevelPercent();
    bool    contaminated  = isFuelContaminated();

    // Update OLED display
    displayData(fuelPercent, contaminated);

    // Send to IoT dashboard (Blynk virtual pins)
    Blynk.virtualWrite(V0, fuelPercent);
    Blynk.virtualWrite(V1, contaminated ? 1 : 0);

    // Check alert conditions
    if (contaminated) {
        triggerAlert("Fuel contamination detected!");
    } else if (fuelPercent < LOW_FUEL_PERCENT) {
        triggerAlert("Low fuel: " + String(fuelPercent) + "%");
    } else {
        clearAlert();
    }

    delay(2000);    // Update every 2 seconds
}
```

### System Flow Diagram

```
[ESP32 Powers On → Connects to WiFi]
         |
    Every 2 seconds:
         |
   ┌─────┴──────┐
   │            │
[Read Level]  [Read Contamination]
HC-SR04 →     Sensor ADC →
distance cm   0–4095 value
   │            │
   └─────┬──────┘
         │
  [Update OLED Display]
         │
  [Send to IoT Dashboard]
         │
  [Check Alert Conditions]
    /          \
Contaminated?  Low Fuel?
or Low Level?
    YES              NO
     │                │
[triggerAlert()]  [clearAlert()]
Red LED + Buzzer  Green LED ON
```

---

## Folder Structure

```
iot-diesel-fuel-monitor/
│
├── src/
│   └── fuel_monitor.ino           ← Main ESP32 sketch
│
├── circuit/
│   ├── wiring_diagram.png          ← Circuit diagram image
│   └── schematic.pdf               ← Printable schematic
│
├── simulation/
│   └── proteus_simulation.pdsprj   ← Proteus project (optional)
│
├── dashboard/
│   └── blynk_template.json         ← Blynk dashboard export (optional)
│
├── docs/
│   └── project_report.pdf          ← Detailed report (optional)
│
├── images/
│   ├── prototype_front.jpg
│   └── dashboard_screenshot.png
│
└── README.md                       ← This file
```

---

## How to Upload & Run

### Prerequisites

- [Arduino IDE](https://www.arduino.cc/en/software) with ESP32 board package installed
- Libraries: `Adafruit SSD1306`, `Adafruit GFX`, `Blynk` (install via Library Manager)
- A WiFi network
- Blynk account at [blynk.cloud](https://blynk.cloud) (free tier works)

### Steps

1. **Clone the repository**
   ```bash
   git clone https://github.com/YOUR_USERNAME/iot-diesel-fuel-monitor.git
   ```

2. **Open the sketch** in Arduino IDE → `src/fuel_monitor.ino`

3. **Install ESP32 board support**
   - `File → Preferences → Additional Board URLs:`
   - Add: `https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`
   - `Tools → Board Manager` → search "esp32" → Install

4. **Update your credentials** in the sketch:
   ```cpp
   const char* SSID     = "Your_WiFi_Name";
   const char* PASSWORD = "Your_WiFi_Password";
   char auth[]          = "Your_Blynk_Auth_Token";
   ```

5. **Select board and port**
   - `Tools → Board → ESP32 Dev Module`
   - `Tools → Port → COM_X` (your port)

6. **Upload** → Open Serial Monitor at 115200 baud to verify WiFi connection

7. **Test**
   - Place hand near ultrasonic sensor → fuel level should drop on OLED and dashboard
   - Dip contamination sensor in water → Red LED ON, buzzer sounds, alert appears on Blynk

---

## Future Improvements

- [ ] **SMS / WhatsApp Alert** — Send real-time fuel alerts via Twilio or WA Business API
- [ ] **Data Logging** — Log fuel readings to Google Sheets or InfluxDB for trend analysis
- [ ] **Theft Detection** — Detect sudden fuel level drops and flag as potential theft event
- [ ] **Multiple Tanks** — Scale to monitor several tanks with a single ESP32 and multiplexer
- [ ] **Battery + Solar** — Add solar panel and LiPo battery for remote off-grid deployment
- [ ] **Temperature Sensor** — Add DS18B20 for fuel temperature monitoring (affects density)
- [ ] **Mobile App** — Build a custom Flutter/React Native app instead of Blynk
- [ ] **PCB Design** — Design a custom PCB using KiCad for a professional enclosure build

---

## License

This project is open-source under the [MIT License](LICENSE).

---

## Author

**Thirumala Rao Bommineni**
B.Tech ECE — Alturi Mastan Reddy Memorial College of Engineering & Technology
📧 thirumal.ofcl@gmail.com
