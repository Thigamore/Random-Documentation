---
title: ESP Async Web Server Documentation
shortTitle: Async Web Server Docs # Max 31 characters
intro: 'Some documentation and a guide on how to use the ESPAsyncWebServer library.'
---

# Introduction

First of all, thank you for attending the esp32 workshop!  
The following will be an appendix for the workshop which will go through each part of the code in more detail and outline some other important concepts so you can work with the ESP32 with confidence.  
The guide assumes that you have a working ESP32 Wroom connected to your computer and the arduino IDE installed, which can be downloaded at [arduino.cc](https://www.arduino.cc/en/software).
  
## TOC
- [Introduction](#introduction)
  - [TOC](#toc)
- [Setup](#setup)
  - [Install ESP32 Board](#install-esp32-board)
  - [Install the Libraries](#install-the-libraries)
  - [Connect to the ESP32](#connect-to-the-esp32)
- [LED Demo](#led-demo)
  - [Blink Demo](#blink-demo)
  - [Setting up WebServer](#setting-up-webserver)
  - [HTML and JS for the LED](#html-and-js-for-the-led)
  - [Final Code](#final-code)
- [Appendix](#appendix)
  - [References](#references)
  - [Important Websocket Concepts](#important-websocket-concepts)
    - [Introduction](#introduction-1)
    - [Outline of Websockets](#outline-of-websockets)
    - [Opening Handshake](#opening-handshake)
    - [Data sending/receiving](#data-sendingreceiving)
      - [Frame Structure](#frame-structure)
      - [Frame Fragmentation](#frame-fragmentation)
      - [Ping Pong](#ping-pong)
      - [Masking](#masking)
    - [Closing Handshake](#closing-handshake)
    - [Pros and Cons of Websockets](#pros-and-cons-of-websockets)
  - [Coding Reference](#coding-reference)
    - [Aliases](#aliases)
      - [ArRequstHandlerFunction](#arrequsthandlerfunction)
      - [ArUploadHandlerFunction](#aruploadhandlerfunction)
      - [ArBodyHandlerFunction](#arbodyhandlerfunction)
      - [AwsTemplateProcessor](#awstemplateprocessor)
    - [AwsEventHandler](#awseventhandler)
    - [Enums](#enums)
      - [WebRequestMethod](#webrequestmethod)
      - [SendStatus](#sendstatus)
      - [AwsFrameType](#awsframetype)
      - [AwsEventType](#awseventtype)
    - [Classes/Structs](#classesstructs)
      - [AsyncWebServer](#asyncwebserver)
      - [AsyncWebSocketClient](#asyncwebsocketclient)
      - [AsyncWebSocket](#asyncwebsocket)
      - [AsyncWebServerRequest](#asyncwebserverrequest)
      - [AwsFrameInfo](#awsframeinfo)

# Setup

This section will guide you on how to setup your ESP32 so that you can use it with your computer.    
  
## Install ESP32 Board
First we have to install the board to ensure that our computer can communicate with the ESP32. 
1. Go to preferences.
2. Click the button next to *Additional boards manager URLS*.
3. In a new line, paste `https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`
4. On the left hand side, click on the board manager icon (should be the second one from the top).
5. Search *ESP32* and install *esp32 by Espressif Systems*.

## Install the Libraries
Now that we have installed the board for the ESP32, we have to install the libraries that we will be using.  
1. On the left hand side, click the libraries icon (should be the third one from the top and looks like a row of boooks).
2. Search *ESPAsyncWebServer* and install the one by *ESP32Async*.
3. Search *ESPAsyncTCP* and install the one by *ESP32Async*.

## Connect to the ESP32
Finally, we have installed all we need to make a webserver on the ESP32. All that's left is actually connecting the computer to the ESP32.
1. Plug in the ESP32 into the computer.
2. Click on the connection at the top of the IDE and click *Select other board or port...*
3. For the board, search and select *uPesy ESP32 Wroom DevKit*.
4. For the Port, if you are on Windows, select the Com with USB next to it. If you are on Mac, select the Com with ttl/usb next to it.

Now the ESP32 is connected to our computer with all the proper software. All that's left is to start coding!

# LED Demo
In this section, we will be running a simple web server to turn on and off an LED using a website. During this section, the various concepts of Web servers and Web sockets will be introduced in a project based environment. If you need any more information about Webservers or sockets look at the Appendix for extra resources.

## Blink Demo
Before we even start, let's just test that the ESP32 works and send it a basic blink demo.  
First, wire the ESP32 as you see below.  

<img src="LED-diagram.png" width=400rem>  

Then, copy paste the code below into the arduino ide and upload it.
```cpp
int LED_BUILTIN = 12;

void setup() {
    pinMode (LED_BUILTIN, OUTPUT);
}

void loop() {
    digitalWrite(LED_BUILTIN, HIGH);
    delay(1000);
    digitalWrite(LED_BUILTIN, LOW);
    delay(1000);
}
```
Now you should have a blinking LED on your breadboard.

## Setting up WebServer
Before writing the actual server code, we need some basic code to setup the LED and the connection with the ESP32.
```cpp
// Imports needed 
#include <WiFi.h>
#include <AsyncTCP.h>
#include <ESPAsyncWebServer.h>

// Set up the LED state (whether the led is on or off) and the GPIO pin for the LED
bool ledState = 0;
const int ledPin = 12;

// A default function that runs once when the MCU first runs the code
void setup() {
    // Starts the Serial connection between the ESP and computer at 115200Hz
    Serial.begin(115200);

    // Set that the LED pin is for output
    pinMode(ledPin, OUTPUT);
    // Set the output of the LED pin to low
    digitalWrite(ledPin, LOW);
}

// A default function that runs continuously after the setup function until the MCU running
void loop() {
}
```
Now onto the basic server code.
```cpp
...
// Stores the wifi name. Replace <Wifi-Name> with you wifi's name. DO NOT USE UofT WIFI! It won't work, use a hotspot.
const char *ssid = "<Wifi-Name>";

// Stores the wifi password. Replace <Password> with your wifi's password
const char *password = "<Password>";

// The actual web server and the socket that goes along with it
AsyncWebServer server(80);
AsyncWebSocket ws("/ws");

void setup() {
    ...

    // Prints what connection the server is connecting to 
    Serial.println("Connecting to ");
    Serial.println(ssid);

    // Start the wifi connection with the wifi name and password
    Wifi.begin(ssid, password);

    // Wait until we are connected to the wifi
    while(WiFi.status() != WL_CONNECTED) {
        delay(1000);
        Serial.print(".");
    }

    // Let the user know that the server is connected to the wifi and the ip address of the server
    Serial.println("");
    Serial.println("Connected..!");
    Serial.print("Got IP: ");
    Serial.println(WiFi.localIP());
    ...
}
...
```
You can see more information about the [AsyncWebServer](#asyncwebserver) and [AsyncWebSocket](#asyncwebsocket) in the appendix.  
Next, we will start handling requests that the server is receiving.
```cpp
...

// Handles the messages that a web socket receives
void handleWebSocketMessage(void *arg, uint8_t *data, size_t len) {

    // Stores the information being sent
    // More information about what a frame is and what it stores is in the appendix
    AwsFrameInfo *info = (AwsFrameInfo *)arg;
    // Checks to make sure that the information sent is text
    if (info->final && info->index == 0 && info->len == len && info->opcode == WS_TEXT) {
      data[len] = 0;

      // Checks to see if the data sent is the toggle word
      if (strcmp((char *)data, "toggle") == 0) {
        // Updates LED state
        ledState = !ledState;

        // Sends a message back to the client
        ws.textAll(String(ledState));
      }
    }
}

// Processes the HTML (that will be mentioned in the next section) and replaces the variables with % around them with the String returned (Example: %STATE% will be replaced with ON or OFF) 
String processor(const String &var) {
    if (var == "STATE") {
        return ledState ? "ON" : "OFF";
    }
    if (var == "CHECK") {
        return ledState ? "checked" : "";
    }
    return String();
}


// A function that deals with events that are coming into the web socket
void eventHandler(AsyncWebSocket *server, AsyncWebSocketClient *client, AwsEventType type, void *arg, uint8_t *data, size_t len) {

    // Deals with different types of events
  switch (type) {

    // A connection event
    case WS_EVT_CONNECT:
        Serial.printf("WebSocket client #%u connected from %s\n", client->id(), client->remoteIP().toString().c_str());
        break;

    // A disconnect event
    case WS_EVT_DISCONNECT:
        Serial.printf("WebSocket client #%u disconnected\n", client->id());
        break;

    // An event where data is being sent
    case WS_EVT_DATA:
        // Handle the data sent from the web socket in another function and update the LED's state
        handleWebSocketMessage(arg, data, len);
        digitalWrite(ledPin, ledState);
        break;
  }
}

void setup() {
    ...
    // Sets up the handler for the web socket
    ws.onEvent(eventHandler);
    // Sets up the handler for the server
    server.addHandler(&ws);

    // Routes / (the default adress of the website) to a specific page (index_html) after being processed (processor) with a success error code (200)
    server.on("/", HTTP_GET, [](AsyncWebServerRequest *request) {
        request->send_P(200, "text/html", index_html, processor);
    });

    // Starts the server
    server.begin();
}

void loop() {
    // Removes any connections that are closed
    ws.cleanupClients();
}
...
```
A lot of this code is probably very confusing, which is normal as there is a lot of background knowledge about networking, severs, and websockets needed. However, start by going through the setup function and for anything that you don't understand, look at the appendix.

## HTML and JS for the LED
Finally, we will add the index_html that was in the previous code:
```html
const char index_html[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html>
  <head>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>ESP32 WebSocket Server</title>
    <style>
    html{font-family: Helvetica; display: inline-block; margin: 0px auto; text-align: center;}
    body{margin-top: 50px;}
    h1{color: #444444;margin: 50px auto;}
    p{font-size: 19px;color: #888;}
    #state{font-weight: bold;color: #444;}
    .switch{margin:25px auto;width:80px}
    .toggle{display:none}
    .toggle+label{display:block;position:relative;cursor:pointer;outline:0;user-select:none;padding:2px;width:80px;height:40px;background-color:#ddd;border-radius:40px}
    .toggle+label:before,.toggle+label:after{display:block;position:absolute;top:1px;left:1px;bottom:1px;content:""}
    .toggle+label:before{right:1px;background-color:#f1f1f1;border-radius:40px;transition:background .4s}
    .toggle+label:after{width:40px;background-color:#fff;border-radius:20px;box-shadow:0 2px 5px rgba(0,0,0,.3);transition:margin .4s}
    .toggle:checked+label:before{background-color:#4285f4}
    .toggle:checked+label:after{margin-left:42px}
    </style>
  </head>
  <body>
    <h1>ESP32 WebSocket Server</h1>
    <div class="switch">
      <input id="toggle-btn" class="toggle" type="checkbox" %CHECK%>
      <label for="toggle-btn"></label>
    </div>
    <p>On-board LED: <span id="state">%STATE%</span></p>
  </body>
</html>
<!----------------------- THE IMPORTANT PART -------------------->
<script>
    // We add an event listener so that when the website first loads, the websocket is started with the server
	  window.addEventListener('load', function() {

        // Initialize the server with its URI (remember the "/ws" we put in the websocket's constructor in the esp32 code)
		var websocket = new WebSocket(`ws://${window.location.hostname}/ws`);
        
        // Action the websocket will take when the connection is opened
		websocket.onopen = function(event) {
		  console.log('Connection established');
		}

        // Action the websocket will take when the connection is closed
		websocket.onclose = function(event) {
		  console.log('Connection died');
		}

        // Action the websocket will take when there is an error
		websocket.onerror = function(error) {
		  console.log('error');
		};

        // Action the websocket will take when it receives a message
		websocket.onmessage = function(event) {
            // Checks what data was send (the current state of the LED) and flips it
		    if (event.data == "1") {
			    document.getElementById('state').innerHTML = "ON";
			    document.getElementById('toggle-btn').checked = true;
		    }
		    else {
			    document.getElementById('state').innerHTML = "OFF";
			    document.getElementById('toggle-btn').checked = false;
		    }
		};
		
        // Makes the toggle button send a message to the server to toggle
		document.getElementById('toggle-btn').addEventListener('change', function() { websocket.send('toggle'); });
	  });
	</script>
)rawliteral";
```
This tutorial is not meant to teach you HTML or JavaScript and as such there will not be too much detail on this. However, you can find more information on the websocket API at [MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API).

## Final Code
Here's the final code:
```cpp
#include <WiFi.h>
#include <AsyncTCP.h>
#include <ESPAsyncWebServer.h>

bool ledState = 0;
const int ledPin = 12;

const char* ssid = "<Wifi-Name>";
const char* password = "<Password-name>";

AsyncWebServer server(80);
AsyncWebSocket ws("/ws");

const char index_html[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html>
  <head>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>ESP32 WebSocket Server</title>
    <style>
    html{font-family: Helvetica; display: inline-block; margin: 0px auto; text-align: center;}
    body{margin-top: 50px;}
    h1{color: #444444;margin: 50px auto;}
    p{font-size: 19px;color: #888;}
    #state{font-weight: bold;color: #444;}
    .switch{margin:25px auto;width:80px}
    .toggle{display:none}
    .toggle+label{display:block;position:relative;cursor:pointer;outline:0;user-select:none;padding:2px;width:80px;height:40px;background-color:#ddd;border-radius:40px}
    .toggle+label:before,.toggle+label:after{display:block;position:absolute;top:1px;left:1px;bottom:1px;content:""}
    .toggle+label:before{right:1px;background-color:#f1f1f1;border-radius:40px;transition:background .4s}
    .toggle+label:after{width:40px;background-color:#fff;border-radius:20px;box-shadow:0 2px 5px rgba(0,0,0,.3);transition:margin .4s}
    .toggle:checked+label:before{background-color:#4285f4}
    .toggle:checked+label:after{margin-left:42px}
    </style>
  </head>
  <body>
    <h1>ESP32 WebSocket Server</h1>
    <div class="switch">
      <input id="toggle-btn" class="toggle" type="checkbox" %CHECK%>
      <label for="toggle-btn"></label>
    </div>
    <p>On-board LED: <span id="state">%STATE%</span></p>
  </body>
</html>

<script>
	  window.addEventListener('load', function() {
		var websocket = new WebSocket(`ws://${window.location.hostname}/ws`);
		websocket.onopen = function(event) {
		  console.log('Connection established');
		}
		websocket.onclose = function(event) {
		  console.log('Connection died');
		}
		websocket.onerror = function(error) {
		  console.log('error');
		};
		websocket.onmessage = function(event) {
		  if (event.data == "1") {
			document.getElementById('state').innerHTML = "ON";
			document.getElementById('toggle-btn').checked = true;
		  }
		  else {
			document.getElementById('state').innerHTML = "OFF";
			document.getElementById('toggle-btn').checked = false;
		  }
		};
		
		         document.getElementById('toggle-btn').addEventListener('change', function() { websocket.send('toggle'); });
	  });
	</script>

)rawliteral";

void eventHandler(AsyncWebSocket *server, AsyncWebSocketClient *client, AwsEventType type, void *arg, uint8_t *data, size_t len) {
  switch (type) {
    case WS_EVT_CONNECT:
      Serial.printf("WebSocket client #%u connected from %s\n", client->id(), client->remoteIP().toString().c_str());
      break;
    case WS_EVT_DISCONNECT:
      Serial.printf("WebSocket client #%u disconnected\n", client->id());
      break;
    case WS_EVT_DATA:
      handleWebSocketMessage(arg, data, len);
      digitalWrite(ledPin, ledState);
      break;

  }
}

void handleWebSocketMessage(void *arg, uint8_t *data, size_t len) {
  AwsFrameInfo *info = (AwsFrameInfo *)arg;
  if (info->final && info->index == 0 && info->len == len && info->opcode == WS_TEXT) {
    data[len] = 0;
    if (strcmp((char *)data, "toggle") == 0) {
      ledState = !ledState;
      ws.textAll(String(ledState));
    }
  }
}

String processor(const String &var) {
    if (var == "STATE") {
      return ledState ? "ON" : "OFF";
    }
    if (var == "CHECK") {
      return ledState ? "checked" : "";
  }
    return String();
 }

void setup() {
    Serial.begin(115200);
    
    pinMode(ledPin, OUTPUT);
    digitalWrite(ledPin, LOW);

    Serial.print("Connecting to ");
    Serial.println(ssid);

    // Connect to Wi-Fi
    WiFi.begin(ssid, password);

    while (WiFi.status() != WL_CONNECTED) {
      delay(1000);
      Serial.print(".");
    }

    Serial.println("");
    Serial.println("Connected..!");
    Serial.print("Got IP: ");
    Serial.println(WiFi.localIP());

    ws.onEvent(eventHandler);
    server.addHandler(&ws);

    server.on("/", HTTP_GET, [](AsyncWebServerRequest *request) {
    request->send_P(200, "text/html", index_html, processor);
  });

  // Start server
  server.begin();


}

void loop() {
  ws.cleanupClients();
}
```





# Appendix

## References
- [ESPAsyncWebServer Github](https://github.com/ESP32Async/ESPAsyncWebServer)
- [RFC6455](https://www.rfc-editor.org/rfc/pdfrfc/rfc6455.txt.pdf) (The websocket specification)
- [Websocket Wikipedia](https://en.wikipedia.org/wiki/WebSocket#Closing_handshake)
- [Javascript Websocket Reference MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)

## Important Websocket Concepts

### Introduction
Websockets are a protocol using TCP that create a connection between a server and client so that both the server and client can send each other data bidirectionally and reliably.  

### Outline of Websockets
A websocket connection has 3 main states:  
1. Opening handshake
    - The client sends a request to the server to open a websocket connection and the server accepts.
2. Data sending/receiving
    - The client and server are both connected and are ready to either send or receive data from each other.
3. Closing handshake  
    - The client or server sends a request to close the websocket connection.

### Opening Handshake
The client sends an HTTP Upgrade to the server and the server will respond with a 101 response if the websocket will be opened or anything else if not.  
Note that websockets is only available on HTTP/1.1 or greater as HTTP Upgrade is not supported on HTTP/1.0.

### Data sending/receiving
All data is sent in the form of frames with a frame header and message data. The frame header containes information such as the type of message being sent and the length of the message while the message data is the actual data being sent.  

FIN and the opcode are both parts of the header which will be described in the next section.
#### Frame Structure
Some important frame fields and their values is described below:
- FIN:
  - Refers to which part of a fragmented message a message belongs to. See [Frame Fragmentation](#frame-fragmentation).
  - Values:
    - 0 - Not the final frame of a message.
    - 1 - The final frame of a message. 
- Opcode:
  - The type of frame being sent.
  - Values:
    - 0 - A continuation frame of a fragmented message. Should be set if frame is middle or final frame.
    - 1 - A Text Data Frame. Application data stores UTF-8 encoded data.
    - 2 - A binary Data Frame. Application data stores binary data
    - 8 - A close frame. Initiates a closing handshake.
    - 9 - Ping frame. See [Ping Ping](#ping-pong).
    - 10 - Pong frame. See [Ping Ping](#ping-pong).
- Masked:
  - Tells if the message is masked or not.
  - Values:
    - 0 - Frame is not masked.
    - 1 - Frame is masked.
- Payload length:
  - Tells how long the payload is.
  - Values:
    - 0-125: This value is the payload length.
    - 126: The next 16 bits is the payload length.
    - 127: the next 64 bits is the payload length.
- Masking Key:
  - The key to mask the data if need be.
- Payload:
  - Stores the data that is being sent.

#### Frame Fragmentation
Messages can also be split into multiple parts, which is called fragmentation. There are three parts to a fragmented message:  
1. The first frame which will have an opcode $\ne$ 0 and FIN = 0.
2. Any middle frame which will have an opcode = 0 and FIN = 0.
3. The final frame which will have an opcode = 0 and FIN = 1.    

#### Ping Pong
If either member of the connection sends a ping frame, the other member must respond with a pong frame.  
This is commonly used to check latency, or to make sure the connection is still alive.

#### Masking
The client must mask the frames sent to the server while a server must not mask the frames sent to the client. Masking is done through the a xor with the masking key and payload data.

### Closing Handshake
The client or server sends a closing frame. Then the recipient of the closing frame must send a closing frame back to finish the connection.  
The computer that initiated the closing handshake must still accept all data and until it receives a closing frame back. This is to ensure no data is lost due to latency.  
The closing frame can contain data which would be the reason why the connection was closed.


### Pros and Cons of Websockets
The main benefit of websockets is that a constant connection is made between the client and server. This is a big improvement over HTTP/S where a connection is created and closed every time a piece of data is sent. Additionally, this bidirectional connection allows data to be sent from the server to the client and the client to server without the client initiating it. This is different from HTTP/S as since HTTP/S is stateless, the server retains no information about the client and thus cannot send data to the client without the client's request.  
  
However, no technology is perfect and similarly neither is websockets. The main downside of websockets is the face that websockets require a stateful connection (the server keeping track of the current session information). This causes a lot of issues with scaling horizontally (by just adding more servers) as websites often use load balancers to move traffic from one server to another. Thus, if a client is moved to another server, the state data from their session could be lost. This requires more complicated logic to properly implement a horizontally scalable websocket system.


## Coding Reference

### Aliases
#### ArRequstHandlerFunction
`function<void(AsyncWebServer *request)>`  
See [AsyncWebServer](#asyncwebserver).
#### ArUploadHandlerFunction
`function<void(AsyncWebServerRequest *request, const String &filename, size_t index, uint8_t *data, size_t len, bool final)>`  
See [AsyncWebServerRequest](#asyncwebserverrequest)
#### ArBodyHandlerFunction
`function<void(AsyncWebServerRequest *request, uint8_t *data, size_t len, size_t index, size_t total)>`
See [AsyncWebServerRequest](#asyncwebserverrequest)
#### AwsTemplateProcessor
`function<size_t(uint8_t*, size_t, size_t)>`

### AwsEventHandler
`function<void(AsyncWebSocket *server, AsyncWebSocketClient *client, AwsEventType type, void *arg, uint8_t *data, size_t len)>`  
- server is the [websocket server](#asyncwebsocket) of the connection.
- client is the [websocket client](#asyncwebsocketclient) of the connection.
- type is the [event type](#awseventtype) of the event.
- arg is the argument of the event.
  - For a disconnect or data is sent, arg is [AwsFrameInfo](#awsframeinfo).
  - For a ping or pong event, arg is NULL.
  - For an error, arg is the reason code.
- data is the data being sent.
  - For a disconnect or data is sent, data is the data sent with the frame.
  - For a ping or pong event, data is NULL.
  - For an error, data is a string with the reason behind the error.
- len is the length of the message.


### Enums
#### WebRequestMethod
```cpp
  HTTP_GET = 0b00000001
  HTTP_POST = 0b00000010
  HTTP_DELETE = 0b00000100
  HTTP_PUT = 0b00001000
  HTTP_PATCH = 0b00010000
  HTTP_HEAD = 0b00100000
  HTTP_OPTIONS = 0b01000000
  HTTP_ANY = 0b01111111
```
#### SendStatus
```cpp
DISCARDED = 0
ENQUEUED = 1
PARTIALLY_ENQUEUED = 2
```
- Discarded - All messages sent were unsucessfull.
- Enqueued - All mesages sent were sucessfull.
- Partially enqueued - Some messages were successfull and other weren't

#### AwsFrameType
```cpp
WS_CONTINUATION
WS_TEXT
WS_BINARY
WS_DISCONNECT = 0x08
WS_PING
WS_PONG
```

#### AwsEventType
```cpp
WS_EVT_CONNECT
WS_EVT_DISCONNECT
WS_EVT_PING
WS_EVT_PONG
WS_EVT_ERROR
WS_EVT_DATA
```

### Classes/Structs

#### AsyncWebServer
Description:  
The server that will host the website and connect the requests from the user to the MCU.  
Implemented using a TCP server on the port defined by the constructor.  
In this example, will be used to host the server but all connections will be handled by the websocket.   

Constructor: `AsyncWebServer(int port)`  
Methods:
- `AsyncCallbackWebHandler& on(const char *uri, WebRequestMethodComposite method, ArRequestHandlerFunction onRequest, ArUploadHandlerFunction onUpload = nullptr, ArBodyHandlerFunction onBody = nullptr)` with variation
  - Handles different parts of requests to a specific uri, `uri`.
  - The `method` is the HTTP method that will be used in the response.
  - The `onRequest` [function](#arrequsthandlerfunction) that will handle the intital request.
  - The `onUpload` [function](#aruploadhandlerfunction) will handle an uploaded file (I think???).
  - The `onBody` [function](#arbodyhandlerfunction) will handle the body of the request (I think???).
  - The last two aren't required for our use case.
- `AsyncWebHandler& addHandler(AsyncWebHandler* handler)`
  - Adds a handler for incoming events to the server
  - For the sockets, you can use a [AsyncWebSocket](#asyncwebsocket) as it is a child of AsyncWebHandler
  - Returns a reference to the handler added.
- `bool removeHandler(AsyncWebHandler* handler)`
  - Removes an event handler for the server that was added to the server
  - Returns true if the handler was found and successfully removed.
- `void begin()`
  - Begins the server.
- `void end()`
  - Ends the server.

#### AsyncWebSocketClient
- The client associated with a specific websocket connection.
- Constructor ommitted as the library will usually create it for you.
- Methods have the same meaning as those in [AsyncWebSocket](#asyncwebsocket) as thus the explanation will be omitted.
- Methods:
  - `bool text(const uint8_t *message, size_t len)` with variations
  - `bool binary(const uint8_t *message, size_t len)` with variations
  - `bool ping(const uint8_t *data = NULL, size_t len = 0)` with variations
  - `void close(uint16_t code = 0, const char *message = NULL)` with variations

#### AsyncWebSocket
Description:  
A web handler that can be used in the [AsyncWebServer](#asyncwebserver) to handle events using a websocket.  
Can hold a variety of clients and send to either individual clients or all of them at once.

Constructor: `AsyncWebSocket(const String& URI)`  
Methods: 
- `void onEvent(AwsEventHandler handler)`
  - Set a [function](#awseventhandler) to be called when events occur on the web socket.   
- `bool ping(uint32 id, uint16_t *data = NULL, size_t len = 0)`
  - See [Ping Pong](#ping-pong).
  - Sends a ping frame to client with id `id`.
  - The data of the frame will be `data` with a length of `len`.
  - Returns true if ping is sent sucessfully.
- `SendStatus pingAll(const uint8_t *data = NULL, size_t len=0)`
  - See [Ping Pong](#ping-pong).
  - Sends a ping frame to all connected clients.
  - The data of the frame will be `data` with a length of `len`.
  - Returns a [SendStatus](#sendstatus).
- `bool text(uint32_t id, const uint8_t* message, size_t len)` with extra variations
  - See [Data Sending/Receiving](#data-sendingreceiving).
  - Sends a text frame to the client with id `id`.
  - The message sent will be `message` and message length is `len`.
  - Returns whether the frame was successfully sent.
- `SendStatus textAll(const uint8_t* message, size_t len)` with extra variations
  - See [Data Sending/Receiving](#data-sendingreceiving).
  - Sends a text frame to the all connected clients.
  - The message sent will be `message` and message length is `len`.
  - Returns [SendStatus](#sendstatus).
- `bool binary(uint32_t id, const uint8_t* message, size_t len)` with extra variations
  - See [Data Sending/Receiving](#data-sendingreceiving).
  - Sends a binary frame to the client with id `id`.
  - The message sent will be `message` and message length is `len`.
  - Returns whether the frame was successfully sent.
- `SendStatus binaryAll(const uint8_t* message, size_t len)` with extra variations
  - See [Data Sending/Receiving](#data-sendingreceiving).
  - Sends a binary frame to the all connected clients.
  - The message sent will be `message` and message length is `len`.
  - Returns [SendStatus](#sendstatus).
- `void close(uint32_t id, uint16_t code = 0, const char* message = NULL)`
  - See [Closing Handshake](#closing-handshake).
  - Sends a closing handshake to client with id `id`.
  - The optional message sent will be `message`.
- `void closeAll(uint16_t code = 0, const char* message = NULL)`
  - See [Closing Handshake](#closing-handshake).
  - Sends a closing handshake to all connected clients.
  - The optional message sent will be `message`.
- `void cleanupClients(uint16_t maxClients = DEFAULT_MAX_WS_CLIENTS)`
  - Goes through the client list in the websocket and removes any that are disconnected/non-existant.

#### AsyncWebServerRequest
- A request that a client made to the server.
- Stores the client that made the request, client, and information about the request.
- This is more related to HTTP and Servers and thus will not be covered in detail.
- You shouldn't make one yourself and as such the constructor is omitted.
- Methods:
  - `AsyncClient *client()`
    - Returns the client that sent the request.
  - `void send(...)` parameters ommitted due to large number of variations.
    - Sends a response to the request the client made.
  - `void send_P(int code, const String& contentType, const uint8_t* content, size_t len, AwsTemplateProcessor callback = nullptr)` with variations
    - Sends a reponse to the request in a specified format.
    - `code` is the HTTP code that will be the response.
    - `contentType` is the type of content being sent. Ex. `"text/html"`.
    - `content` is the content being sent.
    - `len` is the len of the content being sent.
    - `callback` is the [function](#awstemplateprocessor) that will proces the content and the return what will be sent to client.

#### AwsFrameInfo
- A struct with the header information of a frame.
- See [Frame Structure](#frame-structure)
- Values:
  - `uint8_t message_opcode`
    - The opcode of the message.
    - Can be [AwsFrameType](#awsframetype).
    - Only WS_TEXT adn WS_BINARY will be availble to the server. The rest will be handled by the library.
  - `uint32_t num`
    - The frame number of a fragmented message.
  - `uint8_t final`
    - Is this frame the final frame of the fragmented message.
  - `uint8_t masked`
    - Is this frame masked.
  - `uint8_t opcode`
    - The opcode of the specific frame.
    - Difference with message_opcode is that if the message is fragmented, this can be WS_CONTINUATION while if it isn't it will be the same as message_opcode.
  - `uint64_t length`
    - Length of the data sent.
  - `uint8_t mask[4]`
    - Mask key
  - `uint64_t index`
    - offset of the data inside the frame.