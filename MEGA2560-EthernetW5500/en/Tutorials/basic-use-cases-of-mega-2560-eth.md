# Basic Use-Cases of Mega 2560 ETH

## Table of Contents

[**Data Logger with MySQL**](#data-logger-with-mysql)

  [Hardware](#the-hardware)

  [The Software](#the-software)

[**FTP Server using SD Card**](#ftp-server-using-sd-card)

[**Relay Shield Example**](#relay-shield-example)

  [Schematics](#schematics)

  [Code](#code)

[**Online Graph Using Mega 2560 ETH Web Server with No External Services**](#online-graph-using-mega-2560-eth-web-server-with-no-external-services)

  [Connecting a Sensor](#connecting-a-sensor)

  [Preparing the Web Page on SD Card](#preparing-the-web-page-on-sd-card)

  [Preparing the sketch](#preparing-the-sketch)

  [The main sketch](#the-main-sketch)

  [The Results](#the-results)

[**Web Server**](#web-server)

  [Creating the Web Page](#Creating the Web Page)

  [Preparing the Web-Server Sketch](#preparing-the-web-server-sketch)

[**Saving Temperature Sensor Data to Google Docs**](#saving-temperature-sensor-data-to-google-docs)

  [Get Accounts](#get-accounts)

  [Create a Google Form and Get FORM KEY](#create-a-google-form-and-get-form-key)

  [Pushingbox, Service and Scenarios](#pushingbox-service-and-scenarios)

  [Sketch for the Mega ETH](#sketch-for-the-mega-eth)
  

## Data Logger with MySQL

In this example we will build a simple temperature data logger that will send data to a remote MySQL database. Every measurement will be accompanied with a corresponding timestamp, hence we'll need a sensor module and a real time clock module

### The Hardware

1. Mega ETH board
2. DHT11 temperature/humidity sensor
3. RTC (Real Time Clock) DS1307 module
4. one breadboard
5. connecting wires

The RTC modules' I2C pins SDA and SCL, DHT11 - to pins 7 and 8 of the Mega ETH board.


## The Software

First we need to set up a MySQL database server. Let's assume that we install it on a computer with an ip address of 192.168.1.33 on our local network and create a database `robologger`:

```sql

mysql> create database robologger
Query OK, 1 row affected (0.00 sec)
mysql> use robologger;
database changed
mysql> create table sensdata (id float(6,0) not null auto_increment primary key, date date, time time, temp float (5,2), humi float(4,2) );
Query OK, 0 rows affected (0.01 sec)
mysql> insert into sensdata (date, time, temp, humi) values ('2019-11-10','14:11:00', 32.4, 97.8);
Query OK, 1 row affected (0.00 sec)

```
Next we will grant access to the user robotdyn to this database:

```sql

mysql> grant all privileges ON robologger.* to 'robotdyn'@'192.168.1.%' identified by 'robotdyn' with grant option;
Query OK, 0 rows affected (0.00 sec)
mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
mysql> quit;

```

Now, in our sketch we will need to connect to the Internet and set the RTC using NTP protocol. Then, using the Real Time Clock and every 10 minutes we log time, sensor data and send it to the database.

We will need an updated [Time library](http://www.pjrc.com/teensy/td_libs_Time.html). Download and extract it to your project folder.

For DHT11 we will use [this library](https://github.com/practicalarduino/SHT1x)
Please rename the extracted SHT1x-master to SHT1xmaster.

Then, we will need the MySQL connector libraries from [https://launchpad.net/mysql-arduino](https://launchpad.net/mysql-arduino).

In mysql.h uncomment the line:

```cpp

#define WITH_SELECT // Uncomment this for use without SELECT capability

```

Try compiling the code. You can find additional info on the use of MySQL connector on [Chuck's blog](http://drcharlesbell.blogspot.ro/2013/04/introducing-mysql-connectorarduino_6.html).

The code:
```cpp

#include <SPI.h>
#include <Ethernet.h>
#include <Dns.h>
#include <Time.h>
#include <Wire.h>
#include <DS1307RTC.h>
#include <SHT1x.h>
#include <sha1.h>
#include <mysql.h>
#include <string.h>

/* ******** Ethernet Card Settings ******** */
// Set this to your Ethernet Card Mac Address
byte mac[] = {0x90, 0xA2, 0xDA, 0x0D, 0xFE, 0x43 };

/* ******** NTP Server Settings ******** */
/* us.pool.ntp.org NTP server
(Set to your time server of choice) */
IPAddress timeServer;

/* Set this to the offset (in seconds) to your local time
This example is GMT + 2 */
const long timeZoneOffset = 7200L;

/* Syncs to NTP server every 15 seconds for testing,
   set to 1 hour or more to be reasonable */
unsigned int ntpSyncTime = 15;        

/* ALTER THESE VARIABLES AT YOUR OWN RISK */
// local port to listen for UDP packets
unsigned int localPort = 8888;
// NTP time stamp is in the first 48 bytes of the message
const int NTP_PACKET_SIZE= 48;      
// Buffer to hold incoming and outgoing packets
byte packetBuffer[NTP_PACKET_SIZE];  
// A UDP instance to let us send and receive packets over UDP
EthernetUDP Udp;                    
       

/* ********** RTC Time settings ********* */
tmElements_t tm;
int minutes_now;
int minutes_next;

/* ******* Settings for the SHT1x sensor  */
#define dataPin  9
#define clockPin 8
SHT1x sht1x(dataPin, clockPin);
float temp_c;
float humidity;

/* ***** MySQl Server settings ********** */
Connector my_conn;        // The Connector/Arduino reference
IPAddress server_addr(192, 168, 1, 133);  // IP address of MySQL server
char user[] = "robotdyn";
char password[] = "robotdyn";
// maximum tries to connect
int num_fails;
#define MAX_FAILED_CONNECTS 5
// We define some strings
char myquery[150];  // more than enough; stores command to print table content
char insquery[200]; // more than enough; stores command to insert new data
const char testselect[] ={"SELECT * FROM robologger.robodata ORDER BY id DESC LIMIT %s"};
// general format to insert data is:
//const char insertdata[] = "INSERT INTO ardulogger.ardudata (date,time,temp,humi) VALUES ('2014-11-11','11:45:11', 25.8 , 31.21)";
//const char insertdata[] = {"INSERT INTO ardulogger.ardudata (date,time,temp,humi) VALUES ('%s-%s-%s','%s:%s:%s', %s , %s)"};
const char insertdata[] = {"INSERT INTO robologger.robodata (date,time,temp,humi) VALUES ('%s-%s-%s','%s:%s:%s', %s , %s)"};
// buffer for number to character conversion
char yearbuf[5];
char monthbuf[3];
char daybuf[3];
char hourbuf[3];
char minutebuf[3];
String temp_buf;
char secbuf[3];
char tempbuf[6];
char humibuf[5];
int timebuf;

 
void setup() {
   Serial.begin(9600);
   
   // Ethernet shield and NTP setup
   int i = 0;
   int DHCP = 0;
   DHCP = Ethernet.begin(mac);
   //Try to get dhcp settings 30 times before giving up
   while( DHCP == 0 && i < 30){
     delay(1000);
     DHCP = Ethernet.begin(mac);
     i++;
   }
   if(!DHCP){
    Serial.println("DHCP FAILED");
     for(;;); //Infinite loop because DHCP Failed
   }
   Serial.println("DHCP Success");  
 
   //Just for testing we print the obtained IP address  
   Serial.print("My IP address: ");
   for (byte thisByte = 0; thisByte < 4; thisByte++) {
    // print the value of each byte of the IP address:
   Serial.print(Ethernet.localIP()[thisByte], DEC);
   Serial.print(".");
   }
   Serial.println();
 
  // We get the address of a NTP server from NTP pool
  DNSClient dns;
  dns.begin(Ethernet.dnsServerIP());
  dns.getHostByName("ro.pool.ntp.org",timeServer);
  Serial.print("NTP IP from the pool: ");
  Serial.println(timeServer);
 
  // Now we get the time
  Serial.println("trying to get the time");
  int trys=0;
      while(!getTimeAndDate() && trys<10){
        trys++;
      }
      if(trys<10){
        Serial.println("Ntp server update success");
      }
      else{
        Serial.println("Ntp server update failed");
      }
  // Print the time
  clockDisplay();
 
  // We have the right time, now we set the RTC clock
  if (RTC.read(tm)) {
       Serial.println("The DS1307 is running.");
       Serial.println("Updating the RTC time.");
       Serial.println();
       tm.Day = day();
       tm.Month = month();
       tm.Year = year()-1970;
       tm.Hour = hour();
       tm.Minute = minute();
       tm.Second = second();
       RTC.write(tm);   //update the time
  } else if (RTC.chipPresent()) {
       Serial.println("The RTC stopped.");
       Serial.println("Setting the RTC time.");
       Serial.println();
       tm.Day = day();
       tm.Month = month();
       tm.Year = year()-1970;
       tm.Hour = hour();
       tm.Minute = minute();
       tm.Second = second();
       RTC.write(tm);   //update the time
   }
   else {
       Serial.println("RTC read error!  Please check the circuitry.");
       Serial.println();
   }
   // Just in case we read the RTC time for debugging purposes
   // Serial.println("Computing the next DB access time");
    RTC.read(tm);
    minutes_now=tm.Minute;
   // minutes_next=floor(tm.Minute/10)*10+10;
    minutes_next=minutes_now+1; //debugging - write every minute to DB
    if (minutes_next==60)
      minutes_next=0;
    Serial.print("Current minute: ");
    Serial.print(minutes_now);
    Serial.println();
    Serial.print("Minute for the next database write: ");
    Serial.print(minutes_next);
    Serial.println();

} //end of setup


// functions for NTP server
// Do not alter this function, it is used by the system
int getTimeAndDate() {
   int flag=0;
   Udp.begin(localPort);
   sendNTPpacket(timeServer);
   delay(1000);
   if (Udp.parsePacket()){
     Udp.read(packetBuffer,NTP_PACKET_SIZE);  // read the packet into the buffer
     unsigned long highWord, lowWord, epoch;
     highWord = word(packetBuffer[40], packetBuffer[41]);
     lowWord = word(packetBuffer[42], packetBuffer[43]);  
     epoch = highWord << 16 | lowWord;
     epoch = epoch - 2208988800 + timeZoneOffset;
     flag=1;
     setTime(epoch);
     //ntpLastUpdate = now();  not needed anymore as we update the time only once
   }
   return flag;
}
 
// Do not alter this function, it is used by the system
unsigned long sendNTPpacket(IPAddress& address)
{
  memset(packetBuffer, 0, NTP_PACKET_SIZE);
  packetBuffer[0] = 0b11100011;
  packetBuffer[1] = 0;
  packetBuffer[2] = 6;
  packetBuffer[3] = 0xEC;
  packetBuffer[12]  = 49;
  packetBuffer[13]  = 0x4E;
  packetBuffer[14]  = 49;
  packetBuffer[15]  = 52;                  
  Udp.beginPacket(address, 123);
  Udp.write(packetBuffer,NTP_PACKET_SIZE);
  Udp.endPacket();
}

// Clock display of the time and date (Basic)
void clockDisplay(){
  Serial.print(hour());
  printDigits(minute());
  printDigits(second());
  Serial.print(" ");
  Serial.print(day());
  Serial.print(" ");
  Serial.print(month());
  Serial.print(" ");
  Serial.print(year());
  Serial.println();
}
 
// Utility function for clock display: prints preceding colon and leading 0
void printDigits(int digits){
  Serial.print(":");
  if(digits < 10)
    Serial.print('0');
    Serial.print(digits);
}
// End of NTP functions

void print2digits(int number) {
  if (number >= 0 && number < 10) {
    Serial.write('0');
  }
  Serial.print(number);
}



void loop(){
 
    RTC.read(tm);
    minutes_now=tm.Minute;
    if (minutes_now==minutes_next){
    //if (time_now.Minute==minutes_next){  
       // compute the next time for write
       //minutes_next=floor(tm.Minute/10)*10+10; // write every 10 minutes - normal mode
       minutes_next=minutes_now+1; // write every minute - debug mode
       if (minutes_next==60)
       minutes_next=0;
       temp_c = sht1x.readTemperatureC();
       humidity = sht1x.readHumidity();
       // we construct the INSERT code here
       // first we initialize the buffers
       temp_buf=String(tm.Year+1970);
       temp_buf.trim();
       temp_buf.toCharArray(yearbuf,5);
       temp_buf=String(tm.Month);
       temp_buf.trim();
       temp_buf.toCharArray(monthbuf,3);
       temp_buf=String(tm.Day);
       temp_buf.trim();
       temp_buf.toCharArray(daybuf,3);
       temp_buf=String(tm.Hour);
       temp_buf.trim();
       temp_buf.toCharArray(hourbuf,3);
       temp_buf=String(tm.Minute);
       temp_buf.trim();
       temp_buf.toCharArray(minutebuf,3);
       temp_buf=String(tm.Second);
       temp_buf.trim();
       temp_buf.toCharArray(secbuf,3);
       //  float to char conversion
       dtostrf(temp_c, 5 , 2, tempbuf);
       dtostrf(humidity, 5, 2, humibuf);
       // now we build the command for data insertion
       sprintf(insquery,insertdata,yearbuf,monthbuf,daybuf,hourbuf,minutebuf,secbuf,tempbuf,humibuf);
       //ready to work with the database
       Serial.println("DB write: ");
       // Here comes the MySQL data code
       if (my_conn.mysql_connect(server_addr, 3306, user, password))  {
          // check for connection
          if (my_conn.is_connected()) {
            Serial.println("Last 5 DB entries!");
            sprintf(myquery,testselect,"5");
            Serial.println(myquery);
            my_conn.cmd_query(myquery);
            delay(250); // small delay to wait for MySQl server to respond
            my_conn.show_results();
            delay(1000);
            Serial.println("Inserting new data!");
            Serial.println(insquery);
            my_conn.cmd_query(insquery);
            delay(1500); // small delay to wait for MySQl server to respond
            // We verify the last entered data
            Serial.println("New data entered!");
            sprintf(myquery,testselect,"1");
            Serial.println(myquery);
            my_conn.cmd_query(myquery);
            delay(250); // small delay to wait for MySQl server to respond
            my_conn.show_results();
            num_fails = 0;
            } else {
            //my_conn.disconnect();
            Serial.println("Connecting again...");
            if (my_conn.mysql_connect(server_addr, 3306, user, password)) {
            delay(500);
            Serial.println("Success!");
            } else {
            num_fails++;
            Serial.println("Connect failed!");
              if (num_fails == MAX_FAILED_CONNECTS) {
                Serial.println("No MySQL server available...");
                delay(2000);
                }
              }
           }
       // close the database connection
       my_conn.disconnect();   
       }
  }

    // Just prints the time so we know we are not stuck
    Serial.print("Current time is  ");
    print2digits(tm.Hour);
    Serial.write(':');
    print2digits(tm.Minute);
    Serial.write(':');
    print2digits(tm.Second);
    Serial.println();
    
    // Basically we check the time every 5 seconds. You can increase this
    delay(5000);


}

```

## FTP Server using SD Card

Being a fully-capable network device, RobotDyn Mega 2560 ETH can even act as a simple but fully functional FTP server, storing files to and serving from an SD card.

This FTP server is created by Github user galledojm ([https://github.com/gallegojm](https://github.com/gallegojm)).

You can find full code as well as instructions on [Github](https://github.com/gallegojm/Arduino-Ftp-Server/tree/master/FtpServer).


## Relay Shield Example

![Robotdyn Relay Shield](photo_angle01_0g-00004529_relayshielduno_assembled.jpg)

The RobotDyn Relay Shield provides four high quality relays with NO/NC interfaces. LED indicators show the on/off state of each relay.

In this tutorial we will show how to use a relay to control an LED, however, its real purpose is to control high current devices.

### Schematics

![Robotdyn Relay Shield with LED](robotdyn_relayshield_with_led.png)

## Code

```cpp

int RelayControl1 = 7;    
 
 
void setup()
{
    pinMode(RelayControl1, OUTPUT);
}
 
 
void loop()
{
    digitalWrite(RelayControl1,HIGH);// NO1 and COM1 Connected (LED on)
    delay(1000); // wait 1000 milliseconds (1 second)
    digitalWrite(RelayControl1,LOW);// NO1 and COM1 Disconnected (LED off)
    delay(1000); // wait 1000 milliseconds (1 second)
}

```
## Online Graph Using Mega 2560 ETH Web Server with No External Services

In the following example, we will create a sketch that turns your Mega
ETH board to a Web Server that serves a graphical representation of a valure provided by a connected sensor (light sensor for this example). Since microcontroller resources are very limited, it is a common practice to use additional web services for more complex tasks like graphical representation of information. But if we don't want to depend on an external service, free or commercial, we might want to try to implement everything using nothing but the board, which is actually possible.

### Connecting a Sensor

We connect a photo-resistor to 5V, GND and A0 pins of Mega 2560 ETH in such a way that the minimum light intensity is zero, while the maximum is 1024.

![Fig 1. Sensor connection diagram](photosensor.svg)

### Preparing the Web Page on SD Card

Let's create a simple HTML page and store it on an SD card using your
computer and a card reader. While you can use almost any text editor to
write HTML code, it will be much easier if it can highlight the HTML
syntax and automatically close HTML tags. Geany (geany.org) is an
example of such an editor available on almost all operating systems.
Create the following web page in the editor:

```html

<!DOCTYPE HTML>
<html>
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
        <title>Super Graphing Data Logger!</title>
 
        <script type="text/javascript" src="http://ajax.googleapis.com/ajax/libs/jquery/1.8.2/jquery.min.js"></script>
        <script type="text/javascript">
function getDataFilename(str){
    point = str.lastIndexOf("file=")+4;
 
    tempString = str.substring(point+1,str.length)
    if (tempString.indexOf("&") == -1){
    return(tempString);
    }
    else{
        return tempString.substring(0,tempString.indexOf("&"));
    }
         
}
 
query  = window.location.search;
 
var dataFilePath = "/data/"+getDataFilename(query);
 
$(function () {
    var chart;
    $(document).ready(function() {
     
        // define the options
        var options = {
     
            chart: {
                renderTo: 'container',
                zoomType: 'x',
                spacingRight: 20
            },
     
            title: {
                text: 'Light levels recorded by the Arduino'
            },
     
            subtitle: {
                text: 'Click and drag in the plot area to zoom in'
            },
     
            xAxis: {
                type: 'datetime',
                maxZoom: 2 * 3600000
            },
     
            yAxis: {
                title: {
                    text: 'Light Levels (0 - 1024)'
                },
                min: 0,
                startOnTick: false,
                showFirstLabel: false
            },
     
            legend: {
                enabled: false
            },
     
            tooltip: {
                formatter: function() {
                        return '<b>'+ this.series.name +'</b><br/>'+
                        Highcharts.dateFormat('%H:%M - %b %e, %Y', this.x) +': '+ this.y;
                }
            },
     
            plotOptions: {
                series: {
                    cursor: 'pointer',
                    lineWidth: 1.0,
                    point: {
                        events: {
                            click: function() {
                                hs.htmlExpand(null, {
                                    pageOrigin: {
                                        x: this.pageX,
                                        y: this.pageY
                                    },
                                    headingText: this.series.name,
                                    maincontentText: Highcharts.dateFormat('%H:%M - %b %e, %Y', this.x) +':<br/> '+
                                        this.y,
                                    width: 200
                                });
                            }
                        }
                    },
                }
            },
     
            series: [{
                name: 'Light Levels',
                marker: {
                    radius: 2
                }
            }]
        };
     
     
        // Load data asynchronously using jQuery. On success, add the data
        // to the options and initiate the chart.
        //  http://api.jquery.com/jQuery.get/

        jQuery.get(dataFilePath, null, function(csv, state, xhr) {
            var lines = [],
                date,
     
                // set up the two data series
                lightLevels = [];
     
            // inconsistency
            if (typeof csv !== 'string') {
                csv = xhr.responseText;
            }
     
            // split the data return into lines and parse them
            csv = csv.split(/\n/g);
            jQuery.each(csv, function(i, line) {
     
                // all data lines start with a double quote
                line = line.split(',');
                date = parseInt(line[0], 10)*1000;
     
                lightLevels.push([
                    date,
                    parseInt(line[1], 10)
                ]);
                 
            });
     
            options.series[0].data = lightLevels;
     
            chart = new Highcharts.Chart(options);
        });
    });
     
});
        </script>
    </head>
    <body>
        <p style="text-align:center;">Please allow the chart to load, it may take up to 30 seconds </p>
        <hr/>
<script src="http://cdnjs.cloudflare.com/ajax/libs/highcharts/2.3.5/highcharts.js"></script>
 
<!-- Additional files for the Highslide popup effect -->
<script type="text/javascript" src="http://www.highcharts.com/highslide/highslide-full.min.js"></script>
<script type="text/javascript" src="http://www.highcharts.com/highslide/highslide.config.js" charset="utf-8"></script>
<link rel="stylesheet" type="text/css" href="http://www.highcharts.com/highslide/highslide.css" />
 
<div id="container" style="min-width: 400px; height: 400px; margin: 0 auto"></div>
 
    </body>
</html>

```

Store it on the SD card in the root directory as *index.html*.

### Preparing the sketch

Our sketch will be placed in EEPROM and will make use of 2 extra libraries: [EEPROMAnything](http://playground.arduino.cc/Code/EEPROMWriteAnything) and [Time library](http://playground.arduino.cc/Code/Time).

```c

/* ************************************************************************
 * ***            Super Graphing Data Logger - EEPROM config            ***
 * ************************************************************************
 * Everett Robinson, December 2012.
 *
 * The following extra non standard libraries were used, and will need to be
 * added to the libraries folder:
 * - EEPROMAnything:  http://playground.arduino.cc/Code/EEPROMWriteAnyt...
 *
 * This sketch helps you set the values in EEPROM which are necessary for
 * Super Graphing Data Logger. It should only need the be run once before
 * the first time you set up SGDL, or in the unlikely event that the EEPROM
 * becomes corrupted.
 *
 * Please ensure that the values in configuration config are appropriate for
 * your project before uncommenting the EEPROM_writeAnything(0, config); line.
 *
 */
 
#include <EEPROM.h>
#include <EEPROMAnything.h>
 
typedef struct{
    unsigned long newFileTime;
    char workingFilename[19];
  } configuration;
 
//This is a one off thing, so everything is in setup
void setup(){
  Serial.begin(9600);
   
  //Create the config struct to write to EEPROM, change values as appropriate
  //Make sure your filename is not too long for the workingFilename char array 
  configuration config = {1356912000L,"/data/25-12-12.csv"};
  //Write the values to the EEPROM
  //EEPROM_writeAnything(0, config);       //Uncomment when you're sure everything is correct
  configuration config2;                   //Create a second config struct for verification
  EEPROM_readAnything(0,config2);
  Serial.print("The value read from EEPROM for newFileTime is: ");
  Serial.println(config2.newFileTime);
  Serial.print("The value read from EEPROM for workingFilename is: ");
  Serial.println(config2.workingFilename);
  Serial.println("If those values are correct then everything went as planned. Otherwise,");
  Serial.println("please double check that the values declared for the struct config are");
  Serial.println("correct and that that EEPROM_writeAnything line is uncommented.");
}
 
 
void loop(){
}

```
Please mind the commented line `//EEPROM_writeAnything(0, config);` - uncomment it when you are sure that everything is correct.


### The main sketch

You will need to do a few adjustments to the code, like MAC and IP addresses. You can also adjust the timeserver address to your liking.

The code is set up to make measurements every 10 minutes, and to create a new datafile every week. Data files use a dd-mm-yy.csv format.

```c

/* ************************************************************************
 * ***                    Super Graphing Data Logger                    ***
 * ************************************************************************
 * Everett Robinson, December 2012. More at:  http://everettsprojects.com
 *
 * This sketch relies on the SD and ethernet libraries in arduino 1.0 or newer.
 * The following extra non standard libraries were also used, and will need to
 * be added to the libraries folder:
 * - Time:  http://everettsprojects.com
 * - EEPROMAnything:  http://everettsprojects.com
 *
 * If this is your first time setting up this project, please go get the
 * EEPROM_config sketch from  http://everettsprojects.com so that you can 
 * configure the config struct in the EEPROM memory. Usage of the EEPROM 
 * is needed to make the project resilient against a temporary loss of power.
 *
 * You must also ensure that you have the HC.htm file in the root directory
 * of your SD card, as well as a data directory where the datafiles will be
 * stored.
 *
 * This sketch combines the functionality of an existing fileserver example
 * which can be found at  http://everettsprojects.com
 * with the Datalogger example that comes with the new SD library from 1.0,
 * as well as some code from the UdpNtpClient example that cones with the
 * ethernet library. 
 *
 * Added to all of these are some tricks to make it manage and serve up the
 * datafiles in conjunction with a page which uses highcharts JS to graph it.
 * This is basically accomplished using the arduino by itself. Because I
 * actually host the highcharts.js files externally, this is true more in
 * theory than in actual practice, but oh well. It should work just fine to
 * have the highcharts.js file on the arduino's SD card, though loading the 
 * page will be painfully slow.
 *
 * Some of the code this was derived from may or may not be under a GPL
 * license; I'm not entirely sure. I suppose anyone using this should treat 
 * it like it is too, but I don't really care too much.
 * Also if one intends to use this for commercial applications, it may be
 * necessary to purchase a license for Highcharts.
 *
 * Changes:   -------------------------------------------------------------
 * January 2013: Updated so that the dd-mm-yy.csv file format is properly 
 * followed, all single digit days, months, and years will have a leading 
 * zero now. 
 *
 */
 
#include <sd.h>
#include <ethernet.h>
#include <ethernetudp.h>
#include <spi.h>
#include <string.h>
#include <time.h>
#include <eeprom.h>
#include <eepromanything.h>
#include <avr/pgmspace.h>
 
/************ ETHERNET STUFF ************/
byte mac[] = { 0x90, 0xA2, 0xDA, 0x00, 0x4C, 0x64 };
byte ip[] = { 192,168,1, 100 };
EthernetServer server(80);
 
/************** NTP STUFF ***************/
unsigned int localPort = 8888;          // local port to listen for UDP packets
IPAddress timeServer(132, 163, 4, 101); //NIST time server IP address: for more info
                                        //see  http://everettsprojects.com
 
const int NTP_PACKET_SIZE= 48; //NTP time stamp is in the first 48 bytes of the message
byte packetBuffer[ NTP_PACKET_SIZE]; //buffer to hold incoming and outgoing packets 
EthernetUDP Udp;
 
/*** DATA LOGGER AND TIMER CONTROLS ****/
const int analogPin = 0;
unsigned long lastIntervalTime = 0; //The time the last measurement occurred.
#define MEASURE_INTERVAL 600000     //10 minute intervals between measurements (in ms)
unsigned long newFileTime;          //The time at which we should create a new week's file
#define FILE_INTERVAL 604800        //One week worth of seconds
 
//A structure that stores file config variables from EEPROM
typedef struct{                     
    unsigned long newFileTime;      //Keeps track of when a new file should be made.
    char workingFilename[19];       //The path and filename of the current week's file
} configuration;
   
configuration config;               //Actually make our config struct
 
 
// Strings stored in flash mem for the HTML Header (saves ram)
prog_char HeaderOK_0[] PROGMEM = "HTTP/1.1 200 OK";            //
prog_char HeaderOK_1[] PROGMEM = "Content-Type: text/html";    //
prog_char HeaderOK_2[] PROGMEM = "";                           //
 
// A table of pointers to the flash memory strings for the header
PROGMEM const char *HeaderOK_table[] = {   
  HeaderOK_0,
  HeaderOK_1,
  HeaderOK_2
};
 
// A function for easy printing of the headers  
void HtmlHeaderOK(EthernetClient client) {
   
    char buffer[30]; //A character array to hold the strings from the flash mem
     
    for (int i = 0; i < 3; i++) {
      strcpy_P(buffer, (char*)pgm_read_word(&(HeaderOK_table[i]))); 
      client.println( buffer );
    }
} 
   
   
// Strings stored in flash mem for the Html 404 Header
prog_char Header404_0[] PROGMEM = "HTTP/1.1 404 Not Found";     //
prog_char Header404_1[] PROGMEM = "Content-Type: text/html";    //
prog_char Header404_2[] PROGMEM = "";                           //
prog_char Header404_3[] PROGMEM = "<h2>File Not Found!</h2>"; 
 
// A table of pointers to the flash memory strings for the header
PROGMEM const char *Header404_table[] = {   
  Header404_0,
  Header404_1,
  Header404_2,
  Header404_3
};
 
// Easy peasy 404 header function
void HtmlHeader404(EthernetClient client) {
   
    char buffer[30]; //A character array to hold the strings from the flash mem
     
    for (int i = 0; i < 4; i++) {
      strcpy_P(buffer, (char*)pgm_read_word(&(Header404_table[i]))); 
      client.println( buffer );
    }
} 
 
 
void setup() {
  Serial.begin(9600);
   
  pinMode(10, OUTPUT);          // set the SS pin as an output (necessary!)
  digitalWrite(10, HIGH);       // but turn off the W5100 chip! 
   
  // see if the card is present and can be initialized:
  if (!SD.begin(4)) {
    Serial.println("Card failed, or not present");
    // don't do anything more:
    return;
  }
  Serial.println("card initialized.");
   
  // The SD card is working, start the server and ethernet related stuff!
  Ethernet.begin(mac, ip);
  server.begin();
  Udp.begin(localPort);
  EEPROM_readAnything(0,config); // make sure our config struct is syncd with EEPROM
}
 
 
// A function that takes care of the listing of files for the
// main page one sees when they first connect to the arduino.
// it only lists the files in the /data/ folder. Make sure this
// exists on your SD card.
void ListFiles(EthernetClient client) {
   
  File workingDir = SD.open("/data");
   
  client.println("<ul>");
   
    while(true) {
      File entry =  workingDir.openNextFile();
       if (! entry) {
         break;
       }
       client.print("<li><a href=\"/HC.htm?file=");
       client.print(entry.name());        
       client.print("\">");
       client.print(entry.name());
       client.println("</a></li>");
       entry.close();
    }
  client.println("</ul>");
  workingDir.close();
}
 
// A function to get the Ntp Time. This is used to make sure that the data
// points recorded by the arduino are referenced to some meaningful time
// which in our case is UTC represented as unix time (chosen because it 
// works simply with highcharts without too much unnecessary computation).
unsigned long getTime(){
  sendNTPpacket(timeServer); // send an NTP packet to a time server
 
  // wait to see if a reply is available
  delay(1000);  
  if ( Udp.parsePacket() ) {  
    // We've received a packet, read the data from it
    Udp.read(packetBuffer,NTP_PACKET_SIZE);  // read the packet into the buffer
 
    //the timestamp starts at byte 40 of the received packet and is four bytes,
    // or two words, long. First, esxtract the two words:
 
    unsigned long highWord = word(packetBuffer[40], packetBuffer[41]);
    unsigned long lowWord = word(packetBuffer[42], packetBuffer[43]);  
    // combine the four bytes (two words) into a long integer
    // this is NTP time (seconds since Jan 1 1900):
    unsigned long secsSince1900 = highWord << 16 | lowWord;  
    // Unix time starts on Jan 1 1970. In seconds, that's 2208988800:
    const unsigned long seventyYears = 2208988800UL;     
    // subtract seventy years:
    unsigned long epoch = secsSince1900 - seventyYears;  
    // return Unix time:
    return epoch;
  }
}
 
// send an NTP request to the time server at the given address,
// necessary for getTime().
unsigned long sendNTPpacket(IPAddress& address){
   
  // set all bytes in the buffer to 0
  memset(packetBuffer, 0, NTP_PACKET_SIZE); 
  // Initialize values needed to form NTP request
  // (see URL above for details on the packets)
  packetBuffer[0] = 0b11100011;   // LI, Version, Mode
  packetBuffer[1] = 0;     // Stratum, or type of clock
  packetBuffer[2] = 6;     // Polling Interval
  packetBuffer[3] = 0xEC;  // Peer Clock Precision
  // 8 bytes of zero for Root Delay & Root Dispersion
  packetBuffer[12]  = 49; 
  packetBuffer[13]  = 0x4E;
  packetBuffer[14]  = 49;
  packetBuffer[15]  = 52;
 
  // all NTP fields have been given values, now
  // you can send a packet requesting a timestamp:         
  Udp.beginPacket(address, 123); //NTP requests are to port 123
  Udp.write(packetBuffer,NTP_PACKET_SIZE);
  Udp.endPacket(); 
}
 
 
// How big our line buffer should be for sending the files over the ethernet.
// 75 has worked fine for me so far.
#define BUFSIZ 75

void loop(){
  if ((millis() % lastIntervalTime) >= MEASURE_INTERVAL){ //Is it time for a new measurement?
      
    char dataString[20] = "";
    int count = 0;
    unsigned long rawTime;
    rawTime = getTime();
 
    while((rawTime == 39) && (count < 12)){     //server seems to send 39 as an error code
      delay(5000);                              //we want to retry if this happens. I chose
      rawTime = getTime();                      //12 retries because I'm stubborn/persistent.
      count += 1;                               //NIST considers retry interval of <4s as DoS
    }                                           //attack, so fair warning.
     
    if (rawTime != 39){                         //If that worked, and we have a real time
       
      //Decide if it's time to make a new file or not. Files are broken
      //up like this to keep loading times for each chart bearable.
      //Lots of string stuff happens to make a new filename if necessary.
      if (rawTime >= config.newFileTime){
        int dayInt = day(rawTime);
        int monthInt = month(rawTime);
        int yearInt = year(rawTime);
        char newFilename[18] = "";
        char dayStr[3];
        char monthStr[3];
        char yearStr[5];
        char subYear[3];
        strcat(newFilename,"data/");
        itoa(dayInt,dayStr,10);
        if (dayInt < 10){
          strcat(newFilename,"0");
        }
        strcat(newFilename,dayStr);
        strcat(newFilename,"-");
        itoa(monthInt,monthStr,10);
        if (monthInt < 10){
          strcat(newFilename,"0");
        }
        strcat(newFilename,monthStr);
        strcat(newFilename,"-");
        itoa(yearInt,yearStr,10);
        //we only want the last two digits of the year
        memcpy( subYear, &yearStr[2], 3 );
        strcat(newFilename,subYear);
        strcat(newFilename,".csv");
         
        //make sure we update our config variables:
        config.newFileTime += FILE_INTERVAL;
        strcpy(config.workingFilename,newFilename);
        //Write the changes to EEPROM. Bad things may happen if power is lost midway through,
        //but it's a small risk we take. Manual fix with EEPROM_config sketch can correct it.
        EEPROM_writeAnything(0, config); 
      }
         
      //get the values and setup the string we want to write to the file
      int sensor = analogRead(analogPin);  
      char timeStr[12];
      char sensorStr[6];
       
      ultoa(rawTime,timeStr,10); 
      itoa(sensor,sensorStr,10);
       
      strcat(dataString,timeStr);
      strcat(dataString,",");
      strcat(dataString,sensorStr);
       
      //open the file we'll be writing to.
      File dataFile = SD.open(config.workingFilename, FILE_WRITE);
   
      // if the file is available, write to it:
      if (dataFile) {
        dataFile.println(dataString);
        dataFile.close();
        // print to the serial port too:
        Serial.println(dataString);
      }  
      // if the file isn't open, pop up an error:
      else {
        Serial.println("Error opening datafile for writing");
      }
    }
    else{
      Serial.println("Couldn't resolve a time from the Ntp Server.");
    }
    //Update the time of the last measurment to the current timer value
    lastIntervalTime = millis();
  }
  //No measurements to be made, make sure the webserver is available for connections.
  else{
    char clientline[BUFSIZ];
    int index = 0;
     
    EthernetClient client = server.available();
    if (client) {
      // an http request ends with a blank line
      boolean current_line_is_blank = true;
       
      // reset the input buffer
      index = 0;
       
      while (client.connected()) {
        if (client.available()) {
          char c = client.read();
           
          // If it isn't a new line, add the character to the buffer
          if (c != '\n' && c != '\r') {
            clientline[index] = c;
            index++;
            // are we too big for the buffer? start tossing out data
            if (index >= BUFSIZ) 
              index = BUFSIZ -1;
             
            // continue to read more data!
            continue;
          }
           
          // got a \n or \r new line, which means the string is done
          clientline[index] = 0;
           
          // Print it out for debugging
          Serial.println(clientline);
           
          // Look for substring such as a request to get the root file
          if (strstr(clientline, "GET / ") != 0) {
            // send a standard http response header
            HtmlHeaderOK(client);
            // print all the data files, use a helper to keep it clean
            client.println("<h2>View data for the week of (dd-mm-yy):</h2>");
            ListFiles(client);
          }
          else if (strstr(clientline, "GET /") != 0) {
            // this time no space after the /, so a sub-file!
            char *filename;
             
            filename = strtok(clientline + 5, "?"); // look after the "GET /" (5 chars) but before
            // the "?" if a data file has been specified. A little trick, look for the " HTTP/1.1"
            // string and turn the first character of the substring into a 0 to clear it out.
            (strstr(clientline, " HTTP"))[0] = 0;
             
            // print the file we want
            Serial.println(filename);
            File file = SD.open(filename,FILE_READ);
            if (!file) {
              HtmlHeader404(client);
              break;
            }
             
            Serial.println("Opened!");
                       
            HtmlHeaderOK(client);
             
            int16_t c;
            while ((c = file.read()) > 0) {
                // uncomment the serial to debug (slow!)
                //Serial.print((char)c);
                client.print((char)c);
            }
            file.close();
          }
          else {
            // everything else is a 404
            HtmlHeader404(client);
          }
          break;
        }
      }
      // give the web browser time to receive the data
      delay(1);
      client.stop();
    }
  }
}
 
```

### The Results

![Fig.1](online-graph-results.jpg)
![Fig.2](online-graph-results2.jpg)
![Fig.3](online-graph-results3.jpg)



## Web Server

In the following example, we will create a sketch that turns your Mega
ETH board to a Web Server that serves just one static HTML page that it
reads from a file on an SD card.

### Creating the Web Page

Let's create a simple HTML page and store it on an SD card using your
computer and a card reader. While you can use almost any text editor to
write HTML code, it will be much easier if it can highlight the HTML
syntax and automatically close HTML tags. Geany (geany.org) is an
example of such an editor available on almost all operating systems.
Create the following web page in the editor:

``` {.sourceCode .HTML}
<!DOCTYPE html>
<html>
    <head>
        <title>Test SD Card Web Page</title>
    </head>
    <body>
        <h1>Hello from the Mega ETH Web Server!</h1>
        <p>An example web page stored on the SD card.</p>
    </body>
</html>
```

Store it on the SD card in the root directory as *index.html*.

### Preparing the Web-Server Sketch

We will need to configure the pin 4 for the SD card reader to work, then
we initialize the network library, generate a presumably unique MAC
address following our previous recommendations of setting the first byte
of MAC address to 0xDE for better network equipment compatibility and
store that MAC address in the microcontroller's EEPROM.

We will use DHCP to query an available IP address from the network
router and display the IP address on a 16x2 LCD monitor because we need
to type that IP address into a browser, that must be running on a
computer on the same local network.

We assume that the HTML file is already stored on the SD card and the
card is inserted into the onboard card reader.

Let's get down to the code:

```cpp
#include <SPI.h>
#include <Ethernet3.h>
#include <EEPROM.h>
#include <LiquidCrystal.h>
#include <SD.h>

byte mac[6] = { 0xDE, 0x00, 0x00, 0x00, 0x00, 0x00 };
char macstr[18];

// initialize the LCD library with the numbers of the interface pins
LiquidCrystal lcd(12, 11, 5, 4, 3, 2);

EthernetServer server(80);  // create a server at port 80

void setup() {

  lcd.begin(16, 2); //Initialize the 16x2 LCD screen

  // Check EEPROM byte with offset==0 for “#” char. The presence of “#” in the first byte of EEPROM will be a sign that the MAC address was already generated and stored in bytes 1-7 of EEPROM. Otherwise, we need to generate a random MAC, put “#” at offset 0 and the MAC in the following 6 bytes
  if (EEPROM.read(0) == '#') {
    for (int i = 1; i <= 6; i++) {
      mac[i-1] = EEPROM.read(i);
    }
  } else {
    randomSeed(analogRead(0));
    mac[0] = 0xde;
    EEPROM.write(1, mac[0]);
    for (int i = 1; i < 6; i++) {
      mac[i] = random(0, 255);
      EEPROM.write(i+1, mac[i]);
    }
    EEPROM.write(0, '#');
  }

  // Startup networking, get IP, DNS, and subnet from the DHCP server

  if (Ethernet.begin(mac) == 0) {
    if (Ethernet.linkStatus() == LinkOFF) {
      lcd.print("LINK FAILURE");
      exit()
    }
    else {
            lcd.print("DHCP ERROR!");
            exit(0)
    }
  } else // success, display your local IP address:
         lcd.print(Ethernet.localIP()); //You can enter this IP in your web browser
}

void loop() {
  // listen for incoming clients
  EthernetClient client = server.available();
  if (client) {  // got client?
        boolean currentLineIsBlank = true;
        while (client.connected()) {
            if (client.available()) {   // client data available to read
                char c = client.read(); // read 1 byte (character) from client
                // last line of client request is blank and ends with \n
                // respond to client only after last line received
                if (c == '\n' && currentLineIsBlank) {
                    // send a standard http response header
                    client.println("HTTP/1.1 200 OK");
                    client.println("Content-Type: text/html");
                    client.println("Connection: close");
                    client.println();
                    // send web page
                    webFile = SD.open("index.htm");        // open web page file
                    if (webFile) {
                        while(webFile.available()) {
                            client.write(webFile.read()); // send web page to client
                        }
                        webFile.close();
                    }
                    break;
                }
                // every line of text received from the client ends with \r\n
                if (c == '\n') {
                    // last character on line of received text
                    // starting new line with next character read
                    currentLineIsBlank = true;
                } 
                else if (c != '\r') {
                    // a text character was received from client
                    currentLineIsBlank = false;
                }
            } // end if (client.available())
        } // end while (client.connected())
        delay(1);      // give the web browser time to receive the data
        client.stop(); // close the connection
    } // end of session with client
}

```

## Saving Temperature Sensor Data to Google Docs


This example uses a free helper service from pushingbox.com to store
temperature sensor data to a Google Sheets document.

### Get Accounts


1.  A regular Google account. Don\'t use your main Gmail account please)
2.  Get a user account on pushingbox.com. FREE!

### Create a Google Form and Get FORM KEY

----------------------------------------------------------------------------
  ![Fig 1. Create a new form](getformkey.jpg)   ![Fig 2. Make a simple TEXT form and click Send Form](getformkey2.jpg)
  ![Fig 3. Copy form URL](getformkey3.jpg)      ![Fig 4. View page source and copy the Text input box ID](getformkey4.jpg)
----------------------------------------------------------------------------

Go to your new Google Drive and create a form. In this example I use a
single question TEXT response form. Click *send form* when done.

There will be a pop-up shown with an option to copy a URL - click that
to copy this URL. Open this URL in another tab of your browser. This
will get you to the newly created form. Now,right-click and select
*Inspect Elements*. You will see the HTML code of the form. What you
need to find is a line of code that defines the Text input box,
something like *entry.123456789456123* (numbers will be different). This
is your **form key**, you will need to copy it.

### Pushingbox, Service and Scenarios

 ----------------------------------------------------------------------------------------------
  ![Fig 1. Create a new custom service](pushingbox.jpg)   ![Fig 2. Copy response page URL and paste to pushingbox.com Service URL field](pushbox2.jpg)
  ------------------------------------------------------- 

Log in to pushingbox.com using the same Google account. Go to *My
Services* and create a new custom service. Now please go to your Google
form and submit something. This loads a response page of a form. Copy
it\'s URL and paste to pushingbox.com Service URL field. Give it some
good name and set method to GET.

Now go to my scenarios and paste into a scenario field:

```html

?entry.1917223082=$status$&submit=Submit

```

Change the numbers to your own FORM KEY. Copy the whole string and paste
it into the *add scenario field*. Click ADD. Pushingbox should confirm.
Click **ADD AN ACTION** and select the Service you created a moment ago.

This will give you a DEVICE ID remember this, in fact. write it down or
copy and paste it into a notepad doc. This will be used in the Mega ETH
code.

Congrats, you are done with that and now its time to work on the sketch
for the Mega ETH.

### Sketch for the Mega ETH

Here we post a very simple code to run this example. You will find a
section of things to be edited. This code will send responses to the
form you created in the form of data. You will see google then creates a
spreadsheet and logs all the responses WITH time stamps! Use this to
create data logs of temperature, light levels, humidity, motion whatever
you want! The example attached uses a DHT11 thermo/humidity sensor. You
can easily modify it for your actual purpose.

```c

#include <SPI.h>
#include <Ethernet3.h>
#include <EthernetUdp.h>
#include <SPI.h>
#include <dht11.h> 

#undef int
#undef abs
#undef double
#undef float
#undef round
dht11 DHT11;
#define DHT11PIN 3

///////////////////////////////
///      EDIT THIS STUFF     //
///////////////////////////////

byte mac[] = {0xAA, 0xBB, 0xCC, 0xDD, 0xEE, 0xFF };  //Replace with your MAC
byte ip[] = {192,168,0,50};     // Your Mega ETH device IP address

char devid = v42FE15BC09B20df  // THIS IS THE DEVICE ID FROM PUSHINGBOX

int del=300;  // Amount of seconds delay between posting to google docs.

///////////////////////////////
//       DONE EDITING        //
///////////////////////////////

char postmsg[100];
int k=0;
int temp_av = 0;
char server[] = "api.pushingbox.com";
EthernetClient client;

void setup()
{
  Serial.begin(9600);
  Ethernet.begin(mac, ip);
  delay(1000);
  Serial.println("connecting...");
}

void loop(){

// average temp reading for 'del' time.........................................

  for(int j=0; j<del;j++)
  {
    // Read local temp........................................
    int chk = DHT11.read(DHT11PIN);
    int temp = Fahrenheit(DHT11.temperature);
    temp_av=temp_av+temp;
    delay(1000);
  }

  int avtemp=temp_av/(del);
  temp_av=0;




  // Post to Google Form.............................................
  if (client.connect(server, 80)) 
  {
    k=0;
    Serial.println("connected");
    sprintf(postmsg,"GET /pushingbox?devid=%c&status=%d HTTP/1.1",devid,avtemp);  // NOTE** In this line of code you can see where the temperature value is inserted into the wed address. It follows 'status=' Change that value to whatever you want to post.
    client.println(postmsg);
    client.println("Host: api.pushingbox.com");
    client.println("Connection: close");
    client.println();

    Serial.println(postmsg);
    Serial.println("Host: api.pushingbox.com");
    Serial.println("Connection: close");
    Serial.println();

    delay(1000);
    client.stop();
  }
  delay(1000);

  if (!client.connected()) 
  {
    Serial.println();
    Serial.println("disconnecting.");
    client.stop();
    k==1;
    return;
  }


}


double Fahrenheit(double celsius) // Function to convert to Fahrenheit
{
    return 1.8 * celsius + 32;
}
    
```




