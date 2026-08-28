# MQTT Chat Between NodeMCU (ESP8266) and Raspberry Pi

This guide sets up a simple two-way "chat" using MQTT: the Raspberry Pi runs the MQTT broker (Mosquitto), and both the Pi and the NodeMCU act as clients that publish and subscribe to topics.

## Architecture

```
[NodeMCU] <-- WiFi --> [Mosquitto Broker on Raspberry Pi] <--> [Python client on Pi]

publish:   chat/nodemcu        publish:   chat/pi
subscribe: chat/pi             subscribe: chat/nodemcu
```

- The **NodeMCU** publishes messages to `chat/nodemcu` and listens for messages on `chat/pi`.
- The **Python client on the Pi** publishes messages to `chat/pi` and listens for messages on `chat/nodemcu`.
- The **Mosquitto broker** (also running on the Pi) routes messages between the two — neither client talks directly to the other.

---

## Part 1 — Raspberry Pi: Install the MQTT Broker

### 1. Update the system

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install Mosquitto broker and command-line clients

```bash
sudo apt install -y mosquitto mosquitto-clients
```

### 3. Enable Mosquitto to start on boot

```bash
sudo systemctl enable mosquitto
sudo systemctl start mosquitto
sudo systemctl status mosquitto
```

### 4. Allow network connections (recommended)

By default, newer Mosquitto versions only listen on `localhost`. Create a config file:

```bash
sudo nano /etc/mosquitto/conf.d/default.conf
```

Paste the following:

```
listener 1883
allow_anonymous true
```

For real use, set `allow_anonymous` to `false` and create a username/password instead:

```bash
sudo mosquitto_passwd -c /etc/mosquitto/passwd pi_user
```

Then add `password_file /etc/mosquitto/passwd` to the config file above. Restart the service:

```bash
sudo systemctl restart mosquitto
```

### 5. Find the Pi's IP address (NodeMCU will need this)

```bash
hostname -I
```

### 6. Test the broker locally

Open two terminals on the Pi:

```bash
# Terminal A - subscribe
mosquitto_sub -h localhost -t "chat/test"
```

```bash
# Terminal B - publish
mosquitto_pub -h localhost -t "chat/test" -m "Hello broker"
```

You should see `"Hello broker"` appear in Terminal A. This confirms the broker is working before adding networked devices.

---

## Part 2 — NodeMCU (ESP8266): Arduino Sketch

### 1. Install required libraries

In the Arduino IDE, install these via **Library Manager**:
- `PubSubClient` by Nick O'Leary
- `ESP8266WiFi` (bundled with the ESP8266 board package)

Make sure the **ESP8266 board package** is installed via **Boards Manager** as well.

### 2. Upload the sketch

```cpp
#include <ESP8266WiFi.h>
#include <PubSubClient.h>

// WiFi credentials
const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";

// MQTT broker (Raspberry Pi) details
const char* mqtt_server = "PI_IP_ADDRESS";   // e.g. 192.168.1.50
const int mqtt_port = 1883;
// If you enabled authentication in Part 1, set these:
const char* mqtt_user = "";       // leave blank if using allow_anonymous true
const char* mqtt_pass = "";

const char* topic_publish   = "chat/nodemcu";
const char* topic_subscribe = "chat/pi";

WiFiClient espClient;
PubSubClient client(espClient);

void setup_wifi() {
  delay(10);
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println();
  Serial.println("WiFi connected");
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());
}

// Called whenever a message arrives on a subscribed topic
void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Message arrived on [");
  Serial.print(topic);
  Serial.print("]: ");

  String message;
  for (unsigned int i = 0; i < length; i++) {
    message += (char)payload[i];
  }
  Serial.println(message);
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    String clientId = "NodeMCU-" + String(ESP.getChipId());

    bool connected;
    if (strlen(mqtt_user) > 0) {
      connected = client.connect(clientId.c_str(), mqtt_user, mqtt_pass);
    } else {
      connected = client.connect(clientId.c_str());
    }

    if (connected) {
      Serial.println("connected");
      client.subscribe(topic_subscribe);
      client.publish(topic_publish, "NodeMCU is online");
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" trying again in 5 seconds");
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  setup_wifi();
  client.setServer(mqtt_server, mqtt_port);
  client.setCallback(callback);
}

unsigned long lastMsg = 0;

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  // Send a heartbeat message every 10 seconds
  unsigned long now = millis();
  if (now - lastMsg > 10000) {
    lastMsg = now;
    String msg = "Hello from NodeMCU at " + String(now / 1000) + "s";
    client.publish(topic_publish, msg.c_str());
    Serial.println("Published: " + msg);
  }
}
```

Update `ssid`, `password`, and `mqtt_server` with your own network details before uploading.

---

## Part 3 — Raspberry Pi: Python MQTT Client

### 1. Install the Python MQTT library

```bash
pip3 install paho-mqtt
```

### 2. Create the chat client script

```bash
nano pi_chat.py
```

```python
import paho.mqtt.client as mqtt
import threading

BROKER = "localhost"
PORT = 1883
TOPIC_PUBLISH = "chat/pi"
TOPIC_SUBSCRIBE = "chat/nodemcu"

def on_connect(client, userdata, flags, rc):
    print(f"Connected to broker with result code {rc}")
    client.subscribe(TOPIC_SUBSCRIBE)

def on_message(client, userdata, msg):
    print(f"\n[NodeMCU] {msg.payload.decode()}")
    print("You: ", end="", flush=True)

client = mqtt.Client()
client.on_connect = on_connect
client.on_message = on_message

client.connect(BROKER, PORT, 60)
client.loop_start()

print("MQTT chat started. Type a message and press Enter to send. Ctrl+C to quit.")
try:
    while True:
        message = input("You: ")
        client.publish(TOPIC_PUBLISH, message)
except KeyboardInterrupt:
    print("\nExiting chat.")
    client.loop_stop()
    client.disconnect()
```

Save with `Ctrl + X`, then `Y`, then `Enter`.

### 3. Run the chat client

```bash
python3 pi_chat.py
```

Anything you type in this terminal is published to `chat/pi` and shows up on the NodeMCU's serial monitor. Anything the NodeMCU publishes to `chat/nodemcu` appears in this terminal.

---

## Testing the Full Chat

1. Power on the NodeMCU with the sketch uploaded — check the Serial Monitor for `"WiFi connected"` and `"connected"` (to MQTT).
2. Run `python3 pi_chat.py` on the Pi.
3. Type a message on the Pi — it should appear on the NodeMCU's Serial Monitor.
4. Wait for the NodeMCU's periodic heartbeat message — it should appear in the Pi's terminal.

## Troubleshooting

| Issue | Likely Cause |
|---|---|
| NodeMCU won't connect to WiFi | Wrong SSID/password, or ESP8266 doesn't support 5GHz networks |
| NodeMCU can't reach the broker | Wrong `mqtt_server` IP, or `allow_anonymous true` / firewall not configured |
| No messages appear on either side | Topic names don't match, or the broker isn't running (`sudo systemctl status mosquitto`) |
| `Connection refused` error | Mosquitto is only listening on localhost — check the `listener 1883` line in the config |

## Notes

- `allow_anonymous true` is fine for testing on a private local network but should not be used in production — switch to username/password (or TLS) authentication before deploying.
- The Pi's IP address can change if it's using DHCP; consider setting a static IP or using mDNS (`raspberrypi.local`) for a more stable setup.
