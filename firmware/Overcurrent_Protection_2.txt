const int sensorPin = A1;
const int outputPin = 7;
const int buttonPin = 4;

const float sensitivity = 0.066;     // ACS712-30A (V/A)
const float tripThreshold = 0.5;     // Trip current (A)

bool tripState = false;

void setup() {
  Serial.begin(9600);

  pinMode(outputPin, OUTPUT);
  pinMode(buttonPin, INPUT_PULLUP);

  digitalWrite(outputPin, HIGH);     // Normal state
}

void loop() {

  const int samples = 100;           // Reduced from 1000

  // Calculate average ADC value (dynamic offset)
  long adcSum = 0;
  for (int i = 0; i < samples; i++) {
    adcSum += analogRead(sensorPin);
  }
  float adcOffset = adcSum / (float)samples;

  // Calculate RMS current
  float sumSquares = 0;
  for (int i = 0; i < samples; i++) {
    float adc = analogRead(sensorPin);

    float voltage = ((adc - adcOffset) * 5.0) / 1023.0;
    float current = voltage / sensitivity;

    sumSquares += current * current;
  }

  float rmsCurrent = sqrt(sumSquares / samples);

  // Low-pass filter
  static float filteredCurrent = 0;
  filteredCurrent = 0.9 * filteredCurrent + 0.1 * rmsCurrent;

  // Print continuously
  Serial.print("RMS Current: ");
  Serial.print(filteredCurrent, 3);
  Serial.println(" A");

  // Trip only once (latch)
  if (!tripState && filteredCurrent > tripThreshold) {
    tripState = true;
    Serial.println(">>> TRIPPED <<<");
  }

  // Reset button (D4 to GND)
  if (digitalRead(buttonPin) == LOW) {
    tripState = false;
    Serial.println(">>> RESET <<<");
    delay(300);   // Debounce
  }

  // Output control
  if (tripState) {
    digitalWrite(outputPin, LOW);
  } else {
    digitalWrite(outputPin, HIGH);
  }
}