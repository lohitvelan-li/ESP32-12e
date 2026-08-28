#include <ESP8266WiFi.h>
#include <PubSubClient.h>

// ---------- CONFIGURATION ----------
const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";

const char* mqtt_server = "PI_IP_ADDRESS"; // e.g. "192.168.1.50"
const int mqtt_port = 1883;
const char* mqtt_user = ""; // set if you enable mosquitto auth
const char* mqtt_pass = "";

const char* topic_publish   = "chat/nodemcu";
const char* topic_subscribe = "chat/pi";
// -----------------------------------

WiFiClient espClient;
PubSubClient client(espClient);

void setup_wifi() {
  delay(10);
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.begin(ssid, password);
  // Wait for connection
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println();
  Serial.println("WiFi connected");
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());
}

// Called when a subscribed message arrives
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
  // Loop until we're reconnected
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    String clientId = "NodeMCU-" + String(ESP.getChipId());

    bool connected;
    if (mqtt_user && strlen(mqtt_user) > 0) {
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
      Serial.println(" - retrying in 5s");
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
#includes: ESP8266WiFi.h handles Wi‑Fi; PubSubClient.h handles MQTT client logic.
Configuration block: set your Wi‑Fi SSID/password and mqtt_server to the Pi’s IP. If you enable Mosquitto authentication, set mqtt_user and mqtt_pass.
setup_wifi(): connects the ESP to your Wi‑Fi network and prints the assigned IP to Serial — useful to verify connectivity.
callback(): called by PubSubClient when a message arrives on any subscribed topic; it prints the topic and payload to Serial, so the Pi sees these messages in its terminal and you see NodeMCU messages on Serial.
reconnect(): ensures there is an MQTT connection. If authentication is provided it uses username/password; otherwise it connects anonymously. On success it subscribes to the Pi topic and announces “NodeMCU is online”.
setup(): starts Serial, connects Wi‑Fi, sets the MQTT server and callback.
loop(): keeps the MQTT client alive (client.loop()) and publishes a simple heartbeat every 10 seconds.
import paho.mqtt.client as mqtt
import threading

BROKER = "localhost"   # if broker runs on the same Pi
PORT = 1883
TOPIC_PUBLISH = "chat/pi"
TOPIC_SUBSCRIBE = "chat/nodemcu"

def on_connect(client, userdata, flags, rc):
    print(f"Connected to broker (rc={rc})")
    client.subscribe(TOPIC_SUBSCRIBE)

def on_message(client, userdata, msg):
    # Print incoming messages from NodeMCU
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
    paho.mqtt.client: standard MQTT client for Python.
BROKER: "localhost" because Mosquitto runs on the Pi. If Mosquitto is on another host use its IP.
TOPIC_PUBLISH / TOPIC_SUBSCRIBE: the Pi publishes to chat/pi and subscribes to chat/nodemcu to receive NodeMCU messages.
on_connect(): called when the client connects; subscribes to the NodeMCU topic.
on_message(): called when a message arrives; prints NodeMCU messages and then reprints the prompt.
client.loop_start(): runs the MQTT network loop in a background thread so the main thread can block on input() to send messages.
The main loop reads user input and publishes it. KeyboardInterrupt cleanly stops the loop and disconnects.
listener 1883: tells Mosquitto to listen on the standard MQTT port on all interfaces (not just localhost).
allow_anonymous true: allows clients to connect without credentials — convenient for local testing only. For security, set to false and create users with mosquitto_passwd.
Commands to run on the Raspberry Pi (step-by-step)

Update and install Mosquitto:
sudo apt update && sudo apt upgrade -y
sudo apt install -y mosquitto mosquitto-clients
Create the config file so Mosquitto listens on the network:
sudo nano /etc/mosquitto/conf.d/default.conf (paste the config above)
sudo systemctl restart mosquitto
sudo systemctl enable mosquitto
Verify broker is running:
sudo systemctl status mosquitto
(Optional) If you enable authentication:
sudo mosquitto_passwd -c /etc/mosquitto/passwd pi_user
edit the config and add: password_file /etc/mosquitto/passwd
restart mosquitto: sudo systemctl restart mosquitto
Install Python MQTT client:
pip3 install paho-mqtt
Quick broker test (on Pi)

Terminal A:
mosquitto_sub -h localhost -t "chat/test"
Terminal B:
mosquitto_pub -h localhost -t "chat/test" -m "Hello broker" You should see "Hello broker" in Terminal A.
