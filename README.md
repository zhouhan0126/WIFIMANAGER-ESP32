# WiFiManager
ESP32&&ESP8266 WiFi Connection manager with fallback web configuration portal


The configuration portal is of the captive variety, so on various devices it will present the configuration dialogue as soon as you connect to the created access point.

First attempt at a library. Lots more changes and fixes to do. Contributions are welcome.

## Libray
[WebServer  https://github.com/zhouhan0126/WebServer-esp32](https://github.com/zhouhan0126/WebServer-esp32)

[DNSServer   https://github.com/zhouhan0126/DNSServer---esp32](https://github.com/zhouhan0126/DNSServer---esp32)

## How It Looks
![ESP8266 WiFi Captive Portal Homepage](http://i.imgur.com/YPvW9eql.png) ![ESP8266 WiFi Captive Portal Configuration](http://i.imgur.com/oicWJ4gl.png)


### Using
- Include in your sketch
```cpp
#if defined(ESP8266)
#include <ESP8266WiFi.h>          
#else
#include <WiFi.h>          
#endif

//needed for library
#include <DNSServer.h>
#if defined(ESP8266)
#include <ESP8266WebServer.h>
#else
#include <WebServer.h>
#endif
#include <WiFiManager.h>         

```

- Initialize library, in your setup function add
```cpp
WiFiManager wifiManager;
```

- Also in the setup function add
```cpp
//first parameter is name of access point, second is the password
wifiManager.autoConnect("AP-NAME", "AP-PASSWORD");
```
if you just want an unsecured access point
```cpp
wifiManager.autoConnect("AP-NAME");
```
or if you want to use and auto generated name from 'ESP' and the esp's Chip ID use
```cpp
wifiManager.autoConnect();
```

After you write your sketch and start the ESP, it will try to connect to WiFi. If it fails it starts in Access Point mode.
While in AP mode, connect to it then open a browser to the gateway IP, default 192.168.4.1, configure wifi, save and it should reboot and connect.

Also see [examples](https://github.com/tzapu/WiFiManager/tree/master/examples).

## Documentation

#### Password protect the configuration Access Point
You can and should password protect the configuration access point.  Simply add the password as a second parameter to `autoConnect`.
A short password seems to have unpredictable results so use one that's around 8 characters or more in length.
The guidelines are that a wifi password must consist of 8 to 63 ASCII-encoded characters in the range of 32 to 126 (decimal)
```cpp
wifiManager.autoConnect("AutoConnectAP", "password")
```

#### Callbacks
##### Enter Config mode
Use this if you need to do something when your device enters configuration mode on failed WiFi connection attempt.
Before `autoConnect()`
```cpp
wifiManager.setAPCallback(configModeCallback);
```
`configModeCallback` declaration and example
```cpp
void configModeCallback (WiFiManager *myWiFiManager) {
  Serial.println("Entered config mode");
  Serial.println(WiFi.softAPIP());

  Serial.println(myWiFiManager->getConfigPortalSSID());
}
```

##### Save settings
This gets called when custom parameters have been set **AND** a connection has been established. Use it to set a flag, so when all the configuration finishes, you can save the extra parameters somewhere.

See [AutoConnectWithFSParameters Example](https://github.com/tzapu/WiFiManager/tree/master/examples/AutoConnectWithFSParameters).
```cpp
wifiManager.setSaveConfigCallback(saveConfigCallback);
```
`saveConfigCallback` declaration and example
```cpp
//flag for saving data
bool shouldSaveConfig = false;

//callback notifying us of the need to save config
void saveConfigCallback () {
  Serial.println("Should save config");
  shouldSaveConfig = true;
}
```

#### Configuration Portal Timeout
If you need to set a timeout so the ESP doesn't hang waiting to be configured, for instance after a power failure, you can add
```cpp
wifiManager.setConfigPortalTimeout(180);
```
which will wait 3 minutes (180 seconds). When the time passes, the autoConnect function will return, no matter the outcome.
Check for connection and if it's still not established do whatever is needed (on some modules I restart them to retry, on others I enter deep sleep)

#### On Demand Configuration Portal
If you would rather start the configuration portal on demand rather than automatically on a failed connection attempt, then this is for you.

Instead of calling `autoConnect()` which does all the connecting and failover configuration portal setup for you, you need to use `startConfigPortal()`. __Do not use BOTH.__

Example usage
```cpp
void loop() {
  // is configuration portal requested?
  if ( digitalRead(TRIGGER_PIN) == LOW ) {
    WiFiManager wifiManager;
    wifiManager.startConfigPortal("OnDemandAP");
    Serial.println("connected...yeey :)");
  }
}
```
See example for a more complex version. [OnDemandConfigPortal](https://github.com/tzapu/WiFiManager/tree/master/examples/OnDemandConfigPortal)

#### Custom Parameters
You can use WiFiManager to collect more parameters than just SSID and password.
This could be helpful for configuring stuff like MQTT host and port, [blynk](http://www.blynk.cc) or [emoncms](http://emoncms.org) tokens, just to name a few.
**You are responsible for saving and loading these custom values.** The library just collects and displays the data for you as a convenience.
Usage scenario would be:
- load values from somewhere (EEPROM/FS) or generate some defaults
- add the custom parameters to WiFiManager using
 ```cpp
 // id/name, placeholder/prompt, default, length
 WiFiManagerParameter custom_mqtt_server("server", "mqtt server", mqtt_server, 40);
 wifiManager.addParameter(&custom_mqtt_server);

 ```
- if connection to AP fails, configuration portal starts and you can set /change the values (or use on demand configuration portal)
- once configuration is done and connection is established [save config callback]() is called
- once WiFiManager returns control to your application, read and save the new values using the `WiFiManagerParameter` object.
 ```cpp
 mqtt_server = custom_mqtt_server.getValue();
 ```  
This feature is a lot more involved than all the others, so here are some examples to fully show how it is done.
You should also take a look at adding custom HTML to your form.

- Save and load custom parameters to file system in json form [AutoConnectWithFSParameters](https://github.com/tzapu/WiFiManager/tree/master/examples/AutoConnectWithFSParameters)
- *Save and load custom parameters to EEPROM* (not done yet)

#### Custom IP Configuration
You can set a custom IP for both AP (access point, config mode) and STA (station mode, client mode, normal project state)

##### Custom Access Point IP Configuration
This will set your captive portal to a specific IP should you need/want such a feature. Add the following snippet before `autoConnect()`
```cpp
//set custom ip for portal
wifiManager.setAPStaticIPConfig(IPAddress(10,0,1,1), IPAddress(10,0,1,1), IPAddress(255,255,255,0));
```

##### Custom Station (client) Static IP Configuration
This will make use the specified IP configuration instead of using DHCP in station mode.
```cpp
wifiManager.setSTAStaticIPConfig(IPAddress(192,168,0,99), IPAddress(192,168,0,1), IPAddress(255,255,255,0));
```
There are a couple of examples in the examples folder that show you how to set a static IP and even how to configure it through the web configuration portal.

#### Custom HTML, CSS, Javascript
There are various ways in which you can inject custom HTML, CSS or Javascript into the configuration portal.
The options are:
- inject custom head element
You can use this to any html bit to the head of the configuration portal. If you add a `<style>` element, bare in mind it overwrites the included css, not replaces.
```cpp
wifiManager.setCustomHeadElement("<style>html{filter: invert(100%); -webkit-filter: invert(100%);}</style>");
```
- inject a custom bit of html in the configuration form
```cpp
WiFiManagerParameter custom_text("<p>This is just a text paragraph</p>");
wifiManager.addParameter(&custom_text);
```
- inject a custom bit of html in a configuration form element
Just add the bit you want added as the last parameter to the custom parameter constructor.
```cpp
WiFiManagerParameter custom_mqtt_server("server", "mqtt server", "iot.eclipse", 40, " readonly");
```

#### Filter Networks
You can filter networks based on signal quality and show/hide duplicate networks.

- If you would like to filter low signal quality networks you can tell WiFiManager to not show networks below an arbitrary quality %;
```cpp
wifiManager.setMinimumSignalQuality(10);
```
will not show networks under 10% signal quality. If you omit the parameter it defaults to 8%;

- You can also remove or show duplicate networks (default is remove).
Use this function to show (or hide) all networks.
```cpp
wifiManager.setRemoveDuplicateAPs(false);
```

#### Debug
Debug is enabled by default on Serial. To disable add before autoConnect
```cpp
wifiManager.setDebugOutput(false);
```


### Contributions and thanks
The support and help I got from the community has been nothing short of phenomenal. I can't thank you guys enough. This is my first real attept in developing open source stuff and I must say, now I understand why people are so dedicated to it, it is because of all the wonderful people involved.

__THANK YOU__

[Shawn A](https://github.com/tablatronix)

[Maximiliano Duarte](https://github.com/domonetic)

[alltheblinkythings](https://github.com/alltheblinkythings)

[Niklas Wall](https://github.com/niklaswall)

[Jakub Piasecki](https://github.com/zaporylie)

[Peter Allan](https://github.com/alwynallan)

[John Little](https://github.com/j0hnlittle)

[markaswift](https://github.com/markaswift)

[franklinvv](https://github.com/franklinvv)

[Alberto Ricci Bitti](https://github.com/riccibitti)

[SebiPanther](https://github.com/SebiPanther)

[jonathanendersby](https://github.com/jonathanendersby)

[walthercarsten](https://github.com/walthercarsten)

Sorry if i have missed anyone.

#### Inspiration
- http://www.esp8266.com/viewtopic.php?f=29&t=2520
