Often in our projects we need to display some information, and quite often, it is digital, like, time or temperature. In such a case, a TM1637-based LED is a very convenient choice.

In this example, we connect RobotDyn 4-Digit 7-Segment Tube to an MCU (RobotDyn Uno or Arduino Uno, etc.) and run some code to display various information. 6-digit, 8-digit variants of TM1637 LEDs work in a similar way.

This module comes either with a colon (for time display):

![](PHOTO==TOP==0G-00004781==Mod-LED-Display-4D-7S==Green-clock-colon.jpg)

or decimal points after every digit:

![](PHOTO==TOP==0G-00004781==Mod-LED-Display-4D-7S==Green-decimal-point.jpg)



## Connection

![](tm1637-and-arduino-2.jpg) 

The LED module has 4 pins: 5v, Gnd, CLK and DIO. We connect 5v and Gnd to the corresponding pins of MCU, CLK and DIO to any free digitals pins.

## Software

It this example we will use this library: [tm1637.h](https://github.com/tehniq3/TM1637-display)

Other classic Arduino libraries, like TM1637.h, TM1637display.h by AVISHORPE and SevenSegmentTM1637.h should work as well.

<!--- When you download the library, you find several examples in the ```examples``` folder. Let's use some of them. --->


### Code

The example below displays **```1234```**, or any other 4 digits as they are preset in the sketch, and flashes the colon every 1 second:

```cpp

#include "TM1637.h"   // import the library
int8_t DispMSG[] = {1, 2, 3, 4};   // set the digits to be displayed

//Set up the pins where CLK and DIO are connected to
#define CLK 3
#define DIO 2

//Create a TM1637 class object
//and set the pin numbers
TM1637 tm1637(CLK, DIO);
void setup()
{
  tm1637.init();

  //Set up brightness
  /*
     BRIGHT_TYPICAL = 2 Medium
     BRIGHT_DARKEST = 0 Dark
     BRIGHTEST = 7      Bright
  */
  tm1637.set(BRIGHT_TYPICAL);
}
void loop()
{
  //Set colon on
  tm1637.point(true);
  //Display the contents of DispMSG array
  tm1637.display(DispMSG);
  //Pause 1s
  delay(1000);
  //Set colon off
  tm1637.point(false);
  //Display the contents of DispMSG array
  tm1637.display(DispMSG);
  //Pause 1s
  delay(1000);
}

```

### Display temperature and humidity

We will need a temperature/humidity sensor to make a digital thermometer/hygrometer.

```cpp

#include "dht.h"
#include "TM1637.h"
//{0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15};
//0~9,A,b,C,d,E,F
#define dht_pin 4 // Pin for sensor
#define CLK 3//Pins for TM1637      
#define DIO 2
TM1637 tm1637(CLK, DIO);
dht DHT;
void setup() {
  tm1637.init();
  tm1637.set(BRIGHT_TYPICAL);
  //BRIGHT_TYPICAL = 2,BRIGHT_DARKEST = 0,BRIGHTEST = 7   0-7;
  delay(1500);//Delay
}
void loop() {
  DHT.read11(dht_pin);
  int temp = DHT.temperature;
  int humidity = DHT.humidity;
  int digitoneT = temp / 10;
  int digittwoT = temp % 10;
  int digitoneH = humidity / 10;
  int digittwoH = humidity % 10;
  tm1637.display(1, digitoneT);
  tm1637.display(2, digittwoT);
  tm1637.display(3, 12); //  C
  delay (5000);
  tm1637.display(1, digitoneH);
  tm1637.display(2, digittwoH);
  tm1637.display(3, 15); //  F
  delay(5000);
}

```

Here we use the standard DHT.h Arduino library to read data from the sensor.

We use pin 4 to connect the temperature sensor. Then, when we set everything up, we can display the values, digit by digit.

You can also refer to the examples included in the library.