# ARDUINO-TimeBox-WLED

```
//https://randomnerdtutorials.com/esp32-flash-memory/
//https://forum.arduino.cc/t/how-to-get-current-time-using-esp8266/435120/8

#include <EEPROM.h>
#include "time.h"
#include <ESP8266WiFi.h>
#include <tinyxml2.h>
#include <ESP8266HTTPClient.h>

int switch_pin = D3;

const char* ssid     = "Joop02";                    // your network SSID (name)
const char* password = "joopcoolzero";                    // your network password

String mainURL ="http://192.168.178.129";
String ColorGreen="/win&R=0&G=128&B=0";
String ColorNormal="/win&R=255&G=184&B=112";
String ColorPreset="/win&PL=1";

const char* ntpServer = "pool.ntp.org";
const long  gmtOffset_sec = 0;
const int   daylightOffset_sec = 3600;

// Variables will change:
int buttonState;             // the current reading from the input pin
int lastButtonState = LOW;   // the previous reading from the input pin

// the following variables are unsigned longs because the time, measured in
// milliseconds, will quickly become a bigger number than can be stored in an int.
unsigned long lastDebounceTime = 0;  // the last time the output pin was toggled
unsigned long debounceDelay = 50;    // the debounce time; increase if the output flickers

using namespace tinyxml2;

/*
  %a Abbreviated weekday name 
  %A Full weekday name 
  %b Abbreviated month name 
  %B Full month name 
  %c Date and time representation for your locale 
  %d Day of month as a decimal number (01-31) 
  %H Hour in 24-hour format (00-23) 
  %I Hour in 12-hour format (01-12) 
  %j Day of year as decimal number (001-366) 
  %m Month as decimal number (01-12) 
  %M Minute as decimal number (00-59) 
  %p Current locale’s A.M./P.M. indicator for 12-hour clock 
  %S Second as decimal number (00-59) 
  %U Week of year as decimal number,  Sunday as first day of week (00-51) 
  %w Weekday as decimal number (0-6; Sunday is 0) 
  %W Week of year as decimal number, Monday as first day of week (00-51) 
  %x Date representation for current locale 
  %X Time representation for current locale 
  %y Year without century, as decimal number (00-99) 
  %Y Year with century, as decimal number 
  %z %Z Time-zone name or abbreviation, (no characters if time zone is unknown) 
  %% Percent sign 
  You can include text literals (such as spaces and colons) to make a neater display or for padding between adjoining columns. 
  You can suppress the display of leading zeroes  by using the "#" character  (%#d, %#H, %#I, %#j, %#m, %#M, %#S, %#U, %#w, %#W, %#y, %#Y) 
*/

long int LastTriggered;
//char buffer[80];

long int printLocalTime()
{

       time_t now = time(nullptr);
//       Serial.println(ctime(&now));
//        Serial.println(now);

  //------------------------------
  
//  time_t rawtime;
//  struct tm * timeinfo;
//  time (&rawtime);                      // Unix Timestamp
//  timeinfo = localtime (&rawtime);      // Apply localzone
//  
    //  printf("rawtime = %i \n", timeinfo);  // printout the timestamp
    //  sprintf(buffer,"%i",timeinfo);
    //  LastTriggered = buffer;
    //  Serial.println(LastTriggered);
  
   
        //formatting:
      //      strftime (buffer,80," %d %B %Y %H:%M:%S ",timeinfo);
      //      Serial.println(buffer);

    return now;
}

void CallAPI(String myString, bool output) {
  WiFiClient client;

  HTTPClient http;

  XMLDocument xmlDocument;

  Serial.print("[HTTP] begin...");
  if (http.begin(client, mainURL+myString)) {  // HTTP

    Serial.print("[HTTP] GET...");
    // start connection and send HTTP header
    int httpCode = http.GET();

    // httpCode will be negative on error
    if (httpCode > 0) {
      // HTTP header has been send and Server response header has been handled
            
      Serial.printf("[HTTP] GET... code: %d", httpCode);

      // file found at server
      if (output){
       if (httpCode == HTTP_CODE_OK || httpCode == HTTP_CODE_MOVED_PERMANENTLY) {
         String payload = http.getString();

         // Get Current Settings of WLED
         XMLDocument doc;
         doc.Parse( payload.c_str() );

          XMLElement* wled = doc.FirstChildElement( "vs" );

          const char* R = wled->FirstChildElement( "cl" )->GetText();
          const char* G = wled->FirstChildElement( "cl" )->NextSiblingElement()->GetText();
          const char* B = wled->FirstChildElement( "cl" )->NextSiblingElement()->NextSiblingElement()->GetText();

          Serial.println("");
          Serial.println(R);
          Serial.println(G);
          Serial.println(B);
       }
      }
    } else {
      Serial.printf("[HTTP] GET... failed, error: %s", http.errorToString(httpCode).c_str());
    }

    http.end();
  } else {
    Serial.printf("[HTTP} Unable to connect");
  }
    
}


void setup() 
{
  Serial.begin(115200);
  delay(10);
   
  pinMode(switch_pin, INPUT);
  
  // We start by connecting to a WiFi network
  Serial.print("\n\nConnecting to ");
  Serial.println(ssid);
  
  /* Explicitly set the ESP8266 to be a WiFi-client, otherwise, it by default,
    would try to act as both a client and an access-point and could cause
  network-issues with your other WiFi-devices on your WiFi-network. */
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) 
  {
    delay(500);
    Serial.print(".");
  }
  
  Serial.println();
  Serial.println("WiFi connected");
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());
  
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  
  Serial.println("\nWaiting for time");
  unsigned timeout = 5000;
  unsigned start = millis();
  while (!time(nullptr)) 
  {
    Serial.print(".");
    delay(1000);
  }
  delay(1000);
  
  Serial.println("Got Time...");
}

void loop() 
{
    
  int switchStatus = digitalRead(switch_pin);   // read status of switch
  
  // If the switch changed, due to noise or pressing:
  if (switchStatus != lastButtonState) {
    // reset the debouncing timer
    lastDebounceTime = millis();
  }

//  Serial.println("LastTriggered: ");
//  Serial.println(LastTriggered);
//  Serial.println("vs");
//  Serial.println(printLocalTime());

  //  DIFF:
  long diff_t = difftime(printLocalTime(),LastTriggered);
//  Serial.println(diff_t); // in seconds

  // check if Last Triggered was an hour ago
  if (diff_t > 3600){       // 60min
    // set color green
    CallAPI(ColorGreen,false);    // Reset to Preset    
  }
 
  if ((millis() - lastDebounceTime) > debounceDelay) {
    // whatever the reading is at, it's been there for longer than the debounce
    // delay, so take it as the actual current state:
    
    // if the button state has changed:
    if (switchStatus != buttonState) {
      buttonState = switchStatus;

      // only toggle the LED if the new button state is HIGH
      if (buttonState == HIGH) {
        //do stuff, when pressed the switch

        CallAPI("/win",true);          // get & save current RGB value    
        
        LastTriggered = printLocalTime();              // save time..
        
        CallAPI(ColorPreset,false);    // Reset to Preset
        
      }
    }

  }

  // save the reading. Next time through the loop, it'll be the lastButtonState:
  lastButtonState = switchStatus;
//  delay(10000); // 1 sec
}
```
