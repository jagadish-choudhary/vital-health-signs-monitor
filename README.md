# vital-health-signs-monitor

arduino code : 

#include <Wire.h>
#include "MAX30105.h"
#include "heartRate.h"
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// =====================================
// OLED SETTINGS
// =====================================
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// =====================================
// MAX30102 SENSOR OBJECT
// =====================================
MAX30105 particleSensor;

// =====================================
// HEART RATE VARIABLES (4-Beat Average)
// =====================================
const byte RATE_SIZE = 4;
byte rates[RATE_SIZE] = {0};
byte rateSpot = 0;
byte validRateCount = 0;

long lastBeat = 0;
float beatsPerMinute = 0;
int beatAvg = 0;

// =====================================
// SpO2 OPTION 1.5 VARIABLES (Min/Max)
// =====================================
long irValue = 0;
long redValue = 0;

long irMax = 0;
long irMin = 999999;
long redMax = 0;
long redMin = 999999;

int sampleCount = 0;
float finalSpO2 = 0; // The actual displayed value

unsigned long lastOLEDUpdate = 0;

void setup() {
  Serial.begin(115200);
  Wire.begin(21, 22);

  // 1. Initialize OLED
  if(!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("SSD1306 allocation failed");
    while(1);
  }
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 20);
  display.println("System Starting...");
  display.display();

  // 2. Initialize MAX30102
  if (!particleSensor.begin(Wire, I2C_SPEED_FAST)) {
    Serial.println("MAX30102 not found. Check wiring.");
    while (1);
  }

  // Configure sensor for clean cardiac reads
  particleSensor.setup(0x3F, 4, 2, 100, 411, 4096);
  particleSensor.setPulseAmplitudeRed(0x3F);
  particleSensor.setPulseAmplitudeIR(0x3F);
  particleSensor.setPulseAmplitudeGreen(0);
}

void loop() {
  // =====================================
  // 1. DRAIN THE SENSOR BUFFER
  // =====================================
  particleSensor.check();

  while (particleSensor.available()) {
    irValue = particleSensor.getFIFOIR();
    redValue = particleSensor.getFIFORed();

    // THRESHOLD LOWERED TO 30000 TO PREVENT MICRO-TWITCH RESETS
    if (irValue > 30000) { 
      
      // =====================================
      // 2. HEART RATE DETECTION
      // =====================================
      if (checkForBeat(irValue)) {
        long now = millis();
        long delta = now - lastBeat;
        lastBeat = now;

        if (delta > 0) {
          beatsPerMinute = 60000.0 / delta;

          if (beatsPerMinute >= 40 && beatsPerMinute <= 180) {
            rates[rateSpot] = (byte)beatsPerMinute;
            rateSpot = (rateSpot + 1) % RATE_SIZE;

            if (validRateCount < RATE_SIZE) validRateCount++;

            int total = 0;
            for (byte i = 0; i < validRateCount; i++) {
              total += rates[i];
            }
            beatAvg = total / validRateCount;
          }
        }
      }

      // =====================================
      // 3. SpO2 AC/DC ESTIMATOR (Option 1.5)
      // =====================================
      if (irValue > irMax) irMax = irValue;
      if (irValue < irMin) irMin = irValue;
      if (redValue > redMax) redMax = redValue;
      if (redValue < redMin) redMin = redValue;
      sampleCount++;

      // Process every ~1 second (100 samples)
      if (sampleCount >= 100) {
        float irAC = irMax - irMin;
        float irDC = irMin;
        float redAC = redMax - redMin;
        float redDC = redMin;

        // BEDSHEET PROTECTOR: Only calculate if there is an actual physical pulse
        if (irDC > 0 && redDC > 0 && irAC > 50) {
          float ratio = (redAC / redDC) / (irAC / irDC);
          
          float calculatedSpO2 = 110.0 - (25.0 * ratio);
          
          if (calculatedSpO2 > 100) calculatedSpO2 = 100;
          if (calculatedSpO2 < 0) calculatedSpO2 = 0;
          
          finalSpO2 = calculatedSpO2;
        } 
        
        // Reset window for the next second
        irMax = 0; irMin = 999999;
        redMax = 0; redMin = 999999;
        sampleCount = 0;
      }

    } else {
      // =====================================
      // 4. FINGER REMOVED: RESET VALUES
      // =====================================
      beatAvg = 0;
      finalSpO2 = 0;
      validRateCount = 0;
      rateSpot = 0;
      lastBeat = 0;
      for (byte i = 0; i < RATE_SIZE; i++) rates[i] = 0;
      
      irMax = 0; irMin = 999999;
      redMax = 0; redMin = 999999;
      sampleCount = 0;
    }
    
    // Move to the next sample in the FIFO buffer
    particleSensor.nextSample();
  }

  // =====================================
  // 5. NON-BLOCKING OLED UPDATE (1 Hz)
  // =====================================
  if (millis() - lastOLEDUpdate >= 1000) {
    lastOLEDUpdate = millis();

    display.clearDisplay();
    display.setTextSize(1);
    
    display.setCursor(0, 0);
    display.println("CARDIAC MONITOR");
    display.drawLine(0, 12, 128, 12, SSD1306_WHITE);

    // THRESHOLD LOWERED TO 30000 HERE AS WELL
    if (irValue > 30000) {
      display.setCursor(0, 25);
      display.print("Heart Rate: ");
      display.print(beatAvg);
      display.println(" BPM");

      display.setCursor(0, 45);
      display.print("SpO2: ");
      
      // OLED DOUBLE GATE: Show data as soon as the first valid beats are detected
      if (finalSpO2 > 80 && validRateCount > 0) {
        display.print(finalSpO2, 1);
        display.println(" %");
      } else {
        display.println("-- %");
      }
    } else {
      display.setCursor(0, 30);
      display.println("Place Finger on");
      display.setCursor(0, 45);
      display.println("Sensor...");
    }
    
    display.display(); 
  }
}
