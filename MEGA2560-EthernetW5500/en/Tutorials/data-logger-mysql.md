# Data Logger with MySQL

In this example we will build a simple temperature data logger that will send data to a remote MySQL database. Every mesurement will be accompanied with a corresponding timestamp, hence we'll need a sensor moduole and a real time clock module

## Hardware

1. Mega ETH board
2. DHT11 temperature/humidity sensor
3. RTC (Real Time Clock) DS1307 module
4. one breadboard
5. connecting wires

The RTC moduleuses I2C pins SDA and SCL, DHT11 - to pins 7 and 8 of the Mega ETH board.


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

```
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
    minutes_next=minutes_now+1; //debgging - write every minute to DB
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
 

