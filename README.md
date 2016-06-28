#include <Wire.h>
#include <ESP8266WiFi.h>
#include <WiFiClient.h>
#include <ESP8266WebServer.h>
#include "Adafruit_TCS34725.h"

Adafruit_TCS34725 tcs = Adafruit_TCS34725(TCS34725_INTEGRATIONTIME_50MS, TCS34725_GAIN_4X);
  #define bluepin 12
  #define redpin 13
  #define greenpin 14
  #define commonAnode false

    byte gammatable[256];
    uint16_t lux,colorTemp,redcolor,bluecolor,greencolor,red,green,blue;    
    String readString; 
    const char* ssid = "MySSID";
    const char* password = "MyPassword";
// Create an instance of the server
// specify the port to listen on as an argument
WiFiServer server(80);

void setup() {
  //***************************************
  Wire.begin(5,4);     //GPIO5- SDA, GPIO4- SCL 
  Serial.begin(9600);
   if (tcs.begin()) {
    Serial.println("Found sensor");
  } else {
    Serial.println("No TCS34725 found ... check your connections");
    while (1); // halt!
  }
  // use these three pins to drive an LED
  pinMode(bluepin, OUTPUT);
  pinMode(redpin, OUTPUT);
  pinMode(greenpin, OUTPUT);  
  // this gamma table helps convert RGB colors to what humans see
  for (int i=0; i<256; i++) {
    float x = i;
    x /= 255;
    x = pow(x, 2.5);
    x *= 255;
    if (commonAnode) {
      gammatable[i] = 255 - x;
    } else {
      gammatable[i] = x;      
    }
  }
  Serial.begin(9600);
  delay(10);
  // Connect to WiFi network
  Serial.println();
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);  
  WiFi.begin(ssid, password);  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("");
  Serial.println("WiFi connected");  
  // Start the server
  server.begin();
  Serial.println("Server started");
  // Print the IP address
  Serial.println(WiFi.localIP());
}

void loop() {
  
  uint16_t r, g, b, c, total;

  tcs.getRawData(&r, &g, &b, &c);
  colorTemp = tcs.calculateColorTemperature(r, g, b);
  lux = tcs.calculateLux(r, g, b);
  redcolor = r;
  greencolor = g;
  bluecolor = b;
  uint32_t suma = r+g+b;
  float red, green, blue;
  red=r; red /= suma;
  green = g; green /= suma;
  blue = b; blue /= suma;
  red *= 256; green *= 256; blue *= 256;
 // Serial.print("Cherveno: "); Serial.print((int)red); Serial.print(" ");
  if (isnan(colorTemp) || isnan(lux) || isnan(redcolor) || isnan(greencolor) || isnan(bluecolor)) {
      Serial.println("Failed to read from DHT sensor!");
      return;
    }

  Serial.print("Color Temp: "); Serial.print(colorTemp, DEC); Serial.print(" K - ");
  Serial.print("Lux: "); Serial.print(lux, DEC); Serial.print(" - ");
  Serial.print("R: "); Serial.print(r, DEC); Serial.print(" ");
  Serial.print("G: "); Serial.print(g, DEC); Serial.print(" ");
  Serial.print("B: "); Serial.print(b, DEC); Serial.print(" ");
  Serial.print("C: "); Serial.print(c, DEC); Serial.print(" ");
  Serial.println(" ");
  
//********************LED*************************
  Serial.println();

 //visualization

  analogWrite(bluepin,gammatable[(int)blue]);
  analogWrite(redpin, gammatable[(int)red]);
  analogWrite(greenpin, gammatable[(int)green]);
  delay(2000);

  // Check if a client has connected
  WiFiClient client = server.available();
    if (client) {
    while (client.connected()) {
      if (client.available()) {
        char c = client.read();
        //read char by char HTTP request
        if (readString.length() < 100) {
          //store characters to string 
          readString += c; 
        } 
        //if HTTP request has ended
        if (c == '\n') {
          Serial.println(readString); //print to serial monitor for debuging 
        //now output HTML data header
           if(readString.indexOf('?') >=0) { //don't send new page
            client.println("HTTP/1.1 204");
            client.println();
            client.println();  
             }
           else {
            client.println("HTTP/1.1 200 OK"); //send new page
            client.println("Content-Type: text/html");
            client.println();
            client.println("<HTML>");
            client.println("<HEAD>");
            client.println("<TITLE>Dushkov's Project</TITLE>");
            client.println("</HEAD>");
            client.println("<BODY>");
            client.println("<H1>ESP8266</H1>");
            client.println("<p>");
            client.println("The measured values from the RGB color sensor are :");
            client.println("</p>");
            client.println("<p>");
            client.println("Red=" + String((int)redcolor) + "  Green=" + String((int)greencolor) + "  Blue=" + String((int)bluecolor));
            client.println("</p>");
            client.println("<p>");  
            client.println("<p>");
            client.println("The calculated values for color temperature and illuminance are :");
            client.println("</p>");
            client.println("ColorTemp=" + String((int)colorTemp) + "  Lux=" + String((int)lux));          
            client.println("</p>");  
            client.println("</body>");
            client.println("<p>");
            client.println("If you want to measure the color temperature of an object, you should turn ON the LED");
            client.println("</p>");
            client.println("<a href=\"/?on\" target=\"inlineframe\">TURN ON THE LED</a>"); 
            client.println("<p>");
            client.println("If you want to measure the color temperature of a light source, you should turn OFF the LED");
            client.println("</p>");
            client.println("<a href=\"/?off\" target=\"inlineframe\">TURN OFF THE LED</a>");           
            client.println("<IFRAME name=inlineframe style=\"display:none\" >");          
            client.println("</IFRAME>");
            client.println("</BODY>");
            client.println("</HTML>");
             }

          delay(1);
          //stopping client
          client.stop();

          // control arduino pin
          if(readString.indexOf("on") >0)//checks for on
          {
            tcs.setInterrupt(false);    // set pin 4 high
            Serial.println("Led On");
          }
          if(readString.indexOf("off") >0)//checks for off
          {
            tcs.setInterrupt(true);    // set pin 4 low
            Serial.println("Led Off");
          }
          //clearing string for next read
          readString="";
        }
      }
    }
  }
  }
