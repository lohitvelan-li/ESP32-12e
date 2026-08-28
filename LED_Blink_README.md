# LED Blink — Arduino/ESP Basic Sketch

A minimal sketch that blinks an LED connected to pin 5, turning it ON and OFF every second.

## Code

```cpp
void setup() {
  // put your setup code here, to run once:
  pinMode(5, OUTPUT);
}

void loop() {
  // put your main code here, to run repeatedly:
  digitalWrite(5, HIGH);
  delay(1000);
  digitalWrite(5, LOW);
  delay(1000);
}
```

## How It Works

| Line | Explanation |
|---|---|
| `pinMode(5, OUTPUT);` | Configures pin 5 as an output pin, so it can drive a connected LED. Runs once when the board starts. |
| `digitalWrite(5, HIGH);` | Sets pin 5 to 5V (or 3.3V depending on the board), turning the LED **ON**. |
| `delay(1000);` | Pauses execution for 1000 milliseconds (1 second). |
| `digitalWrite(5, LOW);` | Sets pin 5 to 0V, turning the LED **OFF**. |
| `delay(1000);` | Pauses for another second before the loop repeats. |

Since `loop()` runs continuously, the LED blinks ON for 1 second and OFF for 1 second, forever.

## Wiring

- Connect the **anode (+)** of the LED to pin 5 through a current-limiting resistor (e.g. 220Ω–330Ω).
- Connect the **cathode (–)** of the LED to **GND**.

## Requirements

- Any Arduino-compatible board (Uno, Nano, ESP32, ESP8266, etc.) with a digital pin 5 available.
- Arduino IDE with the correct board selected under **Tools > Board**.

## Customization

- Change the pin number in `pinMode()` and both `digitalWrite()` calls to use a different GPIO.
- Change the `delay()` values (in milliseconds) to blink faster or slower.
