+++
title = "Wireless Neopixels With Esp32 and Artnet"
date = "2022-04-03T20:09:44+02:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = ["esp32","art-net","neopixels"]
+++

Here's how to set up ESP32 devices as a wireless access point, Art-Net server and Art-Net Nodes, to control neo-pixels wirelessly with DMX.

These are the devices we'll be coding:

* [ESP32 Access Point](#esp32-access-point) - creates the Art-Net wireless newtork.
* [ESP32 Art-Net Node](#esp32-art-net-node) - connects to the ESP32 Access Point and receves DMX data, which it sents to connected neopixels.
* [ESP32 Art-Net Server](#esp32-art-net-server) - connects to the ESP32 Access Point and sending DMX data to the ESP32 Nodes.

ESP32 Access Point
------------------

This is the easiest part.  ESP32 has an access point mode called SoftAP (SoftAP) which basically makes it create a wireless network, automatically assign IP addresses and direct
newtork traffic between connected devices as necessary.

```
// esp32ArtNetAP.ino

#include <WiFi.h>
#include <WebServer.h>
#include "esp_wifi.h"

const char *ssid = "Art-Net";
const char *passphrase = "esp32artnetAP";

IPAddress local_IP(2,0,0,1);
IPAddress gateway(2,0,0,1);
IPAddress subnet(255,0,0,0);

WebServer server(80);

void handleRoot() {
  wifi_sta_list_t wifi_sta_list;
  tcpip_adapter_sta_list_t adapter_sta_list;
  memset(&wifi_sta_list, 0, sizeof(wifi_sta_list));
  memset(&adapter_sta_list, 0, sizeof(adapter_sta_list));

  esp_wifi_ap_get_sta_list(&wifi_sta_list);
  tcpip_adapter_get_sta_list(&wifi_sta_list, &adapter_sta_list);

  char html[512] = "Hello from ESP32";

  for (int i=0; i < adapter_sta_list.num; i++) {
    tcpip_adapter_sta_info_t station = adapter_sta_list.sta[i];
    char ipBuffer[16];
    sprintf(ipBuffer, IPSTR, IP2STR(&(station.ip)));
    sprintf(html, "%s\nNode IP %s" , html, ipBuffer);
  }
  server.send(200, "text/plain", html);
}

void setup()
{
  Serial.begin(115200);
  Serial.println();

  Serial.print("Setting soft-AP configuration ... ");
  WiFi.mode(WIFI_AP_STA);
  Serial.println(WiFi.softAPConfig(local_IP, gateway, subnet) ? "Ready" : "Failed!");

  Serial.print("Setting soft-AP ... ");
  Serial.println(WiFi.softAP(ssid,passphrase) ? "Ready" : "Failed!");

  Serial.print("Soft-AP IP address = ");
  Serial.println(WiFi.softAPIP());

  server.on("/", handleRoot);
  server.begin();
  Serial.println("HTTP server started");
}

void loop() {
  server.handleClient();
  delay(2);
}
```

ESP32 Art-Net Node
------------------

The ESP32 Art-Net Node has some neo-pixels connected to pin 2. It connects the to ESP32 Access Point and receives DMX data in the form of individual R,G,B values for each
connected neopixel which is then sends to the connected neopixels.

```
// esp32ArtNetNode.io

#include <WiFi.h>
#include <WiFiUdp.h>
#include <ArtnetWifi.h>
#include <FastLED.h>

//Wifi settings - be sure to replace these with the WiFi network that your computer is connected to

const char* ssid = "Art-Net";
const char* password = "esp32artnetAP";

// LED Strip
const int numLeds = 61; // Change if your setup has more or less LED's
const int numberOfChannels = numLeds * 3; // Total number of DMX channels you want to receive (1 led = 3 channels)
#define DATA_PIN 2 //The data pin that the WS2812 strips are connected to.
CRGB leds[numLeds];

// Artnet settings
ArtnetWifi artnet;
const int startUniverse = 0;

bool sendFrame = 1;
int previousDataLength = 0;

// connect to wifi – returns true if successful or false if not
boolean ConnectWifi(void)
{
  boolean state = true;
  int i = 0;

  WiFi.begin(ssid, password);
  Serial.println("");
  Serial.println("Connecting to WiFi");

  // Wait for connection
  Serial.print("Connecting");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
    if (i > 20){
      state = false;
      break;
    }
    i++;
  }
  if (state){
    Serial.println("");
    Serial.print("Connected to ");
    Serial.println(ssid);
    Serial.print("IP address: ");
    Serial.println(WiFi.localIP());
  } else {
    Serial.println("");
    Serial.println("Connection failed.");
  }

  return state;
}

void onDmxFrame(uint16_t universe, uint16_t length, uint8_t sequence, uint8_t* data)
{
  sendFrame = 1;
  // set brightness of the whole strip
  if (universe == 15)
  {
    FastLED.setBrightness(data[0]);
  }
  // read universe and put into the right part of the display buffer
  for (int i = 0; i < length / 3; i++)
  {
    int led = i + (universe - startUniverse) * (previousDataLength / 3);
    if (led < numLeds)
    {
      leds[led] = CRGB(data[i * 3], data[i * 3 + 1], data[i * 3 + 2]);
    }
  }
  previousDataLength = length;
  FastLED.show();
}

void setup()
{
  Serial.begin(115200);
  ConnectWifi();
  artnet.begin();
  FastLED.addLeds<WS2812, DATA_PIN, GRB>(leds, numLeds);

  // onDmxFrame will execute every time a packet is received by the ESP32
  artnet.setArtDmxCallback(onDmxFrame);
}

void loop()
{
  // we call the read function inside the loop
  artnet.read();
}
```

ESP32 Art-Net Server
--------------------

The ESP32 Art-Net Server can be configured to send RGB Art-Net DMX data to one or more nodes of varying length neo-pixels.  Pin 4 is set up as a rising interrupt pin, so an audio or vibration sensor (or any other sensor that will rise when activate) can be connected to pin 4, and this rising interrupt will trigger a change in colour of all the ESP32 Art-Net Node neopixels.

```
// esp32ArtNetServer.ino

/*
This example will transmit a universe via Art-Net into the Network.
This example may be copied under the terms of the MIT license, see the LICENSE file for details
*/
#include <ArtnetWifi.h>
#include <Arduino.h>
#include <map>

//Wifi settings
const char* ssid = "Art-Net"; // CHANGE FOR YOUR SETUP
const char* password = "esp32artnetAP"; // CHANGE FOR YOUR SETUP

// Artnet settings
ArtnetWifi artnet0;
ArtnetWifi artnet1;

const int startUniverse = 0; // CHANGE FOR YOUR SETUP most software this is 1, some software send out artnet first universe as 0.
const char host0[] = "2.0.0.2"; // CHANGE FOR YOUR SETUP your destination
const char host1[] = "2.0.0.3"; // CHANGE FOR YOUR SETUP your destination

struct RGB {
    byte r;
    byte g;
    byte b;
};

RGB defaultColours[] = {
  {255,0,0},
  {0,255,0},
  {0,0,255},
  {255, 255, 0},
  {0,255, 255}
};

RGB currentColour = {0,0,0};
bool bChangeColour = true;
int artnetRefreshIntervalMS = 40;
int artnetLastRefreshTime = millis();
int microphoneInterruptPin = 4;

void IRAM_ATTR isr() {
  bChangeColour = true;
}

// connect to wifi – returns true if successful or false if not
boolean ConnectWifi(void)
{
  boolean state = true;
  int i = 0;

  WiFi.begin(ssid, password);
  Serial.println("");
  Serial.println("Connecting to WiFi");

  // Wait for connection
  Serial.print("Connecting");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
    if (i > 20){
      state = false;
      break;
    }
    i++;
  }
  if (state){
    Serial.println("");
    Serial.print("Connected to ");
    Serial.println(ssid);
    Serial.print("IP address: ");
    Serial.println(WiFi.localIP());
  } else {
    Serial.println("");
    Serial.println("Connection failed.");
  }

  return state;
}

void changeColour() {
  currentColour = defaultColours[random(sizeof(defaultColours))];

  for (int i=0; i<61; i++) {
    artnet0.setByte(i*3+0, currentColour.r);
    artnet0.setByte(i*3+1, currentColour.g);
    artnet0.setByte(i*3+2, currentColour.b);
  }
  for (int i=0; i<30; i++) {
    artnet1.setByte(i*3+0, currentColour.r);
    artnet1.setByte(i*3+1, currentColour.g);
    artnet1.setByte(i*3+2, currentColour.b);
  }
}

void setup()
{
  Serial.begin(115200);
  pinMode(microphoneInterruptPin, INPUT);
  attachInterrupt(microphoneInterruptPin, isr, RISING);
  ConnectWifi();
  artnet0.begin(host0);
  artnet0.setLength(3*61);
  artnet0.setUniverse(startUniverse);
  artnet1.begin(host1);
  artnet1.setLength(3*30);
  artnet1.setUniverse(startUniverse);
}

void loop() {
  if (bChangeColour) {
    Serial.println("Changing Colour");
    changeColour();
    bChangeColour = false;
  }
  if (millis() - artnetLastRefreshTime >= artnetRefreshIntervalMS) {
    artnet1.write();
    artnet0.write();
    artnetLastRefreshTime = millis();
  }

  delay(1);
}
```
