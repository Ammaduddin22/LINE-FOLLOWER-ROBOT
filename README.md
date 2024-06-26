# LINE-FOLLOWER-ROBOT
#include <Arduino.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <BluetoothSerial.h>

// OLED display configuration
const int SDA_PIN = 5;  // SDA for OLED
const int SCL_PIN = 19;  // SCL for OLED
const int OLED_RESET = -1;  // Not used with I2C
Adafruit_SSD1306 display(128, 64, &Wire, OLED_RESET);

// ADC pins for voltage and current measurement
const int voltagePin = 34;  // Analog pin for voltage sensor
const int currentPin = 35;  // Analog pin for current sensor
  // Encoder pin for RPM measurement
const int ADC_RESOLUTION = 4096;  // 12-bit ADC
const float ADC_REF_VOLTAGE = 3.3;  // Reference voltage for ADC
const float voltageDividerRatio = 7501.0 / (7501.0 + 30002.0);  // Voltage divider ratio
const float SENSOR_SENSITIVITY = 0.185;  // Sensitivity in V/A for ACS712 (5A range)
const float SENSOR_OFFSET = 2.5;  // Output voltage at zero current

// TCS3200 color sensor configuration
const int outPin = 23;  // Frequency output pin for TCS3200
const int s0Pin = 13;  // S0 control pin for frequency scaling
const int s1Pin = 14;  // S1 control pin for frequency scaling
const int s2Pin = 21;  // S2 control pin for color selection
const int s3Pin = 18;  // S3 control pin for color selection
const int measureDuration = 500;  // Measurement duration in milliseconds

const int encoderPin = 15;  // Encoder pin (D15)
const float pulsesPerRotation = 20.0;  // Pulses per rotation
const float distancePerPulse = 0.1;   // Distance per pulse (adjust as needed)
const int updateInterval = 1000;      // Interval for updating distance in ms


// Bluetooth configuration
BluetoothSerial SerialBT;
volatile int pulseCount = 0;
float totalDistance = 0.0;


// ISR for counting encoder pulses (RPM)
void IRAM_ATTR onPulse() {
  pulseCount++;  // Increment pulse count on rising edge
}

// ISR for counting TCS3200 pulses
void IRAM_ATTR countPulse() {
  pulseCount++;  // Increment pulse count
}

// Function to measure frequency with TCS3200
unsigned long measureFrequency() {
  pulseCount = 0;  // Reset pulse count
  unsigned long startTime = millis();  // Start time for measurement

  // Wait for the specified duration
  while ((millis() - startTime) < measureDuration) {
    // ISR handles pulse counting
  }

  return pulseCount * 1000 / measureDuration;  // Return frequency in Hz
}



void setup() {
  Serial.begin(115200);  // Start serial communication
  SerialBT.begin("ESP32_BT"); // Bluetooth device name

  // Set up OLED with I2C
  Wire.begin(SDA_PIN, SCL_PIN);
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("Failed to initialize OLED display");
    while (true);  // Loop indefinitely if OLED initialization fails
  }

  // Flip the OLED display 180 degrees
  display.setRotation(2); 

  // Set up control pins for TCS3200
  pinMode(s0Pin, OUTPUT);
  pinMode(s1Pin, OUTPUT);
  pinMode(s2Pin, OUTPUT);
  pinMode(s3Pin, OUTPUT);
  pinMode(outPin, INPUT);

  // Configure frequency scaling for TCS3200
  digitalWrite(s0Pin, HIGH);  // 20% frequency scaling
  digitalWrite(s1Pin, LOW);  // 20% frequency scaling

  // Attach interrupt for TCS3200
  attachInterrupt(digitalPinToInterrupt(outPin), countPulse, RISING);  // ISR for TCS3200

  // Set up encoder and attach interrupt for RPM
  pinMode(encoderPin, INPUT);
  attachInterrupt(digitalPinToInterrupt(encoderPin), onPulse, RISING);  // ISR for RPM

  // Set up analog pins for voltage and current measurement
  pinMode(voltagePin, INPUT);
  pinMode(currentPin, INPUT);
}

