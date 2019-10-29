# IP address

IP address is a numerical identifier assigned to each device that is connected to a network that uses Internet Protocol for communication. An IP address can be assigned either manually, by a network administrator, or automatically by a network router or a similar device. The former is called "static IP address setup", the latter is called "dynamic" or DHCP setup, since it uses the DHCP protocol for the IP address quereries and assignements.

## IPv4 vs. IPv6

There are 2 version of the IP protocol and addresses: IPv4 and, newer, IPv6. IPv4 defines an IP address as a 32-bit integer number denoted as 4 0-255 values separated with ".", e.g.: 192.168.0.1
Because of the depletion of the IPv4 IP address space the newer IPv6 protocol uses 128-bit addresses, denoted as 6 hexadecimal numbers (0x0 - 0xFFFF) separated by ":", e.g.: 2001:db8:0:1234:0:567:8:1
The standard Arduino networking libraries (Ethernet3, Ethernet4) support IPv4 only. The W5500 chip itself supports only IPv4 natively, however there is a low-level MACRAW mode for sending Ehternet frames directly and there are some alternative networking libraries (e.g. [Ethersia](https://github.com/njh/ethersia))with IPv6 support built-in.

## Hard-coding IP address into the sketch

The simplest way to assign an IP address to your Mega ETH-based device is to hard-code the IP address into the sketch:

```c
#include <SPI.h>
#include <Ethernet3.h>

// network configuration. dns server, gateway and subnet are optional.

 // the media access control (ethernet hardware) address:
byte mac[] = { 0xDE, 0xAD, 0xBE, 0xEF, 0xFE, 0xED };  

// the dns server ip
IPAddress dnServer(192, 168, 0, 1);
// the router's gateway address:
IPAddress gateway(192, 168, 0, 1);
// the subnet:
IPAddress subnet(255, 255, 255, 0);

//the IP address is dependent on your network
IPAddress ip(192, 168, 0, 2);

void setup() {
  Serial.begin(9600);

  // initialize the ethernet device
  Ethernet.begin(mac, ip, dnServer, gateway, subnet);
  //print out the IP address
  Serial.print("IP = ");
  Serial.println(Ethernet.localIP());
}

void loop() {
}

```

## Dynamic IP address assignement

If you are going to connect several Mega ETH devices to the same network segment, hard-coding an IP address might not be very convinient, since you don't want to change the sketch for every device you deploy. In this case and in all cases where DHCP service is available on the network, it would be much better to automatically obtain an IP address from the DHCP server and store it in non-volatile memory, EEPROM or SD card, because, in most cases, it is desirable for a network device to retain it's assigned IP address every time it is powered up.
In the following example Mega ETH will obtain an IP address via DHCP protocol using the [Ethernet3 library](https://github.com/sstaub/Ethernet3) and display it on a 16x24 LCD monitor:

```c
/*

  This sketch uses the DHCP to get an IP address
  and display the address on an LCD.

 */

#include <SPI.h>
#include <Ethernet3.h>
#include <LiquidCrystal.h>

// Enter a MAC address for your controller below.

byte mac[] = {
  0xDE, 0xAB, 0xCD, 0xEF, 0xDE, 0x02
};
// initialize the LCD library with the numbers of the interface pins
LiquidCrystal lcd(12, 11, 5, 4, 3, 2);

void setup() {
 
  
  lcd.begin(16, 2);

  if (Ethernet.begin(mac) == 0) {
    if (Ethernet.linkStatus() == LinkOFF) {
      lcd.print("LINK FAILURE");
    }
    else lcd.print("DHCP ERROR!");
  } else // success, display your local IP address:
         lcd.print(Ethernet.localIP());

    // do nothing more:
  while (true) {
      delay(1);
  }
}

void loop() {
}
```