void loop() {
   Serial.begin(115200);  // Start serial communication
    SerialBT.begin("ESP32_BT"); // Bluetooth device name

    // Set up OLED with I2C
    Wire.begin(SDA_PIN, SCL_PIN);
    if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
        Serial.println("Failed to initialize OLED display");
        while (true);  // Loop indefinitely if OLED initialization fails
    }

    // Flip the OLED display 180 degrees
    display.setRotation(2); 

    // Set up encoder and attach interrupt for distance measurement
    pinMode(encoderPin, INPUT);
    attachInterrupt(digitalPinToInterrupt(encoderPin), onPulse, RISING);
 
float currentDistance = pulseCount * distancePerPulse;

    // Update total distance
    totalDistance += currentDistance;

    // Clear pulse count for the next interval
    pulseCount = 0;

    // Display total distance on OLED
    display.clearDisplay();   // Clear the display buffer
    display.setTextSize(2);   // Set text size
    display.setTextColor(WHITE); // Set text color
    display.setCursor(0, 0); 
  // Measure voltage
  int voltageAdcValue = analogRead(voltagePin);
  float measuredVoltage = (voltageAdcValue / (float)ADC_RESOLUTION) * ADC_REF_VOLTAGE;
  float actualVoltage = measuredVoltage / voltageDividerRatio;

  // Measure current
  int currentAdcValue = analogRead(currentPin);
  float sensorVoltage = (currentAdcValue / (float)ADC_RESOLUTION) * ADC_REF_VOLTAGE;
  float current = (sensorVoltage - SENSOR_OFFSET) / SENSOR_SENSITIVITY;

  // Measure color with TCS3200
  digitalWrite(s2Pin, LOW);  // Red filter
  digitalWrite(s3Pin, LOW);  // Red filter
  unsigned long redFrequency = measureFrequency();

  digitalWrite(s2Pin, HIGH);  // Green filter
  digitalWrite(s3Pin, HIGH);  // Green filter
  unsigned long greenFrequency = measureFrequency();

  digitalWrite(s2Pin, LOW);  // Blue filter
  digitalWrite(s3Pin, HIGH);  // Blue filter
  unsigned long blueFrequency = measureFrequency();

  // Determine the dominant color
  String detectedColor;
  if (redFrequency > greenFrequency && redFrequency > blueFrequency) {
    detectedColor = "Red";
  } else if (greenFrequency > redFrequency && greenFrequency > blueFrequency) {
    detectedColor = "Green";
  } else if (blueFrequency > redFrequency && blueFrequency > greenFrequency) {
    detectedColor = "Blue";
  } else {
    detectedColor = "Unknown";
  }

  // Display on OLED
  display.clearDisplay();  // Clear the display buffer
  display.setTextSize(1);  // Smaller text size
  display.setTextColor(WHITE); 
  display.setCursor(0, 0);  // Start at the top-left corner 


  display.print("Voltage: ");
  display.print(actualVoltage, 2);  // Show 2 decimal places
  display.println(" V");

  display.print("Current: ");
  display.print(current, 3);  // Show 3 decimal places
  display.println(" A");

  display.print("Color: ");
  display.println(detectedColor);

  
   display.print("Distance: ");
    display.print(totalDistance, 2);  // Display distance with two decimal places
    display.println(" m");

    display.display(); // Update the OLED display

  // Output to Serial Monitor for debugging
  
  Serial.print(", Voltage: ");
  Serial.print(actualVoltage, 2);
  Serial.print(" V, Current: ");
  Serial.print(current, 3);
  Serial.print(" A, Color: ");
  Serial.println(detectedColor);

  // Send data over Bluetooth
 
  SerialBT.print(", Voltage: ");
  SerialBT.print(actualVoltage, 2);
  SerialBT.print(" V, Current: ");
  SerialBT.print(current, 3);
  SerialBT.print(" A, Color: ");
  SerialBT.println(detectedColor);
      SerialBT.print("Distance: ");
    SerialBT.print(totalDistance, 2);  // Output distance with two decimal places
    SerialBT.println(" m");
 delay(updateInterval); // Wait for the next interval

  delay(1000);  // Delay before the next loop iteration
}
